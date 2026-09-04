# 实验项目数据路径

> 阅读完成后，读者能够准确描述三个案例实际存在的数据路径，而不会把 issue 目标当作已实现功能。

## CubeSandbox

当前公开 rootfs 代码构造 host lowerdir，并通过 virtiofs 等方式让 guest 使用；相关讨论还涉及 ext4 image 与 XFS reflink clone。[SRC-CUBE-002] [SRC-CUBE-003] EROFS+pmem 请求 #274 已关闭未计划，不能画成现有路径。[SRC-CUBE-001]

## OpenSandbox

规范暴露 snapshot resource state 与 restore-snapshot API。[SRC-OPENSANDBOX-001] 公开 API 没有给出 page fault、block fetch 或 rootfs range 数据面证据，因此本文只把它用于 control-plane 状态参考。

## urunc shared snapshot view

讨论/原型通过共享只读 snapshot view、lease 与 fallback 减少重复准备；实测讨论指出对小镜像，额外 snapshot/view 管理延迟可能抵消复用收益。[SRC-URUNC-001]
