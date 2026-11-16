# N1 单臂旁路由适配评估

## 当前目标

评估 Honk 在斐讯 N1 / iStoreOS 上作为单臂旁路由的可行性。目标网络为：BE5100 主路由通过 DHCP 将 N1 下发为客户端默认网关；不增加 USB 网卡，不改变现有主路由拓扑。

**重要：Honk 上游当前明确说明单臂网关设置不受支持，相关问题和 bug 报告不在上游支持范围内。** 本项目属于自行研究的实验性适配，不能宣称官方支持或保证可用。

## 已知设备信息（用户实测）

- 设备：斐讯 N1
- 系统：iStoreOS
- 内核：`6.12.112`
- 架构：`aarch64`
- `CONFIG_NETFILTER_NETLINK_QUEUE=m`
- `CONFIG_NFT_QUEUE` 未启用
- 内存：约 1.9 GiB；Swap：0

内核版本满足 Honk 6.12+ 的版本门槛，aarch64 也属于上游发布覆盖的 Linux 架构。但这不代表内核配置或单臂数据路径已满足要求。

## 内核配置与 NFQUEUE

Honk 实际透明代理数据面要求至少具备以下能力（按上游启动指南核查）：

```
CONFIG_BPF=y
CONFIG_BPF_SYSCALL=y
CONFIG_BPF_JIT=y
CONFIG_CGROUP_BPF=y
CONFIG_NET_SCHED=y
CONFIG_NET_CLS_ACT=y
CONFIG_NET_XGRESS=y
CONFIG_NET_NS=y
```

默认启用的 held-first-packet UDP 路径还要求：

```
CONFIG_NF_TABLES=y|m
CONFIG_NF_TABLES_INET=y|m
CONFIG_NFT_CT=y|m
CONFIG_NFT_QUEUE=y|m
```

本机已确认 `CONFIG_NFT_QUEUE` 未启用，因此不能假设默认 NFQUEUE UDP 首包暂存功能可用。Honk 文档提供 `global { nfqueue_enable: false }` 作为关闭该路径的选项；这只关闭 NFQUEUE 暂存，不代表能弥补其他内核缺项，也不等于已经验证完整 UDP 透明代理。

`CONFIG_NETFILTER_NETLINK_QUEUE=m` 仅表示 Netlink queue 支持编译为模块；还需检查模块是否存在、能否加载，以及运行时队列是否可用。它不能替代 `CONFIG_NFT_QUEUE`。

## 单臂网关上游限制与参数

Honk 上游 Quick Start 的 Single-arm gateways 章节明确指出：单臂网关设置不受支持，相关问题和 bug 报告不予回答。

上游说明单臂场景中，LAN 与 WAN 是同一个接口/网桥，主机将客户端流量经同一接口发回上游路由器。上游列出的额外 sysctl 要求为：

```
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.<LAN接口>.send_redirects = 0
net.ipv4.conf.all.rp_filter = 0
```

这里的 LAN 接口必须以设备实测为准。Honk 仅在首个绑定 LAN 接口处理部分 rp_filter 设置，且内核 all/interface 取值规则可能使 all 的设置覆盖接口值。

**本项目当前禁止直接在现网写入上述 sysctl、改防火墙、启动 Honk 或安装测试包。** 必须先审计现有接口、路由、sysctl、内核选项，并准备回滚/带外管理方式。

## 下一步诊断清单

- [ ] 确认实际 LAN bridge 和设备接口名（例如 br-lan，不能预设）。
- [ ] 检查上列 BPF、TCX、namespace 内核选项。
- [ ] 检查 `nfnetlink_queue` 模块文件及加载状态。
- [ ] 检查 nftables 对 queue 表达式的支持；预期当前 `CONFIG_NFT_QUEUE` 缺失。
- [ ] 记录 `send_redirects`、`rp_filter` 当前值，不做修改。
- [ ] 评估在隔离测试环境中 `nfqueue_enable: false` 的功能边界。
- [ ] 决定是否值得维护本地单臂适配补丁；不能把修改参数描述为完整的上游支持。

## 构建策略

aarch64 是上游 Linux 发布架构之一，因此优先评估上游现成二进制是否可用于用户态功能验证。若需要自编译，必须使用包含真实 eBPF 构建步骤和嵌入资源的上游构建流程；普通 `cargo build --release` 不会自动生成真实数据面所需的 eBPF。

在内核选项和单臂限制未验证前，不发布标记为“可安装/可用”的 N1 包。GitHub Actions 首阶段只用于可重复构建与产物检查，不自动部署到路由器。

## 参考资料

- 上游项目：https://github.com/daeuniverse/honk
- 官方 Quick Start（内核要求、Single-arm gateways）：https://daeuniverse.org/honk/how-to-start
- 官方 NFQUEUE 设计：https://daeuniverse.org/honk/design/nfqueue
- 官方配置参考：https://daeuniverse.org/honk/reference/global
