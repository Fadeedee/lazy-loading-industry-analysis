# 懒加载业界调研总览与 Conch 设计结论

> 核对时间：2026-09-04。本文用于快速评审；事实依据、成熟度和更细的数据路径分别见产品文档与附录。

## 一页结论

1. **不存在一条数据面同时适合所有懒加载对象。** 只读 rootfs lower、可写磁盘 upper、guest RAM 的 identity、触发入口和完成动作不同，应统一编排状态，不应强行统一协议。
2. **Conch 的 rootfs 路线仍然成立。** EROFS+DAX 让 guest 文件访问落到 pmem GPA；StratoVirt 通过 HVA missing fault 触发 UFFD；lazyd 按 digest 准备 range 并返回 cache FD；StratoVirt 固定地址 remap 后唤醒 vCPU。
3. **快照恢复应从第一版就预留三路资源图。** rootfs lower 走 lazyd，writable ext4 upper 走块设备/COW，guest RAM 走 memory snapshot page source。Conch 负责统一 `Faultable` 屏障、失败传播和引用生命周期。
4. **应借鉴 Nydus 的实现证据，但不能把其实验协议原样搬入。** Nydus UFFD block service 已合入，证明 Copy 与 FD+`MAP_FIXED` 两种填充路径具有工程实现基础；其当前 zerocopy 默认使用 `MAP_PRIVATE|MAP_FIXED`，而我们仍需以本地 PSS、只读和多 VM 安全性决定 `MAP_SHARED` 还是 `MAP_PRIVATE`。[SRC-NYDUS-004]
5. **Firecracker #5740 只能作为设计来源，不能作为已落地能力。** 该 issue 描述 image service、FD passing 和 fixed remap，但仍是 proposal。[SRC-FC-004]
6. **内存懒恢复已有更成熟参照。** Firecracker external UFFD、Cloud Hypervisor v53、QEMU Fast Snapshot Load、CRIU lazy-pages 和 E2B 均展示了 fault-first、后台填充或 page source 分离；它们不能替代 rootfs 懒加载，但能指导 handler readiness、优先级和失败策略。[SRC-FC-002] [SRC-CH-001] [SRC-QEMU-001] [SRC-CRIU-001] [SRC-E2B-001]

## 先看三类对象

| 对象 | 典型格式 | 共享身份 | 触发入口 | 填充/完成动作 | 推荐 owner |
| --- | --- | --- | --- | --- | --- |
| 只读 rootfs lower | OCI EROFS layer | blob digest | VFS/FUSE 或 DAX HVA fault | cache read、FD remap、wake | lazyd + StratoVirt |
| 可写磁盘 upper | ext4/COW block layers | snapshot lineage + writable head | virtio-blk/NBD block request | parent read、private COW write | block backend |
| guest RAM | full/incremental memory snapshot | snapshot ID + base/delta lineage | RAM HVA fault | page copy/pwrite/mapping + wake | memory source + VMM |

三路可以在一次 checkpoint template restore 中并行，但不能共用一个 `instance_id`：

- rootfs `instance_id` 是 prepared immutable content identity；
- writable disk identity 是 parent chain 与 private head；
- memory identity 是 full/incremental snapshot lineage；
- sandbox/VM ID 只表示一次运行实例。

## 业界路线图

### 文件语义路径

Nydus v2、stargz 和 SOCI 在文件访问层保留 pathname/inode 语义，适合容器 rootfs 与按文件预取。它们通常用 FUSE/virtiofs 捕获 read，再把文件偏移转换为 chunk/span range。[SRC-NYDUS-001] [SRC-STARGZ-001] [SRC-SOCI-002]

优点是文件级观测和预取自然；代价是 guest/VMM 的 DAX 共享映射不是其默认主路径。

### 块语义路径

OverlayBD 和 E2B NBD COW 把对象暴露为块设备，天然支持 ext4、随机读写和增量块层。[SRC-OVERLAYBD-001] [SRC-E2B-001]

这条路线适合可写 upper，不应反过来要求只读 EROFS lower 也走块请求；否则会丢失我们希望利用的 DAX/page-cache 共享路径。

### DAX/UFFD 固定映射路径

Nydus UFFD block service、Firecracker #5740 方案和 Cloud Hypervisor #8239 原型都把 VMM 的缺页地址转换为镜像 offset，由外部服务返回数据或 FD，再通过 Copy 或 fixed remap 完成 fault。[SRC-NYDUS-004] [SRC-FC-004] [SRC-CH-002]

