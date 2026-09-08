# 从用户场景推导三仓方案

> 不从某个 API 开始。先问一次启动或恢复要解决什么，再比较业界做法，决定三个仓共同承担哪些改动。

本文是 2026-09-08 的设计建议。上游事实见[核查基线](01-current-system-boundary.md)；产品证据保留[来源清单](../appendix/source-inventory.md)中的核查版本。本次没有新增产品测试或性能数字。

## 先看设计主线

**准备镜像或快照 → 创建或恢复沙箱 → 按需访问 → 保存快照 → 停止与回收。**

我们希望业务更早可用，不是只让 pull 更快。以下八项分别说明建议、替代方案、证据及验证；接口名和进程数尚未冻结。

**联合设计，分清改动边界：** Conch 管资源和整体流程，lazyd/数据源管内容与供数，StratoVirt 只补设备、地址空间、页完成和运行控制的必要能力。下文候选用于选型，不是首版功能累加清单；VMM 的具体范围见[StratoVirt 职责](05-stratovirt-responsibilities.md)。

| 用户遇到的问题 | 从哪里看 |
| --- | --- |
| 镜像太大，必须全下载才能启动吗 | 第 1 节 |
| 同一镜像运行多台 VM，能共享什么 | 第 2、3 节 |
| 下载中断或缓存损坏，会读错吗 | 第 4 节 |
| 网络慢，按需读取反而更慢怎么办 | 第 5 节 |
| 文件和内存快照怎样恢复 | 第 6、7 节 |
| 失败、删除和回收归谁管 | 第 8 节 |

## 1. 创建一个沙箱：哪些资源需要先准备？

### 场景与建议

用户指定一个 Template，Conch 需要知道“要启动哪个版本、需要哪些设备、哪些内容必须先到位”。建议先形成**资源计划**：只读镜像、磁盘历史层、内存快照和设备状态分别是什么，每项使用完整准备还是按需准备。

这不是给所有对象统一加一个 lazy 开关。小镜像可以完整取回；大镜像可以按需；内存来源是否支持远端读取另行协商。索引和关键启动状态必须足够完整，才能判断后续读取位置。

### 对方做了什么

