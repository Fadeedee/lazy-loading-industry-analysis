# Kata 镜像与沙箱生命周期

> 阅读完成后，读者能够理解 remote snapshot 的引用如何跨 containerd、shim、VM 和 guest，并识别 cleanup 的职责风险。

## 生命周期链

```text
containerd snapshot Prepare
  -> runtime/shim creates sandbox VM
  -> image service/mount ready
  -> guest agent creates container
  -> container stop/delete
  -> snapshot unmount/release/GC
```

任一环节丢失 mount annotation、content identity 或 parent relation，guest 可能看到错误 lower。runtime 退出也不能在其他 container/VM 仍引用共享 blob 时删除全局 cache。

## 写入与快照

remote snapshotter 加速只读 workload image；nydus-snapshotter 已提供 FUSE、virtiofs 和内核 EROFS 等接入形态，但 active upper 仍需单独管理。[SRC-NYDUS-SNAPSHOTTER-001] VM checkpoint 又包含 guest RAM/device state。Kata 的分层说明运行时编排可以统一，但镜像与内存数据面应独立。[SRC-KATA-001] [SRC-KATA-003]

## 兼容性

guest kernel、VMM transport、agent mount 能力和镜像格式是四个独立版本轴。只验证 host 启动参数，不足以证明 guest 内对应设备和文件系统组合可用。
