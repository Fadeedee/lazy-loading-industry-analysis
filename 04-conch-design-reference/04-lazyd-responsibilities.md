# lazyd：从按需取数闭环到可长期运行的数据服务

> 读完能解释：两台 VM 怎样共用一次下载，如何隔离取消与凭据，缓存什么时候可映射、什么时候能回收，以及先改哪些代码。

**当前主线是为 EROFS + pmem/DAX 提供数据。** lazyd 不只是 FETCH 接口包装，但也不能把“已有下载和 bitmap”当作产品化完成。先完善镜像数据服务，再按需扩展快照来源；不以实现全部适配器为首版前置。

本文在 2026-09-09 静态复核固定样本 `751d647fb37f`，未运行产品测试。下文“现状”来自该版本，“建议”是待实现、待验收的目标，不是已完成的性能优化。[SRC-LAZYD-001]

## 现有基础与具体缺口

已有 descriptor prepare、OCI Bearer auth/Range、digest 级 sparse cache、bitmap、去重、重启恢复和 SCM_RIGHTS 数据面。这些提供可复用的算法、行为与测试，不表示 Instance、接口或执行模型应原封不动。先明确下文的目标职责，再决定保留、重组或替换具体代码，不全盘重写，也不逐层包住旧结构。

第一次阅读可以先跳到“目标架构与所有权”和“两台 VM 同时读取会怎样”，代码证据按需展开。

<details>
<summary>现有代码具体有哪些改进点</summary>

