# lazyd 职责与内部重构

> 阅读完成后，读者能够说明 lazyd 如何通过 HTTP 控制面和 FETCH 数据面服务 virtio-pmem 懒加载，并区分本次目标与历史兼容范围。

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
目标接口
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

Conch 调用 HTTP 控制面准备内容；StratoVirt 处理 UFFD event，并向 lazyd 发送按 instance/offset/len 表达的 FETCH。图中的 adapter 是建议的职责划分，不表示代码中已存在同名模块。

## fanotify 的范围说明

当前 Conch + StratoVirt 方案暂不考虑 fanotify，不将其作为目标接口、启动依赖或本次验收项。Guest EROFS+DAX 访问通过 pmem GPA 落到 StratoVirt HVA，不能依靠宿主机 fanotify 为这条路径拦截缺失内容访问；触发入口由 StratoVirt 的 UFFD handler 承担。

已有 fanotify 相关能力属于普通容器/VFS 场景的历史兼容范围，可暂时保留。此次设计不要求新增或重构 fanotify adapter，也不据此删除历史实现；是否继续维护或移除，留待明确其使用需求后独立决定。业界产品章节中的 fanotify 仍用于描述相应产品，不代表本项目采用。

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

### 自适应预取

建议由 `RangeCoordinator` 管理前台需求与后台预取，两者共享 bitmap、inflight 去重和 data-before-ready 顺序。前台所需范围就绪即返回 FD，不等待预测范围；前台命中预取任务时复用并提升优先级。后台预取失败只影响预测任务，不能污染 ready 状态。

保留固定 `fetch.unit_bytes`，动态调整预取单位数和并发。结合近期延迟、吞吐、连续访问和预取利用率调整窗口，不因首次请求慢就扩大下载。配置集中在 lazyd，窗口、并发和带宽均有上限，拥塞时收缩或暂停；不需要修改 FETCH v1 或要求 StratoVirt 增加 PROBE。

这是待实现增强，先做关闭/开启可对比的顺序预取，再引入网络反馈。详细策略、v1 访问流限制和验收矩阵见 [缓存、去重与预取比较](../03-design-comparison/06-cache-dedup-and-prefetch.md)。

### 其他数据对象

lazyd 可以增加新的 immutable object type或 block-diff source，但要使用明确版本化 schema。guest RAM source优先由 StratoVirt/conch-cow 管理，不应因为同样按 range 读取就强行并入 EROFS instance。
