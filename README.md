# Interstellar

> *"We've always defined ourselves by the ability to overcome the impossible."*
> — Cooper, Interstellar

深入网络世界的探索之旅 —— 面向软件开发人员的计算机网络协议栈教程。

---

## 项目简介

Interstellar 系统性地覆盖从物理层到应用层的完整网络协议栈知识体系，同时深入 VPN/SDP/零信任等现代网络安全架构。所有文档均按协议层组织，便于按图索骥、逐层深入。

## 知识图谱

```
                        ┌─────────────────────────────────────────┐
                        │              L7 应用层                   │
                        │  HTTP  DHCP  DNS  CORS  SSH  AAA  RPC  │
                        ├─────────────────────────────────────────┤
                        │              L4 传输层                   │
                        │         TCP  UDP  Socket  长连接         │
                        ├─────────────────────────────────────────┤
                        │              L3 网络层                   │
                        │  IPv4  IPv6  ARP  ICMP  NAT  MTU  TOS  │
                        ├─────────────────────────────────────────┤
                        │            L2 数据链路层                 │
                        │    Ethernet  VLAN  Hub/Switch  PPPoE    │
                        └─────────────────────────────────────────┘
                                              │
                                              ▼
                        ┌─────────────────────────────────────────┐
                        │           VPN 与零信任架构                │
                        │  WireGuard  IPSec  SDP  ZTA  NetBird    │
                        └─────────────────────────────────────────┘
```

## 目录结构

```
Interstellar/
├── L2-数据链路层/          以太网帧格式、MAC地址、VLAN、交换机原理、PPPoE
│   ├── Ethernet.md
│   ├── Hub和Switch工作原理.md
│   ├── VLAN.md
│   ├── PPPOE.md
│   └── 网桥.md
│
├── L3-网络层/              IP编址、子网化、地址解析、地址转换、MTU
│   ├── IPV4.md
│   ├── IPV6.md
│   ├── ARP和ICMP.md
│   ├── NAT和PAT.md
│   ├── MTU.md
│   ├── TOS.md
│   ├── Loopback.md
│   └── Linux 网络设备驱动程序体系结构.jpg
│
├── L4-传输层/              TCP/UDP协议、三次握手、四次挥手、Socket
│   ├── TCP.md
│   ├── Socket.md
│   ├── LongLink.md
│   ├── TCP-IP.gif
│   └── TCP有限状态机.png
│
├── L7-应用层/              HTTP、DNS、DHCP、SSH、CORS、AAA
│   ├── HTTP.md
│   ├── DNS.md
│   ├── DHCP.md
│   ├── Telent-SSH.md
│   ├── CORS跨站资源共享.md
│   ├── Curl.md
│   ├── RPC.md
│   ├── AAA.md
│   └── tls通信.gif
│
├── 综合/                   OSI模型、端到端封装流程
│   ├── OSI七层模型.md
│   ├── HTTP数据包封装过程和分工.md
│   ├── WAN和LAN.md
│   └── TCP-IP.gif
│
└── VPN与零信任/            WireGuard、IPSec、SDP、ZTA、NetBird、Tailscale
    ├── WireGuard三个阶段.md
    ├── wireguard-启动隐蔽操作.md
    ├── wireguard-封包解包流程.md
    ├── iptables-rule.md
    ├── mac到linux隧道通信配置.md
    ├── tun与内核数据交互过程.md
    ├── IPSecVPN.md
    ├── SDP软件定义边界.md
    ├── ZTA零信任架构.md
    ├── 控制服务器启动流程.md
    ├── netbird/
    │   ├── P2P连接建立.md
    │   ├── 客户端注册流程.md
    │   └── 认证流程.md
    └── (13 张架构图)
```

## 文档清单

### L2 数据链路层

| 文档 | 核心内容 |
|------|---------|
| [Ethernet.md](L2-数据链路层/Ethernet.md) | IEEE 802.3 标准、Ethernet_II 与 802.3 帧格式对比、MAC 地址结构、单播/广播/组播帧识别 |
| [Hub和Switch工作原理.md](L2-数据链路层/Hub和Switch工作原理.md) | 三层架构（接入/汇聚/核心）、拓扑分类、Hub 共享带宽、bit 与 Byte |
| [VLAN.md](L2-数据链路层/VLAN.md) | VLAN 技术、Access/Trunk 接口、802.1Q 标签、SVI 虚拟接口、VxLAN |
| [PPPOE.md](L2-数据链路层/PPPOE.md) | PPP 协议三阶段（LCP/PAP/CHAP/NCP）、PPPoE 产生背景、广域网协议演进 |
| [网桥.md](L2-数据链路层/网桥.md) | Hub → Bridge → Switch → Router 的演进与区别 |

### L3 网络层

