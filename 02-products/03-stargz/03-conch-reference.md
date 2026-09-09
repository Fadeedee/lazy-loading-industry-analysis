# eStargz：把启动工作集提前准备

> 阅读后能说明对方解决了什么、哪些经验可用于三仓、选择还需要什么证据。以下借鉴均是建议，不是已冻结接口。

## 对方具体做了什么

eStargz 的 TOC/chunk 支持按文件范围读取；prioritized files 放在 landmark 前，文档描述在容器运行前预取这段数据。它把构建时的文件顺序与运行时准备结合，而不是只靠在线扩大窗口。

第一方参考：[对应文档或源码](https://github.com/containerd/stargz-snapshotter/blob/c2bf18e5a94dcfd959cabf744f4bbb4ef8d980a2/docs/estargz.md)。[SRC-STARGZ-002] 核查日期与成熟度沿用[来源清单](../../appendix/source-inventory.md)，不是本次重新运行产品。

完整解释：[读取路径](01-data-path.md)、[缓存与快照生命周期](02-cache-snapshot-lifecycle.md)。

## 三仓怎样共同借鉴

| 项目 | 可考虑的改动 |
| --- | --- |
| Conch | 在构建/prepare 关联内容版本与工作集，提供启动阶段、等待策略和预算；不把下载优先级全部硬编码进 VMM。 |
| StratoVirt | 若走 DAX，只在必要时提供最小地址观测；文件工作集由构建/数据服务的索引或 trace 还原，不在 VMM 增加文件索引与预取算法。 |
| lazyd | 解析并校验适用于本内容的预取范围，跳过 ready、复用在途任务、允许需求提升优先级；限制排队/字节和会话占用，并与稳定缓存粒度分离。 |

## 对数据服务的具体启发

预取不是在现有 FETCH 外面简单加循环。lazyd 先要有共享任务、前台/后台优先级、可取消等待和容量预算，否则后台准备可能阻塞真正的缺页。

eStargz 的文件工作集不能直接变成 EROFS 字节偏移，必须绑定实际布局；可以借鉴采样方法而不转换为 eStargz。远端跨用户统计是我们的额外建议，不是该来源已实现的功能，详见[预取与工作集](../../03-design-comparison/06-cache-dedup-and-prefetch.md)。

## 选择与差异

当前仍围绕 pmem/DAX；不为借鉴预取增加 FUSE/virtiofs 产品路径。工作集失效或没有样本时保留按需能力，是否等待预取由 Conch 决定，不应成为来源服务的隐式阻塞。

## 如何验证

无预取、静态工作集和在线策略分组测业务首请求与下载放大；验证版本失配、后台拥塞、需求命中预取、取消及两个会话之间的公平性。收益尚待本地测量。

先完成[数据服务基础](../../03-design-comparison/08-data-service-architecture.md)，再按[联合路线](../../04-conch-design-reference/07-phased-roadmap.md)验证优化。
