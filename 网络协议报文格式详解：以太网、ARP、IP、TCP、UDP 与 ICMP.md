# 网络协议报文格式详解：以太网、ARP、IP、TCP、UDP 与 ICMP

## 一、先看协议之间的关系

本文讨论的是常见的**以太网承载 IP** 的场景。这里的“MAC”主要指以太网帧中的 MAC 地址和介质访问控制层格式，而不是一个与 IP、TCP 并列的单独报文协议。所有多字节整数字段在网络上传输时通常采用**网络字节序（大端）**；位图中的字段长度以 bit 为单位。

```text
应用数据
  ├─ TCP 段 / UDP 数据报 / ICMP 报文
  │     └─ IPv4 数据报 / IPv6 包
  │            └─ 以太网帧（目标 MAC、源 MAC、EtherType、负载、FCS）
  └─ ARP 报文 ── 直接放入以太网帧（用于 IPv4 地址解析）
```

|层次|常见标识|本层主要解决的问题|地址或标识的作用范围|
|---|---|---|---|
|以太网 / MAC|目标 MAC、源 MAC、EtherType|同一二层网络中的帧交付|MAC 地址用于当前链路上的下一跳|
|网络层|源/目标 IP、IPv4 Protocol、IPv6 Next Header|跨网络寻址与转发|IP 地址标识网络层端点|
|传输层|TCP/UDP 端口|将数据交给主机上的应用进程|端口区分服务和会话|
|控制报文|ICMP 类型与代码|报告网络错误、执行诊断与控制|ICMP 直接由 IP 承载，没有 TCP/UDP 端口|

路由器转发时，**IP 目标地址通常保持不变，以太网的源/目标 MAC 会按每一跳重新填写**；IPv4 TTL 或 IPv6 Hop Limit 会递减。NAT 等设备可能改写 IP 地址、端口和校验和，因此“通常”不是“永远”。

---

## 二、以太网帧与 MAC 地址

### 2.1 Ethernet II 帧格式

```text
线上发送顺序：前导码 7B | SFD 1B | 目标 MAC 6B | 源 MAC 6B |
             EtherType 2B | 负载 46~1500B | FCS 4B
```

常说的“以太网帧长 64～1518 字节”从**目标 MAC** 算到 **FCS**，不包含前导码、SFD 和帧间间隔；这是未加 VLAN 标签的普通 Ethernet II 帧范围。负载不足 46 字节时需要填充。物理层的前导码与 SFD 常由网卡处理，抓包通常看不到；FCS 也经常被网卡剥离，因此抓包文件不一定包含它。

|字段|长度|作用|
|---|---:|---|
|前导码（Preamble）|7 B|帮助接收端进行位同步，通常为交替的 0/1 位|
|帧起始定界符（SFD）|1 B|标记随后进入帧头|
|目标 MAC|6 B|本链路上的接收者；可以是单播、组播或广播地址|
|源 MAC|6 B|本链路上的发送者；通常是单播地址|
|EtherType|2 B|指示负载协议，例如 `0x0800` 为 IPv4，`0x0806` 为 ARP，`0x86DD` 为 IPv6|
|负载与填充|46～1500 B|承载上层报文；短帧用填充字节满足最小帧长|
|FCS|4 B|基于 CRC-32 检测传输错误；用于检错，不提供纠错或加密|

MAC 地址为 48 bit，通常写作 `00:1A:2B:3C:4D:5E`。`FF:FF:FF:FF:FF:FF` 是广播地址。目标 MAC 是**下一跳**的链路层地址：跨网段发送时通常填写默认网关的 MAC，而不是远端服务器的 MAC。

### 2.2 802.1Q VLAN 标签

带 802.1Q 标签的帧在**源 MAC 与原 EtherType 之间**插入 4 字节：

```text
目标 MAC 6B | 源 MAC 6B | TPID 2B | TCI 2B | 原 EtherType 2B | 负载 | FCS 4B
                                  TCI = PCP 3 bit | DEI 1 bit | VID 12 bit
```

|字段|长度|作用|
|---|---:|---|
|TPID|16 bit|通常为 `0x8100`，表明后面是 VLAN 标签|
|PCP|3 bit|优先级标记，供二层服务质量机制使用|
|DEI|1 bit|丢弃资格标记，供拥塞处理使用|
|VID|12 bit|VLAN 标识；`0` 表示仅携带优先级信息，`4095` 保留|

