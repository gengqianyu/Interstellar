在传统的网络发送流程中，如果要把磁盘上的文件（如 Nginx 提供静态文件下载）通过 Socket 发送出去，数据需要经历 **4 次上下文切换** 和 **4 次内存拷贝**（其中包括 2 次 CPU 拷贝）。

针对这一性能瓶颈，**`sendfile` / Zero-Copy（零拷贝）** 技术应运而生。它不仅避免了 CPU 在用户态与内核态之间搬运数据，而且在内核内部，通过巧妙利用 `sk_buff` 的 **`skb_shared_info`（简称 `skb_shinfo`）** 结构，实现了**数据 Payload 的零 CPU 拷贝**。

---

## 1. 传统网络发送 vs. 零拷贝发送

### 传统路径 (4 次拷贝 + 4 次上下文切换)

```
[ 磁盘 ] ──(DMA 拷贝)──> [ Page Cache (内核) ]
                              │
                        (CPU 拷贝 1: 读到用户态)
                              │
                              ▼ 
                       [ User Buffer (应用) ]
                              │
                        (CPU 拷贝 2: 写回 Socket)
                              │
                              ▼ 
                       [ Socket Buffer / skb (内核) ]
                              │
                            (DMA 拷贝)
                              │
                              ▼ 
                       [ 网卡 (Hardware) ]

```

### 零拷贝路径 (`sendfile` / `splice` / MSG_ZEROCOPY)

数据**完全不经过用户态内存**，并且在内核空间内，**数据字节本身也不会从 Page Cache 拷贝到 `skb` 的数据缓冲区中**。

```
[ 磁盘 ] ──(DMA 拷贝)──> [ Page Cache (内核内存页) ] 
                              │
                    (零 CPU 拷贝！仅将 SKB(头部分)+ Page 物理页地址/指针（playload 数据地址）)
                    (挂载到 skb_shared_info 的 frags 数组)
                              │ 
                              ▼
                    [ sk_buff (仅包含网络包头 + Page 指针) ]
                              │
                    (DMA 散射/聚集 直接读取 Page Cache 发送)
                              │
                              ▼ 
                       [ 网卡 (Hardware) ]

```

---

## 2. 核心魔法机制：`skb_shared_info` 是什么？

每个 `sk_buff` 在内存结构上，都紧跟着一块共享信息区——`skb_shared_info`。

一个 `skb` 的实际内存布局如下：

```
+-------------------------------------------------------------------+
|                        struct sk_buff 元数据                      |
+-------------------------------------------------------------------+
|                                                                   |
|  skb->head ──────> +-------------------------------------------+  |
|                    | Linear Data Area (线性缓冲区)              |  |
|  skb->data ──────> |  [ ETH Header | IP Header | TCP Header ]  |  |
|  skb->tail ──────> |  (物理内存连续，由 CPU 填充网络包头)       |  |
|                    +-------------------------------------------+  |
|  skb->end  ──────> | ...                                       |  |
+--------------------|-------------------------------------------|--+
                     | struct skb_shared_info (非线性/共享数据区) |
                     |                                           |
                     |  nr_frags: 页分片数量                      |
                     |  frags[]: [ Page 1, Page 2, Page 3 ... ]  |
                     |           (分散/聚集 IO 页面指针)          |
                     +-------------------------------------------+

```

### `skb_shared_info` 的关键成员

```c
struct skb_shared_info {
    unsigned char   nr_frags;      /* 非线性区包含的物理内存页 (Page) 数量 */
    skb_frag_t      frags[MAX_SKB_FRAGS]; /* 页分片数组，记录 Page 指针、偏移量和长度 */
    /* ... 还有 GSO/TSO 相关的分片信息 ... */
};

```

1. **线性缓冲区（Linear Area）：** 由 `skb->head` 到 `skb->end` 定义，**必须是物理内存连续的**。这里只用来存放**网络协议头**（以太网头、IP 头、TCP 头）。
2. **非线性缓冲区（Non-linear Area）：** 也就是 `skb_shared_info` 里的 `frags` 数组（Page Frags）。这里保存的**不是数据字节**，而是**物理内存页（`struct page *`）的引用指针、页内偏移量（`page_offset`）和数据长度（`size`）**。

---

## 3. `sendfile` 工作时的内部全过程拆解

当你在应用层调用：

```c
ssize_t sendfile(int out_fd, int in_fd, off_t *offset, size_t count);

```

内核底层（如 Linux `do_sendfile` / `generic_file_splice_read`）会按以下步骤处理：

### 第一步：读取文件到 Page Cache (DMA)

内核检查文件数据是否已在内核的 **Page Cache** 中：

* 如果在，直接拿到该 Page 的物理内存页指针 `struct page*`。
* 如果不在，触发缺页中断，通过 **DMA** 将文件内容从磁盘直接读取到内存 Page Cache（此时 Page 的引用计数 `page_ref_inc` 加 1）。

### 第二步：构建“轻量级” `skb`

