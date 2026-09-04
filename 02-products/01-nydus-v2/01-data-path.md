# Nydus v2 数据路径

> 阅读完成后，读者能够分别跟踪 RAFS 文件读取和 UFFD pmem fault 的触发、定位、取数、校验与完成过程。

## RAFS 文件级路径

```text
container read(path, offset)
  -> FUSE/virtiofs/EROFS
  -> bootstrap: inode/file offset -> blob/chunk
  -> blob backend HTTP range
  -> 解压并校验 chunk digest
  -> blob cache
  -> 返回文件字节
```

RAFS v5 的设计文档描述 bootstrap 与 data blob 分离，默认文件数据按 1 MiB chunk 组织；每个 chunk 的摘要和 blob 内偏移记录在元数据中。[SRC-NYDUS-001] RAFS v6 与内核 EROFS 兼容，但不能把 v5 的所有布局细节无条件套到 v6。

## UFFD block service 路径

```text
guest EROFS+DAX load
  -> VMM HVA missing fault
  -> Nydus UFFD service
  -> flattened block offset -> RAFSv6 meta/data blob range
  -> NydusCore fetch/validate/cache
  -> Copy: UFFDIO_COPY
     或 Zerocopy: SCM_RIGHTS 返回 blob/cache fd 和 mmap region
  -> VMM fixed mmap + wake
```

本地 `service/src/block_uffd.rs` 负责 UFFD fault、region 和 policy；`service/src/block_device.rs` 把 flattened block range 转成可读取或可映射的后端范围；`service/src/uffd_proto.rs` 定义消息和 FD 规则。[SRC-NYDUS-004]

## 粒度

- guest fault 通常按 host page 触发；
- image fetch 和解压按 RAFS chunk/blob 规则执行；
- service 可以返回覆盖 fault 的更大连续 mmap range；
- prefault 在 vCPU 运行前查询并映射已就绪范围，减少恢复后的重复 fault。

这些粒度不能被协议实现错误地收缩成单个 fault page，否则会丢失 range amplification 的收益。

## 重要实现差异

Nydus 当前 UFFD 协议源码中的默认 region flags 是 `MAP_PRIVATE | MAP_FIXED`。[SRC-NYDUS-004] `MAP_PRIVATE` 可复用干净 file page，但写入会 COW；它与目标设计中严格只读的 `MAP_SHARED | MAP_FIXED` 语义不同。Conch 方案需要根据 KVM readonly memslot、host 映射权限和跨 VM PSS 实测决定最终标志，不能只复制常量。
