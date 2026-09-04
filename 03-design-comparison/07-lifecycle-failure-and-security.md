# 生命周期、失败与安全比较

> 阅读完成后，读者能够审查一个懒加载方案在 daemon 崩溃、网络失败、缓存损坏、多租户共享和资源回收时是否具有完整处理路径。

## 启动屏障

vCPU/工作负载运行前，至少满足：

- resource descriptor 与 identity 已校验；
- source/cache service 可访问；
- HVA/memslot/UFFD 或 block/FUSE backend 已注册；
- handler event loop 已运行；
- guest device/mount metadata 完整；
- 上层已登记失败回调和生命周期引用。

## 失败矩阵

| 失败 | 错误风险 | 所需策略 |
| --- | --- | --- |
| registry timeout/401 | fault 长时间阻塞 | deadline、token refresh、有界 retry |
| digest mismatch | guest 读错误内容 | reject range、invalidate ready、fatal |
| partial cache write | bitmap 误报 ready | data sync 后再 bitmap ready |
| lazyd crash | FETCH 断开 | reconnect 或 VM fail，不静默等待 |
| StratoVirt handler crash | vCPU 永久阻塞 | VMM fatal channel/shutdown |
| fd/range 越界 | SIGBUS/越权映射 | file size、offset、len、pmem bounds 校验 |
| VM delete | 误删共享 cache | release VM reference，不直接删 content |
| snapshot parent GC | child 无法恢复 | lineage lease/refcount |

Firecracker 官方文档明确提示 UFFD handler 不处理 fault 会导致 VM 挂起。[SRC-FC-002]

## 安全边界

- UDS 应限制路径权限并验证 peer credentials；
- `SCM_RIGHTS` 接收的 fd 必须检查文件类型、大小和只读语义；
- instance ID 只用于 lookup，不能替代 digest 校验和授权；
- 多租户共享只允许不可变、非秘密内容；
- 从同一 memory snapshot 克隆时更新随机数、网络身份和凭据；
- KVM readonly memslot、host VMA protection 与 guest device readonly 要一致；
- cache 目录不能由 guest 或不可信进程写入。

Firecracker 对 snapshot clone 和 pmem 共享都有专门安全提示。[SRC-FC-001] [SRC-FC-003]

## GC 原则

```text
content object lease
  <- image/snapshot reference
  <- running VM mapping/fd
  <- in-flight fetch/remap
```

只有所有引用消失，且对象不在 in-flight/failed-recovery 中，GC 才可删除 cache/bitmap。第一阶段没有完整 lease 时应保守保留，不能让单 VM cleanup 删除共享 instance。

## 版本升级

bitmap、data protocol、lazy-rootfs metadata 和 snapshot lineage 都要带 version。升级必须定义：旧格式是否只读、是否重建、是否拒绝，以及 rolling upgrade 中不同 Conch/lazyd/StratoVirt 版本如何协商。
