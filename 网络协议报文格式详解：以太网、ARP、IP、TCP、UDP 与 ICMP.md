# 网络协议报文格式详解：以太网、ARP、IP、TCP、UDP 与 ICMP

## 一、先看协议之间的关系

本文以常见的**以太网承载 IP** 场景为主，也介绍 Wi-Fi 的 802.11 MAC 帧。这里的“MAC”指链路层地址和介质访问控制层格式，而不是一个与 IP、TCP 并列的单独报文协议。下文 IP、TCP、UDP、ICMP、ARP 等协议中的多字节整数字段通常采用**网络字节序（大端）**；802.11 控制字段按自身规范解析。位图中的字段长度以 bit 为单位。

```text
应用数据
  ├─ TCP 段 / UDP 数据报 / ICMP 报文
  │     └─ IPv4 数据报 / IPv6 包
  │            └─ 以太网帧或 Wi-Fi MAC 帧
  └─ ARP 报文 ── 由链路层直接承载（用于 IPv4 地址解析）
```

|层次|常见标识|本层主要解决的问题|地址或标识的作用范围|
|---|---|---|---|
|以太网 / Wi-Fi MAC|链路层地址、帧类型或上层协议标识|同一二层网络中的帧交付|MAC 地址用于当前链路上的下一跳|
|网络层|源/目标 IP、IPv4 Protocol、IPv6 Next Header|跨网络寻址与转发|IP 地址标识网络层端点|
|传输层|TCP/UDP 端口|将数据交给主机上的应用进程|端口区分服务和会话|
|控制报文|ICMP 类型与代码|报告网络错误、执行诊断与控制|ICMP 直接由 IP 承载，没有 TCP/UDP 端口|

路由器转发时，**IP 目标地址通常保持不变，链路层地址会按每一跳重新填写**；IPv4 TTL 或 IPv6 Hop Limit 会递减。NAT 等设备可能改写 IP 地址、端口和校验和，因此“通常”不是“永远”。

宽带接入若使用 PPPoE，则在以太网帧内先封装 PPPoE 和 PPP，再承载 IP；具体格式见第九节。

---

## 二、以太网与 Wi-Fi MAC 帧

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

**VLAN 的作用**是把同一物理交换网络划分为多个逻辑二层网络。不同 VLAN 的广播流量通常分别限制在各自的广播域内，便于隔离部门或业务流量、控制广播范围；不同 VLAN 之间通信通常需要三层转发。交换机之间的链路可用标签区分同一链路承载的多个 VLAN，而连接普通终端的端口通常收发不带标签的帧。PCP 还可标记二层转发优先级；VLAN 本身不等于加密或完整的安全边界。

单个 VLAN 标签使常见的最大帧长由 1518 B 变为 **1522 B**（均含 FCS）。VLAN 标签不改变 IP 或 TCP/UDP 头的内部结构。

### 2.3 Wi-Fi（IEEE 802.11）数据帧格式

Wi-Fi 的数据帧与 Ethernet II 帧不同，常见的 802.11 数据帧按以下顺序排列。前三个地址字段始终存在；Address 4、QoS Control 和 HT Control 只在相应条件下出现。普通三地址数据帧的 MAC 首部为 **24 B**（不计可选字段）。

```text
Frame Control 2B | Duration/ID 2B | Address 1 6B | Address 2 6B |
Address 3 6B | Sequence Control 2B | Address 4 6B（可选）|
QoS Control 2B（可选）| HT Control 4B（可选）|
Frame Body（可变长）| FCS 4B
```

|字段|长度|作用|
|---|---:|---|
|Frame Control|2 B|标明帧类型、子类型及 To DS、From DS、Retry、Protected 等标志|
|Duration/ID|2 B|通常表示本次无线交换预计占用信道的时间，供其他设备设置虚拟载波侦听计时|
|Address 1～3|各 6 B|携带无线链路上的接收者、发送者及 BSSID、原始源或最终目的地址；具体角色由 To DS、From DS 决定|
|Sequence Control|2 B|含序列号和分片号，用于识别重复帧及处理分片|
|Address 4|6 B，可选|To DS 和 From DS 都为 1 的四地址数据帧中使用|
|QoS Control|2 B，可选|QoS 数据帧中使用，含流量优先级等控制信息|
|HT Control|4 B，可选|特定帧中的附加无线链路控制信息|
|Frame Body|可变|承载上层数据或相应帧类型的数据；受保护帧还包含加密相关开销|
|FCS|4 B|用于检错；抓包中可能因网卡处理而看不到|

