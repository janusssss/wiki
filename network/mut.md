# 知识点：TLS handshake timeout 与路径 MTU 问题

## 一、现象

Go 服务连接内网 HTTPS 接口时报错：

```
net/http: TLS handshake timeout
Post "https://10.37.24.228:10002/uas/v1/api/machine/login"
```

关键特征：

- TCP 连接能建立成功
- 错误发生在 TLS 握手阶段，表现为超时，而非明确的连接拒绝或证书错误
- `openssl s_client` 手动测试有时能成功
- 调整 `insecure_skip_verify`、排查代理、调整 `GODEBUG` 均无效

## 二、排查过程与关键发现

### 1. openssl 测试：握手链路本身是通的

`openssl s_client -connect 10.37.24.228:10002` 能完成完整 TLS 握手（TLSv1.2 + ECDHE-RSA-AES128-GCM-SHA256），说明：

- TCP 能通
- 服务端 TLS 服务正常
- 无代理拦截、无防火墙丢弃握手包

同时暴露证书问题（自建 CA `acs-ca`、只下发叶子证书），但这**不是超时的原因**。

### 2. 排除 Go 1.23 的 ClientHello 膨胀

Go 1.23 默认启用后量子密钥交换 `X25519Kyber768Draft00`，使 ClientHello 从约 250 字节膨胀到约 1471 字节，可能被中间设备丢弃。

设置 `GODEBUG=tlskyber=0` 后，ClientHello 降回 249 字节，**但问题依旧**，排除此因素。

### 3. 抓包定位：SACK 暴露大包丢失

tcpdump 关键输出（相对序列号）：

```
客户端 → 服务端  [P.] seq 1:250, length 249        # ClientHello，正常送达
服务端 → 客户端  [.]  ack 250                        # 确认收到
服务端 → 客户端  [P.] seq 1389:2347, length 958      # TLS 响应后半段
客户端 → 服务端  [.]  ack 1, sack {1389:2347}        # 收到 958 段，但连续确认卡在 1
服务端 → 客户端  [P.] seq 1389:2347, length 958      # 重传同一段（无效）
客户端 → 服务端  [.]  ack 1, sack {1389:2347}        # 仍 SACK 同一段
... 10 秒无进展 ...
客户端 → 服务端  [F.] 放弃
```

解读：

- 客户端 **SACK 了 `1389:2347`**，证明这 958 字节段**收到了**
- 但连续 `ack` 停在 1，说明**更早的 `seq 1:1389`（约 1389 字节）没收到**
- 服务端没有以合适尺寸重传缺失段，10 秒后 Go `TLSHandshakeTimeout`（默认 10 秒）到期

### 4. 确认根因：容器 MTU 与路径 MTU 不匹配

```
ip route get 10.37.24.228
→ via 192.168.122.1 dev enp1s0 src 192.168.122.50

ip link show enp1s0
→ mtu 1300
```

- 宿主机网卡 `enp1s0` MTU = **1300**
- Docker 容器 `eth0` 默认 MTU = **1500**（不继承宿主机网卡 MTU）
- 容器 SYN 通告 `mss 1460`，证明它按 MTU 1500 发包

## 三、链路断开的精确过程

```
[Go 进程]
   ↓
[容器 eth0: MTU 1500]        ← 按 1500 算，TCP 载荷最大 1460
   ↓
[docker0 / veth: MTU 1500]
   ↓
[宿主机 enp1s0: MTU 1300]    ← 超过 1300 的 IP 包装不下，被静默丢弃
   ↓
[10.37.24.228:10002]
```

| 阶段 | 事件 | 结果 |
|---|---|---|
| 三次握手 | SYN/SYN-ACK 都是小包 | 成功 |
| ClientHello | 249 字节，小于 1300 | 成功送达 |
| 服务端响应 `seq 1:1389` | 约 1389+40 ≈ 1429 字节，超过 1300 | **被 enp1s0 静默丢弃** |
| 服务端响应 `seq 1389:2347` | 约 958+40 ≈ 998 字节，小于 1300 | 通过，客户端 SACK |
| PMTUD | 本应回 ICMP "Fragmentation Needed" 通知容器调小 MSS | **未回到容器，失效** |
| 重传 | 服务端重传仍是大包或无效段 | 再次被丢，僵局 |
| 超时 | Go 默认 `TLSHandshakeTimeout` 10 秒 | 报 `TLS handshake timeout` |

