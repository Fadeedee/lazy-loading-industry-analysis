# Cloud Hypervisor：恢复服务与 VMM 如何分工

> 阅读后能说明对方解决了什么、哪些经验可用于三仓、选择还需要什么证据。以下借鉴均是建议，不是已冻结接口。

## 对方具体做了什么

v53 发布说明包含 demand-paged guest memory restore、snapshot/restore offload 和后台 prefault。另一个 PMEM 外部 UFFD PR #8239 关闭未合入，不能把两项能力混为一项。

第一方参考：[对应文档或源码](https://github.com/cloud-hypervisor/cloud-hypervisor/releases/tag/v53.0)。[SRC-CH-001] [SRC-CH-002] 核查日期与成熟度沿用[来源清单](../../appendix/source-inventory.md)，不是本次重新运行产品。

完整解释：[读取路径](01-data-path.md)、[缓存与快照生命周期](02-cache-snapshot-lifecycle.md)。

## 对数据服务的具体启发

将恢复工作移出 VMM 后，仍要区分“VM 可以运行”“当前缺页已满足”和“后台恢复已完成”。对于我们的数据服务，缓存 ready 只能说明字节可用，不表示该 VM 所有页都已填充，更不表示来源可以立即删除。

lazyd 可以用独立适配器服务上游 RAM 恢复入口，复用不可变内容缓存、共享任务和有界取数，而把写保护、驻留状态与恢复完成条件留在对应恢复组件。发布说明足以证明宣布了恢复/offload 能力，但不足以证明远端鉴权、FD 协议、全局调度或 GC 细节；不能据此宣称对方已经采用我们这套数据服务架构。

## 三仓怎样共同借鉴

| 项目 | 可考虑的改动 |
| --- | --- |
| Conch | 分别呈现运行、恢复和后台完成状态；在仍可能缺页时保护快照来源，关闭时等待相关使用关系结束。 |
| StratoVirt | 复用自身上游恢复机制，只补必要交接和错误处理；不从 Cloud Hypervisor 发布说明推定 pmem 协议。 |
| lazyd | 将来源读取和内容缓存与 RAM 页状态分开；通过适配器接已有服务，明确任务取消、超时与完成通知，不重写整套恢复控制。 |

## 选择与差异

共享 UFFD 或传输框架可以评估，但 region kind、写权限和代次要明确。首版先复用 StratoVirt 已有恢复契约，不以统一 pmem/RAM 框架为前置。某 PR 未合入不是强制两套服务或拒绝统一协议的技术证明；也不是我们必须重构 VMM 的理由。

## 如何验证

后续 RAM 阶段验证前台缺页不被后台队列饿死、EOF/超时可结束等待、后台完成后来源何时可释放，以及恢复服务版本/能力不匹配时的明确错误。若修改共享组件，再分别回归 pmem 和 RAM，不能由一种路径通过推出另一种通过。

参见[数据服务横向比较](../../03-design-comparison/08-data-service-architecture.md)与[内存恢复比较](../../03-design-comparison/03-memory-snapshot-restore.md)。
