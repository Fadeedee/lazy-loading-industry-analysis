# SOCI 索引、缓存与生命周期

> 阅读完成后，读者能够理解旁路索引为何引入额外身份和 GC 关系，以及 SOCI 没有处理的写时快照和内存恢复边界。

## 身份关系

SOCI 至少涉及三类对象：image manifest、原 OCI layer blob、派生 SOCI index/zTOC。运行时必须确认索引指向正确的 manifest 和 layer digest，不能只按 tag 选择。[SRC-SOCI-002]

## 缓存

远端仍是原始 layer，节点缓存已下载 span。相同 layer digest 和 zTOC 可跨容器复用。registry token、scope 和 retry 属于 control/data plane 的共同基础；v0.15.0 release 也包含 auth、可观测性和 FD 生命周期相关改进。[SRC-SOCI-003]

## 生命周期

- build/create index；
- push index artifact；
- pull image 时发现索引；
- mount 后按需读取 span；
- image/index/layer 需要关联 GC。

可写 upper 仍由 overlay snapshot 管理，VM memory checkpoint 不在 SOCI 范围内。
