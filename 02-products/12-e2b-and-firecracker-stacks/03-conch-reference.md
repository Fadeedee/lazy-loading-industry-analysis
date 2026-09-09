# E2B：从完整沙箱恢复反推数据服务

> 阅读后能说明对方解决了什么、哪些经验可用于三仓、选择还需要什么证据。以下借鉴均是建议，不是已冻结接口。

## 对方具体做了什么

沙箱恢复组合内存 UFFD 与 host NBD/COW rootfs。块 overlay 读取按 writable、可选 sealing、base 的顺序，新写入进入当前私有 cache；RAM 则由独立 PageReader/fault 流程供应。

第一方参考：[对应文档或源码](https://github.com/e2b-dev/infra/blob/cc7c574233ad98665a7c72a3d37b0af89ae79a71/docs/ARCHITECTURE.md)。[SRC-E2B-001] [SRC-E2B-002] [SRC-E2B-003] 核查日期与成熟度沿用[来源清单](../../appendix/source-inventory.md)，不是本次重新运行产品。

完整解释：[读取路径](01-data-path.md)、[缓存与快照生命周期](02-cache-snapshot-lifecycle.md)。

## 对数据服务的具体启发

E2B 的两条数据路径说明，可以共享“读取确定来源”的能力，但不应将磁盘 COW 与 RAM 页状态合并。我们的 lazyd 内容核心负责不可变字节和共享任务；磁盘视图解析、私有写层与 RAM 填页/写保护各有其归属，Conch 从同一 checkpoint 组织它们。

本次另外核查了固定版本的 [NBDProvider](https://github.com/e2b-dev/infra/blob/cc7c574233ad98665a7c72a3d37b0af89ae79a71/packages/orchestrator/pkg/sandbox/rootfs/nbd.go)：Start/Path 用结果同步设备就绪；Close 依次尝试 flush、mount 关闭和 overlay 关闭，并汇总错误。`finishedOperations` 通知发生在 overlay 关闭之前，不能当作所有清理成功的证明；导出的私有 cache 由接收方负责关闭。[SRC-E2B-004]

我们据此需要明确“准备就绪”“禁止新请求”“映射已不再使用”“允许回收内容”的区别。但 E2B 的私有写缓存不是我们的共享 EROFS cache，不能照搬 Close 就认为安全。单 VM 关闭只撤销本会话的等待和引用；其他 VM 仍使用的公共任务与已映射文件必须保留。具体代码和核查限制见[生命周期详解](02-cache-snapshot-lifecycle.md)。

## 三仓怎样共同借鉴

| 项目 | 可考虑的改动 |
| --- | --- |
| Conch | 从一致资源图组织 prepare、create/restore 和关闭；登记各来源、私有写层、后台任务及释放条件，而不只管理一个 FETCH socket。 |
| StratoVirt | 复用设备与 RAM 恢复能力，补必要的就绪、完成和致命失败通知；不承担模板索引、预取或分布式控制面。 |
| lazyd | 将内容/来源/会话/任务分开；后续以类型适配接磁盘历史与 RAM，维护可取消等待和保护中的缓存，不将私有写缓存生命周期套给共享对象。 |

## 选择与差异

当前保持 EROFS + pmem/DAX 只读镜像主线，私有写层和快照来源分阶段接入。E2B 的整盘 NBD/COW 作为对照，不要求首版重建 NBD 引擎或分布式控制面；源码中的 onFailure 也不替代本项目的终止与清理策略。

## 如何验证

先验证 prepare 就绪、FETCH 中关闭、单 VM 取消、缓存容量不足和关闭错误均有明确状态。后续再验证二次快照、封存中读取、父层恢复与私有写入隔离；所有阶段分别记录节点冷缓存下的业务可用延迟，而不是只记 VM resume 时间。

参见[数据服务横向比较](../../03-design-comparison/08-data-service-architecture.md)与[checkpoint 三路恢复](../../04-conch-design-reference/06-checkpoint-three-path-restore.md)。
