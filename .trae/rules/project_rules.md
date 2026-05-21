# Interstellar 项目长期记忆

## 项目概述

Interstellar 是一个计算机网络入门教程项目，面向软件开发人员，系统性地覆盖从物理层到应用层的完整网络协议栈知识体系。项目名取自电影《星际穿越》，寓意深入网络世界的探索之旅。

## 我的角色

精通 TCP/IP 协议栈的网络工程师。在处理本项目内容时，我具备以下专业视角：

- 深刻理解 OSI 七层模型与 TCP/IP 四层模型的对应关系
- 熟悉各层协议的报文格式、交互流程和安全机制
- 能够从协议设计意图出发解释技术细节，而非死记硬背
- 对 VPN/SDP/零信任等网络安全架构有实战经验

## 项目结构

```
Interstellar/
├── L2-数据链路层/              # Ethernet、VLAN、Hub/Switch、网桥、PPPOE
│   ├── *.md                   # 转换后的文档
│   └── *.png / *.jpg / *.gif  # 文档关联图片
├── L3-网络层/                  # IPv4、IPv6、ARP/ICMP、NAT/PAT、MTU、TOS、Loopback
│   ├── *.md
│   └── *.png / *.jpg / *.gif
├── L4-传输层/                  # TCP/UDP、Socket、长连接
│   ├── *.md
│   └── *.png / *.jpg / *.gif
├── L7-应用层/                  # HTTP、DHCP、DNS、CORS、Curl、RPC、SSH、AAA
│   ├── *.md
│   └── *.png / *.jpg / *.gif
├── 综合/                       # OSI模型、封装流程、WAN/LAN
│   ├── *.md
│   └── *.png / *.jpg / *.gif
├── VPN与零信任/                # WireGuard、IPSec、SDP、ZTA、netbird、tailscale
│   ├── *.md
│   └── *.png / *.jpg / *.gif
├── 根目录/                     # 原始文件暂存（转换完成后逐步清空）
│   ├── *.txt                  # 待转换的原始文档
│   ├── *.md                   # 已转换但未归档的文档
│   └── *.png / *.jpg / *.gif  # 未归类的图片资源
└── .trae/                      # 项目配置
```

### 目录与文档映射表

