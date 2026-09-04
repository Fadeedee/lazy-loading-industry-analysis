# 实验项目的生命周期教训

> 阅读完成后，读者能够识别共享视图、跨节点快照和 API 状态设计中仍缺失的正确性条件。

## API 不等于数据面

`restore-snapshot` 可以是全量恢复、copy、reflink、远端块或 UFFD。没有源码/文档说明触发和填充方式时，只能确认 control plane 能力，不能标记为 lazy loading。

## 跨节点不仅是上传文件

CubeSandbox 的跨节点 snapshot 讨论涉及 memory、disk 和 config 的远端同步。[SRC-CUBE-004] 真正恢复还要解决：

- parent lineage；
- 版本/硬件兼容；
- 一致性时间点；
- partial download 与校验；
- lease、失败清理和重试。

## 共享并非总能提速

urunc 原型说明共享 view 会增加 control-plane、mount/device 和 lease 成本。[SRC-URUNC-001] 只有在内容大、复用率高或准备成本显著时，收益才更可能超过固定开销。
