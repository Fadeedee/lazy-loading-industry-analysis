# 沙箱懒加载：业界调研与三仓联合设计

> 从一次沙箱启动或快照恢复出发，理解哪些数据可以晚点加载，以及 Conch、StratoVirt、lazyd 怎样共同完成这件事。

## 从这里开始

1. **先看当前要决定什么**：[核心设计选型与验证计划](04-conch-design-reference/09-core-design-selection.md)。区分固定需求与未定实现，对照缓存布局、持久化、共享任务和 handler 位置；实验尚未执行。
2. **需要更简短的入门**：[我们要解决什么问题](01-overview.md)。先分清镜像文件、运行时磁盘和内存，不必先懂 UFFD。
3. **需要继续深入**：按操作读[冷启动](05-flows/01-cold-rootfs-lazy-start/README.md)或[快照恢复](05-flows/03-checkpoint-template-restore/README.md)；想知道选择依据，再读[八个设计问题与证据](04-conch-design-reference/08-design-decisions-and-evidence.md)。

到这里即可讨论整体架构。函数、地址术语和源码版本放在详细章节，不需要逐页读完整个仓库。

需要连贯的流程和时序图，再读[整体候选方案](04-conch-design-reference/00-overall-design.md)。它以单文件缓存、外部 handler 为例解释三仓协作，具体实现待选型验证。

## 这份方案如何形成

三个项目都在设计和修改范围内。Conch 面向用户组织启动、运行、快照和删除；StratoVirt 提供虚机执行能力；lazyd 的服务范围可按整体需求演进。分工由场景和工程约束推导，不以某个现有 API 为边界。

调研先问内容身份、并发请求、完成条件和关闭责任，再决定哪些代码保留或重组。产品事实不是改造方案；[数据服务横向比较](03-design-comparison/08-data-service-architecture.md)将第一方证据、待查缺口和三仓设计推导放在一起，不默认 lazyd 的现有结构不变。

当前只读 rootfs 主线是 **EROFS + pmem/DAX**，整盘块方案只是对照，不是并列开发任务。lazyd 先完善共享内容、会话隔离、有界调度和缓存安全，再扩展预取与快照来源；详情见 [lazyd 架构与改进](04-conch-design-reference/04-lazyd-responsibilities.md)。

Conch、StratoVirt 的开发基线是**开发开始时最新 upstream/dev 的明确 commit**。已核查的版本见[上游能力基线](04-conch-design-reference/01-current-system-boundary.md)。正文分别标注产品事实、设计建议和待验证选项。

## 按问题查资料

lazyd 定位为独立通用供数服务，Conch 是调用方之一；外部 handler 为待验证主方案，StratoVirt 改动收敛。服务模型、扩展和调度研究从 [lazyd 设计专题](06-lazyd-design/README.md)开始阅读，具体协议与缓存格式仍待验证。

| 你想知道 | 入口 |
| --- | --- |
| 当前选型与下一步验证 | [核心设计选型与验证计划](04-conch-design-reference/09-core-design-selection.md) |
| 连贯理解一种完整候选 | [整体候选方案与时序图](04-conch-design-reference/00-overall-design.md) |
| page、COW、DAX、UFFD 是什么 | [基础概念](00-concepts/)与[术语表](appendix/glossary.md) |
| 某个产品究竟怎么做 | [产品分析](02-products/) |
| 共享缓存之外，数据服务还要设计什么 | [身份、任务、持久化与生命周期比较](03-design-comparison/08-data-service-architecture.md) |
| 文件、块、pmem 哪个合适 | [技术路线比较](03-design-comparison/) |
| 三个仓各改哪里 | [Conch](04-conch-design-reference/03-conch-responsibilities.md)、[StratoVirt](04-conch-design-reference/05-stratovirt-responsibilities.md)、[lazyd](04-conch-design-reference/04-lazyd-responsibilities.md) |
| 怎么安排开发与验收 | [联合实施路线](04-conch-design-reference/07-phased-roadmap.md) |
| 结论来自哪里、版本是否可靠 | [来源清单](appendix/source-inventory.md)、[成熟度矩阵](appendix/maturity-matrix.md) |

## 用图辅助理解

- [三仓协作图](04-conch-design-reference/assets/target-responsibilities.html)：职责建议，尚未固定数据面接口。
- [冷启动时序](05-flows/01-cold-rootfs-lazy-start/assets/cold-rootfs.html)：先准备可读取能力，再启动业务。
- [Checkpoint 恢复时序](05-flows/03-checkpoint-template-restore/assets/checkpoint-three-path.html)：协调磁盘、内存、镜像与设备状态。
- [缓存生命周期](04-conch-design-reference/assets/content-cache-lifecycle.html)：什么时候可读、什么时候可回收。
- [业界机制全景](assets/overview/industry-panorama.html)与[触发路径](03-design-comparison/assets/mechanism-paths.html)：产品机制参考，不是本项目选型结论。

HTML 可以直接在浏览器打开，不需要运行产品服务。整体方案内也提供直接可读的 PNG。图形检查范围与截图见[既有图形验证](appendix/validation-2026-09-08.md)和[整体方案图形验证](appendix/validation-2026-09-09-overall-design.md)。

## 阅读时保留两个区别

**对象不同，不代表必须是三个进程或三套协议。** 镜像、磁盘历史层和内存可以共享取数、缓存与调度基础设施，但写入、覆盖顺序和运行状态必须有明确类型。

**虚机开始运行，不代表所有远端数据已经下载。** 懒加载把一部分等待转移到了运行期，是否更好要看业务可用时间、首次请求、尾延迟和实际下载量。

维护规范见 [CONTRIBUTING.md](CONTRIBUTING.md)。本仓只保存调研、设计与图形，不保存产品源码副本、凭据和临时构建产物。
