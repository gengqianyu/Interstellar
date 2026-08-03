Answering your question about the Linux kernel network stack requires a structured approach. I will break down the entire journey of a packet—from the moment it hits the physical network interface card (NIC) up through the kernel, where it either stays locally or gets forwarded—focusing on key execution stages, kernel functions, hooks, and protocols involved.

---

## 1. Linux 内核网络协议栈完整架构流程图

下面的架构图完整梳理了一个数据包入站（Ingress）后，在 Linux 内核内部经过硬件中断、软中断（NAPI）、网络层（IP）、Netfilter 防火墙 Hook 点、路由决策，最终走向本地接收（Local Deliver）**或**转发（Forward/Egress）的全过程。

```
[ 物理网线 / 光纤 ]
        │
        ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
| 1. 硬件层 & 网卡驱动层 (NIC & Driver)                                             |
|                                                                                 |
|  [ 物理网卡 (NIC) ] ──(DMA 传输)──> [ 接收环形缓冲区 (Rx Ring Buffer / RAM) ]       |
|         │                                                                       |
|         ├───> [ 触发 硬件中断 (Hard IRQ) ]                                        |
|         │             │                                                         |
|         │             ▼                                                         |
|         │       [ 执行 IRQ Handler ] ──> 禁用硬件中断，唤醒 NAPI                    |
|         │                                    │                                  |
|         ▼                                    ▼                                  |
|  [ eBPF / XDP Hook 点 ] <─────────── [ 触发 软中断 (SoftIRQ / NET_RX_SOFTIRQ) ]   |
|  (网卡驱动层极速处理)                          │                                   |
|         │ (XDP_PASS)                         ▼                                  |
|         └─────────────────────────> [ napi_gro_receive() ]                      |
|                                     (合并包/分配 sk_buff)                         |
└────────────────────────────────────────────────┬────────────────────────────────┘
                                                 │
                                                 ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
| 2. 链路层 (Link Layer / Layer 2)                                                 |
|                                                                                 |
|  [ __netif_receive_skb() ] ──> [ TC (Traffic Control) Ingress Hook / eBPF ]     |
|                                      │                                          |
|                                      ▼                                          |
|  [ 协议类型识别 (eth_type) ] ──> ARP 包 ──> [ arp_rcv() ]                         |
|                               ──> IP 包  ──> [ ip_rcv() ] (进入网络层)            |
└────────────────────────────────────────────────┬────────────────────────────────┘
                                                 │
                                                 ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
| 3. 网络层 (Network Layer / Layer 3 - IP)                                         |
|                                                                                 |
|  [ ip_rcv() ]                                                                   |
|        │                                                                        |
|        ▼                                                                        |
|  [ Netfilter Hook 1: PREROUTING ]  <─── (raw -> mangle -> nat)                  |
|        │                                                                        |
|        ▼                                                                        |
|  [ 路由决策 (ip_route_input_noref) ]                                             |
|        │                                                                        |
|        ├───────────────────────────────────────┐                                |
|        │ (目标 IP == 本机 IP)                   │ (目标 IP != 本机 IP)             |
|        ▼                                       ▼                                |
|  [ 本地接收分支 ]                       [ 转发分支 (Forwarding) ]                  |
|        │                                       │                                |
|        ▼                                       ▼                                |
|  [ Netfilter Hook 2: INPUT ]            [ Netfilter Hook 3: FORWARD ]           |
|  (mangle -> filter -> security)         (mangle -> filter)                      |
|        │                                       │                                |
|        ▼                                       ▼                                |
|  [ ip_protocol_deliver_rcu() ]          [ ip_forward() ]                        |
|  (查找 L4 协议处理函数)                           │                                |
|        │                                       ▼                                |
|        │                                [ Netfilter Hook 4: POSTROUTING ]       |
|        │                                (mangle -> nat)                         |
|        │                                       │                                |
|        │                                       ▼                                |
|        │                                [ ip_output() / dev_queue_xmit() ]      |
|        │                                       │                                |
|        │                                       ▼                                |
|        │                                [ 发送到目标网卡出站 (Egress) ]         |
└────────┼────────────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
| 4. 传输层 (Transport Layer / Layer 4 - TCP/UDP)                                  |
|                                                                                 |
|  [ L4 协议入口 ]                                                                 |
|        ├─── TCP 包 ──> [ tcp_v4_rcv() ] ──> 校验和/状态机/ACK                     |
|        └─── UDP 包 ──> [ udp_rcv() ]   ──> 校验和/查找 Socket                     |
|                                │                                                |
|                                ▼                                                |
|  [ eBPF / SockOps / Sockmap Hook 点 ] (可选：实现同节点跨 Socket 近零拷贝转发)       |
|                                │                                                |
|                                ▼                                                |
|  [ 查找 Socket (sock_lookup) ] ──> 写入 Socket 接收队列 (sk_receive_queue)         |
└────────────────────────────────┬────────────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
| 5. 应用层 (Application Layer / Layer 7)                                          |
|                                                                                 |
|  [ 用户态进程 (User Process) ]                                                    |
|  通过 sys_read() / sys_recvfrom() / epoll 唤醒                                   |
|  从 Socket 缓冲区读取数据包，完成内核态 -> 用户态内存拷贝 (Copy to User)               |
└─────────────────────────────────────────────────────────────────────────────────┘

```

