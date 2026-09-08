# SOCI：索引如何绑定不可变内容

> 阅读后能说明对方解决了什么、哪些经验可用于三仓、选择还需要什么证据。以下借鉴均是建议，不是已冻结接口。

## 对方具体做了什么

zTOC 保存 TAR 文件位置及压缩流检查点状态，让 span 可以独立解压。SOCI 的 v1 外置索引与 v2 构建期转换有区别，不能笼统描述为完全不改镜像。

第一方参考：[对应文档或源码](https://github.com/awslabs/soci-snapshotter/blob/238af848f32fcb887072c144b09ee65a3a895f9c/docs/glossary.md)。[SRC-SOCI-004] [SRC-SOCI-001] 核查日期与成熟度沿用[来源清单](../../appendix/source-inventory.md)，不是本次重新运行产品。

完整解释：[读取路径](01-data-path.md)、[缓存与快照生命周期](02-cache-snapshot-lifecycle.md)。

## 三仓怎样共同借鉴

| 项目 | 可考虑的改动 |
| --- | --- |
| Conch | 负责选择正确的镜像/派生索引，保护两者引用与授权，定义未准备完整时的状态。 |
| StratoVirt | 通过 source 能力使用数据，不依赖 registry tag 或压缩流细节。 |
| lazyd | 实现从逻辑范围到可解码内容的定位、缓存和去重，局部校验要有可信依据。 |

## 选择与差异

原生 EROFS 可能不需要 zTOC；兼容普通压缩 OCI 时则需要比较转换成本和在线索引成本。span 默认值不能直接当通用缓存单位。

## 如何验证

索引/内容不匹配、压缩 span 边界、随机读放大、凭据刷新和重复 prepare。

回到[联合设计决策](../../04-conch-design-reference/08-design-decisions-and-evidence.md)，比较该经验与其他候选，而不是单独据此定方案。