单个 VLAN 标签使常见的最大帧长由 1518 B 变为 **1522 B**（均含 FCS）。VLAN 标签不改变 IP 或 TCP/UDP 头的内部结构。

---

## 三、ARP：把 IPv4 下一跳地址解析为 MAC

ARP 用于以太网等链路上的 **IPv4 地址到链路层地址**的解析。ARP 自身不封装在 IP 中；Ethernet II 的 EtherType 为 `0x0806`。下表给出常见的“以太网 + IPv4”ARP 报文，报文本体为 28 B。

```text
硬件类型 2B | 协议类型 2B | 硬件地址长度 1B | 协议地址长度 1B |
操作码 2B   | 发送方 MAC 6B | 发送方 IPv4 4B |
目标 MAC 6B | 目标 IPv4 4B
```

|字段|长度|作用|
|---|---:|---|
|硬件类型（HTYPE）|16 bit|`1` 表示以太网|
|协议类型（PTYPE）|16 bit|`0x0800` 表示 IPv4|
|硬件地址长度（HLEN）|8 bit|以太网 MAC 地址长为 `6` B|
|协议地址长度（PLEN）|8 bit|IPv4 地址长为 `4` B|
|操作码（OPER）|16 bit|`1` 为请求，`2` 为应答|
|发送方 MAC（SHA）|48 bit|发送方的链路层地址|
|发送方 IPv4（SPA）|32 bit|发送方的 IPv4 地址|
|目标 MAC（THA）|48 bit|目标的链路层地址；请求中通常填 `00:00:00:00:00:00`|
|目标 IPv4（TPA）|32 bit|待解析的 IPv4 地址|

例如主机要访问另一网段的服务器，会先查路由表选出**默认网关的 IPv4 地址**，再用 ARP 查询网关的 MAC。ARP 请求一般用以太网广播发送，知道映射的设备通常以单播应答。IPv6 不使用 ARP，而使用基于 ICMPv6 的邻居发现（NDP）。

---

## 四、IPv4：网络层数据报

### 4.1 固定部分与可选部分

IPv4 首部最少 **20 B**，最多 **60 B**。下图每行 32 bit：

```text
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-------+-------+---------------+-----------------------------------+
|版本 4b|IHL  4b |DSCP 6b|ECN 2b|总长度 16b                          |
+-------------------------------+-----------------------------------+
|标识 16b                      |标志 3b|片偏移 13b                   |
+-------------------------------+-----------------------------------+
|TTL 8b         |协议 8b        |首部校验和 16b                     |
+-------------------------------------------------------------------+
|源 IPv4 地址 32b                                                   |
+-------------------------------------------------------------------+
|目标 IPv4 地址 32b                                                 |
+-------------------------------------------------------------------+
|选项与填充（0～40B，可选）                                         |
+-------------------------------------------------------------------+
|数据负载：TCP / UDP / ICMP 等                                      |
```

|字段|长度|作用|
|---|---:|---|
|Version|4 bit|值为 `4`，标识 IPv4|
|IHL|4 bit|首部长度，单位为 **4 B**；值 `5` 表示 20 B|
|DSCP / ECN|6 / 2 bit|分别用于差分服务流量分类、显式拥塞通知|
|Total Length|16 bit|**整个 IPv4 数据报**的长度，包含 IPv4 首部和负载；最大值 65535 B|
|Identification|16 bit|供分片及重组时识别同一原始数据报的各片|
|Flags|3 bit|保留位、DF（不允许分片）、MF（后面还有分片）|
|Fragment Offset|13 bit|本片数据在原始负载中的偏移，单位为 **8 B**|
|TTL|8 bit|每经过一次路由转发至少减 1；减至 0 时丢弃，可触发 ICMP 超时通知|
|Protocol|8 bit|指示负载协议：`1` ICMPv4、`6` TCP、`17` UDP|
|Header Checksum|16 bit|仅校验 **IPv4 首部**；转发时因 TTL 变化需要更新|
|源 / 目标地址|各 32 bit|网络层发送者和接收者地址|
|Options / Padding|0～40 B|可选功能；填充使首部长度成为 4 B 的整数倍|

