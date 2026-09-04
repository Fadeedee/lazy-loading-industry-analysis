# SOCI 对 Conch 方案的借鉴

> 阅读完成后，读者能够把 SOCI 的 descriptor、派生元数据和 registry 工程经验用于 Conch/lazyd，而不引入无必要的 gzip 索引层。

## 直接采用

- Conch 显式选择 rootfs descriptor，lazyd 不自行猜测复杂 image/index 中哪一层是 rootfs。
- cache identity 基于 immutable digest，不基于 tag 或 layer index。
- registry Bearer challenge/token、scope cache 和一次 401 refresh retry 作为公共能力。
- metadata、range、cache hit、remote bytes 和 first-read latency 分开观测。

## 需要适配

- `fetch.unit_bytes` 的基准应使用 EROFS workload 数据，不照搬 SOCI span 默认值。
- lazy-rootfs metadata 要记录 OCI descriptor 和 lazyd prepared identity，但无需再生成 zTOC。
- GC 要处理 Conch snapshot 引用、lazyd cache lease 与 OCI content store 的边界。

## 不建议采用

- 当前输入已经是原生 EROFS layer，不再增加 SOCI index/zTOC。
- 不让 lazyd解析 Conch 专有 manifest 编排语义。
- 不把 SOCI rootfs remote snapshot 等同于 writable disk 或 memory snapshot。
