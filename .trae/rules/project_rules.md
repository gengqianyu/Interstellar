# Interstellar 项目长期记忆

## 项目概述

Interstellar 是一个计算机网络入门教程项目，面向软件开发人员，系统性地覆盖从物理层到应用层的完整网络协议栈知识体系。项目名取自电影《星际穿越》，寓意深入网络世界的探索之旅。

**所有 txt 文档已全部转换为 md 格式，并按协议层归入对应目录。**

## 我的角色

精通 TCP/IP 协议栈的网络工程师。在处理本项目内容时，我具备以下专业视角：

- 深刻理解 OSI 七层模型与 TCP/IP 四层模型的对应关系
- 熟悉各层协议的报文格式、交互流程和安全机制
- 能够从协议设计意图出发解释技术细节，而非死记硬背
- 对 VPN/SDP/零信任等网络安全架构有实战经验

## 项目结构

```
Interstellar/
├── L2-数据链路层/              Ethernet、VLAN、Hub/Switch、网桥、PPPOE
├── L3-网络层/                  IPv4、IPv6、ARP/ICMP、NAT/PAT、MTU、TOS、Loopback
├── L4-传输层/                  TCP/UDP、Socket、长连接
├── L7-应用层/                  HTTP、DHCP、DNS、CORS、Curl、RPC、SSH、AAA
├── 综合/                       OSI模型、封装流程、WAN/LAN
├── VPN与零信任/                WireGuard、IPSec、SDP、ZTA、netbird、tailscale
└── .trae/                      项目配置
```

### 各目录文件清单

**L2-数据链路层** (5 个 md)
- Ethernet.md — IEEE 802.3 标准、帧格式、MAC 地址、单播/广播/组播
- Hub和Switch工作原理.md — 三层架构、拓扑分类、Hub 共享带宽
- VLAN.md — VLAN、Access/Trunk、802.1Q、VxLAN
- PPPOE.md — PPP 三阶段、PPPoE、广域网协议
- 网桥.md — Hub/Bridge/Switch/Router 区别

**L3-网络层** (7 个 md + 1 张图片)
- IPV4.md — IP 编址、子网化、CIDR
- IPV6.md — IPv6 地址分类、NDP、DAD、DHCPv6
- ARP和ICMP.md — ARP 原理、网络内/间通信、ARP 代理、免费 ARP
- NAT和PAT.md — 静态/动态 NAT、PAT、Easy IP、DMZ
- MTU.md — MTU/MSS/分片、jumbo frame
- TOS.md — TSO 原理、MSS 增大 → 关联图片: Linux 网络设备驱动程序体系结构.jpg
- Loopback.md — 回环接口

**L4-传输层** (3 个 md + 2 张图片)
- TCP.md — TCP/UDP 报头、三次握手、四次挥手、SYN Flooding → 关联图片: TCP-IP.gif、TCP有限状态机.png
- Socket.md — socket() 系统调用
- LongLink.md — HTTP 长连接、TCP Fast Open

**L7-应用层** (8 个 md + 1 张图片)
- HTTP.md — HTTP 协议、缓存、状态码、HTTP/2.0
- DNS.md — DNS 报文格式、查询过程、记录类型
- DHCP.md — DORA 过程、租期、DHCPv6
- Telent-SSH.md — SSH、SSL/TLS、HTTPS → 关联图片: tls通信.gif
- CORS跨站资源共享.md — XSS/CSRF/CORS 机制
- Curl.md — curl 命令用法
- RPC.md — 远程过程调用
- AAA.md — 认证/授权/计费、RADIUS/HWTACACS

**综合** (3 个 md + 1 张图片)
- OSI七层模型.md — 七层/四层模型对比 → 关联图片: TCP-IP.gif
- HTTP数据包封装过程和分工.md — 端到端封装流程
- WAN和LAN.md — 广域网/局域网

**VPN与零信任** (8 个 md + 13 张图片 + netbird 子目录)
- WireGuard三个阶段.md — 握手/数据传输/流转图
- wireguard-启动隐蔽操作.md — AllowedIPs 三重职责
- wireguard-封包解包流程.md — 封包/解包机制
- iptables-rule.md — NAT 节点 iptables 规则
- mac到linux隧道通信配置.md — Mac/Linux WireGuard 配置
- tun与内核数据交互过程.md — tun 虚拟网卡数据交互
- IPSecVPN.md — AH/ESP/IKE、SA、传输/隧道模式、GRE
- SDP软件定义边界.md — SDP 架构、SPA → 关联图片: SDP架构.png 等 7 张
- ZTA零信任架构.md — 零信任模型、PDP/PEP → 关联图片: 零信任访问抽象图.png 等 6 张
- 控制服务器启动流程.md — Tailscale 源码路径
- netbird/ — P2P连接建立.md、客户端注册流程.md、认证流程.md

## 转换状态

**全部完成** ✅ — 所有 30 个原始文档（22 个 txt + 8 个已有 md）均已转换为 md 格式并归入协议层目录。

## 文档格式规范

### md 文件格式规范
- **标题层级**: `#` 一级标题，`##` 主要章节，`###`/`####`/`#####` 子章节
- **分隔线**: `---` 分隔章节
- **加粗强调**: `**加粗**` 标记关键术语和核心概念
- **代码块**: 围栏代码块，标注语言类型（`bash`、`text`、`ini` 等）
- **图片引用**: `![文件名.扩展名](文件名.扩展名)`，alt text 与文件名一致，相对路径同目录
- **断句空格**: 保留原文中用于阅读理解的空格（如 "3 层网络设备"、"源目 MAC 地址"）
- **文本图示**: ASCII/Unicode 字符画放入 ` ```text ``` ` 代码块，Tab 替换为空格对齐

### 三种写作风格

| 风格 | 代表文件 | 特征 |
|------|---------|------|
| 正式文档型 | DNS.md、SDP、ZTA | 结构严谨，章节编号，表格流程图，技术白皮书风格 |
| 对话教学型 | WireGuard 两篇 md | 称呼"你"，先结论后展开，blockquote 突出，emoji 标记步骤 |
| 精简笔记型 | netbird 三篇 md | 极简步骤列表 + 源码路径标注，架构速查卡风格 |

## 后续维护规则

1. 新增文档时，按协议层放入对应目录，命名与目录中现有文件风格一致
2. 图片资源与文档同目录存放，使用相对路径引用
3. 修正错别字时保留原文的断句空格风格
4. 文本图示放入 `text` 代码块，不要使用 Tab 缩进
5. SDP软件定义边界.md 和 ZTA零信任架构.md 中存在 4 空格缩进段落，后续审查时需修正为普通段落
