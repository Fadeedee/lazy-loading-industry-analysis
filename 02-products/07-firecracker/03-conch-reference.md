# Firecracker 对 Conch 方案的借鉴

> 阅读完成后，读者能够识别 Firecracker 已验证的外部 UFFD 边界，以及 #5740 仍需由本项目自行实现和验证的部分。

## 直接采用

- handler 先监听 UDS，VMM 完成 HVA/UFFD registration 后再传 UFFD/layout，最后才恢复 vCPU。
- UFFD 权限、jail/socket 可见性和 peer credential 属于部署契约。
- memory file、VMM state、disk backing 分开管理。
- handler timeout/崩溃必须导致明确 VM failure，而不是静默卡住。

## 需要适配

- 当前设计将 rootfs fault handler 放在 StratoVirt 内部，lazyd 只提供 FETCH/FD；不是 Firecracker memory restore 的外部 handler 拓扑。
- #5740 的 PROBE/FETCH 和 fixed remap 可参考，但需要以 Nydus 已合入实现和 StratoVirt spike/E2E 为更强代码证据。
- Firecracker memory snapshot 的 `MAP_PRIVATE` 共享适用于 guest RAM；rootfs cache mapping 还要满足 EROFS readonly 与 range readiness。

## 不建议采用

- 不把 issue #5740 写成 Firecracker 已支持 lazy pmem。
- 不复用同一 `instance_id` 同时标识 OCI layer 与 memory snapshot。
- 不接受“handler 失败后 vCPU 无限等待”作为产品错误策略。
