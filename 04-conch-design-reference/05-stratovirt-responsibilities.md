# StratoVirt：让按需资源成为可安全访问的设备和内存

> 明确首版要补哪些虚拟化能力、哪些逻辑不进入 VMM。三仓都能修改，不等于 StratoVirt 要承担内容服务和编排职责。

## 首版只补必要的虚拟化能力

Conch 决定用哪些资源及何时运行，lazyd/数据源解释内容并供数。StratoVirt 只接入已选定的资源，在自己的地址空间和设备生命周期内安全完成访问。

当前新增 rootfs 路径围绕 EROFS + pmem/DAX；仍需验证的是 handler 接入和页完成细节，不是同时实现 pmem 与整盘块两条产品路线。lazyd 的共享任务、来源/会话隔离和有界调度不进入 VMM，参见其[具体改进](04-lazyd-responsibilities.md)。

| 必要职责 | 最小改动边界 |
| --- | --- |
| 设备与区域接入 | 复用 backend、Region、AddressSpace 和 KVM；补足所选按需来源的配置与注册 |
| 缺页接入 | 按选定路径交接 UFFD/区域，或在内部处理事件；首版不同时新增两套 handler |
| 页完成 | 校验目标区域、源 FD、范围与权限；执行所选机制需要由 VMM 完成的映射或填页 |
| 运行与关闭 | 接入就绪、致命错误上报、取消和句柄/映射释放；复用现有运行控制 |

StratoVirt **不新增** registry/OCI 解析、远端下载与鉴权、内容 digest/cache key 解析、bitmap 缓存管理、预取算法、快照父层内容索引或缓存 GC。Conch 组织资源与引用，数据后端定位字节并管理缓存；VMM 仍必须校验本次连接、句柄和内存操作的权限。

这些边界不要求回退已有上游功能，也不以减少代码量为由省略安全检查。每项新增接口都要说明：为什么现有能力不足、为什么必须由 VMM 执行、错误怎样结束等待。

## 从最新上游接入

[基线](01-current-system-boundary.md)确认普通 pmem 已使用 MemoryBackend/Region/AddressSpace，内存恢复已有外部 UFFD MISSING/WP 交接。实现时从最新 upstream/dev 建分支，复用实际抽象，不增加重复的 syscall 或设备生命周期实现。

| 已有路径 | 可借用什么 | 不能直接假定什么 |
| --- | --- | --- |
| 普通 pmem | transport、配置空间、backend、迁移挂钩 | sparse 文件洞不会自动触发远端取数 |
| RAM UFFD | API negotiation、注册、FD 交接与脏页接口 | 外部服务已就绪；pmem remap 通道也已存在 |
| AddressSpace/KVM | region、映射与 memslot 管理 | host 文件只读打开等于 guest 全路径写保护 |
| machine lifecycle | 停止、恢复和错误上报接入 | 外部 daemon 的失败已自动上升成 VM failure |

## handler 选型不等于开发两套产品路径

**内部 handler：** VMM 读 UFFD，将事件转换为区域内偏移，向数据源请求范围，再完成映射或填页。内容索引和远端取数仍在数据源侧，VMM 不理解镜像或快照父链。地址生命周期集中，但 VMM 需要更多数据面代码和并发管理。

**外部 handler：** VMM 注册并传递 UFFD/布局，外部服务定位数据。COPY 或共享 memfd 填充可能在外部完成；若替换 VMM 的 VMA，外部需发命令/文件范围，由 VMM 严格校验后执行。

内部与外部方式均先审计现有能力，再对无法静态确认的风险做最小原型，按[核心选型计划 H1/H2](09-core-design-selection.md)比较新增 VMM 代码、协议、权限、故障处理和回归成本。已有 helper 或旧实验可以降低试验成本，但不能直接决定胜出方案，也不要求把两套同时产品化。

首版新增 pmem 路径只落地一组 handler/页完成方案；这不要求把已有 RAM 恢复改成同一机制。RAM 的 global backend 状态若不能安全承载多个区域，只做经测试证明必需的局部调整，不借此重构整个内存恢复框架。

## 如果选择 pmem 文件映射

1. 预留不会把未下载内容当作有效零的地址区域。
2. 在 guest 或设备恢复访问之前建立缺页处理和源可用条件。
3. 校验事件所属区域及区域内偏移。数据源负责把逻辑范围解析成内容文件/偏移，并保证尾页或显式零的语义；VMM 只依据已校验的区域长度、权限和完成操作处理边界，不解析 layer/digest 或父层索引。
4. 校验 FD、文件大小、范围覆盖、偏移、溢出、页/设备对齐。
5. 由 VMM 映射整个确认就绪的范围，并正确完成相应等待。
6. 管理重复/过期事件、相邻未处理区域和撤销中的 mapping。
7. 在销毁前停止新请求，等待或取消在途任务，再解除映射和句柄。

这是语义顺序，不预设特定 CLI 或 JSON 字段。VMM 接口只表达必要的区域/来源不透明标识、布局、权限、运行代次和完成结果，不暴露 registry 或缓存内部结构。MAP_PRIVATE/MAP_SHARED 要按只读性、COW 和共享结果选择；不能允许外部命令任意指定地址和映射权限。

## 与 RAM 恢复并存

先复用已有 UFFD helper、RAM backend、dirty tracking 和恢复生命周期。首版不以统一 pmem/RAM 协议、事件循环或迁移框架为前置；共用内容服务也不要求统一 VMM 协议。只有出现实际重复且能保持兼容时，再单独评估抽象。新增操作仍需明确所属区域、写权限和有效代次。

RAM 的 WP/REMOVE、dirty generation、晚到填充与 checkpoint 一致性不能套用只读 rootfs 的 ready 状态。若 inherited memfd PR 合入，复用最终接口而非继续实现一条重复恢复路径。

整盘块/COW 对照优先利用已有块设备和外部后端，不为比较方案在 StratoVirt 中新增 NBD/COW 引擎。远端预取由数据服务调度；先利用 handler 的事件和已有指标，不把新增 VMM 访问追踪框架列为启动前置。

## 失败与关闭

handler/source 有 deadline、取消和致命错误上报。运行中的不可恢复缺页应触发明确 VM 失败；准备阶段失败应阻止启动。共享服务崩溃要明确影响哪些 attachments，不能只记 warning 后无限等待。

除 vCPU 外，设备状态恢复也可能访问 RAM。readiness 屏障要覆盖真实访问时机，不只在文档上写“resume 前就绪”。

## 合入门槛

真实 KVM/guest 验证映射和唤醒；普通 pmem 与 RAM restore/WP 回归；只读保护；两 VM 同内容共享；关闭/取消竞态；PCI/MMIO 与目标架构能力。缺少某个测试环境时明确标注，不从 mock 推定通过。

参考：[Nydus](../02-products/01-nydus-v2/01-data-path.md)、[Firecracker](../02-products/07-firecracker/01-data-path.md)、[CH](../02-products/08-cloud-hypervisor/03-conch-reference.md)。
