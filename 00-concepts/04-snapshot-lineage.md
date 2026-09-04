# 快照谱系与三路恢复

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

## 4. 三路可以并行，但启动门槛不同

### 路径 A：只读 rootfs lower

启动前只需取得 descriptor、文件系统元数据及足够的 cache/bitmap 状态；文件数据可在 guest 访问时通过 pmem/DAX fault 获取。

### 路径 B：可写 disk diff

启动前必须建立可读的 parent chain 和新的 writable head；具体 block 可以按需从本地或远端 parent 读取。

### 路径 C：guest RAM

启动前必须恢复关键 CPU/设备状态，并为 guest memory 建立可 fault 的地址空间；其余 RAM 页可通过 UFFD/post-copy 取回。

三条路径的就绪状态可以表达为统一资源状态机：

```text
Declared -> MetadataReady -> Faultable -> Running -> Materialized/Idle
                    \-> Failed
```

但 `Faultable` 的技术含义不同：rootfs 是 lazyd 能供应内容并且 pmem HVA 已注册；disk 是 block parent 可读；RAM 是 snapshot page source 和 UFFD handler 已就绪。

## 5. 为什么需要统一资源管理器

统一管理的重点不是代替每个数据面，而是维护：

- checkpoint 到三类对象的引用；
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

E2B 同时使用 Firecracker UFFD memory、模板 memfile prefetch、只读 template rootfs、每 sandbox NBD COW cache，并分别导出内存和磁盘 diff。[SRC-E2B-001] 它是“三路分离、上层统一编排”的最完整公开样本之一。

## 7. 对 Conch + lazyd + StratoVirt 的建议模型

```text
Conch Restore Coordinator
  ├── Rootfs resource -> lazyd -> StratoVirt pmem/UFFD -> guest EROFS+DAX
  ├── Disk lineage    -> block backend -> guest writable filesystem
  └── Memory lineage  -> StratoVirt snapshot/UFFD restore
```

职责建议：

- **Conch**：资源图、策略、启动门槛、并行度、取消、失败状态和生命周期。
- **lazyd**：不可变内容/范围身份、远端读取、校验、cache 和并发去重。
- **StratoVirt**：两条明确分离的 UFFD consumer：pmem rootfs fault 与 guest RAM restore；可复用底层 UFFD helper，但不要模糊协议语义。
- **guest/guestd**：稳定设备映射、EROFS+DAX lower 和 writable upper 组装。

## 8. 分阶段实现顺序

1. 先稳定 cold rootfs lazy start：验证内容正确性、完整失败传播和多 VM cache/page 复用。
2. 将 writable upper/block snapshot 作为独立资源纳入 Conch lineage，不改变 rootfs FETCH v1 协议。
3. 接入 StratoVirt 已有或新增的 guest RAM lazy restore，定义独立 memory source 协议。
4. 最后实现 checkpoint/template 三路并发恢复、统一 lease/GC 和跨节点分发。

这样可以从第一阶段开始保留最终资源模型，又不会让未验证的三种数据面在一个大协议中互相耦合。
