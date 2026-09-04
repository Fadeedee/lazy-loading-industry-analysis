# OverlayBD

> 阅读完成后，读者能够理解 OverlayBD 为什么把容器镜像表示成可叠加块设备，以及这条路线对 ext4 可写层和磁盘增量快照的意义。

## 定位

OverlayBD 是面向容器和微虚机的块级远程镜像方案。它把只读镜像层组织成可叠加的虚拟块设备，通过 TCMU 等后端处理读写请求，并提供本地 cache、prefetch、commit 和 live snapshot 相关能力。[SRC-OVERLAYBD-001] [SRC-OVERLAYBD-002]

## 与文件级方案的根本差别

OverlayBD 后端看到 sector/range，而不是文件路径。guest 或 host 可以在这个块设备上使用 ext4 等普通读写文件系统，因此 writable upper 和磁盘 snapshot 更自然；代价是后端很难直接知道“哪个文件属于启动热路径”。

## 边界

它处理 rootfs/disk block，不恢复 guest RAM。块 cache 的共享也不等于 guest 内存页共享。

来源：[SRC-OVERLAYBD-001] [SRC-OVERLAYBD-002]
