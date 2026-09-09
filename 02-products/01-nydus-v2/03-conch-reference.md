# Nydus：内容服务与外部缺页处理

> 阅读后能说明对方解决了什么、哪些经验可用于三仓、选择还需要什么证据。以下借鉴均是建议，不是已冻结接口。

## 对方具体做了什么

服务端接收 UFFD FD 和 region metadata，直接处理内核缺页。它把逻辑镜像范围定位到后端缓存，可复制填页或返回文件映射区域。prefault 异步枚举已缓存内容，不等于远端预测下载；部分错误分支仅记录 warning。

第一方参考：[对应文档或源码](https://github.com/dragonflyoss/nydus/blob/8aa80aee6e77a0c4d529581fc6339e9ff3066736/service/src/block_uffd.rs)。[SRC-NYDUS-005] [SRC-NYDUS-006] [SRC-NYDUS-007] 核查日期与成熟度沿用[来源清单](../../appendix/source-inventory.md)，不是本次重新运行产品。

完整解释：[读取路径](01-data-path.md)、[缓存与快照生命周期](02-cache-snapshot-lifecycle.md)。

本次另核对 [ChunkMap/RangeMap](https://github.com/dragonflyoss/nydus/blob/8aa80aee6e77a0c4d529581fc6339e9ff3066736/storage/src/cache/state/mod.rs)和 [BlobStateMap](https://github.com/dragonflyoss/nydus/blob/8aa80aee6e77a0c4d529581fc6339e9ff3066736/storage/src/cache/state/blob_state_map.rs)：底层记录 ready，适配层登记 pending 并通知等待者；等待结束后还要复查 ready，超时也不自动证明公共任务已清理。[SRC-NYDUS-009] [SRC-NYDUS-010]

## 三仓怎样共同借鉴

| 项目 | 可考虑的改动 |
| --- | --- |
| Conch | 分开登记共享内容与本次 attachment，传递授权、运行代次和失败回调；重复准备不重建其他 VM 正在用的对象。 |
| StratoVirt | 内部/外部 handler 按共同标准选型，首版只实现选定路径；文件 remap 仍由 VMM 校验并执行。 |
| lazyd | 把内容 ready、共享取数任务和会话等待者分开；实现原子领取缺失范围、完成通知和失败清理；UFFD 适配按所选路径接入，来源更新不替换公共任务。 |

## 对数据服务的具体启发

已有 bitmap 只能回答“数据是否可读”，不能独自解决“谁正在下载、谁还在等待”。lazyd 需要在内容级维护任务，让重复/部分重叠请求复用进度；一个会话取消只撤销自己的等待。

借鉴状态分层而不是复制同步 Condvar：异步服务需要非阻塞等待、有界执行和 deadline。Nydus 的这两个状态文件不足以证明我们的租户隔离、掉电提交或 GC 已解决，仍须单独设计。

## 选择与差异

不因 Nydus 使用外部 handler 就优先选择它，也不复制 RAFS 布局或同名协议。按[核心选型计划](../../04-conch-design-reference/09-core-design-selection.md)比较原生 EROFS 的缓存、提交、任务和 handler；自有 lazyd 实现只是资产。服务端已合入不代表 VMM 端任意接法都可用。

## 如何验证

双 VM 重叠读取、任务失败和单等待者取消、重复 prepare/凭据刷新、完整范围映射、相邻缺页、只读保护和 PSS；检查远端请求计数与等待者是否全部结束，不能只看缓存最终存在。

汇总见[数据服务问题与证据](../../03-design-comparison/08-data-service-architecture.md)，落地边界见 [lazyd 架构](../../04-conch-design-reference/04-lazyd-responsibilities.md)。
