# StratoVirt 职责与重写边界

> 阅读完成后，读者能够说明 lazy pmem 如何接入最新 StratoVirt 的 pmem、address_space、KVM 和 machine lifecycle，并知道旧分支哪些内容可以迁移。

## 核心职责

- 解析并校验 lazy virtio-pmem 配置；
- 为 lazy pmem 预留 anonymous HVA；
- 把 HVA region 加入 AddressSpace/KVM memslot；
- 用 UFFD missing 注册该 HVA；
- 在 guest 可访问前启动内部 fault loop；
- fault offset 分类为 data、tail 或 padding；
- 调 lazyd FETCH、收 SCM_RIGHTS fd、严格校验；
- fixed remap 完整 ready range并 wake；
- 失败时通过现有 machine shutdown request 终止 VM；
- unrealize/exit 时停止 handler、关闭 fd、解除注册和 mapping。

## 与普通 pmem 分流

```text
virtio-pmem
  ├── regular: memdev -> memory-backend-file -> existing behavior
  └── lazy:    size/instance/socket -> anonymous HVA -> UFFD -> file remap
```

regular path 不认识 lazy 参数；lazy path 不接受 sparse cache `memdev`。两者共享 virtio transport/config space，但 backend realization 分开。

## 与最新上游衔接

`origin/dev` 已有：file-backed pmem、2 MiB address/size 对齐、readonly backing realization、iothread、snapshot UFFD/WP helper 和 machine shutdown机制。[SRC-SV-001]

重写时应：

- 复用 `HostMemMapping`、`Region`、AddressSpace listener/KVM slot；
- 扩展 UFFD wrapper 支持 missing read/zeropage/wake，不修改 snapshot WP fallback；
- 复用项目现有 Unix/FD helper或 `vmm_sys_util`，减少手写 libc ABI；
- 将 lazy backend做成小模块，`pmem.rs` 只保留选择和 lifecycle wiring；
- 让 readonly 最终落实到 KVM memslot/region，而不只是在 CLI 校验 `readonly=on`。

## 旧分支可迁移内容

- protocol golden types 与 mock lazyd tests；
- range/page/file-size 校验算法；
- tail/padding classification；
- fixed remap+wake spike 和相邻页继续 fault 测试；
- timeout 与 shutdown request；
- 配置组合测试。

旧分支不能直接搬运完整 `pmem.rs` 和 1000 行 `pmem_lazy.rs`，因为上游 ownership/abstraction 已变化。[SRC-SV-003]

## 与 memory incremental 共存

PR #2017 的 inherited memfd是 guest RAM backend输入，不是 pmem cache FD。[SRC-SV-002] 若它与 Conch #155 合入：

- memory UFFD：conch-cow 写 memfd + wake；
- pmem UFFD：StratoVirt FETCH lazyd + fixed map cache fd + wake；
- 两者可共享低层 UFFD ioctl wrapper和 fatal channel；
- region type、source client、mapping权限、metrics和 teardown分开。

## 合入门槛

- readonly KVM memslot闭合；
- fault error触发 VM shutdown并可被 Conch观察；
- snapshot UFFD/WP regression通过；
- x86 q35/PCI 与目标 aarch64/virt transport分别验证；
-真实 guest EROFS+DAX E2E；
- 多 VM 同 range不重复下载、映射和 PSS 数据可观察。