`To DS` 表示帧发往分布系统（通常经 AP 进入网络），`From DS` 表示帧从分布系统发出。`DA`/`SA` 是当前二层通信的目的/源地址，`RA`/`TA` 是当前无线链路上的接收/发送地址，`BSSID` 标识基本服务集。数据帧的地址含义如下：

|To DS|From DS|Address 1|Address 2|Address 3|Address 4|
|---:|---:|---|---|---|---|
|0|0|DA|SA|BSSID|无|
|1|0|BSSID（RA）|SA（TA）|DA|无|
|0|1|DA（RA）|BSSID（TA）|SA|无|
|1|1|RA|TA|DA|SA|

例如终端经 AP 发送数据时，Address 1 是 AP 的 BSSID，Address 2 是终端 MAC，Address 3 是该二层数据的 DA；若 IP 目标在其他网段，DA 通常是默认网关的 MAC。802.11 MAC 首部没有 Ethernet II 的 EtherType 字段；承载 IP、ARP 等协议时，帧体通常通过 LLC/SNAP 标识上层协议。Wi-Fi 还包括 Beacon 等**管理帧**和 ACK 等**控制帧**，它们的字段组合不一定与上述数据帧相同。常见 Wi-Fi 接口的 **IP MTU 为 1500 B**，它不等于整个 802.11 帧长度。

### 2.4 SSID、BSSID 与 DA 的区别

**SSID** 是用户看到的 Wi-Fi 网络名称，例如 `OfficeWiFi`；同一名称可以由多个 AP、多个频段共同提供。**BSSID** 是标识一个基本服务集（BSS）的 48 bit MAC 地址，通常对应某个 AP 无线接口上提供的一个网络。即使同一台 AP 的 2.4 GHz 和 5 GHz 使用相同 SSID，它们的 BSSID 通常也不同；多台 AP 提供同一 SSID 时也各有 BSSID。SSID 可出现在 Beacon 等管理帧的信息元素中，普通数据帧的 MAC 首部不直接携带 SSID。

**DA** 是数据在当前二层网络中的目的 MAC，由通信目标和下一跳决定，不等于网络名称，也不固定等于 BSSID：

|场景|BSSID 与 DA 的关系|
|---|---|
|终端经 AP 发往其他网段（`To DS=1`）|Address 1 是当前 AP 的 BSSID；Address 3 是 DA，通常为默认网关 MAC|
|AP 向终端发送（`From DS=1`）|Address 2 是 AP 的 BSSID；Address 1 是 DA，即目标终端的 MAC|
|终端在同一局域网内从 AP A 漫游到 AP B|连接的 BSSID 改变；若仍通过同一个默认网关访问外网，DA 可保持不变|
|终端改连到另一个独立路由器的网络|BSSID 改变；默认网关通常也不同，因此访问外网时 DA 通常改变|

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

### 4.3 常见 IPv4 地址及用途

IPv4 地址为 32 bit，常写成四段十进制数。`/n` 表示前 `n` bit 为网络前缀；同一地址是否属于某个子网，需要结合前缀长度判断。

|地址或范围|常见用途|
|---|---|
|公网单播地址|由地址管理机构分配，供互联网上的单播通信使用；具体可达性仍取决于路由和访问策略|
|`10.0.0.0/8`、`172.16.0.0/12`、`192.168.0.0/16`|私有地址，常用于家庭或企业内网；访问公网通常经 NAT 等方式转换|
|`100.64.0.0/10`|共享地址空间，常用于运营商级 NAT（CGNAT）网络，不属于上述私有地址块|
|`127.0.0.0/8`|环回地址，只用于本机通信；常见 `127.0.0.1`|
|`169.254.0.0/16`|链路本地地址，可在没有正常地址配置时用于同一链路通信，不由路由器转发|
|`0.0.0.0`|未指定地址，可在主机尚未获得地址等场景作为源地址；也常在配置中表示监听所有本地 IPv4 地址|
|`224.0.0.0/4`|组播地址，将报文发给加入相应组的接收者|
|`255.255.255.255`|受限广播地址，仅在本地链路使用；某个子网还可有其定向广播地址，例如 `192.168.1.255/24`|
|`192.0.2.0/24`、`198.51.100.0/24`、`203.0.113.0/24`|文档和示例专用地址，不用于真实公网通信|

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
|Flow Label|20 bit|由发送方标记同一报文流，便于网络设备识别并一致地处理该流；`0` 表示未标记|
|Payload Length|16 bit|固定首部**之后**的字节数，通常包括扩展首部和上层负载；超大负载有专门扩展机制|
|Next Header|8 bit|指示紧随固定首部的扩展首部或上层协议；常见值 `6` TCP、`17` UDP、`58` ICMPv6|
|Hop Limit|8 bit|功能类似 IPv4 TTL，逐跳递减|
|源 / 目标地址|各 128 bit|IPv6 地址|

