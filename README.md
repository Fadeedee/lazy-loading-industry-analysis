# 业界懒加载方案调研

> 阅读完成后，读者能够选择按概念、产品、技术问题或 Conch 设计四种路径进入整套调研，并识别每项结论的证据和成熟度。

本目录分析容器 rootfs、可写磁盘和虚机内存三类按需加载方案，并将业界经验映射到 Conch、lazyd、StratoVirt 和 guest/guestd 的职责。

## 阅读路径

- 快速评审结论：先读 [总览与 Conch 设计结论](01-overview.md)。
- 逐项解释设计依据：读 [设计决策与业界依据](04-conch-design-reference/08-design-decisions-and-evidence.md)，每项包含我们的设计、参考链接、对方实现和采用差异。
- 第一次理解懒加载：从 [概念基线](00-concepts/01-lazy-loading-boundaries.md) 开始。
- 调研某个产品：进入 [产品分析](02-products/)。
- 比较技术路线：进入 [设计比较](03-design-comparison/)。
- 规划自研方案：进入 [Conch 设计参考](04-conch-design-reference/)。
- 理解完整时序：进入 [流程分析](05-flows/)。
- 核对证据：查看 [来源清单](appendix/source-inventory.md) 和 [成熟度矩阵](appendix/maturity-matrix.md)。

## 可交互图

| 图 | 用途 |
| --- | --- |
| [业界懒加载机制全景](assets/overview/industry-panorama.html) | 区分 rootfs、可写磁盘和 guest RAM 三类对象 |
| [四类触发与完成路径](03-design-comparison/assets/mechanism-paths.html) | 比较文件、块、DAX/UFFD 和 RAM fault |
| [目标职责边界](04-conch-design-reference/assets/target-responsibilities.html) | 解释 Conch、lazyd、StratoVirt、KVM 与 guest 的分工 |
| [冷启动 Rootfs 时序](05-flows/01-cold-rootfs-lazy-start/assets/cold-rootfs.html) | 从 Template prepare 到首次 DAX fault |
| [Checkpoint 三路恢复](05-flows/03-checkpoint-template-restore/assets/checkpoint-three-path.html) | rootfs、disk 与 memory 并行恢复 |
| [内容缓存生命周期](04-conch-design-reference/assets/content-cache-lifecycle.html) | digest identity、VM 引用、失败重试和 GC |

## 分析对象

本文档始终区分三类数据：

1. **只读 rootfs lower**：镜像提供的不可变文件系统数据。
2. **可写磁盘或 upper**：运行时写入及其增量快照。
3. **guest RAM**：虚机运行内存及其全量或增量快照。

三者可以在一次 checkpoint/template 恢复中并行出现，但触发机制、数据格式和正确性条件不同，不预设它们必须共用同一种数据面。

## 证据与成熟度

正文事实引用格式为 `[SRC-项目-编号]`。来源状态统一使用：

- `released`：已进入正式版本；
- `merged`：已合入主分支但未确认发布；
- `experimental`：官方明确标记为实验性；
- `open-pr`：仍处于开放 PR；
- `closed-unmerged`：PR 已关闭且未合入；
- `prototype`：个人或研究原型；
- `proposal`：只有方案或讨论。

实验、未合入 PR 和个人原型不会被描述为现有稳定能力。

来源字段、版本快照和成熟度判定规则统一记录在来源清单中。[SRC-META-001]

## 文档维护

新增或更新内容请遵循 [文档撰写与维护原则](CONTRIBUTING.md)，其中包含设计依据模板、证据与版本规则、图文同步和提交检查。

## 输出边界

本目录只包含分析文档和 Archify 图形源/产物。第三方源码位于 `/tmp/lazy-loading-research-sources/`，不会复制到此目录；本任务也不修改 Conch、lazyd 和 StratoVirt。
