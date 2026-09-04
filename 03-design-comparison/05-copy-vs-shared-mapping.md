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

Nydus 已合入 zerocopy 代码当前使用 `MAP_PRIVATE | MAP_FIXED`。[SRC-NYDUS-004] Conch 目标是只读 EROFS lower，因此最终选择不能只看共享潜力，还要确保 guest 无法通过 pmem 写坏共享 cache：`PROT_READ`、virtio-pmem readonly 和 KVM readonly memslot 应形成防御链。

## 一次 fault 映射多少

lazyd 若返回一个已下载、校验、page-aligned 的完整 ready range，StratoVirt 应映射该 range，而不是只映射当前 4 KiB page。wake 覆盖已 remap range；未 remap 相邻页继续由 UFFD 触发。

## 推荐验证

- anonymous UFFD range 被 fixed remap 后当前 fault 能 wake；
- 相邻匿名页继续产生 UFFD event；
- KVM vCPU 重试能 fault-in file page；
- tail page 真实字节正确、padding 为零；
- 两 VM 相同 inode+offset 的 `smaps`/PSS/page cache 复用；
- remap 失败不会留下部分可读错误映射。
