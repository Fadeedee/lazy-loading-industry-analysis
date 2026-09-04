# Cloud Hypervisor 对 Conch 方案的借鉴

> 阅读完成后，读者能够用 Cloud Hypervisor 的正式能力和未合入 PR 约束 StratoVirt 的模块边界与提交策略。

## 直接采用

- guest RAM demand-paged restore 作为独立 VMM 能力和独立资源源。
- snapshot/restore daemon 与运行 VM 的生命周期、可观测性解耦。
- sparse memory snapshot 不等于 rootfs sparse cache。

## 需要适配

- StratoVirt 可让 PMEM fault 与 memory restore 复用 `userfaultfd` syscall wrapper、event polling、range validation 和 wake helper。
- 两者应使用不同 region type、source identity、protocol handler 和 metrics。
- Conch 统一编排两者 ready barrier，但不把 lazyd EROFS FETCH 用作 RAM page server。

## 不建议采用

- 不照搬已关闭的 #8239 作为外部稳定协议。
- 不在缺乏问题定义和分层说明时提交同时改变 PMEM 与 snapshot 的大 VMM PR。
- 不以“都是 UFFD”推导出相同的 cache、GC 或错误恢复语义。
