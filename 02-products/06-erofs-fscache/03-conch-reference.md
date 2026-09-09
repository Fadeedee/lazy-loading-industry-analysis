# EROFS/CacheFiles：区分文件格式和缺失内容机制

> 阅读后能说明对方解决了什么、哪些经验可用于三仓、选择还需要什么证据。以下借鉴均是建议，不是已冻结接口。

## 对方具体做了什么

CacheFiles on-demand 用对象/范围请求通知用户态，用户态填充数据并报告读取完成。普通 EROFS file-backed mount 依赖文件已有可靠字节，文件稀疏洞不会自动变成远端请求。

第一方参考：[对应文档或源码](https://docs.kernel.org/filesystems/caching/cachefiles.html)。[SRC-EROFS-002] [SRC-EROFS-004] 核查日期与成熟度沿用[来源清单](../../appendix/source-inventory.md)，不是本次重新运行产品。

完整解释：[读取路径](01-data-path.md)、[缓存与快照生命周期](02-cache-snapshot-lifecycle.md)。

## 对数据服务的具体启发

最值得借鉴的是“数据缺失”与“请求完成”的明确握手，不是把本地文件路径交出去就算准备完成。对我们的服务，文件存在、范围已分配、数据写完、ready 已持久化、某个 VM 已完成映射，必须是可分别判断的状态。

lazyd 应将内容对象和本次请求的关联分开管理：请求带有有效会话/区域代次，成功只能在可读范围提交后发布；错误也必须结束等待。重连后的旧完成消息不能完成新请求。这里借鉴的是对象与请求边界，不声称 CacheFiles 已替我们的用户态缓存解决持久化、权限和回收问题。

## 三仓怎样共同借鉴

| 项目 | 可考虑的改动 |
| --- | --- |
| Conch | 在 prepare 与启动之间检查内容服务和 guest 挂载条件；保留正确的 snapshotter/mount 契约，不把 sparse 路径存在作为可启动证明。 |
| StratoVirt | 对 pmem/DAX 拦截缺失访问，核对区域代次、范围和权限，再完成对应等待；不自行把文件洞解释为有效零。 |
| lazyd | 维护独立的 ready 和在途请求状态；保证写入、持久发布、完成通知的顺序，并在断连/恢复时处理未完成请求。 |

## 选择与差异

当前仍按 EROFS + pmem/DAX 设计，不新增 CacheFiles 或 fanotify 入口。若未来另选 fscache on-demand，必须重新核对目标内核支持和维护限制；本页沿用已登记的历史机制说明，不能视为最新内核支持承诺。EROFS 格式本身也不强制某一种设备路径。

## 如何验证

验证真实缺失数据不能读成 sparse 零洞；覆盖 blob 尾页、请求失败、过期完成消息、服务退出和重开恢复。ready 状态与文件洞检测分别验证，不用 SEEK_DATA/HOLE 代替内容正确性检查。

参见[数据服务横向比较](../../03-design-comparison/08-data-service-architecture.md)与[lazyd 的持久化和回收设计](../../04-conch-design-reference/04-lazyd-responsibilities.md)。
