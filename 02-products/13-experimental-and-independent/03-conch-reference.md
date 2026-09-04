# 实验项目对 Conch 方案的借鉴

> 阅读完成后，读者能够把反例转成可验证的设计约束，避免仅凭 API 名称或共享直觉做架构决定。

## 直接采用的审查规则

- 每项能力都记录 `released/merged/experimental/proposal`，issue 不算实现。
- snapshot API 与 lazy data path 分开验收。
- 所有共享层都做小镜像、单 VM 和冷 cache 基准，允许 full/local fast path。
- 跨节点恢复把 memory、disk、config 和 lineage 作为完整资源图。

## 需要适配

- Conch 可使用统一 snapshot state model，但每个 resource driver 报告自己的 `MetadataReady/Faultable/Failed`。
- lazyd 对小对象可选择直接完整 fetch，避免 bitmap/UDS/fault 的固定成本。
- guestd/StratoVirt 保留 full/local fallback，不能让 lazy 服务不可用时所有本地镜像都无法启动。

## 不建议采用

- 不把 CubeSandbox EROFS issue 当作实现参考。
- 不从 OpenSandbox API 推断 UFFD 或按需块恢复。
- 不把 cache sharing 写成无条件性能收益。
