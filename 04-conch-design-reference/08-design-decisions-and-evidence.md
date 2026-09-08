# 设计决策与业界依据

> 阅读完成后，读者能够逐项说明我们准备怎样设计、参考项目具体做了什么、参考资料在哪里，以及哪些细节仍是本项目的待验证建议。

本文于 **2026-09-08 重新核查**以下八项决策涉及的第一方文档、关键源码和 PR 状态，版本见 [来源清单](../appendix/source-inventory.md)。这是证据核查，不是重新运行产品测试或端到端验证；下文“验收”均为后续执行条件，“我们的设计”表示目标，不表示已在当前上游实现。未涉及的产品章节仍保留原核查日期。

## 阅读索引

| 我们的设计问题 | 主要参考 | 本文章节 |
| --- | --- | --- |
| 谁解析镜像、谁管理运行资源 | Conch 原生 stores、Kata | 1 |
| 如何识别和复用只读内容 | Nydus、SOCI | 2 |
| 缺页后如何共享文件页 | Nydus UFFD service | 3 |
| 哪些范围真正 ready | 当前 lazyd、Nydus chunk 校验 | 4 |
| 网络慢时如何预取 | QEMU、stargz、Nydus | 5 |
| 可写磁盘快照如何恢复 | E2B、OverlayBD | 6 |
| Guest RAM 如何按需恢复 | Firecracker、Cloud Hypervisor、Conch 增量恢复 | 7 |
| 如何统一恢复与回收 | E2B、Conch stores | 8 |

## 1. Conch 管资源关系，lazyd 接收明确的内容描述

**我们的设计。** Conch 解析 Boot Index，选择 rootfs EROFS descriptors，调用 lazyd prepare，并把准备结果关联到 Template/Sandbox 的资源引用。lazyd 负责内容准备与 FETCH；StratoVirt 消费明确的 lazy pmem source。当前不引入 fanotify。

