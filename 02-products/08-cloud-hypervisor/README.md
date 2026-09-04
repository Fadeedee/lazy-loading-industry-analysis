# Cloud Hypervisor

> 阅读完成后，读者能够区分 v53 已发布的 guest RAM demand-paged restore 与 PR #8239 未合入的 pmem external UFFD 设计。

## 已发布能力

Cloud Hypervisor v53 release 宣布 UFFD-based demand-paged VM restore、snapshot/restore daemon 和 sparse memory snapshot 改进。[SRC-CH-001] 这属于 guest RAM 恢复。

## 未合入能力

PR #8239 曾尝试让外部 UFFD handler 同时服务 snapshot memory 和 PMEM，PMEM 支持 Copy/FD+MAP_FIXED。该 PR 已关闭且未合入；reviewer 质疑在缺少 issue/设计共识时把两类功能放进同一个协议。[SRC-CH-002]

## 对调研的价值

它同时提供正面和反面证据：VMM 内存按需恢复已经成为正式功能，但共享底层 UFFD helper 不代表 PMEM 与 snapshot 应共享同一业务接口。

来源：[SRC-CH-001] [SRC-CH-002]