这是 Conch rootfs 最接近的参考路径。核心正确性条件是：

1. vCPU 可访问前，anonymous/reserved HVA、KVM memslot、UFFD registration 和 fault loop 必须全部 ready；
2. lazyd 先写完整 range 并完成 durability barrier，再把 bitmap 标成 ready；
3. StratoVirt 校验 FD、file size、offset、alignment、`pmem_size` 后才 remap；
4. remap/zeropage 后明确 wake；失败必须超时并进入 VMM fatal shutdown，而不是静默阻塞 vCPU；
5. 相邻尚未 remap 的 VMA 区域继续受 UFFD missing 管理。

### 内存快照路径

Firecracker、Cloud Hypervisor、QEMU、CRIU 和 E2B 都把恢复延迟从“恢复全部内存后运行”改成“先提供 faultable source，再按需补页”。[SRC-FC-001] [SRC-FC-002] [SRC-CH-001] [SRC-QEMU-001] [SRC-CRIU-001] [SRC-E2B-001]

共同经验：

- faulting page 优先于后台预取；
- handler/source readiness 是 resume 前置条件；
- 页面索引、snapshot lineage 和 VM 私有内存不能复用 rootfs digest 语义；
- 后台 materialization 可缩短后续 fault，但必须受带宽和并发预算限制。

## 对当前三仓的直接结论

### Conch

Conch 应成为资源图和恢复事务 owner：

- 从 containerd Boot Index/Template/Sandbox store 选择 rootfs、disk、memory 和 VM assets；
- 把 rootfs EROFS descriptors 交给 lazyd，不自己解释 bitmap；
- 保存 prepared rootfs 的稳定引用和 GC closure；
- 启动 StratoVirt 时传 typed source，不拼接“sparse file 就是普通 pmem backend”的错误语义；
- 等 rootfs、disk、memory 三路都达到 `Faultable` 后再恢复 vCPU；
- 任一路 fatal 时取消整个 restore，并按引用关系清理本次运行资源。

当前 Conch 已经由 containerd 原生 store 管理 Boot Index、Template 和 Sandbox，因此不建议再引入一套独立 snapshotter metadata owner。[SRC-CONCH-001] [SRC-CONCH-002]

### lazyd

lazyd 应继续收敛为不可变内容服务：

- canonical key 使用严格校验后的 OCI digest；
- descriptor-based prepare，不解析 Conch 专有镜像编排；
- 管 sparse EROFS cache、bitmap、range amplification、inflight fan-out、registry auth 和 FD export；
- `ready=true` 是持久化承诺，必须晚于 cache data sync；
- 对多个 VM 返回同一内容文件的只读 FD，让 host page cache 成为共享物理页来源；
- VM 退出只释放 lease/reference，不直接删除共享 cache。

现有实现可复用，但在产品化前应补：只读 FD 导出、条件变量/通知式 inflight fan-out、可信 recovery 校验、credential provider/rotation、lease/refcount 与 GC。[SRC-LAZYD-001]

### StratoVirt

StratoVirt 应只负责 VMM 地址空间和 fault completion：

- lazy pmem 与普通 file-backed pmem 使用不同 backend；
- 每个 VM 建立独立 HVA、GPA/memslot、UFFD registration 和 VMA；
- 将 `fault_hva - base_hva` 转换为内容 offset；
- 向 lazyd FETCH 并通过 SCM_RIGHTS 接收 cache FD；
- 对完整 ready range 执行 checked fixed remap，而非只处理单个 4 KiB fault page；
- padding 走 zeropage/wake；
- timeout、protocol error、mmap error 上升为明确的 VMM shutdown reason。

旧 `lazy-pmem-full` 分支只应作为算法与测试证据，不能直接 rebase 到当前 dev；它落后且横跨旧 abstraction。[SRC-SV-001] [SRC-SV-003]

## 快照恢复目标形态

```text
Checkpoint Template / Boot Index
  ├── PreparedRootfs       -> lazyd -> EROFS cache FD -> pmem/DAX fault
  ├── WritableDiskSnapshot -> block parent + private COW head -> blk I/O
  ├── MemorySnapshot       -> full/incremental source -> RAM UFFD fault
  └── VM assets            -> kernel + initrd + VMM/device state

Conch prepare all
  -> Rootfs Faultable
  -> Disk Faultable + Writable
  -> Memory Faultable
  -> VMM handlers ready while paused
  -> Resume vCPU
```

