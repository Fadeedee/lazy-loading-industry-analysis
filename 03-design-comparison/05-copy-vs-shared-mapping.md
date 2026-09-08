# UFFDIO_COPY 与文件固定映射比较

> 阅读完成后，读者能够准确说明 copy 从哪里到哪里、fixed mmap 改变了什么，以及 `MAP_PRIVATE`、`MAP_SHARED` 对复用和写入语义的影响。

## Copy

```text
cache file
  -> pread/read into handler buffer
  -> UFFDIO_COPY
  -> VMM anonymous HVA physical page
```

copy 不是 HPA 到 HVA 的复制。HVA 是地址；真正复制的是 cache/file page 中的字节到为 VMM anonymous mapping 分配的物理页。每个 VM fault 都可能形成一份匿名页。

优点：UFFD resolution 语义直接，成功 COPY 默认可唤醒。缺点：额外内存 copy，跨 VM 不自动共享最终匿名页。[SRC-FC-002]

## Fixed file mapping

```text
SCM_RIGHTS cache fd + dev_off
  -> mmap(fault_hva, len, PROT_READ, MAP_* | MAP_FIXED)
  -> anonymous VMA 子区间被 file-backed VMA 替换
  -> UFFDIO_WAKE/等价 resolve
```

优点：没有 cache-to-anonymous copy；相同 inode+offset 的干净文件页可以由 host page cache 复用。风险：VMA split/replacement、UFFD register 边界、wake、权限、SIGBUS/file-size 和 rollback 都需要验证。[SRC-NYDUS-004] [SRC-FC-004]

## PRIVATE 与 SHARED

| 标志 | 读取 | 写入 | 共享含义 |
| --- | --- | --- | --- |
| `MAP_PRIVATE` | file-backed | 写时 COW | 干净 file page 可共享，修改页私有 |
| `MAP_SHARED` | file-backed | 可传播到 backing file，受 fd/protection 约束 | 同 inode+offset file page 共享更直接 |

Nydus 核查版本的 zerocopy 使用 `MAP_PRIVATE | MAP_FIXED`。[SRC-NYDUS-004] 对只读文件映射候选，必须验证 host VMA 权限和 KVM 写入处理；打开只读 FD 或声明 guest 文件系统只读，并不自动证明 KVM memslot 已受写保护。对 RAM 则还需允许私有写入及正确追踪脏页。

## 一次 fault 映射多少

完整 ready range 映射可减少后续 fault；映射预算、VMA 数和预取放大也需要衡量。每次映射都必须覆盖当前等待页并校验边界，wake/resolve 与实际完成的区间一致；未处理相邻页仍可触发 UFFD。不要把固定 4 KiB 页或服务返回的 fetch unit 当作所有对象的唯一粒度。

## handler 放在外部时的额外条件

外部进程可以持有 UFFD 并执行 COPY，但不能通过自己的 mmap 替换 VMM 的 HVA。文件映射路径需要把映射计划/FD 交给 VMM 执行并确认完成，再解决等待。现有 UFFD FD 传递能力不等于现成的跨进程 remap 协议。

## 推荐验证

- anonymous UFFD range 被 fixed remap 后当前 fault 能 wake；
- 相邻匿名页继续产生 UFFD event；
- KVM vCPU 重试能 fault-in file page；
- tail page 真实字节正确、padding 为零；
- 两 VM 相同 inode+offset 的 `smaps`/PSS/page cache 复用；
- remap 失败不会留下部分可读错误映射。
