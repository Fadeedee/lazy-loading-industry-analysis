# 从内存快照到按需恢复

> 阅读完成后，能够区分“完整文件已在本地，按需读页”和“文件仍在远端，缺页时下载”。

## 先看目前已经有什么

本次核查的 Conch 上游恢复路径准备本地快照文件，并向 StratoVirt 传入 mapped restore 参数。StratoVirt 另有外部 UFFD MISSING 接入，可把 UFFD 和区域信息交给外部处理者；它不能单凭 FD 发送就证明来源已就绪。具体版本与符号见[基线审计](../../04-conch-design-reference/01-current-system-boundary.md)。[SRC-CONCH-001] [SRC-SV-001]

```text
完整快照文件已在本地
  -> 建立 RAM 文件映射
  -> 恢复 CPU/设备状态
  -> guest 运行
  -> 本地文件页按需驻留
```

这可以减少预读内存和恢复等待，但不省启动前的远端下载。

## 要增加的能力：远端内容也按需供应

```text
Conch 解析 checkpoint 的内存视图及兼容配置
  -> 检查父链、版本、来源和引用
  -> 内存适配器建立“逻辑页 -> 最终来源层”的索引
  -> 内容服务准备按需读取，不必下载所有数据
  -> StratoVirt 建立可恢复的 guest RAM backend
  -> handler、来源和失败通道就绪
  -> 恢复 CPU/设备状态后 resume
```

第一次访问时，handler 从 HVA 定位**内存视图中的逻辑页**，索引决定从哪一层读取，内容服务供应所需字节，内存适配器按选定 backend 完成填充和唤醒。它不是简单地对每一层使用同一个文件 offset。

内存可写。填充方式可以是 COPY、私有文件映射或明确的 memfd/COW 路径，但必须证明 VM 写入不污染共享快照、再次 checkpoint 能正确识别脏页。

## 增量恢复不能忽略的三个状态

- **继承：**本层没有覆盖，继续查父层。
- **显式零：**本层定义该页为零，不能继续继承旧数据。
- **远端缺失：**已知应有数据但未取到，需要下载或报错，不能返回零。

Conch #155 与 StratoVirt #2017 提供了增量内存视图、外部处理和 dirty tracking 的参考。两者在 2026-09-08 核查时仍为 open，不能把提案路径当作上游已完成接口。详细差异以合入版本为准。[SRC-CONCH-003] [SRC-SV-002]

## 与 rootfs、磁盘恢复怎样配合

它们可以共用内容服务、认证、缓存、并发和预取预算，甚至使用带类型的公共协议；但 RAM 的写隔离/dirty tracking 不能等同于文件内容的 cache ready。

Conch 统一等待本次恢复实际需要的资源达到可服务状态，StratoVirt 保证 CPU/设备状态恢复顺序正确。handler 在内部还是外部，是需要连同 pmem、RAM 一起验证的选择，不在这里先固定两套进程。

## 验证重点

1. 完整本地文件恢复作为正确性基准，远端冷缓存恢复与之对拍。
2. 覆盖 base + 多层增量、显式零、跨 region、父层缺失。
3. VM 写入后再次 checkpoint，再恢复检查值与 dirty generation。
4. 两 VM 从同一快照恢复，写入、身份和运行引用彼此隔离。
5. 与磁盘视图一起检查一致时间点；网络失败不能导致无限等待。

整体恢复流程见[Checkpoint](../03-checkpoint-template-restore/README.md)，内存机制详解见[横向比较](../../03-design-comparison/03-memory-snapshot-restore.md)。