**断点一句话概括**：服务端 TLS 响应中约 1429 字节的大段在经过 MTU 1300 的宿主机网卡时被静默丢弃；客户端只收到较小的 958 字节段（SACK 可证），缺失的前半段因 PMTUD 失效始终没有以合适尺寸重传，TLS 消息永远拼不完整，10 秒后 Go 握手超时。

## 四、为什么改宿主机 MTU 没用

把 `enp1s0` 的 MTU 改成 1300 无效，因为**发包源头是容器**：

- 容器内 `eth0` 仍是 1500，按 1500 决定 TCP 段大小
- 包离开容器时就已经是约 1460 字节载荷
- 宿主机网卡只是负责丢弃超尺寸包，不能让容器少发

必须从**容器接入的网络层**限制 MTU。

## 五、解决方案

### 方案一：创建自定义 Docker 网络，指定 MTU（推荐）

```bash
docker rm -f jqsj-lowpower-proxy

docker network create \
  --driver bridge \
  --opt com.docker.network.driver.mtu=1280 \
  jqsj-mtu-net

docker run --name jqsj-lowpower-proxy \
  --network jqsj-mtu-net \
  -p 12358:8080 \
  -e GODEBUG=tlskyber=0 \
  --volume /etc/jqsj-lowpower-proxy/local.yaml:/app/config/local.yaml \
  jqsj-lowpower-proxy:arm-old
```

验证：

```bash
docker exec jqsj-lowpower-proxy ip link show eth0
# 应显示 mtu 1280
```

容器 MTU 设为 1280 后，TCP 载荷最大约 1240，加头约 1280，不超过路径 1300，所有包都能通过，握手完成。

### 方案二：修改 Docker 全局默认 MTU

编辑 `/etc/docker/daemon.json`：

```json
{
  "mtu": 1280
}
```

```bash
sudo systemctl restart docker
```

影响所有新启动的容器，已有容器需重建。

## 六、PMTUD 为什么会失效

PMTUD（Path MTU Discovery）依赖中间设备在丢弃超尺寸包时回送 ICMP "Fragmentation Needed" 消息，通知发送方调小 MSS。失效常见原因：

- 中间防火墙丢弃 ICMP Type 3 Code 4
- NAT 转换后 ICMP 无法正确关联回原连接
- 容器网络栈收不到该 ICMP
- 安全策略禁止 ICMP

PMTUD 失效时，发送方不知道包太大，持续发大包、持续被丢，形成死循环。

## 七、同类问题的判断线索

出现以下组合时，优先怀疑路径 MTU 问题：

- TCP 能建连，但 TLS 握手、HTTP 大响应、文件传输卡住或超时
- 抓包看到客户端反复 SACK 某一段、服务端重传无效、大包缺失小包正常
- `openssl` 有时通有时不通，或 `curl` 小请求正常、大响应卡死
- 容器/虚拟机/隧道网络中，各层 MTU 不一致

验证方法：

```bash
# 查看路由和出口网卡
ip route get <目标IP>
ip link show <网卡名>

# 逐级试探路径 MTU（容器内执行）
ping -M do -s 1272 <目标IP>
```

## 八、核心结论

1. **`TLS handshake timeout` 不等于证书问题或代理问题**，要区分握手卡在哪一步。
2. **SACK 是定位丢包的关键证据**：SACK 某段说明该段收到了，连续 ack 推不上去说明更早的段缺失。
3. **容器 MTU 不继承宿主机网卡 MTU**，Docker 默认 1500，多层网络叠加时容易超出实际路径 MTU。
4. **PMTUD 失效会让 MTU 问题表现为静默丢包 + 超时**，而非明确错误。
5. **改 MTU 要改发包源头**（容器网络），改宿主机网卡只负责丢包，解决不了问题。
6. 根治还需确认 PMTUD 为何失效（中间设备是否放行 ICMP Type 3 Code 4），否则同路径其他服务可能再踩坑。
