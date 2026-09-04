# E2B 与 Firecracker 沙箱栈

> 阅读完成后，读者能够理解一个公开沙箱系统如何同时组合 guest RAM 懒恢复、只读模板 rootfs、每实例 COW 写盘和两类增量快照。

## 定位

E2B Infra 的公开架构使用 Firecracker microVM、template cache 和 UFFD memory loading；rootfs 使用只读 template 加每 sandbox 的 COW cache，并通过进程内 NBD 暴露。pause/export 时，memory diff 与 rootfs block diff 分开保存。[SRC-E2B-001]

## 为什么重要

这是当前调研中最清晰的“三路分离、统一编排”样本：

- guest RAM：UFFD + template memfile + prefetch；
- rootfs/disk：NBD + read-only template + COW blocks；
- lifecycle：template build/cache、sandbox start/pause/resume/kill。

它没有证明 EROFS+DAX+pmem，但证明统一资源管理器不需要把所有数据塞进同一种后端。

来源：[SRC-E2B-001]
