# Nydus v2 对 Conch 方案的借鉴

> 阅读完成后，读者能够列出 Nydus 已合入 UFFD service 可直接复用的设计、需要适配的差异和不应照搬的边界。

## 直接采用

- flattened pmem device offset 到多个 image object/range 的查找模型。
- UFFD registration 与 vCPU 可运行之间的 ready barrier。
- Copy/Zerocopy 明确分支及 SCM_RIGHTS FD 生命周期。
- FETCH 返回完整连续 ready range，而非只返回 fault page。
- prefault/PROBE 作为后续性能增强，不作为 cold-start MVP 正确性前提。
- reconnect、request/region 校验和针对 UFFD fault 的独立测试结构。[SRC-NYDUS-004]

## 需要适配

- 当前 lazyd 使用原生 EROFS layer、bitmap 和 descriptor-based prepare，不能假设 RAFS bootstrap/blob 拆分完全相同。
- Nydus 协议与 lazyd FETCH v1 字段不同，借鉴状态机和安全检查，不直接造成跨仓 ABI 漂移。
- `MAP_PRIVATE` 与目标 `MAP_SHARED`/readonly 语义需通过 StratoVirt KVM memslot 和多 VM 基准重新确定。
- NydusCore 内嵌式能力可作为长期简化方向，但当前先保持 lazyd 独立进程，降低 StratoVirt 对镜像格式的耦合。

## 不建议采用

- 不把 Nydus v3 实验分支的性能数字当作 Nydus v2 正式能力。
- 不把 UFFD block service 扩展为 Conch 的 VM lifecycle manager。
- 不继续押注已被 EROFS 官方标记退场的 fscache on-demand ABI 作为唯一主路径。[SRC-EROFS-004]
- 不因共享 cache 就在单个 VM delete 时删除内容级 instance。

## 对三仓的映射

| 项目 | 建议 |
| --- | --- |
| Conch | 编排 layer/device、保存稳定 content identity、管理 VM 引用 |
| lazyd | 复用 Nydus 的 range、校验、prefetch 和 FD 供应经验 |
| StratoVirt | 重点对照 UFFD service 的 region、policy、fixed remap、wake 和 reconnect |

Nydus PR #1921 是当前实现审阅的首要对照代码，但不是可以不经差异分析直接复制的协议标准。
