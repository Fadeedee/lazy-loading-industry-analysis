# 开发前先确认：上游已经有什么

> 这里记录代码事实，不替我们决定新架构。每次开始实现时重新查询 upstream/dev，并记录实际开发 commit。

## 本次核查基线

2026-09-08 通过远端 Git 查询分支，Conch 在临时副本拉取并读取源码；StratoVirt 的远端 commit 与本地 Git 对象一致。没有修改产品仓分支或运行产品测试。

| 项目 | 本次核查 commit | 证据 |
| --- | --- | --- |
| Conch upstream/dev | `8248022005542407f53b606a8be979379a3fd38b` | [固定源码](https://gitcode.com/openeuler/Conch/tree/8248022005542407f53b606a8be979379a3fd38b) [SRC-CONCH-001] |
| StratoVirt upstream/dev | `0f948b653b1f695ad9a527059e42baff77e07a3f` | [固定源码](https://gitcode.com/openeuler/stratovirt/tree/0f948b653b1f695ad9a527059e42baff77e07a3f) [SRC-SV-001] |
| lazyd 可复用能力样本 | `751d647fb37fa2e01f6fb1784003e8b26aa00244` | [固定源码](https://github.com/Fadeedee/lazyd/tree/751d647fb37fa2e01f6fb1784003e8b26aa00244) [SRC-LAZYD-001] |

最后一行是现有实现样本，不是限制设计的协议标准，也不是本轮查询出的远端最新提交。

## Conch：不是一个只负责拼命令的客户端

当前路径是 Template 对应 Boot Index，解析 rootfs、memory、VM components，形成启动布局后交给 VMM。Template/Sandbox store 保存持久状态和资源引用。

| 代码位置 | 观察到的行为 | 对设计意味着什么 |
| --- | --- | --- |
| [bootindex_pull.go](https://gitcode.com/openeuler/Conch/blob/8248022005542407f53b606a8be979379a3fd38b/internal/image/bootindex_pull.go) | `WithPulledBootIndex` 使用带 lease 的 `client.Fetch`，wrapper 检查根类型，没有按需 payload 筛选 | selective pull 不只是增加 CLI 开关，要区分元数据与数据的遍历及完整性检查 |
| [boot.go](https://gitcode.com/openeuler/Conch/blob/8248022005542407f53b606a8be979379a3fd38b/internal/sandbox/boot.go) | `BootPreparer` 经 snapshot backend 得到布局；`BootSpec.PmemPaths` 表达本地路径 | 启动契约需表达本地文件或按需资源，不能用缺文件假装完整布局 |
| [sandbox/store.go](https://gitcode.com/openeuler/Conch/blob/8248022005542407f53b606a8be979379a3fd38b/internal/adapters/containerd/sandbox/store.go) | Sandbox extension 和 Boot Index/runtime snapshot GC 引用 | 复用资源 owner；外部缓存保护需明确连接，不会由 GC label 自动获得 |
| [stratovirt.go](https://gitcode.com/openeuler/Conch/blob/8248022005542407f53b606a8be979379a3fd38b/internal/vmm/stratovirt/stratovirt.go) | 为 pmem 生成 file backend；恢复使用 `incoming file:...,mapped=true` | 普通启动和本地恢复可作为回归基线，不等于远端按需取数 |
| [daemon.go](https://gitcode.com/openeuler/Conch/blob/8248022005542407f53b606a8be979379a3fd38b/internal/daemon/daemon.go) | Shutdown 与 HTTP Serve 的停止关系已收口 | 新增长任务和数据源连接必须纳入停止/取消路径 |

guestd 位于 `internal/agent/guestd`，归 Conch 仓维护。设备识别、挂载和失败反馈属于三仓联调的一部分。[SRC-CONCH-001]

## StratoVirt：已有外部 UFFD，不要重复造一套

[pmem.rs](https://gitcode.com/openeuler/stratovirt/blob/0f948b653b1f695ad9a527059e42baff77e07a3f/virtio/src/device/pmem.rs) 的普通 pmem 使用 file backend，设备地址和大小要求 2 MiB 对齐；共享标志影响后端打开/映射行为。**后端文件只读打开，不等于我们已验证 guest 对共享页面的全部写保护。**

[address_space/uffd.rs](https://gitcode.com/openeuler/stratovirt/blob/0f948b653b1f695ad9a527059e42baff77e07a3f/address_space/src/uffd.rs) 已有：

- `UffdMemoryBackend::register_region`：清除已驻留页后注册 MISSING，可协商 WP；
- `send_to_external_uffd_daemon`：通过 UnixStream/SCM_RIGHTS 发 UFFD FD 与区域描述，保留连接；
- dirty/resident bitmap、WP 回退，以及相应测试。

这为外部 handler 提供现实起点，但函数**不等待 ready ACK**，也没有因此获得 pmem 文件 remap 通道。事件能排队不等于数据源可用、失败可传播。新设计应补齐可观察的 attachment 就绪和失败语义，而非机械添加某个 ACK 字节。[SRC-SV-001]

## 增量恢复依赖：明确条件，不提前当作基线

2026-09-08 官方 API 查询结果：

| PR | 状态与 head | 用法 |
| --- | --- | --- |
| [Conch #155](https://gitcode.com/openeuler/Conch/pull/155) | open，`1ca332dca323` | 内存谱系、conch-cow attachment、memfd 写页方案参考 |
| [StratoVirt #2017](https://gitcode.com/openeuler/stratovirt/pull/2017) | open，`d70c76b78f9e` | inherited memfd backend 参考 |

两项均不是本次 dev 已合入能力。[SRC-CONCH-003] [SRC-SV-002] 用户计划它们先于懒加载落地，因此实施前必须重新确认合入版本和最终接口，再决定复用范围。

## lazyd：能力可复用，接口可重新设计

固定样本包含 descriptor prepare、registry Bearer auth、按范围取数、digest cache、bitmap、inflight 协调及 FD 数据面。此前核查确认数据同步先于 bitmap ready。[SRC-LAZYD-001]

这些可以降低实现成本，但不要求继续使用同名 API、固定单位或固定服务职责。只读句柄、缓存完整性、授权、跨服务回收等仍要按新方案审计；不能把样本已有测试等同于三仓新版本联调通过。

## 实施前的检查

先记录三个实际开发 tip，再看 upstream 是否已改变上述路径。保留正常启动、快照恢复和关闭行为的回归用例；从需求出发增加 source/attachment 能力，不直接把路径参数或历史实验分支当成设计约束。
