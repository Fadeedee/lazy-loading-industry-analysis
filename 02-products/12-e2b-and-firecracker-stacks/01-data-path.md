# E2B 数据路径

> 阅读完成后，读者能够分别追踪 E2B 的内存 fault 和 rootfs block I/O，理解两条路径如何在 sandbox 启动中并行。

## Memory

```text
template memory metadata/memfile
  -> Firecracker guest RAM + UFFD
  -> resume vCPU
  -> missing page fault
  -> memory page provider
  -> requested page + prefetch
```

## Rootfs/disk

```text
guest ext4 read/write
  -> virtio block/NBD
  -> per-sandbox COW cache
       read miss -> read-only template rootfs/object cache
       write -> private changed block
```

两条路径共享 sandbox/template identity 和调度资源，但访问粒度和数据结构不同。[SRC-E2B-001]

## 关键性能思想

- 先让 template metadata 和 fault backend ready，再恢复执行；
- fault 请求优先，邻近/trace prefetch 隐藏后续延迟；
- template cache 可从 object storage 按需补齐；
- private COW 只存变更块，避免复制完整 rootfs。
