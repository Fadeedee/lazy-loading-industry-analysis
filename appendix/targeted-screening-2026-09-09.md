# Dragonfly 与 JuiceFS 定向源码筛查

> 判断共享任务、缓存发布与回收机制能否细化 lazyd 的选型实验，不展开完整产品架构。

## 范围与结论

核查日期：2026-09-09。Dragonfly client 固定为 `d5bccb022e805944080334236eb61d3ac5103bee`，JuiceFS 固定为 `5f250030ec437d6c676355e7a99f92b8ec480747`。使用官方 API 获取版本、读取对应文件；完整 shallow clone 未完成，已停止，改用定向下载。未编译或运行两个项目，不评估其整体可靠性、性能和发布成熟度。

| 对象 | 筛查结果 | 对当前决策的影响 |
| --- | --- | --- |
| Dragonfly | owner/等待者分离和先注册通知再复查状态值得参考；超时会触及共享状态 | 细化 E3 竞争与取消测试，不据此决定 T1/T2 |
| JuiceFS | 临时文件发布、容量与 inode 双约束值得参考；需区分原子可见与持久发布 | 细化 E1/E2 缓存实验，不据此决定 S1/S2 |

当前停止扩大两项目范围。只有实验暴露具体未知再追查相关调用链；分布式调度、完整文件系统、P2P 与新设备适配均非前置。

## Dragonfly：通知结束不等于成功

[`PieceNotifier`](https://github.com/dragonflyoss/client/blob/d5bccb022e805944080334236eb61d3ac5103bee/dragonfly-client-storage/src/piece_notifier.rs)按 piece ID 在 DashMap 中原子领取 owner；其他调用者得到共享 Notify。终态清理唤醒等待者，metadata 仍是结果依据。文件内测试覆盖单 owner、通知唤醒和 key 隔离；本次只读测试代码，未执行。[SRC-DF-001]

[`wait_for_piece_finished`](https://github.com/dragonflyoss/client/blob/d5bccb022e805944080334236eb61d3ac5103bee/dragonfly-client-storage/src/lib.rs)先启用通知等待，再重查 metadata，结合通知、兜底间隔和总超时等待。超时分支调用 metadata 的失败处理，并移除通知器、唤醒其他等待者；`download_piece_failed` 同样执行终态通知。[SRC-DF-002]

**可采用的机制：** 任务领取与结果状态分离；注册等待后再次检查，减少检查/订阅窗口内丢通知的风险；失败也需要通知。

**适用差异：** 本次未核查完整下载调度与取消链，不能推导每个消费者都具备独立取消语义。尤其该超时路径会触及共享 metadata，不直接用作 lazyd 单会话超时逻辑。进程内 piece 去重也不等于跨安全域授权和公平调度。

**E3 增补用例：** 两个等待者在 owner 完成前后分别订阅；通知先于等待；一个等待者超时但另一个仍需要结果；owner 失败后立即重试；旧任务清理与新任务领取并发。检查结果归属、代次、通知与公共任务存活，不仅统计下载次数。

## JuiceFS：文件发布与回收的承诺

[`diskCache.flushPage`](https://github.com/juicedata/juicefs/blob/5f250030ec437d6c676355e7a99f92b8ec480747/pkg/chunk/disk_cache.go)写 `.tmp` 文件，可追加 checksum 和 staging footer，关闭后 rename。所读 `writeFile/closeFile/renameFile` 分别包装 Write、Close 和 Rename，这条路径未见显式文件/目录 sync。[SRC-JFS-001]

同文件 `cleanupFull` 同时考虑容量、条目数与空闲 inode，通过 eviction iterator 选项后删除缓存路径。该代码没有展示跨 VMM 导出映射的引用管理；不能将其删除流程直接当作 lazyd 的映射保护协议。

**可采用的机制：** 发布前临时对象隔离；同时衡量字节与 inode 预算。**适用差异：** 可重新下载的文件系统读缓存与承诺持久 ready 的供数对象不同。原子 rename 不构成数据及目录掉电持久性证明；有缓存 checksum 也不等于具有可信远端分块摘要。

**E1/E2 增补用例：** rename 前后注入失败；数据屏障完成而索引未发布；已导出 FD 时删除目录项，观察物理占用而非仅逻辑计数；仍有消费者时禁止截断/原地覆盖；重启扫描遇到临时文件。unlink 不等于立即释放仍被引用的 inode，需在隔离实验中测量实际回收。

## JuiceFS：共享读取的边界

[`Controller.Execute`](https://github.com/juicedata/juicefs/blob/5f250030ec437d6c676355e7a99f92b8ec480747/pkg/chunk/singleflight.go)按 key 合并请求，以 WaitGroup 等待并共享 Page/错误，按重复调用数取得 Page 引用。接口不接收等待者 context。[SRC-JFS-002]

[`rSlice.ReadAt`](https://github.com/juicedata/juicefs/blob/5f250030ec437d6c676355e7a99f92b8ec480747/pkg/chunk/cached_store.go)在相应整块读取分支使用 `group.Execute`，下载闭包使用发起执行的调用上下文。源码可证明该分支共享执行与结果，不能证明等待者单独取消或 owner 上下文取消不影响其他等待者。[SRC-JFS-003]

对 lazyd 的启发是共享字节结果与每调用者等待关系分别管理。E3 应验证共享缓冲区/导出对象寿命、独立取消及 owner 退出，不直接把同步等待结构搬入异步事件循环。

## 后续动作

将以上用例纳入[核心验证计划](../04-conch-design-reference/09-core-design-selection.md)。先执行最小实验，不新增两个完整产品目录，也不扩展首期范围。缓存布局、任务模型与任何性能收益继续保持待验证状态。
