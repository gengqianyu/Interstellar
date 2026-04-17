# DNS 域名系统详解

## 前言

现在，你已经知道机器是如何获取其 IP 地址的。你还知道，连接到互联网的所有机器（电脑、服务器、路由器等）都有 IP 地址。正是 IP 地址允许机器相互通信。

但是，IP 地址也给我们带来了一些问题。我们可能有很好的记忆力，但大脑很难记住 `13.250.177.223` 这样的一系列无规则的数字。我们宁愿记住 `www.github.com` 这样的名称。

我们之前说过，`13.250.177.223` 这样的 IP 地址格式是 IPv4 的。现在 IPv4 的 IP 地址已经用完了，以后越来越多的 IP 地址将类似下面这样的 IPv6 格式：

```
1504:02c9:41a4:85d3:0000:0000:a217:bca6
```

对于人类来说，要记住全是数字或字母的一长串 IP 地址谈何容易。

因此，这不是一个技术问题，而是一个**命名问题**。因为互联网配合 IP 地址已经能很好地运转了，但是我们这些"可怜"的人类没有机器那么强大的记忆力。为此，人们发明了一种系统，用于简化对互联网的访问，这就是 **DNS 系统**。

---

## DNS 协议

DNS 是 **Domain Name System（域名系统）** 的缩写。

DNS 简单来说就是将域名和 IP 地址相互映射的一个**分布式数据库**，能够使人们更方便地访问互联网。DNS 的作用非常简单，就是根据域名查出 IP 地址。你可以把它想象成一个巨大的电话本。

举例来说，你要访问域名 `math.stackexchange.com`，首先要通过 DNS 查出它的 IP 地址是 `151.101.129.69`。

---

## 查询过程

虽然只需要返回一个 IP 地址，但是 DNS 的查询过程非常复杂，分成多个步骤。

在 Linux 系统中可以使用 `dig` 命令查询 DNS 的整个过程：

```bash
dig math.stackexchange.com
```

**第一段：查询参数和统计**

```
; <<>> DiG 9.16.1-Ubuntu <<>> math.stackexchange.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 54356
;; flags: qr rd ad; QUERY: 1, ANSWER: 4, AUTHORITY: 0, ADDITIONAL: 0
;; WARNING: recursion requested but not available
```

**第二段：查询内容**

```
;; QUESTION SECTION:
;math.stackexchange.com.                IN      A
```

上面结果表示，查询域名 `math.stackexchange.com` 的 **A 记录**，A 是 Address 的缩写，也就是说查询这个域名的 IP 地址。

**第三段：DNS 服务器的答复**

```
;; ANSWER SECTION:
math.stackexchange.com. 0       IN      A       151.101.65.69
math.stackexchange.com. 0       IN      A       151.101.193.69
math.stackexchange.com. 0       IN      A       151.101.129.69
math.stackexchange.com. 0       IN      A       151.101.1.69
```

上面结果显示，`math.stackexchange.com` 有 4 个 A 记录，即 4 个 IP 地址。`0` 是 TTL 值（Time to Live 的缩写），表示缓存时间，即 0 秒之内不用重新查询，也就是不缓存。

**第四段：DNS 服务器的传输信息**

```
;; Query time: 0 msec
;; SERVER: 172.17.88.65#53(172.17.88.65)
;; WHEN: Fri Feb 26 15:29:59 CST 2021
;; MSG SIZE  rcvd: 126
```

上面结果显示，本机的 DNS 服务器是 IP 为 `172.17.88.65` 的主机，服务端口是 `53`（DNS 服务器的默认端口），以及回应长度是 126 字节。

---

## DNS 服务器

下面我们根据前面这个例子，一步步还原，本机到底怎么得到域名 `math.stackexchange.com` 的 IP 地址。

首先，本机一定要知道 DNS 服务器的 IP 地址，否则上不了网。通过 DNS 服务器，才能知道某个域名的 IP 地址到底是什么。

DNS 服务器的 IP 地址，有可能是**动态分配**给本地主机的，每次上网时由网关分配，这叫做 **DHCP（动态主机配置协议）**；也有可能是人为指定的**固定地址**。

Linux 系统里面，DNS 服务器的 IP 地址保存在 `/etc/resolv.conf` 文件：

```bash
$ cat /etc/resolv.conf
nameserver 172.17.88.65
```

查看结果和使用 `dig` 命令最后给出的 DNS 服务器的 IP 地址一致。

