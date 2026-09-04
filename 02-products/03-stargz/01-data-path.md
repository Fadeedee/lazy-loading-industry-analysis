# stargz 数据路径

> 阅读完成后，读者能够从应用文件读取追踪到 eStargz TOC、HTTP Range、chunk 校验和 FUSE 回复。

```text
container open/read
  -> mounted remote snapshot
  -> stargz FUSE filesystem
  -> TOC: file offset -> compressed chunk/range
  -> registry HTTP Range
  -> 解压 + chunk digest 校验
  -> memory/disk cache
  -> FUSE reply
```

eStargz TOC 位于可优先发现的位置，记录文件和 chunk 信息；格式仍可被普通 gzip/tar 工具处理，但不理解扩展的工具不会得到 lazy/prefetch 优势。[SRC-STARGZ-002]

## 粒度与开销

- 触发粒度是文件 read；
- 下载粒度是压缩 chunk/range；
- 首次读取有 FUSE、网络和解压开销；
- warm cache 读取避免远端访问；
- 小文件散布会增加 range 数量，构建期排序和 prioritized files 用于改善启动工作集。

这是一条 guest/host VFS 路径，不建立 pmem GPA 到 VMM HVA 的映射，也不处理 UFFD event。