内核为这次发送分配一个 `sk_buff`，但与传统发送不同：

* **线性区：** 只保留几十个字节的空间，由 CPU 动态构建 TCP/IP 协议头部。
* **数据 Payload：** **完全不申请物理内存来放数据**！

### 第三步：把 Page 挂载到 `skb_shared_info`

内核直接将 Page Cache 中的 `struct page*` 赋值给 `skb_shared_info` 的 `frags` 数组：

```c
// 伪代码逻辑展示：
skb_shinfo(skb)->frags[0].bv_page   = page_cache_page; // 物理页指针
skb_shinfo(skb)->frags[0].bv_offset = offset;          // 页内偏移
skb_shinfo(skb)->frags[0].bv_len    = len;             // 数据长度
skb_shinfo(skb)->nr_frags++;                           // 分片数 +1

```

> **重点：** 这个过程**只拷贝了几个 64 位的内存指针和数值**，数据字节本身（比如 1GB 的视频文件）**完全没有发生任何 CPU 拷贝**！

### 第四步：网卡通过 DMA Scatter-Gather (分散/聚集) 物理发送

数据包下发到网卡驱动时，现代网卡几乎都支持 **DMA Scatter-Gather（SG，分散/聚集 DMA）** 硬件功能：

1. 驱动填充网卡的 Tx Ring Buffer 描述符：

* **描述符 1：** 指向 `skb` 线性区（传输 TCP/IP 头）。
* **描述符 2：** 指向 `skb_shinfo->frags[0]` 对应的 Page Cache 物理地址（传输文件 Payload 数据）。

1. 网卡收到发包指令（Doorbell）后，**硬件 DMA 控制器** 会根据描述符：

* 先从线性区读入包头；
* 紧接着直接从 Page Cache 物理内存页读入文件内容；
* 将它们**在网卡内部拼成一个完整的以太网帧**发送到网线上！

### 第五步：发送完成与释放 (Clean Up)

网卡发送完毕触发 Tx 完成中断后：

1. 内核释放 `skb` 结构体。
2. 递减 `skb_shinfo->frags` 中挂载的 Page 的引用计数。
3. Page Cache 依然保留在内存中，供后续其他进程读取复用。

---

## 4. 衍生与拓展：应用程序内存的零拷贝 (`MSG_ZEROCOPY`)

`sendfile` 主要是针对 **“文件 FD $\rightarrow$ Socket FD”** 的场景。但如果数据是**应用程序在内存里动态生成的**（比如内存缓存系统 Redis/Memcached，或者 RPC 框架），怎么做零拷贝？

Linux 4.14 引入了 **`socket(..., MSG_ZEROCOPY)`** 机制：

1. **机制：** 应用层调用 `send(fd, buf, len, MSG_ZEROCOPY)`。
2. **内核操作：** 内核不会把 `buf` 的数据拷贝到 `skb`，而是直接将**用户态内存对应的虚拟页（Virtual Pages）进行 Pin 锁定**（防止被 Swap 或释放），同样把这些 User Pages 挂载到 `skb_shared_info` 的 `frags` 数组上。
3. **网卡 DMA：** 网卡直接从**用户态内存**通过 DMA 抓取数据发送出去。
4. **异步通知：** 发送完成后，内核通过 Socket 错误队列（`ERRQUEUE`）异步通知应用程序：“这块内存数据网卡已经发完了，你可以安全修改或释放 `buf` 了”。

---

## 5. 总结：零拷贝技术的技术精髓

| 维度 | **传统 `write()` / `send()**` | **`sendfile` / `splice` (零拷贝)** |
| --- | --- | --- |
| **内存拷贝途径** | 磁盘 $\rightarrow$ Page Cache $\rightarrow$ **用户 Buffer** $\rightarrow$ **Socket Buffer** $\rightarrow$ 网卡 | 磁盘 $\rightarrow$ Page Cache $\rightarrow$ 网卡 |
| **CPU 拷贝次数** | **2 次**（Page Cache $\rightarrow$ User $\rightarrow$ Socket） | **0 次**（数据字节完全由 DMA 搬运） |
| **数据载体** | 拷贝并填充到 `skb` 的**线性数据区** | 仅将 Page 页面指针挂载在 **`skb_shinfo->frags`** 数组 |
| **网卡硬件要求** | 普通 DMA 网卡 | 需要网卡支持 **DMA Scatter-Gather (SG)** |
| **典型应用场景** | 动态数据加密、压缩、修改后发送 | Nginx / Kafka / RocketMQ 静态文件与日志高效传输 |

正是因为 `skb_shared_info` 这种将“协议控制头（线性区）”与“载荷数据（非线性页区）”解耦的设计，使得 Linux 内核能够极其优雅地支撑起高性能的零拷贝体系！

---
**=零拷贝（Zero-Copy）核心概念中最容易被误解的地方！**

简短回答你的两个问题：

