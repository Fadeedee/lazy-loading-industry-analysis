# Kata Containers 与 Dragonball

> 阅读完成后，读者能够理解 Kata 中 host runtime、VMM、guest agent、remote snapshotter 和 Nydus 的边界，并判断 Dragonball 在其中是什么角色。

## 系统边界

Kata 用 VM 隔离 container workload。host runtime/shim 负责编排，VMM 创建 VM，guest agent 在 guest 内完成容器和挂载操作。guest kernel/initrd/rootfs image 是 VM guest assets，workload OCI image 是另一类资源。[SRC-KATA-001] [SRC-KATA-004]

## Nydus 接入

Kata Nydus 设计把 remote snapshotter 产生的 mount metadata 传给 shim/guest，host upper 与 guest 可见的 RAFS lower 按部署方式组装。[SRC-KATA-002] [SRC-KATA-003]

## Dragonball

官方架构资料将 Dragonball 描述为 Kata runtime-rs 中使用的 VMM 方向，而不是独立的镜像 lazy-loading service。[SRC-KATA-004] 本调研因此把它放在 VMM/运行时边界中，不虚构独立 lazy image protocol。

来源：[SRC-KATA-001] [SRC-KATA-002] [SRC-KATA-003] [SRC-KATA-004]
