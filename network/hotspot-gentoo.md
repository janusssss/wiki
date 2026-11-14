# Gentoo (OpenRC) 有线口热点配置

网口分工：`wlan0` 上网（DHCP 客户端），`enp12s0` 开热点（静态 IP + DHCP 服务端）。

## 1. 静态地址（netifrc）

`/etc/conf.d/net`：

```bash
config_enp12s0="192.168.10.1/24"
```

init 脚本要软链到 `net.lo`（**不是** `/etc/conf.d/net`，后者是配置文件）：

```bash
cd /etc/init.d
sudo ln -s net.lo net.enp12s0
sudo rc-update add net.enp12s0 default
sudo rc-service net.enp12s0 start
```

坑：

- 软链错目标会报 `service is not executable`。
- dhcpcd 作为 manager 会接管所有接口，抢掉静态配置。若对面网线插着路由器，它还会从那拿到 DHCP 地址。让 dhcpcd 别碰热点口：

```bash
# /etc/dhcpcd.conf
denyinterfaces enp12s0
```

## 2. DHCP 服务端（dnsmasq）

dhcpcd 只是客户端，做热点的 DHCP 服务端用 dnsmasq（包名 `net-dns/dnsmasq`，不是 net-misc）。

`/etc/dnsmasq.conf`：

```bash
interface=enp12s0
bind-interfaces
dhcp-range=192.168.10.50,192.168.10.150,12h
dhcp-option=option:router,192.168.10.1
dhcp-option=option:dns-server,223.5.5.5,114.114.114.114
```

网关必须写热点口的实际地址。启用：

```bash
sudo rc-update add dnsmasq default
sudo rc-service dnsmasq start
```

## 3. NAT 与转发（iptables）

```bash
echo 'net.ipv4.ip_forward=1' | sudo tee /etc/sysctl.d/forward.conf
sudo sysctl -p /etc/sysctl.d/forward.conf

# MASQUERADE 挂出口网卡（wlan0），不是热点口
sudo iptables -t nat -A POSTROUTING -o wlan0 -j MASQUERADE
# FORWARD 策略默认 DROP，必须显式放行两个方向
sudo iptables -A FORWARD -i enp12s0 -o wlan0 -j ACCEPT
sudo iptables -A FORWARD -i wlan0 -o enp12s0 -m state --state ESTABLISHED,RELATED -j ACCEPT
```

## 4. 持久化（local.d，OpenRC 没有 iptables.service）

```bash
sudo mkdir -p /etc/iptables
sudo iptables-save | sudo tee /etc/iptables/rules.v4

sudo tee /etc/local.d/iptables.start <<'EOF'
#!/bin/sh
iptables-restore < /etc/iptables/rules.v4
EOF
sudo chmod +x /etc/local.d/iptables.start
sudo rc-update add local default
```

坑：Docker 启动时会重建自己的链并覆盖规则。整份 `iptables-restore` 要跑在 Docker 之后，或者只把热点规则写成幂等脚本（先 `-D` 再 `-A`）追加，不要整体 restore。

## 5. 与 mihomo（TUN 代理）共存

mihomo 的 `auto-route` 策略路由（ip rule 9000+）会把热点客户端的流量劫持进 Meta 虚拟网卡。症状：抓包看得到流量进 enp12s0，但 FORWARD 一直丢包（只放行了 enp12s0→wlan0 没放行 enp12s0→Meta），客户端上不了网。

修法：

- 让热点流量走代理：放行 `FORWARD -i enp12s0 -o Meta` 和回程（ESTABLISHED）。
- 让热点流量直连（当前采用）：加一条更高优先级的路由策略，整体绕开 mihomo 查主表：

```bash
sudo ip rule add from 192.168.10.0/24 lookup main priority 8000
```

持久化 `/etc/local.d/hotspot.start`：

```bash
#!/bin/sh
ip rule add from 192.168.10.0/24 lookup main priority 8000 2>/dev/null
```

## 6. 代理不命中的原因记录（redir-host / fake-ip 都只剩 IP）

规则全是 `DOMAIN-KEYWORD`，必须有域名。客户端如果走加密 DNS 就拿不到：

- 安卓"私人 DNS"（DoT 853）绕过 mihomo 的 `dns-hijack: any:53`；
- QUIC / DoH（UDP 443）里的域名 sniffer 抓不到；
- redir-host 下旧连接复用没有新握手。

彻底修法（当时未采用，选择直连）：客户端关私人 DNS + 掐 UDP 443/853 逼回明文 53，或切 fake-ip。

## 排查命令备忘

```bash
ip route get <ip> from 192.168.10.50 iif enp12s0   # 模拟热点包看路由走向
sudo iptables -L FORWARD -v -n                  # 看 DROP 计数
ip rule show                                    # 看 mihomo 劫持规则
rc-service net.enp12s0 status
```
