# stargz 缓存与生命周期

> 阅读完成后，读者能够理解 remote snapshot mount、缓存和 prefetch 的作用，并区分 containerd snapshot 与 VM checkpoint。

containerd 调用 remote snapshotter 准备 mount；snapshotter 根据镜像 layer 提供可立即挂载的视图，后续读取再取数据。[SRC-STARGZ-003]

cache 以镜像 blob/chunk 为可复用内容，多个容器可避免重复下载。overlay active snapshot 仍在只读 remote lower 上建立可写 upper；commit 该 upper 不会自动变成 eStargz optimized layer。

“containerd snapshot”在此指文件系统 mount/parent 关系，不包含 vCPU、设备或 guest RAM。要恢复 VM checkpoint，仍需独立 memory/disk state。

prefetch 是性能策略，不应改变校验和 fallback：未被预取的 chunk 继续由首次 read 触发；预取失败也必须能被同步 read 重试或明确报错。
