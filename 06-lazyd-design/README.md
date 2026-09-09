# lazyd 独立数据服务设计

> 理解 lazyd 如何独立供数、与调用方解耦，以及哪些能力属于可靠性基础、哪些属于后续研究。

## 定位与约束

lazyd 是可被不同产品使用的按需内容服务，Conch 是调用方之一。首期实现原生 EROFS + pmem 路径，其他来源按实际需求扩展，不预建通用插件框架。

| 组件 | 边界 |
| --- | --- |
| 调用方（如 Conch） | 解析产品资源、授权并准备内容、建立使用关系、决定启动和失败策略 |
| lazyd 内容核心 | 内容定位、共享任务、校验、缓存发布、有界调度和回收 |
| lazyd pmem 适配模块 | 消费 UFFD、区域定位、内容范围转换、完成与失败关联 |
| StratoVirt | 注册与交接区域、映射校验、页完成和运行控制；改动保持收敛 |

Conch 不进入逐次缺页热路径，lazyd 不解析 Sandbox、Template 或 Conch 专有镜像分类。优先级和预算等提示是可选的；未提供时仍能正常按需供数。

默认部署为一个 lazyd 进程服务多个会话，不按 VM 创建独立进程。进程隔离需求另行评估，运行约束见运行模型章节。

## 阅读顺序

1. [核心模型](01-core-model.md)：内容、来源、会话、任务与使用关系。
2. [接口与扩展](02-api-and-extensibility.md)：核心字段、labels、extensions 与行为参数。
3. [运行架构](03-runtime-architecture.md)：进程、适配模块与失败处理。
4. [缓存与生命周期](04-cache-and-lifecycle.md)：发布、保护与回收。
5. [调度研究](05-scheduling-research.md)：候选机制与对照实验。

## 文档边界

本目录定义 lazyd 的通用设计要求；[三仓职责](../04-conch-design-reference/04-lazyd-responsibilities.md)解释产品接入；[核心选型登记](../04-conch-design-reference/09-core-design-selection.md)统一保存决定与验证状态。具体 API、缓存格式与算法尚未冻结，实验尚未执行。

稳定接口与实现说明在进入开发后归入 lazyd 代码仓并随代码评审，本仓保留调研和选型依据。