[Kata Nydus 集成设计](https://github.com/kata-containers/kata-containers/blob/26c2e1630457977d931bce0527c4df92a3b8f15d/docs/design/kata-nydus-design.md)把 mount metadata 从 host 传给运行时和 guest，由 guest 组装文件系统。[E2B 架构](https://github.com/e2b-dev/infra/blob/cc7c574233ad98665a7c72a3d37b0af89ae79a71/docs/ARCHITECTURE.md)则将磁盘和内存恢复放在同一个沙箱操作里。两者说明“提供数据”与“把资源组合成可运行沙箱”是不同职责。[SRC-KATA-003] [SRC-E2B-001]

详细过程见 [Kata](../02-products/11-kata-and-dragonball/01-data-path.md)和 [E2B](../02-products/12-e2b-and-firecracker-stacks/01-data-path.md)。

### 三仓怎样协作

Conch 解析用户意图、组织资源引用和失败回滚；lazyd 或已有数据源适配器准备内容；StratoVirt 报告设备/内存接入能力。guestd 负责 guest 内的设备匹配与挂载。

可以在 Conch 原生 BootPreparer 增加按需 source，也可以借助现有 snapshotter/mount 契约。选择取决于谁真实管理相应资源；不能为了走通接口而宣称未就绪数据已经完整 unpack，也不预先排除语义真实的 snapshotter 方案。

**验证：** selective pull 不完整取回目标 payload；元数据缺失会失败；准备中取消不发布“可启动”；普通启动行为保持正确。

## 2. 两台 VM 用同一镜像：先复用内容，再管理使用关系

### 场景与建议

VM1 下载了一个库，VM2 应在授权允许时直接复用。缓存身份要描述**哪份内容**，而不是“哪台 VM 的第几层”。

建议分清三种身份：不可变内容、某次磁盘/内存快照视图、某个运行 attachment。具体字段名可重新设计；同一摘要也不能绕过授权或忽略格式、大小冲突。

### 对方做了什么

[Nydus 镜像设计](https://github.com/dragonflyoss/nydus/blob/8aa80aee6e77a0c4d529581fc6339e9ff3066736/docs/nydus-design.md)用 metadata 定位 chunk/blob。[SOCI zTOC](https://github.com/awslabs/soci-snapshotter/blob/238af848f32fcb887072c144b09ee65a3a895f9c/docs/glossary.md)保存文件位置和压缩流检查点，让读者定位可独立处理的范围。共同点是把读取位置绑定到确定内容，不依赖可变 tag；并不是双方都有同名 prepare API。[SRC-NYDUS-001] [SRC-SOCI-004]

### 三仓怎样协作

Conch 保存可跨节点使用的资源描述与授权引用；lazyd 管内容索引和本地缓存；StratoVirt 使用本次 attachment 的区域布局和不透明来源标识，不解析镜像 tag、digest 或 cache key。

可以按整 blob、chunk 或组合 artifact 缓存。若使用可映射文件，共享物理页还要求映射到同一实际文件页；两个相同 digest 的独立缓存文件不会自动共享 page cache。

**验证：** 不同 Template/VM 引用相同内容不重复取数；越权访问被拒绝；节点迁移不依赖另一台机器的 socket、FD 数字或缓存路径。

## 3. 第一次读文件：谁接缺页，怎样把数据交给 VM？

### 先看两个独立选择

以两台 VM 读取 Python 为例。“下载一次”只说明网络复用；若再复制进各自的匿名内存，仍可能占多份物理页。

要进一步共享文件页，可以采用 pmem/DAX 路径；也可以用块设备供应整盘，换取更统一的读写快照管理。我们优先验证前者的共享收益，同时保留后者作为端到端成本对照，不预设 pmem 一定更快。

选了 UFFD 后，还要分别决定：

| 决策 | 候选 A | 候选 B |
| --- | --- | --- |
| 谁接内核缺页事件 | StratoVirt 内部 handler | 外部 handler，可放在 lazyd 的类型化适配层 |
| 怎样完成当前页 | COPY 到 VM 私有目标页 | 文件 FD + 映射，由 VMM 替换自己的地址区间 |

handler 外置，不意味着外部进程能直接对 StratoVirt 的 HVA 执行普通 mmap。选择文件映射时，VMM 仍需执行或严格控制自身地址空间的修改。

### Nydus 具体做了什么

[Nydus block_uffd.rs](https://github.com/dragonflyoss/nydus/blob/8aa80aee6e77a0c4d529581fc6339e9ff3066736/service/src/block_uffd.rs)在握手时接收 VMM 的 UFFD FD 和区域描述，再直接读取内核缺页事件。服务定位镜像逻辑范围，补齐缓存，可走复制填页，也可把文件区域交回 VMM。[block_device.rs](https://github.com/dragonflyoss/nydus/blob/8aa80aee6e77a0c4d529581fc6339e9ff3066736/service/src/block_device.rs)负责逻辑块到后端数据的定位。[SRC-NYDUS-005] [SRC-NYDUS-007]

[uffd_proto.rs](https://github.com/dragonflyoss/nydus/blob/8aa80aee6e77a0c4d529581fc6339e9ff3066736/service/src/uffd_proto.rs)的映射描述默认 READ + PRIVATE + FIXED；实际 mmap 由客户端完成。干净的 PRIVATE 文件页也能共享，因此不能只凭 SHARED 这个名字选标志。[SRC-NYDUS-006]

详见 [Nydus 数据路径](../02-products/01-nydus-v2/01-data-path.md)和 [Copy/映射比较](../03-design-comparison/05-copy-vs-shared-mapping.md)。

### 我们先验证哪条？

**先验证外部 handler，首版只落地一种新增 pmem 路径。** 理由是本次核查的 StratoVirt 上游已有 RAM 的外部 UFFD 交接机制，可评估复用；这不是已证明 pmem 可直接套用。内部方式先作设计或已有实验对照，只有遇到具体问题才补最小原型，不要求两套同时产品化。权限、连接中断、事件代次、remap 完成通知和就绪条件仍是必需边界。

Conch 管 attachment 生命周期；lazyd 可增加独立于内容核心的 UFFD 适配模块；StratoVirt 管内存区域、接收已就绪文件范围、校验和完成映射。若外置方式明显扩大故障范围或协议复杂度，再选择内部 handler，不为复用而强行外置。无论哪种位置，内容索引、下载、缓存和预取仍在数据源侧，不能把它们随 handler 搬进 VMM。

VM2 仍有自己的地址和首次缺页。数据已缓存时不需再下载，但它仍需建立自己的映射。**共享内容不等于自动同步所有 VM 的页表。**

### 必须测什么

对照完整镜像读取内容；测试尾页与对齐补零；验证只读权限、整段映射及唤醒、相邻未处理页、并发事件、超时和 handler 崩溃。两 VM 同时读时同时观察远端字节和 PSS，不能只看 RSS 求和。

<details>
<summary>地址与协议细节：按需展开</summary>

HVA 是 VMM 的宿主机虚拟地址，GPA 是 guest 物理地址。handler 根据区域布局把缺页地址转换成区域内偏移；数据源再将逻辑范围解析成内容文件/偏移。后一步不一定等于 `fault_hva - base_hva`，组合设备或增量层的内容索引留在数据源侧，VMM 不承担这类解析。

UFFD FD 是缺页通知/处理句柄，缓存文件 FD 是数据访问句柄。它们可通过 SCM_RIGHTS 传递，但生命周期和权限不同。

`mmap(MAP_FIXED)` 会替换 VMA 子区间，必须检查文件类型、大小、偏移、范围、对齐、映射存活期和 UFFD wake/resolve 行为；队列中重复或过期事件也必须有规则。普通 sparse 文件 mmap 不能将“未下载”区别于零，需真正的缺失数据拦截机制。

[Nydus #1921](https://github.com/dragonflyoss/nydus/pull/1921)是已合入服务端证据；[Firecracker #5740](https://github.com/firecracker-microvm/firecracker/issues/5740)是提案与作者原型报告；[CH #8239](https://github.com/cloud-hypervisor/cloud-hypervisor/pull/8239)截至来源记录日期关闭未合入。不能将它们等同于本项目已完成的实现或测试。[SRC-NYDUS-004] [SRC-FC-004] [SRC-CH-002]

</details>

## 4. 下载到一半断电：什么才叫“可读”？

建议把临时下载、完成校验、当前可读、可持久复用区分清楚。如果 ready 记录用于重启后跳过下载，它必须晚于相应数据持久化。

[Nydus validate_chunk_data](https://github.com/dragonflyoss/nydus/blob/8aa80aee6e77a0c4d529581fc6339e9ff3066736/storage/src/cache/mod.rs)检查长度并按配置/格式决定校验；CRC 不等于密码学摘要。[lazyd 样本](https://github.com/Fadeedee/lazyd/blob/751d647fb37fa2e01f6fb1784003e8b26aa00244/src/instance.rs)已有先 cache sync、再 bitmap ready/sync 的顺序。这些分别是校验和持久化证据，不能混为一谈。[SRC-NYDUS-008] [SRC-LAZYD-001]

可选持久 bitmap，也可使用原子发布的分块文件或日志索引。选择根据同步成本、并发、掉电恢复和格式兼容决定，不先固定 bitmap 格式。SEEK_DATA/HOLE 只能发现明显洞；任意 range 的强校验需要可信局部摘要等依据。

Conch 只接收清楚的准备状态；内容服务负责数据正确性；StratoVirt 不映射未经承诺的缺失范围。可批量同步，但不能让持久 ready 领先数据。

**验证：** 部分写入、同步失败、重开文件、重启、缓存损坏及错误响应。常规单测不能代替真实掉电测试。

## 5. 网络慢：谁决定多拉一点？

用户关心“多久能用”，不关心下载服务是否提前返回。建议分别验证三种优化：启动工作集预取、根据访问反馈调整窗口、提前映射已经缓存的范围。

[eStargz](https://github.com/containerd/stargz-snapshotter/blob/c2bf18e5a94dcfd959cabf744f4bbb4ef8d980a2/docs/estargz.md)把 prioritized files 放在 landmark 前，文档描述启动前预取；[QEMU Fast Snapshot Load](https://github.com/qemu/qemu/blob/35500e5c41aec76cde59befe750600dac7a9e37a/docs/devel/migration/fast-snapshot-load.rst)的后台线程用于完成 RAM 恢复，与当前缺页协调页面领取，不是网络预测算法。[SRC-STARGZ-002] [SRC-QEMU-001]

Nydus 的 prefault 是握手后异步枚举已缓存范围，不代表预测下载，也不保证启动前全部映射完。[SRC-NYDUS-007]

建议 Conch 提供场景、截止时间和节点预算，lazyd 调度下载。先使用 handler 已收到的事件和现有指标；只有测量证明必要时，才评估最小的 VMM 访问/驻留反馈接口。新增访问追踪框架不是首版前置，网络策略和预取算法不进入 StratoVirt。

单次请求慢可能来自鉴权、连接或拥塞，不能据此无限扩大范围。当前需求优先，后台请求有大小和并发上限；启动前等待预取是单独策略，必须计入启动成本。

**验证：** full、纯按需、工作集预取和提前映射分别对照；覆盖高延迟、限带宽、随机访问、小镜像和多 VM。详见[网络与预取](../03-design-comparison/06-cache-dedup-and-prefetch.md)。

## 6. 恢复写入的文件：分离镜像与磁盘，还是统一整盘？

有两条合理候选：只读镜像 pmem + 私有块写层，或整盘块快照 + COW。前者可能节省只读页复制，后者可能减少设备和快照组合复杂度。不要先因“已有 pmem”排除整盘方案。

[E2B Overlay](https://github.com/e2b-dev/infra/blob/cc7c574233ad98665a7c72a3d37b0af89ae79a71/packages/orchestrator/pkg/sandbox/block/overlay.go)读取依次检查 writable cache、封存中的 cache 和 base，新写入只进入当前 cache。[OverlayBD](https://github.com/containerd/overlaybd/blob/f63addfd8e51bd9eeecc12f5f4670c281a74d6e8/README.md)提供只读层加可写顶层，并支持提交或切换写层。[SRC-E2B-002] [SRC-OVERLAYBD-001]

借鉴的是“历史内容不变，新写入进入私有状态”，不是直接选定 NBD、TCMU 或某个文件系统。Conch 管快照资源与设备布局；数据后端解释块覆盖和写层；StratoVirt 暴露设备并参与一致性屏障。

整盘候选先利用已有块设备及外部后端作对照，不要求 StratoVirt 为选型新增一套 NBD/COW 引擎；若现有接入不足，先记录缺口和成本，再决定是否扩大范围。

**验证：** 先读最高优先级历史块；区分继承与显式零；封存期间没有读空窗；新写入不污染其他 VM。详见[磁盘方案比较](../03-design-comparison/02-writable-upper-and-snapshot.md)。

## 7. 恢复运行内存：本地懒恢复是否足够？

快照文件已完整在本地时，内核按需装入页面就能降低初始驻留。但跨节点或冷缓存时，文件的完整下载仍可能最慢。要单独测网络准备与恢复后的缺页时间。

[Firecracker 外部 handler 文档](https://github.com/firecracker-microvm/firecracker/blob/7699746649826d1dfcdde626b3131bac08f28e0d/docs/snapshotting/handling-page-faults-on-snapshot-resume.md)展示 UFFD FD/layout 交接和 COPY；[Cloud Hypervisor v53](https://github.com/cloud-hypervisor/cloud-hypervisor/releases/tag/v53.0)提供 snapshot/restore offload 和按需/后台恢复；[E2B faultPage](https://github.com/e2b-dev/infra/blob/cc7c574233ad98665a7c72a3d37b0af89ae79a71/packages/orchestrator/pkg/sandbox/uffd/userfaultfd/userfaultfd.go)体现页来源、COPY、重试和失败回调。[SRC-FC-002] [SRC-CH-001] [SRC-E2B-003]

建议先复用上游恢复和增量索引，再评估把远端不可变快照范围交给 lazyd 通用内容核心。**可以共用数据服务，但不以重构 StratoVirt 的统一 UFFD/迁移框架为前置，也不能用镜像 ready bitmap 代替某台 VM 的 RAM 驻留/脏页状态。**

Conch 定位一致快照与父层，数据源的内存适配器复用索引解析逻辑页来源；StratoVirt 保留 RAM/设备恢复和 dirty tracking，handler 按既有恢复契约请求与完成填页。COPY、memfd 写页或私有文件映射按上游实际 backend 选择，不因 pmem 选型而重写 RAM 恢复。恢复后业务写入不能被后台重复填页覆盖。

**验证：** 跨多层页查询、显式零、重复 fault、取消、脏页/WP/REMOVE、二次 checkpoint 和私有写入。详见[内存恢复](../03-design-comparison/03-memory-snapshot-restore.md)。

## 8. 删除或失败：怎样避免一台 VM 拖坏其他 VM？

建议由 Conch 管一次启动/恢复的最终状态，数据源与 VMM 提供 readiness、进度、错误和释放接口。具体状态/API 名称需要协商，不能只以 socket 已连接作为“能运行”。

[E2B 的多来源恢复](https://github.com/e2b-dev/infra/blob/cc7c574233ad98665a7c72a3d37b0af89ae79a71/docs/ARCHITECTURE.md)提供编排参考；[Conch Sandbox store](https://gitcode.com/openeuler/Conch/blob/8248022005542407f53b606a8be979379a3fd38b/internal/adapters/containerd/sandbox/store.go)提供持久引用接入点。它们不自动解决外部缓存的租约。[SRC-E2B-001] [SRC-CONCH-001]

准备失败时取消兄弟任务并释放本次 attachment；运行中不可恢复错误终止受影响 VM 并报告原因；删除只释放当前使用关系。共享服务要考虑错误隔离，不能一次超时终止所有租户。

回收需覆盖 Template、快照父层、运行映射和 in-flight 操作。打开 FD 可能延长 inode 生存期，但不等于路径重开、bitmap、重启恢复和远端依赖都受保护。

**验证：** 两 VM 共用内容、删除其中一个、服务重启、并发 GC、过期 attachment、快照发布失败。见[完整保存与恢复流程](06-checkpoint-three-path-restore.md)。

## 现在可以决定什么？

可以确定联合流程、VMM 最小职责和正确性门槛；handler 位置、磁盘组合、协议/进程组织、预取策略仍按[实施路线](07-phased-roadmap.md)进行有顺序的验证。每个选择都记录接受条件和否决条件，首版只实现选定路径，再冻结其必要跨仓契约，不把所有候选扩展一并加入 StratoVirt。
