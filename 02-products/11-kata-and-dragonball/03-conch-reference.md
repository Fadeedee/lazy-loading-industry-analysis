# Kata/Dragonball 对 Conch 方案的借鉴

> 阅读完成后，读者能够用 Kata 的分层方法约束 Conch、StratoVirt、lazyd 和 guestd 的接口。

## 直接采用

- 明确区分 guest assets 与 workload image。
- host 编排只传稳定 metadata，实际 mount 由 guest agent 完成。
- VMM 不解析 OCI，image service 不管理 vCPU。
- snapshotter/content 生命周期与 sandbox 生命周期通过引用连接，不直接等同。

## 需要适配

- Conch 已有 containerd-native service/store 结构，应把 lazy metadata 挂在当前 image/snapshot ownership 上，而不是恢复旧 CLI 私有 JSON 流程。
- guestd 接收 `layer index -> pmem stable identity -> mountpoint`，不要依赖 `/dev/pmemN` 自然枚举顺序。
- StratoVirt 的 PCI/MMIO transport 必须与 machine type 和 guest kernel能力分别验证。

## 不建议采用

- 不为了模仿 Kata 引入第二套 shim/snapshotter ownership。
- 不把 Dragonball 当现成 lazy-pmem backend。
- 不让 guestd负责 registry auth、range cache 或 VM lifecycle。