**分片示例：**假设原始 IPv4 负载为 2000 B，路径 MTU 为 1500 B，IPv4 首部为 20 B，允许分片。前一片最多装 1480 B 数据（也是 8 B 的整数倍），`MF=1`、片偏移为 `0`；后一片装 520 B，`MF=0`、片偏移为 `1480/8=185`。两片共用同一个 Identification。分片的重组通常由最终目的主机完成。

### 4.2 MTU 与实际长度

以太网常见 MTU 为 **1500 B**，指以太网负载中可承载的最大网络层报文长度，不包含以太网头和 FCS。若 IPv4 首部 20 B、TCP 首部 20 B，且没有其他选项，单个 1500 B IP 包中最多放 1460 B TCP 数据。这是典型 MSS 的来源；实际路径 MTU、IPv4/TCP 选项、隧道封装等都会改变可用空间。

---

## 五、IPv6：固定首部与扩展首部

IPv6 固定首部为 **40 B**，没有 IPv4 的首部校验和字段；扩展首部根据需要串接。

```text
|Version 4b|Traffic Class 8b|Flow Label 20b             |
|Payload Length 16b          |Next Header 8b|Hop Limit 8b|
|源 IPv6 地址 128b                                      |
|目标 IPv6 地址 128b                                    |
|扩展首部（可选）| TCP / UDP / ICMPv6 等负载          |
```

|字段|长度|作用|
|---|---:|---|
|Version|4 bit|值为 `6`|
|Traffic Class|8 bit|流量分类与拥塞通知，语义与 DSCP/ECN 相关|
|Flow Label|20 bit|标识需要一致处理的报文流|
|Payload Length|16 bit|固定首部**之后**的字节数，通常包括扩展首部和上层负载；超大负载有专门扩展机制|
|Next Header|8 bit|指示紧随固定首部的扩展首部或上层协议；常见值 `6` TCP、`17` UDP、`58` ICMPv6|
|Hop Limit|8 bit|功能类似 IPv4 TTL，逐跳递减|
|源 / 目标地址|各 128 bit|IPv6 地址|

如果存在扩展首部，固定首部的 Next Header 先指向**第一个扩展首部**，再由各扩展首部的 Next Header 字段依次指向下一项，最后才到 TCP、UDP 或 ICMPv6。IPv6 路由器**不对过大的包进行普通分片**；源主机可使用 Fragment 扩展首部进行分片，并依靠路径 MTU 发现调整报文大小。IPv6 邻居发现依赖 ICMPv6，不能简单把 ICMPv6 当作可一律屏蔽的“ping 协议”。

---

## 六、TCP：可靠的字节流传输

### 6.1 TCP 首部格式

TCP 首部最少 **20 B**，常规首部长度字段最多可表达 **60 B**。

```text
|源端口 16b                         |目标端口 16b                    |
|序列号 Sequence Number 32b                                         |
|确认号 Acknowledgment Number 32b                                   |
|Data Offset 4b|保留 3b|NS 1b|CWR ECE URG ACK PSH RST SYN FIN 各 1b|
|窗口大小 Window 16b              |校验和 Checksum 16b             |
|紧急指针 Urgent Pointer 16b      |选项与填充（可选）              |
|数据负载                                                         |
```

|字段|长度|作用|
|---|---:|---|
|源 / 目标端口|各 16 bit|标识通信两端的应用端点|
|Sequence Number|32 bit|本段首个数据字节的序号；SYN/FIN 各占用一个序号|
|Acknowledgment Number|32 bit|当 ACK=1 时，表示**下一字节期望收到的序号**，属于累计确认|
|Data Offset|4 bit|TCP 首部长度，单位为 **4 B**；值 `5` 表示 20 B|
|保留位|3 bit|通常置零，留作协议扩展|
|控制位|9 bit|NS、CWR、ECE、URG、ACK、PSH、RST、SYN、FIN；具体作用见下表|
|Window|16 bit|接收方通告的可接收窗口；启用 Window Scale 选项后可扩大有效窗口|
|Checksum|16 bit|覆盖 TCP 首部、数据和 IP **伪首部**；TCP 校验和必须计算|
|Urgent Pointer|16 bit|URG=1 时与紧急数据机制相关；现代应用很少依赖|
|Options / Padding|0～40 B|MSS、Window Scale、SACK Permitted/SACK、Timestamp 等；填充到 4 B 边界|

