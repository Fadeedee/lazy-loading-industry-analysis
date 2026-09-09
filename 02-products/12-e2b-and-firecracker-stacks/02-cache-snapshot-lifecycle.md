# E2B cache 与 checkpoint 生命周期

> 阅读完成后，读者能够理解 template、sandbox COW、memory diff 和 disk diff 的身份与回收关系。

## 对象图

```text
Template
  ├── read-only rootfs base
  ├── base memory/state
  └── runtime configuration

Sandbox instance
  ├── rootfs COW cache
  ├── memory dirty state
  └── network/process lifecycle
```

pause/export 产生两类增量：rootfs changed blocks 和 memory diff。恢复时它们分别叠加到对应 template parent。[SRC-E2B-001]

## 生命周期约束

以下是据沙箱恢复场景列出的正确性约束，不表示本次已逐条核查全部实现；后面的源码小节单独说明实际确认的行为。

- template 不能在 sandbox 或 snapshot child 仍引用时回收；
- COW head 必须每 sandbox 隔离；
- pause 需要协调 vCPU、disk writes 和 export consistency；
- resume 前两条 fault backend 都要 ready；
- cache miss/fetch failure 应进入 sandbox failure，而不是无限挂起；
- 多租户共享 template 时还要处理秘密、随机数和网络身份刷新。

E2B 架构说明统一 orchestration 的价值主要在依赖、状态和失败，而不是数据复制函数复用。

## 就绪和关闭的具体边界

2026-09-09 定向复核固定版本 `cc7c574233ad` 的 [NBDProvider](https://github.com/e2b-dev/infra/blob/cc7c574233ad98665a7c72a3d37b0af89ae79a71/packages/orchestrator/pkg/sandbox/rootfs/nbd.go)，不代表重新测试了整个 E2B 恢复路径。[SRC-E2B-004]

- `NewNBDProvider` 把只读设备、私有 block.Cache 和 overlay 组合成 mount provider，同时持有 ready 结果和 finishedOperations 信号。
- `Start` 尝试打开设备，并向 ready 写入设备路径或错误；`Path` 等待该结果，不是无条件返回配置中的路径。
- `Close` 依次尝试 flush、mount.Close、通知 finishedOperations 和 overlay.Close，最后汇总错误。发出 finishedOperations 不等于所有清理都已成功；特别是 mount.Close 的错误不会阻止执行后续代码。
- `ejectAndStopSandbox` 将私有 cache 摘出，触发沙箱关闭并等待设备释放信号；等待超时会处理摘出 cache 的关闭。成功返回后，调用方拥有该 cache 并负责后续关闭。

借鉴点是准备结果、运行操作和导出对象有不同生存期。这里的 cache 是沙箱私有写层，不能据此设计“关闭一个 VM 就删除公共 EROFS cache”。重复关闭、所有调用方取消路径和进程崩溃后的全局回收没有在本次局部核查中验证。
