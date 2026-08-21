# Agent Note: Refuse JSONL appends that overlap committed seqs

Status: implemented

English | [中文](2026-08-20-jsonl-append-seq-overlap-refusal.zh.md)

## Problem

A JSONL session log is append-only, and every event row's `seq` must equal its global row index. `Session.append` assigns `seq` from its in-memory log length, and the backend coordinator validates batch seqs against its own in-memory cursor. When a second backend instance or process writes the same session file (documented as unsupported), the durable log can advance past those in-memory counters: a cold `load()` crash-repairs an open turn by committing synthetic interrupted closers through `commitRepair`, which bypasses the first instance's cursor. The first instance then resumes from its stale counters and re-appends rows whose seqs duplicate already-committed events. A real deployment produced a log whose committed region ended at seq 47259 and then replayed `assistant/chunk` rows from seq 47258, leaving the artifact unloadable.

The reader compounded the problem: the scanner latches a `seq gap in committed region` defect, but `readZstdPrefix` reported the generic `corrupt Zstandard session log: complete frame contains a torn JSONL record` whenever committed bytes fell short of input bytes, because a logically defective frame is still structurally complete after the gap. Operators diagnosed byte corruption instead of the actual two-writer divergence.

## Decision

The JSONL backend records the durable file identity (the `FileRevisionIdentity` — dev/ino/size/mtimeNs/ctimeNs — shared with [storage identity tracking](2026-07-20-jsonl-storage-identity.md)) of each session after its own successful materialization and appends. When an append's pre-write stat differs from that recorded identity, the append re-reads the stored prefix and refuses the batch if its first `seq` is lower than the durable event count; the refusal message names the session, the batch start, the committed count, and the remedy (reopen the session or start a new turn). A batch that still continues the durable count appends normally. A durable log that ends in a torn tail left by a foreign writer is refused on the same check: appending after the torn bytes would let the reader's torn-tail recovery silently discard the new batch, so the append defers to a `load()`, which commits the truncation. The guard keeps the existing rollback and one-live-writer contract intact: it converts silent corruption into a loud, actionable failure at the point of write. The identity map is per-backend-instance memory and carries no format change.

On the read side, `SessionLogScanner` exposes its latched defect through `firstIssue`, and `readZstdPrefix` throws that defect when a complete frame froze `committedBytes` below `inputBytes`, falling back to the torn-record message only when no logical defect was latched.

## Alternatives considered

**Re-synchronize a stale writer from the durable log instead of refusing.** A stale append could reload the stored prefix and rewind its cursor. Reloading a log a live session is actively appending to risks dropping unflushed events and reintroduces the divergence the guard exists to catch; refusing leaves remediation to the operator.

**Bump `SESSION_FORMAT_VERSION` or change the on-disk rows.** The artifact is produced by unsupported two-writer use, not by the format. Rewriting frames or rows would break reading of all existing logs for a defect that a writer-side check prevents.

## Consequences

A second backend instance or process advancing a shared session now fails the next append loudly instead of producing a corrupt artifact, and a log that already holds a duplicate or gap in its committed region loads with a `seq gap in committed region` diagnosis instead of a misleading torn-record message. The one-live-writer-per-session limitation remains: this is a blast-radius backstop, not a multi-writer coordination mechanism. On-disk rows, frames, and `SESSION_FORMAT_VERSION` are unchanged. Tests pin the refusal of an overlapping batch, the allowed equal-count continuation, the seq-gap diagnosis, and an untouched repaired log after a refused append.
