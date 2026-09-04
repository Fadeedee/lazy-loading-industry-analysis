# QEMU 对 Conch 方案的借鉴

> 阅读完成后，读者能够将 QEMU 的 fault-priority、后台收敛和 page-state 思路用于后续 StratoVirt memory restore。

## 直接采用的原则

- faulting vCPU 请求优先于后台 prefetch。
- 同一 page/range 的 fault 与 background worker 去重。
- 使用明确 pending/ready/failed 状态，不从 sparse extent 猜测正确性。
- restore failure 进入 VMM 级失败路径。

## 需要适配

- rootfs range 可以长期保持部分 materialized；guest RAM restore 是否后台全量收敛要单独配置。
- lazyd bitmap 面向 immutable content，memory pending bitmap 应位于 StratoVirt snapshot subsystem。
- Conch 可统一显示进度，但不合并两套 bitmap 文件格式。

## 不建议采用

- 不为 rootfs FETCH 引入完整 QEMU migration protocol。
- 不把 postcopy page server 当 OCI registry client。
- 不让后台全量恢复挤占首次业务 rootfs fault 的 I/O 带宽，需做优先级和并发预算。
