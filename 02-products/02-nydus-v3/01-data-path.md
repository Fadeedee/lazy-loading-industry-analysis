# Nydus v3 数据路径

> 阅读完成后，读者能够理解 v3 的 EROFS-native artifact、O(1) logical lookup、压缩组和多前端如何组合。

## Artifact

实验分支把一层组织为 `data + bootstrap + blob meta + footer` 的自包含 blob，并以 SHA256 命名；也可以生成单独的 metadata bootstrap。启动先取得紧凑元数据，文件数据继续按需拉取。[SRC-NYDUSV3-001]

## Core read path

```text
frontend request
  -> EROFS logical address
  -> O(1) group lookup
  -> backend range read
  -> CRC32C validation
  -> zstd decode
  -> shared cache/readiness
  -> frontend-specific completion
```

`--chunk-size` 控制 BLAKE3 去重单位，`--compress-size` 控制压缩和读取单位，默认描述为 4 MiB。把二者分开可以改善压缩，但一次小读取仍可能产生 compression-group 级下载放大。[SRC-NYDUSV3-001]

## 多前端

- FUSE：用户态文件请求；
- NBD/ublk：内核 EROFS 产生块请求；
- fanotify：内核文件访问前事件；
- UFFD：virtio-pmem/DAX 对应 HVA fault；
- embeddable core：VMM 或其他进程直接复用 read path。

这些前端不是同时作用于同一次访问，而是对同一 core 的不同部署选择。
