# 镜像文件：文件服务、块设备还是 pmem？

> 先比较业务需求与工程成本，再选择设备。以下方案不是互斥的产品标签，同一系统可以组合使用。

| 路线 | 读取入口 | 主要优势 | 主要代价 | 参考 |
| --- | --- | --- | --- | --- |
| 文件服务/FUSE/virtiofs | 文件操作 | 路径语义清楚，易做按文件预取 | guest/host 接入和缓存层次需分析 | [Nydus](../02-products/01-nydus-v2/01-data-path.md)、[stargz](../02-products/03-stargz/01-data-path.md)、[SOCI](../02-products/04-soci/01-data-path.md) |
| 按需块设备 | 块请求 | 与可写盘、块快照容易组合，guest 文件系统选择多 | 后端缺少文件语义；guest RAM 中仍可能复制缓存 | [OverlayBD](../02-products/05-overlaybd/01-data-path.md)、[E2B](../02-products/12-e2b-and-firecracker-stacks/01-data-path.md) |
| pmem/DAX + 缺页供数 | VMM 地址访问 | 可映射只读文件页，减少部分重复复制 | VMM、内核、映射/唤醒及设备规模复杂度 | [Nydus UFFD](../02-products/01-nydus-v2/01-data-path.md) |
| 本地完整文件 | 文件或设备读取 | 简单、运行期不依赖远端 | 下载可能阻塞启动 | 作为所有实验的基线 |

文件系统格式与设备不是一一绑定。EROFS 可以放在块设备上；pmem/DAX 也不是所有工作负载的默认最优解。

## 怎么判断

需要考虑：镜像准备/转换成本、guest 能力、多层设备数量、业务可用时间、首次请求、下载放大、跨 VM PSS、可写盘和 checkpoint 的组合成本。

对只读库共享需求强的场景，优先验证 pmem 文件映射；对整盘快照和读写恢复更重要的场景，块路径可能更简单。文件服务是有路径策略需求时的候选，不因当前某 API 不支持就排除。

本表用于保留技术取舍依据。**本项目当前主线已经明确为 EROFS + pmem/DAX**，其他路线按具体问题开展对照，不要求全部实现或比较完成后才推进主线。

## 必须避免的误解

普通 sparse 文件的洞会读成零，不能直接把它当作自动远端读取设备。每条按需路线都要有明确的缺失数据触发和完成机制。

CacheFiles/EROFS on-demand 的内核版本与维护限制见[产品说明](../02-products/06-erofs-fscache/README.md)。不把历史内核接口当作无需验证的新部署依赖。

具体取舍见[联合决策](../04-conch-design-reference/02-adopt-adapt-reject.md)。