Conch PR #155 与 StratoVirt PR #2017 若按当前方向合入，可提供 incremental memory 的 memfd/attachment 基础，但不会自动解决 rootfs：memory source 向 VM 私有 memfd 写页，而 rootfs source 返回不可变 cache FD 并建立 file-backed VMA。[SRC-CONCH-003] [SRC-SV-002]

## 采用、适配、拒绝

**直接采用：** digest identity、descriptor prepare、fault-first priority、handler-before-resume、data-before-ready、FD/range 全校验、内容 cache 与 VM 生命周期分离。

**按当前代码适配：** Nydus UFFD zerocopy、stargz/Nydus prefetch、OverlayBD/E2B block COW、Cloud Hypervisor/QEMU memory restore、Kata snapshot metadata。

**明确拒绝：** 重放旧大分支、fake snapshot、sparse EROFS 整体 file mmap、让 lazyd 解析 Boot Index、把三类数据塞进同一协议、无 timeout 的 fault path、把 proposal/open PR 描述成发布能力。

完整清单见 [采用、适配与拒绝](04-conch-design-reference/02-adopt-adapt-reject.md)。

## 分阶段落地

1. **协议和对象边界冻结**：定义 `PreparedRootfs`、typed VMM source、状态与 fatal reason；不先写数据面。
2. **lazyd 收敛**：只读 FD、持久化顺序、inflight fan-out、lease/refcount 和 recovery contract。
3. **StratoVirt 当前 dev 原生实现**：lazy backend、UFFD、FETCH、checked remap/wake、shutdown channel。
4. **Conch 原生 store 接入**：prepare/publish metadata、启动参数、资源引用和 cleanup。
5. **guest/guestd 接入**：稳定 pmem ID、`EROFS ro,dax`、overlay lower 顺序。
6. **checkpoint 三路恢复**：rootfs + block COW + incremental RAM 的统一 readiness barrier。
7. **性能和故障验收**：冷启动、首触发、P50/P99、下载放大、PSS、双 VM 共享、daemon/VMM/registry 故障注入。

## 尚需实验决定

网络感知预取已纳入后续设计：保持 bitmap 基础单位固定，由 lazyd 根据近期延迟、吞吐、连续访问和预取利用率调整后台窗口；当前缺页范围就绪立即返回，首次请求慢不会直接触发扩大下载。该能力尚未实现，先验证有界顺序预取，再加入自适应反馈，详见 [缓存、去重与预取比较](03-design-comparison/06-cache-dedup-and-prefetch.md)。

| 决策 | 必测指标 |
| --- | --- |
| `MAP_PRIVATE` 或 `MAP_SHARED` | 双 VM PSS、page cache 共享、写保护、VMA 行为 |
| fetch unit 默认值 | 首 fault 延迟、请求数、下载放大、顺序读吞吐 |
| 小镜像 full-fetch 阈值 | prepare 时间、运行期 fault 数、总字节 |
| metadata prefetch/PROBE | 启动收益、额外下载量、失败复杂度 |
| 每 layer 一个 pmem 或预合并 artifact | 设备数量、mount/overlay 成本、cache sharing |
| cache GC 策略 | 命中率、磁盘水位、lease 泄漏恢复 |

## 图形入口

- [业界懒加载机制全景（HTML）](assets/overview/industry-panorama.html)
- [四类触发与完成路径（HTML）](03-design-comparison/assets/mechanism-paths.html)
- [Conch + lazyd + StratoVirt 目标职责（HTML）](04-conch-design-reference/assets/target-responsibilities.html)
- [冷启动 Rootfs 懒加载时序（HTML）](05-flows/01-cold-rootfs-lazy-start/assets/cold-rootfs.html)
- [Checkpoint Template 三路懒恢复（HTML）](05-flows/03-checkpoint-template-restore/assets/checkpoint-three-path.html)
- [内容缓存生命周期（HTML）](04-conch-design-reference/assets/content-cache-lifecycle.html)

## 继续阅读

- 产品成熟度：[成熟度矩阵](appendix/maturity-matrix.md)
- 事实和版本：[来源清单](appendix/source-inventory.md)
- 对比分析：[设计比较](03-design-comparison/)
- 分仓设计：[Conch 设计参考](04-conch-design-reference/)
- 术语核对：[术语表](appendix/glossary.md)
