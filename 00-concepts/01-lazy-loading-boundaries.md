# 懒加载的对象与边界

> 阅读完成后，读者能够判断一次“懒加载”究竟延后了下载、落盘、映射还是内存恢复，并能区分 rootfs lower、可写磁盘和 guest RAM 的不同数据语义。

## 1. 懒加载不是一种固定技术

“懒加载”只描述一个共同策略：**先让工作负载具备继续执行的最低条件，剩余数据在真正需要时再提供**。它没有规定触发入口、缓存格式或填充方式。

一个完整访问通常包含以下阶段：

```text
远端内容 -> 本地持久化数据 -> 内核文件页/块缓存 -> 进程或 VM 地址映射 -> CPU 读取
```

不同方案延后的阶段不同：

| 延后的动作 | 第一次访问时补什么 | 典型方案 |
| --- | --- | --- |
| 下载 | 从 registry/object storage 获取 chunk 或 range | Nydus、stargz、SOCI、OverlayBD |
| 本地物化 | 将内容写入 sparse file、块缓存或 CacheFiles | Nydus cache、OverlayBD cache、EROFS on-demand |
| 文件页驻留 | 从本地文件读入 host page cache | 普通 file-backed `mmap`、Firecracker snapshot `MAP_PRIVATE` |
| 地址映射 | 把已就绪文件范围替换到 faulting HVA | Nydus UFFD block service、Firecracker #5740 提案 |
| 内存恢复 | 从内存快照或远端 page server 注入 guest/process RAM 页 | Firecracker UFFD、Cloud Hypervisor v53、QEMU、CRIU |

因此，“出现缺页”不等于“需要联网”。本地 cache 已经有数据时，缺页可能只需要建立 file-backed 映射，或者让内核把文件页 fault-in。

## 2. 三类必须分开的数据对象

### 2.1 只读 rootfs lower

它来自 OCI 镜像，是按 digest 标识的不可变内容。适合按文件、chunk、span 或块范围拉取，并在不同 VM/容器之间共享。

典型链路：

```text
OCI descriptor
  -> 只读镜像格式及索引
  -> 本地内容缓存
  -> guest 文件系统 lower
```

Nydus、stargz 和 SOCI 都属于这一类，但索引格式和请求入口不同。[SRC-NYDUS-001] [SRC-STARGZ-001] [SRC-SOCI-001]

### 2.2 可写磁盘或 upper

它记录 workload 运行后产生的写入。内容不再只由原始镜像 digest 决定，而与 sandbox、时间点和父快照有关。它通常需要 COW、块级脏数据跟踪和 snapshot lineage。

典型链路：

```text
共享只读 lower + sandbox 私有 upper
  -> 运行时写入
  -> 增量磁盘快照
  -> 下次恢复时叠加父子层
```

OverlayBD 同时建模只读 lower 与可写 upper；E2B 使用只读模板 rootfs 加每 sandbox 的 NBD COW cache，并在暂停时导出磁盘 diff。[SRC-OVERLAYBD-001] [SRC-E2B-001]

### 2.3 guest RAM

它是某一运行时刻的 CPU/设备状态所依赖的内存内容。RAM 页需要由内存视图和逻辑页位置定位；底层快照对象也可以按内容 digest 寻址，但单个对象 digest 不能替代整份内存视图。恢复正确性还依赖 vCPU 状态、设备状态和快照兼容性。

典型链路：

```text
VM 状态 + memory snapshot
  -> 预建 guest memory mapping
  -> 恢复 vCPU
  -> fault 时补 RAM page
```

Firecracker、Cloud Hypervisor、QEMU 和 CRIU 的懒恢复都属于这一类。[SRC-FC-002] [SRC-CH-001] [SRC-QEMU-001] [SRC-CRIU-001]

## 3. 同一次恢复为什么可能有三条路径

checkpoint/template 恢复可能同时需要：

| 对象 | 共享边界 | 首次访问触发 | 常见填充 |
| --- | --- | --- | --- |
| 镜像 lower | 同一内容 digest | VFS、块 I/O、DAX fault | range fetch + cache/mmap |
| 可写 upper/disk diff | 同一快照 lineage | block I/O | COW block cache/remote block |
| guest RAM | 同一 memory snapshot | UFFD/page fault | copy page 或映射 snapshot file |

它们可以统一编排，并共用内容供应与带类型的协议，但不能丢掉各自的写入、继承和完成语义。E2B 将 Firecracker UFFD memory 与 NBD COW rootfs 配合使用，并分别导出 diff；其 rootfs 在磁盘路径中，不是另加一条 pmem 路径。[SRC-E2B-001]

## 4. 四个容易混淆的“尚未就绪”

1. **远端未下载**：本地没有请求范围的可信字节，需要网络读取和内容校验。
2. **文件存在稀疏洞**：路径和逻辑长度存在，但该范围没有落盘；直接读取可能得到零，不能据此判断真实内容就是零。
3. **文件页未驻留**：文件字节已经在本地，host page cache 尚无该页；内核可以从存储 fault-in。
4. **anonymous HVA 缺页**：VMM 已预留虚拟地址，但尚无匿名物理页或 file-backed VMA 覆盖；UFFD 可以接管此 missing fault。

例如，pmem/UFFD 候选会把“远端未下载”与“映射尚未填充”连接起来；块设备候选则在块请求中解决远端缺失。不能因为两者都需要内容缓存，就假定它们有相同的 fault handler。

## 5. 判断一个方案是否真的“懒”

应分别测量，而不是只看 VM 启动耗时：

- 启动前下载的远端字节数；
- 启动前实际占用的本地磁盘块；
- 首次业务请求前恢复的 guest RAM 页数；
- 首次 fault 的网络、校验、写盘和映射延迟；
- 后台预取是否最终下载全部内容；
- 第二个 VM 是否复用下载、磁盘数据和 host file page。

“先启动、后台立即全量下载”仍然减少了关键路径，但它和真正的长期按需物化不是同一种资源行为。

## 6. 用这些概念读我们的设计

先判断一次用户操作需要哪些对象，再比较设备、内容供应和恢复方式。三类语义不等于三个固定进程，也不预先指定各仓库只能修改哪一层。

下一步看[三仓联合设计](../04-conch-design-reference/08-design-decisions-and-evidence.md)；需要解释术语时看[术语表](../appendix/glossary.md)。
