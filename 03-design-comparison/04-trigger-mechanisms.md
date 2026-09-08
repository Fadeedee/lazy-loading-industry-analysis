# 触发机制比较

> 阅读完成后，读者能够在 FUSE、块 I/O、CacheFiles、fanotify、UFFD 和 postcopy 之间识别事件来源、可见语义与阻塞对象。

| 机制 | 监听对象 | 知道文件语义 | 阻塞对象 | 供应完成方式 | 典型用途 |
| --- | --- | --- | --- | --- | --- |
| FUSE | 文件操作 | 是 | calling thread | FUSE reply | container rootfs |
| block backend | sector request | 否 | I/O/vCPU | block completion | ext4/disk |
| CacheFiles | kernel cache range | 部分 | VFS read | write fd + ioctl complete | EROFS on-demand |
| fanotify pre-content | 文件访问前事件 | 是 | VFS access | daemon fills backing file/response | host filesystem cache |
| UFFD | registered process HVA | 否 | faulting thread/vCPU | COPY/ZEROPAGE/remap+wake | pmem/RAM |
| postcopy | missing migration page | RAM block/page | vCPU | page receive/install | VM memory restore |

来源：[SRC-NYDUS-001] [SRC-OVERLAYBD-001] [SRC-EROFS-002] [SRC-NYDUSV3-001] [SRC-FC-002] [SRC-QEMU-001]

## 选择不是只看 syscall 次数

要同时考虑：

- 需要文件路径还是只需 offset；
- guest kernel 是否必须改配置；
- fault handler 是否在安全 sandbox 内可访问 source；
- warm path 是否仍进入 daemon；
- 同一次 fault 是否需要网络、解压和校验；
- failure 能否终止等待者；
- 目标对象是 immutable data、writable block 还是 volatile RAM。

## 对本项目

- 当前目标暂不考虑 fanotify；已有普通容器/VFS 能力只属于历史兼容范围，不作为本次开发依赖；
- virtio-pmem+DAX 的 UFFD handler 位于 StratoVirt，lazyd 接收 seqpacket FETCH；
- 未来 writable snapshot 采用 block request frontend；
- 未来 memory restore 采用独立 UFFD/postcopy frontend；
- lazyd core 可以复用内容取数能力，但 frontend-specific metadata 不进入通用 cache identity。
