在 Linux 内核网络协议栈中，**`sk_buff`（简称 `skb`，即 Socket Buffer）** 是最核心的数据结构，它就像是一个“包裹承载盒”，贯穿了网络数据包从网卡接收、协议栈解析、Socket 缓存，直到最终被用户态读取并释放的全过程。

下面为你详细梳理 `skb` 从**分配（产生）**、**流转（修改/传递）** 到 **销毁（释放）** 的整个生命周期主要节点。

---

## 一、 `skb` 的生命周期概览图

```
[ 物理网卡 (DMA) ]
        │
        ▼
 (1) 分配 (Allocation) ──> napi_alloc_skb() / netdev_alloc_skb()  <-- 内存池分配
        │
        ▼
 (2) 链路层构建          ──> eth_type_trans()                      <-- 填入 skb->protocol, mac_header
        │
        ▼
 (3) 网络层解析 (L3)     ──> ip_rcv() / ip_route_input()           <-- 填入 network_header, skb_dst
        │
        ▼
 (4) 传输层处理 (L4)     ──> tcp_v4_rcv() / udp_rcv()              <-- 填入 transport_header, skb->sk
        │
        ▼
 (5) Socket 队列入队     ──> skb_queue_tail(&sk->sk_receive_queue) <-- 挂载到套接字接收缓存
        │
        ▼
 (6) 用户态读取          ──> sys_recvfrom() / skb_copy_datagram()   <-- 拷贝数据到用户空间 Memory
        │
        ▼
 (7) 销毁 (Free)        ──> kfree_skb() / consume_skb()            <-- 归还内存至 slab/NAPI cache

```

---

## 二、 详细生命周期节点拆解

### 节点 1：产生与分配 (Allocation Stage)

在网卡驱动通过 NAPI 轮询收包时，`skb` 结构体和它的数据缓冲区（Data Buffer）被正式创建。

* **关键内核 API：** `napi_alloc_skb()` 或 `netdev_alloc_skb()`
* **内部动作：**

1. **申请内存：** 从内核的 `skbuff_head` slab 分配器（或 NAPI 本地内存页缓存）中分配两块内存：

* **`sk_buff` 结构体本身：** 存放元数据（指针、长度、头位置等）。
* **连续的数据缓冲区（Data Buffer）：** 存放实际的网络包字节（包含 head/data/tail/end 指针）。

1. **网卡 DMA 填充：** 驱动将这块数据缓冲区的物理地址赋给网卡的 Rx Ring Buffer，网卡通过 DMA 将物理链路上的报文内容写入该区域。
2. **关联设备：** 设置 `skb->dev = dev`（记录接收该数据包的网卡设备）。

---

### 节点 2：二层/链路层处理 (Link Layer - L2)

* **关键函数：** `eth_type_trans()` $\rightarrow$ `__netif_receive_skb()`
* **内部动作：**

1. **定位 Mac Header：** 调用 `skb_reset_mac_header(skb)`，将 `skb->mac_header` 指针指向二层帧头（以太网头）。
2. **识别三层协议：** `eth_type_trans()` 剥离以太网帧头，提取 `EtherType`（如 IPv4 `0x0800`），将其写入 `skb->protocol`。
3. **移动数据指针：** 调用 `skb_pull(skb, ETH_HLEN)`，把 `skb->data` 指针向后移动 14 字节（跳过以太网头），此时 `skb->data` 正式指向 IP 报文头。

---

### 节点 3：三层/网络层处理 (Network Layer - L3)

* **关键函数：** `ip_rcv()` $\rightarrow$ `ip_route_input_noref()` $\rightarrow$ `ip_local_deliver()`
* **内部动作：**

1. **定位 Network Header：** 执行 `skb_reset_network_header(skb)`，记录 IP 头的起始偏移位置。
2. **设置路由信息（`skb_dst`）：**

* 调用路由查找函数后，将查询到的路由条目（`dst_entry`）绑定到 `skb` 的 `skb->_skb_refdst` 属性上。
* 此时 `skb` 知道了自己是需要“本地接收（Local Deliver）”还是“转发（Forward）”。

1. **剥离 IP 头：** 确认是本地接收后，调用 `skb_pull(skb, ip_hdrlen(skb))`，`skb->data` 再次向后移动，跳过 IP 头，指向 L4 报文头（TCP/UDP 头）。