| 文档 | 核心内容 |
|------|---------|
| [IPV4.md](L3-网络层/IPV4.md) | IP 报头格式、编址、地址分类、公有/私有地址、FLSM/VLSM 子网化、CIDR、路由汇总 |
| [IPV6.md](L3-网络层/IPV6.md) | IPv6 地址格式与分类（GUA/ULA/LLA）、组播/任播、EUI-64、NDP、DAD、DHCPv6 |
| [ARP和ICMP.md](L3-网络层/ARP和ICMP.md) | ARP 地址解析、网络内/间通信的 ARP 过程、ARP 代理、免费 ARP、ICMP ping/traceroute |
| [NAT和PAT.md](L3-网络层/NAT和PAT.md) | IPv4 危机、静态/动态 NAT、PAT/NAPT、Easy IP、5 元组、DMZ |
| [MTU.md](L3-网络层/MTU.md) | MTU/MSS/分片机制、PPPoE 对 MTU 的影响、TCP/UDP 分片策略、jumbo frame |
| [TOS.md](L3-网络层/TOS.md) | TSO(TCP Segment Offload) 原理、MSS 增大机制、内核 TCP/IP 协议栈 |
| [Loopback.md](L3-网络层/Loopback.md) | 回环接口 127.0.0.1、nginx 虚拟域名绑定、监听机制 |

### L4 传输层

| 文档 | 核心内容 |
|------|---------|
| [TCP.md](L4-传输层/TCP.md) | TCP/UDP 报头格式、三次握手、四次挥手、TCP 状态机、SYN Flooding 攻击、UDP 安全性（SPA） |
| [Socket.md](L4-传输层/Socket.md) | socket() 系统调用、协议族（PF_INET）、SOCK_STREAM/SOCK_DGRAM/SOCK_RAW |
| [LongLink.md](L4-传输层/LongLink.md) | HTTP 长连接、TCP 连接复用、TCP Fast Open |

### L7 应用层

| 文档 | 核心内容 |
|------|---------|
| [HTTP.md](L7-应用层/HTTP.md) | 请求方法、请求头/响应头、Content-Type、缓存机制、状态码、Cookie/Session、HTTP/2.0 |
| [DNS.md](L7-应用层/DNS.md) | DNS 报文格式、查询过程、域名层级、DNS 服务器类型、记录类型 |
| [DHCP.md](L7-应用层/DHCP.md) | DHCP DORA 过程、租期定时器、免费 ARP 冲突检测、DHCPv6 |
| [Telent-SSH.md](L7-应用层/Telent-SSH.md) | Telnet/SSH、对称/非对称加密、CA 认证、SSL/TLS 握手、HTTPS |
| [CORS跨站资源共享.md](L7-应用层/CORS跨站资源共享.md) | XSS/CSRF 攻击、CORS 简单/非简单请求、预检请求、withCredentials |
| [Curl.md](L7-应用层/Curl.md) | curl 命令用法：下载、上传、伪造 referer/UA、Cookie、POST/GET、调试 |
| [RPC.md](L7-应用层/RPC.md) | 远程过程调用概念、基本通信模型 |
| [AAA.md](L7-应用层/AAA.md) | 认证/授权/计费框架、RADIUS/HWTACACS 协议、AAA 域 |

### 综合

| 文档 | 核心内容 |
|------|---------|
| [OSI七层模型.md](综合/OSI七层模型.md) | OSI 七层与 TCP/IP 四层模型对比、上三层/下四层分工、封装格式 |
| [HTTP数据包封装过程和分工.md](综合/HTTP数据包封装过程和分工.md) | 从 HTTP 到物理网络的端到端封装流程，每层由哪个软件/硬件组件负责 |
| [WAN和LAN.md](综合/WAN和LAN.md) | 广域网/局域网定义、家用路由器端口区别 |

### VPN 与零信任

| 文档 | 核心内容 |
|------|---------|
| [WireGuard三个阶段.md](VPN与零信任/WireGuard三个阶段.md) | 握手阶段（3次交互）、数据传输阶段、数据流转总图 |
| [wireguard-启动隐蔽操作.md](VPN与零信任/wireguard-启动隐蔽操作.md) | AllowedIPs 三重职责、路由注入、双向约束、SDP 下的战略意义 |
| [wireguard-封包解包流程.md](VPN与零信任/wireguard-封包解包流程.md) | WireGuard L3 VPN 封包/解包机制、内层 IP 包源地址替换 |
| [IPSecVPN.md](VPN与零信任/IPSecVPN.md) | IPSec 架构（AH/ESP/IKE）、SA、传输模式/隧道模式、GRE+IPSec |
| [SDP软件定义边界.md](VPN与零信任/SDP软件定义边界.md) | SDP 架构组件、SPA 单包授权、mTLS 双向认证 |
| [ZTA零信任架构.md](VPN与零信任/ZTA零信任架构.md) | 零信任抽象模型、PDP/PEP、七大原则、三种部署模式 |
| [netbird/](VPN与零信任/netbird/) | P2P 连接建立、客户端注册流程、OIDC 认证流程 |

## License

MIT License. 转载请注明出处。
