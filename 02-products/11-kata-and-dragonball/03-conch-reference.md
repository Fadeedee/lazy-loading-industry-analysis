# Kata：把 host 准备和 guest 组装放在同一流程

> 阅读后能说明对方解决了什么、哪些经验可用于三仓、选择还需要什么证据。以下借鉴均是建议，不是已冻结接口。

## 对方具体做了什么

集成设计把 mount 的来源、配置和目录信息从 snapshotter 经 runtime/shim 传到 guest，由 guest 组装 lower/upper。guest 启动资产与 workload 镜像是不同对象，不能只验证 host 参数。

第一方参考：[对应文档或源码](https://github.com/kata-containers/kata-containers/blob/26c2e1630457977d931bce0527c4df92a3b8f15d/docs/design/kata-nydus-design.md)。[SRC-KATA-003] [SRC-KATA-001] 核查日期与成熟度沿用[来源清单](../../appendix/source-inventory.md)，不是本次重新运行产品。

完整解释：[读取路径](01-data-path.md)、[缓存与快照生命周期](02-cache-snapshot-lifecycle.md)。

## 对数据服务的具体启发

host 准备内容、VMM 暴露设备、guest 组装文件系统是同一次操作的不同阶段。对我们的服务，prepare 的结果必须能关联确定内容、设备布局和层顺序，但不应该把 host 上某个缓存路径作为跨节点永远有效的身份。

lazyd 需要幂等地准备内容，并为本次运行建立独立的使用关系；Conch 将这些结果绑定到设备和 guest 挂载元数据。guest 挂载失败时释放本次使用关系，不能误删其他 VM 共用的缓存。这是从完整流程推导的要求；Kata 的集成文档并不证明其数据后端已经采用我们提出的共享任务、持久化或回收策略。

## 三仓怎样共同借鉴

| 项目 | 可考虑的改动 |
| --- | --- |
| Conch | 在现有资源 owner 中贯通准备、设备参数和 guestd；保留 layer index 与稳定设备身份，处理挂载失败的回滚。 |
| StratoVirt | 使用与 machine type 匹配的设备/transport，暴露可匹配的布局；不理解 OCI 层语义或管理 guest lowerdir。 |
| lazyd | 维护可重复准备的内容对象与独立运行引用；返回经过授权的可用来源，路径变更或凭据刷新不重建公共取数状态。 |

## 选择与差异

可以采用 snapshotter 契约，也可采用 Conch 原生 source；关键是谁真实拥有 mount、状态与引用。不要为了避开某个插件而新造重复 owner，也不为模仿 Kata 增加完整 runtime 栈。

## 如何验证

验证设备重排、多层覆盖顺序、重复 prepare、跨节点重新绑定和 guest mount 失败回滚；其他 VM 仍使用该内容时不得删除缓存。再验证目标内核/transport 与恢复布局，不只检查生成的命令行。

参见[Conch 的资源编排职责](../../04-conch-design-reference/03-conch-responsibilities.md)与[数据服务横向比较](../../03-design-comparison/08-data-service-architecture.md)。
