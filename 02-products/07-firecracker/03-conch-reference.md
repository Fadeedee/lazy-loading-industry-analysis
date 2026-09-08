# Firecracker：外部 UFFD 的职责和失败边界

> 阅读后能说明对方解决了什么、哪些经验可用于三仓、选择还需要什么证据。以下借鉴均是建议，不是已冻结接口。

## 对方具体做了什么

VMM 把 UFFD FD 与布局交给外部 handler，示例由 handler 找到页并 COPY。文档同时说明 handler 未处理缺页可能让 VM 挂起；磁盘和内存快照并非由同一文件自动管理。

第一方参考：[对应文档或源码](https://github.com/firecracker-microvm/firecracker/blob/7699746649826d1dfcdde626b3131bac08f28e0d/docs/snapshotting/handling-page-faults-on-snapshot-resume.md)。[SRC-FC-002] [SRC-FC-001] 核查日期与成熟度沿用[来源清单](../../appendix/source-inventory.md)，不是本次重新运行产品。

完整解释：[读取路径](01-data-path.md)、[缓存与快照生命周期](02-cache-snapshot-lifecycle.md)。

## 三仓怎样共同借鉴

| 项目 | 可考虑的改动 |
| --- | --- |
| Conch | 管理 attachment 就绪、磁盘/内存一致性、handler 失败和 VM 最终状态。 |
| StratoVirt | 可复用外部交接思路，但需检查自己的设备恢复访问时机、错误通道和 UFFD 状态。 |
| lazyd | 可以承接外部 handler 或仅提供 page source；按权限和进程隔离需求选择。 |

## 选择与差异

外部 handler 是候选而非必须新开 binary。Issue #5740 的文件 FD/remap 思路另属提案，不代表正式 pmem 已支持远端按需。

## 如何验证

handler 断开、源不可达、启动竞态、COPY/映射对照、重复页和错误隔离。

回到[联合设计决策](../../04-conch-design-reference/08-design-decisions-and-evidence.md)，比较该经验与其他候选，而不是单独据此定方案。
