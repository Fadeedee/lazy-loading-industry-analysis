# 分阶段开发与合入路线

> 阅读完成后，读者能够按当前上游状态组织 lazyd、StratoVirt 和 Conch 三个大 PR 的 commit，并在每阶段得到可独立验证的结果。

## 前置条件

- Conch PR #184 已合入，可直接以最新 `upstream/dev` 为 metadata owner 基线。[SRC-CONCH-002]
- Conch PR #155 与 StratoVirt PR #2017 当前开放；开始 rootfs lazy 实现前再次确认是否已合入并 rebase。[SRC-CONCH-003] [SRC-SV-002]
- 旧 Conch/StratoVirt lazy 分支只读保留，不继续堆新修复。

## 阶段 1：lazyd 完整 PR

建议 commit：

1. durable range map and digest identity；
2. descriptor-based EROFS prepare and registry auth；
3. range coordination and crash recovery；
4. seqpacket FETCH and read-only SCM_RIGHTS fd；
5. tests, metrics and protocol docs。

验收：单元/mock registry/FD passing/restart recovery；相同 digest 不同 image/index只创建一份 cache；错误请求无 FD 泄漏。

## 阶段 2：StratoVirt 完整 PR

建议 commit：

1. lazy pmem config/validation；
2. address_space UFFD missing helpers；
3. anonymous lazy backend and readonly memslot；
4. lazyd protocol/FD transport；
5. fault classification, remap and wake；
6. lifecycle/fatal shutdown；
7. tests and docs。

验收：每个 commit边界可解释；snapshot UFFD回归；mock lazyd；真实 KVM/guest smoke；不改变 regular pmem。

## 阶段 3：Conch 完整 PR

建议 commit：

1. lazy config and PreparedRootfs schema；
2. content-store persistence and GC references；
3. lazyd control client；
4. Template selective pull/prepare；
5. typed rootfs source and BootPreparer split；
6. StratoVirt lazy device args；
7. guestd stable pmem mapping and EROFS+DAX；
8. lifecycle, tests and docs。

验收：不创建 fake snapshot；full/CLH/regular SV回归；prepare失败不发布 Template；sandbox delete不删共享 lazyd cache。

## 阶段 4：端到端与性能

- cold registry/cold cache lazy start；
- full 与 lazy内容对拍；
- tail/padding；
- lazyd/registry故障触发明确 VM failure；
- 两 VM相同 layer只下载一次；
- `MAP_PRIVATE`/`MAP_SHARED` 的 RSS/PSS/page cache；
- 多 layer 与大量 layer；
- x86_64/aarch64、PCI/MMIO transport；
- full rootfs、memory full/incremental回归。

## 阶段 5：checkpoint 三路恢复

在 rootfs路径稳定后：

1. 将 writable ext4 upper定义为 block snapshot resource；
2. 接入 #155 类 memory incremental resource；
3. 实现 Restore Coordinator并行 prepare/统一失败；
4. 加 lease/refcount、cache prune和跨节点分发；
5. 做 fault优先级和全局 I/O budget。

## 可独立推进的增强：网络感知预取

在阶段 4 建立无预取性能基线后推进，不作为 rootfs 正确性首版的合入前置：

1. 补齐首字节/传输/排队时间、吞吐、预取利用率观测。
2. 实现有上限、可关闭的顺序预取，保持固定 bitmap unit 和现有 FETCH v1。
3. 保证前台当前范围就绪即返回；后台任务共享 inflight，支持前台等待者提升优先级。
4. 加入基于近期样本的窗口扩大/收缩，限制后台请求大小、并发和带宽。
5. 通过隔离的测试代理或 network namespace 对比高延迟、限带宽、抖动和随机访问，验证应用可用时间、首次请求、fault p99 与下载放大。

先由 lazyd 约束 rootfs 下载预算；checkpoint 三路共存后，再由 Conch 协调各后端预算，避免内存恢复、块读取和 rootfs 预取相互挤占。策略详见 [自适应预取建议](../03-design-comparison/06-cache-dedup-and-prefetch.md)。

## 合入顺序

推荐 `lazyd -> StratoVirt -> Conch -> E2E/benchmark`。每个仓可以一个完整 PR，但 commit必须按逻辑拆分；评审中不要用“后续 commit会修”掩盖当前 commit的错误路径。
