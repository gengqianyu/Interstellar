太棒了！把数据包从**发送端应用层**出站（Egress）到**网卡物理发送**的全过程补齐，你就在脑海里构建出了 Linux 网络协议栈**闭环的完整全景图**。

与入站（Ingress）的“中断驱动+轮询拉取”不同，出站（Egress）本质上是**主动的“系统调用驱动+层层封装”**。

---

## 1. Linux 内核网络出站 (Egress) 完整流程图

数据包从用户态通过系统调用下沉，经过 Socket 缓存、TCP/UDP 封装、IP 路由与防火墙、L2 邻居表（ARP）、QDisc 队列，最后交由驱动映射 DMA 并触发网卡发包。

```
[ 用户态进程 (User Space App) ]
  执行 write() / send() / sendto()
        │
        ▼  (内存拷贝: Copy from User)
┌─────────────────────────────────────────────────────────────────────────────────┐
| 1. 套接字与传输层 (Socket & Layer 4 - TCP/UDP)                                    |
|                                                                                 |
|  [ sys_sendmsg() / inet_sendmsg() ]                                             |
|        │                                                                        |
|        ├─── UDP 途径 ──> [ udp_sendmsg() ] ──> 构建 UDP 头                        |
|        └─── TCP 途径 ──> [ tcp_sendmsg() ] ──> 写入发送队列 (sk_write_queue)       |
|                                                     │                           |
|                                                     ▼                           |
|                                          [ tcp_push_pending_frames() ]          |
|                                          (拥塞控制/滑动窗口/构建 TCP 头)            |
└────────────────────────────────┬────────────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
| 2. 网络层 (Network Layer / Layer 3 - IP)                                         |
|                                                                                 |
|  [ ip_queue_xmit() (TCP) / ip_send_skb() (UDP) ]                                |
|        │                                                                        |
|        ▼                                                                        |
|  [ 路由查找 (ip_route_output_ports) ] ──> 确定出口网卡、源/目的 IP、下一跳            |
|        │                                                                        |
|        ▼                                                                        |
|  [ Netfilter Hook 1: NF_INET_LOCAL_OUT ] (raw -> mangle -> nat -> filter)       |
|        │                                                                        |
|        ▼                                                                        |
|  [ ip_output() ] ──> 填充 IP 头部 (TTL, Checksum, Protocol)                      |
|        │                                                                        |
|        ▼                                                                        |
|  [ Netfilter Hook 2: NF_INET_POST_ROUTING ] (mangle -> nat / SNAT)              |
|        │                                                                        |
|        ▼                                                                        |
|  [ ip_finish_output() ] ──> 分片检查 (ip_fragment) 或 GSO 处理                    |
└────────────────────────────────┬────────────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
| 3. 链路层 & 排队规则 (Link Layer / Layer 2 & QDisc)                               |
|                                                                                 |
|  [ ip_finish_output2() ]                                                        |
|        │                                                                        |
|        ▼                                                                        |
|  [ 邻居表/ARP 查找 (__ipv4_neigh_lookup) ] ──> 获取下一跳的目标 MAC 地址             |
|        │                                                                        |
|        ▼                                                                        |
|  [ 构建二层以太网头 (eth_header) ] ──> 填入 Src MAC, Dst MAC, EtherType            |
|        │                                                                        |
|        ▼                                                                        |
|  [ dev_queue_xmit() ] ──> [ TC (Traffic Control) Egress Hook / eBPF ]           |
|                                      │                                          |
|                                      ▼                                          |
|  [ 排队规则 QDisc (Queueing Discipline) ] (如 fq_codel / pfifo_fast 限速排队)      |
└────────────────────────────────┬────────────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
| 4. 网卡驱动与硬件发送层 (Driver & NIC Hardware)                                    |
|                                                                                 |
|  [ 驱动发送入口 (ndo_start_xmit) ]                                                |
|        │                                                                        |
|        ▼                                                                        |
|  [ DMA 映射 (dma_map_single) ] ──> 将 skb 虚拟内存映射为网卡可读的物理地址            |
|        │                                                                        |
|        ▼                                                                        |
|  [ 挂载到发送环形缓冲区 (Tx Ring Buffer) ]                                         |
|        │                                                                        |
|        ▼                                                                        |
|  [ 敲响网卡门铃 (Doorbell Register Write) ] ──> 告知网卡有新数据可以发送              |
|        │                                                                        |
|        ▼                                                                        |
|  [ 物理网卡 (NIC) ] ──(DMA 读内存)──> 转换为电/光信号发送至网线                       |
|        │                                                                        |
|        ▼                                                                        |
|  [ 触发 Tx 完成中断 (Tx Interrupt) ] ──> 清理 Tx Ring Buffer，释放 skb 内存         |
└─────────────────────────────────────────────────────────────────────────────────┘

```

---

