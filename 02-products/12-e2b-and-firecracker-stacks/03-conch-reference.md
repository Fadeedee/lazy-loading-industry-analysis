# E2B：从完整沙箱恢复反推数据服务

> 阅读后能说明对方解决了什么、哪些经验可用于三仓、选择还需要什么证据。以下借鉴均是建议，不是已冻结接口。

## 对方具体做了什么

沙箱恢复组合内存 UFFD 与 host NBD/COW rootfs。块 overlay 读取按 writable、可选 sealing、base 的顺序，新写入进入当前私有 cache；RAM 则由独立 PageReader/fault 流程供应。

第一方参考：[对应文档或源码](https://github.com/e2b-dev/infra/blob/cc7c574233ad98665a7c72a3d37b0af89ae79a71/docs/ARCHITECTURE.md)。[SRC-E2B-001] [SRC-E2B-002] [SRC-E2B-003] 核查日期与成熟度沿用[来源清单](../../appendix/source-inventory.md)，不是本次重新运行产品。

完整解释：[读取路径](01-data-path.md)、[缓存与快照生命周期](02-cache-snapshot-lifecycle.md)。

## 三仓怎样共同借鉴

| 项目 | 可考虑的改动 |
| --- | --- |
| Conch | 从 Template/checkpoint 一致资源图组织恢复、私有写层、后台任务和回收，用户 API 不被单一数据面主导。 |
| StratoVirt | 同时接入设备和内存恢复，管理运行时访问及失败；不要求与 Firecracker 完全相同的接口。 |
| lazyd | 可共用磁盘历史块/内存快照的内容核心，类型适配维护不同逻辑视图和完成语义。 |

## 选择与差异

把整盘块快照当正式候选，与 pmem 镜像+写层组合对照。E2B 是机制和编排样本，不意味着要重建它的分布式控制面；源码中的 onFailure 不替代本项目终止策略。

## 如何验证

二次快照、封存中读取、父层恢复、私有写入、节点冷缓存和整体业务延迟。

回到[联合设计决策](../../04-conch-design-reference/08-design-decisions-and-evidence.md)，比较该经验与其他候选，而不是单独据此定方案。
