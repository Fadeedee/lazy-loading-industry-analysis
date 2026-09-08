# QEMU Fast Snapshot Load 数据路径

> 阅读完成后，读者能够理解 snapshot page 的按需读取、优先级和后台加载如何配合。

```text
load VM/device state + mapped-ram metadata
  -> establish RAM mapping and pending-page bitmap
  -> resume destination VM
  -> page fault enters postcopy path
  -> claim pending page against background loader
  -> synchronously load requested page
  -> resolve fault and resume vCPU
  -> background loader processes remaining pages
```

mapped-ram 为 RAM blocks 提供可按 offset 定位的文件布局，避免恢复器必须顺序扫描完整 migration stream。[SRC-QEMU-001]

## 调度意义

2026-09-08 复核的 [官方设计](https://github.com/qemu/qemu/blob/35500e5c41aec76cde59befe750600dac7a9e37a/docs/devel/migration/fast-snapshot-load.rst) 区分 fault thread 与 eager thread：前者按 fault 直接读本地快照；后者遍历其余页面。`RAMBlock->pending_bmap` 协调页面领取，避免二者重复装入并覆盖运行中的 RAM。这提供按需路径与后台路径的协作依据，不等于文档证明了任意 I/O 都可被高优先级请求抢占。

eager thread **还承担最终完成恢复的职责**，不只是性能预取。若永不访问的冷页始终未装入，VM 可能长期停留在 migration 状态；因此它不能原样类比为可随时取消的预测下载。这条路径使用本地 snapshot 文件，不是远端 RTT 自适应下载实现。

这里协调的是恢复中的 memory pages，不是不可变镜像缓存的 ready 状态。

本次仅核对官方文档，没有运行 QEMU 恢复测试。Linux/UFFD、multifd 和 vhost-user 的适用限制仍以该版本文档为准。[SRC-QEMU-001]
