# Guest RAM 懒恢复比较

> 阅读完成后，读者能够比较内核 file fault、external UFFD、internal UFFD 和 postcopy/page server，并为 StratoVirt 选择独立的 memory restore 路径。

## 比较矩阵

| 路线 | 样本 | page source | fault handler | 后台收敛 | 共享潜力 |
| --- | --- | --- | --- | --- | --- |
| file `MAP_PRIVATE` | Firecracker | local memory file | host kernel | 内核按访问 | 干净 file page 可共享，写时 COW |
| external UFFD copy | Firecracker、CRIU | memory file/page server | 独立 daemon | 可选 | 每 VM/process anonymous page |
| internal UFFD restore | Cloud Hypervisor v53 | snapshot ranges | VMM/restore daemon | 可选 | 依实现而定 |
| postcopy infrastructure | QEMU | mapped-ram/migration source | migration subsystem | 是 | 主要目标是恢复，不是共享 cache |

来源：[SRC-FC-001] [SRC-FC-002] [SRC-CH-001] [SRC-QEMU-001] [SRC-CRIU-001]

## 选择问题

### 本地 template memory file

优先考虑 `MAP_PRIVATE`：实现简单、内核 fault-in、干净页可复用。但 source 必须在 VM 生命周期内保持可访问，远端缺失页不能直接由普通 file mapping 表达。

### 远端或分层 memory source

需要 UFFD/postcopy：handler 可以按 fault page 拉取，也可后台加载。代价是故障处理、timeout、page state、source lease 和安全面更复杂。

### 增量快照

必须明确 base + diff 的页覆盖顺序。恢复器应先建立 page lookup view，再允许 vCPU fault，避免 fault path 临时遍历不稳定的远端 lineage。

## 与 rootfs UFFD 的共用边界

可以共用：

- userfaultfd 创建、API negotiation、register/unregister；
- event poll、地址范围检查、zeropage/copy/wake wrapper；
- handler thread lifecycle 与 fatal reporting。

不应共用：

- content identity；
- FETCH schema 和 range semantics；
- bitmap/on-disk state；
- source lifecycle 和 GC；
- prefault/prefetch policy。

Cloud Hypervisor #8239 的未合入结果直接支持这种谨慎分层。[SRC-CH-002]
