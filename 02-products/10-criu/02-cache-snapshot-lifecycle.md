# CRIU page image 与生命周期

> 阅读完成后，读者能够理解 lazy-pages daemon 的启动顺序、page source 生命周期和恢复失败边界。

恢复进程开始执行前，UFFD、page image/page server 和 daemon 必须都可用。daemon 需要处理 fault 请求，也可以在后台继续传输剩余页。[SRC-CRIU-001]

生命周期关键点：

- source page image 不能在未完成恢复时回收；
- page server 断连必须让 restore 失败可见；
- 同页并发 fault/background copy 要去重；
- UFFD event 与目标进程退出需要正确竞态处理；
- restore 完成后才能释放 lazy page source。

这里的 page image identity 属于 checkpoint lineage，不应和 container image digest 共用命名空间。
