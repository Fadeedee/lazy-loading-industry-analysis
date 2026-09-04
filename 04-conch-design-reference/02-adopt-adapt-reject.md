# 采用、适配与拒绝清单

> 阅读完成后，读者能够把业界调研转成明确工程决策，区分可直接采用的原则、必须按现有代码适配的机制和不应引入的路线。

## 直接采用

| 决策 | 依据 |
| --- | --- |
| rootfs、writable disk、guest RAM 三种对象独立建模 | E2B、Firecracker、QEMU |
| immutable cache 以 digest 为 identity | Nydus、stargz、SOCI |
| descriptor-based prepare，image selection 留在 orchestrator | SOCI 与当前 lazyd 边界 |
| fault request 优先于 background prefetch | QEMU、CRIU、E2B |
| UFFD handler/source 必须早于 vCPU ready | Firecracker、Nydus UFFD |
| data durable 后才能标记 ready | cache correctness 基线 |
| SCM_RIGHTS FD、range/file/alignment 全校验 | Nydus UFFD、SV 旧原型 |
| 单 VM delete 不删除共享 content cache | 内容身份与多 VM 共享 |
| snapshot API 与 lazy data path 分开验收 | OpenSandbox 反例 |

来源：[SRC-E2B-001] [SRC-FC-002] [SRC-NYDUS-004] [SRC-QEMU-001] [SRC-SOCI-002] [SRC-OPENSANDBOX-001]

## 需要适配

| 上游经验 | 当前项目适配 |
| --- | --- |
| Nydus UFFD Zerocopy | 保留 lazyd FETCH v1，按 StratoVirt abstraction 重写，不复制协议 |
| Nydus `MAP_PRIVATE` | 评估 `MAP_SHARED`/`MAP_PRIVATE` 与 readonly memslot，做 PSS/安全测试 |
| stargz/Nydus file prefetch | 从 EROFS metadata/trace生成 range hint |
| OverlayBD/E2B block COW | 只用于 ext4 writable upper，不替换 readonly pmem lower |
| CH/QEMU memory restore | 复用 UFFD helper，不复用 rootfs content identity |
| Kata snapshotter metadata | 映射到 Conch Boot Index/content store，不引入第二套 snapshotter owner |
| Firecracker external handler | StratoVirt rootfs handler内置，lazyd只做 content service |

## 明确拒绝

- 不直接 rebase 旧 Conch/StratoVirt lazy 分支。
- 不创建 fake committed snapshot 表示尚未 unpack 的 lazy rootfs。
- 不把 sparse EROFS cache 整体作为普通 `memory-backend-file`。
- 不把 fscache on-demand 作为 Linux 6.12+ 新系统唯一依赖。[SRC-EROFS-004]
- 不让 lazyd 解析 Conch Boot Index 或选择 rootfs component。
- 不用同一个 `instance_id` 标识 layer、sandbox 和 memory snapshot。
- 不把 Copy 与 mmap 描述成相同内存复用效果。
- 不在没有 timeout/fatal channel 时允许 vCPU 进入 faultable region。
- 不为统一协议而让 rootfs、block diff、RAM page 共用同一 JSON schema。
- 不把未合入 PR 或实验分支写成上游正式能力。

## 需要本地基准后再决定

- `MAP_PRIVATE` 对比 `MAP_SHARED` 的 PSS、COW 与安全收益；
- fetch unit 默认值；
- PROBE/prefault 是否进入首版；
- 小镜像直接 full fetch 阈值；
- background prefetch 并发和带宽预算；
- 每 layer 一个 pmem device 与预合并 EROFS artifact 的规模上限。