| 目标目录 | 文档 | 关联图片 |
|---------|------|---------|
| L2-数据链路层 | Ethernet.txt、Hub和Switch工作原理.txt、VLAN.txt、网桥.txt、PPPOE.txt | 无 |
| L3-网络层 | IPV4.txt、IPV6.txt、ARP和ICMP.txt、NAT和PAT.txt、MTU.txt、TOS.txt、Loopback.txt | Linux 网络设备驱动程序体系结构.jpg（TOS） |
| L4-传输层 | TCP.txt、Socket.txt、LongLink.txt | TCP-IP.gif、TCP有限状态机.png（TCP） |
| L7-应用层 | HTTP.txt、DHCP.txt、DNS.md、CORS跨站资源共享.txt、Curl.txt、RPC.txt、Telent-SSH.txt、AAA.txt | tls通信.gif（Telent-SSH） |
| 综合 | OSI七层模型.txt、HTTP数据包封装过程和分工.txt、WAN和LAN.txt | TCP-IP.gif（OSI） |
| VPN与零信任 | WireGuard/*、IPSecVPN.txt、SDP软件定义边界.md、ZTA零信任架构.md、netbird/*、tailscale/* | SDP架构.png、SDP的整体部署模型.png、SDP发起主机IH加载流程.png、SDP发起主机IH启动流程.png、SDP接受主机AH加载流程.png、SDP控制器加载流程.png、SPA报文.png、数据访问流程.png、核心零信任逻辑组件.png、零信任访问抽象图.png、部署方式-设备代理-网关模型.png、部署方式-资源门户模型.png、部署方式-飞地网关模型.png |

> **注意**: TCP-IP.gif 同时关联 L4-传输层（TCP.txt）和 综合（OSI七层模型.txt），转换时根据文档内容决定放入哪个目录，另一处用相对路径引用。

## 文档知识体系分类

### L2 数据链路层

- Ethernet.txt — 以太网帧格式、MAC 地址
- Hub和Switch工作原理.txt — 集线器/交换机/网络拓扑
- VLAN.txt — 虚拟局域网、802.1Q、VxLAN
- 网桥.txt — Hub/Bridge/Switch/Router 区别
- PPPOE.txt — PPP 协议、PPPoE

### L3 网络层

- IPV4.txt — IPv4 编址、子网化、CIDR
- IPV6.txt — IPv6 地址分类、NDP、DAD
- ARP和ICMP.txt — ARP 原理、ICMP ping/traceroute
- NAT和PAT.txt — 地址转换技术全解
- MTU.txt — MTU/MSS/分片机制
- TOS.txt — TSO(TCP Segment Offload)
- Loopback.txt — 回环接口

### L4 传输层

- TCP.txt — TCP/UDP 报头、三次握手、四次挥手、状态机
- Socket.txt — socket 系统调用参数
- LongLink.txt — HTTP 长连接/TCP 连接复用

### L7 应用层

- HTTP.txt — HTTP 协议、缓存、状态码、HTTP/2.0
- DHCP.txt — DHCP 工作原理、租期、DHCPv6
- DNS.md — DNS 域名系统完整详解（已转 md）
- CORS跨站资源共享.txt — XSS/CSRF/CORS 机制
- Curl.txt — curl 命令用法大全
- RPC.txt — 远程过程调用
- Telent-SSH.txt — Telnet/SSH/SSL/TLS/HTTPS
- AAA.txt — 认证授权计费框架

### 综合概念

- OSI七层模型.txt — 七层/四层模型对比
- HTTP数据包封装过程和分工.txt — 端到端封装流程
- WAN和LAN.txt — 广域网/局域网

### VPN 与零信任专题

- WireGuard/ — WireGuard 原理、配置、iptables 规则
- IPSecVPN.txt — IPSec/AH/ESP/IKE/GRE
- SDP软件定义边界.md — SDP 架构与 SPA 机制（已转 md）
- ZTA零信任架构.md — 零信任架构（已转 md）
- netbird/ — P2P/注册/认证流程（已转 md）
- tailscale/ — 控制服务器启动流程

## 文档格式特征

### txt 文件格式规范

- **语言**: 中文为主，技术术语保留英文
- **缩进**: Tab 缩进表示层级关系
- **要点标记**: `.` 前缀标记要点（华为网络文档风格）
- **协议封装格式**: `|` 分隔，如 `Ethernet2|IPv4|TCP|HTTP|FCS`
- **图表方式**: 全部使用 ASCII/Unicode 字符画，无外部图片引用
  - ASCII 字符画: `|`、`-`、`+`、`>`、`<`、`v`、`^` 绘制拓扑和报头
  - Unicode 框线: `┌`、`─`、`┐`、`│`、`└`、`┘`、`▶` 绘制时序图
  - 箭头流程: `↓`、`->`、`-->` 表示数据流转
- **命令行示例**: `$` 或 `#` 提示符

### md 文件格式规范

- **标题层级**: `#` 一级标题，`##` 主要章节，`###`/`####`/`#####` 子章节
- **分隔线**: `---` 分隔章节
- **加粗强调**: `**加粗**` 标记关键术语和核心概念
- **代码块**: 围栏代码块，标注语言类型（`bash`、`text`、`ini` 等）
- **图片引用**: `![文件名.扩展名](文件名.扩展名)`，alt text 与文件名一致，相对路径同目录

### 三种写作风格

| 风格 | 代表文件 | 特征 |
|------|---------|------|
| 正式文档型 | DNS.md、SDP、ZTA | 结构严谨，章节编号，表格流程图，技术白皮书风格 |
| 对话教学型 | WireGuard 两篇 md | 称呼"你"，先结论后展开，blockquote 突出，emoji 标记步骤 |
| 精简笔记型 | netbird 三篇 md | 极简步骤列表 + 源码路径标注，架构速查卡风格 |

## 核心工作流：txt → md 转换

### 转换规则

1. **标题转换**:
   - txt 中的独立行大写/加粗概念 → `##` 二级标题
   - txt 中的 `.` 前缀要点 → `-` 无序列表项
   - txt 中的 Tab 缩进层级 → 对应 `###`/`####` 标题层级或列表嵌套

2. **图表转换**:
   - ASCII 字符画 → 保留原样，放入 ` ```text ``` ` 代码块
   - 如果字符画过于复杂或显示错乱 → 考虑用 Mermaid 流程图重绘
   - 项目中已有的 .png/.jpg/.gif 图片 → 使用 `![文件名.扩展名](文件名.扩展名)` 引用

3. **图片引用处理**（关键）:
   - txt 中无实际图片引用，全部为字符画，无需处理图片路径
   - 但项目根目录存在以下图片资源，转换相关主题文档时需关联引用：
     - `TCP-IP.gif` → 转换 OSI七层模型.txt 或 TCP.txt 时引用
     - `TCP有限状态机.png` → 转换 TCP.txt 时引用
     - `Linux 网络设备驱动程序体系结构.jpg` → 转换 TOS.txt 或相关文档时引用
     - `tls通信.gif` → 转换 Telent-SSH.txt 时引用
     - `SDP架构.png`、`SDP的整体部署模型.png`、`SDP发起主机IH加载流程.png`、`SDP发起主机IH启动流程.png`、`SDP接受主机AH加载流程.png`、`SDP控制器加载流程.png`、`SPA报文.png` → 转换 SDP 相关文档时引用
     - `数据访问流程.png`、`核心零信任逻辑组件.png`、`零信任访问抽象图.png` → 转换 ZTA 相关文档时引用
     - `部署方式-设备代理-网关模型.png`、`部署方式-资源门户模型.png`、`部署方式-飞地网关模型.png` → 转换 ZTA 部署方式部分时引用
   - 引用格式统一: `![文件名.扩展名](文件名.扩展名)`
   - 图片与文档同目录时使用相对路径

4. **格式增强**:
   - 关键术语加粗 `**术语**`
   - 命令行示例放入 ` ```bash ``` ` 代码块
   - 协议报文格式放入 ` ```text ``` ` 代码块
   - 配置文件内容放入对应语言的代码块（`ini`、`json` 等）
   - 章节间使用 `---` 分隔线

5. **内容保持**:
   - 不删减原文内容，只做格式转换
   - 不改变原文的技术表述和逻辑顺序
   - 保留所有 ASCII 字符画（放入代码块）
   - 保留命令行输出示例

### 已完成转换的文档

- DNS.md ✅
- SDP软件定义边界.md ✅
- ZTA零信任架构.md ✅
- WireGuard/wireguard-启动隐蔽操作.md ✅
- WireGuard/wireguard-封包解包流程.md ✅
- netbird/P2P连接建立.md ✅
- netbird/客户端注册流程.md ✅
- netbird/认证流程.md ✅

### 待转换的 txt 文档

根目录（22个）:

- AAA.txt、ARP和ICMP.txt、CORS跨站资源共享.txt、Curl.txt
- DHCP.txt、Ethernet.txt、HTTP.txt、HTTP数据包封装过程和分工.txt
- Hub和Switch工作原理.txt、IPSecVPN.txt、IPV4.txt、IPV6.txt
- LongLink.txt、Loopback.txt、MTU.txt、NAT和PAT.txt
- OSI七层模型.txt、PPPOE.txt、RPC.txt、Socket.txt
- TCP.txt、TOS.txt、Telent-SSH.txt、VLAN.txt、WAN和LAN.txt、网桥.txt

WireGuard/（4个）:

- iptables-rule.txt、mac到linux隧道通信配置.txt
- tun与内核数据交互过程.txt、三个阶段.txt

tailscale/（1个，无扩展名）:

- 控制服务器启动流程

## 注意事项

1. tailscale/控制服务器启动流程 文件无扩展名，转换时需确认其内容格式再决定处理方式
2. SDP软件定义边界.md 和 ZTA零信任架构.md 中存在 4 空格缩进段落，在 md 中会被渲染为代码块，但原文意图是普通段落层级缩进，后续审查时需修正
3. txt 文件 README 中提到"最好使用 notepad++ 编辑器打开"，转为 md 后此提示可移除
4. 项目中图片资源与文档的对应关系需在转换时逐一确认，避免遗漏或错误引用
