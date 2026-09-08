# Cloud Hypervisor 数据路径

> 阅读完成后，读者能够理解 v53 memory restore 的能力边界，并复盘 #8239 的外部 PMEM 协议为何仍有架构争议。

## v53 demand-paged restore

公开 release 信息确认使用 UFFD 按需恢复 guest memory，并配套 snapshot/restore daemon。[SRC-CH-001]

```text
snapshot metadata/memory ranges ready
  -> VMM creates faultable guest RAM
  -> resume
  -> UFFD fault
  -> restore requested page/range
  -> continue vCPU
  -> optional background population
```

release note 不足以证明其 PMEM rootfs 数据协议；本文不把未由源码/正式文档确认的细节补成事实。

## PR #8239

该 PR 的 external UFFD protocol 让 VMM 传 UFFD 和 memory/pmem region；对 PMEM 可由 handler copy，也可返回 FD mapping region 给 VMM fixed remap。[SRC-CH-002]

该 PR 的核查状态是 `closed-unmerged`。它可用来理解接口和 review 风险，不能作为上游 ABI 依赖。
