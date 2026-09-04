# EROFS、CacheFiles on-demand 与 native snapshotter

> 阅读完成后，读者能够区分 EROFS 文件系统、CacheFiles 按需填充机制和 containerd native EROFS snapshotter，并理解 fscache 路线的版本风险。

## 三个概念

1. **EROFS**：只读文件系统格式，负责 inode、目录和文件数据布局。[SRC-EROFS-001]
2. **CacheFiles on-demand**：内核向用户态发 range READ 请求，daemon 填充 cache fd 并通过 ioctl 标记完成。[SRC-EROFS-002]
3. **containerd EROFS snapshotter**：以原生 EROFS layer/file-backed mount 组织 containerd snapshot，可配合 active overlay upper。[SRC-EROFS-003]

它们可以组合，但不是同一个组件。

## 成熟度提示

EROFS 与 native snapshotter 是当前方向；EROFS 官方资料说明 fscache-based on-demand mode 自 Linux 6.12 起废弃，file-backed mount 是替代方向。[SRC-EROFS-004] 因此 CacheFiles 仍值得研究事件/完成协议，但不适合成为新方案唯一长期基础。

来源：[SRC-EROFS-001] [SRC-EROFS-002] [SRC-EROFS-003] [SRC-EROFS-004]