上例的 DNS 服务器是 `172.17.88.65`，这是一个内网地址。有一些公网的 DNS 服务器也可以使用，其中最有名的就是 Google 的 `8.8.8.8` 和 Level 3 的 `4.2.2.2`。

本机只向自己的 DNS 服务器查询，`dig` 命令有一个 `@` 参数，可以指定本机 DNS 查询服务器地址：

```bash
dig @8.8.8.8 math.stackexchange.com
```

---

## 域名层级

当你要访问 GitHub 时，DNS 系统负责将所请求的域名转换为 IP 地址（这被称为"**域名解析**"）。

DNS 服务器怎么会知道每个域名的 IP 地址呢？答案是**分级查询**。

请仔细看前面的例子，查询请求段的域名的尾部都多了一个点：

```
;; QUESTION SECTION:
;math.stackexchange.com.                IN      A
```

域名 `math.stackexchange.com` 显示为 `math.stackexchange.com.`。这不是疏忽，而是所有域名的尾部实际上都有一个**根域名**。这个根域名 `.`（点），对于所有域名都是一样的，因此平常是省略的。但是当我们配置自己的 DNS 服务器时，你会发现这个点（`.`）变得非常重要。

根域名的下一级叫做 **TLD（Top Level Domain）顶级域名**，例如 `.com`、`.cn`。

### 国家顶级域名

| 域名 | 国家/地区 |
|------|----------|
| `.cn` | 中国大陆 |
| `.us` | 美国 |
| `.jp` | 日本 |
| `.uk` | 英国 |
| `.de` | 德国 |
| `.eu` | 欧盟 |

### 通用顶级域名

| 域名 | 用途 |
|------|------|
| `.com` | 商业机构 |
| `.org` | 无限制 |
| `.net` | 网络服务供应商 |
| `.edu` | 教育机构 |
| `.biz` | 商业使用 |

再下一级叫做**二级域名**，又称 **SLD（Second Level Domain）次级域名**。比如 `www.baidu.com` 里面的 `.baidu`。这一级域名用户是可以注册的。`.baidu` 是 `.com` 的子域，`cn` 顶级域包含以 `.cn` 结尾的所有子域。

再下一级，在大部分的域名中叫做**主机名（host）**。最常见最著名的 `www` 代表此域名代表着一个 Web 服务。也就是说 `www` 是 `google.cn` 域下一台 Web 服务器的名字。

> Internet 中的主要服务有万维网（WWW）、文件传输（FTP）、电子邮件（E-mail）、远程登录（Telnet）等。

三级域名下面也可以再分支出四级域名，四级域名下还可以有五级域名，以此类推。

总结一下，一般的域名层级如下：

```
host.sld.tld.root
```

展开的结构就像一棵倒置的树：

```
                              ┌───────┐
                              │   .   │  根域
                              └───┬───┘
                                  │
                ┌─────────────────┼─────────────────┐
                │                 │                 │
                ▼                 ▼                 ▼
           ┌─────────┐      ┌─────────┐      ┌─────────┐
           │  .com   │      │  .net   │      │  .cn    │  顶级域
           └────┬────┘      └─────────┘      └────┬────┘
                │                                  │
         ┌──────┴──────┐                    ┌──────┴──────┐
         │             │                    │             │
         ▼             ▼                    ▼             ▼
    ┌─────────┐   ┌─────────┐         ┌─────────┐   ┌─────────┐
    │ .google │   │ .github │         │  .com   │   │  .edu   │  二级域
    └─────────┘   └─────────┘         └────┬────┘   └────┬────┘
                                           │             │
                                           ▼             ▼
                                      ┌─────────┐  ┌─────────┐
                                      │ .baidu  │  │  .sina  │  子域
                                      └────┬────┘  └────┬────┘
                                           │            │
                                           ▼            ▼
                                      ┌─────────┐  ┌─────────┐
                                      │   www   │  │  mail   │  主机/服务
                                      └─────────┘  └─────────┘
```

最多可以分支 **127 层**，每层最多由 **63 个字符**组成，中间用点（`.`）分割。域名长度不能超过 **255 个字符**，不区分大小写。因此，你可以拥有类似这样的域名：`www.cn.7.new.super.google.cn`。

每个部分被称为**标签**，所有标签组合在一起构成一个 **FQDN（Fully Qualified Domain Name）完全限定域名**。一个 FQDN 是唯一的。按照惯例，FQDN 以点（`.`）结尾，因为在 TLD（顶级域名）上方树的最顶部是 DNS 根域，用一个 `.` 表示。

