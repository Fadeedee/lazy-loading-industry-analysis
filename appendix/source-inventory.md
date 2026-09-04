# 第一方来源清单

> 阅读完成后，读者能够从正文的 Source ID 回到具体官方仓库、文档、PR、issue 或论文，并确认结论对应的版本和成熟度。

## 记录规则

每个来源使用唯一 ID：`[SRC-项目-编号]`。同一个网页或源码 commit 只登记一次；正文可以重复引用该 ID。

| Source ID | 项目 | 第一方资料 | URL | 本地证据 | commit/tag | 访问日期 | 成熟度 | 支撑结论 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| [SRC-META-001] | 本调研 | 来源记录规则 | 本文件 | 不适用 | 不适用 | 2026-09-04 | released | 定义来源字段和成熟度用法 |
| [SRC-NYDUS-001] | Nydus v2 | Nydus design | https://github.com/dragonflyoss/nydus/blob/master/docs/nydus-design.md | `/root/nydus/docs/nydus-design.md` | master `8aa80aee6e77`；本地证据 `00d0a520cbf4` | 2026-09-04 | released | RAFS v5/v6、bootstrap/blob、FUSE/virtiofs、chunk 按需加载 |
| [SRC-NYDUS-002] | Nydus v2 | Nydus image formats | https://github.com/dragonflyoss/nydus/blob/master/docs/nydus-image.md | `/root/nydus/docs/nydus-image.md` | master `8aa80aee6e77` | 2026-09-04 | released | Native/Zran/Tarfs 的格式、缓存和 lazy-loading 能力 |
| [SRC-NYDUS-003] | Nydus v2 | EROFS fscache user guide | https://github.com/dragonflyoss/nydus/blob/master/docs/nydus-fscache.md | `/root/nydus/docs/nydus-fscache.md` | master `8aa80aee6e77` | 2026-09-04 | released | EROFS+fscache 用户态 daemon 与内核配置路径 |
| [SRC-NYDUS-004] | Nydus UFFD service | PR #1921 | https://github.com/dragonflyoss/nydus/pull/1921 | `/root/nydus/service/src/block_uffd.rs` | merged 2026-05-22；本地 `00d0a520cbf4` | 2026-09-04 | merged | RAFSv6 UFFD block service、Copy/Zerocopy、SCM_RIGHTS、pre-fault |
| [SRC-NYDUSV3-001] | Nydus v3 | 实验分支 README | https://github.com/dragonflyoss/nydus/blob/copilot/nydus-v3-chunk-digest-optimization/README.md | `/root/virtiolazyd/nydus-v3-chunk-digest-optimization/README.md` | `9d769780aeb7` | 2026-09-04 | experimental | EROFS-native v3、chunk/compress 分离、trace prefetch、UFFD/fanotify/ublk 目标 |
| [SRC-NYDUS-SNAPSHOTTER-001] | nydus-snapshotter | README | https://github.com/containerd/nydus-snapshotter | 无本地副本 | main `aab11e826dd7` | 2026-09-04 | released | containerd remote snapshotter、FUSE/virtiofs/in-kernel EROFS 接入 |
| [SRC-STARGZ-001] | stargz snapshotter | README | https://github.com/containerd/stargz-snapshotter | 无本地副本 | main `c2bf18e5a94d` | 2026-09-04 | released | eStargz remote snapshot、按需 chunk 拉取、启动与运行期开销 |
| [SRC-STARGZ-002] | eStargz | Format and prefetch | https://github.com/containerd/stargz-snapshotter/blob/main/docs/estargz.md | 无本地副本 | main `c2bf18e5a94d` | 2026-09-04 | released | TOC、chunkDigest、prioritized files、landmark prefetch |
| [SRC-STARGZ-003] | stargz snapshotter | Architecture overview | https://github.com/containerd/stargz-snapshotter/blob/main/docs/overview.md | 无本地副本 | main `c2bf18e5a94d` | 2026-09-04 | released | containerd remote snapshotter 的 Prepare/mount 生命周期 |
| [SRC-SOCI-001] | SOCI | README | https://github.com/awslabs/soci-snapshotter | 无本地副本 | main `238af848f32f` | 2026-09-04 | released | SOCI index、原 OCI 层、lazy fetch 和 v1/v2 取舍 |
| [SRC-SOCI-002] | SOCI | CLI usage | https://github.com/awslabs/soci-snapshotter/blob/main/docs/cli-usage.md | 无本地副本 | main `238af848f32f` | 2026-09-04 | released | zTOC、4 MiB 默认 span、10 MiB 最小层、v2 强绑定与 prefetch |
| [SRC-SOCI-003] | SOCI | Release v0.15.0 | https://github.com/awslabs/soci-snapshotter/releases/tag/v0.15.0 | 无本地副本 | v0.15.0 `7716bd6` | 2026-09-04 | released | 并行 metadata、registry auth、观测和 FD 泄漏修复 |
| [SRC-OVERLAYBD-001] | OverlayBD | README | https://github.com/containerd/overlaybd | 无本地副本 | main `f63addfd8e51` | 2026-09-04 | released | 远程块镜像、TCMU、lower/upper、commit 与 live snapshot |
| [SRC-OVERLAYBD-002] | OverlayBD | Project documentation | https://github.com/containerd/overlaybd/blob/main/docs/README.md | 无本地副本 | main `f63addfd8e51` | 2026-09-04 | released | on-demand fetch、prefetch、VM/微沙箱适用性与项目状态 |
| [SRC-EROFS-001] | Linux EROFS | Kernel documentation | https://docs.kernel.org/filesystems/erofs.html | Linux 官方文档 | current docs | 2026-09-04 | released | 只读镜像文件系统、块对齐、写入重定向到其他文件系统 |
| [SRC-EROFS-002] | CacheFiles | Kernel on-demand read | https://docs.kernel.org/filesystems/caching/cachefiles.html | Linux 官方文档 | current docs | 2026-09-04 | released | `/dev/cachefiles` 请求、READ range、anonymous fd 与 READ_COMPLETE ioctl |
| [SRC-EROFS-003] | containerd EROFS | Native snapshotter guide | https://github.com/containerd/containerd/blob/main/docs/snapshotters/erofs.md | 无本地副本 | main `84ae70638948` | 2026-09-04 | merged | 原生 EROFS layer、file-backed mount、FSDAX、OverlayFS active snapshot |
| [SRC-EROFS-004] | EROFS | Project documentation PDF | https://erofs.docs.kernel.org/_/downloads/en/latest/pdf/ | EROFS 官方文档 | 2026-09 文档 | 2026-09-04 | released | fscache on-demand 自 Linux 6.12 起废弃，file-backed mount 替代方向 |
| [SRC-FC-001] | Firecracker | Snapshot support | https://github.com/firecracker-microvm/firecracker/blob/main/docs/snapshotting/snapshot-support.md | `/root/virtiolazyd/firecracker/docs/snapshotting/snapshot-support.md` | main `9cbb96f9b5b5`；本地 `14108ca14ef1` | 2026-09-04 | released | memory file MAP_PRIVATE 按需加载、COW、磁盘由用户管理、diff snapshot 状态 |
| [SRC-FC-002] | Firecracker | Handling page faults | https://github.com/firecracker-microvm/firecracker/blob/main/docs/snapshotting/handling-page-faults-on-snapshot-resume.md | `/root/virtiolazyd/firecracker/docs/snapshotting/handling-page-faults-on-snapshot-resume.md` | main `9cbb96f9b5b5` | 2026-09-04 | released | external UFFD handler、UDS 传 FD/layout、UFFDIO_COPY |
| [SRC-FC-003] | Firecracker | virtio-pmem guide | https://github.com/firecracker-microvm/firecracker/blob/main/docs/pmem.md | `/root/virtiolazyd/firecracker/docs/pmem.md` | main `9cbb96f9b5b5` | 2026-09-04 | released | file-backed pmem、read_only、设备顺序、跨 VM 共享安全警告 |
| [SRC-FC-004] | Firecracker | Issue #5740 | https://github.com/firecracker-microvm/firecracker/issues/5740 | 无产品代码 | open 2026-03-09 | 2026-09-04 | proposal | virtio-pmem UFFD、PROBE/FETCH、SCM_RIGHTS、MAP_FIXED 设计提案 |
| [SRC-CH-001] | Cloud Hypervisor | Release v53.0 | https://github.com/cloud-hypervisor/cloud-hypervisor/releases/tag/v53.0 | `/root/cloud/cloud-hypervisor` 为 v51 系分支，不作为 v53 源码证据 | v53.0 `9ed824d` | 2026-09-04 | released | UFFD demand-paged restore、snapshot/restore daemon、sparse memory improvements |
| [SRC-CH-002] | Cloud Hypervisor | PR #8239 | https://github.com/cloud-hypervisor/cloud-hypervisor/pull/8239 | 无 upstream 合入代码 | head `934c76a`，closed 2026-06-08 | 2026-09-04 | closed-unmerged | pmem external UFFD、SCM_RIGHTS、MAP_FIXED、Copy fallback 及 reviewer 架构意见 |
| [SRC-QEMU-001] | QEMU | Fast Snapshot Load | https://www.qemu.org/docs/master/devel/migration/fast-snapshot-load.html | QEMU 官方文档 | master `99e54ab5e7a6` | 2026-09-04 | merged | mapped-ram、postcopy infrastructure、UFFD fault、background load 与 pending bitmap |
| [SRC-CRIU-001] | CRIU | CLI reference | https://github.com/checkpoint-restore/criu/blob/criu-dev/Documentation/criu.txt | 无本地副本 | criu-dev `c5ba2abb9731` | 2026-09-04 | released | `--lazy-pages`、lazy-pages daemon、page server、按需注页与后台填充 |
| [SRC-KATA-001] | Kata Containers | Guest assets architecture | https://github.com/kata-containers/kata-containers/blob/main/docs/design/architecture/guest-assets.md | 无本地副本 | main `84a479111f1a` | 2026-09-04 | released | guest kernel、initrd/rootfs image 与 workload image 的区别 |
| [SRC-KATA-002] | Kata Containers | Guest image management | https://github.com/kata-containers/kata-containers/blob/main/docs/design/kata-guest-image-management-design.md | 无本地副本 | main `84a479111f1a` | 2026-09-04 | released | remote snapshotter、guest pull、Nydus FUSE/virtiofs/EROFS 边界 |
| [SRC-KATA-003] | Kata Containers | Kata Nydus design | https://github.com/kata-containers/kata-containers/blob/main/docs/design/kata-nydus-design.md | 无本地副本 | main `84a479111f1a` | 2026-09-04 | merged | snapshotter mount metadata 传入 shim/guest、host upper 与 guest RAFS lower 组装 |
| [SRC-KATA-004] | Kata Containers | Virtualization architecture | https://github.com/kata-containers/kata-containers/blob/main/docs/design/virtualization.md | 无本地副本 | main `84a479111f1a` | 2026-09-04 | released | QEMU/CH/Firecracker/Dragonball 角色与 Dragonball 内嵌 VMM 边界 |
| [SRC-E2B-001] | E2B Infra | Architecture | https://github.com/e2b-dev/infra/blob/main/docs/ARCHITECTURE.md | 无本地副本 | main `baab2207ee67` | 2026-09-04 | merged | Firecracker UFFD memory、NBD+COW rootfs、diff snapshot、template cache 与预取 |
| [SRC-CUBE-001] | CubeSandbox | EROFS feature request #274 | https://github.com/TencentCloud/CubeSandbox/issues/274 | 无本地副本 | closed-not-planned 2026 | 2026-09-04 | closed-unmerged | ext4 现状及 EROFS/pmem 建议未被项目接受 |
| [SRC-CUBE-002] | CubeSandbox | Rootfs implementation | https://github.com/TencentCloud/CubeSandbox/blob/master/Cubelet/pkg/container/rootfs/rootfs.go | 无本地副本 | master `3b063e75c54b` | 2026-09-04 | released | host lowerdir 通过 virtiofs 共享及 container rootfs 构造 |
| [SRC-CUBE-003] | CubeSandbox | XFS reflink FAQ #311 | https://github.com/TencentCloud/CubeSandbox/issues/311 | 无本地副本 | master discussion | 2026-09-04 | released | ext4 image 的 XFS reflink clone、每 sandbox 可写层和存储约束 |
| [SRC-CUBE-004] | CubeSandbox | Cross-node snapshot #1197 | https://github.com/TencentCloud/CubeSandbox/issues/1197 | 无本地副本 | open 2026 | 2026-09-04 | proposal | memory/disk/config 远端同步与跨节点恢复需求 |
| [SRC-OPENSANDBOX-001] | OpenSandbox | Sandbox lifecycle spec | https://github.com/opensandbox-group/OpenSandbox/blob/main/specs/sandbox-lifecycle.yml | 无本地副本 | main `fa2568546f8d` | 2026-09-04 | merged | snapshot 资源状态与 restore-snapshot API；未证明底层懒恢复 |
| [SRC-URUNC-001] | urunc | Rootfs snapshot view discussion #523 | https://github.com/urunc-dev/urunc/discussions/523 | 无本地副本 | main `58a25a23fdde`，prototype discussion | 2026-09-04 | prototype | shared read-only snapshot view、lease、fallback 和小镜像性能反例 |
| [SRC-CONCH-001] | Conch | upstream dev source | https://gitcode.com/openeuler/Conch/commits/dev | `/root/virtiolazyd/Conch` 的 `upstream/dev` | `0b405ce0d4d2` | 2026-09-04 | merged | Boot Index、Template/containerd image store、Sandbox/containerd store、BootPreparer 和普通 StratoVirt 路径 |
| [SRC-CONCH-002] | Conch | PR #184 sandbox containerd store | https://gitcode.com/openeuler/Conch/pull/184 | `upstream/dev` 已包含 `ae1e29a` | merged into `dev` | 2026-09-04 | merged | Sandbox metadata owner、Boot Index/runtime snapshot GC references、删除旧 metadata owner |
| [SRC-CONCH-003] | Conch | PR #155 incremental memory checkpoints | https://gitcode.com/openeuler/Conch/pull/155 | `/root/virtiolazyd/Conch` 的 `upstream/pr-155` | head `1ca332dca323`，open | 2026-09-04 | open-pr | conch-cow、incremental-v1 memory layers、memfd+UFFDIO_WAKE、checkpoint lineage |
| [SRC-SV-001] | StratoVirt | upstream dev source | https://gitcode.com/openeuler/stratovirt/commits/dev | `/root/virtiolazyd/stratovirt` 的 `origin/dev` | `0f948b653b1f` | 2026-09-04 | merged | 普通 file-backed virtio-pmem、snapshot UFFD/WP、memslot 和 machine lifecycle |
| [SRC-SV-002] | StratoVirt | PR #2017 inherited memfd | https://gitcode.com/openeuler/stratovirt/pull/2017 | `/root/virtiolazyd/stratovirt` 的 `origin/pr-2017` | head `d70c76b78f9e`，open | 2026-09-04 | open-pr | `memory-backend-memfd,fd=`、FD 复制/seal/size 校验，服务增量内存恢复 |
| [SRC-SV-003] | StratoVirt | 旧 lazy pmem feature branch | https://gitcode.com/lsxlee99/stratovirt/commits/lazy-pmem-full | `/root/virtiolazyd/stratovirt` 的 `lazy-pmem-full` | `50f80dd3`；落后 `origin/dev` 131 commits | 2026-09-04 | prototype | lazy config、anonymous HVA、UFFD、FETCH/SCM_RIGHTS、fixed remap、padding、fatal shutdown 的算法与测试证据 |
| [SRC-LAZYD-001] | lazyd | lazy UFFD range feature branch | https://github.com/Fadeedee/lazyd/tree/feature/lazy-uffd-range-api | `/root/virtiolazyd/lazyd` | `751d647` | 2026-09-04 | prototype | bitmap/range、descriptor prepare、registry auth、seqpacket FETCH、SCM_RIGHTS 和持久 instance |

## 未纳入候选

只有具备公开源码、可定位数据路径且可确认维护状态的项目才进入产品正文。未纳入候选在此记录项目、候选来源和缺失证据，避免后续重复调查。

| 候选 | 结果 | 原因 |
| --- | --- | --- |
| Dragonball 独立仓库 | 合并到 Kata 分析 | 当前官方文档将 Dragonball 作为 Kata runtime-rs 内嵌 VMM，单独仓库搜索未提供额外懒加载证据 |
| OpenSandbox | 仅作 API 对照 | 有 snapshot lifecycle API，但没有确认底层 rootfs/guest RAM 懒恢复实现，不作为机制样本 |
| CubeSandbox EROFS | 仅作反例与生命周期案例 | EROFS 提案已关闭未合入，当前实现仍以 ext4/virtiofs/reflink 为主 |
