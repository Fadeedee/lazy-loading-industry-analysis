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

这些边界用于区分 host runtime、VMM 和 guest agent，并不限定新的系统必须采用同样协议。对本项目的取舍单列在[借鉴分析](03-conch-reference.md)。
