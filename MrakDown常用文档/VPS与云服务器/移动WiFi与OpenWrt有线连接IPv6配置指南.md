# 中国电信 5G 移动 Wi-Fi + OpenWrt 有线连接 IPv6 配置指南

> 适用日期：2026-09-05
> 
> 当前已知拓扑：
>
> ```text
> 中国电信 5G 移动 Wi-Fi
>   LAN IPv4：192.168.0.1
>   IPv6：240e:476:b488:5829:b467:94ff:fe4f:b5f4（设备页面显示的示例地址）
>       │ 网线
>       ▼
> OpenWrt
>   LAN IPv4：192.168.0.2/24
>       │
>       ▼
> 电脑/无线客户端
> ```

## 0. 目标与重要结论

目标是让电脑获得真正可用的全球 IPv6 地址和默认 IPv6 路由，从而连接日本 IPv6 节点：

```text
电脑全球 IPv6
  → 日本 [2001:470:24:7c2:1::3]:58443
  → 日本 Xray
  → 美国 IPv4 172.245.182.136:31900
```

移动 Wi-Fi 页面显示它自身已有公网 IPv6，但这不等于它已经把 IPv6 前缀、RA（Router Advertisement）和默认路由稳定传给 OpenWrt/电脑。

**不要把日本 VPS 的 `2001:470:24:7c2:1::3` 填到移动 Wi-Fi 或 OpenWrt 的 IPv6 静态地址中。**这是远端服务器地址，不是家庭网络前缀。

优先级：

1. **首选：有线二层桥接/AP模式**，让移动 Wi-Fi 直接给电脑发布 IPv6；
2. 如果必须让 OpenWrt 做 IPv6 路由，尝试 **WAN6 DHCPv6 + DHCPv6-PD**；
3. 如果移动 Wi-Fi 不提供前缀委托，使用 **IPv6 Relay**；
4. 不要先手工猜测静态 IPv6 前缀，也不要启用 NAT66 作为首选方案。

OpenWrt 标准 IPv6 路由通常依赖 WAN6 获取上游 IPv6 与 DHCPv6-PD，再由 LAN 的 `ip6assign` 分配子网；如果上游不提供前缀委托，可以使用 relay/桥接兼容方案。

---

## 1. 先确认物理连接和角色

你确认移动 Wi-Fi 与 OpenWrt 是**网线连接**，因此优先按 AP/桥接方式配置。

### 推荐网线连接

```text
移动 Wi-Fi LAN 口  ──网线──  OpenWrt LAN 口
```

不要在首选方案中使用：

```text
移动 Wi-Fi LAN 口 → OpenWrt WAN 口
```

否则 OpenWrt 很可能继续作为二级路由器，IPv6 RA/邻居发现/前缀委托可能无法透明传递。

### 现有地址关系

```text
移动 Wi-Fi：192.168.0.1
OpenWrt：192.168.0.2
```

OpenWrt 的 `192.168.0.2` 必须是移动 Wi-Fi LAN 网段内未被占用的地址。建议在移动 Wi-Fi 的 DHCP 地址池中为 `192.168.0.2` 做地址保留，避免地址冲突。

---

## 2. 方案 A：推荐——OpenWrt 作为有线桥接/AP

该方案由移动 Wi-Fi 统一负责：

- IPv4 DHCP；
- IPv6 RA；
- DHCPv6；
- IPv6 默认网关；
- IPv6 前缀分配。

OpenWrt 只负责交换和无线接入，不再做第二层路由。

### 2.1 登录 OpenWrt

浏览器打开：

```text
http://192.168.0.2/cgi-bin/luci/admin/network/network
```

进入：

```text
网络 → 接口
```

### 2.2 配置 LAN 地址

进入：

```text
网络 → 接口 → LAN → 编辑 → 常规设置
```

设置：

```text
协议：静态地址
IPv4 地址：192.168.0.2
IPv4 子网掩码：255.255.255.0
IPv4 网关：192.168.0.1
DNS：192.168.0.1
```

不要填写日本 VPS IPv6。

如果当前 OpenWrt 的 LAN 已经是 `192.168.0.2/24`，只需确认网关和 DNS 正确，不要重复修改地址。

