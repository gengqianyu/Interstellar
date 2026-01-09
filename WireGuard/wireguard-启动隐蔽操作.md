是的，**你的理解是完全正确的**，而且这是 **WireGuard 工作机制里一个“必须牢牢记住”的点**。
我先给**确定结论**，再把**细节、边界情况、以及你在 SDP 场景下要特别注意的坑**一次性讲清楚。

---

## 一、直接结论（斩钉截铁）

> **在 Linux / macOS / Windows 上：**
>
> ```ini
> AllowedIPs = 10.213.148.0/24
> ```
>
> **WireGuard 接口启动时，会自动向系统路由表中添加一条路由：**
>
> ```text
> 10.213.148.0/24 → wg0
> ```

也就是说：

> **所有目的地址是 10.213.148.0/24 的流量，都会被送进 wg0 tun 设备**

你刚才关于“包为什么会进 wg0”这一整套推理，**前提条件就在这里**，而且你现在已经完全踩中了。

---

## 二、为什么 WireGuard 一定要这么做？

因为 WireGuard 的设计理念是：

> **AllowedIPs = 路由表 + 安全策略 的合体**

它不是“只是一个 ACL”，而是：

### AllowedIPs 同时承担三件事

1. **路由选择（Routing）**
2. **Peer 选择（Which peer to encrypt to）**
3. **接收包的合法性校验（Source validation）**

---

## 三、AllowedIPs → 路由 的具体行为

### 1️⃣ 接口启动时

当你执行：

```bash
wg-quick up wg0
```

wg-quick 会做几件事（简化版）：

```bash
ip link add wg0 type wireguard
ip addr add 10.0.0.2/32 dev wg0
ip link set wg0 up

wg set wg0 peer <peer-pubkey> allowed-ips 10.213.148.0/24

ip route add 10.213.148.0/24 dev wg0
```

👉 **这条 route 就是你关心的那条**

---

### 2️⃣ 当应用访问目标服务

```bash
curl 10.213.148.96:8080
```

内核查路由表：

```text
10.213.148.96 ∈ 10.213.148.0/24
```

于是：

```text
→ 发给 wg0
```

---

## 四、AllowedIPs ≠ 只有“出方向”的路由

这是很多人第一次用 WireGuard 时会误解的地方。

### AllowedIPs 还是“入方向的安全规则”

当 PE 解密一个包时：

```text
SRC = 10.0.0.2
```

WireGuard 会检查：

```text
这个 SRC 是否 ∈ 该 peer 的 AllowedIPs？
```

如果不在：

> ❌ **直接丢包**

📌 所以 AllowedIPs 是 **“双向约束”**

---

## 五、AllowedIPs 配错时的经典现象（你以后一定会遇到）

### ❌ 情况 1：没配 10.213.148.0/24

表现：

* `curl` 一直超时
* wg 隧道看似是 “up” 的
* tcpdump wg0 上没有包

原因：

```text
包根本没进 wg0
```

---

### ❌ 情况 2：AllowedIPs 配得太宽

```ini
AllowedIPs = 0.0.0.0/0
```

表现：

* 客户端“全局代理”
* DNS、apt、ssh 全走隧道
* PD / PE 自己都可能被绕进去

📌 **在 SDP 架构里，这是必须严控的**

---

## 六、在你这个 SDP 架构下，AllowedIPs 的“战略意义”

你现在可以非常清晰地看到：

> **PD 实际上就是在“远程控制客户端的路由表”**

### PD 下发 AllowedIPs =

* ❌ 不是“你能访问什么 IP”
* ✅ 而是：

  * **你能把哪些流量送进加密隧道**
  * **你能和哪些 PE 建立数据平面**

在 SDP 世界里，这就是：

> **Capability 的数据面体现**

---

## 七、你现在已经具备继续往下走的全部前置条件了

你已经搞清楚了：

* ✔ WireGuard 包的真实形态
* ✔ wg0 上的源地址是怎么来的
* ✔ AllowedIPs 为什么等于路由
* ✔ 为什么 NAT 规则这样写是对的

---

## 八、下一步我建议你做的（非常自然）

你现在有三个**非常合理**的延伸方向：

1. **AllowedIPs 动态变更**

   * PD 下发变更
   * 客户端不中断隧道
   * 路由即时生效

2. **多 PE 场景**

   * 同一个 Client
   * 多个 Peer
   * 不同 AllowedIPs 指向不同 PE

3. **AllowedIPs 与“服务策略”的映射关系**

   * 一个 Service → 一个 CIDR
   * 一个 Identity → 一组 CIDR

你选一个，我们继续往下拆。
