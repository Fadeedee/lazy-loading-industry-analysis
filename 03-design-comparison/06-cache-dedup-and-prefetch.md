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

## 面向 lazyd 的自适应预取建议

以下是本项目的待实现设计建议，不代表现有代码已经支持，也不代表业界方案已经证明它在本项目中有效。

首次 FETCH 慢不能直接触发扩大下载范围：连接建立、Bearer 鉴权、服务端处理、网络往返和带宽限制都可能造成等待。应分别记录连接/鉴权耗时、首字节等待、传输耗时、吞吐和前台排队时间，结合近期多个请求及访问连续性判断；首字节等待只能作为综合延迟信号，不能直接等同于网络 RTT。

| 观察条件 | 建议动作 |
| --- | --- |
| 连续访问、高固定请求开销、传输吞吐充足 | 逐步扩大向后预取窗口，合并相邻基础单位 |
| 随机访问或预取利用率低 | 缩小窗口，回到按需读取 |
| 前台排队、超时增多、带宽预算不足 | 限制或暂停后台预取，为前台保留容量 |
| 单次慢请求、冷连接或首次鉴权 | 收集后续样本，保持当前窗口 |

### 前台完成与后台预测分离

```text
FETCH 当前缺失范围
  -> 按固定 unit 放大，优先 ensure_range
  -> 当前范围写入并持久化，bitmap ready
  -> 立即返回当前 ready range + cache FD
  -> 后台在预算内预取后续范围
```

后台预测范围不得成为当前 FETCH 返回的前置条件。已经发送的 HTTP 请求未必能够抢占，因此需限制单次后台请求大小和并发，并为前台保留下载容量；不能仅靠队列排序宣称前台不会受影响。

当新前台请求落入预取中的范围时，复用相同 inflight 状态，将等待任务提升为前台优先级，避免重复下载；预取失败不得标 ready，后续前台访问仍可按正常错误/重试策略取数。

### 固定 bitmap 单位，动态预取窗口

`fetch.unit_bytes` 继续作为持久 bitmap 的固定基础单位。动态调整的是预取单位数量、窗口和并发，不修改已有 bitmap header 的 unit，也不改变 FETCH v1 JSON 或 cache FD 语义。

预取只推进本地内容 ready，不会自动建立其他 VM 的 HVA 映射。VM 后续访问仍可能触发 UFFD，但可命中缓存；它与 PROBE/prefault 是独立能力。

窗口按内容访问流观察，网络预算按远端和节点汇总；多个 VM 的请求合并后可能看起来随机，因此不能只用全局相邻 offset 推断每个 VM 的连续性。v1 无访问流标识时，先采用保守的内容级判断，不新增跨仓字段。

### 配置与落地顺序

由 lazyd 集中管理启用开关、初始/最大窗口单位数、后台并发、带宽预算、观测窗口和收缩条件。具体配置字段及默认值在实现前冻结，本阶段不新增 API 契约。所有预测范围必须检查加法溢出并裁剪到真实 blob 边界，尾页零填充继续沿用已有规则。

先实现可关闭、有上限的顺序预取，并测量利用率；再加入延迟/吞吐反馈和扩大、收缩阈值，避免频繁振荡。启用前以关闭预取为基线，证明应用可用时间和首次请求收益，且没有不可接受的尾延迟或下载放大。

## 验收指标

- cold pull bytes、first-fault p50/p99；
- fetch amplification = remote bytes / requested useful bytes；
- duplicate remote bytes under N VMs；
- cache disk allocated bytes；
- prefetched-but-unused bytes；
- HVA RSS/PSS 和 host page-cache reuse；
- background I/O 对业务 fault 的排队影响。

没有实测前，只能把预取和共享描述为预期收益。

新增测试矩阵应覆盖低延迟、高延迟、限带宽、抖动/超时四类网络，顺序/随机访问及单 VM/多 VM，比较 full、lazy 无预取和 lazy 有预取。记录应用可服务时间、首次业务请求、fault p50/p99、前台排队、远端字节和预取利用率；预取利用率按固定观测窗口内真正被需求访问的预取字节计算，避免把后台下载完成误当成收益。网络模拟应限定在测试代理或独立 network namespace 内。
