# Socket

```c
int socket(int domain, int type, int protocol)
```

**socket()** 创建通信端点并返回描述符。

**域参数**（domain）指定将在其中进行通信的通信域；这将选择应该使用的协议族。

这些族在包含文件 `<sys/socket.h>` 中定义。

目前了解的格式是：

- **PF_LOCAL** — 主机内部协议，以前称为 PF_UNIX
- **PF_UNIX** — 主机内部协议，不推荐使用，请使用 PF_LOCAL
- **PF_INET** — Internet 版本 4 协议（IPv4 协议）
- **PF_ROUTE** — 内部路由协议
- **PF_KEY** — 内部密钥管理功能
- **PF_INET6** — Internet 版本 6 协议（IPv6 协议）
- **PF_SYSTEM** — 系统域
- **PF_NDRV** — 对网络设备的原始访问
- **PF_VSOCK** — VM 套接字协议

---

套接字具有指定的**类型**（type），该类型指定通信的语义。

当前定义的类型包括：

- **SOCK_STREAM**
- **SOCK_DGRAM**
- **SOCK_RAW**

**SOCK_STREAM** 类型提供基于顺序、可靠、双向连接的字节流。可能支持带外数据传输机制。

**SOCK_DGRAM** 套接字支持数据报（最大长度固定（通常较小）的无连接、不可靠消息）。

**SOCK_RAW** 套接字提供对内部网络协议和接口的访问。类型 SOCK_RAW，仅对超级用户可用。

---

**protocol** 指定要与套接字一起使用的特定协议。

通常只有一个协议支持给定协议家族中的特定套接字类型。
然而，可能存在许多协议，在这种情况下，必须以这种方式指定特定的协议。
要使用的协议号特定于要进行通信的"通信域"；见协议（5）。

SOCK_STREAM 类型的套接字是全双工字节流，类似于管道。
流套接字必须处于连接状态，才能在其上发送或接收任何数据。
使用 connect(2) 或 connectx(2) 调用创建到另一个套接字的连接。
连接后，可以使用 read(2) 和 write(2) 调用或 send(2) 与 recv(2) 的某些变体来传输数据。
会话完成后，可以执行 close(2)。
带外数据也可以按照 send(2) 中的描述进行传输，并按照 recv(2) 的描述进行接收。

### protocol 参数

协议名称数据库，有 ip、icmp、igmp、tcp、udp、ipv4、ipv6 等等，以什么样的协议创建 socket。