---

## 2. 全过程六大阶段深度拆解

### 阶段一：硬件接收与中断响应 (Layer 1 ~ Layer 2 前段)

1. **DMA 传输到 Rx Ring Buffer：**

* 当数据帧（Frame）从物理介质到达网卡（NIC）时，网卡通过 **DMA（Direct Memory Access，直接内存访问）** 技术，将数据帧直接写入主机内存中预先分配好的 **Rx Ring Buffer（接收环形缓冲区）**。
* **特点：** 这个过程完全由网卡硬件完成，**不需要 CPU 参与**。

1. **触发硬件中断 (Hard IRQ)：**

* 数据写入内存后，网卡向 CPU 发送一个**硬件中断请求（Hard IRQ）**。
* CPU 暂停当前任务，执行网卡驱动注册的硬中断处理函数（Interrupt Handler）。
* **硬中断处理流程：** 为防止频繁打断 CPU，硬中断处理函数只做两件极快的事：

1. 暂时禁用（Mask）该网卡的硬件中断。
2. 触发一个**软中断（SoftIRQ: `NET_RX_SOFTIRQ`）**，将后续耗时的收包工作交由 Linux 的 **NAPI（New API）机制** 处理，然后立即退出硬中断。

---

### 阶段二：软中断与驱动层处理 (NAPI & XDP & GRO)

1. **软中断轮询 (ksoftirqd & NAPI)：**

* 内核的软中断守护进程 `ksoftirqd` 监听到 `NET_RX_SOFTIRQ` 事件，调用网卡驱动注册的 `poll()` 方法。
* NAPI 采用**轮询模式（Polling）**，从 Rx Ring Buffer 中批量拉取数据包，极大地降低了高并发流量下的中断上下文切换开销。

1. **XDP（eXpress Data Path）Hook 点（Linux 内核网络极速通道）：**

* 在网卡驱动拿到数据包、但尚未为其分配 `sk_buff`（内核网络缓冲区结构体）的极早时刻，数据包会经过 **eBPF / XDP** 挂载点。
* **XDP 可做出的决策：**
* `XDP_DROP`：直接丢弃（常用于百 G 级 DDoS 攻击防护）。
* `XDP_TX`：原网卡反弹发回。
* `XDP_REDIRECT`：绕过内核协议栈，直接重定向到其他网卡、AF_XDP Socket 或 SmartNIC。
* `XDP_PASS`：放行，继续走传统 Linux 协议栈。

1. **创建 `sk_buff` 与 GRO 合并：**

* **`sk_buff`（Socket Buffer）：** 内核为数据包分配的元数据结构体，贯穿整个内核网络栈的始终。
* **GRO（Generic Receive Offload）：** 如果开启了 GRO，内核会在链路层将多个同流的小 TCP/IP 包合并成一个巨型包，从而减少后续协议栈逐包处理的 CPU 开销。
* 随后，调用 `napi_gro_receive()` 并最终进入 `__netif_receive_skb()`，将数据包送入二层（链路层）。

---

### 阶段三：二层分发与 TC Hook (Link Layer)

1. **TC（Traffic Control）Ingress Hook 点：**

