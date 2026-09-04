# 触发、取数与缺页消解

> 阅读完成后，读者能够沿着一次首次访问定位阻塞发生在哪里、哪个组件收到事件、哪个组件取数，以及访问通过何种机制恢复。

## 1. 一次按需访问的五个角色

无论方案采用 FUSE、块设备还是 UFFD，都可以拆成五个角色：

1. **访问者**：进程、guest kernel 或 vCPU。
2. **触发器**：VFS/FUSE 请求、块 I/O、DAX fault、UFFD event 或恢复器。
3. **定位器**：把文件 offset、sector 或 fault HVA 转成远端对象和 range。
4. **供应者**：从 registry、object storage、page server 或本地 cache 取得可信数据。
5. **消解器**：填充缓存、完成 I/O、copy page、建立 file mapping 或 wake 等待者。

这些角色可以位于同一进程，也可以跨内核、VMM 和 daemon。评审设计时应先定位角色，再判断进程边界是否合理。

## 2. 四类主流触发路径

### 2.1 文件级：VFS/FUSE

```text
应用 read/open
  -> guest/host VFS
  -> FUSE daemon
  -> image index 定位 chunk
  -> HTTP Range
  -> local cache
  -> FUSE reply
```

优势是天然理解路径、inode 和文件范围，容易做按文件预取。代价是每次首次文件访问要经过用户态文件系统路径。Nydus v2、stargz 和 SOCI 是主要样本。[SRC-NYDUS-001] [SRC-STARGZ-003] [SRC-SOCI-002]

### 2.2 块级：virtio-blk/NBD/TCMU/ublk

```text
guest 文件系统 read
  -> guest block request
  -> virtio/NBD/TCMU backend
  -> sector 到远端 range
  -> block cache
  -> 完成 block request
```

优势是 guest 可以继续使用 ext4 等普通块文件系统，也自然容纳可写 upper 和磁盘快照；后端只看到扇区，不直接知道文件语义。OverlayBD 和 E2B NBD COW rootfs 属于此类。[SRC-OVERLAYBD-001] [SRC-E2B-001]

### 2.3 内核按需缓存：EROFS + CacheFiles

```text
应用读取 EROFS 文件
  -> EROFS 发现 cache range 不存在
  -> CacheFiles 通过 /dev/cachefiles 发 READ 请求
  -> 用户态 daemon 填充 anonymous fd 对应范围
  -> READ_COMPLETE ioctl
  -> 内核继续文件访问
```

这条路径把文件系统访问和缓存状态放在内核框架内，daemon 负责取数。Linux 官方 CacheFiles 文档定义了请求和完成协议。[SRC-EROFS-002] Nydus 也提供过 EROFS fscache 接入。[SRC-NYDUS-003] 但 EROFS 官方资料说明该 on-demand 模式自 Linux 6.12 起废弃，因此新设计不能只因为它减少用户态跳转就忽略维护方向。[SRC-EROFS-004]

### 2.4 地址级：DAX + UFFD

```text
guest 读取 EROFS+DAX 文件
  -> guest 页表把文件 offset 映射到 pmem GPA
  -> KVM 通过 memslot 找到 StratoVirt HVA
  -> HVA missing fault
  -> UFFD event
  -> FETCH cache fd/range
  -> mmap(MAP_FIXED) 或 UFFDIO_COPY
  -> wake vCPU
```

DAX 绕过 guest 文件数据 page cache，把文件访问变成对 pmem 映射区的 load。UFFD 监听的是 **StratoVirt 进程中的 HVA range**，并不直接监听 GVA 或 GPA。Nydus 已合入的 UFFD block service同时提供 Copy 与 Zerocopy 路径。[SRC-NYDUS-004] Firecracker #5740 描述了相似的 FD passing + fixed remap 方案，但仍是提案。[SRC-FC-004]

## 3. 地址如何关联

在 lazy pmem 场景中，一次读取可按下面的关系理解：

```text
guest process GVA
  --guest page table--> pmem GPA
  --KVM memslot------> base_hva + (GPA - pmem_gpa_base)
  --host VMA/page table--> anonymous page 或 cache file page
```

