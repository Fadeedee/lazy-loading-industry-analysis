# 从远端镜像到首次业务请求

> 阅读完成后，能够解释谁选择启动资源、什么时候可以开机，以及懒加载为什么不能只测“VM 已启动”。

这是**待验证的角色级流程**，不是当前上游已实现的协议。先看[交互时序图](assets/cold-rootfs.html)，再按下面三步理解；图里的“供数角色”可以包含 lazyd 与类型适配器，不代表固定进程数。

## 1. 准备镜像，但不下载所有 rootfs 数据

用户请求准备镜像。Conch 解析 Boot Index、manifest 和组件描述，确定 kernel/initrd、rootfs 等各自用途；启动必需的小型状态和资产先完整就绪。

随后 Conch 把明确的数据对象交给内容供应服务。服务完成来源授权、索引/大小检查、缓存打开及引用登记，返回可按需读取的资源句柄。Conch 只有在整组资源准备成功后才发布可用记录。

这里要先补齐当前上游拉取路径的 payload 筛选，不能只加一个 lazy 标签后继续递归 Fetch 所有内容。现状见[代码基线](../../04-conch-design-reference/01-current-system-boundary.md)。

**要观察：**启动前远端 rootfs 字节数、实际落盘量、失败准备是否留下可见但不可用的记录。

## 2. 连接数据来源，再允许 guest 运行

Conch 按 EROFS + pmem/DAX 主线及独立可写层生成启动配置。StratoVirt 建立设备和地址空间，并与相应数据来源完成就绪确认。guestd 是 Conch 的 guest 内组件，负责设备识别及文件系统组装。

以下区分主线与技术对照，不是两项并列开发任务：

| 路线定位 | guest 如何读取 | 启动前必须就绪 |
| --- | --- | --- |
| 当前主线：只读 EROFS + pmem/DAX，加独立可写层 | 文件数据通过 pmem 映射访问 | 文件系统能力、地址映射、fault 处理及数据来源 |
| 可选对照：rootfs 纳入块设备/COW 视图 | 文件系统产生块请求 | 块视图、父层索引、读写后端及私有 writable head |

本页流程图说明外部 UFFD handler 主方案，交接与完成仍需验证；内部为备选，状态见[核心选型计划](../../04-conch-design-reference/09-core-design-selection.md)。块方案利用已有设备/外部后端，不为比较另写 VMM 块存储引擎。**发送过 FD 不等于数据来源已就绪**；注册事件处理、来源确认、失败通道要一起满足启动门槛。

## 3. 第一次读文件时发生什么

当前 pmem/DAX 主线的文件映射流程如下，具体 handler/完成协议仍需验证：

```text
guest 访问文件数据
  -> pmem GPA 对应的 VMM HVA 尚未填充
  -> host kernel 产生 UFFD missing event，当前访问等待
  -> handler 定位区域及区域内偏移
  -> 数据源解析内容范围，命中缓存或下载、校验并发布 ready
  -> VMM 安全映射已就绪文件范围
  -> resolve/wake，guest 重试并读到数据
```

若 handler 在外部进程，它不能直接用自己的 mmap 改写 VMM 地址空间，需要 VMM 映射执行通道或另选填充机制。COPY 与 file mapping 的收益及风险见[比较](../../03-design-comparison/05-copy-vs-shared-mapping.md)。具体请求名、FD 结构和握手尚未冻结。

尾页中真实数据之外的字节必须按设备格式定义处理；不能把未知、未下载的内容当作零。块设备候选则通过块请求完成通知恢复访问，不要求把这段 UFFD 流程照搬过去。

## 第二个 VM 会复用什么

内容服务可复用同一授权域中同一内容的缓存，并合并重叠下载。lazyd 应让两个运行会话引用稳定内容对象，通过共享任务等待同一缺失范围；单会话取消不影响其他等待者，导出文件在使用期间受引用保护。具体状态与验收见 [lazyd 架构](../../04-conch-design-reference/04-lazyd-responsibilities.md)。每个 VM 仍独立建立设备、地址映射和运行状态；不是 VM1 下载后 VM2 就自动拥有所有映射。

文件映射方案还可以复用同一 inode/offset 的干净物理文件页。是否优于 COPY，需要用下载量、私有页和 PSS 等指标比较。

## 验收清单

- 同镜像 full 与 lazy 启动执行相同业务，文件内容对拍。
- 分别测 prepare、guest ready、应用 ready 和首请求延迟。
- 冷缓存、热缓存、多 VM、高 RTT 下验证 pmem 主线；整盘设备对照按需开展，不作为本闭环前置。
- 同内容重复 prepare、一个等待者取消、空闲连接及超额请求不破坏其他会话的进度和数据。
- 取数失败、handler 退出、VMM 取消都能有限时间结束等待并上报。
- 单 VM 退出不破坏其他 VM 正在使用的共享内容。

设计依据见[决策 1、2、4、7](../../04-conch-design-reference/08-design-decisions-and-evidence.md)。