在 DNS 级别上，`www.google.cn` 不能算是严格意义上的 FQDN，因为它漏写了最后的一个点（`.`）。互联网上的任何 FQDN 都必须以点结尾，例如 `www.github.com.` 就是 FQDN，因为我们确定上面没有更高层的域（到点为止）。

---

## 域名服务器

DNS 服务器根据域名的层级，进行**分级查询**。

需要明确的是，每一级域名都有自己的 **NS（Name Server）记录**（各级域名对应各自 DNS 服务器的域名），NS 记录指向该域名的域名服务器。这些服务器知道下一级域名的各种记录。

所谓分级查询，就是从根域名开始，依次查询每一级域名的 NS 记录，直到查到最终的 IP 地址：

1. 从 **根域名服务器** 查到 **顶级域名服务器** 的 NS 记录和 A 记录（IP 地址）
2. 从 **顶级域名服务器** 查到 **次级域名服务器** 的 NS 记录和 A 记录
3. 从 **次级域名服务器** 查出 **主机名** 的 IP 地址

> 根域名服务器的 NS 记录和 IP 地址一般是不变的，因此是内置在 DNS 服务器里面的。你会常常看到电脑的 DNS 设置为 `8.8.8.8`，这就是一台根域名服务器，代表 `.` 这个 NS 记录。

---

## 分级查询实例

域名解析（Domain Name Resolution）就是从域名得到 IP 地址的转换过程。域名的解析工作由 DNS 服务器完成。

假设你已经连接网络，DHCP 服务器为你分配了 IP 地址、子网掩码、默认网关以及 DNS 服务器。

想象一下，你在浏览器中输入 `www.github.com`，你的电脑就会开始将其解析为 IP 地址。因此，你将向从 DHCP 收到的 DNS 服务器请求解析。DNS 服务器有两种方法可以为你提供答案：

1. **它自己知道答案**
2. **它不知道答案，必须向另外一台服务器请求答案**

在大多数情况下，你的 DNS 服务器所知不多，需要请求其他服务器为其提供答案。实际上，每个 DNS 服务器都负责一个域或少量域，域名解析的宗旨就是在正确的服务器上获取正确的信息。

`dig` 命令的 `+trace` 参数可以显示 DNS 的整个分级查询过程：

```bash
dig +trace math.stackexchange.com
```

### 第一步：查询根域名服务器

列出根域名 `.` 的所有 NS 记录，即所有根域名 DNS 服务器：

```
; <<>> DiG 9.16.1-Ubuntu <<>> +trace math.stackexchange.com
;; global options: +cmd
列出顶级域名 `.` 的所有 NS 记录：
.                       36213   IN      NS      j.root-servers.net.
.                       36213   IN      NS      d.root-servers.net.
.                       36213   IN      NS      k.root-servers.net.
.                       36213   IN      NS      l.root-servers.net.
.                       36213   IN      NS      h.root-servers.net.
.                       36213   IN      NS      g.root-servers.net.
.                       36213   IN      NS      b.root-servers.net.
.                       36213   IN      NS      f.root-servers.net.
.                       36213   IN      NS      i.root-servers.net.
.                       36213   IN      NS      c.root-servers.net.
.                       36213   IN      NS      a.root-servers.net.
.                       36213   IN      NS      m.root-servers.net.
.                       36213   IN      NS      e.root-servers.net.
;; Received 228 bytes from 172.17.88.65#53(172.17.88.65) in 840 ms
```

本地 DNS 服务器根据内置的根域名 DNS 服务器的 IP 地址，向所有根域名 DNS 服务器发出查询请求，询问 `math.stackexchange.com` 的顶级域名服务器 `com.` 的 NS 记录。最先回复的根域名 DNS 服务器会被缓存到本地，以后只向这台服务器发请求。

### 第二步：查询顶级域名服务器

列出顶级域名 `com.` 的所有 NS 记录：

