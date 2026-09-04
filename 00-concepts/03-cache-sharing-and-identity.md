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

因此，“两个 VM 指向同一 EROFS 文件”只是共享的必要条件之一。若 handler 使用 `pread + UFFDIO_COPY`，每个 VM 仍会创建自己的匿名物理页；若每个 VM 都 `MAP_SHARED` 同一 cache inode 和 offset，host kernel 才有机会让它们引用同一 file-backed page。

## 2. 三种对象需要三种身份

### 2.1 不可变镜像内容

首选 OCI blob digest 作为 canonical identity：

```text
cache identity = digest + format/version parameters
```

同一 digest 即使来自不同 tag、image_ref、layer index 或 VM，也应指向同一 sparse cache、bitmap 和 prepared content instance。`instance_id` 可以保留为协议字段，但语义应是 opaque prepared-content identity，而不是 sandbox identity。

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

不能拿 OCI layer `instance_id` 直接标识 guest RAM。QEMU mapped-ram、Firecracker memory file 和 Cloud Hypervisor restore 都把 memory snapshot 作为独立对象。[SRC-QEMU-001] [SRC-FC-001] [SRC-CH-001]

## 3. cache key 要防止什么

可靠 cache key 必须避免：

- 把 `:`、`/` 简单替换后造成不同 digest 字符串碰撞；
- 同 digest 但格式版本、加密域或压缩解释不同却错误共享；
- image tag 更新后仍把 tag 当不可变身份；
- sandbox 删除时误删仍被其他 VM 使用的内容缓存；
- bitmap 与 cache 文件来自不同 digest 或 unit size。

建议 cache 目录中保留可验证 header：magic、version、digest、blob size、unit size、slot count。打开已有 cache 时先校验 header，不只信任路径名。

## 4. range 状态与持久化顺序

bitmap 的 `ready` 是一个正确性承诺：对应 cache range 已完整写入并可被映射。持久化顺序必须是：

```text
write complete range
  -> cache durability barrier
  -> set bitmap ready
  -> bitmap durability barrier
```

否则崩溃后可能看到 `ready=1`，但 cache 中仍是洞、旧数据或部分写入。`SEEK_DATA/SEEK_HOLE` 只能说明是否分配了 extent，不能证明字节内容与 digest 相符。

清除 ready 也要先让 bitmap 的非 ready 状态可靠可见，再允许修复或覆盖数据。同步频率需要按 fetch unit 批处理评估，但不能牺牲“ready 不早于 data”的约束。

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
- 启动时可选的 PROBE/prefault 可以提前映射已有 ready range，但不是 MVP 正确性的前提。

## 6. 为什么 FD + shared mapping 更适合跨 VM 文件页复用

### copy 路径

```text
cache file -> pread/user buffer -> UFFDIO_COPY -> VM1 anonymous page
                                      \-------> VM2 anonymous page
```

优点是语义直接、UFFD resolution 成熟。代价是每个 VM 都保留独立匿名页，并发生额外 copy。

### shared mapping 路径

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
- 没有 lease/refcount 前，不应由单 VM cleanup 删除共享 lazyd instance；
- cache GC 应考虑引用、最近使用、磁盘水位和恢复中的 in-flight；
- writable snapshot 和 memory snapshot 使用独立引用图；
- 强制回收前必须阻止新映射并等待现有 fd/VMA 生命周期闭合。

urunc 的共享 snapshot view 原型还给出一个重要反例：对小镜像增加共享层和 lease 准备成本，可能比直接准备更慢。[SRC-URUNC-001] 因而共享策略需要按对象大小、复用概率和启动关键路径做基准，而不是默认层级越多越好。

## 8. 对当前项目的直接结论

- lazyd 以 digest 为 immutable layer 的 canonical identity，并负责 inflight 去重和 ready fan-out。
- Conch 记录 sandbox 到 prepared content、disk snapshot、memory snapshot 的引用关系。
- StratoVirt 把 `instance_id` 当 opaque ID，只校验协议和映射范围。
- `mmap(MAP_SHARED | MAP_FIXED)` 面向 host file page 复用；是否产生实际 RSS/PSS 收益必须用多 VM 基准验证。
- lease/refcount 可以后续引入，但在此之前 cleanup 策略必须偏保守，避免误删共享 cache。
