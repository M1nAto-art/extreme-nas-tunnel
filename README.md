# 🚀 Extreme-NAS-Tunnel: 基于 frp + Hysteria 2 双重隧道的本地 NAS (fnOS/Docker) 高性能弱网穿透与流媒体加速方案

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

专治晚高峰卡顿、高丢包、无公网 IPv4/IPv6 的终极内网穿透方案！

本方案打破常规单纯使用 `frp` 或 `DDNS` 的思维定势，独创性地将 **Hysteria 2 (基于 QUIC 协议)** 的暴力抗丢包传输特性，与 **frp** 的多服务内网穿透能力深度缝合。专门用于加速远程访问本地 **fnOS / 群晖 / 个人本地服务器**，以及流畅无感串流 **Emby / Jellyfin** 4K 蓝光影音。

---

## 🧭 核心痛点与终极解法

### 常规穿透的阴间体验：
1. **TCP 握手死锁**：使用常规 frp 或 WebDAV 远程连接时，一旦经过廉价 VPS 或是遭遇晚高峰，网络丢包率飙升，TCP 协议的拥塞控制会导致穿透通道频繁断联、握手超时。
2. **影音串流卡成PPT**：远程在外想看家里的 Emby 蓝光大片，带宽明明够，却因为网络延迟和丢包，视频疯狂转圈，根本无法流畅播放。

### 本方案的黑魔法：
本方案在本地 NAS 与公网 VPS 之间架设一条 **Hysteria 2 强力隧道**，随后强制 **frp 客户端 (frpc) 顺着 Hys2 的本地回环加密通道（UDP/QUIC）走数据**。
利用 Hysteria 2 极其恐怖的弱网吞吐和拥塞控制算法，**即便公网链路丢包率高达 30%，依然能将穿透带宽跑满，实现远程 Emby 秒开、NAS 后台丝滑响应！**

---

## 🏗️ 架构拓扑全景

```text
[家里 fnOS / 本地 Docker 存储] 
      │ (局域网)
      ▼
[本地 frpc 客户端] ──(绑定本地 127.0.0.1)──> [本地 Hysteria 2 客户端]
                                                       │
                                                       ▼ (QUIC 暴力抗丢包隧道)
                                                【大墙 / 晚高峰弱网环境】
                                                       │
                                                       ▼
[公网 VPS (如 RackNerd)] <───────────────── [公网 Hysteria 2 服务端]
      │ 
      ▼ (绑定公网域名端口 / 前置 Cloudflare)
[手机/外网笔记本 (RaiDrive/Emby Theater)]
```

---

## 📦 部署指南

### 第一阶段：公网 VPS 端配置（服务端）

你需要同时在 VPS 上开放 Hysteria 2 的 UDP 端口和 frps 的控制端口。

#### 1. Hysteria 2 服务端配置 (`/etc/hysteria/config.yaml`)
```yaml
listen: :36666 # Hys2 监听的公网 UDP 端口

# 这里的证书可直接白嫖 Certbot 申请的域名证书，或者使用自签证书
cert: /etc/letsencrypt/live/[yourdomain.com/fullchain.pem](https://yourdomain.com/fullchain.pem)
key: /etc/letsencrypt/live/[yourdomain.com/privkey.pem](https://yourdomain.com/privkey.pem)

auth:
  type: password
  password: "你的Hys2强密码"

masquerade:
  type: proxy
  proxy:
    url: [https://bing.com](https://bing.com) # 完美的伪装站，防止被探测
```

#### 2. frps 服务端配置 (`/etc/frp/frps.toml`)
```toml
bindPort = 7000 # frp 基础控制端口
# 如果需要开启网页端控制台查看状态
dashboardPort = 7500
dashboardUser = "admin"
dashboardPwd = "你的frp控制台密码"
```

---

### 第二阶段：本地 NAS / 内网端配置（客户端）

这是整个生态的核心，**巧妙利用本地回环（127.0.0.1）将两条隧道焊死在一起。**

#### 1. 内网 Hysteria 2 客户端配置 (`client.yaml`)
让 Hys2 客户端在内网本地把公网通道转化为一个本地的 Socks5/HTTP 代理，或者使用 `tcpOut` 转发。
```yaml
server: 公网VPS_IP:36666
auth: "你的Hys2强密码"

tls:
  sni: yourdomain.com

# 在本地回环开放一个 Socks5 代理端口，供 frpc 借道起飞
socks5:
  listen: 127.0.0.1:10808
```

#### 2. 内网 frpc 客户端配置 (`frpc.toml`)
**核心看这里！** 我们通过 `transport.proxyURL` 参数，强制让 frp 走我们刚刚建立好的本地 Hys2 强力抗丢包通道：
```toml
serverAddr = "127.0.0.1" # 欺骗 frpc，让它以为服务器在本地
serverPort = 7000        # 对应 VPS 上的 frps 端口

# 核心灵魂：强制 frpc 借道 Hys2 的 Socks5 隧道向外发射流量
transport.proxyURL = "socks5://127.0.0.1:10808"

# 映射本地 fnOS 网页后台
[[proxies]]
name = "nas-web"
type = "tcp"
localIP = "127.0.0.1"
localPort = 8000 # 对应你 fnOS 或群晖的本地网页端口
remotePort = 8080 # 对应通过 VPS 访问的公网端口

# 映射本地 Emby 多媒体流服务
[[proxies]]
name = "nas-emby"
type = "tcp"
localIP = "127.0.0.1"
localPort = 8096 # 本地 Emby 端口
remotePort = 8960 # 映射到 VPS 的公网端口，外网 Emby 客户端填这个
```

---

## 🚦 启动与全线守望

1. **VPS 端**顺次启动服务：
   ```bash
   systemctl start hysteria-server
   systemctl start frps
   ```
2. **本地 NAS/内网端**顺次启动服务（推荐使用 Docker 或 Systemd 守候）：
   ```bash
   # 确保 Hys2 客户端先建立好本地 10808 端口的代理，frpc 随后跟进
   ./hysteria-client -c client.yaml &
   ./frpc -c frpc.toml &
   ```

---

## 📊 战果复盘

配置大功告成后，你在外网用手机或电脑访问 `http://VPS_IP:8960` 连接 Emby。
数据在跨越公网弱网环境时，会**全量被重构为 Hysteria 2 的高并发 QUIC 数据包**，以极度激进的算法撕裂丢包封锁；落地到 VPS 后，再由 frps 解包精准送达。

远程拖动 4K 进度条，你会发现原本死锁转圈的视频，现在**两秒内直接解码起飞**，穿透链路性能被彻底榨干到极限！

---

## 📄 许可证

本项目基于 **[MIT License](LICENSE)** 协议开源。
方案由网络折藤流、NAS 骨灰级玩家纯手工调优落盘，欢迎提交 Issue 交流更激进的拥塞控制参数。
