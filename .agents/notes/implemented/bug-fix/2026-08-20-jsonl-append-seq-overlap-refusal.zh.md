# Agent Note: 拒绝与已提交 seq 重叠的 JSONL 追加

Status: implemented

[English](2026-08-20-jsonl-append-seq-overlap-refusal.md) | 中文

## 问题

JSONL 会话日志是只追加的，每条事件行的 `seq` 必须等于全局行索引。`Session.append` 根据内存中的日志长度分配 `seq`，后端协调器也只对照自己的内存游标校验批次的 seq。当第二个后端实例或进程写入同一会话文件（已被文档标记为不支持）时，持久化日志可能推进到这些内存计数器之前：冷 `load()` 为打开中的 turn 生成合成的 interrupted closers，并通过 `commitRepair` 提交——该路径绕过了第一个实例的游标。随后第一个实例从过期计数器继续，重放的 `seq` 与已提交事件重复。真实部署中出现过已提交区域止于 seq 47259、随后从 seq 47258 重放 `assistant/chunk` 行的产物，日志因此不可加载。

读取侧加重了问题：扫描器会锁存 `seq gap in committed region` 缺陷，但只要已提交字节数低于输入字节数，`readZstdPrefix` 就报出通用的 `corrupt Zstandard session log: complete frame contains a torn JSONL record`——因为出现缺口后帧在结构上仍是完整的。运维人员因此误判为字节损坏，而不是真正的双写方发散。

## 决策

JSONL 后端在自己每次成功的物化与追加之后，记录每个会话的持久化文件身份（`FileRevisionIdentity`——dev/ino/size/mtimeNs/ctimeNs，与 [storage identity 跟踪](2026-07-20-jsonl-storage-identity.md) 共用）。追加前的 stat 与该记录不一致时，追加会重新读取已存前缀；若批次首个 `seq` 低于持久化事件计数则拒绝写入，错误信息包含会话 id、批次起始 seq、已提交数量与处置方式（重新打开会话或开始新 turn）。仍能延续持久化计数的批次正常追加。若持久化日志以外部写入方留下的 torn 尾收尾，同一检查也会拒绝追加：在残缺字节之后追加会让读取侧的 torn 尾恢复静默丢弃该批次，因此追加让位于一次 `load()`（由它提交截断）。该守卫保留既有的回滚与单写者契约：把静默污染变成写入点上响亮、可操作的失败。身份映射是各后端实例的内存状态，不引入格式变更。

读取侧：`SessionLogScanner` 通过 `firstIssue` 暴露其锁存的缺陷，`readZstdPrefix` 在完整帧使 `committedBytes` 低于 `inputBytes` 时抛出该缺陷，仅在没有锁存逻辑缺陷时才回退到 torn-record 文案。

## 考虑过的替代方案

**不匹配时从持久化日志重同步过期的写入方，而不是拒绝。** 过期的追加可以重新加载已存前缀并回卷游标。重载一个活动会话正在追加的日志，会冒丢失未 flush 事件的风险，并重新引入该守卫要捕获的发散；拒绝则把补救交给操作者。

**提升 `SESSION_FORMAT_VERSION` 或改写磁盘行。** 该产物来自不支持的双写方使用，而非格式问题。改写帧或行会为了一个写侧检查即可阻止的缺陷，破坏对所有既有日志的读取。

## 后果

第二个后端实例或进程推进共享会话时，下一次追加会响亮地失败，而不是产生损坏产物；已提交区域内已有重复或缺口的日志加载时报出 `seq gap in committed region` 诊断，而不是误导性的 torn-record 消息。每会话单写者的限制保持不变：这是限制爆炸半径的兜底，不是多写者协调机制。磁盘行、帧与 `SESSION_FORMAT_VERSION` 均未改变。测试固定了重叠批次的拒绝、允许的等量延续、seq-gap 诊断，以及拒绝后已修复日志保持原样。
