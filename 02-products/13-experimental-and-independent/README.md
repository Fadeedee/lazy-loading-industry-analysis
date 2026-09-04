# 实验项目与反例

> 阅读完成后，读者能够从 CubeSandbox、OpenSandbox 和 urunc 的公开实现或讨论中识别“未实现能力”“只有 API”与“原型性能反例”。

## 纳入原则

这些项目不用于证明某条机制已经成熟，而用于校正设计假设：

- CubeSandbox：EROFS/pmem 诉求未被接受，当前 rootfs 仍以 ext4/virtiofs/reflink 为主。
- OpenSandbox：定义 snapshot lifecycle API，但 API 不证明底层采用 lazy restore。
- urunc：shared snapshot view 原型展示 lease/fallback，也显示小镜像可能因额外层变慢。

来源：[SRC-CUBE-001] [SRC-CUBE-002] [SRC-CUBE-003] [SRC-CUBE-004] [SRC-OPENSANDBOX-001] [SRC-URUNC-001]
