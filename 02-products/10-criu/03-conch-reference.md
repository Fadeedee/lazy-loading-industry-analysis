# CRIU 对 Conch 方案的借鉴

> 阅读完成后，读者能够把 CRIU 的 page server 和 post-copy 生命周期用于后续 checkpoint 设计，同时保持 rootfs 协议独立。

## 直接采用的原则

- fault daemon/source 在恢复执行前 ready。
- 同步 fault 与后台 page copy 使用共同状态去重。
- page source 生命周期覆盖整个 lazy restore phase。
- source 失败必须传播到 restored workload。

## 需要适配

- StratoVirt UFFD 对象是 VMM HVA region，不是 guest process UFFD。
- memory page source 可由本地 snapshot、object storage gateway 或远端 server 提供，但使用独立于 lazyd EROFS FETCH 的协议。
- Conch 负责 memory snapshot lease、restore phase 和终止清理。

## 不建议采用

- 不让 CRIU/lazy-pages 介入 pmem EROFS rootfs。
- 不把 guest RAM page 存入 layer sparse cache/bitmap。
- 不因为系统调用相同就合并两类 content identity。
