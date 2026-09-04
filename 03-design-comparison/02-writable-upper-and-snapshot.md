# 可写 Upper 与磁盘快照比较

> 阅读完成后，读者能够理解只读 lazy lower 上方如何承载写入，并为增量 rootfs 快照选择文件级 overlay 或块级 COW。

## 方案比较

| 方案 | lower | write path | 增量对象 | 适合 guest ext4 | 主要样本 |
| --- | --- | --- | --- | --- | --- |
| OverlayFS upperdir | 文件系统 mount | host/guest upper files | upperdir/file tree | 取决于部署 | stargz/Nydus/containerd EROFS |
| 块级 COW upper | read-only virtual disk | changed blocks | block diff/child layer | 是 | OverlayBD、E2B |
| 复制完整磁盘 | full disk image | private disk | full image | 是 | 简单但放大明显 |
| pmem backing write | pmem mapping | mapped file pages | 未自然形成 layer | 不适合作为当前 lower 写层 | Firecracker pmem warning |

来源：[SRC-OVERLAYBD-001] [SRC-E2B-001] [SRC-EROFS-003] [SRC-FC-003]

## 为什么顶层常用 blk

ext4 是读写块文件系统，会更新 inode、journal、allocation bitmap 和 data blocks。virtio-blk/NBD/OverlayBD 后端天然观察并保存这些 block writes；只读 EROFS+DAX pmem 则设计为不可变 lower，不适合承载 journal 和共享写入。

因此可以同时存在：

```text
lower0..N: EROFS + DAX + virtio-pmem (shared, readonly)
upper:     ext4 + virtio-blk + COW     (sandbox-private, writable)
```

guest 通过 overlay 或运行时定义的组合获得最终 rootfs 视图。设备数量、layer merge 和 mount strategy 需要 Conch/guestd 统一管理。

## snapshot 一致性

创建 checkpoint 时不能只复制 upper blocks：

1. pause/quiesce workload 或建立一致性屏障；
2. flush guest filesystem/block backend；
3. 固化 writable head 为 snapshot child；
4. 保存与其同一时间点的 VM/device state 和 memory snapshot；
5. 记录 parent lineage 和内容校验；
6. 创建新的 writable head 后再 resume。

Firecracker 明确把 disk consistency 留给使用者，说明 Conch 必须承担这层协调。[SRC-FC-001]

## 推荐

- 第一阶段：rootfs lower 只读懒加载，不改现有 writable 行为。
- 第二阶段：把 ext4 upper 建模为独立 block resource，支持 full/diff snapshot。
- 第三阶段：与 memory snapshot 形成统一 checkpoint transaction。