同一条流（例如一条 TCP 连接）的报文通常使用相同的 Flow Label，不同流尽量使用不同值。路由器可结合源地址、目标地址和 Flow Label 识别流，将同一流的报文稳定地分配到同一路径，同时把不同流分散到不同路径；遇到分片等不便读取 TCP/UDP 端口的情况时尤其有用。**Flow Label 不是优先级字段**，不保证带宽或服务质量；流量分类与拥塞通知主要由 Traffic Class 表示。

如果存在扩展首部，固定首部的 Next Header 先指向**第一个扩展首部**，再由各扩展首部的 Next Header 字段依次指向下一项，最后才到 TCP、UDP 或 ICMPv6。IPv6 邻居发现依赖 ICMPv6，不能简单把 ICMPv6 当作可一律屏蔽的“ping 协议”。

### 5.1 IPv6 分片与 Fragment 扩展首部

IPv4 将 Identification、Flags 和 Fragment Offset 放在基本首部中，沿途路由器在允许时也可分片。IPv6 的 **40 B 固定首部没有这些字段**：只有需要分片时，发送主机才加入 **8 B 的 Fragment 扩展首部**。其前一个首部的 `Next Header=44` 指向 Fragment 首部；Fragment 首部内部的 Next Header 再指示后续内容。

```text
|Next Header 8b|Reserved 8b|Fragment Offset 13b|Res 2b|M 1b|
|Identification 32b                                         |
```

|字段|长度|作用|
|---|---:|---|
|Next Header|8 bit|标识 Fragment 首部之后的首部或上层协议|
|Reserved / Res|8 / 2 bit|保留位，发送时置零|
|Fragment Offset|13 bit|本片数据在原始可分片部分中的偏移，单位为 **8 B**|
|M|1 bit|`1` 表示后面还有分片；最后一片为 `0`|
|Identification|32 bit|与源/目标地址等信息一起供接收端识别并重组同一原始包的分片|

IPv6 **路由器不分片**。如果包超过下一段链路的 MTU，路由器丢弃它并返回 ICMPv6 **Packet Too Big**，报告可用 MTU；发送主机据此调整报文大小（路径 MTU 发现），必要时自行分片，最终由接收端重组。分片首部因而只出现在需要分片的包里，而普通 IPv6 包保持固定首部简洁。

### 5.2 常见 IPv6 地址及用途

IPv6 地址为 128 bit，通常写成冒号分隔的十六进制数；连续的零字段可用一次 `::` 压缩。IPv6 接口可能同时拥有多种地址，例如链路本地地址和全球单播地址。

|地址或前缀|常见用途|
|---|---|
|全球单播地址（当前主要从 `2000::/3` 分配）|用于跨网络的单播通信；该大范围内也有特殊用途地址，不能仅凭前 3 bit 判断某个地址可在公网路由|
|`fc00::/7`（实际本地分配通常使用 `fd00::/8`）|唯一本地地址（ULA），用于组织内部通信，通常不在公网路由|
|`fe80::/10`|链路本地单播地址，只在当前链路有效；邻居发现、路由器发现和下一跳通信会用到|
|`::1/128`|环回地址，相当于 IPv4 的 `127.0.0.1`|
|`::/128`|未指定地址，可在尚未获得地址时作为源地址，不能作为普通目的地址|
|`ff00::/8`|组播地址，按组把报文交给多个接收者；IPv6 不使用 IPv4 式广播|
|`2001:db8::/32`|文档和示例专用前缀，不用于真实公网通信|

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

### 6.2 常见 TCP 选项

TCP 选项位于固定 20 B 首部之后，由 Data Offset 确定整个首部的长度，选项与填充合计最多 **40 B**。`EOL` 和 `NOP` 各占 1 B；其他常见选项按 `Kind 1B | Length 1B | Value ...` 编码，其中 **Length 包含 Kind 和 Length 自身**。

