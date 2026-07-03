# EX2200-C 配置仓库

## 设备

- **型号**: Juniper EX2200-C-12T-2G（12 口，非 PoE，无风扇）
- **Junos**: 12.3R9.4（降级到 12.3R6.6 备份分区启动，后重启恢复）
- **CPU**: Marvell 88F6281 ARM, 512 MB DRAM
- **管理口 me0**: `192.168.11.2/24`，独立于 VLAN，直连无 VLAN 的设备（检修用）

## 拓扑

```
VLAN 9  (upstream)  ← ge-0/1/0/1  ← ISP (untagged, access)
VLAN 10 (routing)   ← ge-0/0/0 trunk (tagged 9 10) ← Debian 路由器
VLAN 11 (management) ← 暂空
VLAN 12 (personal)  ← ge-0/0/1-11 ← 个人设备
VLAN 13 (vm)        ← 暂空
```

## 文件

| 文件 | 说明 |
|---|---|
| `README.md` | 仓库说明、执行顺序与 DHCPv6 配置详解 |
| `configured.conf` | **交换机当前目标配置**（完整 Junos 配置块格式） |
| `debian-router-config.txt` | Debian 路由器当前配置（interfaces、sysctl、服务等） |
| `migration-guide.md` | 网络迁移指南（含 PPPoE 拨号配置章节） |
| `in-progress/infact.conf` | 交换机迁移前运行配置（`show configuration` 输出） |
| `in-progress/apply-commands.txt` | 交换机 set 命令分步清单（须提交 2 次，me0 会断会话） |
| `in-progress/network-design.txt` | 完整网络拓扑、VLAN 规划、端口映射、路由走向 |
| `in-progress/debian-router-setup.txt` | Debian 路由器首次配置步骤（单网口 + VLAN 子接口） |
| `in-progress/fix-routing.txt` | 删除交换机上冗余的 `route 192.168.10.0/24` 静态路由 |
| `in-progress/ipv6-plan.txt` | IPv6 初步规划（ULA 地址方案） |
| `in-progress/ipv6-deploy.txt` | IPv6 部署步骤（交换机 + 路由器双端配置） |
| `in-progress/juniper-h3c-note.txt` | Juniper 与 H3C 同 OEM 源，指令多通用 |
| `in-progress/log` | 首次串口启动日志（PAM 崩溃、备份分区启动） |
| `in-progress/log2` | 第二次串口日志（备份分区警告、SSH 提示） |
| `in-progress/log3` | 正常启动日志（`request system reboot slice alternate` 修复后） |

## 故障时间线

| 序号 | 问题 | 诊断 | 解决 |
|---|---|---|---|
| 1 | 串口输入无响应 | minicom Flow Control 默认开启 | `Ctrl+A → O → Serial port setup → Hardware Flow Control: No` |
| 2 | `login:` 回车后死机 ALM 红灯 | 双分区版本不一致 → `mgd: error: rename failed for /var/etc/pam.conf` | `request system reboot slice alternate` 切回主分区 |
| 3 | J-Web CLI 工具提交配置报错 `L3-interface must be a vlan.xx interface` | EX2200 在 Junos 12.3 上用 `vlan.xx` 而非 `irb.xx` | 全部 `irb` 替换为 `vlan` |
| 4 | VLAN 12 设备 ping 不通 192.168.10.2 | 交换机多了一条 `route 192.168.10.0/24 next-hop 192.168.10.1`——画蛇添足 | `delete routing-options static route 192.168.10.0/24` |
| 5 | Debian 路由器 `dhclient: command not found` | 未安装 DHCP 客户端 | `apt install isc-dhcp-client` |
| 6 | Debian 路由器 `Cannot find device "eth0"` | systemd 预测性命名，接口名为 `enp0s31f6` | 全部 `eth0` 替换为 `enp0s31f6` |
| 7 | Debian 路由器安装 isc-dhcp-server 时报错退出 | dhcpd.conf 未配置，启动即失败 | 不影响——计划中 DHCP 由交换机负责，路由器不跑 DHCP 服务 |
| 8 | 客户端无 IPv6 | 交换机和路由器均未配置 IPv6 | ULA + NAT66 方案（见 `ipv6-deploy.txt`） |

## 注意事项

- **me0 IP 变更会断开当前 web 会话**，需手动配静态 IP 重连
- J-Web 的 CLI 工具可能无响应，此时只能改文本配置编辑器
- Console 口出厂默认 RJ-45（9600 8N1），`set port-type` 可切换 Mini-USB 但需重启
- 硬件复位：上电时按住 Mode 按钮直到 SYS LED 变橙色再松手
- 双分区（disk0s1/disk0s2），升级时须确保版本一致
- PoE 对此设备（12T）无效，`poe_attach: returned 19` 是预期行为
- EX2200 Junos 12.3 的 VLAN L3 接口名是 `vlan.xx`**不是** `irb.xx`
- `configured.conf` 须用 `load merge` 提交（密码 hash 在文件内，`load override` 会丢失未列出的配置）
- Debian 路由器单个网口 `enp0s31f6`，VLAN 子接口通过 trunk 与交换机通信
- 交换机 trunk ge-0/0/0 仅带 VLAN 9 + 10；VLAN 12 的 IPv6 通过 DHCPv6 relay 解决，不进 trunk
- VLAN 12 的 DHCP（IPv4）由交换机负责；DHCPv6 由路由器 relay 上的 wide-dhcpv6-server 负责
- VLAN 10 无 DHCP，路由器用静态 IP `192.168.10.2/24`
- ⚠️ **Debian 13 sysctl 坑**：`systemd-sysctl` 只读 `/etc/sysctl.d/*.conf`，不读 `/etc/sysctl.conf`。
  持久化内核参数必须写到 `/etc/sysctl.d/` 下，否则重启后丢失。

## 关键命令备忘

```cli
# 操作模式
show vlans                           # VLAN 列表
show vlans detail                    # VLAN 详细信息
show ethernet-switching table        # MAC 地址表
show interfaces terse                # 接口状态一览
show interfaces terse vlan           # vlan 接口状态
show configuration | display set     # set 格式配置
show chassis alarms                  # 硬件告警
show system alarms                   # 系统告警
show ipv6 route                      # IPv6 路由表
show log messages                    # 系统日志

# 配置模式
configure
commit confirmed <minutes>           # 提交并设回滚定时器
rollback <n>                         # 回滚
load merge /var/tmp/<file>           # 合并加载配置文件
load override /var/tmp/<file>        # 覆盖加载配置文件
```
