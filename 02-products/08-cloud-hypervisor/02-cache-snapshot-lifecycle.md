# Cloud Hypervisor 快照生命周期

> 阅读完成后，读者能够理解 demand-paged restore 对启动关键路径的影响，以及为什么它不能替代 rootfs 和可写盘资源管理。

v53 将 snapshot/restore daemon 与 demand paging 放进正式发布能力，说明 VMM 可以把 memory restore 从“启动前全量读入”改成“metadata ready 后恢复、页按需供应”。[SRC-CH-001]

但恢复仍依赖：

- VM configuration/device state 兼容；
- memory snapshot/range 完整可访问；
- fault service 在 vCPU resume 前 ready；
- fault 失败能转成 VM failure；
- disk/rootfs backing 由独立路径达到可访问状态。

PR #8239 试图复用 external handler protocol；review 的核心教训是，复用 UFFD fd 传递、region lookup、wake 等底层实现可以，但 PMEM image range 与 guest memory page 的 identity、source、失败和生命周期必须分别定义。[SRC-CH-002]