```
com.                    172800  IN      NS      a.gtld-servers.net.
com.                    172800  IN      NS      f.gtld-servers.net.
com.                    172800  IN      NS      c.gtld-servers.net.
com.                    172800  IN      NS      d.gtld-servers.net.
com.                    172800  IN      NS      l.gtld-servers.net.
com.                    172800  IN      NS      j.gtld-servers.net.
com.                    172800  IN      NS      g.gtld-servers.net.
com.                    172800  IN      NS      e.gtld-servers.net.
com.                    172800  IN      NS      i.gtld-servers.net.
com.                    172800  IN      NS      b.gtld-servers.net.
com.                    172800  IN      NS      h.gtld-servers.net.
com.                    172800  IN      NS      k.gtld-servers.net.
com.                    172800  IN      NS      m.gtld-servers.net.
;; Received 1213 bytes from 192.36.148.17#53(i.root-servers.net) in 70 ms
```

`.com` 域名的 13 条 NS 记录，同时返回的还有每一条记录对应的 IP 地址。

### 第三步：查询次级域名服务器

本地 DNS 服务器向顶级域名服务器发起查询请求，询问 `stackexchange.com.` 的 NS 记录：

```
stackexchange.com.      172800  IN      NS      ns-925.awsdns-51.net.
stackexchange.com.      172800  IN      NS      ns-1832.awsdns-37.co.uk.
stackexchange.com.      172800  IN      NS      ns-cloud-d1.googledomains.com.
stackexchange.com.      172800  IN      NS      ns-cloud-d2.googledomains.com.
;; Received 825 bytes from 192.42.93.30#53(g.gtld-servers.net) in 230 ms
```

上面结果显示，`stackexchange.com` 域名有 4 条 NS 记录，同时返回每一条 NS 记录对应的 IP 地址。

### 第四步：查询主机名的 IP 地址

DNS 服务器向上面这四台 NS 服务器查询 `math.stackexchange.com` 的主机名：

```
math.stackexchange.com. 3600    IN      A       151.101.193.69
math.stackexchange.com. 3600    IN      A       151.101.65.69
math.stackexchange.com. 3600    IN      A       151.101.1.69
math.stackexchange.com. 3600    IN      A       151.101.129.69
stackexchange.com.      172800  IN      NS      ns-1832.awsdns-37.co.uk.
stackexchange.com.      172800  IN      NS      ns-925.awsdns-51.net.
stackexchange.com.      172800  IN      NS      ns-cloud-d1.googledomains.com.
stackexchange.com.      172800  IN      NS      ns-cloud-d2.googledomains.com.
;; Received 252 bytes from 205.251.199.40#53(ns-1832.awsdns-37.co.uk) in 130 ms
```

结果显示，`math.stackexchange.com` 有 4 条 A 记录，即这 4 个 IP 地址都可以访问到网站。最先返回结果的上级 NS 服务器是 `ns-1832.awsdns-37.co.uk`，IP 地址是 `205.251.199.40`，端口 `53`。

`dig` 命令可以单独查看每一级域名的 NS 记录：

```bash
dig ns com
dig ns stackexchange.com
```

---

## DNS 解析过程

下面用一个完整的流程图展示 DNS 解析 `www.163.com` 的全过程：

```
                    ② 缓存未命中                          ④ 查询次级域名
                    查询根服务器                           服务器 .163.com
                ┌──────────────────┐                 ┌──────────────────┐
                │                  │                 │                  │
                ▼                  │                 ▼                  │
  ┌──────────────────────┐         │   ┌──────────────────────┐         │
  │  DNS 根(.)域服务器     │         │   │  .com 域服务器        │         │
  │                      │         │   │                      │         │
  │  a.root-server.net   │         │   │  a.gtld-servers.net  │         │
  │  b.root-server.net   │         │   │  b.gtld-servers.net  │         │
  │  c.root-server.net   │         │   │  c.gtld-servers.net  │         │
  │  ...                 │         │   │  ...                 │         │
  └──────────┬───────────┘         │   └──────────┬───────────┘         │
             │                     │              │                     │
             │ ③ 返回 .com 域      │              │ ⑤ 返回 .163.com 域  │
             │   服务器地址          │              │   服务器地址         │
             ▼                     │              ▼                     │
  ┌────────────────────────────────┴──────────────────────────────────────┐
  │                          本地 DNS 服务器                               │
  └──────────┬─────────────────────────────────────────────┬─────────────┘
             │                                             │
        ① 请求解析                                    ⑥ 查询主机名
        www.163.com                                   www.163.com
        的 IP 地址                                    的 IP 地址
             │                                             │
             │                                             ▼
             │                                  ┌──────────────────────┐
             │                                  │  .163.com 域服务器    │
             │         ⑧ 返回结果                │                      │
             │         IP: 1.1.1.1              │  www.163.com         │
             │         (同时写入缓存)             │  mail.163.com        │
             │                                  └──────────┬───────────┘
             ▼                                             │
  ┌──────────────────┐                    ⑦ 返回 IP 地址    │
  │    网络客户端     │ ◄──────────────────  1.1.1.1  ◄─────┘
  └──────────────────┘
```

