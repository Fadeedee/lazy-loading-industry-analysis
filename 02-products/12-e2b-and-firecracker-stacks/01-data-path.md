# E2B 数据路径

> 阅读完成后，读者能够分别追踪 E2B 的内存 fault 和 rootfs block I/O，理解两条路径如何在 sandbox 启动中并行。

## Memory

```text
template memory metadata/memfile
  -> Firecracker guest RAM + UFFD
  -> resume vCPU
  -> missing page fault
  -> memory page provider
  -> source.ReadAt -> UFFDIO_COPY
  -> 独立的预取流程
```

## Rootfs/disk

```text
guest ext4 read/write
  -> virtio block -> host NBD device/server
  -> per-sandbox block Overlay
       read -> writable cache -> optional sealing cache -> base device
       write -> private changed block
```

两条路径共享 sandbox/template identity 和调度资源，但访问粒度和数据结构不同。[SRC-E2B-001]

## 2026-09-08 源码复核

本节固定到 `cc7c574233ad`，不是只根据架构图推断实现，也不表示本地运行过 E2B。

- [`NewNBDProvider`](https://github.com/e2b-dev/infra/blob/cc7c574233ad98665a7c72a3d37b0af89ae79a71/packages/orchestrator/pkg/sandbox/rootfs/nbd.go) 为只读 rootfs 创建 cache，再构造 `block.NewOverlay`，启动后通过 ready future 返回设备路径。NBD 是 host 侧后端，不是 guest 直接访问远端 NBD 服务。[SRC-E2B-004]
- [`Overlay.ReadAt/WriteAt`](https://github.com/e2b-dev/infra/blob/cc7c574233ad98665a7c72a3d37b0af89ae79a71/packages/orchestrator/pkg/sandbox/block/overlay.go) 实现上述读取优先级及私有写入。`SwapCache` 把旧 cache 放入冻结的 sealing 槽位，让新写进入新 cache；`FoldSealing` 只将新 cache 缺少的块补入，失败时保持旧层可读。不能在封存任务刚开始时就释放旧块。[SRC-E2B-002]
- [`Userfaultfd.faultPage`](https://github.com/e2b-dev/infra/blob/cc7c574233ad98665a7c72a3d37b0af89ae79a71/packages/orchestrator/pkg/sandbox/uffd/userfaultfd/userfaultfd.go) 从独立 `PageReader` 读取，检查短读，有限退避重试后执行 copy；最终取数失败调用非空 `onFailure` 并返回错误。RAM 还有 WP/REMOVE 状态，不能照搬为只读 rootfs range bitmap。[SRC-E2B-003]

这些代码明确展示了后端 ready、读取覆盖顺序和私有 head 切换。面向三仓的候选比较见[借鉴分析](03-conch-reference.md)。`onFailure` 在调用点是否触发 VM 停止，也需独立追踪，不能仅凭该函数认定生命周期闭环。

## 关键性能思想

- 先让 template metadata 和 fault backend ready，再恢复执行；
- fault 请求优先，邻近/trace prefetch 隐藏后续延迟；
- template cache 可从 object storage 按需补齐；
- private COW 只存变更块，避免复制完整 rootfs。
