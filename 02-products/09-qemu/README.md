# QEMU Fast Snapshot Load

> 阅读完成后，读者能够理解 QEMU 如何复用 migration/postcopy 基础设施按需恢复 guest RAM，并区分它与 rootfs image lazy loading。

## 定位

QEMU Fast Snapshot Load 使用 mapped-ram snapshot layout 与 postcopy infrastructure，使 VM 可以在 memory snapshot 尚未全部加载时恢复执行；faulting page 优先加载，其余页后台补齐。[SRC-QEMU-001]

## 关键价值

- fault priority 与 background loader 协同；
- bitmap 跟踪哪些 page 尚未加载；
- mapped-ram layout 便于按 page offset 定位；
- 复用成熟 migration/postcopy 机制，而不是另写一套 page fault engine。

## 边界

对象是 guest RAM，不是 OCI rootfs，也不负责 EROFS/DAX pmem。其调度和状态机可借鉴，数据格式不能直接复用。

来源：[SRC-QEMU-001]
