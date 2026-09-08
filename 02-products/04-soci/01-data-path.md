# SOCI 数据路径

> 阅读完成后，读者能够从普通 OCI gzip layer、SOCI index 和 zTOC 追踪到一次 span 按需读取。

```text
OCI manifest + unchanged layer blobs
  -> discover SOCI index
  -> load per-layer zTOC
  -> file read maps to span
  -> registry HTTP Range on original blob
  -> gzip state reconstruction/decode
  -> verification/cache
  -> FUSE reply
```

zTOC 记录可跳转位置和文件到 span 的关系，使运行时不必从 gzip 流头部顺序解压到目标文件。[SRC-SOCI-001]

## 粒度

- 文件 read 是触发粒度；
- span 是主要远端读取与缓存粒度；
- 一个 span 可包含多个文件片段；
- v2 prefetch 可在启动阶段提前拉取 span。

span 越大，range 请求越少但小读取下载放大越高。SOCI span 还受 gzip seek/decode 布局约束，不是任意可选的文件页大小；不能直接把默认值搬到其他镜像格式。
