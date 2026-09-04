# Nydus v3 实验分支

> 阅读完成后，读者能够理解 Nydus v3 试图怎样把 EROFS、缓存和多种触发前端统一到一个核心，并正确看待其尚未稳定的成熟度。

## 定位与成熟度

该分支自述为 Nydus image format v3 的 ground-up Rust redesign，并明确警告磁盘格式、CLI 和 API 尚无兼容保证、不可用于生产。[SRC-NYDUSV3-001] 因此本文把它标为 `experimental`，不是 Nydus 当前稳定版替代品。

## 目标结构

- 每层成为自包含 EROFS blob artifact，可选 metadata-only bootstrap。
- dedup chunk 与 compression/read group 解耦。
- 统一 `NydusCore` 读取能力，由 FUSE、NBD、ublk、UFFD 和 fanotify 等前端调用。
- 共享 readiness bitmap、per-blob prefetch lock 和 trace-driven optimized blob。
- 面向 Kata/microVM 的目标路径是 EROFS over virtio-pmem + UFFD。

## 最值得关注的意义

它展示了一个比“为每种触发器各写一套下载逻辑”更合理的方向：**镜像解析、内容校验、cache 和 prefetch 属于共享 core；FUSE/块设备/fanotify/UFFD 只是不同触发与回复适配层**。[SRC-NYDUSV3-001]

来源：[SRC-NYDUSV3-001]
