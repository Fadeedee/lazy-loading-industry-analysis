# stargz snapshotter

> 阅读完成后，读者能够理解 eStargz 如何在保持 tar/gzip 生态兼容的同时支持文件级按需读取，以及它为何不是 pmem/UFFD 或 VM 内存恢复方案。

## 定位

stargz snapshotter 是 containerd remote snapshotter。eStargz 在 gzip-compatible layer 中增加 TOC 和可独立验证的 chunk，使容器不必在启动前下载并解包完整 layer。[SRC-STARGZ-001] [SRC-STARGZ-002]

## 核心能力

- 先取得 TOC/必要元数据；
- FUSE 文件读取映射到 registry blob range；
- chunk digest 校验；
- prioritized files 和 landmark 支持启动文件预取；
- 通过 containerd snapshotter Prepare/mount 生命周期接入。[SRC-STARGZ-003]

## 边界

它处理的是只读 rootfs lower。overlay upper、VM writable disk 和 guest RAM snapshot 由其他组件负责。其 file-aware prefetch 值得借鉴，但数据面不能直接用于 DAX HVA fault。

来源：[SRC-STARGZ-001] [SRC-STARGZ-002] [SRC-STARGZ-003]