* 数据包进入 `__netif_receive_skb()` 后，首先经过 **TC（Traffic Control）Inbound 挂载点**。
* Cilium 等基于 eBPF 的 CNI 插件常挂载于此，在此处进行 BPF 路由选择、网络策略匹配或容器间流量重定向。

1. **协议识别与分发（Eth_Type）：**

* 内核检查以太网帧头部的 `EtherType` 字段：
* 如果是 `0x0806`（**ARP 协议**），交由 `arp_rcv()` 处理，更新 ARP 缓存表。
* 如果是 `0x0800`（**IPv4 协议**），去掉以太网帧头，剥离出 IP 报文，调用 `ip_rcv()` 进入三层（网络层）。
* 如果是 `0x86DD`（**IPv6 协议**），调用 `ipv6_rcv()`。

---

### 阶段四：三层路由与 Netfilter 防火墙 (Network Layer)

网络层是 Linux 网络栈的核心，所有 Netfilter（iptables）的核心 Hook 点和路由决策都在此发生。

```
                 [ 数据包进入 ip_rcv() ]
                            │
                            ▼
               [ 1. Netfilter: PREROUTING ]
                            │
                            ▼
                 [ 2. 路由决策查找 (Route) ]
                            │
           ┌────────────────┴────────────────┐
           │ (目标地址 == 本机)                │ (目标地址 != 本机)
           ▼                                 ▼
[ 3a. Netfilter: INPUT ]         [ 3b. Netfilter: FORWARD ]
           │                                 │
           ▼                                 ▼
   [ 进入 L4 传输层 ]               [ 4b. Netfilter: POSTROUTING ]
   (tcp_v4_rcv/udp_rcv)                      │
                                             ▼
                                    [ 5b. dev_queue_xmit() ]
                                       (通过目标网卡转发发出去)

```

1. **`ip_rcv()` 入口：**

* 校验 IP 头部校验和（Checksum）、检查 IP 版本与报文长度。

1. **Netfilter Hook 1：`NF_INET_PRE_ROUTING**`

* 数据包经过 iptables 的 **`raw` $\rightarrow$ `mangle` $\rightarrow$ `nat (PREROUTING)**` 表。
* **经典应用：** DNAT（目标地址转换），例如 K8s `kube-proxy` (iptables 模式) 将 ClusterIP 转换成具体的 Pod IP 地址。

1. **路由决策（Route Lookup - `ip_route_input_noref()`）：**

* 内核查询路由表（FIB），决定数据包的命运：
* **分支 A（Local Deliver）：** 如果目标 IP 地址属于**本机某个网卡**（或 Loopback），数据包走**本地接收流程**。
* **分支 B（Forward）：** 如果目标 IP **不是本机** 且系统开启了 IP 转发（`sysctl net.ipv4.ip_forward = 1`），数据包走**转发流程**。
* **分支 C（Drop）：** 不是本机 IP 且未开启转发，直接丢弃（或返回 ICMP Destination Unreachable）。

---

### 阶段五（分支 A）：本地接收流程 (Local Deliver Path)

如果路由决策确定数据包是发给本机的：

1. **Netfilter Hook 2：`NF_INET_LOCAL_IN**`

* 数据包经过 iptables 的 **`mangle` $\rightarrow$ `filter (INPUT)` $\rightarrow$ `security**` 表。
* **经典应用：** 本地防火墙规则（如拒绝某个 IP 访问本机 `iptables -A INPUT -s x.x.x.x -j DROP`）。

1. **传输层协议分发 (`ip_protocol_deliver_rcu()`)：**

* 剥离 IP 头部，检查 IP 头部的 `Protocol` 字段：
* **协议号 6 (TCP)：** 调用 `tcp_v4_rcv()`。
* **协议号 17 (UDP)：** 调用 `udp_rcv()`。
* **协议号 1 (ICMP)：** 调用 `icmp_rcv()`。

1. **传输层处理 (L4 - TCP/UDP)：**

* **UDP 路径（`udp_rcv()`）：** 计算校验和，通过 `__udp4_lib_lookup_skb()` 根据 `<源 IP, 源端口, 目的 IP, 目的端口>` 寻找匹配的 Socket，将 `sk_buff` 追加到该 Socket 的接收缓冲区队列 `sk_receive_queue` 中。
* **TCP 路径（`tcp_v4_rcv()`）：** 过程更复杂，涉及 TCP 状态机（SYN/ACK 处理、滑动窗口、序号排列、丢包重传）。处理完成后，同样将数据装入匹配 Socket 的接收队列。

