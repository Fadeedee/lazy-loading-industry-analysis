# Checkpoint Template 三路恢复流程

> 阅读完成后，读者能够说明 checkpoint restore 的并行准备、统一 ready barrier、三种 fault 和失败回滚，并据此编写端到端验收用例。

## 资源解析

```text
RestoreSandbox(template name/id)
  -> Template store resolves immutable Boot Index digest
  -> validate component closure and captured VMM/config
  -> Rootfs resource: PreparedRootfs + EROFS descriptors
  -> Disk resource: writable snapshot parent chain
  -> Memory resource: full-v1 or incremental-v1 manifest
  -> VM assets: kernel/initrd/external state
```

## 并行准备

```text
Conch Restore Coordinator
  |-- A. lazyd restore/prepare rootfs instances
  |-- B. block backend open parent chain + create writable head
  |-- C. memory file or conch-cow Attach + inherited memfd
  `-- D. kernel/initrd/VMM state local availability
```

每路返回：resource identity、ready level、runtime handle/lease和failure channel。任一路准备失败，Conch取消其他任务并按逆序释放已获得资源。

## VMM 构造与 ready barrier

1. Conch生成 regular/lazy pmem、writable blk、guest memory和VM state参数。
2. StratoVirt建立 guest RAM和pmem HVA/AddressSpace/KVM memslots。
3. memory UFFD source与pmem UFFD handler分别完成交接/启动。
4. block backend确认可处理读写。
5. StratoVirt恢复设备/vCPU state，但保持paused。
6. guest-visible device identity与checkpoint记录匹配。
7. 所有 resource达到`Faultable`后，Conch允许resume。

## 运行期三类访问

| guest动作 | 触发 | source | completion |
| --- | --- | --- | --- |
| 执行缺失RAM页 | guest memory HVA UFFD | memory snapshot/conch-cow | memfd pwrite + wake |
| 读只读rootfs | EROFS+DAX pmem HVA UFFD | lazyd OCI cache | FD fixed remap + wake |
| 读写运行数据 | virtio-blk request | block parent/COW head | block completion |

## 创建后续 checkpoint

```text
pause/quiesce
  -> freeze/flush writable block head
  -> query memory dirty generation and export delta
  -> capture external VMM/device state
  -> publish disk + memory descriptors
  -> reuse existing immutable PreparedRootfs refs
  -> build/validate new Boot Index
  -> atomically advance checkpoint head
  -> install new disk/memory generations
  -> resume
```

只有 Boot Index和所有引用发布成功后才推进head。失败后原head仍可恢复；当前运行实例进入是否可继续checkpoint的状态必须明确。

## 验收场景

- cold/cold：三条source都未缓存；
- warm rootfs、cold memory；
- warm memory base、cold rootfs range；
- rootfs共享的两个VM、各自private writable/memory；
- parent disk/memory layer缺失；
- lazyd或conch-cow在首fault时退出；
- registry/object storage超时和内容校验失败；
- restore取消时没有遗留VMM、attachment、runtime snapshot或FD；
-新checkpoint失败后旧template仍可恢复。

该流程借鉴 E2B 的多路径组合、QEMU/CRIU 的 fault-priority 与 Conch PR #155 的 memory lineage，但使用本项目自己的 EROFS+pmem rootfs路径。[SRC-E2B-001] [SRC-QEMU-001] [SRC-CRIU-001] [SRC-CONCH-003]
