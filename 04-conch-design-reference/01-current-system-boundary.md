# 当前三仓边界与代码现状

> 阅读完成后，读者能够基于 2026-09-04 的最新远端引用说明 Conch、lazyd、StratoVirt 已有什么、旧 lazy 分支为何不能直接合入，以及新实现应从哪里接入。

## 1. 审计基线

| 仓库 | 当前事实基线 | lazy 参考分支 | 结论 |
| --- | --- | --- | --- |
| Conch | `upstream/dev` `0b405ce0d4d2` | `lazy-pmem-conch-pr5` | 旧分支落后 182 commits，不能整体 rebase |
| lazyd | feature `751d647` | 当前 feature 本身 | 核心能力保留，做内部重构/收口 |
| StratoVirt | `origin/dev` `0f948b653b1f` | `lazy-pmem-full` | 旧分支落后 131 commits，按最新 pmem/UFFD 重写 |

来源：[SRC-CONCH-001] [SRC-LAZYD-001] [SRC-SV-001] [SRC-SV-003]

## 2. Conch 当前所有权

PR #184 已进入 `dev`，使 containerd 成为 Template 与 Sandbox metadata 的持久 owner。[SRC-CONCH-002]

当前关键路径：

```text
Template name
  -> containerd image record
  -> immutable Boot Index digest
  -> rootfs / mem-snapshot / sandbox component descriptors
  -> ResolveBoot
  -> BootPreparer
  -> snapshot BootLayout
  -> VMM ResourceArgs
```

关键代码位置：

| 路径 | 当前职责 | lazy 接入点 |
| --- | --- | --- |
| `internal/image/bootindex.go` | 构建/校验 Boot Index 及 component closure | 给 rootfs component 增加 prepared-rootfs descriptor/annotation 关系 |
| `internal/adapters/containerd/template/store.go` | Template name 到 Boot Index target，维护 content children labels | 保护 lazy metadata content，不保存 cache 路径 |
| `internal/adapters/containerd/sandbox/store.go` | Sandbox record、Boot Index 和 runtime snapshot GC refs | 保护运行中 prepared-rootfs descriptor 引用 |
| `internal/sandbox/boot.go` | ResolveBoot、singleflight、BootLayout 与 BootSpec | 按 typed rootfs source 分 full/lazy，不再假设都已 unpack |
| `internal/snapshot/server.go` | rootfs/memory/VM runtime snapshot mount/layout | lazy external rootfs 不创建 fake snapshot/view |
| `internal/vmm/stratovirt/stratovirt.go` | 普通 StratoVirt 命令、file-backed pmem、snapshot restore | 从 typed lazy source 生成 lazy pmem args |

当前 `BootSpec.PmemPaths []string` 只表达 file-backed EROFS 路径，无法安全表达 lazyd instance、blob size 和 socket。新实现要增加显式联合类型或独立 `PmemSources`，不能继续让路径字符串承担两种语义。

## 3. StratoVirt 当前所有权

`origin/dev` 的 `virtio/src/device/pmem.rs` 只接受 `memory-backend-file`，要求 pmem GPA 和 size 2 MiB 对齐，并在 `MemoryBackend` 上建立 file mapping。[SRC-SV-001]

上游同时已有 snapshot UFFD/write-protect 逻辑和 seccomp allowlist。新 lazy missing backend 必须复用 `address_space` 的 UFFD wrapper，但不能改变 snapshot WP negotiation/fallback。

旧 `lazy-pmem-full` 已验证：

- lazy 参数及 regular/lazy 分流；
- anonymous HVA、UFFD missing registration；
- lazyd `FETCH`/SCM_RIGHTS client；
- data/padding classification；
- `mmap(MAP_FIXED)` 和 wake；
- fetch timeout、range/file 校验和 fatal shutdown request。[SRC-SV-003]

这些是算法和测试证据，不是可合入基线。尤其 `pmem.rs` 上游已经变化，旧分支新增约 2359 行且落后 131 commits，机械移植会重新引入重复 abstraction。

## 4. lazyd 当前能力

feature 分支已经具备：[SRC-LAZYD-001]

- canonical `sha256:<64 lowercase hex>` 校验与 digest-addressed cache key；
- sparse `layer.erofs`、versioned range bitmap 与 configurable fetch unit；
- descriptor-based EROFS prepare，Conch 不必让 lazyd 解析 Boot Index；
- registry Bearer challenge/token 与 Range backend；
- `ensure_range` 放大、inflight 去重、data sync 后 bitmap ready；
- `SOCK_SEQPACKET` JSON FETCH v1 和 SCM_RIGHTS cache FD；
- blob tail-page 真实字节拉取及 padding 零语义；
- 持久 `instance.json` 与 restart restore。

当前需收敛的问题：

- `Instance.target` 以读写方式打开并 `try_clone` 后发送，导出的 fd 权限应收紧为只读；
- inflight 等待采用固定 5 ms sleep，可改为 completion notification/fan-out；
- recovery 的 `SEEK_DATA/SEEK_HOLE` 只能发现明显洞，不能证明内容摘要正确；
- auth 持久化虽使用 `0600` 和原子 rename，仍需凭据更新、脱敏和长期 credential provider 设计；
- instance unregister 会删除持久 state，没有 lease/refcount 前 Conch 不应在单 VM cleanup 中调用它。

## 5. 即将合入能力的影响

Conch PR #155 和 StratoVirt PR #2017 当前均为开放 PR，不属于 `dev` 现有能力。[SRC-CONCH-003] [SRC-SV-002]

如果先于 rootfs lazy 合入：

- PR #155 已建立 memory format、base/delta lineage、conch-cow attachment 和 memory fault 生命周期，Conch 应复用其 checkpoint resource model；
- PR #2017 允许外部 `memfd` 作为 guest RAM backend，服务 memory incremental restore；
- 它们不替代 rootfs lazy pmem，因为 guest RAM memfd 和 EROFS cache 的身份、读路径和生命周期不同；
- StratoVirt 可以复用 FD 验证、UFFD helper 和 fatal handling，但必须保留 memory/pmem 两种 region/source。

## 6. 当前设计判断

旧设计中仍有效的是 `anonymous HVA -> UFFD -> lazyd FETCH -> fd -> fixed remap -> wake` 数据面。已失效的是 fake rootfs snapshot、路径型 `lazy-rootfs.json`、旧 image-first pull 和旧 `PmemPaths` 扩展方式。新版必须围绕 Boot Index、containerd content/image/sandbox store 和 BootPreparer 重建接入。
