# OverlayBD 对 Conch 方案的借鉴

> 阅读完成后，读者能够判断为何 Conch 后续增量 rootfs 快照更适合独立块路径，而不是把所有写入塞进只读 pmem/DAX 协议。

## 直接采用的原则

- immutable lower 与 per-sandbox writable upper 明确分层。
- snapshot 以 parent lineage 表达，不用镜像 layer index 代替快照身份。
- block range cache、合并请求和 trace/prefetch 作为可写磁盘路径能力。
- checkpoint 时由上层协调 block snapshot 与 memory snapshot 一致性。

## 需要适配

- 当前只读 EROFS lower 继续走 pmem+DAX；顶层 ext4 writable disk 可走 virtio-blk/block backend。
- Conch 统一记录 lower content identity、disk snapshot lineage 和 memory snapshot lineage。
- lazyd 是否扩展 block object backend 应作为独立接口，不污染现有 EROFS FETCH v1。

## 不建议采用

- 不为了统一而把只读 EROFS lower 全部改回块级 ext4。
- 不让 StratoVirt 理解 OverlayBD layer/OCI 策略。
- 不把 block cache hit 当作 host file-page 共享的证明。

这支持“底层镜像 lower 用 pmem+DAX，运行时写入和增量盘用 blk”的组合，而不是要求一个设备同时承担相反的只读共享与可写隔离语义。
