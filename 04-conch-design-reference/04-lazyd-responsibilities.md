# lazyd 职责与内部重构

> 阅读完成后，读者能够说明 lazyd 如何作为内容服务支持 fanotify 和 virtio-pmem 两类 frontend，并列出当前代码在合入前需要收敛的点。

## 核心定位

lazyd 是 immutable range content service，不是 VMM handler 或 sandbox manager。它负责：

- OCI registry auth、descriptor 与 HTTP Range；
- digest/size/media type 校验；
- digest-addressed sparse cache；
- versioned bitmap 和 recovery；
- range amplification、inflight 去重与 completion fan-out；
- prepare control API；
- FETCH seqpacket API 和 SCM_RIGHTS cache fd；
- cache/instance metrics 与后续 lease/GC。

## 推荐内部结构

```text
frontends
  ├── fanotify adapter
  ├── HTTP control adapter
  └── seqpacket FETCH adapter
          |
          v
content core
  ├── InstanceRegistry
  ├── RangeCoordinator
  ├── RangeMap/bitmap
  ├── CacheFile
  └── RemoteBackend(OCI/...)
```

fanotify 与 UFFD 场景并列存在，但 UFFD event 本身由 StratoVirt 处理。lazyd只接收按 instance/offset/len 表达的 FETCH。

## 已有正确性基础

当前代码先 `write_all_at -> target.sync_data -> set_range_ready -> bitmap.sync_data`，满足 ready 不早于 data 的核心顺序。[SRC-LAZYD-001]

cache key 严格接受 canonical SHA256，`instance_id = erofs-<cache_key>`，相同 digest 在不同 image ref/index 下复用同一内容。已有 bitmap header 包含 magic/version/unit/digest/blob size/slot count。

## 合入前收敛

1. **只读 FD 导出**：FETCH 完成后重新以 `O_RDONLY|CLOEXEC` 打开 cache，SCM_RIGHTS 不发送内部写句柄。
2. **等待通知**：将 5 ms poll 替换为 `Notify`/shared future；完成后 fan-out，错误也要唤醒等待者。
3. **recovery 边界**：承认 `SEEK_DATA/HOLE` 只能清理明显洞；强校验可按 checksum/version 或保守清 ready 设计。
4. **凭据**：维持 `0600`/原子写，日志脱敏，定义 token refresh 和 credential rotation。
5. **生命周期**：`DELETE instance` 只卸载运行态注册还是删除 persisted state要明确；无 lease 前不暴露会误删共享 cache 的 Conch cleanup。
6. **权限**：socket owner/mode、peer credential、request size/concurrency quota。
7. **观测**：remote bytes、cache hit、waiters、fetch latency、bitmap recovery、FD count。

## 后续扩展

lazyd 可以增加新的 immutable object type或 block-diff source，但要使用明确版本化 schema。guest RAM source优先由 StratoVirt/conch-cow 管理，不应因为同样按 range 读取就强行并入 EROFS instance。
