# Nydus v2 数据路径

> 阅读完成后，读者能够分别跟踪 RAFS 文件读取和 UFFD pmem fault 的触发、定位、取数、校验与完成过程。

## RAFS 文件级路径

```text
container read(path, offset)
  -> FUSE/virtiofs/EROFS
  -> bootstrap: inode/file offset -> blob/chunk
  -> blob backend HTTP range
  -> 解压，按格式和配置执行 chunk 校验
  -> blob cache
  -> 返回文件字节
```

RAFS v5 的设计文档描述 bootstrap 与 data blob 分离，默认文件数据按 1 MiB chunk 组织；每个 chunk 的摘要和 blob 内偏移记录在元数据中。[SRC-NYDUS-001] RAFS v6 与内核 EROFS 兼容，但不能把 v5 的所有布局细节无条件套到 v6。

## UFFD block service 路径

```text
VMM 握手传入 UFFD FD + VMA regions
  -> Nydus service 直接监听该 FD
guest EROFS+DAX load
  -> VMM HVA missing fault
  -> Nydus UFFD service
  -> flattened block offset -> RAFSv6 meta/data blob range
  -> BlockDevice::fetch_ranges -> DataBlob::async_fetch/cache
  -> Copy: UFFDIO_COPY
     或 Zerocopy: SCM_RIGHTS 返回 blob/cache fd 和 mmap region
  -> VMM fixed mmap + wake
```

2026-09-08 复核 `8aa80aee6e77` 的源码，具体分工如下；本次没有运行 Nydus/VMM 联调。

| 阶段 | 源码与实际行为 |
| --- | --- |
| 握手/事件 | [`UffdWorker::handle_handshake/handle_conn`](https://github.com/dragonflyoss/nydus/blob/8aa80aee6e77a0c4d529581fc6339e9ff3066736/service/src/block_uffd.rs)：接收 UFFD FD，持有 `AsyncFd<OwnedFd>` 并读取事件，不是每次等待 VMM FETCH |
| 地址定位 | 同文件 `UffdCore::handle_page_fault`：`region.offset + fault_hva - base_hva`，按 region 中的 `page_size` 字段计算处理窗口并裁剪边界 |
| 后端定位 | [`BlockDevice::fetch_ranges`](https://github.com/dragonflyoss/nydus/blob/8aa80aee6e77a0c4d529581fc6339e9ff3066736/service/src/block_device.rs)：metadata blob 按已就绪处理；data blob 先 `async_fetch`；hole 跳过，由 UFFD service zero |
| 数据完成 | Copy 由 service 读取并 `UFFDIO_COPY`；Zerocopy 发送 FD 与 `blob_offset/block_offset/len`，VMM 客户端承担映射和唤醒 |
| 协议 | [`uffd_proto.rs`](https://github.com/dragonflyoss/nydus/blob/8aa80aee6e77a0c4d529581fc6339e9ff3066736/service/src/uffd_proto.rs)：Handshake/Stat/PageFault 等消息，与本项目 seqpacket FETCH v1 不是同一协议 |

[SRC-NYDUS-005] [SRC-NYDUS-006] [SRC-NYDUS-007]

## 粒度

- fault 的页粒度来自实际 host mapping/UFFD 配置，不能固定假定为 4 KiB；
- image fetch 和解压按 RAFS chunk/blob 规则执行；
- service 可以返回覆盖 fault 的更大连续 mmap range；
- prefault 在握手后异步查询并推送已就绪范围，减少后续 fault；源码不保证它在 vCPU 启动前完成。

这些粒度不能被协议实现错误地收缩成单个 fault page，否则会丢失 range amplification 的收益。

## 重要实现差异

Nydus `VmaRegion` 默认 `prot=PROT_READ`、`flags=MAP_PRIVATE | MAP_FIXED`，实际 mmap 由客户端执行。[SRC-NYDUS-006] PRIVATE 也可共享干净文件页；只有映射允许写入时，写入才会产生 COW，不能把“默认 PRIVATE”解释成默认可写。我们的 SHARED+READ、KVM readonly 和跨 VM PSS 必须分别验证。

prefault 使用 `probe_only=true`，只枚举缓存 ready 范围，不代表从远端下载预测内容。另一个边界是错误处理：service fault loop 存在 `handle_uffd_event` 失败仅记 warning 后继续的路径，不能借其已合入状态替本项目的 fatal shutdown 策略背书。[SRC-NYDUS-005]

内容校验也不是无条件执行：[`validate_chunk_data`](https://github.com/dragonflyoss/nydus/blob/8aa80aee6e77a0c4d529581fc6339e9ff3066736/storage/src/cache/mod.rs) 受 `need_validation`、CRC32/强制参数及 legacy stargz 分支约束。它可说明校验能力，不能证明所有缓存读取都已通过密码学摘要检查。[SRC-NYDUS-008]
