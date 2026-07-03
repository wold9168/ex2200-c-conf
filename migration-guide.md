# EX2200-C 网络迁移指南

## 当前网络拓扑

```
                    ISP（192.168.1.0/24）
                     │
          ge-0/1/0 ──┤├── ge-0/1/1（access VLAN 9）
                     │
               EX2200-C-12T-2G
               ┌──────┴──────┐
               │              │
         ge-0/0/0 trunk      ge-0/0/1-11 access
      (tagged 9 10 12)    VLAN 12（192.168.12.0/24）
               │              │
         Debian 路由器        个人设备
    enp0s31f6.9  = VLAN 9     192.168.12.10-253
    enp0s31f6.10 = VLAN 10    fd00:12::/64
    enp0s31f6.12 = 未启用
               │
          me0 192.168.11.2/24（独立管理口）
```

## VLAN 规划

| VLAN | ID | 子网 | 说明 |
|---|---|---|---|
| upstream | 9 | 192.168.1.0/24（ISP） | 上行，路由器 DHCP |
| routing | 10 | 192.168.10.0/24 | 交换机↔路由器 transit |
| management | 11 | 192.168.11.0/24 | 暂空，me0 同网段 |
| personal | 12 | 192.168.12.0/24 + fd00:12::/64 | 个人设备 |
| vm | 13 | 192.168.13.0/24 + fd00:13::/64 | 暂空 |

## 迁移到新环境需要修改的地址

以下地址**硬编码**，新环境下必须重新指定：

### 交换机（configured.conf）

| 位置 | 当前值 | 说明 |
|---|---|---|
| `vlan.10 family inet` | 192.168.10.1/24 | 交换机侧 routing 网段 |
| `vlan.11 family inet` | 192.168.11.1/24 | management 网段 |
| `vlan.12 family inet` | 192.168.12.1/24 | personal 网段 |
| `vlan.13 family inet` | 192.168.13.1/24 | vm 网段（暂空） |
| `vlan.10 family inet6` | fd00:10::1/64 | IPv6 ULA |
| `vlan.12 family inet6` | fd00:12::1/64 | IPv6 ULA |
| `me0` | 192.168.11.2/24 | 管理口 |
| `route 0/0` | 192.168.10.2 | 默认路由指向路由器 |
| `rib inet6.0 route ::/0` | fd00:10::2 | IPv6 默认路由 |
| `dhcp pool` | 192.168.11.0/24, 192.168.12.0/24 | DHCP 地址池 |
| `router-advertisement` | fd00:12::/64 | IPv6 RA 前缀 |

### 路由器（debian-router-config.txt）

| 位置 | 当前值 | 说明 |
|---|---|---|
| `enp0s31f6.9` | DHCP | 新 ISP 可能静态 IP 或不同子网 |
| `enp0s31f6.10` | 192.168.10.2/24 | 与交换机 vlan.10 一致 |
| `enp0s31f6.9` IPv6 默认路由 | fe80::1 | ISP 网关 link-local 地址 |
| `up ip -6 route add fd00:xx::/64` | fd00:10::1 | 回程路由下一跳为交换机 |

### ISP 依赖项

| 项目 | 迁移时需要确认 |
|---|---|
| VLAN 9 的 ISP 接入方式是 DHCP 还是静态 IP | 若静态需改路由器 enp0s31f6.9 配置 |
| ISP 网关的 fe80::1 是否相同 | 大概率不同，需用 `ip -6 neigh` 找新网关 |
| ISP 是否分配公网 IPv6 | 若不分配，IPv6 NAT66 不可用 |
| 192.168.1.0/24 是否冲突 | 新环境的 ISP 子网可能不同 |

## 迁移步骤

### 准备阶段

1. 在新环境部署好物理连线：
   - 交换机 ge-0/1/0 或 ge-0/1/1 → 新 ISP 设备
   - 交换机 ge-0/0/0 → Debian 路由器
   - 交换机 ge-0/0/1~11 → 个人设备
   - me0 → 管理笔记本（可选）
2. 确定新 ISP 的接入方式（DHCP / 静态 IP）
3. 确定新环境的 IP 规划（是否沿用 ULA / 用新 ULA）

### 交换机配置

1. 通过 me0 或串口（9600 8N1）登录交换机
2. 确认出厂设置未被篡改 → `show configuration`
3. 修改 `configured.conf` 中的 IP 地址为新环境
4. 上传配置 → `load merge /var/tmp/configured.conf; commit`

### 路由器配置

1. 修改 `/etc/network/interfaces`：
   - 更新 `enp0s31f6.9` 的接入方式
   - 更新 `enp0s31f6.10` 的静态 IP（与交换机 vlan.10 一致）
   - 更新 `up ip route add` 的下一跳
   - 更新 `up ip -6 route add` 的下一跳
2. 更新 `/etc/radvd.conf` 中的前缀（若 ULA 更换）
3. 更新 `/etc/wide-dhcpv6/dhcp6s.conf` 中的 DNS / 前缀
4. 启用 IP 转发：`sudo sysctl -p /etc/sysctl.d/99-ipforward.conf`
   （⚠️ Debian 13 的 `systemd-sysctl` 只读 `/etc/sysctl.d/*.conf`，
    不读 `/etc/sysctl.conf`。必须将参数写在 `sysctl.d/` 下。）
5. 重启网络：`sudo systemctl restart networking`

### 验证

1. 路由器 ping 通交换机：`ping 192.168.10.1`
2. 路由器 ping 通 ISP：`ping 192.168.1.1`
3. 本机 ping 通路由器：`ping -I enp5s0f4u1u1u1 192.168.10.2`
4. 本机 ping 通外网：`ping -I enp5s0f4u1u1u1 8.8.8.8`
5. IPv6：路由器 ping ISP 网关 → `ping6 240e::xxxx`
6. IPv6 本机：`ping6 -I enp5s0f4u1u1u1 2001:4860:4860::8888`

## 迁移时大概率需要修改的文件

| 文件 | 修改原因 |
|---|---|
| `configured.conf` | IP 地址、DHCP 池、路由、RA 前缀 |
| `debian-router-config.txt` | interfaces IP、默认路由下一跳 |
| `in-progress/ipv6-deploy.txt` | ULA 前缀、RA 配置 |
| `in-progress/network-design.txt` | 新拓扑 |

无需修改：`juniper-h3c-note.txt`、`log`、`fix-routing.txt`