|字段|长度|作用|
|---|---:|---|
|SYN|1 bit|建立连接时同步初始序列号|
|ACK|1 bit|确认号字段有效|
|FIN|1 bit|发送方已发送完数据，正常关闭该方向的字节流|
|RST|1 bit|复位连接或拒绝无效连接|
|PSH|1 bit|提示尽快将已收到的数据交给应用；并非消息边界标志|
|URG|1 bit|紧急指针有效|
|ECE|1 bit|参与显式拥塞通知 ECN 的协商和反馈|
|CWR|1 bit|发送方已响应拥塞通知的标记|
|NS|1 bit|ECN 相关标记，实际使用较少|

TCP 是**字节流**：一次应用层 `write` 不保证对应一个 TCP 段或接收端的一次 `read`。TCP 通过序列号、确认、重传、接收窗口和拥塞控制提供有序可靠交付，但不保留应用消息边界。TCP Checksum 所用的伪首部来自 IP 层，含源/目标 IP、协议号和 TCP 长度等信息；它参与计算，却不作为 TCP 首部字节实际传输。

### 6.2 三次握手与字段变化

```text
客户端                               服务端
SYN, Seq=x                   ───────>
                             <───────  SYN+ACK, Seq=y, Ack=x+1
ACK, Seq=x+1, Ack=y+1        ───────>
```

SYN 消耗一个序列号；此后传输的每个数据字节也各占一个序列号。例如从 `Seq=x+1` 发送 100 B 数据，接收端完整收到后可回 `Ack=x+101`。ACK 位本身不消耗序列号。

---

## 七、UDP：保留数据报边界的传输

UDP 首部固定 **8 B**，格式如下：

```text
|源端口 16b|目标端口 16b|
|长度 16b  |校验和 16b  |
|应用数据                 |
```

|字段|长度|作用|
|---|---:|---|
|源端口|16 bit|发送端口；特定场景可为 `0`，表示未使用|
|目标端口|16 bit|接收端口|
|Length|16 bit|**UDP 首部 + UDP 数据**的总长度，最小为 8 B|
|Checksum|16 bit|覆盖 UDP 首部、数据和 IP 伪首部；检测传输错误|

UDP 保留数据报边界，但协议本身不提供连接建立、有序交付、自动重传和拥塞控制。应用可在 UDP 之上自行实现所需机制。**IPv4 中 UDP 校验和为 0 表示未使用**；在普通 IPv6 UDP 通信中校验和必须使用（特定隧道场景另有例外）。以太网 FCS 与 UDP 校验和检查的范围和作用层次不同，不能互相替代。

---

## 八、ICMPv4 与 ICMPv6：错误报告和网络控制

ICMP 直接由 IP 承载，不依赖 TCP/UDP 端口。两者的通用开头都是 Type、Code、Checksum，但**不同 Type 后面的字段并不相同**。

```text
ICMPv4：|Type 8b|Code 8b|Checksum 16b|类型专有字段与数据...|
ICMPv6：|Type 8b|Code 8b|Checksum 16b|类型专有字段与数据...|
```

### 8.1 常见 ICMPv4 类型

IPv4 首部的 `Protocol=1` 表示 ICMPv4。

|字段|长度|作用|
|---|---:|---|
|Type|8 bit|确定报文大类，例如回显请求、目标不可达、超时|
|Code|8 bit|细分原因；取值含义依赖 Type|
|Checksum|16 bit|校验整个 ICMPv4 报文|
|类型专有字段与数据|可变|由 Type/Code 决定；错误消息通常携带原始报文的一部分，便于定位|

|Type|名称|常见用途|
|---:|---|---|
|0|Echo Reply|回应 ping|
|3|Destination Unreachable|目标不可达；不同 Code 可表示网络、主机、端口等不可达，或分片受限|
|8|Echo Request|发起 ping|
|11|Time Exceeded|TTL 耗尽或分片重组超时；traceroute 常利用 TTL 耗尽响应|
|12|Parameter Problem|IPv4 首部等参数有问题|

Echo Request/Reply 在前 4 B 通用头后还有以下字段：

