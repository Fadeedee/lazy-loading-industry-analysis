# 独立项目与反例：让设计允许被实验否定

> 阅读后能说明对方解决了什么、哪些经验可用于三仓、选择还需要什么证据。以下借鉴均是建议，不是已冻结接口。

## 对方具体做了什么

urunc 的共享 snapshot view 原型讨论 lease、fallback 及小镜像额外准备成本。CubeSandbox 的 EROFS 请求不等于已实现路径；OpenSandbox 的 snapshot API 也不能证明底层有按需供数。

第一方参考：[对应文档或源码](https://github.com/urunc-dev/urunc/discussions/523)。[SRC-URUNC-001] [SRC-CUBE-001] [SRC-OPENSANDBOX-001] 核查日期与成熟度沿用[来源清单](../../appendix/source-inventory.md)，不是本次重新运行产品。

完整解释：[读取路径](01-data-path.md)、[缓存与快照生命周期](02-cache-snapshot-lifecycle.md)。

## 对数据服务的具体启发

urunc 的原型讨论提醒我们，懒加载服务本身也有固定成本。lazyd 的收益评估需要拆出 prepare/索引、连接和 FD、缓存提交同步、队列等待与后台任务成本；在小镜像、单 VM、无复用时，这些成本可能超过省下的下载时间。先测基线，再决定小对象是否完整准备，不能仅增加更多 adapter 就称为优化。

这三个样本提供的证据不同：urunc 可用于讨论共享视图和成本反例；CubeSandbox 的请求用于识别需求；OpenSandbox 的 API 用于看用户接口。后两者都不能用来证明已有数据服务实现，urunc 的 lease 讨论也不能替代我们的内容授权、持久化和任务取消设计。下面是我们的验证要求，不是已测出的产品结论。

## 三仓怎样共同借鉴

| 项目 | 可考虑的改动 |
| --- | --- |
| Conch | 暴露真实准备/运行/失败状态与明确的 full/lazy 策略；小对象或特定环境选择全量时记录原因，不隐式改变恢复语义。 |
| StratoVirt | 回归普通 pmem、块设备和本地恢复；按需能力不成为所有场景的强制依赖，不加入业务大小阈值。 |
| lazyd | 提供取数、提交、排队、复用和资源占用指标；在预算内执行所选策略，共享内容核心不因 full/lazy 分支复制成两套。 |

## 选择与差异

个人原型的价值在可复现机制和反例，不在项目知名度。缺少源码定位时明确证据边界，不从名称、API 或宣传补出实现。

## 如何验证

比较小镜像/大镜像、单 VM/并发 VM、冷/热缓存与不同网络条件，分开记录准备时间、首个业务请求、下载字节、sync 耗时及 FD/内存峰值。验证满队列、关闭和完整准备策略的错误路径；公开 API 与实际缺失读取链路分别验证。

参见[数据服务横向比较](../../03-design-comparison/08-data-service-architecture.md)与[分阶段验收路线](../../04-conch-design-reference/07-phased-roadmap.md)。
