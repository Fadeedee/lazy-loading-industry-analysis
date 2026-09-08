# CRIU：恢复任务需要有明确的供页生存期

> 阅读后能说明对方解决了什么、哪些经验可用于三仓、选择还需要什么证据。以下借鉴均是建议，不是已冻结接口。

## 对方具体做了什么

CRIU lazy-pages 与 page server 面向 Linux 进程地址空间，支持把部分页面延迟到访问时供应。需要可用的页来源，供页阶段与恢复后执行阶段可能重叠。

第一方参考：[对应文档或源码](https://github.com/checkpoint-restore/criu/blob/criu-dev/Documentation/criu.txt)。[SRC-CRIU-001] 核查日期与成熟度沿用[来源清单](../../appendix/source-inventory.md)，不是本次重新运行产品。

完整解释：[读取路径](01-data-path.md)、[缓存与快照生命周期](02-cache-snapshot-lifecycle.md)。

## 三仓怎样共同借鉴

| 项目 | 可考虑的改动 |
| --- | --- |
| Conch | 保护恢复依赖并管理取消/失败，不只记录一个快照 API 调用成功。 |
| StratoVirt | 区分 host VMM HVA 和 guest 进程地址空间；不把 CRIU 本身当作 VM RAM backend。 |
| lazyd | 可供应只读快照来源或连接 page service，共用基础设施时保留类型语义。 |

## 选择与差异

借鉴供页生命周期和后台协调，不复制 CRIU 进程恢复格式。没有必要预设内存与镜像必须是两套传输，只需正确区分状态和布局。

## 如何验证

源提前回收、进程/VM 取消、断网、背景任务与需求读取协调。

回到[联合设计决策](../../04-conch-design-reference/08-design-decisions-and-evidence.md)，比较该经验与其他候选，而不是单独据此定方案。
