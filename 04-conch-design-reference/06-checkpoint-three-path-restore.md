# Checkpoint/Template 三路恢复设计

> 阅读完成后，读者能够解释一次 checkpoint template 恢复时 rootfs lower、writable disk 和 guest RAM 如何并行准备、分别 fault，并由 Conch 统一失败和生命周期。

## 资源图

```text
Checkpoint Template / Boot Index
  ├── PreparedRootfs
  │    └── EROFS layer digest -> lazyd instance
  ├── WritableDiskSnapshot
  │    └── parent + changed block layers
  ├── MemorySnapshot
  │    └── full-v1 或 incremental-v1 base/delta
  └── Sandbox/VM assets
       └── kernel + initrd + VMM state
```

## 准备阶段

Conch 并行启动或连接：

1. **rootfs**：确认 lazyd prepared instances 可恢复、StratoVirt lazy pmem specs 完整。
2. **disk**：建立 read-only parent chain 和新的 private writable head。
3. **memory**：full file 可访问，或 conch-cow Attach 返回 memfd/attachment。
4. **VM assets**：kernel/initrd/VMM state完整且版本匹配。

只有每路达到 `Faultable`，才启动/恢复 vCPU。

## 运行阶段

```text
rootfs file read
  -> EROFS+DAX -> pmem GPA -> lazy HVA UFFD
  -> StratoVirt -> lazyd -> cache fd/remap/wake

writable file read/write
  -> ext4 -> virtio-blk
  -> block parent/COW cache

guest instruction/data access
  -> guest RAM HVA UFFD
  -> memory restore source/conch-cow -> memfd page/wake
```

三路 fault 可以并发。Conch 不处理每次 fault，但必须监听 StratoVirt、block backend 和 conch-cow 的 fatal status，并将任一不可恢复错误转成 Sandbox failure/cleanup。

## 与 PR #155 的组合

PR #155 的 `incremental-v1` 用 build map确定每段 guest memory最终所属 layer，conch-cow 将页 `pwrite` 到 inherited memfd 后 `UFFDIO_WAKE`。[SRC-CONCH-003] 这与 rootfs FD remap不同：

- memory写入一个 VM 私有 memfd；
- rootfs映射共享、不可变 EROFS cache fd；
- memory build map按 checkpoint lineage；
- rootfs bitmap按 OCI digest/range。

## 创建新 checkpoint

1. pause/quiesce VM；
2. 固化 writable disk changed blocks；
3. 获取 memory dirty generation并生成 base/delta；
4. 保存外置 VMM/device state；
5. 发布每类 descriptor 和新的 Boot Index；
6. 原子推进 checkpoint head；
7. 建立新的 writable/dirty generation 后 resume。

immutable rootfs lower通常只被新 Boot Index继续引用，不重新保存；若镜像配置发生变化，则创建新的 PreparedRootfs引用集合。

## 统一状态，不统一数据协议

推荐每个 resource driver暴露：

- `Prepare()`；
- `ReadyLevel()`：MetadataReady/Faultable/Materialized；
- `Failure()`；
- `Acquire/Release()`；
- metrics/progress。

其内部数据面继续分别使用 lazyd FETCH、block I/O 和 memory UFFD。这样统一恢复事务，又不把不同正确性语义混在一起。
