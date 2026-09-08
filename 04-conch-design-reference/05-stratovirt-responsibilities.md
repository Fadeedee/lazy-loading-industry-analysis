# StratoVirt：让按需资源成为可安全访问的设备和内存

> VMM 的核心责任是地址空间、设备与执行状态。handler 放在哪里可以选，但 VMM 对自己映射和运行状态的责任不能外包。

## 从最新上游接入

[基线](01-current-system-boundary.md)确认普通 pmem 已使用 MemoryBackend/Region/AddressSpace，内存恢复已有外部 UFFD MISSING/WP 交接。实现时从最新 upstream/dev 建分支，复用实际抽象，不增加重复的 syscall 或设备生命周期实现。

| 已有路径 | 可借用什么 | 不能直接假定什么 |
| --- | --- | --- |
| 普通 pmem | transport、配置空间、backend、迁移挂钩 | sparse 文件洞不会自动触发远端取数 |
| RAM UFFD | API negotiation、注册、FD 交接与脏页接口 | 外部服务已就绪；pmem remap 通道也已存在 |
| AddressSpace/KVM | region、映射与 memslot 管理 | host 文件只读打开等于 guest 全路径写保护 |
| machine lifecycle | 停止、恢复和错误上报接入 | 外部 daemon 的失败已自动上升成 VM failure |

## 两种 handler 候选

**内部 handler：** VMM 读 UFFD，向内容服务请求范围，在自己的地址空间完成映射或填页。地址生命周期集中，但 VMM 需要更多数据面代码和并发管理。

**外部 handler：** VMM 注册并传递 UFFD/布局，外部服务定位数据。COPY 或共享 memfd 填充可能在外部完成；若替换 VMM 的 VMA，外部需发命令/文件范围，由 VMM 严格校验后执行。

先验证外部方式复用上游的程度，以内部方式为复杂度和时延对照。RAM 的现有 global backend 状态不一定适合多个 pmem/session；应审计注册、注销和状态归属后再决定抽象层级。

## 如果选择 pmem 文件映射

1. 预留不会把未下载内容当作有效零的地址区域。
2. 在 guest 或设备恢复访问之前建立缺页处理和源可用条件。
3. 根据明确的逻辑布局定位数据，区分真实内容、尾页和合法补零。
4. 校验 FD、文件大小、范围覆盖、偏移、溢出、页/设备对齐。
5. 由 VMM 映射整个确认就绪的范围，并正确完成相应等待。
6. 管理重复/过期事件、相邻未处理区域和撤销中的 mapping。
7. 在销毁前停止新请求，等待或取消在途任务，再解除映射和句柄。

这是语义顺序，不预设特定 CLI 或 JSON 字段。MAP_PRIVATE/MAP_SHARED 要按只读性、COW 和共享结果选择，不按名称判断。

## 与 RAM 恢复并存

可以共用 UFFD 绑定、传输、事件循环和错误框架，也可以使用类型化统一协议。必须分别表达 region kind、布局、写权限和代次。

RAM 的 WP/REMOVE、dirty generation、晚到填充与 checkpoint 一致性不能套用只读 rootfs 的 ready 状态。若 inherited memfd PR 合入，复用最终接口而非继续实现一条重复恢复路径。

## 失败与关闭

handler/source 有 deadline、取消和致命错误上报。运行中的不可恢复缺页应触发明确 VM 失败；准备阶段失败应阻止启动。共享服务崩溃要明确影响哪些 attachments，不能只记 warning 后无限等待。

除 vCPU 外，设备状态恢复也可能访问 RAM。readiness 屏障要覆盖真实访问时机，不只在文档上写“resume 前就绪”。

## 合入门槛

真实 KVM/guest 验证映射和唤醒；普通 pmem 与 RAM restore/WP 回归；只读保护；两 VM 同内容共享；关闭/取消竞态；PCI/MMIO 与目标架构能力。缺少某个测试环境时明确标注，不从 mock 推定通过。

参考：[Nydus](../02-products/01-nydus-v2/01-data-path.md)、[Firecracker](../02-products/07-firecracker/01-data-path.md)、[CH](../02-products/08-cloud-hypervisor/03-conch-reference.md)。
