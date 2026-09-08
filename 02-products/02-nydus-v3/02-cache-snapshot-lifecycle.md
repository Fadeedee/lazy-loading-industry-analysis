# Nydus v3 缓存、预取与生命周期

> 阅读完成后，读者能够判断 v3 的共享 readiness、trace prefetch 和 optimized blob 对多实例冷启动的意义，以及它尚未覆盖的快照问题。

## 缓存协调

分支文档描述多实例共享 cache readiness bitmap 和 per-blob prefetch lock，使并发冷启动只执行一次 warmup。[SRC-NYDUSV3-001] 这对应两种状态：

- content range 是否已经可信地写入 cache；
- 某个 prefetch/fetch 是否正在进行。

二者必须分别持久化和并发保护，不能用单个“文件存在”代替。

## trace-driven optimize

`nydus optimize` 将 workload trace 重排成 hot-data ondemand blob，把离散 range 请求转成更顺序的 prefetch。它改善的是已知工作集，不保证未知路径没有首次 fault。

## 生命周期局限

README 描述 metrics、trace 和 Unix-socket apiserver，但未形成稳定的跨产品 lease/GC ABI。磁盘写入和 guest RAM snapshot 也不是该实验分支已经稳定交付的统一能力。[SRC-NYDUSV3-001]

## 性能证据使用边界

分支给出了 openclaw 镜像的实验数据，但数据来自该分支自身、特定镜像和环境。可用于形成假设，不能替代目标环境中对 registry、kernel、VMM 和并发 VM 的实测。