---

### 节点 4：四层/传输层处理 (Transport Layer - L4)

* **关键函数：** `tcp_v4_rcv()` / `udp_rcv()`
* **内部动作：**

1. **定位 Transport Header：** 执行 `skb_reset_transport_header(skb)`，记录 TCP/UDP 头的偏移。
2. **寻找 Socket 并绑定（`skb->sk`）：**

* 根据 `<源 IP, 源端口, 目的 IP, 目的端口>` 在内核四元组 Hash 表中查找匹配的 Socket（`struct sock`）。
* 查找到之后，执行 `skb_set_owner_r(skb, sk)`，将 `skb->sk` 指针指向该 Socket，并增加该 Socket 缓冲区的已用内存计数（`sk_rmem_alloc`）。

1. **校验和与排序：**

* **UDP：** 简单校验后直接准备入队。
* **TCP：** 处理 TCP 顺序、滑动窗口、重排乱序包。如果是乱序包，`skb` 会被临时挂在 TCP 的乱序队列（`out_of_order_queue`）上。

---

### 节点 5：Socket 接收队列入队 (Queueing Stage)

* **关键函数：** `sock_queue_rcv_skb()` $\rightarrow$ `skb_queue_tail()`
* **内部动作：**

1. **加入链表：** 将 `skb` 追加到套接字接收队列 `sk->sk_receive_queue` 的末尾（双向链表结构）。
2. **唤醒进程：** 调用 `sk->sk_data_ready(sk)`，唤醒因 `read()` / `recv()` / `epoll_wait()` 阻塞在该 Socket 上的用户态线程。

---

### 节点 6：用户态读取与数据拷贝 (Copy to User)

当用户态应用程序执行 `read(fd, buf, len)` 或 `recvfrom()` 系统调用时：

* **关键函数：** `tcp_recvmsg()` / `udp_recvmsg()` $\rightarrow$ `skb_copy_datagram_msg()`
* **内部动作：**

1. **出队：** 从 `sk_receive_queue` 链表中取出头部的 `skb`。
2. **内存拷贝：** 执行 `copy_to_user()`，将 `skb` 中 Payload（有效载荷）部分的数据从内核态数据缓冲区拷贝到应用程序的内存 `buf` 中。
3. **注：** 此过程**只拷贝应用层有效数据**，`skb` 结构体和包头元数据依然保留在内核空间。

---

### 节点 7：销毁与内存回收 (Destruction Stage)

数据拷贝完毕后，`skb` 的使命结束，进入销毁流程。

* **关键内核 API：** `kfree_skb()` 或 `consume_skb()`
* **内部动作（递减引用计数并回收）：**

1. **引用计数递减：** `skb` 内部维护着引用计数 `users`（通过 `atomic_dec_and_test(&skb->users)`）。只有当 `users` 减为 0 时，才真正触发释放。
2. **解绑 Socket（`skb_orphan`）：** 调用 `skb_orphan(skb)`，解绑 `skb->sk`，并相应减少 Socket 的 `sk_rmem_alloc` 内存配额。
3. **释放路由关联（`dst_release`）：** 归还/递减绑定的 `dst_entry` 引用计数。
4. **归还内存：**

* 释放 `skb` 的数据缓冲区（Data Buffer）内存。
* 释放 `sk_buff` 结构体本身，将其放回 Slab 缓存池（如 `skbuff_head_cache`）或 NAPI 本地 Cache，供后续的新数据包**复用**。

---

## 三、 特殊情况下的 `skb` 路径

1. **转发（Forwarding）场景：**

* 在三层路由查找后，如果发现不是发给本机的包，`skb` **不会**经历节点 4、5、6。
* 它会直接进入 `ip_forward()` $\rightarrow$ Netfilter `POSTROUTING` $\rightarrow$ `dev_queue_xmit()`，从另一个网卡发出去。发完后由网卡 TX 完成中断触发 `consume_skb()` 销毁。

1. **XDP 丢弃 / 防火墙 DROP 场景：**

* 如果数据包在 XDP 被 `XDP_DROP`，或者被 iptables / eBPF 规则拦截，`skb` 会在对应的 Hook 点直接调用 `kfree_skb()` 被**提前销毁**，不再向上层传递。
