# Firecracker 快照与生命周期

> 阅读完成后，读者能够理解 memory/state/disk 的独立管理、diff snapshot、共享安全和 UFFD handler 失败责任。

## 快照对象

Firecracker snapshot 包含 microVM state file 和 guest memory file；磁盘 backing files 由使用者管理，并且创建 snapshot 时不会自动替调用者保证磁盘 flush/一致性。[SRC-FC-001]

full memory snapshot 可独立恢复；diff snapshot 记录自上次跟踪点以来的脏页，通常要与 base 合并或按文档支持的层顺序使用。memory file 可能是 sparse file。

## 共享与安全

`MAP_PRIVATE` 允许多个 VM 从同一 memory file 恢复并共享未修改的文件页，但克隆 VM 的随机数、网络身份、凭据和外部资源必须重新处理。Firecracker 文档专门说明从同一状态多次恢复的安全风险。[SRC-FC-001]

普通 virtio-pmem 的 backing file 跨 VM 共享也有写入安全警告；只读策略必须落实到设备和 host/KVM 权限，不只依赖命令行约定。[SRC-FC-003]

## UFFD 失败

官方文档明确指出：handler 不处理 fault 时 Firecracker 会等待，可能永久挂起；使用者需要监控 handler、设置 timeout，并在 handler 错误时终止 peer VM。[SRC-FC-002] 这是 Conch/StratoVirt 必须补齐 VMM fatal shutdown 通道的直接证据。