|字段|长度|作用|
|---|---:|---|
|Identifier|16 bit|与 Sequence Number 一起匹配请求和应答|
|Sequence Number|16 bit|区分同一标识符下的不同探测报文|
|Data|可变|请求中的数据通常由应答原样返回|

`ping` 是使用 ICMP Echo 的诊断程序，**ICMP 本身不等于 ping**。

### 8.2 常见 ICMPv6 类型

IPv6 最终的 `Next Header=58` 表示 ICMPv6。

|字段|长度|作用|
|---|---:|---|
|Type|8 bit|确定报文大类，例如回显请求、目标不可达、邻居发现|
|Code|8 bit|细分原因；取值含义依赖 Type|
|Checksum|16 bit|校验 ICMPv6 报文，计算时还纳入 IPv6 伪首部|
|类型专有字段与数据|可变|由 Type/Code 决定；Echo、错误报告和邻居发现消息的结构各不相同|

|Type|名称|常见用途|
|---:|---|---|
|1|Destination Unreachable|报告目标不可达|
|2|Packet Too Big|报告可用 MTU；IPv6 路径 MTU 发现的重要组成部分|
|3|Time Exceeded|Hop Limit 耗尽等|
|4|Parameter Problem|IPv6 首部或扩展首部参数错误|
|128 / 129|Echo Request / Echo Reply|IPv6 ping|
|133 / 134|Router Solicitation / Advertisement|发现路由器、获取网络前缀等配置|
|135 / 136|Neighbor Solicitation / Advertisement|邻居地址解析、可达性检测等|

邻居发现（NDP）使用 ICMPv6 消息并依赖 IPv6 组播等机制；其作用覆盖 IPv4 里 ARP 的地址解析功能，也包含路由器发现等功能。错误报文并非任何情况下都会生成：例如某些广播/组播目标或对另一个 ICMP 错误报文的响应，协议通常限制继续发送错误报文，以免引发风暴。

---

## 九、把一次访问拆成各层字段

假设主机 `192.0.2.10` 要向另一网段服务器 `198.51.100.20:443` 发送 TCP 数据，默认网关为 `192.0.2.1`；以下地址仅作说明。

1. 主机先用路由表确定下一跳是 `192.0.2.1`，必要时通过 ARP 获得网关 MAC。
2. TCP 段填写源端口（例如 `53000`）、目标端口 `443`，以及序列号、确认号和控制位。
3. IPv4 数据报填写源 IP `192.0.2.10`、目标 IP `198.51.100.20`、`Protocol=6`，并据实际长度和路径设置相关字段。
4. 以太网帧填写源 MAC 为主机网卡 MAC，目标 MAC 为**网关 MAC**，`EtherType=0x0800`。
5. 网关转发时剥去收到的二层帧，递减 TTL，重新封装下一段链路的帧；下一跳 MAC 随之改变。

将 TCP 改为 UDP 时，IPv4 的 `Protocol` 改为 `17`，负载开头改为固定 8 B 的 UDP 首部。IPv4 ICMP 报文则使用 `Protocol=1`，没有 TCP/UDP 端口。IPv6 的对应关系由 Next Header 链指示。

## 十、快速辨认报文时的检查顺序

1. **看链路层类型：**检查 Ethernet II 的 EtherType；有 VLAN 标签时先跳过 4 B 标签，读取原 EtherType。
2. **看网络层长度：**IPv4 用 `IHL × 4` 找到负载起点，IPv6 固定首部先跳过 40 B，再按 Next Header 解析扩展首部。
3. **看上层协议：**IPv4 查 Protocol；IPv6 沿 Next Header 链查到 TCP、UDP 或 ICMPv6。
4. **看传输层长度：**TCP 用 `Data Offset × 4` 找到应用数据；UDP 的数据从固定 8 B 首部后开始。
5. **核对长度和校验：**区分以太网负载长度、IPv4 Total Length、IPv6 Payload Length、UDP Length；抓包中的校验和异常可能来自网卡校验和卸载，需结合抓包位置判断。

> **记忆要点：**MAC 管当前链路的下一跳，IP 管网络层目标，TCP/UDP 端口管主机上的应用；ARP 服务于 IPv4 地址解析，ICMP 服务于 IP 错误报告、诊断和控制。解析报文时以**长度字段和协议标识字段**逐层定位，不能假定每层首部永远只有最小长度。
