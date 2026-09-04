# Kata/Nydus 数据路径

> 阅读完成后，读者能够追踪 image metadata 从 containerd snapshotter 经 shim/VMM 到 guest mount 和 nydusd 的路径。

```text
containerd image + remote snapshotter
  -> Prepare returns mount/annotations
  -> Kata shim/runtime selects VM/VMM
  -> host or guest nydusd serves RAFS content
  -> virtiofs/FUSE/EROFS path enters guest
  -> guest agent mounts/assembles container rootfs
```

Kata 的不同部署可以把文件系统 daemon 放在 host 或 guest，传输通过 virtiofs 等路径；设计重点是把 snapshotter metadata 保真地传到真正执行 mount 的组件。[SRC-KATA-002] [SRC-KATA-003]

guest asset rootfs 不能和 workload rootfs 混淆：前者用于启动 VM 用户态/agent，后者是某个容器的镜像层。[SRC-KATA-001]

对 Conch 来说，guestd 类似 guest 内负责 mount/组装的 agent；StratoVirt 类似 VMM；lazyd 类似独立 image service，但协议和镜像格式由本项目定义。
