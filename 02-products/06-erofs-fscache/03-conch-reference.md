# EROFS/CacheFiles 对 Conch 方案的借鉴

> 阅读完成后，读者能够选择 EROFS 作为稳定镜像格式，同时避免把已退场的 fscache on-demand ABI固化成新的跨仓依赖。

## 直接采用

- 原生 EROFS 作为只读、可验证、可 DAX 映射的 rootfs lower。
- file-backed mount/page cache 作为 range 已物化后的标准本地读取路径。
- active writable upper 与 immutable lower 分离。
- CacheFiles 的 request/complete、对象身份和 daemon crash recovery 思路。

## 需要适配

- 远端未就绪状态由 lazyd bitmap 表达，不能由 sparse hole 单独表达。
- guest DAX fault 在 StratoVirt UFFD 处理，不依赖 guest VFS 到 host CacheFiles 回调。
- kernel capability probe 应检查 EROFS+DAX，而不是假设所有内核都支持相同 mount option。

## 不建议采用

- 不新建对 Linux 6.12 已标记废弃的 EROFS fscache on-demand ABI 的长期硬依赖。[SRC-EROFS-004]
- 不把 sparse EROFS cache 直接整体传给普通 `memory-backend-file`。
- 不把 EROFS readonly lower 用作 ext4 writable snapshot 的替代品。

最终组合应是：lazyd 供应可信 EROFS ranges，StratoVirt 将 ready file range 映射到 pmem HVA，guest 以 EROFS+DAX 只读挂载。
