# 缓存、去重与预取比较

> 阅读完成后，读者能够把 cache identity、ready bitmap、inflight 去重、prefetch 和 prefault 分成独立机制，并为它们设置可测指标。

## 五个不同动作

| 动作 | 解决的问题 | 例子 |
| --- | --- | --- |
| persistent cache | 下次不再远端下载 | Nydus blob cache、SOCI span cache |
| inflight dedup | 并发请求不重复下载 | per digest/range future/lock |
| prefetch | fault 前取远端数据 | eStargz prioritized files、Nydus trace |
| prefault/PROBE | fault 前建立本地映射 | Nydus UFFD prefault、FC #5740 PROBE |
| background fill | 执行恢复后最终物化 | QEMU/CRIU memory pages |

来源：[SRC-NYDUS-001] [SRC-NYDUS-004] [SRC-STARGZ-002] [SRC-SOCI-002] [SRC-QEMU-001] [SRC-CRIU-001]

## 身份

- immutable layer：digest + format/version/security domain；
- fetch state：content identity + aligned range/unit；
- writable snapshot：snapshot ID + parent lineage；
- memory page：memory snapshot ID + RAM block/page offset。

## 粒度调节

fetch unit 越大，网络往返少但下载放大高；越小，首次访问精确但 bitmap、HTTP 和 syscall 开销增加。SOCI span、Nydus chunk/compression group 和 lazyd unit 都体现相同权衡，但默认值不能跨格式照搬。[SRC-SOCI-002] [SRC-NYDUSV3-001]

## 预取策略层级

1. static：固定首段/metadata；
2. build hint：构建时文件列表；
3. runtime trace：历史工作集；
4. adaptive：连续访问/邻近 range；
5. fleet coordination：同节点并发只 warm 一次。

## 验收指标

- cold pull bytes、first-fault p50/p99；
- fetch amplification = remote bytes / requested useful bytes；
- duplicate remote bytes under N VMs；
- cache disk allocated bytes；
- prefetched-but-unused bytes；
- HVA RSS/PSS 和 host page-cache reuse；
- background I/O 对业务 fault 的排队影响。

没有实测前，只能把预取和共享描述为预期收益。
