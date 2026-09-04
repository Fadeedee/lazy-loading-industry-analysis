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

- template 不能在 sandbox 或 snapshot child 仍引用时回收；
- COW head 必须每 sandbox 隔离；
- pause 需要协调 vCPU、disk writes 和 export consistency；
- resume 前两条 fault backend 都要 ready；
- cache miss/fetch failure 应进入 sandbox failure，而不是无限挂起；
- 多租户共享 template 时还要处理秘密、随机数和网络身份刷新。

E2B 架构说明统一 orchestration 的价值主要在依赖、状态和失败，而不是数据复制函数复用。
