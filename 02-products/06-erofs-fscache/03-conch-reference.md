# EROFS/CacheFiles：区分文件格式和缺失内容机制

> 阅读后能说明对方解决了什么、哪些经验可用于三仓、选择还需要什么证据。以下借鉴均是建议，不是已冻结接口。

## 对方具体做了什么

CacheFiles on-demand 用对象/范围请求通知用户态，用户态填充数据并报告读取完成。普通 EROFS file-backed mount 依赖文件已有可靠字节，文件稀疏洞不会自动变成远端请求。

第一方参考：[对应文档或源码](https://docs.kernel.org/filesystems/caching/cachefiles.html)。[SRC-EROFS-002] [SRC-EROFS-004] 核查日期与成熟度沿用[来源清单](../../appendix/source-inventory.md)，不是本次重新运行产品。

完整解释：[读取路径](01-data-path.md)、[缓存与快照生命周期](02-cache-snapshot-lifecycle.md)。

## 三仓怎样共同借鉴

| 项目 | 可考虑的改动 |
| --- | --- |
| Conch | 根据目标内核、挂载路径和布局选择 rootfs 方案；可以复用正确的 snapshotter/mount 契约。 |
| StratoVirt | 若选 pmem/DAX，需要真正拦截缺失地址访问，并验证设备与映射权限。 |
| lazyd | 提供确定对象的内容和完成状态；不把文件洞作为可信下载索引。 |

## 选择与差异

借鉴请求/完成与对象管理，重新核对 EROFS fscache on-demand 的内核支持和维护限制，不把它作为无需验证的新依赖。EROFS 格式并不强制某一种设备路径。

## 如何验证

guest kernel 能力、真实缺失数据读取、尾部零、服务退出和缓存恢复。

回到[联合设计决策](../../04-conch-design-reference/08-design-decisions-and-evidence.md)，比较该经验与其他候选，而不是单独据此定方案。
