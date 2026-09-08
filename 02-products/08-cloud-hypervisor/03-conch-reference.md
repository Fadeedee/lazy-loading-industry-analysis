# Cloud Hypervisor：恢复服务与 VMM 如何分工

> 阅读后能说明对方解决了什么、哪些经验可用于三仓、选择还需要什么证据。以下借鉴均是建议，不是已冻结接口。

## 对方具体做了什么

v53 发布说明包含 demand-paged guest memory restore、snapshot/restore offload 和后台 prefault。另一个 PMEM 外部 UFFD PR #8239 关闭未合入，不能把两项能力混为一项。

第一方参考：[对应文档或源码](https://github.com/cloud-hypervisor/cloud-hypervisor/releases/tag/v53.0)。[SRC-CH-001] [SRC-CH-002] 核查日期与成熟度沿用[来源清单](../../appendix/source-inventory.md)，不是本次重新运行产品。

完整解释：[读取路径](01-data-path.md)、[缓存与快照生命周期](02-cache-snapshot-lifecycle.md)。

## 三仓怎样共同借鉴

| 项目 | 可考虑的改动 |
| --- | --- |
| Conch | 把恢复状态、后台完成与运行状态分别呈现，管理服务和快照引用。 |
| StratoVirt | 参考恢复基础机制与 source 交接，但不从发布说明推断具体 pmem 协议。 |
| lazyd | 可以承担公共取数/缓存，也可以只适配已有恢复服务，避免重复实现。 |

## 选择与差异

共享 UFFD 或传输框架可以评估，但 region kind、写权限和代次要明确。首版先复用 StratoVirt 已有恢复契约，不以统一 pmem/RAM 框架为前置。某 PR 未合入不是强制两套服务或拒绝统一协议的技术证明；也不是我们必须重构 VMM 的理由。

## 如何验证

后台恢复完成、前台等待、服务故障、版本协商，以及复用框架后的 pmem/RAM 回归。

回到[联合设计决策](../../04-conch-design-reference/08-design-decisions-and-evidence.md)，比较该经验与其他候选，而不是单独据此定方案。