### 2.3 关闭 OpenWrt DHCP 服务

进入：

```text
网络 → 接口 → LAN → 编辑 → DHCP 服务器 → 基本设置
```

勾选：

```text
忽略此接口（Ignore interface）
```

这会关闭 OpenWrt 的 IPv4 DHCP，避免和移动 Wi-Fi 的 DHCP 冲突。

### 2.4 关闭 OpenWrt 的 IPv6 发布服务

在同一个 LAN 接口页面，进入：

```text
DHCP 服务器 → IPv6 设置
```

设置：

```text
Router Advertisement-Service：disabled
DHCPv6-Service：disabled
NDP-Proxy：disabled
```

原因是首选方案让移动 Wi-Fi 作为唯一的 IPv6 RA/DHCPv6 发布者，避免两个设备同时发布默认路由。

### 2.5 禁用或删除独立 WAN 配置

进入：

```text
网络 → 接口
```

如果 OpenWrt 存在独立 WAN/WAN6，并且网线接的是 LAN 口，建议：

- 不让 WAN 参与 LAN 客户端的 IPv6 发布；
- 不要让 WAN6 与 LAN 同时运行另一套 RA；
- 保留管理所需的 LAN 配置即可。

不要在没有确认前删除接口；先停用不使用的 WAN/WAN6，确认网络稳定后再决定是否删除。

### 2.6 保存并应用

点击：

```text
保存并应用
```

应用期间网页可能短暂断开。等待 30–60 秒，然后：

1. 电脑断开 Wi-Fi；
2. 关闭并重新开启电脑 Wi-Fi；
3. 重新连接 OpenWrt 的无线网络；
4. 等待 10–30 秒。

### 2.7 检查电脑是否获得全球 IPv6

Windows PowerShell 执行：

```powershell
Get-NetIPAddress `
  -InterfaceAlias "WLAN" `
  -AddressFamily IPv6 |
  Format-Table IPAddress,PrefixLength,AddressState,PreferredLifetime,ValidLifetime
```

正常应同时看到：

```text
240e:...       全球 IPv6 地址
fe80::...      链路本地地址
```

仅看到 `fe80::...` 不算成功。

检查默认路由：

```powershell
Get-NetRoute `
  -InterfaceAlias "WLAN" `
  -AddressFamily IPv6 |
  Format-Table DestinationPrefix,NextHop,RouteMetric,State
```

必须存在：

```text
::/0
```

并且 `NextHop` 通常是移动 Wi-Fi 的链路本地网关地址。

### 2.8 测试日本 IPv6 节点

```powershell
ping.exe -6 -n 20 2001:470:24:7c2:1::3
```

```powershell
Test-NetConnection `
  -ComputerName "2001:470:24:7c2:1::3" `
  -Port 58443 `
  -InformationLevel Detailed