|选项|Kind|总长度|格式与作用|
|---|---:|---:|---|
|EOL（End of Option List）|0|1 B|结束选项列表；其后的字节用于填充|
|NOP（No-Operation）|1|1 B|占位，常用于对齐后续选项|
|MSS（Maximum Segment Size）|2|4 B|`Kind(1B) + Length(1B) + MSS(2B)`；在 SYN 中通告自己愿意接收的最大 TCP 数据段负载长度，不含 IP/TCP 首部|
|Window Scale|3|3 B|`Kind(1B) + Length(1B) + Shift(1B)`；在 SYN 中协商窗口扩大因子，使 16 bit Window 字段可表示更大的接收窗口|
|SACK Permitted|4|2 B|`Kind(1B) + Length(1B)`；在 SYN 中声明支持选择确认|
|SACK|5|`2 + 8n` B|`Kind(1B) + Length(1B) + n 组左右边界（每组 8B）`；在连接协商支持后，接收端可在 ACK 中指出已收到的不连续数据区间，帮助发送端有针对性地重传|
|Timestamp|8|10 B|`Kind(1B) + Length(1B) + TSval(4B) + TSecr(4B)`；用于往返时间测量，并帮助识别旧报文段|

MSS、Window Scale 和 SACK Permitted 通常在建立连接的 SYN/SYN+ACK 中协商；SACK 区间则在后续确认报文中报告。Timestamp 也在建立连接时协商，启用后可随报文段携带。选项末尾按需填充，使整个 TCP 首部长度为 **4 B 的整数倍**。

### 6.3 三次握手与字段变化

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

以下专有字段都接在 4 B 通用头之后：

|Type|类型专有字段|后续数据|
|---:|---|---|
|0 / 8：Echo Reply / Request|Identifier 2 B、Sequence Number 2 B，用于匹配请求和应答|可变长回显数据；应答通常原样返回请求中的数据|
|3：Destination Unreachable|通常为未使用的 4 B；Code=4（需要分片但设置了 DF）时，后 2 B 可携带下一跳 MTU|引发错误的原始 IPv4 首部及其后至少 8 B 数据|
|11：Time Exceeded|未使用的 4 B|引发错误的原始 IPv4 首部及其后至少 8 B 数据|
|12：Parameter Problem|Pointer 1 B、未使用的 3 B；Pointer 指向原始 IPv4 首部中发现错误的字节|引发错误的原始 IPv4 首部及其后至少 8 B 数据|

部分 ICMP 错误报文还可按扩展规范携带更多原始报文内容或扩展对象。错误报文中的“后续数据”用于识别哪个包出错，并非新发送的应用数据。

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

以下专有字段同样接在 4 B 通用头之后：

|Type|类型专有字段|后续数据|
|---:|---|---|
|1：Destination Unreachable|未使用的 4 B|尽可能多地引用引发错误的原始 IPv6 包|
|2：Packet Too Big|MTU 4 B，表示下一跳可用 MTU|引用引发错误的原始 IPv6 包|
|3：Time Exceeded|未使用的 4 B|引用引发错误的原始 IPv6 包|
|4：Parameter Problem|Pointer 4 B，指向原始包中发现错误的位置|引用引发错误的原始 IPv6 包|
|128 / 129：Echo Request / Reply|Identifier 2 B、Sequence Number 2 B|可变长回显数据|
|133：Router Solicitation|Reserved 4 B|可选的 NDP 选项，如源链路层地址|
|134：Router Advertisement|Cur Hop Limit 1 B、标志 1 B、Router Lifetime 2 B、Reachable Time 4 B、Retrans Timer 4 B|可选的 NDP 选项，如前缀信息、MTU、源链路层地址|
|135：Neighbor Solicitation|Reserved 4 B、Target Address 16 B|可选的 NDP 选项，如源链路层地址|
|136：Neighbor Advertisement|R/S/O 标志及保留位共 4 B、Target Address 16 B|可选的 NDP 选项，如目标链路层地址|

ICMPv6 错误报文引用原始包的长度受整个错误报文大小限制，通常尽量多带原始包内容，以便定位。NDP 的“后续数据”是按类型编码的选项，不是原始出错包。

邻居发现（NDP）使用 ICMPv6 消息并依赖 IPv6 组播等机制；其作用覆盖 IPv4 里 ARP 的地址解析功能，也包含路由器发现等功能。错误报文并非任何情况下都会生成：例如某些广播/组播目标或对另一个 ICMP 错误报文的响应，协议通常限制继续发送错误报文，以免引发风暴。

---

## 九、PPPoE：在以太网上建立点对点连接

PPP（Point-to-Point Protocol）用于在两端之间封装并传送 IP 等网络层报文，还通过控制协议配置链路。**PPPoE（PPP over Ethernet）**将 PPP 放入以太网帧，常见于宽带路由器向运营商接入设备“拨号”的场景。PPPoE 分为**发现阶段**和**会话阶段**：前者找到对端并获得会话标识，后者才传送 PPP 控制报文和 IP 报文。家中终端到路由器的 Wi-Fi 或以太网通信通常仍按前文格式进行，PPPoE 会话由宽带路由器在上联侧建立。