**解析流程说明：**

| 步骤 | 说明 |
|------|------|
| ① | 客户端向本地 DNS 服务器发起请求：`www.163.com` 的 IP 地址是多少？ |
| ② | 本地 DNS 缓存中没有该记录，向根域名服务器查询 |
| ③ | 根服务器返回：该域名由 `.com` 区域管理，给出 `.com` 域服务器地址 |
| ④ | 本地 DNS 向 `.com` 域服务器查询 |
| ⑤ | `.com` 域服务器返回：负责 `163.com` 的服务器地址 |
| ⑥ | 本地 DNS 向 `.163.com` 域服务器查询主机名 |
| ⑦ | `.163.com` 域服务器返回 `www.163.com` 对应的 IP 地址 `1.1.1.1` |
| ⑧ | 本地 DNS 将结果返回给客户端，同时写入缓存以备后续查询 |

---

## 授权 DNS 与缓存

那些不需要向其他服务器请求信息，就能提供域名解析的服务器称为**授权 DNS 服务器**（又称权威 DNS 服务器）。

DNS 服务器也可以使用**缓存系统**，因此它们不必重复请求信息，可以将信息缓存起来。但是它们不能作为授权服务器，因为存储在缓存中的信息在一段时间后可能不再有效。

那么，是否存在用于将 IP 地址转换为域名的协议呢？答案是不需要额外的协议。DNS 协议本身就知道如何做到这一点，这被称为**反向 DNS（Reverse DNS）** 和**反向解析（Reverse Resolution）**。但是出于安全方面的原因，反向 DNS 很少被使用。

即使 DNS 系统对于互联网的运作不是必需的，它也是必不可少的元素。域名系统由一个名为 **ICANN** 的非营利性组织管理。ICANN 是 Internet Corporation for Assigned Names and Numbers（互联网名称与数字地址分配机构）。

ICANN 负责管理 13 台 DNS 根服务器，这 13 台服务器管理 DNS 的根。它们知道管理 TLD（`.cn`、`.us`、`.com`、`.org` 等）的 DNS 服务器的 IP 地址。

**那么，真的只有 13 台根服务器吗？**

是，也不是。实际上，在根服务器被多次攻击之后，我们意识到只有 13 台根服务器的脆弱性以及这可能对互联网运行构成的威胁。因此建立了一个复制系统，可以复制 13 台根服务器的数据。实际上今天有数百个甚至更多的根服务器，它们从 13 个原始根服务器中复制信息。

---

## DNS 的记录类型

域名与 IP 之间的对应关系，称为**记录（Record）**。根据使用场景，记录可以分成不同的类型（type），前面已经看到了 A 记录和 NS 记录。

常见的 DNS 记录类型如下：

| 类型 | 全称 | 说明 |
|------|------|------|
| **A** | Address | 地址记录，返回域名指向的 IP 地址 |
| **NS** | Name Server | 域名服务器记录，返回保存下一级域名信息的服务器地址。该记录只能设置为域名，不能设置为 IP 地址 |
| **MX** | Mail Exchange | 邮件记录，返回接收电子邮件的服务器地址 |
| **CNAME** | Canonical Name | 规范名称记录，返回另一个域名，即当前查询的域名是一个域名的跳转 |
| **PTR** | Pointer Record | 逆向查询记录，只用于从 IP 地址查询域名 |

一般来说，为了服务的安全可靠，至少应该有两条 NS 记录，而 A 记录和 MX 记录也可以有多条，这样就提供了服务的冗余性，防止出现单点故障。

### CNAME 记录示例

CNAME 记录主要用于域名的内部跳转，为服务器配置提供灵活性，用户感知不到。举例来说，`www.lagou.com` 这个域名就是一个 CNAME 记录：

```bash
dig www.lagou.com
```

```
;; QUESTION SECTION:
;www.lagou.com.                 IN      A

;; ANSWER SECTION:
www.lagou.com.          0       IN      CNAME   lgmain.cdn.lagou.com.
lgmain.cdn.lagou.com.   0       IN      A       117.50.36.103
```

