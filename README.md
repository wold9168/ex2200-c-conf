# ex2200-c-conf

Juniper EX2200-C-12T-2G 交换机的配置仓库，含配套 Debian 路由器设置。

## 文件说明

| 文件 | 用途 |
|---|---|
| `configured.conf` | **交换机目标配置**（Junos 配置块格式）。用 `load merge` 提交到交换机 |
| `debian-router-config.txt` | Debian 路由器当前配置（interfaces、sysctl、服务等） |
| `in-progress/infact.conf` | 交换机迁移前运行配置，对比用 |
| `in-progress/apply-commands.txt` | 交换机 `set` 命令分步清单，从出厂状态到完整配置 |
| `in-progress/network-design.txt` | 网络拓扑、VLAN 规划、端口映射、路由走向 |
| `in-progress/debian-router-setup.txt` | Debian 路由器安装配置（单网口 VLAN 子接口 + NAT） |
| `in-progress/fix-routing.txt` | 删除交换机上一条冗余静态路由的修正步骤 |
| `in-progress/ipv6-plan.txt` | IPv6 ULA 地址规划 |
| `in-progress/ipv6-deploy.txt` | IPv6 交换机 + 路由器双端部署步骤 |
| `in-progress/juniper-h3c-note.txt` | Juniper / H3C 命令对照备忘 |
| `in-progress/log` / `log2` / `log3` | 串口启动日志（交换机启动故障排查记录） |
| `AGENTS.md` | 给 OpenCode 的指令文件 |

## 执行顺序

```
出厂状态
     │
      ├── 1. in-progress/apply-commands.txt   →  交换机基础配置
      │                              （VLAN、接口、DHCP、路由）
      │
      ├── 2. in-progress/debian-router-setup →  Debian 路由器
      │                              （VLAN 子接口、NAT、路由）
      │
      ├── 3. in-progress/fix-routing.txt     →  如有需要，删除冗余路由
      │
      └── 4. in-progress/ipv6-deploy.txt     →  IPv6 双端部署
                                    （交换机 inet6 + 路由器 radvd + NAT66）
```