## 2. 出站五大核心阶段深度拆解

### 阶段一：应用层发起与内核态切换 (User Space $\rightarrow$ Syscall)

1. **系统调用触发：**

* 应用层程序调用 `write(fd, buf, len)` 或 `sendto()`。
* CPU 从用户态切换到内核态，进入 `sys_sendmsg()` 系统调用入口，再通过 `inet_sendmsg()` 路由给对应协议的发送接口。

1. **分配 `skb` 并拷贝数据：**

* **分配内存：** 内核调用 `sock_alloc_send_skb()` 为这个待发送的数据包 分配 `sk_buff` 结构体以及连续的数据缓冲区。
* **数据拷贝 (`copy_from_user`)：** 用户态内存区域（`buf`）中的数据 被拷贝到 内核分配的 `skb` 数据缓冲区（Data Buffer）中。
* *注：如果使用了 `sendfile` 或 `zero-copy` 技术，可以绕过这次内存拷贝，直接将 Page 挂在 `skb` 的分片结构（`skb_shinfo`）上。*

---

### 阶段二：传输层封装 (Layer 4 - TCP/UDP)

根据 Socket 类型的不同（SOCK_STREAM 或 SOCK_DGRAM），处理逻辑有所差异：

#### 1. UDP 路径 (`udp_sendmsg()`)

* UDP 是无连接的，处理非常直接：
* 计算 UDP 报文头部（包含源端口、目的端口、数据长度）。
* 填充 UDP Checksum（校验和）。
* 直接把构建好的 `skb` 传递给三层的 `ip_send_skb()`。

#### 2. TCP 路径 (`tcp_sendmsg()`)

* TCP 必须保证可靠传输与拥塞控制：
* 数据包**不会立刻发出去**，而是先追加到该 Socket 的发送队列 `sk_write_queue` 中。
* **触发发送 (`tcp_push_pending_frames()`)：** 结合 **Nagle 算法**、**滑动窗口（Sliding Window）** 和 **拥塞窗口（cwnd）** 决定现在是否可以发送，发送多少字节。
* **填充 TCP Header：** 设置序列号（Sequence Number）、确认号（ACK Number）、标志位（SYN/ACK/FIN/PSH）、窗口大小（Window Size）等。
* 调用 `ip_queue_xmit()` 将数据推送到网络层。

---

### 阶段三：网络层路由与防火墙 (Layer 3 - IPv4)

数据包到达网络层后，开始经历 Netfilter 和 IP 协议栈的加工：

1. **路由查找 (`ip_route_output_ports()`)：**

* 内核查找路由表（FIB），根据目的 IP 地址确定：
* 从**哪张网卡**发出去（Output Device）。
* **源 IP 地址**应该填什么。
* 数据包的**下一跳（Next Hop）IP 地址**是谁。

1. **Netfilter Hook 1：`NF_INET_LOCAL_OUT` (OUTPUT 链)**

* 数据包穿过 iptables 的 **`raw` $\rightarrow$ `mangle` $\rightarrow$ `nat (OUTPUT)` $\rightarrow$ `filter` $\rightarrow$ `security**` 表。
* **应用：** 本地发出的流量匹配规则，或者做 DNAT（如本地发出的请求被重定向到特定端口）。
* *注意：如果在 OUTPUT 链中发生了 DNAT（改了目的 IP），内核会在此处触发二次路由重新查找。*

1. **构建 IP 头部 (`ip_output()`)：**

* 填充 IP 头字段：IP 版本号（IPv4）、TTL（默认 64）、Protocol（6-TCP / 17-UDP）、源 IP、目的 IP、计算 IP 头校验和。

1. **Netfilter Hook 2：`NF_INET_POST_ROUTING` (POSTROUTING 链)**

* 数据包穿过 iptables 的 **`mangle` $\rightarrow$ `nat (POSTROUTING)**` 表。
* **经典应用：SNAT / MASQUERADE**。把 本地私网 IP 替换为 宿主机出口网卡的公网 IP。

1. **分片处理 (Fragmentation) 或 GSO：**

* 如果数据包大小大于网卡 MTU（例如 1500 字节）：
* **若开启了网卡硬件 Offload（TSO/GSO）：** 保持巨型包不动，直接交给网卡去切片（性能极高）。
* **若未开启：** 调用 `ip_fragment()` 将大 `skb` 拆分成多个小于 MTU 的小 IP 分片包，分别填充分片偏移量。

---

### 阶段四：二层邻居表与排队规则 (Layer 2 & QDisc)

1. **查找下一跳 MAC 地址 (`ip_finish_output2()`)：**

* 数据包需要封装成以太网帧，内核去**邻居表（ARP Table - `__ipv4_neigh_lookup_noref()`）** 查询下一跳 IP 对应的 MAC 地址。
* **如果命中：** 直接拿到目的 MAC 地址。
* **如果未命中：** 将该 `skb` 挂起，触发一次 ARP Request 广播，拿到 MAC 地址后再继续。

