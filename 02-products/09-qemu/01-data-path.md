# QEMU Fast Snapshot Load 数据路径

> 阅读完成后，读者能够理解 snapshot page 的按需读取、优先级和后台加载如何配合。

```text
load VM/device state + mapped-ram metadata
  -> establish RAM mapping and pending-page bitmap
  -> resume destination VM
  -> page fault enters postcopy path
  -> synchronously load requested page
  -> clear pending state and resume vCPU
  -> background loader processes remaining pages
```

mapped-ram 为 RAM blocks 提供可按 offset 定位的文件布局，避免恢复器必须顺序扫描完整 migration stream。[SRC-QEMU-001]

## 调度意义

faulting vCPU 请求是高优先级；后台加载用于减少未来 fault。两者需要共享线程安全的 page state，否则可能重复读取、覆盖或错误唤醒。

这与 lazyd 的 `inflight[digest, range]` 很相似，但一个追踪 memory pages，一个追踪 image content ranges。
