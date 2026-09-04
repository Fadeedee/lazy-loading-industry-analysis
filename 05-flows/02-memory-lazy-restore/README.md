# Guest RAM 懒恢复流程

> 阅读完成后，读者能够区分本地 file-backed memory restore 与 incremental memfd+UFFD restore，并知道它们如何与 rootfs lazy 并行而不共用数据协议。

## Full/File 路径

```text
Conch resolves full-v1 memory component
  -> prepare complete memory/state files
  -> StratoVirt -incoming file:...,mapped=true
  -> map memory snapshot file
  -> restore device/vCPU state
  -> resume
  -> host kernel faults local memory file pages on demand
```

这条路径的“懒”发生在本地文件页驻留，不包含远端 OCI range fetch。若启动前必须先完整下载 memory component，网络阶段仍是 eager。

## Incremental/COW 路径

以下流程对应 Conch PR #155 与 StratoVirt PR #2017 所表达的待合入能力，不是当前 `dev` 已有功能。[SRC-CONCH-003] [SRC-SV-002]

```text
Conch resolves incremental-v1 memory manifest
  -> verify base/delta layers + build_map
  -> conch-cow Attach(sandbox, snapshot root)
  -> conch-cow creates memfd
  -> SCM_RIGHTS memfd to Conch
  -> Conch starts StratoVirt with inherited memory-backend-memfd fd
  -> StratoVirt maps guest RAM and registers UFFD
  -> StratoVirt passes UFFD + mappings to conch-cow
  -> WaitAttachmentReady
  -> restore vCPU/device state and resume
```

## 首次 RAM page fault

```text
vCPU accesses missing guest RAM page
  -> KVM touches StratoVirt HVA
  -> UFFD event delivered to conch-cow
  -> fault_hva -> guest memory offset
  -> build_map selects final base/delta layer
  -> pread layer page
  -> pwrite page into memfd at guest offset
  -> UFFDIO_WAKE
  -> private VM memory mapping retries successfully
```

这里不使用 `UFFDIO_COPY`；memfd是一个 VM attachment 的 memory backend。它与 rootfs cache FD不同：rootfs FD对应不可变共享 EROFS内容，memory memfd对应该次恢复的 guest RAM view。

## 与 rootfs lazy 的并行

vCPU开始运行后，可能同时产生：

- instruction/data 所需 RAM page fault -> conch-cow；
- guest 文件读取所需 pmem HVA fault -> StratoVirt internal handler -> lazyd。

Conch需要在恢复前等待两条 handler ready，并在运行期监听二者 fatal状态。I/O scheduler应优先服务当前 fault，再执行两条路径各自的背景预取。

## 恢复完成与清理

- memory后台是否最终全量物化由 memory policy决定；
- rootfs cache可以长期保持部分物化；
- Sandbox delete触发 conch-cow Detach和StratoVirt teardown；
- memory layer lease跟随 checkpoint/sandbox引用；
- lazyd内容 cache不因该 VM退出而删除。

完整资源模型见[三路恢复设计](../../04-conch-design-reference/06-checkpoint-three-path-restore.md)。