结果显示，`www.lagou.com` 的 CNAME 记录指向 `lgmain.cdn.lagou.com`，也就是说，用户查询 `www.lagou.com` 的时候，实际返回的是 `lgmain.cdn.lagou.com` 主机的 IP 地址。

这样的好处是，变更服务器 IP 地址的时候，只要修改 `lgmain.cdn.lagou.com` 这个域名就可以了，用户的 `www.lagou.com` 域名不用修改。

> **注意：** 由于 CNAME 记录就是一个替换，所以一个域名一旦设置 CNAME 记录后，就不能再设置其他记录了（比如 A 和 MX 记录），这是为了防止产生冲突。

### PTR 记录示例

PTR 记录用于从 IP 地址反查域名。`dig` 命令的 `-x` 参数用于查询 PTR 记录：

```bash
dig -x 127.0.0.1
```

```
;; QUESTION SECTION:
;1.0.0.127.in-addr.arpa.                IN      PTR

;; ANSWER SECTION:
1.0.0.127.in-addr.arpa. 0       IN      PTR     kubernetes.docker.internal.
```

上面结果显示 `127.0.0.1` 这台本地服务器的域名是 `kubernetes.docker.internal`。

逆向查询的一个应用，可以防止垃圾邮件，即验证发送邮件的 IP 地址，是否真的有它所声称的域名。

`dig` 命令可以查看指定的记录类型：

```bash
dig a github.com
dig ns github.com
dig mx github.com
```

---

## 其他 DNS 工具

### host 命令

`host` 命令可以看作 `dig` 命令的简化版本，返回当前请求域名的各种记录：

```bash
$ host github.com
github.com has address 52.74.223.119
github.com mail is handled by 5 alt1.aspmx.l.google.com.
github.com mail is handled by 10 alt4.aspmx.l.google.com.
github.com mail is handled by 5 alt2.aspmx.l.google.com.
github.com mail is handled by 10 alt3.aspmx.l.google.com.
github.com mail is handled by 1 aspmx.l.google.com.
```

`host` 也可以用作逆向查询，即从 IP 地址查询域名：

```bash
$ host 192.30.252.153
153.252.30.192.in-addr.arpa domain name pointer lb-192-30-252-153-iad.github.com.
```

### whois 命令

`whois` 命令用来查看域名的注册情况：

```bash
$ whois github.com
Domain Name: GITHUB.COM
Registry Domain ID: 1264983250_DOMAIN_COM-VRSN
Registrar WHOIS Server: whois.markmonitor.com
Updated Date: 2020-09-08T09:18:27Z
Creation Date: 2007-10-09T18:20:50Z
Registry Expiry Date: 2022-10-09T18:20:50Z
Registrar: MarkMonitor Inc.
...
Name Server: DNS1.P08.NSONE.NET
Name Server: DNS2.P08.NSONE.NET
Name Server: DNS3.P08.NSONE.NET
Name Server: DNS4.P08.NSONE.NET
Name Server: NS-1283.AWSDNS-32.ORG
Name Server: NS-1707.AWSDNS-21.CO.UK
Name Server: NS-421.AWSDNS-52.COM
Name Server: NS-520.AWSDNS-01.NET
```

---

## 域名劫持

如果你管理着一个网络，并且拥有域名 `mydomain.com`，则可以将一台域名为 `www.google.cn.mydomain.com` 的机器添加到 DNS 服务器。

`www.google.cn` 域名被劫持的前提是：**有人使用你的 DNS 服务器请求 `www.google.cn`，却忘了在结尾处加上点（`.`）**，那么就会转到 `www.google.cn.mydomain.com` 这台主机。

在 DNS 服务的体系结构中，每个标签负责直接在下面的级别。例如：

- 根（`.`）负责域名 `.com`
- `.com` 负责域名 `google.com`
- `google.com` 负责域名 `www.google.com`

以此类推。

当然，Google 公司希望自己管理 `google.com` 域。因此，管理 `.com` 域的组织将此域名的管理**委派**给 Google 公司。每个想要在互联网上拥有一个域名的人都需要购买域名，但是购买了域名后必须管理 DNS 服务器以发布其地址。

但是，大多数出售域名的公司（**域名注册商，Domain Name Registrar**，国外比较有名的有 Namesilo、GoDaddy、Namecheap）都会主动提供管理 DNS 记录的功能。

因此，我们知道 DNS 是以大型树形结构的形式组织的，并且树形结构的每个部分都可以由拥有它的人来管理。
