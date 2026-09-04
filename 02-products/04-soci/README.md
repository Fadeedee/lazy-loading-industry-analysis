# SOCI snapshotter

> 阅读完成后，读者能够理解 SOCI 如何通过旁路索引为普通 OCI gzip layer 提供按需访问，并看清它与转换型镜像格式的取舍。

## 定位

SOCI 为原有 OCI image 生成独立 SOCI index 和每层 zTOC，不要求重写原 layer blob。运行时 snapshotter 根据 zTOC 将文件读取定位到压缩流 span，再从 registry 按范围读取。[SRC-SOCI-001] [SRC-SOCI-002]

## 核心取舍

- 优点：原 OCI layer 可保持不变，索引可旁路发布。
- 代价：索引与原 manifest/layer 的绑定、发现和 GC 更复杂。
- SOCI v2 强化索引到 image manifest 的绑定，并支持 span prefetch；默认文档示例中 span size 为 4 MiB，较小 layer 可不生成 zTOC。[SRC-SOCI-002]

## 边界

SOCI 是文件级 rootfs lower 方案，不提供 virtio-pmem HVA remap，也不恢复 guest RAM。它对 Conch 最有价值的是“descriptor 与派生索引身份分离”及 registry auth/observability 的工程经验。

来源：[SRC-SOCI-001] [SRC-SOCI-002] [SRC-SOCI-003]
