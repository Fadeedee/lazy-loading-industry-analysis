# 冷启动 Rootfs 懒加载流程

> 阅读完成后，读者能够从 Lazy Template 准备一路跟踪到 guest 首次文件读取，并定位每个持久对象、地址转换、FD 传递和失败责任。

## 阶段 A：Template 准备

```text
User/API
  -> Conch Template pull/prepare (lazy rootfs mode)
  -> pull Boot Index + manifests/config/descriptors + kernel/initrd assets
  -> select native EROFS rootfs layer descriptors
  -> lazyd Prepare(descriptors, fetch unit, alignment)
  -> lazyd create/reopen digest cache + bitmap + persistent instance
  -> Conch write PreparedRootfs JSON to containerd content store
  -> publish Template image record -> Boot Index/metadata GC references
```

关键约束：

- 不完整下载 EROFS layer；
- lazyd只接收明确 descriptors，不解析 Conch Boot Index；
- prepare失败时不发布可见 Template；
- cache以 digest复用，Template name不是 cache identity；
- registry credential留在 lazyd，不写 PreparedRootfs。

## 阶段 B：Sandbox/VMM 初始化

```text
CreateSandbox(template)
  -> Conch resolve Template -> Boot Index -> PreparedRootfs
  -> BootPreparer selects LazyPmem rootfs source
  -> build StratoVirt lazy pmem device specs
  -> StratoVirt validate lazy/readonly/size/instance/socket
  -> mmap anonymous HVA region
  -> register HVA in AddressSpace/KVM memslot as pmem GPA
  -> UFFDIO_REGISTER(MISSING)
  -> start internal fault loop + lazyd client
  -> expose virtio-pmem and start/resume vCPU
  -> guestd resolves stable pmem IDs
  -> mount each lower as EROFS ro,dax and assemble overlay
```

`HVA/memslot/UFFD/fault loop ready` 是 vCPU 运行前屏障。Conch 不创建 fake rootfs snapshot，StratoVirt 不接收 sparse path 作为普通 file backend。

## 阶段 C：首次读取

```text
guest process reads file GVA
  -> guest page table maps EROFS DAX offset to pmem GPA
  -> KVM memslot maps GPA to StratoVirt fault_hva
  -> host kernel queues UFFD_EVENT_PAGEFAULT
  -> StratoVirt handler: off = page_hva - base_hva
```

### Data range

```text
StratoVirt FETCH(instance_id, off, page_len)
  -> lazyd align/amplify to fetch unit
  -> ready? yes: skip remote
           no: one inflight owner fetches OCI Range
               -> validate/write cache
               -> cache sync
               -> bitmap ready + bitmap sync
               -> notify all waiters
  -> send fetch_ok(range) + read-only cache FD via SCM_RIGHTS
  -> StratoVirt fstat/range/alignment/coverage validation
  -> mmap ready range at base_hva+off with MAP_FIXED
  -> UFFDIO_WAKE remapped range
  -> vCPU retries and reads file-backed page
```

### Padding range

`off >= round_up(blob_size, host_page_size)` 且 `< pmem_size` 时，不请求 lazyd，直接 `UFFDIO_ZEROPAGE`/wake。最后一个 data page 内 `[blob_size, page_end)` 由 sparse cache初始化零语义保证。

## 阶段 D：第二个 VM

第二个 VM仍有自己的 GPA、HVA、memslot 和首次 UFFD fault。lazyd命中同一 digest/range bitmap，不重复远端下载；StratoVirt接收同一 cache inode的新 FD引用并映射。是否共享最终 host physical file page取决于映射标志、页是否干净和内核行为，需要 PSS/page-cache实测。

## 失败

- lazyd timeout/协议/FD/range错误：StratoVirt请求 machine shutdown；
- VMM退出：Conch标记 Sandbox失败并清理运行资源；
- 单 Sandbox删除：释放 mapping/device，不删除共享 lazyd cache；
- registry/digest错误：lazyd不设置 ready，返回失败；
- guest不支持 pmem/DAX：在启动/挂载阶段明确失败或按策略走 full fallback，不静默读零。

代码与职责说明见[当前系统边界](../../04-conch-design-reference/01-current-system-boundary.md)。
