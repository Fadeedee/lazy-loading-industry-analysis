# QEMU：前台缺页和后台恢复怎样协调

> 阅读后能说明对方解决了什么、哪些经验可用于三仓、选择还需要什么证据。以下借鉴均是建议，不是已冻结接口。

## 对方具体做了什么

Fast Snapshot Load 按 mapped-ram 布局读取本地快照，fault thread 与 eager thread 使用页面状态协调领取。后台线程不只是预测，它还负责让恢复最终结束，避免未访问冷页令迁移状态长期悬挂。

第一方参考：[对应文档或源码](https://github.com/qemu/qemu/blob/35500e5c41aec76cde59befe750600dac7a9e37a/docs/devel/migration/fast-snapshot-load.rst)。[SRC-QEMU-001] 核查日期与成熟度沿用[来源清单](../../appendix/source-inventory.md)，不是本次重新运行产品。

完整解释：[读取路径](01-data-path.md)、[缓存与快照生命周期](02-cache-snapshot-lifecycle.md)。

## 三仓怎样共同借鉴

| 项目 | 可考虑的改动 |
| --- | --- |
| Conch | 区分可运行与恢复完成，按场景设置后台物化策略及跨数据源预算。 |
| StratoVirt | 确保重复填页不覆盖恢复后的写入，并让状态与停止路径一致。 |
| lazyd | 可以复用内容去重和队列，但 RAM 页领取/安装代次不等于镜像缓存 ready。 |

## 选择与差异

本地快照调度可借鉴，不宣称它已实现远端网络自适应。是否全量后台物化由产品目标决定，不能统一套到长期按需镜像。

## 如何验证

冷页从不访问、并发需求/后台页、业务新写入、取消和恢复完成判定。

回到[联合设计决策](../../04-conch-design-reference/08-design-decisions-and-evidence.md)，比较该经验与其他候选，而不是单独据此定方案。