1. **填充二层帧头 (`eth_header()`)：**

* 在 `skb` 头部前面推入（`skb_push`）14 字节的以太网头：
* 目标 MAC 地址（Dst MAC）
* 源 MAC 地址（Src MAC，即本机出口网卡的 MAC）
* 协议类型（EtherType，如 `0x0800` IPv4）

1. **TC (Traffic Control) Egress Hook 与 QDisc 入列：**

* 数据包进入 `dev_queue_xmit()`，首先会触发 **TC Egress Hook**（eBPF 程序可以在这里拦截或改写出站数据包）。
* **QDisc（Queueing Discipline 排队规则）：** 数据包被推入网卡的排队规则队列（如 `pfifo_fast` 经典三队列、`fq_codel` 流量整形）。这是 Linux 进行 QoS 流量限速、优先级调度的核心机制。

---

### 阶段五：驱动与物理网卡发送 (Driver & Hardware)

1. **调用驱动发送函数 (`ndo_start_xmit`)：**

* QDisc 调度器吐出数据包，调用网卡驱动注册的发送回调函数 `ndo_start_xmit()`（如 `ixgbe_xmit_frame`）。

1. **DMA 内存映射 (`dma_map_single`)：**

* 网卡硬件无法直接理解 CPU 的虚拟内存地址。
* 驱动程序将 `skb->data` 所在的**虚拟内存地址转换为网卡可以直接访问的 DMA 物理内存地址**。

1. **写入 Tx Ring Buffer 与敲响门铃：**

* 驱动将这个 DMA 物理地址和数据长度填入网卡的 **Tx Ring Buffer（发送环形缓冲区）** 的描述符中。
* **Doorbell（敲门指令）：** 驱动写一次网卡的物理寄存器（Register Write），通知网卡：“Tx Ring 中有新数据包到了，赶紧来拿！”

1. **网卡 DMA 搬运与物理发送：**

* 网卡（NIC）收到通知，通过 **DMA 控制器** 直接读取 主机内存中的 `skb` 数据包 内容。
* 网卡内部芯片加上前导码（Preamble）、FCS 帧校验序列，将数据转化为**光电信号**发往网线。

1. **发送完成中断与 `skb` 销毁 (Tx Completion Interrupt)：**

* 网卡发包完成后，向 CPU 发送一个 **Tx 完成硬件中断**。
* 驱动响应中断，解除 DMA 内存映射。
* 调用 `dev_kfree_skb_any()` 将已经成功发送完毕的 `skb` 引用计数清零，归还内存至 Slab 缓存池。**至此，`skb` 的生命周期完美终结！**

---

## 3. 出站过程中的核心硬件加速技术 (Offloading)

在现代高性能网络中，为了避免内核频繁切片和计算校验和消耗 CPU，很多阶段都被下沉到网卡硬件完成：

* **TSO (TCP Segmentation Offload)：** TCP 可以直接把高达 64KB 的巨型数据包一路传到网卡，由网卡物理芯片将其切分成 1500 字节的 MTU 报文。
* **Tx Checksum Offload：** L3 的 IP Checksum 和 L4 的 TCP/UDP Checksum 计算完全交给网卡硬件，内核只需填 0。
* **GSO (Generic Segmentation Offload)：** 如果网卡不支持 TSO，Linux 会延迟切片时机，直到进入驱动最后一刻才用 CPU 批量切片，尽量减少前面协议栈的处理次数。

---

## 4. 入站 (Ingress) vs 出站 (Egress) 核心对比总结

| 维度 | **收包 (Ingress)** | **发包 (Egress)** |
| --- | --- | --- |
| **驱动力** | 外部网络电信号/硬件中断驱动 | 本地应用层系统调用（`write`/`send`）驱动 |
| **内存分配** | 在驱动极早期由内核/NAPI 分配 `skb` | 在系统调用入口由应用层请求内核分配 `skb` |
| **Netfilter 顺序** | `PREROUTING` $\rightarrow$ `INPUT` | `OUTPUT` $\rightarrow$ `POSTROUTING` |
| **流量控制** | TC Ingress / XDP（早期丢包） | QDisc / TC Egress（排队、整形、限速） |
| **销毁时机** | 用户态 `read` 拷贝完数据后释放 `skb` | 网卡发送完成触发 Tx 中断后释放 `skb` |

现在，从**应用层调用 -> 内核协议栈加工 -> 网卡 DMA 出站**，以及**网卡入站 -> 协议栈解包 -> 写入 Socket -> 应用层读取**，加上 **eBPF (XDP/TC/kprobe) 挂载点** 以及 **`skb` 的生命周期**，你已经完整掌握了 Linux 内核网络的底层底层运行图景！
