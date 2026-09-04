# CRIU lazy-pages 数据路径

> 阅读完成后，读者能够追踪恢复进程的一次 missing fault 到 lazy-pages daemon 和 page server。

```text
criu restore --lazy-pages
  -> create process VMAs without all pages
  -> register UFFD and hand it to lazy-pages daemon
  -> resume process
  -> process VA missing fault
  -> daemon locates page in image/page server
  -> UFFDIO_COPY/ZEROPAGE
  -> process continues
  -> optional background page transfer
```

`lazy-pages` 与远端 page server 组合时接近 post-copy migration：目标端先运行，源/存储端按请求提供页。[SRC-CRIU-001]

每个 fault 关联的是恢复进程 VMA 和 virtual address；这与 guest GVA 不同。VMM UFFD 注册的是自己的 HVA，KVM memslot 再把 guest GPA 引到该 HVA。
