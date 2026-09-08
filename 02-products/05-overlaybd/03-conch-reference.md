# OverlayBD：把快照作为分层块视图

> 阅读后能说明对方解决了什么、哪些经验可用于三仓、选择还需要什么证据。以下借鉴均是建议，不是已冻结接口。

## 对方具体做了什么

逻辑块从最新层查到父层，远端缺失块按需取回。只读历史层上方有可写顶层，commit 或 live snapshot 将变化保存/切换，使新写入进入后续私有状态。

第一方参考：[对应文档或源码](https://github.com/containerd/overlaybd/blob/f63addfd8e51bd9eeecc12f5f4670c281a74d6e8/README.md)。[SRC-OVERLAYBD-001] 核查日期与成熟度沿用[来源清单](../../appendix/source-inventory.md)，不是本次重新运行产品。

完整解释：[读取路径](01-data-path.md)、[缓存与快照生命周期](02-cache-snapshot-lifecycle.md)。

## 三仓怎样共同借鉴

| 项目 | 可考虑的改动 |
| --- | --- |
| Conch | 管理磁盘谱系、设备布局和 checkpoint 时间点，选择混合镜像/写层还是统一整盘。 |
| StratoVirt | 提供标准块设备与状态保存，协同排空 I/O；不必解析镜像 tag。 |
| lazyd | 可供应不可变块层，也可连接专用块后端；共用取数核心不意味着由它承担所有写入实现。 |

## 选择与差异

整盘块方案是正式候选，不只限于 pmem 上方的 upper。要比较共享内存收益与减少设备/快照复杂度的收益，TCMU/NBD/其他后端另行选型。

## 如何验证

父层优先级、显式零/继承、封存切换、磁盘与 RAM 一致性、双 VM 写入隔离。

回到[联合设计决策](../../04-conch-design-reference/08-design-decisions-and-evidence.md)，比较该经验与其他候选，而不是单独据此定方案。
