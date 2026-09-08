# eStargz：把启动工作集提前准备

> 阅读后能说明对方解决了什么、哪些经验可用于三仓、选择还需要什么证据。以下借鉴均是建议，不是已冻结接口。

## 对方具体做了什么

eStargz 的 TOC/chunk 支持按文件范围读取；prioritized files 放在 landmark 前，文档描述在容器运行前预取这段数据。它把构建时的文件顺序与运行时准备结合，而不是只靠在线扩大窗口。

第一方参考：[对应文档或源码](https://github.com/containerd/stargz-snapshotter/blob/c2bf18e5a94dcfd959cabf744f4bbb4ef8d980a2/docs/estargz.md)。[SRC-STARGZ-002] 核查日期与成熟度沿用[来源清单](../../appendix/source-inventory.md)，不是本次重新运行产品。

完整解释：[读取路径](01-data-path.md)、[缓存与快照生命周期](02-cache-snapshot-lifecycle.md)。

## 三仓怎样共同借鉴

| 项目 | 可考虑的改动 |
| --- | --- |
| Conch | 构建、转换和 Template prepare 都可携带版本化工作集；决定是否等待预取并计入启动时间。 |
| StratoVirt | 若走 DAX，访问只看到地址，需要额外索引/trace 才能还原文件工作集；也可比较文件服务路线。 |
| lazyd | 执行有界预取、缓存校验和前台去重，按可用信息选择文件或范围策略。 |

## 选择与差异

可以借鉴工作集方法而不采用 eStargz 格式；若普通 OCI 兼容成本更重要，也可把该文件路线纳入候选。不要在比较前排除 FUSE/virtiofs。

## 如何验证

无预取、工作集预取、在线预测的业务首请求和下载放大；工作集版本失配应被拒绝。

回到[联合设计决策](../../04-conch-design-reference/08-design-decisions-and-evidence.md)，比较该经验与其他候选，而不是单独据此定方案。
