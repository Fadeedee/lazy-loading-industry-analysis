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
| 内容服务或适配器崩溃 | 正在等待的访问无法完成 | 有界恢复或 VM fail，不静默等待 |
| 内部/外部 handler 退出 | vCPU 永久阻塞 | VMM/Conch 监控和明确停止策略 |
| fd/range 越界 | SIGBUS/越权映射 | file size、offset、len、pmem bounds 校验 |
| VM delete | 误删共享 cache | release VM reference，不直接删 content |
| snapshot parent GC | child 无法恢复 | lineage lease/refcount |

Firecracker 官方文档明确提示 UFFD handler 不处理 fault 会导致 VM 挂起。[SRC-FC-002]

## 安全边界

- UDS 应限制路径权限并验证 peer credentials；
- `SCM_RIGHTS` 接收的 fd 必须检查文件类型、大小和只读语义；
- 内容/视图 ID 只用于 lookup，不能替代校验和授权；
- 跨租户缓存复用必须同时满足内容不可变、授权和隔离要求；秘密内容需专门安全策略，不能仅凭 digest 相同跨域共享；
- 从同一 memory snapshot 克隆时更新随机数、网络身份和凭据；
- 只读映射必须验证 KVM 写入处理和 host VMA 权限；不能假设设备声明等于实际写保护；
- cache 目录不能由 guest 或不可信进程写入。

Firecracker 对 snapshot clone 和 pmem 共享都有专门安全提示。[SRC-FC-001] [SRC-FC-003]

## GC 原则

```text
content object lease
  <- image/snapshot reference
  <- running VM mapping/fd
  <- in-flight fetch/remap
```

运行引用与持久 checkpoint 引用需要区分：可重新获取的缓存可以在无活跃读取/映射并满足策略时驱逐，但被 checkpoint 引用的唯一持久来源不能直接删除。进行中的 fetch、恢复和映射必须受到保护；没有可靠引用管理前采用保守保留策略。

## 版本升级

缓存格式、数据协议、准备记录和 snapshot lineage 都要定义版本与兼容规则。字段名和存储位置可以共同设计，不预设既有 lazy-rootfs.json 是新接口。升级需明确旧格式重建/拒绝、版本协商及运行实例迁移；公共协议也必须保留文件、磁盘、RAM 的类型语义。
