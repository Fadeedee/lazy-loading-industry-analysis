# OverlayBD 数据路径

> 阅读完成后，读者能够从 guest 文件读取追踪到块设备 lower/upper、远端 range 和本地块缓存。

```text
guest application read/write
  -> ext4/VFS
  -> block I/O
  -> virtual block device
  -> OverlayBD layer lookup
       read: upper miss -> lower chain -> remote/cache
       write: allocate/update writable upper
  -> complete block request
```

## 读取

逻辑块按 layer chain 从新到旧解析。命中本地 cache 时无需远端读取；未命中时按 range 拉取，并可结合 prefetch 改善顺序启动工作集。[SRC-OVERLAYBD-001]

## 写入

写请求进入私有 writable layer，而只读镜像 lower 保持可共享。对 ext4 来说，它面对的是正常块设备，不需要知道底层是远端 layer。

## 粒度与放大

- guest 请求粒度由文件系统和块层决定；
- 远端读取可按更大 range 合并；
- 小随机 I/O 可能增加请求数；
- 文件系统 metadata 更新会形成块级脏数据；
- cache/prefetch 不具备文件路径语义，需由 trace 或上层 hint 补充。
