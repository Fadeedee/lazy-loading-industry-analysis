# E2B 对 Conch 方案的借鉴

> 阅读完成后，读者能够把 E2B 的完整沙箱资源模型映射到 Conch，同时保留 EROFS+DAX 的自有优势。

## 直接采用

- template/checkpoint 明确引用 memory、rootfs base 和 writable diff。
- sandbox startup 对多条 backend 做并行 prepare 和统一 ready barrier。
- fault-priority + bounded prefetch。
- memory diff 与 disk diff 分别生成、分发和恢复。
- template cache 与 sandbox-private COW 生命周期分离。

## 需要适配

- Conch 的 immutable lower 使用原生 EROFS+pmem+DAX，而不是把整盘都放进 NBD。
- writable ext4 upper/增量 disk 可借鉴 NBD/block COW，并继续通过 StratoVirt virtio-blk 暴露。
- StratoVirt 同时具备 pmem fault 与 memory restore 时，应保留两类 region/source。
- lazyd 可服务 immutable range，但 writable block diff 需要独立 object type/protocol。

## 不建议采用

- 不因为 E2B 使用 NBD 就放弃 EROFS file-page 映射复用目标。
- 不把 template ID 当 OCI digest。
- 不在第一阶段同时重建 E2B 的整套分布式控制面。

E2B 最值得采纳的是资源图和生命周期，不是具体设备选择。