1. **应用层读取（L7 - User Space）：**

* 用户态应用程序（如 Go、Java、Nginx）阻塞在 `read()`、`recv()` 或被 `epoll` 异步事件唤醒。
* 执行系统调用，内核将数据从 Socket 接收缓冲区**拷贝到应用层内存空间（Copy to User）**，并释放 `sk_buff`，全流程完成。

---

### 阶段五（分支 B）：转发流程 (Forwarding Path)

如果路由决策确定数据包需要被转发给其他主机或虚拟机/容器：

1. **Netfilter Hook 3：`NF_INET_FORWARD**`

* 数据包经过 iptables 的 **`mangle` $\rightarrow$ `filter (FORWARD)**` 表。
* 防火墙检查是否允许该流量跨网卡转发。

1. **转发处理 (`ip_forward()`)：**

* 递减 IP 头的 **TTL（Time To Live）** 值。如果 TTL $\le 0$，丢弃数据包并向源地址发送 ICMP Time Exceeded 报文。
* 重新计算 IP 头部的校验和（Checksum）。

1. **Netfilter Hook 4：`NF_INET_POST_ROUTING**`

* 数据包经过 iptables 的 **`mangle` $\rightarrow$ `nat (POSTROUTING)**` 表。
* **经典应用：** SNAT（源地址转换）或 Masquerade（网卡伪装）。把 Pod/局域网私网 IP 替换为宿主机的公网出口 IP，以便回程报文能顺利找回来。

1. **出站发送 (`ip_output()` & `dev_queue_xmit()`)：**

* 查找邻居表（ARP Cache），获取下一跳（Next Hop）物理设备的 MAC 地址。
* 构建全新的二层以太网帧头（Ethernet Header）。
* 将数据包送入目标出站网卡的传输队列（Tx Queue / QDisc），最终由网卡通过 DMA 读出并物理发送至网络介质。

---

## 3. 关键节点与对应协议/技术总结表

| 网络层级 | 核心函数 / 节点 | 关键技术 / 协议 | 核心职责与做了什么 |
| --- | --- | --- | --- |
| **Layer 1/2 (物理/网卡)** | NIC / Ring Buffer | **DMA / Hard IRQ** | 网卡收到电信号，通过 DMA 写入主机内存，发硬中断告知 CPU。 |
| **Layer 2 (驱动/软中断)** | `ksoftirqd` / NAPI | **eBPF (XDP) / GRO** | 轮询提取数据包；可在 XDP 极速丢包/转发；合并数据包，封装为 `sk_buff`。 |
| **Layer 2 (链路层)** | `__netif_receive_skb` | **Ethernet / TC (Inbound)** | 解析以太网类型（ARP/IPv4/IPv6）；运行 TC 级 eBPF 策略；分发到 L3。 |
| **Layer 3 (网络层入口)** | `ip_rcv()` | **Netfilter (PREROUTING)** | 检查 IP 头；运行 DNAT 规则（如 ClusterIP 转换）。 |
| **Layer 3 (路由引擎)** | `ip_route_input_noref` | **FIB (路由表) / BGP** | **核心枢纽：** 判定数据包是属于“本地接收”还是“转发出站”。 |
| **Layer 3 (本地分支)** | `ip_local_deliver` | **Netfilter (INPUT)** | 执行 INPUT 表防火墙过滤，放行后递交给 L4 协议 Handler。 |
| **Layer 3 (转发分支)** | `ip_forward()` | **Netfilter (FORWARD/POSTROUTING)** | 扣减 TTL，执行 SNAT/MASQUERADE 转换，查找下一跳 MAC。 |
| **Layer 4 (传输层)** | `tcp_v4_rcv` / `udp_rcv` | **TCP / UDP / Sockmap** | 处理 TCP 状态机/校验和，查找 Socket 并把数据写入 Socket 接收队列。 |
| **Layer 7 (应用层)** | `sys_recvfrom` / `read` | **Syscall / epoll** | 应用程序从 Socket 缓冲区唤醒并读取数据，完成内存从内核态到用户态的拷贝。 |
