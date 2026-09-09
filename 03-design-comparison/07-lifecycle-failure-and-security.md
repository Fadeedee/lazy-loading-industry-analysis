# 生命周期、失败与安全比较

> 阅读完成后，读者能够审查一个懒加载方案在 daemon 崩溃、网络失败、缓存损坏、多租户共享和资源回收时是否具有完整处理路径。

本页列出我们的设计检查项，不代表下面所有策略都已在被调研产品中实现。具体来源和差异见[数据服务横向比较](08-data-service-architecture.md)；当前实现主线为 pmem，块/文件入口仅用于横向比较或后续快照讨论。

## 启动屏障

vCPU/工作负载运行前，至少满足：

- resource descriptor 与 identity 已校验；
- source/cache service 可访问；
- HVA/memslot/UFFD 或 block/FUSE backend 已注册；
- handler event loop 已运行；
- guest device/mount metadata 完整；
- 上层已登记失败回调和生命周期引用。

## 失败矩阵

| 失败 | 错误风险 | 所需策略 |
| --- | --- | --- |
| registry timeout/401 | fault 长时间阻塞或错误复用别人的凭据 | deadline、授权范围内刷新 token、有界 retry；凭据更新不替换公共内容状态 |
| 校验不匹配或已提交缓存损坏 | guest 读错误内容 | 未发布数据不得标 ready；已映射对象隔离并报告受影响会话，清 bitmap 不等于撤销映射，不原地截断/重写 |
| partial cache write / sync 失败 | bitmap 误报 ready | 完整写入并检查、cache barrier、bitmap ready/barrier、再通知；失败不给可读承诺 |
| 内容服务或适配器崩溃 | 正在等待的访问无法完成 | 有界恢复或 VM fail，不静默等待 |
| 内部/外部 handler 退出 | vCPU 永久阻塞 | VMM/Conch 监控和明确停止策略 |
| 单 VM 取消，共享任务还有等待者 | 另一台 VM 被连带中断 | 取消本会话等待；公共任务保留或按剩余需求调整优先级 |
| 队列、缓冲或磁盘容量耗尽 | 无界占用挤死健康请求 | 连接/任务/字节预算、前台优先、背压或明确拒绝，不靠无限线程补救 |
| 重复 prepare / 区域重建后收到旧结果 | 重复下载或把数据填进错误区域 | 内容对象稳定，事件和结果校验会话/区域代次 |
| fd/range 越界 | SIGBUS/越权映射 | file size、offset、len、pmem bounds 校验 |
| VM delete | 误删共享 cache | release VM reference，不直接删 content |
| snapshot parent GC | child 无法恢复 | lineage lease/refcount |

Firecracker 官方文档明确提示 UFFD handler 不处理 fault 会导致 VM 挂起。[SRC-FC-002]

## 通知、完成与关闭不是同一个状态

内容任务失败也必须通知等待者，所以收到通知后应检查结果，不能直接认为 ready。Nydus 的 pending 清理会唤醒 Slot 等待者，等待者仍检查底层 ready map；这是任务状态与内容状态分离的具体例子。[SRC-NYDUS-009] [SRC-NYDUS-010]

对我们的方案，内容 ready、某个 VMM 完成映射、会话不再发请求、映射已释放、对象可以回收分别判断。E2B NBDProvider 的关闭过程也说明中间通知不能代替最终清理结果，但其私有写缓存流程不是共享文件回收的直接实现。[SRC-E2B-004]

因此 Conch 负责操作最终状态与引用，lazyd 负责共享任务及内容保护，StratoVirt/所选 handler 负责访问完成和区域退出。取消、错误与重启都需在这些边界间闭环，不能只给某个 FETCH handler 加一个 timeout。

## 安全边界

- UDS 应限制路径权限并验证 peer credentials；
- `SCM_RIGHTS` 接收的 fd 必须检查文件类型、大小和只读语义；
- 外发 FD 与内部写缓存句柄分开打开；dup 不会降权。FD 通常能访问整个文件，range JSON 不是子区间权限令牌；
- 内容/视图 ID 只用于 lookup，不能替代校验和授权；
- 跨租户缓存复用必须同时满足内容不可变、授权和隔离要求；秘密内容需专门安全策略，不能仅凭 digest 相同跨域共享；
- 从同一 memory snapshot 克隆时更新随机数、网络身份和凭据；
- 只读映射必须验证 KVM 写入处理和 host VMA 权限；不能假设设备声明等于实际写保护；
- cache 目录不能由 guest 或不可信进程写入。

Firecracker 对 snapshot clone 和 pmem 共享都有专门安全提示。[SRC-FC-001] [SRC-FC-003]

## GC 原则

```text
content object lease
  <- image/snapshot reference
  <- running VM mapping/fd
  <- in-flight fetch/remap
```

运行引用与持久 checkpoint 引用需要区分：可重新获取的缓存可以在无活跃读取/映射并满足策略时驱逐，但被 checkpoint 引用的唯一持久来源不能直接删除。进行中的 fetch、恢复和映射必须受到保护；没有可靠引用管理前采用保守保留策略。

关闭服务端 FD、断开 socket 或租约超时，不会撤销 VMM 已有映射。先停止新请求、协调待完成操作和映射释放，再解除对应保护；服务重启后未完成使用关系协调前，不把空的内存任务表当作“无人使用”。详细规则集中在[lazyd 会话与回收](../04-conch-design-reference/04-lazyd-responsibilities.md)。

## 版本升级

缓存格式、数据协议、准备记录和 snapshot lineage 都要定义版本与兼容规则。字段名和存储位置可以共同设计，不预设既有 lazy-rootfs.json 是新接口。升级需明确旧格式重建/拒绝、版本协商及运行实例迁移；公共协议也必须保留文件、磁盘、RAM 的类型语义。