- **GVA** 属于 guest 进程。
- **GPA** 属于整台 guest VM 的物理地址空间。
- **HVA** 属于宿主机上的 StratoVirt 进程虚拟地址空间。
- **base_hva** 是某个 lazy pmem region 在该进程中的起始 HVA。
- **KVM memslot** 描述一段 GPA 对应哪段 HVA，它不是逐页页表。
- **UFFD registration** 告诉 host kernel：这段 HVA 的 missing fault 交给用户态 handler。

fault offset 通常按 `fault_hva - base_hva` 计算，再与 `blob_size`、`pmem_size` 和 page size 校验。

## 4. resolution 不等于 wake

必须先让重试访问能够得到正确数据，再唤醒等待者。

### UFFDIO_COPY

handler 将用户缓冲区中的字节复制到 faulting anonymous page。除非使用 `DONTWAKE`，成功 copy 通常同时唤醒等待者。每个 VM 得到自己的匿名页，内容相同也不会自动共享物理文件页。

### UFFDIO_ZEROPAGE

为确定语义就是零的范围建立零页，适合 `round_up(blob_size, page_size)` 之后的 pmem padding。不能用它处理 sparse cache 中“尚未下载但真实内容未知”的洞。

### `mmap(MAP_SHARED | MAP_FIXED)` + wake

VMM 用 lazyd 返回的 cache fd 和 `dev_off` 覆盖 faulting HVA 子区间。新的 VMA 是 file-backed mapping，后续读由 host page cache 支持；多个 VM 映射同一 inode+offset 时具备共享文件页的条件。固定重映射本身不等同于完成 UFFD wait，因此需要明确的 wake/resolve 操作。Nydus 已合入实现和 Cloud Hypervisor 未合入 PR 都提供了这一机制的工程证据。[SRC-NYDUS-004] [SRC-CH-002]

### 普通 file-backed fault

如果 VMM 从启动时就把完整文件映射为 backend，内核可自行 fault-in 本地文件页。但 sparse hole 会被解释成合法的零，而不是“去远端下载”，这正是 lazy pmem 不能简单把 sparse EROFS cache 直接作为完整 `memory-backend-file` 的原因。

## 5. 粒度并不只有 page size

一次 4 KiB fault 可以触发更大的工作：

| 粒度 | 示例 | 目的 |
| --- | --- | --- |
| fault 粒度 | host page，通常 4 KiB | 精确阻塞和唤醒 |
| fetch 粒度 | 1 MiB unit、SOCI span | 减少网络往返 |
| 校验粒度 | chunk/span/digest unit | 保证内容完整性 |
| 落盘粒度 | ready range | 持久化与 recovery |
| remap 粒度 | lazyd 返回的 page-aligned ready range | 降低后续 fault 次数 |
| prefetch 粒度 | 启动工作集或 trace range | 隐藏串行 fault 延迟 |

StratoVirt 不应把 lazyd 返回的放大 range 缩回单个 fault page；只要范围、文件长度和 pmem 边界校验通过，映射完整 ready range 才能利用数据面放大和 host page cache。

## 6. 失败时谁负责结束等待

fault 路径最危险的失败不是返回错误，而是 vCPU 永久阻塞且没有上层状态变化。可靠实现至少需要：

- FETCH 超时和有界重试；
- request ID 与 response 校验；
- digest/range/file size/alignment 校验；
- 失败日志包含 VM、region、instance、offset；
- handler 失败能够上报 VMM 级 fatal error，并触发 VM shutdown 或明确失败状态；
- 不能把未经校验的数据映射到 guest 可读地址。

Cloud Hypervisor #8239 的 review 也说明：把 PMEM fault 与 snapshot restore 共用外部协议不是天然合理，必须先定义功能边界和失败模型。[SRC-CH-002]

## 7. 对当前项目的映射

| 阶段 | 主要责任方 |
| --- | --- |
| 选择 OCI layer / snapshot | Conch |
| digest、range、下载、校验、bitmap、cache fd | lazyd |
| HVA、memslot、UFFD、FETCH client、remap、wake | StratoVirt |
| `/dev/pmemN` 识别、EROFS+DAX、overlay | guest/guestd |

这条边界让 lazyd 不需要理解 KVM，让 StratoVirt 不需要理解 OCI，也使未来的 guest RAM lazy restore 可以保留独立后端。