| 现状与代码证据 | 需要改进什么 |
| --- | --- |
| [Instance / register_inner](https://github.com/Fadeedee/lazyd/blob/751d647fb37fa2e01f6fb1784003e8b26aa00244/src/instance.rs#L33)把内容、来源、凭据和执行状态集中在一起；重复注册会重新打开并替换 Instance | 内容对象保持稳定；凭据更新、VM 接入与退出不重建共享缓存/任务状态；并发重注册的影响需要回归验证 |
| [reserve_inflight](https://github.com/Fadeedee/lazyd/blob/751d647fb37fa2e01f6fb1784003e8b26aa00244/src/instance.rs#L479)在单个 Instance 中用范围列表去重，每 5ms 轮询重叠任务 | 按内容共享任务及完成结果；等待可取消，不把所有重复请求变成轮询者 |
| [handle_stream_once](https://github.com/Fadeedee/lazyd/blob/751d647fb37fa2e01f6fb1784003e8b26aa00244/src/data.rs#L94)在异步连接任务中直接阻塞接收；ensure_range 同步写盘和 sync | socket 等待与阻塞磁盘工作分开处理；连接、队列、下载内存和工作线程都有上限 |
| [ensure_range](https://github.com/Fadeedee/lazyd/blob/751d647fb37fa2e01f6fb1784003e8b26aa00244/src/instance.rs#L381)已执行 cache sync 后标 ready；[prepare_fetch_range](https://github.com/Fadeedee/lazyd/blob/751d647fb37fa2e01f6fb1784003e8b26aa00244/src/instance.rs#L413)返回读写 target 的 try_clone | 保留持久化顺序，独立导出只读句柄；共享文件的运行引用与回收保护需要闭环 |
| [OCI read_range](https://github.com/Fadeedee/lazyd/blob/751d647fb37fa2e01f6fb1784003e8b26aa00244/src/remote/oci.rs#L182)检查响应状态和长度，完整读取响应体后才检查上限；该路径未核对 Content-Range 或分块摘要 | 严格验证返回位置、总大小并在读取中限额；另行定义局部内容校验的可信依据 |
| FetchConfig.unit_bytes 同时用于 bitmap 与取数放大 | 稳定的缓存记录粒度与可调整的下载窗口分离，不为调优重写已有 bitmap header |

这些是静态审计发现的边界和风险，不代表已经测出吞吐下降或复现了全部失败场景。

</details>

## 目标架构与所有权

**一次需求的主线：运行会话授权 → pmem 适配器定位内容 → 内容核心命中缓存或等待共享任务 → 提交可读范围 → 适配器完成本次访问。** [三仓协作图](assets/target-responsibilities.html)表示外部分工，下表展开 lazyd 内部；不是新增五个进程或五个 crate 的要求。

| 逻辑部分 | 唯一管理的状态 | 与其他部分的关系 |
| --- | --- | --- |
| 内容核心 | 确定内容/格式/大小、安全域内的缓存对象、ready 与在途任务表 | 相同授权内容共用一个对象；不保存某台 VM 的脏页 |
| 数据源访问 | 远端位置、凭据引用、连接与鉴权状态 | 提供带取消/deadline 的范围读取；更新来源不替换内容对象，不借共享泄漏其他会话凭据 |
| 运行会话 | attachment，即本次 VM 的使用关系；授权、运行代次、取消、引用保护 | Conch 创建和释放；会话可以引用多个内容对象，断开不等于删除内容 |
| pmem 适配器 | 区域布局、合法偏移、事件与完成操作的关联 | 把缺页/范围请求转成内容请求；不执行另一进程地址空间的普通 mmap |
| 有界调度 | 排队、在途字节、并发和前台/后台优先级 | 执行内容任务；可以先做内部组件，不引入独立调度服务 |

**同一节点和允许共享的安全域内，同一 blob 的共享对象必须跨重复 prepare 存活。** 会话的工作负载标签、凭据版本或取消状态不能改变内容身份；格式/大小冲突仍需拒绝。跨租户复用须先满足授权和安全域策略，不因 digest 相同就直接放行；首版不要求跨节点共享在途任务。

Conch 管持久资源引用及沙箱最终状态；lazyd 管本地缓存与取数；StratoVirt 管设备、地址空间和页完成。下文的状态名只用于说明，不提前冻结新的跨仓 schema。

## 两台 VM 同时读取会怎样

1. Conch 为 VM1、VM2 分别建立 attachment，引用同一确定内容，并登记权限、预算和运行代次。
2. 适配器把需求定位到内容偏移，校验范围；命中 ready 的请求取得受保护的只读文件范围，不进入远端队列。
3. 缺失范围按稳定单元划分，在内容级任务表中原子查询/登记。重叠部分加入已有任务，仅为剩余缺失单元创建任务；排队后再检查 ready，避免竞争导致重复下载。
4. 同一任务的调用者通过完成通知等待；锁只保护状态，不跨网络 await 或磁盘同步持有。前台命中预取任务时复用并提升优先级。
5. 数据源按授权读取，调度器限制并发和字节；验证响应、写入缺失范围并按下节顺序提交。不会为了补一个洞重写相邻已映射的 ready 区域。
6. 结果发布给所有仍有效的等待者；完成消息携带所属会话/区域代次，过期结果不能映射进另一台 VM 或重建后的区域。
7. 文件映射方案由各自 StratoVirt 校验并映射同一缓存 inode 的相应页、完成等待。下载可共享，VM 的映射与缺页状态仍各自独立。

VM1 取消只撤销它的等待和使用关系。VM2 仍在等待时继续公共任务；没有等待者后，按明确策略取消或在后台预算内完成，不能无限保留任务。失败要通知全部等待者、释放占用并允许有界重试，不能丢通知或静默挂起。

## 缓存状态与持久化

以下是状态与正确性要求，不指定旧 sparse/bitmap 布局。单文件 ready 索引与不可变 chunk 文件按同一标准比较，见[选型计划 S1/S2](09-core-design-selection.md)；保留或替换都需要依据。

| 状态 | 可以对消费者承诺什么 |
| --- | --- |
| 缺失 | 没有可用数据；文件洞不代表有效零 |
| 排队/取数中 | 有任务，不代表可映射 |
| 完整写入、检查通过、待持久化 | 数据尚未完成持久发布；首版不提前向 VMM 宣告 ready |
| ready 已提交 | 数据和必要的 ready 元数据均已持久化，可在有效引用保护下导出 |
| 失败/隔离 | 不发布 ready；报告错误，按策略重试或重建未被使用的缓存 |

持久发布必须保证：**完整写入及所需检查 → 数据持久化 → 可恢复的 ready 记录 → 通知完成。** bitmap 同步只是 S1 的一种实现；不可变文件还涉及发布与目录持久化。逐任务同步和有界分组提交都参与首版选型，必须知道哪些范围由同一屏障保护，不能先通知 ready 再补 sync。

长度和 Content-Range 检查不等于密码学完整性校验。整 blob digest 不能直接证明任意小范围正确；若要求逐块强校验，需要受信的分块摘要/索引，否则明确依赖的传输、存储信任边界，不声称已验证。SEEK_DATA/HOLE 只用于发现明显缺口，不证明内容正确。

clear ready 也需明确持久化和错误语义。范围若已被 VMM 映射，清 bitmap 不会撤销映射；发现损坏必须隔离对象并报告受影响会话，不能原地重写或截断仍在使用的文件。进程重启先恢复可信缓存状态并协调使用关系，再允许回收。

## 有界 I/O 与请求调度

先解决可靠性和资源上限，再增加预测策略：

- socket 使用非阻塞就绪等待或有界专用 worker；不能让阻塞 recv 占住异步执行线程，也不能无限增加线程/连接补救。
- 同步文件写入和 sync 放进受限的阻塞执行资源；下载缓冲、排队数、在途字节和任务数分别受限。
- 对 OCI Range 核对响应起止位置、总大小和实际长度；不接受与请求不匹配的 200 全量响应，不等完整读入响应体才执行大小限制。
- 排队、远端请求和完成等待都有 deadline/取消。仅可重试的错误按有限预算退避；格式、权限、内容校验失败不能无限重试。
- 当前缺页优先于后台预取，同时限制单会话占用，避免大 VM 挤死其他 VM。已经发出的请求未必能立即抢占，因此还要限制后台单次范围和并发。

固定 bitmap 单元、网络下载窗口、返回映射范围分别定义。后续动态窗口只覆盖确认缺失的单元，预取只在预算内扩展；Conch 提供场景/节点预算，lazyd 利用已有需求事件和指标执行。远端工作集统计不是上述基础改造的前置，详见[预取策略](../03-design-comparison/06-cache-dedup-and-prefetch.md)。

## 会话权限与回收

会话经历准备、可服务、关闭中、已释放；准备失败不发布就绪，运行失败进入关闭处理。关闭中拒绝新请求并撤销本会话等待，只有映射/句柄使用结束得到确认后才释放相应保护。会话状态与内容 ready 分别管理，不把某次 VM 退出解释为内容失效。

内部写缓存句柄与外发只读句柄分开打开，并核对指向同一文件；dup/try_clone 不会自动降低写权限。VMM 仍需校验映射权限和 guest 写保护。文件 FD 通常授予对该文件的访问，不是 JSON 中某个 range 的权限令牌，授权必须匹配导出对象的粒度。

导出范围前建立运行引用保护，直到映射/句柄不再被使用才释放。会话失效可拒绝新请求，但单方面关 socket、关闭服务端 FD 或租约超时都不能撤销 VMM 已有映射。容量不足时施加背压或报错，不能截断仍受保护的共享文件。

Conch 释放当前 attachment；lazyd 回收前同时检查持久使用关系、映射保护、在途任务和重启协调结果。缓存淘汰与删除远端快照对象不是同一件事；后者仍由资源 owner 管理。lease/refcount 的具体实现可复用现有设施，但外部缓存保护不能只靠 containerd GC label。

## UFFD 与快照扩展放在哪里

pmem 的内部/外部 handler 尚未选定。若采用外部方案，事件读取、region lookup 与完成关联进入适配模块；内容核心仍只处理明确内容范围。若选内部 handler，则提供范围入口；产品只新增选定方案。文件 remap 必须由 StratoVirt 执行或严格控制，不为了把 handler 外置而扩大 VMM 的内容职责。缓存和任务模型也需先完成[核心选型](09-core-design-selection.md)，不默认原结构不变。

后续磁盘历史和 RAM 快照分别增加类型适配：解析确定快照视图与父层，向共同内容核心取不可变字节。磁盘当前写层、RAM 脏页、页驻留和晚到填充规则不塞进镜像 bitmap。优先复用上游恢复索引和专用 backend，不要求 lazyd 重写 VMM 恢复框架。

fanotify 不作为本轮 guest pmem/DAX 的触发入口；既有普通容器路径不得因内容核心重组而改变语义，也不默认新增其他文件路径适配器。

## 借鉴来源与我们的差异

| 参考与直接证据 | 对方具体做了什么 | 我们采用什么、没有照搬什么 |
| --- | --- | --- |
| [Nydus UFFD service](https://github.com/dragonflyoss/nydus/blob/8aa80aee6e77a0c4d529581fc6339e9ff3066736/service/src/block_uffd.rs)、[block device](https://github.com/dragonflyoss/nydus/blob/8aa80aee6e77a0c4d529581fc6339e9ff3066736/service/src/block_device.rs) [SRC-NYDUS-005] [SRC-NYDUS-007] | 接收 UFFD/布局，读 fault，并把逻辑块定位到 metadata/data/hole；缓存范围枚举与下载是不同操作 | 借鉴适配器与内容定位分工；不照搬协议、warning 错误策略或预设同样的进程模型 |
| [Nydus cache 校验](https://github.com/dragonflyoss/nydus/blob/8aa80aee6e77a0c4d529581fc6339e9ff3066736/storage/src/cache/mod.rs) [SRC-NYDUS-008] | 检查长度，并按格式/配置决定 chunk 校验；不同摘要模式能力不同 | 明确局部校验依据；不把 bitmap ready 或 CRC 说成已完成密码学校验 |
| [Nydus cache 状态接口](https://github.com/dragonflyoss/nydus/blob/8aa80aee6e77a0c4d529581fc6339e9ff3066736/storage/src/cache/state/mod.rs)、[BlobStateMap](https://github.com/dragonflyoss/nydus/blob/8aa80aee6e77a0c4d529581fc6339e9ff3066736/storage/src/cache/state/blob_state_map.rs) [SRC-NYDUS-009] [SRC-NYDUS-010] | ready map 外单独跟踪在途任务，用 Slot/Condvar 通知；清 pending 也可能唤醒等待者，唤醒后仍检查 ready | 借鉴任务与内容状态分离；适配我们的异步执行、每会话取消和公平性，不照搬同步等待，也不由 set_ready 推断持久化保证 |
| [E2B memory handler](https://github.com/e2b-dev/infra/blob/cc7c574233ad98665a7c72a3d37b0af89ae79a71/packages/orchestrator/pkg/sandbox/uffd/userfaultfd/userfaultfd.go) [SRC-E2B-003] | 通过 PageReader 取页，完成 COPY、有限重试及失败回调，并处理恢复状态 | 借鉴数据来源接口和可结束的失败路径；不把 RAM 的 COPY/WP 状态机套进只读 pmem cache |
| [E2B NBDProvider](https://github.com/e2b-dev/infra/blob/cc7c574233ad98665a7c72a3d37b0af89ae79a71/packages/orchestrator/pkg/sandbox/rootfs/nbd.go) [SRC-E2B-004] | Start/Path 协调设备就绪，Close 分步清理并汇总错误；导出的私有 cache 有接收方关闭责任 | 明确就绪、关闭、回收的不同条件；不将私有写 cache 的关闭规则直接用于跨 VM 共享文件 |

共享任务、凭据与会话分离、有界调度和缓存回收是结合代码与产品需求提出的设计，不声称参考项目已实现完全相同的架构。先读[数据服务横向比较](../03-design-comparison/08-data-service-architecture.md)了解推导过程，再按需查 [Nydus 缓存](../02-products/01-nydus-v2/02-cache-snapshot-lifecycle.md)与 [E2B 生命周期](../02-products/12-e2b-and-firecracker-stacks/02-cache-snapshot-lifecycle.md)。2026-09-09 补查了表中的 Nydus 状态接口/实现与 E2B NBDProvider，其他业界证据沿用已登记版本，没有重新核查所有产品。

## 开发顺序与验收

| 顺序 | 本阶段交付 | 必须观察的结果 |
| --- | --- | --- |
| 先完善镜像核心 | 稳定内容对象、会话/来源分离、共享任务、有界 I/O、只读导出、保护和恢复规则 | 并发 prepare/刷新凭据/读取同内容时不拆出重复任务；单会话取消不影响其他等待者；空闲连接不阻塞健康请求；队列满有明确处理 |
| 再完成 pmem 联调 | 按选型接 UFFD 适配器、区域代次、错误与释放 | 两 VM 内容对拍、同文件页共享、旧事件拒绝、handler 退出可结束等待、停止/重启/GC 无误删 |
| 后续性能优化 | 请求合并、动态窗口、工作集预取、可选批量 sync | 对比固定策略下的业务延迟、下载放大和资源占用；优化不破坏持久化与公平性 |
| 后续快照来源 | 不可变磁盘历史/RAM 来源适配，复用现有恢复能力 | 父层和显式零正确、新写入不被覆盖、跨节点重新绑定、二次 checkpoint 正确 |

基础测试覆盖错误 Content-Range、短/超长响应、超时、部分写入、sync 失败、只读 FD、重开和取消竞态；真实掉电与生产规模尚需专门环境，不能由单测推出。

记录缓存命中、共享任务合并次数、队列等待、远端耗时/字节、持久化耗时、在途字节、会话/FD 数量和回收保护原因；日志不记录凭据。性能目标先测基线再定阈值，不先承诺倍数。接入顺序归入[三仓联合路线](07-phased-roadmap.md)，不是先独立重写完 lazyd 再开始 Conch/StratoVirt。