```

成功目标：

```text
SourceAddress：240e:...
InterfaceAlias：WLAN
TcpTestSucceeded：True
```

---

## 3. 方案 B：OpenWrt 作为 IPv6 路由器（DHCPv6-PD）

只有在移动 Wi-Fi 支持 IPv6-PD/前缀委托，或者支持 IPv6 Passthrough 时使用。

### 3.1 在移动 Wi-Fi 后台开启 IPv6

进入移动 Wi-Fi 的网络/IPv6设置，寻找以下选项：

```text
IPv6：开启
DHCPv6：开启
Router Advertisement：开启
IPv6 Passthrough：开启（如果有）
Prefix Delegation / DHCPv6-PD：开启（如果有）
```

移动 Wi-Fi 页面显示的单个 `240e:...` 地址不代表一定支持 DHCPv6-PD。必须观察是否能向下游委托一个前缀，例如 `/60` 或 `/64`。

### 3.2 配置 OpenWrt WAN6

进入：

```text
网络 → 接口 → 添加新接口
```

填写：

```text
接口名称：wan6
协议：DHCPv6 客户端
设备：与 WAN 相同的物理设备，通常选择 @wan 或 WAN 对应设备
防火墙区域：wan
```

高级/高级设置：

```text
请求 IPv6 地址：try
请求 IPv6 前缀长度：auto
委托 IPv6 前缀：开启
```

不要填写：

```text
2001:470:24:7c2:1::3
240e:476:b488:5829:b467:94ff:fe4f:b5f4
```

它们分别是日本 VPS 地址和移动 Wi-Fi 当前设备地址，不是 OpenWrt 可直接使用的静态 LAN 前缀。

### 3.3 配置 OpenWrt LAN IPv6

进入：

```text
网络 → 接口 → LAN → 编辑
```

设置：

```text
IPv6 assignment length：64
IPv6 assignment hint：留空
```

进入 LAN 的 IPv6/DHCP 设置：

```text
Router Advertisement-Service：server
DHCPv6-Service：server
NDP-Proxy：disabled
```

这样 OpenWrt 会从上游委托前缀中切出一个 `/64` 给 LAN。

### 3.4 检查 WAN6

通过 OpenWrt SSH 执行：

```sh
ifstatus wan6
```

应重点检查：

```text
"up": true
"ipv6-address"
"ipv6-prefix"
"route"
```

再执行：

```sh
ip -6 addr
ip -6 route
```

正常应看到：

```text
WAN6 全球 IPv6 地址
LAN/br-lan 的 IPv6 前缀
default via fe80::...
```

如果 WAN6 没有 `ipv6-prefix`，说明移动 Wi-Fi 没有提供 DHCPv6-PD，应退回方案 A 或方案 C。

---

## 4. 方案 C：IPv6 Relay（没有 DHCPv6-PD 时）

如果移动 Wi-Fi 自身有 IPv6，但不提供前缀委托，使用 relay。

### WAN6

进入：

```text
网络 → 接口 → WAN6 → 编辑 → DHCP 服务器 → IPv6 设置
```

设置：

```text
Router Advertisement-Service：relay
DHCPv6-Service：relay
NDP-Proxy：relay
Master：开启
```

### LAN

进入：

```text
网络 → 接口 → LAN → 编辑 → DHCP 服务器 → IPv6 设置
```

设置：

```text
Router Advertisement-Service：relay
DHCPv6-Service：relay
NDP-Proxy：relay
```

Relay 的目标是把移动 Wi-Fi 的 RA、DHCPv6、NDP 信息延伸给下游电脑。不要把 WAN6 和 LAN 随意配置成两个独立的 `server`，避免默认路由冲突。

如果 relay 无法稳定工作，优先回到方案 A，并使用 LAN-LAN 有线桥接。

---

## 5. 防火墙配置

进入：

```text
网络 → 防火墙
```

如果采用方案 A（桥接/AP）：

- 不需要给 OpenWrt 增加 IPv6 入站放行；
- 由上游移动 Wi-Fi 负责 IPv6 防火墙；
- 不要为了测试直接关闭全部防火墙。

如果采用方案 B/C（OpenWrt 路由/Relay）：

确认允许必要的 IPv6 控制报文：

```text
ICMPv6
DHCPv6 UDP 546/547
```

不要阻断 ICMPv6 的 Packet Too Big、Neighbor Discovery 和 Router Advertisement；否则可能出现 IPv6 TCP 握手成功、传输大数据时卡死。IPv6 PMTU 和邻居发现依赖这些控制报文。([rfc-editor.org](https://www.rfc-editor.org/info/rfc8201?utm_source=openai))

---

## 6. 故障排查顺序

### 6.1 移动 Wi-Fi 自身检查

在移动 Wi-Fi 后台确认：

```text
IPv6 地址是否存在
IPv6 状态是否 connected
IPv6 前缀/PD 是否存在
IPv6 Passthrough 是否开启
```

如果移动 Wi-Fi 自身的 IPv6 地址消失，OpenWrt 无法修复，需要先解决 5G 网络/APN/热点问题。

### 6.2 OpenWrt 检查

```sh
ifstatus wan6
ip -6 addr
ip -6 route
logread -e odhcp6c
logread -e odhcpd
```

重点错误：

```text
No DHCPv6-PD received
```

表示上游没有委托前缀。此时使用方案 A 或 C。

### 6.3 Windows 检查

```powershell
Get-NetIPAddress -InterfaceAlias "WLAN" -AddressFamily IPv6
Get-NetRoute -InterfaceAlias "WLAN" -AddressFamily IPv6
```

如果电脑只有：

```text
fe80::...
```

没有：

```text
240e:...
```

说明 LAN 没有收到有效 RA/IPv6 前缀。

如果有 `240e::` 但没有 `::/0`，说明默认路由/RA 有问题。

### 6.4 测试链路

```powershell
ping.exe -6 -n 20 2001:470:24:7c2:1::3
```

```powershell
Test-NetConnection 2001:470:24:7c2:1::3 -Port 58443 -InformationLevel Detailed
```

如果有全球 IPv6、默认路由，但仍有丢包，测试 USB 网络共享对比：

```text
Wi-Fi 热点 → OpenWrt
USB 共享 → OpenWrt/电脑
```

---

## 7. IPv6 临时地址说明

Windows 可能同时出现：

```text
240e:... 稳定地址
240e:... 临时隐私地址
fe80::... 链路本地地址
```

临时地址定期轮换是正常行为。不要把临时 IPv6 地址手动固定到路由器或 V2rayN。

真正需要保证的是：

```text
至少有一个全球 IPv6 地址
存在 IPv6 默认路由 ::/0
```

如果最终只剩 `fe80::`，不是正常轮换，而是上游 RA/前缀/热点 IPv6 会话失效。

---

## 8. V2rayN 配置

当 Windows 已经获得全球 IPv6 并且 `58443` 测试成功后，使用：

```text
vless://ae7fdd78-66ad-4025-ae23-407fba36c196@[2001:470:24:7c2:1::3]:58443?encryption=none&flow=xtls-rprx-vision&security=reality&sni=www.apple.com&fp=chrome&pbk=0KerkhPa54TPcv4HY5Jd0zWRjk6kvqG0uaYNjiOg6Bc&sid=13b67a5e&type=tcp&headerType=none#JP-IPv6-Relay-US-IPv4
```

V2rayN 建议：

```text
代理目标解析策略：AsIs
连接代理解析策略：留空
Mux：关闭
前置代理：关闭
```

如果浏览器使用 TCP/RAW 代理，建议临时关闭浏览器 QUIC/HTTP3，避免 UDP 流量绕过当前 TCP 节点。

---

## 9. 不要进行的操作

不要：

- 将日本 VPS 地址填入移动 Wi-Fi IPv6 静态地址；
- 猜测并手工填写 `240e:476:b488:5829::/64`；
- 同时让移动 Wi-Fi、OpenWrt 和电脑所在 LAN 发布互相冲突的 RA；
- 在没有确认前缀长度时强行写死 DHCPv6-PD；
- 为了解决 IPv6 问题直接关闭全部防火墙；
- 首先启用 NAT66；
- 公开移动 Wi-Fi 页面中的 IMEI、IMSI、ICCID、手机号和 MAC。

---

## 10. 最终推荐

由于你是移动 Wi-Fi 与 OpenWrt **有线连接**，推荐优先使用：

```text
移动 Wi-Fi LAN
  ↓ 网线
OpenWrt LAN-LAN 桥接/AP
  ↓
电脑
```

并由移动 Wi-Fi 统一负责 IPv6 RA/DHCPv6。该方案最容易保留上游 IPv6 的默认路由和前缀，也最少引入第二层 IPv6 路由问题。

如果你必须让 OpenWrt 做路由，再按顺序尝试：

```text
DHCPv6-PD → IPv6 Relay → 仅在明确理解影响后考虑 NAT66
```

成功标准：

```text
Windows WLAN 有 240e:: 全球 IPv6
Windows WLAN 有 ::/0 默认 IPv6 路由
日本 IPv6 58443 测试 TcpTestSucceeded=True
V2rayN 通过代理出口为 172.245.182.136
```

> 安全提醒：你此前截图暴露了移动 Wi-Fi 的 IMEI、IMSI、ICCID、手机号和 MAC；这些信息应视为敏感信息，建议在分享前打码。路由器 root 密码也不要写入 Markdown 文档，配置完成后建议立即轮换。
