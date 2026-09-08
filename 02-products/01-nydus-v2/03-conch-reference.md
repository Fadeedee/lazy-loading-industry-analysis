# Nydus：内容服务与外部缺页处理

> 阅读后能说明对方解决了什么、哪些经验可用于三仓、选择还需要什么证据。以下借鉴均是建议，不是已冻结接口。

## 对方具体做了什么

服务端接收 UFFD FD 和 region metadata，直接处理内核缺页。它把逻辑镜像范围定位到后端缓存，可复制填页或返回文件映射区域。prefault 异步枚举已缓存内容，不等于远端预测下载；部分错误分支仅记录 warning。

第一方参考：[对应文档或源码](https://github.com/dragonflyoss/nydus/blob/8aa80aee6e77a0c4d529581fc6339e9ff3066736/service/src/block_uffd.rs)。[SRC-NYDUS-005] [SRC-NYDUS-006] [SRC-NYDUS-007] 核查日期与成熟度沿用[来源清单](../../appendix/source-inventory.md)，不是本次重新运行产品。

完整解释：[读取路径](01-data-path.md)、[缓存与快照生命周期](02-cache-snapshot-lifecycle.md)。

## 三仓怎样共同借鉴

| 项目 | 可考虑的改动 |
| --- | --- |
| Conch | 管理 source/attachment、授权和 handler 就绪/失败，不只传镜像名。 |
| StratoVirt | 比较复用外部 UFFD 与内部 handler；文件 remap 仍由 VMM 校验并执行。 |
| lazyd | 可把 UFFD 适配放在内容核心之外，复用取数、缓存与校验；不必固定为只接范围请求。 |

## 选择与差异

优先验证外部 handler，但不直接复制 RAFS 布局。整 blob、chunk 或组合设备映射都按实际镜像需求选择。服务端已合入不代表 VMM 端的任何接法都已可用。

## 如何验证

双 VM 同页、完整范围映射、相邻缺页、只读保护、连接断开、重复事件和 PSS。

回到[联合设计决策](../../04-conch-design-reference/08-design-decisions-and-evidence.md)，比较该经验与其他候选，而不是单独据此定方案。
