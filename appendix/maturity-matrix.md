# 产品成熟度矩阵

> 阅读完成后，读者能够快速判断各产品的懒加载对象、触发和填充方式，以及该能力是正式发布、已合入、实验、未合入还是原型。

## 标签定义

| 标签 | 判定条件 |
| --- | --- |
| `released` | 能力出现在官方正式版本或稳定发布文档中 |
| `merged` | 实现已进入官方主分支，但尚未确认随正式版本发布 |
| `experimental` | 官方代码或文档明确标记为实验性 |
| `open-pr` | 实现只存在于仍开放的官方 PR/MR |
| `closed-unmerged` | PR/MR 已关闭且没有合入主分支 |
| `prototype` | 研究、个人或演示实现，未形成稳定产品能力 |
| `proposal` | 只有设计讨论，未找到可验证实现 |

## 产品矩阵

| 项目/能力 | 懒加载对象 | 触发入口 | 填充方式 | 状态 | 最后核对版本 | Source ID |
| --- | --- | --- | --- | --- | --- | --- |
| Nydus v2 RAFS | rootfs lower | FUSE/virtiofs/EROFS 请求 | chunk fetch + local blob cache | `released` | master `8aa80aee6e77` | [SRC-NYDUS-001] |
| Nydus RAFSv6 UFFD service | pmem rootfs block view | VMM 传入的 UFFD event | Copy 或 FD+MAP_FIXED | `merged` | master 包含 PR #1921 | [SRC-NYDUS-004] |
| Nydus v3 redesign branch | rootfs lower | FUSE/ublk/UFFD/fanotify | EROFS-native chunk cache | `experimental` | `9d769780aeb7` | [SRC-NYDUSV3-001] |
| stargz snapshotter | rootfs lower | FUSE/VFS file read | eStargz HTTP Range + cache | released | main `c2bf18e5a94d` | [SRC-STARGZ-001] |
| SOCI v2 | rootfs lower | FUSE/VFS file read | zTOC span fetch + cache | released | v0.15.0 | [SRC-SOCI-002] |
| OverlayBD | rootfs/块 lower + writable upper | block I/O | remote range + block cache | released | main `f63addfd8e51` | [SRC-OVERLAYBD-001] |
| EROFS + CacheFiles on-demand | rootfs lower | kernel EROFS/CacheFiles miss | daemon 写 cache file + ioctl complete | released | Linux 5.19-6.11；6.12 起废弃 | [SRC-EROFS-002] [SRC-EROFS-004] |
| containerd EROFS snapshotter | rootfs lower + overlay active upper | VFS/mount | 本地 EROFS blob/page cache | merged | main `84ae70638948` | [SRC-EROFS-003] |
| Firecracker snapshot MAP_PRIVATE | guest RAM | host file page fault | kernel file fault + COW | released | main `9cbb96f9b5b5` | [SRC-FC-001] |
| Firecracker external UFFD snapshot handler | guest RAM | UFFD missing event | handler UFFDIO_COPY | released | main `9cbb96f9b5b5` | [SRC-FC-002] |
| Firecracker lazy virtio-pmem #5740 | pmem rootfs | UFFD missing event | image service FD+MAP_FIXED | `proposal` | issue open | [SRC-FC-004] |
| Cloud Hypervisor on-demand restore | guest RAM | UFFD missing event | internal handler 从 memory-ranges 取页 | released | v53.0 | [SRC-CH-001] |
| Cloud Hypervisor pmem external UFFD #8239 | pmem rootfs/guest RAM | UFFD missing event | external handler Copy 或 FD+MAP_FIXED | closed-unmerged | head `934c76a` | [SRC-CH-002] |
| QEMU Fast Snapshot Load | guest RAM | postcopy/UFFD fault | mapped-ram offset read + background load | merged | master `99e54ab5e7a6` | [SRC-QEMU-001] |
| CRIU lazy-pages | Linux process RAM | UFFD missing event | lazy-pages daemon 注页 + background fill | released | criu-dev `c5ba2abb9731` | [SRC-CRIU-001] |
| Kata + Nydus remote snapshot | container workload rootfs | guest/host file read | FUSE/virtiofs/EROFS backend | released | main `84a479111f1a` | [SRC-KATA-002] |
| E2B lazy memory | guest RAM | Firecracker UFFD fault | template memfile page supply + prefetch | merged | main `baab2207ee67` | [SRC-E2B-001] |
| E2B COW rootfs | writable disk | NBD block I/O | read-only template + per-sandbox COW cache | merged | main `baab2207ee67` | [SRC-E2B-001] |
| CubeSandbox EROFS proposal | rootfs lower | 计划中的 pmem/DAX | 未实现 | closed-unmerged | issue #274 | [SRC-CUBE-001] |
| urunc shared snapshot view | unikernel/rootfs artifact | shim 准备 view | devmapper read-only view + lease | prototype | discussion #523 | [SRC-URUNC-001] |

矩阵只做索引，不代替产品正文。一个项目包含多条成熟度不同的路径时必须拆成多行，例如 Firecracker 内存快照和 virtio-pmem 提案不能合并成一项。
