# N1 单臂旁路由适配评估

## 当前目标

在不增加 USB 网卡、不改变 BE5100 主路由拓扑、不覆盖现有 iStoreOS 配置的前提下，评估 Honk 在 N1 单臂旁路由上的可行性。

## 已知设备约束

- 设备：斐讯 N1
- 系统：iStoreOS
- 网络：单臂旁路由；主路由 DHCP 将 N1 设置为客户端默认网关
- 不做：修改主路由拓扑、增加 USB 网卡、在现有设备上直接安装测试包或改写防火墙/UCI 配置

## 已知内核问题

此前设备检查显示，当前 iStoreOS 内核没有启用 `CONFIG_NFT_QUEUE`。

Honk 当前文档说明：

1. 真实透明代理数据面需要 Linux 内核 **6.12 或更新版本**。
2. `nfqueue_enable` 默认开启并使用 NFQUEUE 320。
3. 如内核没有 NFQUEUE 支持，必须在 Honk 配置中显式关闭 `nfqueue_enable`；重新编译 Honk 用户态程序不能补出内核缺失的 `CONFIG_NFT_QUEUE`。
4. `lan_interface` 用于接管从 LAN 进入网关的转发流量；因此单臂模式需要验证实际入口接口、转发路径和回程路径，不能仅凭“支持 LAN interface”就认定兼容。

## 适配判断门槛

在生成可安装包之前，必须先确认：

- [ ] N1 当前运行内核版本是否达到 6.12。
- [ ] 若低于 6.12，确认 Honk 是否存在受支持的旧内核数据面方案；不能通过修改版本检测或强行加载 eBPF 来伪装兼容。
- [ ] 若内核无 `CONFIG_NFT_QUEUE`，验证关闭 `nfqueue_enable` 后，目标功能是否仍可完整工作。
- [ ] 明确单臂旁路由的 `lan_interface`、默认路由、策略路由及流量回程行为。
- [ ] 在隔离测试环境通过后，才考虑发布 N1 专用包。

## 当前阶段

本分支只记录适配要求和风险，不包含可安装固件或已验证可用的 Honk 补丁。GitHub Actions 的交叉编译流程应在内核兼容性确认后添加，避免产出“编译成功但设备无法运行”的误导性安装包。

## 参考资料

- Honk 上游：https://github.com/daeuniverse/honk
- Honk Linux 启动与内核要求：https://github.com/daeuniverse/honk/blob/main/doc/zh/how-to-start.md
- Honk Releases：https://github.com/daeuniverse/honk/releases
