# NAT 节点 B iptables 规则

公网节点 A `10.211.55.25` 通过 NAT 节点 B [公网 `10.211.55.26`][内网 `10.37.129.26`] 与内网节点 C `10.37.129.24` 建立 WireGuard 通信的主要设置。

---

## 节点 B 的网络设置

```bash
gengqianyu@keepalived:/etc/netplan$ sudo cat 00-enps06-config.yaml
[sudo] password for gengqianyu:
```

```yaml
network:
  version: 2
  ethernets:
    enp0s6:
      dhcp4: false
      addresses:
        - 10.37.129.26/24
      nameservers:
        addresses:
          - 8.8.8.8
          - 114.114.114.114
          - 127.0.0.53
      optional: true
```

```bash
gengqianyu@keepalived:/etc/netplan$ sudo cat 00-installer-config.yaml
```

```yaml
# This is the network config written by 'subiquity'
network:
  ethernets:
    enp0s5:
      dhcp4: true
  version: 2
```

---

## 节点 B 的 iptables 规则

```bash
# 清空现有规则（避免冲突）
sudo iptables -F
sudo iptables -t nat -F

# 1. 允许外网（enp0s5）到内网（enp0s6）的流量转发（A 10.211.55.25 → C 10.37.129.24）
sudo iptables -A FORWARD -i enp0s5 -o enp0s6 -s 10.211.55.25 -d 10.37.129.24 -j ACCEPT

# 2. 允许内网（enp0s6）到外网（enp0s5）的"相关/已建立"连接（C 10.37.129.24 → A 10.211.55.25 的响应）
sudo iptables -A FORWARD -i enp0s6 -o enp0s5 -s 10.37.129.24 -d 10.211.55.25 -m state --state RELATED,ESTABLISHED -j ACCEPT

# 3. DNAT：将节点 A 10.211.55.25 发送到节点 B 外网 IP 10.211.55.26 的流量，转发到节点 C 10.37.129.24
# （即节点 A 10.211.55.25 访问节点 B 10.211.55.26 时，实际被转发到 C 10.37.129.24）
sudo iptables -t nat -A PREROUTING -i enp0s5 -s 10.211.55.25 -d 10.211.55.26 -j DNAT --to-destination 10.37.129.24

# 4. SNAT：将节点 C 10.37.129.24 返回给节点 A 10.211.55.25 的流量，源 IP 转换为节点 B 的外网 IP 10.211.55.26
# （确保 A 收到的响应来自 10.211.55.26，与节点 A 的请求目标 10.211.55.26 一致）
sudo iptables -t nat -A POSTROUTING -o enp0s5 -s 10.37.129.24 -d 10.211.55.25 -j SNAT --to-source 10.211.55.26

# 允许 UDP 51820 端口通过节点 B 转发（WireGuard 默认端口）
sudo iptables -A FORWARD -i enp0s5 -o enp0s6 -p udp --dport 51820 -j ACCEPT
sudo iptables -A FORWARD -i enp0s6 -o enp0s5 -p udp --sport 51820 -m state --state RELATED,ESTABLISHED -j ACCEPT

# DNAT：将节点 B 外网 IP 10.211.55.26 的 51820 端口转发到节点 C 10.37.129.24:51820 端口
sudo iptables -t nat -A PREROUTING -i enp0s5 -p udp --dport 51820 -j DNAT --to-destination 10.37.129.24:51820
```

```bash
# 凡是通过 10.37.129.0/24 网络的流量，直接走 enp0s6 mac 链路层转发，不经过路由。
gengqianyu@keepalived:/etc/netplan$ ip route show
10.37.129.0/24 dev enp0s6 proto kernel scope link src 10.37.129.26
```

---

## 节点 A 的 WireGuard 配置

```bash
gengqianyu@haproxy-backup:~/wireguard$ cat wg0.conf
```

```ini
[Interface]
PrivateKey = WJ6QhCSHNkXx9oHwQX0IhvC4W+qM7cGbbxZIZCl38ls=
ListenPort = 51820

[Peer]
PublicKey = EBn6yv7ePfzpAFQuROjaY1tfCahdj5rM9gkLOJojm3c=
AllowedIPs = 10.0.0.1/32
Endpoint = 10.211.55.26:51820  # 重点是这里因为会经过 NAT 才能到达内网节点 C 10.37.129.24 的 WireGuard 服务，因此端点为节点 B 10.211.55.26 需要经过它转发。
PersistentKeepalive = 25
```

### 节点 A 的网络配置

去 `10.37.129.0/24` 的流量直接发送给 `10.211.55.26` 地址，因此在节点直接 ping `10.37.129.24` 也是可以通的。

```bash
gengqianyu@haproxy-backup:~/wireguard$ sudo cat /etc/netplan/00-installer-config.yaml
```

```yaml
# This is the network config written by 'subiquity'
network:
  ethernets:
    enp0s5:
      dhcp4: true
      routes:
        - to: 10.37.129.0/24
          via: 10.211.55.26
  version: 2
```

---

## 节点 C 的 WireGuard 配置

```bash
gengqianyu@haproxy:~/wireguard$ cat wg0.conf
```

```ini
[Interface]
PrivateKey = CJ+k+s1HTgkBX72WXUkUkiITp8nnZDkhRBlS0M7zeFY=
ListenPort = 51820

[Peer]
PublicKey = hyO37XdskPgkD3kVlkmpbTSpXCDoNNFvNxxJpSev9TQ=
AllowedIPs = 10.0.0.2/32
```

### 节点 C 的网络配置

默认流量直接给节点 C `10.37.129.26`，它为网关。

```bash
gengqianyu@haproxy:~/wireguard$ sudo cat /etc/netplan/00-enps06-config.yaml
```

```yaml
network:
  version: 2
  ethernets:
    enp0s6:
      dhcp4: false
      addresses:
        - 10.37.129.24/24
      nameservers:
        addresses:
          - 127.0.0.53
      optional: true
      routes:
        - to: default
          via: 10.37.129.26
          metric: 100
```
