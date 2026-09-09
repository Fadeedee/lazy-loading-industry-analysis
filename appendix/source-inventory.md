# 第一方来源清单

> 阅读完成后，读者能够从正文的 Source ID 回到具体官方仓库、文档、PR、issue 或论文，并确认结论对应的版本和成熟度。

## 记录规则

### 定向筛查补充：2026-09-09

以下仅为固定 commit 的源码静态核查，未运行测试，不推定对应能力已发布。详细范围和实验影响见[筛查记录](targeted-screening-2026-09-09.md)。其余来源的版本与核查日期不变。

| Source ID | 第一方源码 | 核查范围 | 状态 |
| --- | --- | --- | --- |
| [SRC-DF-001] | [Dragonfly PieceNotifier](https://github.com/dragonflyoss/client/blob/d5bccb022e805944080334236eb61d3ac5103bee/dragonfly-client-storage/src/piece_notifier.rs) | owner 领取、共享通知、终态清理及文件内测试 | merged；发布版本未核查 |
| [SRC-DF-002] | [Dragonfly storage](https://github.com/dragonflyoss/client/blob/d5bccb022e805944080334236eb61d3ac5103bee/dragonfly-client-storage/src/lib.rs) | wait_for_piece_finished、download_piece_failed 的通知与超时路径 | merged；完整取消链未核查 |
| [SRC-JFS-001] | [JuiceFS disk_cache.go](https://github.com/juicedata/juicefs/blob/5f250030ec437d6c676355e7a99f92b8ec480747/pkg/chunk/disk_cache.go) | flushPage、写入/关闭/rename helper、cleanupFull | merged；发布版本未核查 |
| [SRC-JFS-002] | [JuiceFS singleflight.go](https://github.com/juicedata/juicefs/blob/5f250030ec437d6c676355e7a99f92b8ec480747/pkg/chunk/singleflight.go) | Controller.Execute 的共享等待与 Page 引用 | merged；未运行测试 |
| [SRC-JFS-003] | [JuiceFS cached_store.go](https://github.com/juicedata/juicefs/blob/5f250030ec437d6c676355e7a99f92b8ec480747/pkg/chunk/cached_store.go) | rSlice.ReadAt 的 group.Execute 调用与闭包 context | merged；非完整读取链审计 |

### 通用规则与既有核查

**来源分类：** lazyd 样本、toolkit/lazyd 中的 bitmap 工作及旧 StratoVirt 实验属于自有实现与实验材料；Nydus、SOCI 等属于外部参考。各来源按所记录的版本和核查范围使用。

每个来源使用唯一 ID：`[SRC-项目-编号]`。同一个网页或源码 commit 只登记一次；正文可以重复引用该 ID。

2026-09-08 重写三仓联合方案时，重新核对了 Conch/StratoVirt 的远端 dev tip、关键源码和 #155/#2017 的状态。业界产品沿用已登记的固定版本研究，其中同日的定向复核记录仍有效，但不表示本次重写又重审了全部产品。GitCode 状态通过官方 API 核对，代码通过 Git 对象读取；本地证据位置只辅助定位，远端读者应使用公开链接与 commit。本次没有运行产品测试或新的端到端/性能实验。

2026-09-09 补充远端工作集建议时，仅复核固定版本 eStargz 文档的 workload-based optimization 章节，确认转换阶段采样及运行前预取的描述。跨实例远端聚合是本项目待验证建议，不是该来源已实现能力；其他来源与三仓基线日期不变。

同日完善 lazyd 设计时，静态核对本地与 `751d647fb37f` 一致的 Instance/register、ensure_range、inflight、FD 导出、socket 数据面和 OCI Range 实现，记录现状及待验证风险；未修改或运行产品代码。Conch/StratoVirt 上游状态未刷新；Nydus/E2B 证据沿用原核查，不把本次建议编辑算作重新审查全部来源。

随后系统整理 13 个产品借鉴章节时，额外读取固定版本 Nydus `state/mod.rs`、`blob_state_map.rs` 和 E2B `rootfs/nbd.go`，分别补足 pending/ready 协调及 provider 就绪/关闭的源码证据。其余产品沿用原研究并明确证据缺口；这不是全产品最新版本审计，也没有新增运行或性能测试。

| Source ID | 项目 | 第一方资料 | URL | 本地证据 | commit/tag | 访问日期 | 成熟度 | 支撑结论 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| [SRC-META-001] | 本调研 | 来源记录规则 | 本文件 | 不适用 | 不适用 | 2026-09-04 | released | 定义来源字段和成熟度用法 |
| [SRC-NYDUS-001] | Nydus v2 | Nydus design | https://github.com/dragonflyoss/nydus/blob/8aa80aee6e77a0c4d529581fc6339e9ff3066736/docs/nydus-design.md | 本次重新下载固定版本 | `8aa80aee6e77` | 2026-09-08 | released | bootstrap/blob、chunk 定位与可选校验、文件/目录 prefetch hints |
| [SRC-NYDUS-002] | Nydus v2 | Nydus image formats | https://github.com/dragonflyoss/nydus/blob/master/docs/nydus-image.md | `/root/nydus/docs/nydus-image.md` | master `8aa80aee6e77` | 2026-09-04 | released | Native/Zran/Tarfs 的格式、缓存和 lazy-loading 能力 |
| [SRC-NYDUS-003] | Nydus v2 | EROFS fscache user guide | https://github.com/dragonflyoss/nydus/blob/master/docs/nydus-fscache.md | `/root/nydus/docs/nydus-fscache.md` | master `8aa80aee6e77` | 2026-09-04 | released | EROFS+fscache 用户态 daemon 与内核配置路径 |
| [SRC-NYDUS-004] | Nydus UFFD service | PR #1921 | https://github.com/dragonflyoss/nydus/pull/1921 | 官方 API `merged=true` | merged 2026-05-22；head `3adf554b0985` | 2026-09-08 | merged | 服务端已合入；PR 作者报告测试不等于本次重跑 |
| [SRC-NYDUS-005] | Nydus UFFD service | Fault/handshake/prefault | https://github.com/dragonflyoss/nydus/blob/8aa80aee6e77a0c4d529581fc6339e9ff3066736/service/src/block_uffd.rs | 固定版本源码 | `8aa80aee6e77` | 2026-09-08 | merged | 持有 UFFD FD、直接读事件、Copy/zero、异步 prefault、部分错误仅 warning |
| [SRC-NYDUS-006] | Nydus UFFD protocol | VmaRegion / HandshakeRequest | https://github.com/dragonflyoss/nydus/blob/8aa80aee6e77a0c4d529581fc6339e9ff3066736/service/src/uffd_proto.rs | 固定版本源码 | `8aa80aee6e77` | 2026-09-08 | merged | region 默认 READ/PRIVATE/FIXED；定义 Handshake/Stat/PageFault 等交互 |
| [SRC-NYDUS-007] | Nydus block device | fetch_ranges / probe_blob_ranges | https://github.com/dragonflyoss/nydus/blob/8aa80aee6e77a0c4d529581fc6339e9ff3066736/service/src/block_device.rs | 固定版本源码 | `8aa80aee6e77` | 2026-09-08 | merged | flattened block 到 metadata/data/hole；probe_only 枚举 ready，不拉新数据 |
| [SRC-NYDUS-008] | Nydus storage | validate_chunk_data | https://github.com/dragonflyoss/nydus/blob/8aa80aee6e77a0c4d529581fc6339e9ff3066736/storage/src/cache/mod.rs | 固定版本源码 | `8aa80aee6e77` | 2026-09-08 | merged | 校验受配置、CRC/强制参数和 legacy stargz 条件约束；CRC 不等于密码学 hash |
| [SRC-NYDUS-009] | Nydus cache state | ChunkMap / RangeMap | https://github.com/dragonflyoss/nydus/blob/8aa80aee6e77a0c4d529581fc6339e9ff3066736/storage/src/cache/state/mod.rs | GitHub raw 固定版本源码 | `8aa80aee6e77` | 2026-09-09 | merged | ready/pending、领取、完成/清理和范围等待接口；不单凭接口证明持久化安全 |
| [SRC-NYDUS-010] | Nydus cache state | BlobStateMap / Slot | https://github.com/dragonflyoss/nydus/blob/8aa80aee6e77a0c4d529581fc6339e9ff3066736/storage/src/cache/state/blob_state_map.rs | GitHub raw 固定版本源码 | `8aa80aee6e77` | 2026-09-09 | merged | 在途表、条件变量限时等待、ready 复查与 pending 清理通知；未证明调用方取消/租约完整性 |
| [SRC-NYDUSV3-001] | Nydus v3 | 实验分支 README | https://github.com/dragonflyoss/nydus/blob/copilot/nydus-v3-chunk-digest-optimization/README.md | `/root/virtiolazyd/nydus-v3-chunk-digest-optimization/README.md` | `9d769780aeb7` | 2026-09-04 | experimental | EROFS-native v3、chunk/compress 分离、trace prefetch、UFFD/fanotify/ublk 目标 |
| [SRC-NYDUS-SNAPSHOTTER-001] | nydus-snapshotter | README | https://github.com/containerd/nydus-snapshotter | 无本地副本 | main `aab11e826dd7` | 2026-09-04 | released | containerd remote snapshotter、FUSE/virtiofs/in-kernel EROFS 接入 |
| [SRC-STARGZ-001] | stargz snapshotter | README | https://github.com/containerd/stargz-snapshotter | 无本地副本 | main `c2bf18e5a94d` | 2026-09-04 | released | eStargz remote snapshot、按需 chunk 拉取、启动与运行期开销 |
| [SRC-STARGZ-002] | eStargz | Format and prefetch | https://github.com/containerd/stargz-snapshotter/blob/c2bf18e5a94dcfd959cabf744f4bbb4ef8d980a2/docs/estargz.md | 固定版本文档；09-09 定向复核工作负载优化章节 | `c2bf18e5a94d` | 2026-09-09 | released | TOC、chunkDigest、landmark；转换时采样文件访问并排序，运行前预取 prioritized range；非跨用户远端聚合证据 |
| [SRC-STARGZ-003] | stargz snapshotter | Architecture overview | https://github.com/containerd/stargz-snapshotter/blob/main/docs/overview.md | 无本地副本 | main `c2bf18e5a94d` | 2026-09-04 | released | containerd remote snapshotter 的 Prepare/mount 生命周期 |
| [SRC-SOCI-001] | SOCI | README | https://github.com/awslabs/soci-snapshotter/blob/238af848f32fcb887072c144b09ee65a3a895f9c/README.md | 固定版本文档 | `238af848f32f` | 2026-09-08 | released | SOCI index、lazy fetch；v1 外置索引与 v2 构建期转换需区分 |
| [SRC-SOCI-002] | SOCI | CLI usage | https://github.com/awslabs/soci-snapshotter/blob/main/docs/cli-usage.md | 无本地副本 | main `238af848f32f` | 2026-09-04 | released | zTOC、4 MiB 默认 span、10 MiB 最小层、v2 强绑定与 prefetch |
| [SRC-SOCI-003] | SOCI | Release v0.15.0 | https://github.com/awslabs/soci-snapshotter/releases/tag/v0.15.0 | 无本地副本 | v0.15.0 `7716bd6` | 2026-09-04 | released | 并行 metadata、registry auth、观测和 FD 泄漏修复 |
| [SRC-SOCI-004] | SOCI | zTOC/span glossary | https://github.com/awslabs/soci-snapshotter/blob/238af848f32fcb887072c144b09ee65a3a895f9c/docs/glossary.md | 固定版本文档 | `238af848f32f` | 2026-09-08 | released | 文件 TAR offset、压缩流 checkpoint、可独立解压 span |
| [SRC-OVERLAYBD-001] | OverlayBD | README | https://github.com/containerd/overlaybd/blob/f63addfd8e51bd9eeecc12f5f4670c281a74d6e8/README.md | 固定版本文档 | `f63addfd8e51` | 2026-09-08 | released | TCMU、只读 lowers、唯一 writable upper、commit 与 live snapshot 切换 |
| [SRC-OVERLAYBD-002] | OverlayBD | Project documentation | https://github.com/containerd/overlaybd/blob/main/docs/README.md | 无本地副本 | main `f63addfd8e51` | 2026-09-04 | released | on-demand fetch、prefetch、VM/微沙箱适用性与项目状态 |
| [SRC-EROFS-001] | Linux EROFS | Kernel documentation | https://docs.kernel.org/filesystems/erofs.html | Linux 官方文档 | current docs | 2026-09-04 | released | 只读镜像文件系统、块对齐、写入重定向到其他文件系统 |
| [SRC-EROFS-002] | CacheFiles | Kernel on-demand read | https://docs.kernel.org/filesystems/caching/cachefiles.html | Linux 官方文档 | current docs | 2026-09-04 | released | `/dev/cachefiles` 请求、READ range、anonymous fd 与 READ_COMPLETE ioctl |
| [SRC-EROFS-003] | containerd EROFS | Native snapshotter guide | https://github.com/containerd/containerd/blob/main/docs/snapshotters/erofs.md | 无本地副本 | main `84ae70638948` | 2026-09-04 | merged | 原生 EROFS layer、file-backed mount、FSDAX、OverlayFS active snapshot |
| [SRC-EROFS-004] | EROFS | Project documentation PDF | https://erofs.docs.kernel.org/_/downloads/en/latest/pdf/ | EROFS 官方文档 | 2026-09 文档 | 2026-09-04 | released | fscache on-demand 自 Linux 6.12 起废弃，file-backed mount 替代方向 |
| [SRC-FC-001] | Firecracker | Snapshot support | https://github.com/firecracker-microvm/firecracker/blob/main/docs/snapshotting/snapshot-support.md | `/root/virtiolazyd/firecracker/docs/snapshotting/snapshot-support.md` | main `9cbb96f9b5b5`；本地 `14108ca14ef1` | 2026-09-04 | released | memory file MAP_PRIVATE 按需加载、COW、磁盘由用户管理、diff snapshot 状态 |
| [SRC-FC-002] | Firecracker | Handling page faults | https://github.com/firecracker-microvm/firecracker/blob/7699746649826d1dfcdde626b3131bac08f28e0d/docs/snapshotting/handling-page-faults-on-snapshot-resume.md | 固定版本文档 | `769974664982` | 2026-09-08 | released | external UFFD FD/layout、COPY 示例、REMOVE、handler 崩溃可能挂起及监控回收责任 |
| [SRC-FC-003] | Firecracker | virtio-pmem guide | https://github.com/firecracker-microvm/firecracker/blob/main/docs/pmem.md | `/root/virtiolazyd/firecracker/docs/pmem.md` | main `9cbb96f9b5b5` | 2026-09-04 | released | file-backed pmem、read_only、设备顺序、跨 VM 共享安全警告 |
| [SRC-FC-004] | Firecracker | Issue #5740 | https://github.com/firecracker-microvm/firecracker/issues/5740 | 官方 issue 页面 | open；创建 2026-03-07 | 2026-09-08 | proposal | PROBE/FETCH、FD/remap 提案；作者报告原型，未证明上游实现 |
| [SRC-CH-001] | Cloud Hypervisor | Release v53.0 | https://github.com/cloud-hypervisor/cloud-hypervisor/releases/tag/v53.0 | 官方发布说明；本地 v51 不作 v53 证据 | v53.0 `9ed824d` | 2026-09-08 | released | UFFD demand paging、offload daemon、后台 prefault；不等于 #8239 pmem 协议 |
| [SRC-CH-002] | Cloud Hypervisor | PR #8239 | https://github.com/cloud-hypervisor/cloud-hypervisor/pull/8239 | 官方 API `merged=false` | head `934c76a02842`，closed 2026-06-08 | 2026-09-08 | closed-unmerged | pmem external UFFD 未合入；不应作为 upstream 依赖 |
| [SRC-QEMU-001] | QEMU | Fast Snapshot Load | https://github.com/qemu/qemu/blob/35500e5c41aec76cde59befe750600dac7a9e37a/docs/devel/migration/fast-snapshot-load.rst | 固定版本文档及官方 HTML | `35500e5c41ae`；HTML 11.1.50 | 2026-09-08 | merged | 本地 mapped-ram、pending_bmap 协调；eager thread 用于最终结束恢复，不是 RTT 自适应预取 |
| [SRC-CRIU-001] | CRIU | CLI reference | https://github.com/checkpoint-restore/criu/blob/criu-dev/Documentation/criu.txt | 无本地副本 | criu-dev `c5ba2abb9731` | 2026-09-04 | released | `--lazy-pages`、lazy-pages daemon、page server、按需注页与后台填充 |
| [SRC-KATA-001] | Kata Containers | Guest assets architecture | https://github.com/kata-containers/kata-containers/blob/main/docs/design/architecture/guest-assets.md | 无本地副本 | main `84a479111f1a` | 2026-09-04 | released | guest kernel、initrd/rootfs image 与 workload image 的区别 |
| [SRC-KATA-002] | Kata Containers | Guest image management | https://github.com/kata-containers/kata-containers/blob/main/docs/design/kata-guest-image-management-design.md | 无本地副本 | main `84a479111f1a` | 2026-09-04 | released | remote snapshotter、guest pull、Nydus FUSE/virtiofs/EROFS 边界 |
| [SRC-KATA-003] | Kata Containers | Kata Nydus design | https://github.com/kata-containers/kata-containers/blob/26c2e1630457977d931bce0527c4df92a3b8f15d/docs/design/kata-nydus-design.md | 固定版本设计文档 | `26c2e1630457` | 2026-09-08 | merged | 文档中的 mount extraoption、shim 解析与 guest overlay；不证明所有 runtime 的当前路径 |
| [SRC-KATA-004] | Kata Containers | Virtualization architecture | https://github.com/kata-containers/kata-containers/blob/main/docs/design/virtualization.md | 无本地副本 | main `84a479111f1a` | 2026-09-04 | released | QEMU/CH/Firecracker/Dragonball 角色与 Dragonball 内嵌 VMM 边界 |
| [SRC-E2B-001] | E2B Infra | Architecture | https://github.com/e2b-dev/infra/blob/cc7c574233ad98665a7c72a3d37b0af89ae79a71/docs/ARCHITECTURE.md | 固定版本文档 | `cc7c574233ad` | 2026-09-08 | merged | 内存 UFFD 与 host NBD/COW 双路、snapshot artifacts、编排职责 |
| [SRC-E2B-002] | E2B block overlay | ReadAt / WriteAt / SwapCache | https://github.com/e2b-dev/infra/blob/cc7c574233ad98665a7c72a3d37b0af89ae79a71/packages/orchestrator/pkg/sandbox/block/overlay.go | 固定版本源码 | `cc7c574233ad` | 2026-09-08 | merged | writable、可选 sealing、base 读取优先级；新写入进入私有 cache |
| [SRC-E2B-003] | E2B memory handler | Userfaultfd.faultPage | https://github.com/e2b-dev/infra/blob/cc7c574233ad98665a7c72a3d37b0af89ae79a71/packages/orchestrator/pkg/sandbox/uffd/userfaultfd/userfaultfd.go | 固定版本源码 | `cc7c574233ad` | 2026-09-08 | merged | PageReader、COPY、有限重试、onFailure、WP/REMOVE 状态；调用方终止策略未完整追踪 |
| [SRC-E2B-004] | E2B NBD rootfs | NewNBDProvider / Start / Close | https://github.com/e2b-dev/infra/blob/cc7c574233ad98665a7c72a3d37b0af89ae79a71/packages/orchestrator/pkg/sandbox/rootfs/nbd.go | 09-09 从 GitHub raw 定向复核固定源码 | `cc7c574233ad` | 2026-09-09 | merged | cache/overlay、ready 结果；flush/mount.Close/通知/overlay.Close 顺序、摘出 cache 所有权；未运行测试或验证全部关闭调用方 |
| [SRC-CUBE-001] | CubeSandbox | EROFS feature request #274 | https://github.com/TencentCloud/CubeSandbox/issues/274 | 无本地副本 | closed-not-planned 2026 | 2026-09-04 | closed-unmerged | ext4 现状及 EROFS/pmem 建议未被项目接受 |
| [SRC-CUBE-002] | CubeSandbox | Rootfs implementation | https://github.com/TencentCloud/CubeSandbox/blob/master/Cubelet/pkg/container/rootfs/rootfs.go | 无本地副本 | master `3b063e75c54b` | 2026-09-04 | released | host lowerdir 通过 virtiofs 共享及 container rootfs 构造 |
| [SRC-CUBE-003] | CubeSandbox | XFS reflink FAQ #311 | https://github.com/TencentCloud/CubeSandbox/issues/311 | 无本地副本 | master discussion | 2026-09-04 | released | ext4 image 的 XFS reflink clone、每 sandbox 可写层和存储约束 |
| [SRC-CUBE-004] | CubeSandbox | Cross-node snapshot #1197 | https://github.com/TencentCloud/CubeSandbox/issues/1197 | 无本地副本 | open 2026 | 2026-09-04 | proposal | memory/disk/config 远端同步与跨节点恢复需求 |
| [SRC-OPENSANDBOX-001] | OpenSandbox | Sandbox lifecycle spec | https://github.com/opensandbox-group/OpenSandbox/blob/main/specs/sandbox-lifecycle.yml | 无本地副本 | main `fa2568546f8d` | 2026-09-04 | merged | snapshot 资源状态与 restore-snapshot API；未证明底层懒恢复 |
| [SRC-URUNC-001] | urunc | Rootfs snapshot view discussion #523 | https://github.com/urunc-dev/urunc/discussions/523 | 无本地副本 | main `58a25a23fdde`，prototype discussion | 2026-09-04 | prototype | shared read-only snapshot view、lease、fallback 和小镜像性能反例 |
| [SRC-CONCH-001] | Conch | upstream dev source | https://gitcode.com/openeuler/Conch/commit/8248022005542407f53b606a8be979379a3fd38b | 隔离 clone `/tmp/lazy-redesign-conch` 的 `FETCH_HEAD` | `8248022005542407f53b606a8be979379a3fd38b` | 2026-09-08 | merged | Boot Index/Store/BootPreparer；Fetch 尚无按需 payload 筛选；本地 PmemPaths 与 mapped restore；guestd 和 daemon 生命周期 |
| [SRC-CONCH-002] | Conch | PR #184 sandbox containerd store | https://gitcode.com/openeuler/Conch/pull/184 | 官方 API + `ae1e29ad8f07` 的 store.go（远端内容核对） | merged 2026-09-03 | 2026-09-08 | merged | Sandbox extension、Boot Index/runtime snapshot GC references；不自动管理 lazyd cache lease |
| [SRC-CONCH-003] | Conch | PR #155 incremental memory checkpoints | https://gitcode.com/openeuler/Conch/pull/155 | 官方 API + 本地同 head 的 internal/cow/uffd.go | head `1ca332dca323`，open | 2026-09-08 | open-pr | PinnedManifest.ReadPage、memfd.WriteAt、UFFDIO_WAKE；本次刷新状态/head，未重新完整 review；不是已合入前置 |
| [SRC-SV-001] | StratoVirt | upstream dev source | https://gitcode.com/openeuler/stratovirt/commit/0f948b653b1f695ad9a527059e42baff77e07a3f | 本地同 commit 源码，远端 dev tip 重新核对 | `0f948b653b1f695ad9a527059e42baff77e07a3f` | 2026-09-08 | merged | file-backed pmem；外部 RAM UFFD MISSING、可选 WP/dirty/resident 跟踪；FD/JSON 发送不含来源 ready ACK；非现成 pmem remap 协议 |
| [SRC-SV-002] | StratoVirt | PR #2017 inherited memfd | https://gitcode.com/openeuler/stratovirt/pull/2017 | 官方 API + 本地 origin/pr-2017 同 head | head `d70c76b78f9e`，open | 2026-09-08 | open-pr | inherited memfd backend；本轮仅核对相关路径，不是完整 PR 重审 |
| [SRC-SV-003] | StratoVirt | 旧 lazy pmem feature branch | https://gitcode.com/lsxlee99/stratovirt/commits/lazy-pmem-full | `/root/virtiolazyd/stratovirt` 的 `lazy-pmem-full` | 历史核查 `50f80dd3`（不是当前开发基线） | 2026-09-04 | prototype | lazy config、anonymous HVA、UFFD、FETCH/SCM_RIGHTS、fixed remap、padding、fatal shutdown 的算法与测试证据 |
| [SRC-LAZYD-001] | lazyd | lazy UFFD range feature branch | https://github.com/Fadeedee/lazyd/tree/751d647fb37fa2e01f6fb1784003e8b26aa00244 | `/root/virtiolazyd/lazyd`，同 commit 静态复核 | `751d647fb37f` | 2026-09-09 | prototype | bitmap/descriptor/FETCH、sync 顺序；Instance 重开、5ms 去重轮询、阻塞数据面 I/O、读写 FD 克隆及 Range 检查边界；未重跑测试 |

## 未纳入候选

只有具备公开源码、可定位数据路径且可确认维护状态的项目才进入产品正文。未纳入候选在此记录项目、候选来源和缺失证据，避免后续重复调查。

| 候选 | 结果 | 原因 |
| --- | --- | --- |
| Dragonball 独立仓库 | 合并到 Kata 分析 | 当前官方文档将 Dragonball 作为 Kata runtime-rs 内嵌 VMM，单独仓库搜索未提供额外懒加载证据 |
| OpenSandbox | 仅作 API 对照 | 有 snapshot lifecycle API，但没有确认底层 rootfs/guest RAM 懒恢复实现，不作为机制样本 |
| CubeSandbox EROFS | 仅作反例与生命周期案例 | EROFS 提案已关闭未合入，当前实现仍以 ext4/virtiofs/reflink 为主 |
