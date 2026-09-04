# 术语表

> 阅读完成后，读者能够用统一词义讨论镜像、块设备、地址空间和快照懒加载，避免把下载、映射、恢复和物理页共享混为一谈。

## 数据对象

- **rootfs lower**：镜像提供的只读根文件系统层。
- **writable upper**：承载运行时文件写入的可写层，可能由 overlay upperdir 或可写块设备实现。
- **guest RAM**：客户机运行时看到的内存，其内容由宿主机 VMM/KVM 后端承载。
- **snapshot lineage**：基础快照与后续增量快照之间的父子依赖关系。

## 请求与填充

- **fault**：访问尚无可用映射或物理页的虚拟地址时产生的缺页事件。
- **trigger**：使系统发现某段数据尚未就绪的入口，例如 VFS 请求、块 I/O、DAX fault 或 UFFD event。
- **materialization**：把远端内容变成可在本地持久或稳定读取的数据范围。
- **mapping**：建立虚拟地址到某个内存对象或文件区间的对应关系。
- **resolution**：补齐缺失数据并让被阻塞访问继续执行的完整动作。

## 缓存与身份

- **content identity**：由内容本身确定的稳定身份，例如 OCI digest；它不等同于 VM 或 sandbox ID。
- **sparse cache**：逻辑长度完整、但未就绪范围不占实际磁盘块的稀疏文件。
- **host page cache**：宿主机内核按文件 inode 和 offset 缓存的文件页。
- **guest page cache**：客户机内核缓存块设备/文件系统数据的页；DAX 文件访问通常绕过这一路径。
- **download deduplication**：多个请求避免重复从远端下载。
- **physical-page sharing**：多个映射最终引用相同宿主机文件页；它比下载去重更强。

## 关键机制

- **UFFD/userfaultfd**：Linux 允许用户态处理指定虚拟地址范围缺页事件的接口。
- **DAX**：文件系统直接访问持久内存映射，绕过传统文件数据 page cache 路径。
- **fscache**：Linux 内核为网络或按需文件系统数据提供的本地缓存框架。
- **post-copy**：先恢复执行，再在访问缺失内存页时从来源端或快照后端取页。
- **SCM_RIGHTS**：Unix domain socket 用于在进程间传递已打开文件描述符的控制消息机制。
- **fixed remap**：使用 `mmap(MAP_FIXED)` 等方式将既有虚拟地址子区间替换为新的 file-backed 映射。

