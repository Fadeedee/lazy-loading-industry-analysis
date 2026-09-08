# 设计决策与业界依据

> 阅读完成后，读者能够逐项说明我们准备怎样设计、参考项目具体做了什么、参考资料在哪里，以及哪些细节仍是本项目的待验证建议。

本文基于本调研已记录的 2026-09-04 证据，设计范围说明更新于 2026-09-08；不是对远端最新状态的再次核查。链接到分支的官方资料可能变化，版本以 [来源清单](../appendix/source-inventory.md) 为准。以下“我们的设计”表示目标，不表示已在当前上游实现。

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

**参考位置。** [Conch PR #184](https://gitcode.com/openeuler/Conch/pull/184)、[Kata Nydus 集成设计](https://github.com/kata-containers/kata-containers/blob/main/docs/design/kata-nydus-design.md)。[SRC-CONCH-002] [SRC-KATA-003]

**对方具体做了什么。** Conch #184 将 Sandbox metadata 纳入 containerd store，并维护 Boot Index/runtime snapshot 的 GC 引用，让运行实例与所需资源的持久关系有统一 owner。Kata 的 Nydus 集成通过 snapshotter 提供 mount metadata，再由 runtime/shim 向 guest 传递和组装；镜像内容服务与 sandbox 生命周期各有职责。这两项共同提供的参考是：内容读取能力可以外置，但运行时必须知道正在使用哪份资源，并负责把挂载信息送到正确的 guest。

**我们怎样采用。** 复用 Conch 当前 store 和 BootPreparer 分层，将 lazy rootfs 建模为显式资源。Kata 的 mount metadata 传递是职责参考，不直接搬用其 snapshotter 或 guest 协议。准备失败时不发布可用 Template。

进一步阅读：[当前代码与接入点](01-current-system-boundary.md)、[Conch 职责](03-conch-responsibilities.md)、[Kata 生命周期分析](../02-products/11-kata-and-dragonball/02-cache-snapshot-lifecycle.md)。

## 2. 用内容摘要复用缓存，按 descriptor 准备

**我们的设计。** lazyd 以严格校验的 blob digest 标识 EROFS 内容；不同 image ref、layer index 和 VM 可引用同一 cache/bitmap/instance。`instance_id` 保持 prepared-content identity，StratoVirt 将它当 opaque ID。租户授权策略需独立校验，内容相同不自动意味着可跨安全域共享。

**参考位置。** [Nydus 镜像设计](https://github.com/dragonflyoss/nydus/blob/master/docs/nydus-design.md)、[SOCI 项目说明](https://github.com/awslabs/soci-snapshotter)。[SRC-NYDUS-001] [SRC-SOCI-001]

**对方具体做了什么。** Nydus 将文件系统 metadata 与 data blob 分离：metadata 描述 inode、文件偏移和 chunk/blob 位置，读取时只获取所需内容，并通过 chunk 摘要验证数据。SOCI 为 OCI 压缩层建立 zTOC 索引，运行时借助索引定位所需 span，避免为了读一个文件先完整解压整个 layer。两者都需要把“我要读的内容”稳定绑定到镜像数据，而不能只依赖会变化的 tag。

**我们怎样采用。** 借鉴稳定内容身份和索引驱动取数。我们的输入已是原生 EROFS descriptor，cache 以 EROFS 文件偏移组织，因此不引入 RAFS bootstrap 或 zTOC。descriptor-based prepare 是结合现有职责得出的本项目接口选择，不声称复制了对方同名 API。

进一步阅读：[Nydus 读取路径](../02-products/01-nydus-v2/01-data-path.md)、[SOCI 读取路径](../02-products/04-soci/01-data-path.md)、[缓存身份概念](../00-concepts/03-cache-sharing-and-identity.md)。

## 3. UFFD 触发取数，FD 映射共享文件页

**我们的设计。** StratoVirt 创建 anonymous/reserved HVA，注册 memslot 和 UFFD missing，启动 handler 后才允许 guest 访问。handler 将缺页地址换算为 offset，向 lazyd FETCH。lazyd 返回 cache FD 与完整 ready range；StratoVirt 校验后执行 fixed remap，再完成 wake。纯 padding 走零页处理。

**参考位置。** [Nydus PR #1921](https://github.com/dragonflyoss/nydus/pull/1921)、[block_uffd.rs](https://github.com/dragonflyoss/nydus/blob/master/service/src/block_uffd.rs)、[block_device.rs](https://github.com/dragonflyoss/nydus/blob/master/service/src/block_device.rs)、[uffd_proto.rs](https://github.com/dragonflyoss/nydus/blob/master/service/src/uffd_proto.rs)。[SRC-NYDUS-004]

**对方具体做了什么。** Nydus 的 UFFD service 接收并处理 VMM 提供的缺页信息，将 flattened block offset 解析到 RAFSv6 metadata/data blob 的后端范围，调用内容读取和缓存能力。Copy 路径通过 `UFFDIO_COPY` 填页；Zerocopy 路径通过 SCM_RIGHTS 传 FD，并描述可映射的 region，由 VMM 执行固定地址映射。其 block device 层负责逻辑块视图到实际后端范围的转换，协议层负责消息与 FD 规则；此外还有 prefault，用于提前映射已就绪范围。

**我们怎样采用。** 采用文件 FD 加范围描述的供数方式，但 handler 留在 StratoVirt，lazyd 不持有 VMM 的 UFFD。原生 EROFS cache 的 offset 关系也不同于 Nydus flattened RAFS view。FETCH v1 保持自己的契约，首版不引入 PROBE。每个 VM 仍独立处理自己的 fault 和 VMA，共享的是后端文件页。

**成熟度与验证。** 调研时 #1921 已合入；[Firecracker #5740](https://github.com/firecracker-microvm/firecracker/issues/5740) 是补充设计提案，[Cloud Hypervisor #8239](https://github.com/cloud-hypervisor/cloud-hypervisor/pull/8239) 已关闭未合入，不能当上游运行时依赖。Nydus 默认 `MAP_PRIVATE|MAP_FIXED`；我们仍需验证映射标志、KVM 只读保护、remap/wake、相邻缺页与双 VM PSS。[SRC-FC-004] [SRC-CH-002]

进一步阅读：[Nydus UFFD 详细路径与差异](../02-products/01-nydus-v2/01-data-path.md)、[Copy 与共享映射](../03-design-comparison/05-copy-vs-shared-mapping.md)、[StratoVirt 职责](05-stratovirt-responsibilities.md)。

## 4. ready 是数据承诺，洞检测不是内容校验

**我们的设计。** 必需范围完整写入 cache 并完成同步后，才能设置并同步 bitmap ready。前后台读取共享 range 状态与 inflight 去重；恢复时保守处理不可信状态。

**参考位置。** [当前 lazyd 实现分支](https://github.com/Fadeedee/lazyd/tree/feature/lazy-uffd-range-api)、[Nydus 镜像格式](https://github.com/dragonflyoss/nydus/blob/master/docs/nydus-image.md)。[SRC-LAZYD-001] [SRC-NYDUS-002]

**对方具体做了什么。** Nydus 镜像 metadata 携带 chunk 位置和校验信息，读取对应数据时可验证 chunk。它说明了“有数据”和“数据可信”是两个条件。当前 lazyd 已有 `write_all_at -> sync_data(cache) -> set_range_ready -> sync_data(bitmap)` 顺序，并将内容身份、大小和 unit 写入 bitmap header。

**我们怎样采用。** 持久化顺序是我们现有正确性要求，不将其包装成未经源码核对的 Nydus 同款实现。`SEEK_DATA/SEEK_HOLE` 只能检查明显空洞；完整 blob digest 也不能单独验证任意局部 range。若要局部强校验，需要可信分块摘要等额外依据，不能宣称仅有 bitmap 就能检测缓存内容损坏。

**验收。** 重开文件与重启恢复、部分写入失败、bitmap 同步失败、并发重叠范围、尾页补零。普通单元测试不能证明真实掉电一致性，需明确故障注入覆盖边界。

进一步阅读：[lazyd 当前基础与缺口](04-lazyd-responsibilities.md)、[缓存失败与安全](../03-design-comparison/07-lifecycle-failure-and-security.md)。

## 5. 当前缺页优先，预取量按访问与网络反馈调整

**我们的设计。** 当前范围 ready 即返回，预测范围后台处理。固定 bitmap unit，动态调整预取窗口、并发和预算。首次请求慢不直接触发扩大下载。

**参考位置。** [QEMU Fast Snapshot Load](https://www.qemu.org/docs/master/devel/migration/fast-snapshot-load.html)、[eStargz 格式与预取](https://github.com/containerd/stargz-snapshotter/blob/main/docs/estargz.md)、[Nydus 设计](https://github.com/dragonflyoss/nydus/blob/master/docs/nydus-design.md)。[SRC-QEMU-001] [SRC-STARGZ-002] [SRC-NYDUS-001]

**对方具体做了什么。** QEMU 将恢复中的待加载页面记录为 pending 状态：运行期缺页通过 postcopy 相关路径优先取得页面，后台加载器处理其余页面，两者必须协调完成状态。eStargz 可在构建时将启动常用文件放到优先区域，并用 landmark 标识预取边界。Nydus 支持文件/目录预取信息，提前准备预计会访问的镜像数据。前者提供前后台调度参考，后两者提供工作集预取参考。

**我们怎样采用。** 将这些思想映射到 lazyd range，而非复制内存页调度代码。延迟、吞吐、连续访问和预取利用率驱动的窗口调整，是本项目待验证建议；不能说这些项目已经实现了我们描述的同一策略。后台请求可能不可抢占，所以必须限制大小和并发，为前台保留容量。

**验收。** 在隔离网络环境比较 full、lazy 无预取、lazy 有预取的应用可用时间、首次请求、fault p99、下载放大和未使用预取字节，覆盖高延迟、限带宽、抖动及随机访问。

进一步阅读：[完整自适应预取策略](../03-design-comparison/06-cache-dedup-and-prefetch.md)、[QEMU 按需与后台加载](../02-products/09-qemu/01-data-path.md)。

## 6. 可写 upper 的历史数据按需读，新写入进入私有 COW

**我们的设计。** 将 writable disk snapshot 与只读 EROFS lower 分别建模。恢复时历史块层可按需读取，新写入进入本次运行的私有 writable head。

**参考位置。** [E2B Infra 架构](https://github.com/e2b-dev/infra/blob/main/docs/ARCHITECTURE.md)、[OverlayBD](https://github.com/containerd/overlaybd)。[SRC-E2B-001] [SRC-OVERLAYBD-001]

**对方具体做了什么。** E2B 的磁盘路径使用只读 template 数据和 per-sandbox COW 缓存：读取先检查本次变更，未命中再到基础数据，写入保存在私有变更块中。其磁盘后端采用 NBD，和 Firecracker 的内存 UFFD 路径分开。OverlayBD 通过块设备请求访问远端只读层，同时提供可写层及提交/快照能力，让普通文件系统在块视图上运行。

**我们怎样采用。** 借鉴 parent chain 加 private head 的语义。实际 block backend、快照格式和协议需按 Conch 已合入能力选择，尚未据此决定采用 NBD 或 OverlayBD。ext4 走块设备是这里的方案选择，并非声称 ext4 在任何环境都只能使用 blk。

进一步阅读：[E2B 双路读取](../02-products/12-e2b-and-firecracker-stacks/01-data-path.md)、[可写 upper 比较](../03-design-comparison/02-writable-upper-and-snapshot.md)。

## 7. 内存懒恢复复用独立 page source

**我们的设计。** Guest RAM 的 full/incremental snapshot 使用独立内存来源与谱系，不使用 rootfs `instance_id`。先确认现有路径是否要求完整下载，再决定是否补远端按需取页。

**参考位置。** [Firecracker page fault 处理](https://github.com/firecracker-microvm/firecracker/blob/main/docs/snapshotting/handling-page-faults-on-snapshot-resume.md)、[Cloud Hypervisor v53](https://github.com/cloud-hypervisor/cloud-hypervisor/releases/tag/v53.0)、[Conch #155](https://gitcode.com/openeuler/Conch/pull/155)、[StratoVirt #2017](https://gitcode.com/openeuler/stratovirt/pull/2017)。[SRC-FC-002] [SRC-CH-001] [SRC-CONCH-003] [SRC-SV-002]

**对方具体做了什么。** Firecracker 可把 UFFD 和 memory layout 通过 Unix socket 交给外部 handler；handler 根据缺页地址定位快照数据并用 `UFFDIO_COPY` 填页。Cloud Hypervisor v53 发布说明确认 demand-paged memory restore，但仅凭发布说明不能推导它具备我们的 rootfs FD 协议。调研时 Conch #155 通过 incremental memory map 解析页来源，由 conch-cow 向 inherited memfd 写页并 wake；StratoVirt #2017 提供 inherited memfd backend，两项当时均为开放 PR。

**我们怎样采用。** 复用内存 source、attachment 和恢复生命周期；rootfs 保持不可变 cache FD 映射。内存按需装入 RAM 不等于快照文件已经支持远端懒下载。合入状态及远端读取能力需在开发前重新核对。

进一步阅读：[内存恢复流程](../05-flows/02-memory-lazy-restore/README.md)、[内存快照比较](../03-design-comparison/03-memory-snapshot-restore.md)。

## 8. 三路统一 readiness、失败与引用管理

**我们的设计。** Conch 管 rootfs、writable disk、Guest RAM 与 VMM state 的资源图。每路达到 Faultable、handler ready 后才 resume；运行期致命错误传到 Sandbox lifecycle。单个 VM 退出仅释放自己的引用，共享内容按 GC 策略回收。

**参考位置。** [E2B Infra 架构](https://github.com/e2b-dev/infra/blob/main/docs/ARCHITECTURE.md)、[Conch #184](https://gitcode.com/openeuler/Conch/pull/184)。[SRC-E2B-001] [SRC-CONCH-002]

**对方具体做了什么。** E2B 在 sandbox/template 编排下组合内存 UFFD 与磁盘 COW 两条数据路径，说明一个恢复操作可以协调多类后端。Conch 原生 stores 维护运行资源与持久 metadata 的引用关系，提供本项目接入生命周期和 GC 的位置。

**我们怎样采用。** `MetadataReady/Faultable/Materialized/Failed` 和统一恢复屏障是本项目建议的状态模型，并非对方同名 API。containerd 引用保护和 lazyd 的外部 cache lease 需要明确桥接；仅写 containerd GC label 不会自动保护 lazyd 文件。没有 lease/refcount 时保守保留共享 cache。

**验收。** 任一路准备失败不 resume；运行期 fatal 终止受影响 VM；两个 VM 共享内容时删除一个不影响另一个；daemon 重启、取消和 GC 与 inflight 并发时引用一致。

进一步阅读：[三路恢复详细设计](06-checkpoint-three-path-restore.md)、[开发顺序与验收](07-phased-roadmap.md)、[三路恢复交互图](../05-flows/03-checkpoint-template-restore/assets/checkpoint-three-path.html)。
