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

## DHCPv6 配置

采用 **SLAAC + 无状态 DHCPv6** 方案：设备通过 SLAAC 自动获取地址，通过 DHCPv6
获取 DNS 等配置信息（M=0, O=1）。

### 拓扑

```
客户端 (VLAN 12)
   │  RA（前缀 fd00:12::/64, O 标志）
   │  DHCPv6（relay → fd00:10::2）
   ▼
交换机 vlan.12              ← RA、DHCPv6 relay
   │      (DHCPv6 relay group VLAN12 → server-group ROUTER)
   │
   ▼
路由器 enp0s31f6.10         ← wide-dhcpv6-server
   │      fd00:10::2
```

### 交换机侧（configured.conf）

```
# RA — 宣告前缀 + 告知设备通过 DHCPv6 获取其他配置
protocols router-advertisement interface vlan.12 {
    other-stateful-configuration;           # O 标志
    prefix fd00:12::/64;
}

# DHCPv6 relay — 将客户端 DHCPv6 请求中继到路由器
forwarding-options dhcp-relay dhcpv6 {
    group VLAN12 {
        interface vlan.12;
    }
    server-group ROUTER {
        fd00:10::2;
    }
    active-server-group ROUTER;
}

# IPv6 默认路由 — 非 VLAN 12 的流量经路由器转发
routing-options rib inet6.0 static {
    route ::/0 next-hop fd00:10::2;
}
```

### 路由器侧（/etc/wide-dhcpv6/dhcp6s.conf）

```
# 所有客户端共享的 DNS 配置
option domain-name-servers fd00:10::2, 2001:4860:4860::8888;
option domain-name "home.lan";

# VLAN 12 — relayed（interface-id 匹配 "vlan.12"）
interface vlan.12 {
};

# VLAN 10 — 直达
interface enp0s31f6.10 {
};
```

### 工作流程

1. VLAN 12 设备接入，发送 RS
2. 交换机回复 RA（包含前缀 `fd00:12::/64`、O 标志）
3. 设备用 SLAAC 生成 `fd00:12::` 地址 + 默认路由指向交换机
4. 设备发送 DHCPv6 INFORMATION-REQUEST
5. 交换机 relay 到路由器 `fd00:10::2`
6. 路由器回复 DNS 等信息，经交换机转回设备

### 要点

- VLAN 12 设备不需要 inet6 地址在交换机之外 —— DHCPv6 relay 走 vlan.10
  到路由器，无需将 VLAN 12 加入 trunk
- 路由器 radvd 仅宣告 VLAN 10 的前缀，VLAN 12 的 RA 由交换机负责
