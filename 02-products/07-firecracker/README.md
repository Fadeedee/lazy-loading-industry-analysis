# Firecracker

> 阅读完成后，读者能够区分 Firecracker 已发布的 memory snapshot 懒恢复、现有 file-backed virtio-pmem 和尚未实现的 lazy pmem issue #5740。

## 三条不同路径

| 路径 | 对象 | 机制 | 状态 |
| --- | --- | --- | --- |
| Snapshot `File` backend | guest RAM | memory file `MAP_PRIVATE`，内核 fault-in | `released` |
| Snapshot `Uffd` backend | guest RAM | 外部 handler + `UFFDIO_COPY` | `released` |
| Issue #5740 lazy pmem | rootfs pmem | anonymous HVA + UFFD + FD/MAP_FIXED | `proposal` |

另外，Firecracker 已支持普通 file-backed virtio-pmem，但要求 backing file 已完整可读，不负责 registry range fetch。[SRC-FC-003]

## 最重要的边界

Firecracker 已证明外部 UFFD handler 可以参与 memory restore，也明确记录 handler 失败可能让 microVM 永久等待。[SRC-FC-002] Issue #5740 在此基础上提出 lazy pmem 协议，但 issue 作者的 prototype 说明不能等同于上游产品代码。[SRC-FC-004]

来源：[SRC-FC-001] [SRC-FC-002] [SRC-FC-003] [SRC-FC-004]
