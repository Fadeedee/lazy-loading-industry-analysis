# Nydus v2 缓存与生命周期

> 阅读完成后，读者能够理解 Nydus 的内容缓存、预取和 reconnect 如何服务 rootfs，并知道它没有自动解决 writable disk 与 guest RAM 快照。

## 内容与缓存身份

RAFS data blob 以内容摘要标识，chunk metadata 保存数据摘要和 blob offset。相同 blob/chunk 可以跨镜像引用，运行时本地 blob cache 避免重复拉取。[SRC-NYDUS-001] [SRC-NYDUS-002]

早期设计文档说明 blob cache 假设只取镜像的一小部分，因此未强调 eviction。[SRC-NYDUS-001] 对长期运行、磁盘受限的节点，这一假设不能直接沿用，必须另加 lease/refcount、水位和 GC。

## prefetch

构建阶段可记录要预取的文件/目录，运行时后台拉取。[SRC-NYDUS-001] UFFD service 又引入 prefault：它不是下载 hint，而是让 VMM 对已经 ready 的 block range 提前建立映射。两者分别优化“远端内容未就绪”和“HVA 尚未映射”。

## 生命周期

- 镜像 prepare：取得 bootstrap/config，建立后端与缓存。
- 首次访问：按 chunk/range 拉取并更新 cache。
- reconnect：VMM/UFFD 数据面重连后重新传递必要状态。
- 多实例：共享 blob cache，但每个 VMM 保有自己的 HVA、memslot 和映射。
- 停止：释放 VMM mapping 不应等同于删除内容 cache。

## 快照边界

Nydus 的核心对象是只读 rootfs lower。容器/VM 写入仍需要 overlay upper 或独立可写块设备；guest RAM 快照也由 VMM 的 snapshot restore 机制负责。把 UFFD 用于 pmem 不会把它自动变成 memory snapshot service。

## 风险

- cache ready 状态必须晚于可信数据持久化；
- FD 与 region metadata 必须绑定正确 blob identity；
- reconnect 不能使旧 UFFD/region 与新 VM 混用；
- Copy 与 Zerocopy 的内存占用、权限和失败策略不同；
- 内核 EROFS fscache on-demand 路线存在版本退场风险。[SRC-EROFS-004]
