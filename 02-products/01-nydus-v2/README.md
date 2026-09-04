# Nydus v2 与已合入 UFFD block service

> 阅读完成后，读者能够区分 Nydus v2 的 RAFS 文件级懒加载、EROFS fscache 接入和 2026 年合入的 UFFD block service，并判断哪些代码可直接参考。

## 定位

Nydus v2 是面向容器镜像的远程文件系统方案。RAFS 将文件系统元数据放在 bootstrap，将文件数据放在一个或多个内容 blob 中；运行时可通过 FUSE、virtiofs 或内核 EROFS 路径按需读取数据。[SRC-NYDUS-001] [SRC-NYDUS-002]

2026 年合入的 PR #1921 又增加了 RAFSv6 UFFD block service：VMM 把 guest pmem 对应的 UFFD 与 region 信息交给 Nydus，Nydus 可选择 `Copy` 或 `Zerocopy` 处理 fault。[SRC-NYDUS-004]

## 能力不能混为一项

| 能力 | 对象 | 入口 | 状态 |
| --- | --- | --- | --- |
| RAFS FUSE/virtiofs | rootfs lower | 文件请求 | `released` |
| RAFSv6 EROFS fscache | rootfs lower | CacheFiles on-demand read | `released`，但内核路线已转向 file-backed mount |
| RAFSv6 UFFD block service | pmem rootfs block view | VMM UFFD fault | `merged` |

Nydus v2 没有替代 VM memory snapshot restore；UFFD block service供应的是镜像文件系统对应的只读设备内容。

## 关键结论

- bootstrap 先到本地，文件数据按固定 chunk 从 blob 读取并校验。
- blob cache 避免重复远端读取；chunk digest 保证数据完整性。
- prefetch 用镜像内 hint 在后台拉取启动工作集。
- UFFD service 已包含协议、SCM_RIGHTS、Copy/Zerocopy、prefault 和 reconnect 相关实现，是当前最接近 Conch 目标路径的第一方代码证据。
- 当前本地代码的 zerocopy region 使用 `MAP_PRIVATE | MAP_FIXED`；不能据此直接宣称跨 VM 共享 dirty/COW 页。

详见[数据路径](01-data-path.md)、[缓存与生命周期](02-cache-snapshot-lifecycle.md)和[本项目借鉴](03-conch-reference.md)。

来源：[SRC-NYDUS-001] [SRC-NYDUS-002] [SRC-NYDUS-003] [SRC-NYDUS-004]
