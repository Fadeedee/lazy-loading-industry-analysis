# Conch：负责一次沙箱操作的整体结果

> Conch 不是数据服务的命令行包装。它决定用哪些资源启动、何时可以运行、发生失败后怎样处理。

## 从用户操作到资源计划

Conch 将产品语义转换为通用内容描述、授权使用关系及可选优先级/deadline/预算；lazyd 不解析 Sandbox、Template、Checkpoint 类型，也不依赖 Conch 回调完成每次缺页。业务关联信息放在调用方拥有的元数据中，规则见[接口与扩展](../06-lazyd-design/02-api-and-extensibility.md)。

创建时选择确定的镜像版本；恢复时选择同一个一致 checkpoint 中的镜像、磁盘、内存和设备状态。为每项资源确定完整/按需策略、授权范围、节点约束和截止时间。

这里的“资源计划”是设计概念，尚未冻结为某个 Go struct 或新服务。优先放进当前 Template、BootPreparer、Sandbox 和 VMM driver 分层。[上游接入点](01-current-system-boundary.md)给出已核查源码。

## 需要共同设计的契约

| 契约 | Conch 要表达什么 | 对方要返回什么 |
| --- | --- | --- |
| 内容准备 | 确定内容/快照视图、读取位置、凭据引用、策略 | 内容可用条件、可恢复身份、错误 |
| 运行 attachment | 哪个 Sandbox 使用哪种 source、设备布局、访问权限 | 本次运行句柄、就绪状态、失败通道 |
| VMM 启动/恢复 | 本地文件或按需 source、设备/内存约束 | 接入结果和可以启动/恢复的条件 |
| 释放 | 释放当前使用关系、取消中的请求 | 幂等结果和仍受保护的共享资源 |

先确定语义，再决定 HTTP、UDS、FD 或已有控制协议的字段。不要把运行句柄、缓存路径、FD 数字当作跨节点持久身份。

## 准备与发布

1. 获取足以解析资源图的 metadata，不默认遍历并下载全部 payload。
2. 授权并建立准备期间的临时引用。
3. 为所选数据源建立索引、缓存或 attachment；必要启动资产完整准备。
4. 校验资源关系和可恢复性后发布 Template/快照引用。
5. 失败时不发布可用状态，按所有权释放临时资源。

可以复用 containerd content/metadata/lease。若缓存由 lazyd 自管，则建立外部保护关系；写一个 containerd GC label 不会自动保护它的文件。

## 启动与运行

BootSpec 当前以路径表达 pmem，要增加清楚的 source 类型或等价表达。可选 native source 与 snapshotter 接入均需保持真实语义：未下载的按需内容不能冒充完整本地 snapshot。

Conch 负责等待数据源和 VMM attachment 达到可处理读取的状态，再启动或恢复。Conch 不必参与每次缺页，但必须收到不可恢复错误并把它转成 Sandbox 状态。

网络策略也不是全部下放：Conch 可提供本次启动/恢复优先级、节点总预算和业务 deadline，数据服务负责具体请求调度。

对 lazyd 的新接入要分开表达共享内容与运行 attachment：同内容重复 prepare 不重建共享任务，凭据更新不改变内容身份，取消和删除只释放当前使用关系。Conch 不逐个管理下载任务；内容服务负责剩余等待者、预算和引用保护，详见 [lazyd 架构](04-lazyd-responsibilities.md)。

## guestd 也在 Conch 的修改范围

guestd 是 `internal/agent/guestd` 中的 guest agent。当前 EROFS+pmem/DAX 主线需要稳定的 layer/view 到设备身份、挂载顺序和能力检查；可写数据按独立写层布局组装。整盘块方案只是对照，不在本轮并行新增另一套 guestd 根盘路径。

恢复时设备身份、地址和布局必须与快照匹配；不能只依靠新 VM 的自然枚举顺序。

## 什么不必进入 Conch 主进程

缺页填充、文件映射和块 I/O 可以在 VMM、lazyd 或隔离 worker 中完成。Conch 管生命周期不意味着亲自执行这些操作，也不意味着它不能提供工作集、预算或完整性策略。

## 验收

prepare 取消、服务就绪竞态、daemon shutdown、VMM 失败、重复删除、重启恢复、跨节点重绑定、普通启动回归都需要测试。一个完整 PR 可覆盖 Conch 接入，但 commits 按资源契约、准备/发布、启动、生命周期、guest 行为拆分。

参考：[Kata 的 host/guest 分工](../02-products/11-kata-and-dragonball/03-conch-reference.md)、[E2B 的恢复组织](../02-products/12-e2b-and-firecracker-stacks/03-conch-reference.md)。
