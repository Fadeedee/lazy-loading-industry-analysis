# stargz 对 Conch 方案的借鉴

> 阅读完成后，读者能够把 stargz 的文件级经验转化为 lazyd 的元数据、校验、预取和观测要求，而不会错误复用其 FUSE 数据面。

## 直接采用

- 元数据优先、小数据先达到 mountable/faultable 状态。
- 每个可独立读取单位保存 digest，首次读取必须校验。
- prioritized-file/trace 驱动的启动工作集预取。
- 将 pull、mount-ready、first-read、background fetch 分开观测。

## 需要适配

- lazyd 的输入是原生 EROFS descriptor，不是 eStargz TOC。
- DAX fault 只有 offset，若要 file-aware prefetch，需要从 EROFS metadata 或离线 trace 建立反向关系。
- Conch 的 prepare 与 containerd remote snapshotter 生命周期不同，应复用原则而非 API。

## 不建议采用

- 不把 FUSE 放进 StratoVirt fault path。
- 不为兼容普通 tar/gzip 牺牲原生 EROFS+DAX 所需的对齐和可映射布局。
- 不把 rootfs snapshotter 当 VM checkpoint manager。