**参考位置。** [Conch PR #184](https://gitcode.com/openeuler/Conch/pull/184) 及其 [store.go](https://gitcode.com/openeuler/Conch/blob/ae1e29ad8f07ad1838a6c4300ac0c8754c655066/internal/adapters/containerd/sandbox/store.go)、[Kata Nydus 集成设计](https://github.com/kata-containers/kata-containers/blob/26c2e1630457977d931bce0527c4df92a3b8f15d/docs/design/kata-nydus-design.md)。[SRC-CONCH-002] [SRC-KATA-003]

**对方具体做了什么。** Conch #184 将 Sandbox metadata 纳入 containerd store，并维护 Boot Index/runtime snapshot 的 GC 引用，让运行实例与所需资源的持久关系有统一 owner。Kata 的 Nydus 集成通过 snapshotter 提供 mount metadata，再由 runtime/shim 向 guest 传递和组装；镜像内容服务与 sandbox 生命周期各有职责。这两项共同提供的参考是：内容读取能力可以外置，但运行时必须知道正在使用哪份资源，并负责把挂载信息送到正确的 guest。

具体到接入点：Conch 的 `Store.Create/Update` 写入 Sandbox extension，并维护 `containerd.io/gc.ref.content.boot-index` 和 snapshot 引用。Kata 这份设计则描述将 RAFS source/config/snapshotdir 编入 mount `extraoption`，由 shim 解析并让 guest 组合 RAFS lower 与共享 upper/workdir。后者是仓库中的集成设计文档，不能仅凭它推定所有当前 Kata runtime 都走同一实现。

**我们怎样采用。** 复用 Conch 当前 store 和 BootPreparer 分层，将 lazy rootfs 建模为显式资源。Kata 的 mount metadata 传递是职责参考，不直接搬用其 snapshotter 或 guest 协议。准备失败时不发布可用 Template。

**状态与验收。** GitCode API 确认 #184 于 2026-09-03 合入。后续测试 prepare 失败不发布、资源引用重启后可恢复、普通完整启动不受影响；GC 引用不等于 lazyd 已实现外部缓存租约。

进一步阅读：[当前代码与接入点](01-current-system-boundary.md)、[Conch 职责](03-conch-responsibilities.md)、[Kata 生命周期分析](../02-products/11-kata-and-dragonball/02-cache-snapshot-lifecycle.md)。

## 2. 用内容摘要复用缓存，按 descriptor 准备

**我们的设计。** lazyd 以严格校验的 blob digest 标识 EROFS 内容；不同 image ref、layer index 和 VM 可引用同一 cache/bitmap/instance。`instance_id` 保持 prepared-content identity，StratoVirt 将它当 opaque ID。租户授权策略需独立校验，内容相同不自动意味着可跨安全域共享。

**参考位置。** [Nydus 镜像设计](https://github.com/dragonflyoss/nydus/blob/8aa80aee6e77a0c4d529581fc6339e9ff3066736/docs/nydus-design.md)、[SOCI zTOC/span 定义](https://github.com/awslabs/soci-snapshotter/blob/238af848f32fcb887072c144b09ee65a3a895f9c/docs/glossary.md)、[SOCI v1/v2 边界说明](https://github.com/awslabs/soci-snapshotter/blob/238af848f32fcb887072c144b09ee65a3a895f9c/README.md#no-image-conversion)。[SRC-NYDUS-001] [SRC-SOCI-004] [SRC-SOCI-001]

**对方具体做了什么。** Nydus 将文件系统 metadata 与 data blob 分离：metadata 描述 inode、文件偏移和 chunk/blob 位置，读取时定位所需内容，chunk 校验能力的生效条件见第 4 节。SOCI 的 zTOC 包含文件在解压后 TAR 中的位置，以及压缩流各检查点的解压状态；运行时由文件范围定位可独立解压的 span，无需从压缩流开头完整解压。SOCI README 特别区分 v1 的外置索引与 v2 的构建期转换，不能概括成所有 SOCI 模式都完全不改镜像。两者都需要把内容位置绑定到确定的镜像数据，不能只依赖可变 tag。

**我们怎样采用。** 借鉴稳定内容身份和索引驱动取数。我们的输入已是原生 EROFS descriptor，cache 以 EROFS 文件偏移组织，因此不引入 RAFS bootstrap 或 zTOC。descriptor-based prepare 是结合现有职责得出的本项目接口选择，不声称复制了对方同名 API。

**验收。** 同 digest、不同 image ref/index/VM 返回相同内容身份和缓存路径；digest 非法或同身份的 size/unit 配置冲突应明确失败；相同内容不能绕过调用方授权。共享 inode/文件页需另测，不能从 digest 相同直接推导物理页已共享。

进一步阅读：[Nydus 读取路径](../02-products/01-nydus-v2/01-data-path.md)、[SOCI 读取路径](../02-products/04-soci/01-data-path.md)、[缓存身份概念](../00-concepts/03-cache-sharing-and-identity.md)。

## 3. UFFD 触发取数，FD 映射共享文件页

**我们的设计。** StratoVirt 创建 anonymous/reserved HVA，注册 memslot 和 UFFD missing，启动 handler 后才允许 guest 访问。handler 将缺页地址换算为 offset，向 lazyd FETCH。lazyd 返回 cache FD 与完整 ready range；StratoVirt 校验后执行 fixed remap，再完成 wake。纯 padding 走零页处理。

**参考位置。** [Nydus PR #1921](https://github.com/dragonflyoss/nydus/pull/1921)、[`UffdCore::handle_page_fault` / `UffdWorker::handle_conn`](https://github.com/dragonflyoss/nydus/blob/8aa80aee6e77a0c4d529581fc6339e9ff3066736/service/src/block_uffd.rs)、[`BlockDevice::fetch_ranges`](https://github.com/dragonflyoss/nydus/blob/8aa80aee6e77a0c4d529581fc6339e9ff3066736/service/src/block_device.rs)、[`HandshakeRequest` / `VmaRegion`](https://github.com/dragonflyoss/nydus/blob/8aa80aee6e77a0c4d529581fc6339e9ff3066736/service/src/uffd_proto.rs)。[SRC-NYDUS-004] [SRC-NYDUS-005] [SRC-NYDUS-006] [SRC-NYDUS-007]

**对方具体做了什么。** VMM 在握手时传入 **UFFD FD 和 VMA regions**，Nydus 持有该 FD 并直接监听内核缺页事件，并非 VMM 每次发送 FETCH。`handle_page_fault` 用 `region.offset + fault_hva - base_hva` 定位 flattened block offset；`fetch_ranges` 再区分 metadata blob、data blob 和逻辑 hole。data blob 未就绪时先 `async_fetch`，随后返回 FD、blob offset、长度和逻辑 block offset。

Copy 分支由 Nydus 读取内容后 `UFFDIO_COPY`；Zerocopy 分支通过 SCM_RIGHTS 返回文件 FD/范围，映射由 VMM 客户端完成。逻辑 hole 和设备外区域由 Nydus zero 处理。可选 `enable_prefault` 在握手后异步调用 `fetch_ranges(..., probe_only=true)` 推送 ready 范围，**不是远端预取，也不构成“所有范围在 vCPU 启动前已映射”的屏障**。

这里的 hole 来自镜像逻辑块视图，不是“本地 sparse cache 尚未下载”的洞；后者必须先取数，不能当作合法零数据交给 guest。

**我们怎样采用。** 采用文件 FD 加范围描述的供数方式，但 handler 留在 StratoVirt，lazyd 不持有 VMM 的 UFFD。原生 EROFS cache 的 offset 关系也不同于 Nydus flattened RAFS view。FETCH v1 保持自己的契约，首版不引入 PROBE。每个 VM 仍独立处理自己的 fault 和 VMA，共享的是后端文件页。

**成熟度与验证。** 本次 API 核验：#1921 已合入；[Firecracker #5740](https://github.com/firecracker-microvm/firecracker/issues/5740) 仍是开放提案，作者报告了原型，但不等于上游支持；[Cloud Hypervisor #8239](https://github.com/cloud-hypervisor/cloud-hypervisor/pull/8239) `merged=false`，不能当上游运行时依赖。Nydus `VmaRegion` 的默认值为 `PROT_READ` 和 `MAP_PRIVATE|MAP_FIXED`，这是协议 region 的默认配置，实际 remap 标志仍由客户端实现决定，不能称为 Nydus 服务端执行了该 mmap。[SRC-FC-004] [SRC-CH-002]

后续分别验证映射权限与 KVM 只读保护、完整 ready range remap/wake、相邻未映射页、重复响应、尾页和双 VM PSS。Nydus fault loop 有处理失败后仅 `warn!` 的分支，不能据此假定已经具备我们要求的 VMM 终止闭环；我们的超时和 fatal 上报仍需单独验收。

进一步阅读：[Nydus UFFD 详细路径与差异](../02-products/01-nydus-v2/01-data-path.md)、[Copy 与共享映射](../03-design-comparison/05-copy-vs-shared-mapping.md)、[StratoVirt 职责](05-stratovirt-responsibilities.md)。

## 4. ready 是数据承诺，洞检测不是内容校验

**我们的设计。** 必需范围完整写入 cache 并完成同步后，才能设置并同步 bitmap ready。前后台读取共享 range 状态与 inflight 去重；恢复时保守处理不可信状态。

**参考位置。** [lazyd `ensure_range`](https://github.com/Fadeedee/lazyd/blob/751d647fb37fa2e01f6fb1784003e8b26aa00244/src/instance.rs)、[bitmap ready 写入](https://github.com/Fadeedee/lazyd/blob/751d647fb37fa2e01f6fb1784003e8b26aa00244/src/range_map.rs)、[Nydus `validate_chunk_data`](https://github.com/dragonflyoss/nydus/blob/8aa80aee6e77a0c4d529581fc6339e9ff3066736/storage/src/cache/mod.rs)。[SRC-LAZYD-001] [SRC-NYDUS-008]

**对方具体做了什么。** Nydus 的 `validate_chunk_data` 先检查解压长度，再根据 `need_validation()`、CRC32 标记或强制校验参数决定是否检查内容，并有 legacy stargz 例外；`check_digest` 也区分 CRC 与 hash。因此“支持校验”不等于所有后端、配置和格式下每次读取都进行了密码学摘要验证。当前 lazyd 源码则确认有 `write_all_at -> sync_data(cache) -> set_range_ready -> sync_data(bitmap)` 顺序，并将内容身份、大小和 unit 写入 bitmap header。

**我们怎样采用。** 持久化顺序是我们现有正确性要求，不将其包装成未经源码核对的 Nydus 同款实现。`SEEK_DATA/SEEK_HOLE` 只能检查明显空洞；完整 blob digest 也不能单独验证任意局部 range。若要局部强校验，需要可信分块摘要等额外依据，不能宣称仅有 bitmap 就能检测缓存内容损坏。

**验收。** 重开文件与重启恢复、部分写入失败、bitmap 同步失败、并发重叠范围、尾页补零。普通单元测试不能证明真实掉电一致性，需明确故障注入覆盖边界。

进一步阅读：[lazyd 当前基础与缺口](04-lazyd-responsibilities.md)、[缓存失败与安全](../03-design-comparison/07-lifecycle-failure-and-security.md)。

## 5. 当前缺页优先，预取量按访问与网络反馈调整

**我们的设计。** 当前范围 ready 即返回，预测范围后台处理。固定 bitmap unit，动态调整预取窗口、并发和预算。首次请求慢不直接触发扩大下载。

**参考位置。** [QEMU Fast Snapshot Load](https://github.com/qemu/qemu/blob/35500e5c41aec76cde59befe750600dac7a9e37a/docs/devel/migration/fast-snapshot-load.rst)、[eStargz 格式与预取](https://github.com/containerd/stargz-snapshotter/blob/c2bf18e5a94dcfd959cabf744f4bbb4ef8d980a2/docs/estargz.md#prioritized-files-and-landmark-files)、[Nydus Prefetch](https://github.com/dragonflyoss/nydus/blob/8aa80aee6e77a0c4d529581fc6339e9ff3066736/docs/nydus-design.md)。[SRC-QEMU-001] [SRC-STARGZ-002] [SRC-NYDUS-001]

**对方具体做了什么。** QEMU 的 fault thread 按 mapped-ram offset 直接读本地快照，eager thread 加载其余页面，`pending_bmap` 用于协调页面领取，防止重复装入覆盖运行中的 RAM。eager thread 还负责让恢复最终完成，否则冷页不再访问时可能长期停留在 migration 状态；它不是预测下一次网络访问的算法。

eStargz 在构建时把 prioritized files 放在 `.prefetch.landmark` 之前，文档描述在容器运行前以 HTTP Range 预取该区域。Nydus 的文件/目录 hints 则用于后台预取。本项目“当前 fault 不等预测范围”不能写成 eStargz 原样采用的启动策略；三者提供的是调度、构建期工作集、后台取数三个不同层面的参考。

**我们怎样采用。** 将这些思想映射到 lazyd range，而非复制内存页调度代码。延迟、吞吐、连续访问和预取利用率驱动的窗口调整，是本项目待验证建议；不能说这些项目已经实现了我们描述的同一策略。后台请求可能不可抢占，所以必须限制大小和并发，为前台保留容量。

**验收。** 在隔离网络环境比较 full、lazy 无预取、lazy 有预取的应用可用时间、首次请求、fault p99、下载放大和未使用预取字节，覆盖高延迟、限带宽、抖动及随机访问。

进一步阅读：[完整自适应预取策略](../03-design-comparison/06-cache-dedup-and-prefetch.md)、[QEMU 按需与后台加载](../02-products/09-qemu/01-data-path.md)。

## 6. 可写 upper 的历史数据按需读，新写入进入私有 COW

**我们的设计。** 将 writable disk snapshot 与只读 EROFS lower 分别建模。恢复时历史块层可按需读取，新写入进入本次运行的私有 writable head。

**参考位置。** [E2B `Overlay.ReadAt/WriteAt`](https://github.com/e2b-dev/infra/blob/cc7c574233ad98665a7c72a3d37b0af89ae79a71/packages/orchestrator/pkg/sandbox/block/overlay.go)、[`NewNBDProvider`](https://github.com/e2b-dev/infra/blob/cc7c574233ad98665a7c72a3d37b0af89ae79a71/packages/orchestrator/pkg/sandbox/rootfs/nbd.go)、[OverlayBD Writable layer / Live Snapshot](https://github.com/containerd/overlaybd/blob/f63addfd8e51bd9eeecc12f5f4670c281a74d6e8/README.md#writable-layer)。[SRC-E2B-002] [SRC-E2B-004] [SRC-OVERLAYBD-001]

**对方具体做了什么。** E2B 的 NBD provider 组合只读 rootfs 与私有 cache；当前 `Overlay.ReadAt` 依次查 writable cache、可选的 sealing cache、基础 device，`WriteAt` 只写当前 cache。后台封存快照时旧 cache 冻结、新写入切到新 cache，旧数据仍可读取，不能直接把“seal 中”当成“可删除”。这条 host NBD 磁盘供数链与内存 UFFD 分开；guest 看到的是 virtio 块设备，不是直接运行 NBD 客户端。

OverlayBD 组合只读块层和最顶层的一个 writable layer，写入差异进入 upper 的 index/data；普通 commit 将它转为后续可用的只读 lower，live snapshot 则在设备运行中切换到新的 upper。这里的块 upper 与 guest OverlayFS 的目录 upper 不是同一抽象。

**我们怎样采用。** 借鉴 parent chain 加 private head 的语义。实际 block backend、快照格式和协议需按 Conch 已合入能力选择，尚未据此决定采用 NBD 或 OverlayBD。ext4 走块设备是这里的方案选择，并非声称 ext4 在任何环境都只能使用 blk。

**验收。** 跨层读取选中最新块，私有写入不污染其他 VM，切换 writable head 时没有读空窗，snapshot 发布失败不丢失旧 head。若引入后台 seal，只有完成必要同步并保持引用后才能发布或回收；此项不是当前 rootfs FETCH v1 已提供的能力。

进一步阅读：[E2B 双路读取](../02-products/12-e2b-and-firecracker-stacks/01-data-path.md)、[可写 upper 比较](../03-design-comparison/02-writable-upper-and-snapshot.md)。

## 7. 内存懒恢复复用独立 page source

**我们的设计。** Guest RAM 的 full/incremental snapshot 使用独立内存来源与谱系，不使用 rootfs `instance_id`。先确认现有路径是否要求完整下载，再决定是否补远端按需取页。

**参考位置。** [Firecracker page fault 处理](https://github.com/firecracker-microvm/firecracker/blob/7699746649826d1dfcdde626b3131bac08f28e0d/docs/snapshotting/handling-page-faults-on-snapshot-resume.md)、[Cloud Hypervisor v53](https://github.com/cloud-hypervisor/cloud-hypervisor/releases/tag/v53.0)、[Conch #155](https://gitcode.com/openeuler/Conch/pull/155) 的 [`handlePageFault`](https://gitcode.com/openeuler/Conch/blob/1ca332dca323a209b3bcdb9534f7fb50b2360290/internal/cow/uffd.go)、[StratoVirt #2017](https://gitcode.com/openeuler/stratovirt/pull/2017)。[SRC-FC-002] [SRC-CH-001] [SRC-CONCH-003] [SRC-SV-002]

**对方具体做了什么。** Firecracker 把 UFFD FD 和 memory layout 交给外部 handler，文档示例由 handler mmap 快照后 `UFFDIO_COPY`。这不是内置 registry 客户端。Cloud Hypervisor v53 发布了 snapshot/restore offload daemon、按需取页与后台 prefault，不能因 #8239 未合入便否认这些内存恢复能力，也不能将它们等同于 pmem FETCH v1。

Conch #155 的 `handlePageFault` 通过 `PinnedManifest.ReadPage` 解析增量页来源，`memfd.WriteAt` 后 `wake`；StratoVirt #2017 提供 inherited memfd backend。**2026-09-08 API 核查两项仍为 open**，head 分别为 `1ca332dca323` 和 `d70c76b78f9e`，不能写成已合入前置。

**我们怎样采用。** 复用内存 source、attachment 和恢复生命周期；rootfs 保持不可变 cache FD 映射。内存按需装入 RAM 不等于快照文件已经支持远端懒下载。合入状态及远端读取能力需在开发前重新核对。

**验收。** 单独记录快照下载量与 RAM 实际填页量；测试最新增量覆盖父层、逻辑零页、缺失父层与恢复后写入隔离。若启用 balloon/discard，还需处理 REMOVE 后再 fault 的语义，不得重新注入已丢弃的旧快照数据。

进一步阅读：[内存恢复流程](../05-flows/02-memory-lazy-restore/README.md)、[内存快照比较](../03-design-comparison/03-memory-snapshot-restore.md)。

## 8. 三路统一 readiness、失败与引用管理

**我们的设计。** Conch 管 rootfs、writable disk、Guest RAM 与 VMM state 的资源图。每路达到 Faultable、handler ready 后才 resume；运行期致命错误传到 Sandbox lifecycle。单个 VM 退出仅释放自己的引用，共享内容按 GC 策略回收。

**参考位置。** [E2B Infra 架构](https://github.com/e2b-dev/infra/blob/cc7c574233ad98665a7c72a3d37b0af89ae79a71/docs/ARCHITECTURE.md)、[E2B `faultPage`](https://github.com/e2b-dev/infra/blob/cc7c574233ad98665a7c72a3d37b0af89ae79a71/packages/orchestrator/pkg/sandbox/uffd/userfaultfd/userfaultfd.go)、[Firecracker Caveats](https://github.com/firecracker-microvm/firecracker/blob/7699746649826d1dfcdde626b3131bac08f28e0d/docs/snapshotting/handling-page-faults-on-snapshot-resume.md#caveats)、[Conch #184](https://gitcode.com/openeuler/Conch/pull/184)。[SRC-E2B-001] [SRC-E2B-003] [SRC-FC-002] [SRC-CONCH-002]

**对方具体做了什么。** E2B 在 sandbox/template 编排下组合内存 UFFD 与磁盘 COW 两条数据路径，说明一个恢复操作可以协调多类后端。Conch 原生 stores 维护运行资源与持久 metadata 的引用关系，提供本项目接入生命周期和 GC 的位置。

失败策略另有直接证据：E2B `faultPage` 对 source 读取执行有限退避重试，最终失败调用 `onFailure`（若已提供）、记录错误并返回；仅凭该函数不能断言所有调用方都已终止 VM。Firecracker 明确要求外部监控 handler 并在崩溃时回收 VM，否则 fault 可能一直等待。这说明 UFFD 供页接口本身不提供完整的 Sandbox 失败策略。

**我们怎样采用。** `MetadataReady/Faultable/Materialized/Failed` 和统一恢复屏障是本项目建议的状态模型，并非对方同名 API。containerd 引用保护和 lazyd 的外部 cache lease 需要明确桥接；仅写 containerd GC label 不会自动保护 lazyd 文件。没有 lease/refcount 时保守保留共享 cache。

**验收。** 任一路准备失败不 resume；运行期 fatal 终止受影响 VM；两个 VM 共享内容时删除一个不影响另一个；daemon 重启、取消和 GC 与 inflight 并发时引用一致。

进一步阅读：[三路恢复详细设计](06-checkpoint-three-path-restore.md)、[开发顺序与验收](07-phased-roadmap.md)、[三路恢复交互图](../05-flows/03-checkpoint-template-restore/assets/checkpoint-three-path.html)。

## 本次核查的边界

本轮保持八项设计方向，修正了 handler 归属、校验前提、prefault 时序、后台恢复目的和 COW 封存期间的读取关系。外部项目的实际实现不等于本项目已完成适配，更不等于本项目性能已经优于对方。

- GitHub 引用已读取固定 commit 的相关文件；GitCode PR 页面工具无法解析时改用官方 API 核验状态，Conch/SV PR 代码对照同一 head 的本地对象。
- lazyd 持久化顺序已复查本地 `751d647` 源码；本次未重跑其测试或真实掉电实验。
- RTT 自适应窗口、三路统一状态和跨服务 lease 仍是本项目设计建议。没有新的网络性能、VM remap/wake 或多 VM 共享实测结果。
