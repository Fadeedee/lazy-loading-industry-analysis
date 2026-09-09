# 数据服务怎样设计：从一次读取到共享与回收

> 不先问现有 lazyd 能复用哪些接口，而是先问：谁在读取哪份数据，谁拥有任务，什么时候完成，失败与退出后谁还能继续使用。

本页是问题与证据索引，不替代 [lazyd 具体架构](../04-conch-design-reference/04-lazyd-responsibilities.md)。当前产品主线仍是 pmem 镜像；磁盘与 RAM 的材料用于验证边界及后续扩展，不要求同时实现全部数据来源。

## 先用同一组问题审查产品

| 问题 | 可参考的具体证据 | 对我们的约束 |
| --- | --- | --- |
| 读取位置属于什么对象 | Nydus 逻辑块到 blob/chunk；SOCI 文件位置与压缩 span；E2B 磁盘 overlay 与 RAM PageReader | 内容、逻辑视图、运行会话分别表达；一个 offset 不能同时代表三者 |
| 两个请求命中同一缺失内容 | Nydus ChunkMap/RangeMap 与 BlobStateMap 的 pending/ready 协调 | 需要共享任务及失败清理，而不只是一个“下载过”的 bitmap |
| 网络取完是否就能交付 | Nydus 条件化校验；掉电提交顺序需要另查实际写入/屏障 | 长度、局部校验、持久化、可映射分别承诺；不能由校验接口推定持久化 |
| 后台任务怎样与前台并存 | eStargz 工作集预取；QEMU 前台/后台页领取 | 调度预算可共用，但预测下载与恢复页安装状态不能混用 |
| 来源服务什么时候算就绪 | E2B NBDProvider 的 ready 结果；Firecracker 外部 handler 的失败警告 | 建立连接不等于可供数；Conch 必须知道来源失败，VMM 必须能结束等待 |
| 关闭后能否删除数据 | E2B 设备与私有 cache 的关闭顺序；Kata mount 所有权；独立项目的 lease 讨论 | 关闭当前会话不等于回收公共内容；VMM 已有 FD/映射仍要受保护 |

资料入口：[Nydus 状态协调](../02-products/01-nydus-v2/02-cache-snapshot-lifecycle.md)、[SOCI](../02-products/04-soci/01-data-path.md)、[eStargz](../02-products/03-stargz/01-data-path.md)、[QEMU](../02-products/09-qemu/01-data-path.md)、[E2B 关闭边界](../02-products/12-e2b-and-firecracker-stacks/02-cache-snapshot-lifecycle.md)。每项事实沿用各自来源版本，不把本表解释为所有产品都已实现同一套服务。

## 例一：bitmap 之外还需要任务状态

bitmap 可记录已就绪范围；在途任务还需要表达领取、等待、取消和完成结果。具体布局与提交策略见[核心选型](../04-conch-design-reference/09-core-design-selection.md)。

[Nydus 的状态接口](https://github.com/dragonflyoss/nydus/blob/8aa80aee6e77a0c4d529581fc6339e9ff3066736/storage/src/cache/state/mod.rs)区分 ready 查询、领取 pending、完成/清理和范围等待。[BlobStateMap 实现](https://github.com/dragonflyoss/nydus/blob/8aa80aee6e77a0c4d529581fc6339e9ff3066736/storage/src/cache/state/blob_state_map.rs)用独立的在途表和条件变量协调同一内容单元，等待后重新检查 ready；清 pending 可以结束等待，但不一定代表数据成功。[SRC-NYDUS-009] [SRC-NYDUS-010]

借鉴的是**缓存状态与并发任务状态分离**。它不证明完整的掉电提交顺序、租户授权、每个调用者可取消或公平调度已解决。我们的建议是内容级共享任务、异步完成结果、独立等待者与预算；不能把同步 Condvar 直接搬到异步执行线程，也不能把某个等待者超时当作删除公共任务的理由。

**验证：** 同一范围只发一次有效取数；部分重叠只补缺失单元；失败通知所有等待者且不标 ready；一个会话取消不影响另一个；重复 prepare/更新来源不拆出第二份任务状态。

## 例二：私有写缓存的关闭不等于共享缓存回收

[E2B NBDProvider](https://github.com/e2b-dev/infra/blob/cc7c574233ad98665a7c72a3d37b0af89ae79a71/packages/orchestrator/pkg/sandbox/rootfs/nbd.go)的 Start 发布设备路径或错误，Path 等待这个结果；Close 尝试 flush、关闭 mount、发送 finishedOperations，再关闭 overlay，并汇总错误。导出路径将摘下的私有 cache 交给调用方，明确后续关闭责任。[SRC-E2B-004]

这说明生命周期需要**就绪结果、进行中操作和资源所有权**，但不能由一个函数推出所有关闭竞态都安全。其私有 writable cache 也不是我们跨 VM 共享的只读 EROFS cache。我们的会话释放必须独立于内容回收，并验证运行映射、在途请求、重启和缓存容量不足时的行为；不能机械复制其 NBD 路径或信号顺序。

**验证：** 准备失败不发布可用、关闭与读取并发、一个 VM 退出、后台导出失败、来源重启以及超时后仍存活的映射。

## 需要补查的部分不能靠推断填满

| 主题 | 当前证据边界 | 实施前应补什么 |
| --- | --- | --- |
| 全局队列、公平性和限额 | 已有调度案例不等于核查了每个产品的节点级实现 | 选定参考后追到 worker/队列、字节上限和取消调用方；无证据时按本项目需求设计并测试 |
| 凭据与缓存共享 | 内容摘要和 registry auth 不证明跨租户访问已隔离 | 校验共享安全域、来源刷新、缓存命中授权和 FD 导出粒度 |
| 掉电与 GC | ready 接口、文件存在或 Close 返回均不足以证明重启安全 | 跟踪真正的写入/同步/提交点、运行引用保护及恢复协调；必要时故障注入 |
| 未核实的产品能力 | CRIU CLI、CH 发布说明、独立项目讨论的证据深度不同 | 不据此断言它们具备完整缓存服务；沿用来源日期并注明待核查 |

本次新增核对限于固定版本 Nydus state 接口/实现和 E2B provider，就这些符号形成静态证据；没有运行它们的测试，也没有刷新所有产品最新版本。

## 推导三仓设计，再决定复用代码

1. Conch 表达确定内容/视图、授权引用、运行会话及截止时间，不逐条调度下载。
2. lazyd 设计稳定内容对象、来源访问、任务状态和引用保护；接口与内部结构都可调整，不以旧 Instance/FETCH 为边界。
3. StratoVirt 只提供所选 pmem 接入、校验后的页完成及错误/关闭能力，不理解内容索引、凭据或缓存 GC。
4. 明确上述正常/失败流程后，再对照各仓现有代码分类为保留、重构、删除或新增。复用必须保持语义，不因为已经有 bitmap 就跳过并发和持久化设计。

优先完成 pmem 冷启动与多 VM 的可验证闭环，再扩展预取、磁盘历史与内存来源；对照 [联合路线](../04-conch-design-reference/07-phased-roadmap.md)和 [lazyd 验收](../04-conch-design-reference/04-lazyd-responsibilities.md#开发顺序与验收)执行。
