# 只读 Rootfs Lower 路线比较

> 阅读完成后，读者能够根据 guest 兼容性、触发层级、镜像格式和共享目标，在文件级、块级、内核 cache 与 pmem/DAX 路线之间做选择。

## 比较矩阵

| 路线 | 样本 | 触发 | 格式/索引 | guest page cache | VMM 修改 | 跨 VM file page 共享潜力 |
| --- | --- | --- | --- | --- | --- | --- |
| FUSE 文件级 | Nydus v2、stargz、SOCI | VFS/FUSE read | RAFS/eStargz/zTOC | 通常经过 | 否或很少 | 取决于 daemon cache，非直接 HVA file mapping |
| 块级 | OverlayBD、NBD | sector/block I/O | block layers/index | 经过 guest FS cache | block device backend | 共享后端 cache，不天然共享 guest/HVA 页 |
| EROFS+CacheFiles | Nydus fscache | kernel READ request | EROFS + cache object | 经过文件路径 | 否 | host kernel cache 可复用 |
| EROFS file-backed | containerd native EROFS | 本地 file fault | 原生 EROFS | 取决于 mount/DAX | pmem 时需要 | 同 inode+offset 有潜力 |
| EROFS+DAX+UFFD | Nydus UFFD、目标方案 | HVA missing fault | EROFS block view | 文件数据绕过 | 是 | FD+fixed mapping 可复用 file page |

来源：[SRC-NYDUS-001] [SRC-STARGZ-001] [SRC-SOCI-001] [SRC-OVERLAYBD-001] [SRC-EROFS-002] [SRC-EROFS-003] [SRC-NYDUS-004]

## 选择依据

### 文件级适用

- 需要按路径/文件做细粒度策略；
- 允许 FUSE/virtiofs 参与访问；
- 希望兼容普通 OCI gzip 或 RAFS 生态；
- 不要求 VMM 层直接共享 EROFS file page。

### 块级适用

- guest 需要 ext4 等普通读写文件系统；
- 可写 upper、block snapshot 和 COW 是主要对象；
- 希望 VMM 只暴露标准块设备；
- 可以接受文件语义不可见。

### pmem/DAX 适用

- lower 是严格只读 EROFS；
- guest kernel 支持 pmem、DAX 与目标 transport；
- 可以修改 VMM 处理 UFFD fault；
- 目标包括减少 guest page-cache duplication 和跨 VM file-backed reuse。

## 对当前方案的判断

Conch 的 workload rootfs lower 已选择原生 EROFS，因此 `EROFS+DAX+UFFD+FD remap` 与对象匹配。后续 writable ext4 disk 不应强行塞进该 readonly path，应作为 virtio-blk/COW 路径。若 guest kernel 或 machine type 不支持 pmem/DAX，必须保留 full/local fallback，而不是退回已废弃的 fscache on-demand 作为唯一方案。[SRC-EROFS-004]
