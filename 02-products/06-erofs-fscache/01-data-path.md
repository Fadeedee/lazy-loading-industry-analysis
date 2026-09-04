# EROFS 与 CacheFiles 数据路径

> 阅读完成后，读者能够跟踪一次 EROFS on-demand READ，也能理解 file-backed EROFS 与 DAX 路径的区别。

## CacheFiles on-demand

```text
VFS read
  -> EROFS requests cache range
  -> CacheFiles emits READ via /dev/cachefiles
  -> userspace daemon receives object/range
  -> daemon fetches and writes anonymous cache fd
  -> READ_COMPLETE ioctl
  -> kernel resumes EROFS read
```

`ioctl` 是进程向内核文件描述符发送设备/子系统特定控制命令的系统调用。这里它完成的不是传输数据本身，而是告诉 CacheFiles 某个 range 已可用。[SRC-EROFS-002]

## 普通 file-backed EROFS

```text
mount EROFS image file
  -> VFS read
  -> filesystem maps image block
  -> host page cache/storage fault-in
```

它要求 backing file 的内容已经可靠存在。sparse hole 会按文件系统稀疏语义读取为零，不会自动转成 registry fetch。

## EROFS + DAX over pmem

guest EROFS 使用 DAX 时，文件 offset 直接对应 pmem address range，文件数据不先进入 guest page cache。真正的缺失数据供应可在 VMM HVA/UFFD 层发生。这是 Conch 目标路径，与 CacheFiles 的 VFS request 路径不同。
