# CRIU lazy-pages

> 阅读完成后，读者能够理解 CRIU 如何对 Linux 进程内存执行 post-copy 式懒恢复，以及这对 VM 内存 handler 的启发和边界。

## 定位

CRIU `--lazy-pages` 在 restore 时为进程地址空间注册 userfaultfd，先恢复执行；缺页由 `lazy-pages` daemon 从本地 image 或远端 page server 取回并注入。[SRC-CRIU-001]

## 与 VMM 的关系

CRIU 处理的是 Linux process VA，而 StratoVirt 处理的是承载 guest GPA 的 VMM HVA。二者都使用 UFFD missing event，但地址归属、状态文件和失败对象不同。

## 边界

CRIU 不提供 OCI rootfs range cache、virtio-pmem 或 EROFS+DAX。它的价值主要是 post-copy 生命周期、page server 和 fault/background copy 协调。

来源：[SRC-CRIU-001]
