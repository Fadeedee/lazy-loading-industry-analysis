# QEMU 内存恢复生命周期

> 阅读完成后，读者能够理解 pending bitmap、后台收敛和失败传播在内存懒恢复中的作用。

## 状态

恢复器至少区分未加载、正在加载、已就绪和失败 page。fault thread 与 background loader 争用同一页时需要去重，并优先完成阻塞 vCPU 的请求。[SRC-QEMU-001]

## 收敛

与长期 remote rootfs cache 不同，memory restore 通常希望后台最终把当前 VM 所需 RAM 收敛到本地/内存，随后 fault service 可以退出或进入稳定状态。是否全量后台加载取决于产品目标，不应直接套用到大镜像 rootfs。

## 生命周期

- 创建/传输 compatible snapshot；
- 初始化 RAM block mapping 与 pending state；
- vCPU fault 和后台加载并发；
- 所有页完成后结束 postcopy phase；
- 任一不可恢复 I/O/校验错误终止 migration/VM restore。

磁盘快照仍由 block layer/外部编排管理。
