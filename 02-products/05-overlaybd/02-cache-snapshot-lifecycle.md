# OverlayBD 可写层与快照生命周期

> 阅读完成后，读者能够理解块级 lower/upper、commit 和 live snapshot 如何形成磁盘谱系，并看清其 GC 与一致性要求。

## Layer chain

```text
immutable base layers
  -> read-only merged view
  -> sandbox writable upper
  -> committed/new snapshot layer
```

只读 lower 可按内容共享；writable upper 属于单个 sandbox 或快照 head。commit 将某个时间点的可写状态固化为可作为后续 parent 的对象。[SRC-OVERLAYBD-001]

## 快照正确性

- snapshot 必须记录 parent chain；
- active writer 与 snapshot/commit 需要一致性屏障；
- 删除 parent 前必须确认没有 child/VM 引用；
- 远端 layer、local cache 和 runtime upper 的 GC 不能只按路径是否打开判断；
- 与 VM memory checkpoint 组合时，需要在 Conch 层协调磁盘与 vCPU 的一致性时间点。

## 共享边界

多个 VM 可以共享 immutable block lower 和节点 cache，但每个 VM 的 ext4 page cache、可写块和 guest RAM 独立。共享后端块不保证 host RSS 以 file-backed page 的形式自然合并。