1. **是的，`sendfile` 省掉的确实只有 Payload（载荷数据）的拷贝。**
2. **是的，`skb` 结构体本身和 TCP/IP 包头依然要经历完整的 Linux 内核网络协议栈，包头也依然是由 CPU 亲自填充/构建（产生极少量的内存写入）的。**

所谓“零拷贝（Zero-Copy）”，指的是**数据本身的字节（Payload）零拷贝**，而不是 “完全不需要 CPU 参与，或者绕过了协议栈”。

---

## 一、 到底什么被省掉了，什么依然存在？

我们可以把一个网络数据包拆解为两部分：

$$\text{数据包} = \underbrace{\text{网络包头 (Protocol Headers)}}_{\text{占比极小 (约 54~66 字节)}} + \underbrace{\text{数据载荷 (Payload)}}_{\text{占比极高 (如 1440+ 字节 或 64KB TSO 巨型包)}}$$

| 模块 | 传统 `read` + `write` | `sendfile` (零拷贝) | 说明 |
| --- | --- | --- | --- |
| **Payload 数据** | **2 次 CPU 拷贝**<br>

<br>(Page Cache $\rightarrow$ User $\rightarrow$ skb) | **0 次 CPU 拷贝** | 文件的海量数据字节完全不经过 CPU 拷贝，由 DMA 直接从内存送往网卡。 |
| **TCP/IP 协议头** | CPU 在内核构建并写入 | **CPU 在内核构建并写入** | 必须由 CPU 按照协议栈逻辑填充（源端口、序列号、校验和等）。 |
| **`skb` 结构体** | 内核分配并经过完整协议栈 | **内核分配并经过完整协议栈** | 路由查找、iptables/Netfilter、QDisc 排队规则**全都要走一遍**。 |

---

## 二、 为什么说完整的 `skb` 依然走完了协议栈？

`sendfile` 并没有走任何“协议栈旁路（Bypass）”机制（如 DPDK 或 XDP 才是真正的旁路）。

当调用 `sendfile` 时，数据包在内核中的出站旅程如下：

1. **传输层 (L4)：** CPU 依然要为这次发送分配 `sk_buff` 结构体，分配几百字节的线性缓冲区用来放包头。TCP 状态机、滑动窗口、拥塞控制算法（CUBIC/BBR）依然在严格运行。
2. **网络层 (L3)：** `skb` 依然要通过 `ip_queue_xmit()`，查找路由表决定出口网卡，穿过 **Netfilter 的 `OUTPUT` 和 `POSTROUTING` 链**（你的 iptables / nftables 规则对 `sendfile` 发出的包完全生效！）。
3. **链路层 (L2)：** 依然要查找 ARP 表、填充以太网头、经过 **TC (Traffic Control) 和 QDisc 排队规则**（限速/QoS 规则同样生效）。

**也就是说：所有网络协议的控制逻辑、防火墙、路由、流控，一个都没有少。**

---

## 三、 那 CPU 拷贝到底“省”在哪？性能提升为什么这么大？

既然 CPU 还要分配 `skb`，还要写包头，凭什么 `sendfile` 性能高那么多？

我们来看一组数据对比：

假设你要通过 Nginx 发送一个 **1GB** 的静态文件。

### 1. 传统 `read()` + `write()`

* **内存拷贝数据量：** CPU 需要把这 1GB 的数据从内核读到用户态，再从用户态写回内核。**CPU 总共搬运了 2GB 的数据字节！**
* **缓存污染（L1/L2/L3 Cache）：** 这 2GB 的数据全部要经过 CPU 的 L1/L2/L3 缓存，把 CPU 缓存里原本高效的指令和热点数据全部挤出去（Cache Pollution）。
* **上下文切换：** 上千次 `read`/`write` 系统调用带来的用户态与内核态频繁切换。

### 2. `sendfile()`

* **内存拷贝数据量：**
* Payload（1GB）：**0 字节** 由 CPU 拷贝，完全交由网卡 DMA 异步搬运。
* 包头（Header）：如果开启了网卡 **TSO (TCP Segmentation Offload)**，TCP 协议栈可以构建一个高达 64KB 的“巨型 skb”。CPU 只需要写 **几十个字节** 的包头，剩下的交给网卡去切片。
* 发送 1GB 文件，CPU 实际亲手写入的包头数据总量可能**只有几百 KB**！

* **缓存保护：** CPU 缓存完全不会被 1GB 的文件内容污染。

---

## 总结

你的直觉非常敏锐：
**`sendfile` 并没有消灭协议栈，也没有消灭 `skb`，更没有让 CPU 完全闲着。它只是通过把 Payload 数据映射为物理页指针（`skb_shinfo`），把最沉重、最耗费 CPU 算力和内存带宽的“海量数据搬运工作”，彻底甩给了网卡 DMA 硬件。**

这也是 Linux 内核设计的极其精妙之处——在**保持完整网络协议栈功能（防火墙、流控、路由）不打折扣**的前提下，拿到了近乎物理极限的传输性能。