### 9.1 PPPoE 报文格式

PPPoE 位于 Ethernet II 的负载中。发现阶段的 EtherType 是 `0x8863`，会话阶段是 `0x8864`；两者使用相同的 **6 B PPPoE 首部**：

```text
目标 MAC 6B | 源 MAC 6B | EtherType 2B |
VER 4b | TYPE 4b | CODE 8b | SESSION_ID 16b | LENGTH 16b |
PPPoE 负载（发现阶段为 Tags；会话阶段为 PPP Protocol + PPP 数据）|
以太网 FCS 4B
```

|字段|长度|作用|
|---|---:|---|
|VER / TYPE|各 4 bit|此版本均为 `1`|
|CODE|8 bit|区分发现报文类型；会话阶段固定为 `0x00`|
|SESSION_ID|16 bit|发现前为 `0`；由接入设备在建立会话时分配，在该会话中保持不变|
|LENGTH|16 bit|PPPoE 负载的字节数，不含以太网头及 6 B PPPoE 首部|

发现阶段的负载由可变长 **Tag** 组成，每个 Tag 采用 `Tag Type 2B + Tag Length 2B + Tag Value（变长）` 格式，可携带服务名称等信息。会话阶段的负载以 PPP Protocol 字段开头，后面才是相应的 PPP 控制报文或 IP 包。PPPoE 不在内部重复使用串行链路式 PPP 的 Flag、Address、Control 和 FCS 字段；外层以太网帧已有 MAC 地址和 FCS。

### 9.2 发现阶段：PADI → PADO → PADR → PADS

|报文|CODE|以太网目标 MAC（Destination MAC）|作用|
|---|---:|---|---|
|PADI|`0x09`|广播地址 `FF:FF:FF:FF:FF:FF`|客户端寻找可提供服务的接入设备|
|PADO|`0x07`|客户端的 MAC|接入设备单播回复可提供的服务|
|PADR|`0x19`|选中的接入设备的 MAC|客户端单播请求建立会话|
|PADS|`0x65`|客户端的 MAC|接入设备单播确认，并分配非零的 SESSION_ID|
|PADT|`0xA7`|会话对端的 MAC|会话建立后，任一端可用它通知终止会话|

前四种报文完成发现和建会话；PADT 是终止通知，不属于四步建会话过程。PADI、PADO 和 PADR 的 SESSION_ID 为 `0`。会话由双方的以太网 MAC 地址及 SESSION_ID 标识；发现完成后，会话流量使用单播以太网帧。

### 9.3 会话阶段：PPP 协商与数据传输

建立 PPPoE 会话后，双方先通过 **LCP** 协商 PPP 链路参数，可按协商结果进行身份认证，再通过 **NCP** 配置所承载的网络层协议，例如 IPv4 的 IPCP 或 IPv6 的 IPv6CP。完成相应配置后，才在会话内传送 IP 包。**PPP Protocol** 与以太网的 EtherType 是不同层次的标识，常见取值如下：

|PPP Protocol|负载|
|---:|---|
|`0xC021`|LCP 链路控制|
|`0x8021`|IPCP，配置 IPv4 相关参数|
|`0x8057`|IPv6CP，配置 IPv6 在 PPP 链路上的参数|
|`0x0021`|IPv4 包|
|`0x0057`|IPv6 包|

因此，抓包中会话阶段的典型封装是 `Ethernet II（0x8864）→ PPPoE → PPP Protocol（0x0021 或 0x0057）→ IPv4 或 IPv6`。**客户端发送会话帧时，外层以太网目标 MAC 是发现阶段选中的接入设备 MAC；接入设备回复时，目标 MAC 是客户端 MAC。**它是 PPPoE 会话对端的 MAC，不随 PPP 内层 IP 包的最终目标地址变化。点对点 PPP 链路本身不需要用 ARP 查找对端 MAC。

### 9.4 为什么 PPPoE 的 MTU 常为 1492 B

普通以太网负载最多 **1500 B**。会话阶段在 IP 包前增加 **6 B PPPoE 首部 + 2 B PPP Protocol**，因此通常能容纳的最大 IP 包为 `1500 - 6 - 2 = 1492 B`。若 IPv4 和 TCP 首部各为 20 B，且没有其他选项，则典型 TCP MSS 为 `1492 - 20 - 20 = 1452 B`。实际路径 MTU 还可能更小；若两端和中间链路支持更大的以太网负载，并按扩展机制协商，也可以使用超过 1492 B 的 PPP MTU。
