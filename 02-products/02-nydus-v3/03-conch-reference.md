# Nydus v3 对 Conch 方案的借鉴

> 阅读完成后，读者能够判断哪些 v3 架构思想适合现在吸收，哪些需要等待上游稳定或本地验证。

## 直接采用的思想

- lazyd 内部把 OCI backend、EROFS 解析、cache、bitmap 和 prefetch 做成共享 core。
- 借鉴接口与内容核心分层：当前 lazyd 提供 HTTP control 和 seqpacket FETCH，复用内容供应逻辑；UFFD handler 位于 StratoVirt。
- dedup identity 与 compression/fetch unit 解耦。
- 并发 VM 对同一 digest/range 使用 inflight 去重和完成 fan-out。
- 预取根据 trace/工作集生成，不把固定全量 read-ahead 当唯一策略。

## 需要适配

- 当前 lazyd 已有 v1 bitmap/FETCH 协议，应渐进重构 core，不为追随实验分支从头重写。
- Conch 的 rootfs、writable disk 和 guest RAM 资源图高于 Nydus image core，不能下沉到 lazyd。
- StratoVirt 只消费稳定数据面，不依赖实验 CLI 或 on-disk format。

## 暂不采用

- 不承诺 v3 artifact/API 兼容。
- 不直接引用分支性能数字作为项目验收目标。
- 当前目标不引入 fanotify，也不照搬 Nydus v3 的全部 FUSE、NBD、ublk、UFFD frontend；已有 fanotify 能力仅属历史兼容范围。

最合理的近期路径是：保留现有三仓实现边界，用 v3 的 core/frontend 分层审视 lazyd 重构，而不是替换整个项目。
