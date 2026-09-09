# 快照谱系与一致恢复

> 阅读完成后，读者能够区分 rootfs 快照、可写磁盘增量和内存快照，理解 checkpoint/template 恢复为什么需要统一编排但不应强行使用同一种懒加载机制。

## 1. “快照”可能指三个不同对象

### 镜像/rootfs lower

它通常是 OCI 镜像中的只读层。容器生态也把 snapshotter 的 mount view 称为 snapshot，但它主要表达文件系统层关系，不包含 VM 运行内存。

### 可写磁盘快照

它保存 workload 对文件系统或块设备的修改。增量模式只记录相对父快照变更的块/文件，需要父子 lineage 才能恢复完整视图。

### VM 内存快照

它保存 guest RAM，并与 vCPU、设备和 VMM 状态共同构成可恢复 checkpoint。内存 diff 表示相对某个父 memory snapshot 的脏页，不是 rootfs 文件变化。

Firecracker 明确将 memory snapshot 与磁盘快照管理分开，磁盘一致性由使用者负责。[SRC-FC-001] E2B 也分别输出 memory diff 和 rootfs block diff。[SRC-E2B-001]

## 2. incremental 的准确含义

`incremental` 表示只保存相对父状态发生变化的部分：

```text
Base S0
  + Diff S1
    + Diff S2
      = Restore View S2
```

对不同对象，脏数据来源不同：

| 对象 | 增量单位 | 常见脏数据来源 |
| --- | --- | --- |
| rootfs writable disk | block/extent/file | COW bitmap、block backend、overlay upper |
| guest RAM | page/range | dirty bitmap、write protection、VMM tracking |
| VMM/device state | structured state | 设备模型序列化 |

“内存快照支持懒恢复”只表示恢复时不必预读所有 RAM 页；它不会自动让 rootfs layer 或 writable disk diff 具备懒加载。

## 3. checkpoint/template 恢复依赖图

一次可执行恢复至少要确认：

```text
Template/Checkpoint
  ├── VM config + device state
  ├── guest RAM lineage
  ├── immutable rootfs lower identities
  └── writable disk lineage
```

其中任何父对象缺失、版本不兼容或校验失败，恢复都不应进入“vCPU 已运行但资源永远取不到”的状态。

OpenSandbox 的 lifecycle 规范提供 snapshot 资源和 restore API，但公开规范本身没有证明底层采用哪种懒恢复机制。[SRC-OPENSANDBOX-001] 这说明 API 层的 snapshot 状态与数据层的 lazy paging 必须分开核实。

## 4. 对象可以并行准备，但启动门槛不同

### 路径 A：只读 rootfs lower

启动前需要可解析的文件系统视图和可服务的数据来源。通用机制包括文件、块或 pmem/DAX；本项目当前选择 EROFS + pmem/DAX 作为只读 rootfs 主线，其他机制作为对照，不是并列开发任务。

### 路径 B：可写 disk diff

启动前必须建立可读的 parent chain 和新的 writable head；具体 block 可以按需从本地或远端 parent 读取。

### 路径 C：guest RAM

启动前必须恢复关键 CPU/设备状态，并为 guest memory 建立可 fault 的地址空间；其余 RAM 页可通过 UFFD/post-copy 取回。

资源的就绪状态可以表达为统一状态机：

```text
Declared -> MetadataReady -> Faultable -> Running -> Materialized/Idle
                    \-> Failed
```

这里的 `Faultable` 泛指“无需全量物化也可服务访问”，不是所有对象都会产生 UFFD event。文件路径、块后端和 RAM handler 各有自己的完成条件。

## 5. 为什么需要统一资源管理器

统一管理的重点不是代替每个数据面，而是维护：

- checkpoint 到实际使用对象的引用，rootfs 已在磁盘视图中时不重复增加设备；
- 父子 lineage 和兼容版本；
- 各资源是否达到启动门槛；
- 并行准备与取消；
- 失败传播和 VM shutdown；
- lease/refcount 与 GC；
- 观测字段和恢复耗时分解。

在当前产品边界中，Conch 最适合承担这个编排角色。lazyd 可以扩展为多种 immutable/range 内容的供应服务，但不应因此接管 vCPU、设备状态或 sandbox 生命周期。

## 6. 业界样本给出的组合方式

### Firecracker

正式能力包括 memory file 的 `MAP_PRIVATE` 恢复和外部 UFFD handler；磁盘状态由调用者另行管理。[SRC-FC-001] [SRC-FC-002] 这证明内存懒恢复和 rootfs 懒加载可独立演进。

### Cloud Hypervisor

v53 发布了 UFFD demand-paged guest memory restore。[SRC-CH-001] 另一个把 PMEM 与 snapshot UFFD 协议组合的 PR #8239 已关闭未合入，review 意见要求先澄清两类功能的边界。[SRC-CH-002]

### QEMU/CRIU

QEMU Fast Snapshot Load 使用 postcopy 基础设施和 mapped-ram layout，在缺页时恢复 RAM 并后台加载。[SRC-QEMU-001] CRIU `lazy-pages` 对 Linux 进程地址空间执行相似的按需注页。[SRC-CRIU-001] 两者都不是容器 rootfs 供应器。

### E2B

E2B 同时使用 Firecracker UFFD memory、模板 memfile prefetch、只读 template rootfs、每 sandbox NBD COW cache，并分别导出内存和磁盘 diff。[SRC-E2B-001] 这是上层组合不同恢复机制的具体样本，但不证明必须同时存在独立镜像、独立磁盘和独立内存三个服务。

## 7. 从概念到三仓设计

Conch 协调恢复图、发布、取消和生命周期；lazyd 分清内容对象、来源、共享任务和运行会话，先服务镜像再按需扩展来源；StratoVirt 提供必要的设备、地址空间和 VM 恢复能力。类型适配器和 handler 的进程归属可以共同调整，复用内容不等于复用每台 VM 的恢复状态。

必须明确的是磁盘写入、内存脏页、不可变缓存 ready 的区别。协议和调度可以复用，不能因复用而丢失这些语义。具体候选见[一致恢复设计](../04-conch-design-reference/06-checkpoint-three-path-restore.md)。

## 8. 实现前先验证哪几件事

在 rootfs 冷启动与 checkpoint 恢复之间保留同一资源模型，但可以分阶段交付。先用小实验比较设备和 handler 方案，再完成冷启动闭环，随后验证增量视图与一致保存/恢复。

不预设必须保留某个 FETCH 版本或独立 memory 协议；依据最新上游和实验结果确定契约。执行次序见[开发路线](../04-conch-design-reference/07-phased-roadmap.md)。
