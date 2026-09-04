# Conch 职责与接入设计

> 阅读完成后，读者能够说明 Conch 在新版 containerd-native 架构中保存什么、调用谁、如何准备 sandbox，以及哪些数据绝不能由 Conch 直接管理。

## 核心职责

Conch 是 orchestration 和 resource relationship owner：

- 解析 Boot Index，选择 rootfs、memory、sandbox components；
- 决定 full/lazy rootfs 与 cold/resume memory mode；
- 调用 lazyd prepare 明确的 EROFS descriptors；
- 将 PreparedRootfs metadata 写入 containerd content store；
- 通过 Template image record 和 Sandbox store labels 建立 GC 引用；
- BootPreparer 生成 typed rootfs/disk/memory sources；
- 构造 StratoVirt 参数并管理 VMM 生命周期；
- 在 checkpoint/restore 中协调三路资源状态和失败。

## 持久化模型

```text
Template image record
  -> Boot Index descriptor
       -> rootfs component descriptor
       -> prepared-rootfs metadata descriptor
       -> mem-snapshot descriptor (resume only)
       -> sandbox component descriptor

Sandbox record
  -> current Boot Index GC ref
  -> runtime snapshot refs
  -> prepared-rootfs metadata ref
```

PreparedRootfs 应保存稳定内容字段：schema version、mapping mode、layer index、blob digest/size、pmem size、media type、lazyd instance ID 和 VM 内 pmem ID。它不保存 registry credentials、bitmap/cache path 或部署 socket。

## Lazy Template prepare

1. Conch 拉取并验证 Boot Index metadata closure，但不完整拉 rootfs layer bytes。
2. 解析 rootfs component 中明确的原生 EROFS layer descriptors。
3. 调 lazyd prepare；lazyd 返回 stable prepared identity/size。
4. Conch 写 PreparedRootfs content blob并建立 children GC label。
5. 所有 component/metadata 都成功后，原子发布/更新 Template image record。
6. 任一步失败不发布可见 Template；lazyd 已产生的 digest cache 可保留复用。

## BootPreparer 分流

现有 `PmemPaths []string` 应演进为显式 source：

```text
RootfsSource
  ├── FileBacked { paths }
  └── LazyPmem { layers, data_socket }
```

full 分支继续创建/挂载 snapshot BootLayout；lazy 分支不创建 fake rootfs runtime snapshot，只加载 PreparedRootfs metadata并产生 external rootfs layout。两者最终都生成 VMM 能理解的 typed device specs。

## 不属于 Conch

- 不读取或更新 lazyd bitmap；
- 不持有 cache data fd；
- 不处理 UFFD event；
- 不解析 EROFS 文件 offset；
- 不在单 sandbox cleanup 中删除共享 cache；
- 不自己实现第二套 registry range downloader；
- 不把 guest RAM page 存进 PreparedRootfs。

## 与 PR #155 的协调

若增量内存 PR 先合入，Conch restore coordinator 同时准备：PreparedRootfs、conch-cow memory attachment 和 writable disk lineage。`WaitAttachmentReady` 与 lazy pmem handler ready 都属于 vCPU resume 前置，但两条 source/attachment 不能互相替代。[SRC-CONCH-003]
