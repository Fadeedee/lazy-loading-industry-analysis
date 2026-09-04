# Firecracker 数据路径

> 阅读完成后，读者能够分别追踪 file-backed RAM restore、external UFFD restore 和 #5740 lazy pmem 提案。

## Snapshot File backend

```text
snapshot load
  -> Firecracker MAP_PRIVATE memory file
  -> register guest memory/KVM
  -> resume vCPU
  -> host page fault
  -> kernel reads memory file page
```

启动不预读全部 guest RAM；干净 memory file page 可被多个恢复实例复用，写入触发 COW。[SRC-FC-001]

## Snapshot UFFD backend

```text
handler listens on UDS
  -> Firecracker maps/registers guest RAM with UFFD
  -> Firecracker sends UFFD fd + guest memory layout
  -> vCPU access emits UFFD event
  -> handler reads memory snapshot page
  -> UFFDIO_COPY
```

UDS 只在初始化时交换 UFFD 与 layout；后续 page fault 通过 UFFD fd 事件队列发生。[SRC-FC-002]

## Lazy pmem issue #5740

提案路径是：VMM 为 pmem 建立 anonymous read-only mapping并注册 UFFD；fault 时向 image service `FETCH`，service 通过 SCM_RIGHTS 返回 backing fd/range；VMM 用 `mmap(MAP_FIXED)` 替换 faulting region，再唤醒访问。[SRC-FC-004]

提案还区分 `PROBE` 与 `FETCH`：前者可在启动时查询已有本地 range 并预映射，后者处理真实 fault。`PROBE` 是优化，不是第二种内容格式。
