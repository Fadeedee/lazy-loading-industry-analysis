# 谁发现数据还没到？

> 触发器决定我们能看到哪些语义，不决定数据服务必须属于哪个项目。

| 机制 | 观察对象 | 等待者 | 如何完成 | 参考 |
| --- | --- | --- | --- | --- |
| FUSE/文件服务 | 文件操作 | 发起文件访问的线程 | 文件服务 reply | Nydus、stargz、SOCI |
| 块后端 | sector/block 请求 | I/O 请求及其等待者 | block completion | OverlayBD、E2B |
| CacheFiles | 内核缓存范围 | 文件读取者 | 填充并报告完成 | EROFS/CacheFiles |
| fanotify pre-content | 特定文件访问前事件 | 被拦截的文件访问者 | 准备内容并响应 | Nydus v3 实验方向 |
| UFFD | 注册地址区域的缺页 | 访问该页的线程/vCPU | COPY、ZERO 或经验证的映射/唤醒 | Nydus、Firecracker |
| postcopy | 迁移/恢复缺页 | vCPU | 页到达并安装 | QEMU |

版本和实际实现边界见[来源清单](../appendix/source-inventory.md)。

## 选型要问什么

事件是否有文件路径？缺页地址如何变成内容位置？缓存命中是否仍进入服务？缺失范围包含压缩/加密单位吗？服务失败如何结束等待？权限与地址空间由谁维护？

UFFD handler 可以在 VMM 内部，也可以外置。外置后它持有 UFFD 不意味着拥有目标进程的普通 mmap 权限。

## 本项目范围

当前不把 fanotify 作为 guest pmem/DAX 的触发方案：直接地址访问不是 host 文件系统的 VFS 请求。它在产品研究中仍有位置，但不能因此自动加入开发计划。

文件、块、pmem 和 RAM 的触发适配可以连接共同内容核心。是否共用协议和进程要看真实兼容与故障域，不预先规定为独立或统一。
