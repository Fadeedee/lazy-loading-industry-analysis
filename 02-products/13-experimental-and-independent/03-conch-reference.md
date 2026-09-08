# 独立项目与反例：让设计允许被实验否定

> 阅读后能说明对方解决了什么、哪些经验可用于三仓、选择还需要什么证据。以下借鉴均是建议，不是已冻结接口。

## 对方具体做了什么

urunc 的共享 snapshot view 原型讨论 lease、fallback 及小镜像额外准备成本。CubeSandbox 的 EROFS 请求不等于已实现路径；OpenSandbox 的 snapshot API 也不能证明底层有按需供数。

第一方参考：[对应文档或源码](https://github.com/urunc-dev/urunc/discussions/523)。[SRC-URUNC-001] [SRC-CUBE-001] [SRC-OPENSANDBOX-001] 核查日期与成熟度沿用[来源清单](../../appendix/source-inventory.md)，不是本次重新运行产品。

完整解释：[读取路径](01-data-path.md)、[缓存与快照生命周期](02-cache-snapshot-lifecycle.md)。

## 三仓怎样共同借鉴

| 项目 | 可考虑的改动 |
| --- | --- |
| Conch | 暴露真实准备/运行/失败状态，允许明确配置完整本地策略，按场景选择共享是否值得。 |
| StratoVirt | 保证普通设备/本地恢复回归，不让按需能力成为所有场景的强制依赖。 |
| lazyd | 可按测量选择小对象完整获取，保持成本可观察，不把额外缓存层当作天然收益。 |

## 选择与差异

个人原型的价值在可复现机制和反例，不在项目知名度。缺少源码定位时明确证据边界，不从名称、API 或宣传补出实现。

## 如何验证

小镜像、单 VM、冷缓存、无复用情况下的额外开销；公开 API 与真实数据路径分别验证。

回到[联合设计决策](../../04-conch-design-reference/08-design-decisions-and-evidence.md)，比较该经验与其他候选，而不是单独据此定方案。
