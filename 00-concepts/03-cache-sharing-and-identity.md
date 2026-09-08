# 缓存共享、内容身份与复用层级

> 阅读完成后，读者能够区分“没有重复下载”“复用同一缓存文件”和“共享同一物理文件页”，并能为镜像 layer、磁盘增量和内存快照选择正确身份。

## 1. 共享不是一个布尔值

两个 VM 使用同一份 rootfs 时，系统可能只实现了下面某一层复用：

| 层级 | 复用了什么 | 仍可能重复什么 |
| --- | --- | --- |
| 元数据复用 | manifest/index/bootstrap | blob range 下载、落盘、内存页 |
| 下载去重 | 同一 range 只访问一次远端 | 每 VM cache 文件和内存 copy |
| 持久缓存复用 | 同一 sparse/cache file | 每 VM 独立匿名 HVA 页 |
| host page cache 复用 | 同一 inode+offset 的 file page | 每 VM 的 GVA/GPA/HVA 和页表项 |
| writable snapshot 复用 | 同一只读 parent/lineage | 每 sandbox 私有写入 |

因此，“两个 VM 指向同一 EROFS 文件”只是共享的必要条件之一。若 handler 使用 `pread + UFFDIO_COPY`，每个 VM 仍会创建自己的匿名物理页；若每个 VM 映射同一 cache inode 和 offset，`MAP_PRIVATE` 的干净页和 `MAP_SHARED` 页都有机会引用同一 file-backed page；私有写入后才产生 COW 分离。

## 2. 三种对象需要三种身份

### 2.1 不可变镜像内容

首选 OCI blob digest 作为 canonical identity：

```text
cache identity = digest + format/version parameters
```

在授权域和格式解释兼容时，同一 digest 即使来自不同 tag、layer index 或 VM，也应具备复用同一内容缓存的条件。内容身份与每次运行的 attachment/lease 要区分；是否使用 instance_id 等字段名由新契约确定，不在概念层冻结。

Nydus、stargz 和 SOCI 都依赖内容摘要或索引中的 chunk/span 校验来建立可复用内容。[SRC-NYDUS-002] [SRC-STARGZ-002] [SRC-SOCI-002]

### 2.2 可写磁盘快照

可写层不能只用内容 digest 表达当前关系，还要记录父快照和写入世代：

```text
snapshot_id -> parent_snapshot_id -> immutable base
```

两个 sandbox 可以共享只读 parent，但当前 writable head 必须隔离。OverlayBD 的 lower/upper 模型和 E2B 的 template rootfs + per-sandbox COW cache 都体现了这一点。[SRC-OVERLAYBD-001] [SRC-E2B-001]

### 2.3 guest RAM 快照

内存页身份至少依赖：

- snapshot/template ID；
- memory range/page offset；
- 父快照或 diff lineage；
- VM 配置和设备状态兼容版本。

不可变内存快照块同样可以用 digest 缓存，但整份 guest RAM 的逻辑视图还需要父链、布局和版本，不能只用一个镜像层身份替代。QEMU mapped-ram、Firecracker memory file 和 Cloud Hypervisor restore 都把 memory snapshot 作为独立对象。[SRC-QEMU-001] [SRC-FC-001] [SRC-CH-001]

## 3. cache key 要防止什么

可靠 cache key 必须避免：

- 把 `:`、`/` 简单替换后造成不同 digest 字符串碰撞；
- 同 digest 但格式版本、加密域或压缩解释不同却错误共享；
- image tag 更新后仍把 tag 当不可变身份；
- sandbox 删除时误删仍被其他 VM 使用的内容缓存；
- bitmap 与 cache 文件来自不同 digest 或 unit size。

若采用 sparse file + bitmap，可以用 header 记录 magic、version、digest、blob size、unit size、slot count。其他索引格式也需要相应校验；这不是指定必须沿用某个文件格式。

## 4. range 状态与持久化顺序

缓存的 `ready` 是一个正确性承诺：对应范围的字节已完整且可供读取。若用持久 bitmap 记录跨重启有效的 ready，顺序必须是：

```text
write complete range
  -> cache durability barrier
  -> set bitmap ready
  -> bitmap durability barrier
```

否则崩溃后可能看到 `ready=1`，但 cache 中仍是洞、旧数据或部分写入。`SEEK_DATA/SEEK_HOLE` 只能说明是否分配了 extent，不能证明字节内容与 digest 相符。

清除 ready 也要有持久化顺序。已有消费者的文件映射不能在清除标志后就被随意覆盖，修复还必须遵守引用和不可变发布规则。同步频率需要按 fetch unit 批处理评估，但不能牺牲“ready 不早于 data”的约束。

## 5. 并发请求的 fan-out

多个 VM 同时 fault 同一 fetch unit 时，理想流程是：

```text
VM1 fault --\
VM2 fault ----> inflight[digest, range] -> 一次远端 fetch/校验/写盘
VM3 fault --/                              -> 向所有等待者 fan-out ready
```

这里的 fan-out 是把一次完成结果通知多个等待请求。它不要求主动给所有 VM 做映射：

- 当前 faulting VM 应立即收到 fd/range 并 remap；
- 其他已经阻塞于同一 range 的 VM 也应被唤醒或分别完成 remap；
- 尚未访问该 range 的 VM 可以等未来 fault，再命中 ready cache；
- 启动前查询已有缓存、prefetch 和 prefault 可以作为候选优化；三者分别影响内容获取、页驻留和映射，是否启用及何时验证由实验决定。

## 6. FD + 文件映射怎样复用物理页

### copy 路径

```text
cache file -> pread/user buffer -> UFFDIO_COPY -> VM1 anonymous page
                                      \-------> VM2 anonymous page
```

优点是语义直接、UFFD resolution 成熟。代价是每个 VM 都保留独立匿名页，并发生额外 copy。

### 文件映射路径

```text
同一 cache inode + offset
  -> VM1 HVA file-backed VMA
  -> VM2 HVA file-backed VMA
  -> host page cache 中同一文件页
```

它省掉 cache 到匿名页的复制，并允许 host file page 共享。每个 VM 的 HVA 地址和页表仍独立；共享的是映射背后的 file page，不是“共享一段 HVA”。Nydus 合入的 Zerocopy 路径提供了最直接的参考。[SRC-NYDUS-004]

## 7. 生命周期必须与身份匹配

内容级 cache 的生命期不能绑定单个 VM：

- VM stop/delete 只释放自己的映射和引用；
- 没有可靠引用管理前，不应由单 VM cleanup 删除共享内容；
- cache GC 应考虑引用、最近使用、磁盘水位和恢复中的 in-flight；
- 磁盘和内存的父链必须分别正确表达，可以纳入同一个带类型的引用图；
- 强制回收前必须阻止新映射并等待现有 fd/VMA 生命周期闭合。

urunc 的共享 snapshot view 原型还给出一个重要反例：对小镜像增加共享层和 lease 准备成本，可能比直接准备更慢。[SRC-URUNC-001] 因而共享策略需要按对象大小、复用概率和启动关键路径做基准，而不是默认层级越多越好。

## 8. 对设计真正有约束力的是什么

应保留的是内容正确性、授权、身份区分、写入隔离和安全回收，而不是某个既有字段名或缓存文件名。Conch 记录运行和快照引用，lazyd 提供内容/缓存能力，StratoVirt 管理地址与设备；三者的接口要共同设计。

候选拆分和多 VM 验证见[决策依据](../04-conch-design-reference/08-design-decisions-and-evidence.md)。
