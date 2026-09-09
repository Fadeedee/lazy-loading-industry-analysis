# Firecracker：外部 UFFD 的职责和失败边界

> 阅读后能说明对方解决了什么、哪些经验可用于三仓、选择还需要什么证据。以下借鉴均是建议，不是已冻结接口。

## 对方具体做了什么

VMM 把 UFFD FD 与布局交给外部 handler，示例由 handler 找到页并 COPY。文档同时说明 handler 未处理缺页可能让 VM 挂起；磁盘和内存快照并非由同一文件自动管理。

第一方参考：[对应文档或源码](https://github.com/firecracker-microvm/firecracker/blob/7699746649826d1dfcdde626b3131bac08f28e0d/docs/snapshotting/handling-page-faults-on-snapshot-resume.md)。[SRC-FC-002] [SRC-FC-001] 核查日期与成熟度沿用[来源清单](../../appendix/source-inventory.md)，不是本次重新运行产品。

完整解释：[读取路径](01-data-path.md)、[缓存与快照生命周期](02-cache-snapshot-lifecycle.md)。

## 对数据服务的具体启发

外部 handler 解决的是“谁接管缺页”，并不自动提供完整的数据服务。我们仍要分开保存区域布局/代次、本次 VM 的等待关系，以及多个 VM 可共享的不可变内容和下载任务。handler 可以向内容核心请求字节，但不能把 fault_hva 当作全局缓存 key。

Firecracker 文档对 handler 失效后 VM 可能挂起的提醒，是设计失败通道的直接依据。我们的要求应进一步落实为：准备失败不发布就绪；每次取数有 deadline；致命错误报告给运行管理者并结束本次恢复，而不是仅退出 handler 线程。示例的 COPY 路径不是这些会话、鉴权、任务调度和回收能力已经齐备的证据。

## 三仓怎样共同借鉴

| 项目 | 可考虑的改动 |
| --- | --- |
| Conch | 建立 attachment、内容授权及来源保护，等待 handler 就绪；服务失败后决定 VM 的最终状态和释放顺序。 |
| StratoVirt | 复用已有恢复接口，补必要的区域交接、就绪与失败边界；pmem 文件 remap 仍由 VMM 执行或严格控制。 |
| lazyd | 由适配器处理事件与区域代次，由内容核心处理缓存和共享任务；单 VM 取消只撤销其等待，不终止仍被其他 VM 使用的下载。 |

## 选择与差异

pmem 主线先验证外部 UFFD 的适用性，但外部 handler 不等于必须新增 binary，也不要求合并 RAM/pmem 状态机。Issue #5740 的文件 FD/remap 思路另属提案，不能用已有 RAM 恢复文档证明正式 pmem 已支持远端按需；状态以已登记版本为限。

## 如何验证

验证 handler 断开、来源不可达和启动竞态都能有界失败；重复页不重复下载，旧区域事件被拒绝，取消一个 VM 不影响另一个 VM。COPY/映射对照须检查各自写权限、完成规则与相邻缺页，不能只比较 API 调用次数。

参见[数据服务横向比较](../../03-design-comparison/08-data-service-architecture.md)与[StratoVirt 的最小职责](../../04-conch-design-reference/05-stratovirt-responsibilities.md)。
