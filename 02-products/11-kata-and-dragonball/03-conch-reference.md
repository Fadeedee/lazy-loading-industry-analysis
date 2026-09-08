# Kata：把 host 准备和 guest 组装放在同一流程

> 阅读后能说明对方解决了什么、哪些经验可用于三仓、选择还需要什么证据。以下借鉴均是建议，不是已冻结接口。

## 对方具体做了什么

集成设计把 mount 的来源、配置和目录信息从 snapshotter 经 runtime/shim 传到 guest，由 guest 组装 lower/upper。guest 启动资产与 workload 镜像是不同对象，不能只验证 host 参数。

第一方参考：[对应文档或源码](https://github.com/kata-containers/kata-containers/blob/26c2e1630457977d931bce0527c4df92a3b8f15d/docs/design/kata-nydus-design.md)。[SRC-KATA-003] [SRC-KATA-001] 核查日期与成熟度沿用[来源清单](../../appendix/source-inventory.md)，不是本次重新运行产品。

完整解释：[读取路径](01-data-path.md)、[缓存与快照生命周期](02-cache-snapshot-lifecycle.md)。

## 三仓怎样共同借鉴

| 项目 | 可考虑的改动 |
| --- | --- |
| Conch | 联合修改原生资源准备、VMM 参数和 guestd，保证设备身份与挂载顺序一致。 |
| StratoVirt | 提供与 machine type 匹配的设备/transport，恢复时保持布局兼容。 |
| lazyd | 准备所选内容和访问句柄，不必承担 guest 的 mount 操作。 |

## 选择与差异

可以采用 snapshotter 契约，也可采用 Conch 原生 source；关键是谁真实拥有 mount、状态与引用。不要为了避开某个插件而新造重复 owner，也不为模仿 Kata 增加完整 runtime 栈。

## 如何验证

设备重排、多层覆盖顺序、guest mount 失败、内核/transport 组合和恢复布局。

回到[联合设计决策](../../04-conch-design-reference/08-design-decisions-and-evidence.md)，比较该经验与其他候选，而不是单独据此定方案。
