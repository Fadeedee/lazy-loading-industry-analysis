# EROFS cache 与 snapshot 生命周期

> 阅读完成后，读者能够理解不可变 EROFS lower、active writable upper、cache object 和生命周期引用之间的关系。

## 不可变 lower

原生 EROFS layer 适合按 digest 共享。containerd native snapshotter 可将 layer 作为只读 parent，并在 active snapshot 上方使用 writable overlay。[SRC-EROFS-003]

## CacheFiles object

on-demand mode 由内核维护 cache object/range 的请求状态，用户态负责取数和 complete。其正确性依赖 daemon、kernel ABI 和 cache 生命周期共同成立，不只是创建一个 sparse file。[SRC-EROFS-002]

## 生命周期风险

- 内核版本升级可能改变可用路径；
- cache object identity 必须绑定正确 EROFS blob；
- mount、daemon 和 cache 删除顺序必须避免仍有 reader；
- active upper commit 形成的新内容与原 immutable lower 身份不同；
- DAX 与普通 page-cache mount 的一致性/写权限不能混用。

EROFS 本身不保存 guest RAM snapshot。snapshotter 的 parent/child 也不等同于 VMM memory lineage。
