# 路由协议原理详解：RIP、OSPF、IS-IS、EIGRP 与 BGP

## 一、路由协议到底解决什么问题

路由器转发一个 IP 包时，查找目标 IP 所匹配的路由，确定出口接口和下一跳；在同一链路上，再把包封装成发往下一跳的链路层帧。**路由协议负责让路由器学到、更新这些路由，并不逐包替路由器转发数据。**

例如，路由器 R1 要访问 `10.3.0.0/24`，需要知道该前缀能否到达、经 R2 还是 R3 转发、所选下一跳是否仍可用。直连路由由接口状态产生，静态路由由管理员配置；RIP、OSPF、IS-IS、EIGRP、BGP 则通过交换控制报文动态学习路由。

区分三个动作：

1. **学习候选路由：**从直连接口、静态配置或路由协议获知前缀。
2. **选择并安装路由：**根据前缀长度、协议来源、协议内部的度量和策略，确定可用路由。不同协议的度量不能直接拿数字互相比大小；当多个来源提供同一前缀时，设备还需按本机的优先级规则选择。
3. **实际转发：**对目标 IP 做**最长前缀匹配**，从转发表得到下一跳与出口。比如同时有 `10.0.0.0/8` 和 `10.3.0.0/24`，发往 `10.3.0.8` 的包优先匹配 `/24`。

本文把同一管理域内交换路由的协议称为 **IGP（内部网关协议）**，把跨自治系统（AS，通常是由一个组织统一管理的路由域）交换路由的 BGP 称为 **域间路由协议**。家用主机通常只需要从 DHCP 或 IPv6 路由器通告获得默认网关，并不直接参与这些路由协议。路由器上的 `0.0.0.0/0` 或 `::/0` 表示没有更具体匹配时使用的默认路由。

## 二、三类计算思路

路由协议都要回答“去某个前缀该走哪里”，但各自掌握的信息不同：

|思路|路由器主要知道什么|怎样选路|代表协议|
|---|---|---|---|
|距离矢量|邻居声称能到哪些前缀，以及邻居到前缀的距离|把到邻居的代价与邻居通告的距离相加，选择较小者|RIP；EIGRP 也基于距离矢量，但使用更复杂的 DUAL 算法|
|链路状态|本区域内各路由器与链路的连接关系及代价|建立拓扑图，从自己出发计算最短路径树|OSPF、IS-IS|
|路径矢量|到前缀所经过的自治系统，以及其他路径属性|先按策略筛选，再根据属性选择允许的路径|BGP|

下面这张拓扑可以看出“最短”的定义并不唯一，括号中是链路开销：

```text
       (10)
R1 -------- R2 ---- 目标网段 P
 \          /
  (5)    (2)
    \    /
      R3
```

从 R1 到 R2，直接走只有 **1 个路由器跳数**，经 R3 走有 **2 跳**，所以按跳数衡量会选直连 R2；若按图中开销衡量，直接走开销为 `10`，经 R3 的开销为 `5 + 2 = 7`，会选 R3。最终到前缀 P 的共同末段不影响这个比较。RIP 使用跳数，OSPF/IS-IS 使用各自配置的链路度量；BGP 还会优先考虑自治系统之间的策略。

## 三、RIP：邻居告诉我“还有几跳”

RIP 是典型的**距离矢量协议**。每台路由器维护“目标前缀、下一跳、跳数”等信息，并向邻居通告。收到邻居的通告后，本机把经过该邻居的代价加上去，再与已有路径比较。这与 Bellman-Ford 的思想相符：

```text
本机经邻居 N 到目标 P 的距离 = 到 N 的代价 + N 通告的到 P 的距离
选择所有可用邻居中距离最小的一条
```

RIP 中的度量是**跳数**，有效路径最多 15 跳，`16` 表示不可达。RIPv2 通常每约 30 秒发送一次完整路由更新；拓扑变化时也可触发更新。RIPv2 携带子网掩码，支持无类别前缀；IPv6 对应的协议是 RIPng。[RIPv2 标准](https://www.rfc-editor.org/rfc/rfc2453.html)、[RIPng 标准](https://www.rfc-editor.org/rfc/rfc2080.html)

**故障时为什么可能产生环路？**假设 R2 原本通过 R1 到达 P，随后 R1 到 P 的链路断开。如果 R2 还没收到故障消息，却向 R1 宣称“我能到 P”，R1 可能错误地改走 R2，而 R2 又把包送回 R1。双方还可能在后续更新中不断增加跳数，形成“计数到无穷”的过程，直到度量达到 `16`。

RIP 用以下办法减轻问题：

- **水平分割：**从某邻居学到的路由，不再向该邻居宣称可达。
- **毒性逆转：**仍向该邻居通告该前缀，但把度量设为 `16`。
- **触发更新与失效计时：**发现变化时尽快告知邻居，过期的路由不继续当作正常路径使用。

这些机制减轻环路和收敛延迟，但不改变 RIP 只按跳数衡量、最大 15 跳的局限。

### RIPv2 报文格式与例子

RIPv2 封装在 **UDP 520** 中。UDP 载荷由 4 字节 RIP 头和最多 25 个普通路由表项（每项 20 字节）构成，字段长度单位为字节：

```text
RIP 头：Command(1) | Version(1) | 保留(2)
路由表项：AFI(2) | Route Tag(2) | IP Address(4)
          | Subnet Mask(4) | Next Hop(4) | Metric(4)
```

`Command=1` 为请求，`2` 为响应/更新；IPv4 表项的 `AFI=2`；`Next Hop=0.0.0.0` 表示经发出更新的路由器转发；`Metric` 为该路由器通告的跳数。认证表项有特殊格式，此处只展示普通路由表项。[RIPv2 字段定义](https://www.rfc-editor.org/rfc/rfc2453.html#section-4)

**例子：**设 P 为 `10.3.0.0/24`。R2 发给 R1 的 Response 包含 `AFI=2，Route Tag=0，IP Address=10.3.0.0，Subnet Mask=255.255.255.0，Next Hop=0.0.0.0，Metric=1`。R1 加上到 R2 的 1 跳，得到经 R2 到 P 的度量 `2`；若 R3 通告到 P 的度量为 `2`，R1 经 R3 的度量为 `3`，所以选 R2。这里展示的是字段值，省略了 UDP/IP 头和校验和。

## 四、OSPF：同步拓扑，再各自计算最短路径

OSPF 是**链路状态协议**，不是只问邻居“你离目标多远”。路由器先通过 Hello 报文发现并维持邻居关系，再与相邻路由器同步链路状态数据库（LSDB）。每台路由器用链路状态通告（LSA）描述自己的连接情况，LSA 在规定范围内可靠泛洪；同一区域内的路由器据此获得一致的拓扑视图。[OSPFv2 标准](https://www.rfc-editor.org/rfc/rfc2328.html)

随后，每台路由器以自己为根运行最短路径优先算法（SPF，通常用 Dijkstra 算法），根据链路 **Cost（开销）** 计算到各目标的路径，再从路径中提取第一跳。Cost 是协议的管理度量，常与链路带宽相关，也可以配置；它不是逐包测出的实时延迟。以上图为例，R1 算出经 R3 到 R2 的开销为 7，小于直连开销 10，就选 R3 为下一跳。

OSPF 处理网络变化的大致过程是：

```text
检测到邻居或链路变化 → 生成新 LSA → 在区域内泛洪并更新 LSDB
→ 重新计算 SPF → 更新路由表和转发表
```

OSPF 用 **Area（区域）** 限制拓扑信息的传播范围，骨干区域为 **Area 0**；区域边界路由器负责在区域之间传播路由信息。在以太网等广播网络上，多个路由器并非都彼此维持完整邻接，而是选举 **DR/BDR** 来减少邻接与同步开销。OSPFv2 主要用于 IPv4；OSPFv3 最初为 IPv6 定义，保留了泛洪、区域和 SPF 等核心机制。[OSPFv3 标准](https://www.rfc-editor.org/rfc/rfc5340.html)

### OSPFv2 报文格式与例子

OSPFv2 直接封装在 **IPv4（协议号 89）**中。五种 OSPF 包共享 24 字节头：

```text
Version(1) | Type(1) | Packet Length(2)
Router ID(4) | Area ID(4)
Checksum(2) | AuType(2) | Authentication(8)
后续：由 Type 决定的报文体
```

`Type=1/2/3/4/5` 分别是 Hello、Database Description、Link State Request、Link State Update（LSU）、Link State Acknowledgment。`Packet Length` 包含 OSPF 头和报文体。`Type=4` 的 LSU 报文体为 `LSA 数量(4) + 若干 LSA`；每个 LSA 又以 20 字节公共头开始，含 LSA 类型、链路状态 ID、发布路由器、序列号等。[OSPFv2 报文格式](https://www.rfc-editor.org/rfc/rfc2328.html#appendix-A.3)

**例子：**设 R1、R2、R3 同在 Area 0，Router ID 分别为 `1.1.1.1`、`2.2.2.2`、`3.3.3.3`。R2 的 Router-LSA 描述到 R1 的链路 Cost `10`、到 R3 的 Cost `2`，以及连接的 P `10.3.0.0/24`。R2 将其装进 `Type=4` 的 LSU，OSPF 头写 `Version=2，Router ID=2.2.2.2，Area ID=0.0.0.0`。R3 泛洪同一 LSA 时，外层 OSPF 头的 Router ID 是 `3.3.3.3`，而 LSA 内的发布路由器仍是 `2.2.2.2`。R1 收齐拓扑信息后比较直达 R2 的 `10` 和经 R3 的 `5+2=7`，选 R3 为到 P 的下一跳。字段值为教学示意，省略了接口地址与动态生成的校验和、序列号。

## 五、IS-IS：也是链路状态，但分层与承载方式不同

IS-IS 同样通过邻居发现、泛洪链路状态信息、建立数据库和运行 SPF 来计算路径。路由器之间交换 Hello（IIH）建立邻接，用链路状态 PDU（LSP）描述拓扑；IP 前缀等信息放在可扩展的 TLV 中。与 OSPF 的控制报文经 IP 承载不同，IS-IS 的协议报文直接在链路层上传送；它计算出的 IP 路由仍用于正常 IP 转发。IPv6 前缀也可通过相应 TLV 通告。[集成 IS-IS 标准](https://www.rfc-editor.org/rfc/rfc1195.html)、[IS-IS 的 IPv6 扩展](https://www.rfc-editor.org/rfc/rfc5308.html)

IS-IS 常用 **Level 1 / Level 2** 组织路由域：

- **Level 1：**在同一 IS-IS 区域内交换拓扑和路由。
- **Level 2：**连接不同区域，维护区域间的可达性。
- **Level 1-2 路由器：**同时参加本区域的 Level 1 与跨区域的 Level 2，承担连接作用。

因此，OSPF 与 IS-IS 都属于“掌握拓扑后运行 SPF”的一类；差异主要体现在报文承载、数据库组织和分层方式等协议设计上，不应把 IS-IS 误认为按跳数选路。[IS-IS IP 扩展](https://www.rfc-editor.org/rfc/rfc1195.html)

### IS-IS PDU 格式与例子

IS-IS 的链路层载荷是 **PDU 公共头 + 各类 PDU 的专有头 + TLV 列表**。常见 PDU 有 IIH（邻居 Hello）、LSP（拓扑通告）、CSNP（数据库摘要）和 PSNP（部分摘要/确认）。公共头包含协议标识、头长度、版本、系统 ID 长度和 PDU 类型等；LSP 专有头还包含 PDU 长度、剩余寿命、LSP ID、序列号与校验和。各类 PDU 的专有头长度不同，不能用一个固定长度套用所有 PDU。TLV 的通用格式如下：

```text
Type(1) | Length(1) | Value(Length 字节)
```

IPv4 前缀常用 **扩展 IP 可达性 TLV 135**，其 Value 可重复放入多个 `Metric(4) + 控制/前缀长度(1) + 前缀字节(0～4) + 可选子 TLV`。控制字节包含 up/down 位、子 TLV 标志位及 6 位前缀长度。邻居及其链路度量可用扩展 IS 可达性 TLV 22 描述。[IS-IS 扩展字段](https://www.rfc-editor.org/rfc/rfc5305.html)

**例子：**设三台路由器在同一 Level 1 区域。R2 的 LSP 用 TLV 135 通告 `Metric=1，Prefix Length=24，Prefix=10.3.0`，表示 P `10.3.0.0/24`；`/24` 只需保存 3 个前缀字节。相关 LSP 用邻居可达性信息描述 R2—R3 的链路度量 `2`，R1—R3、R1—R2 分别为 `5`、`10`。R1 收齐 LSP 后运行 SPF，到 P 经 R3 的总度量为 `5+2+1=8`，直达 R2 为 `10+1=11`，所以选 R3。这里的 `1` 是 R2 到 P 的前缀度量，并非 TLV 类型号。

## 六、EIGRP：距离矢量信息加上 DUAL 无环计算

EIGRP 基于距离矢量技术：邻居通告自己到前缀的距离，本机结合到邻居的链路代价计算可行路径。它的度量主要由带宽和延迟等参数构成，不是 RIP 那样的简单跳数。其核心是 **DUAL（扩散更新算法）**，用于在变化时协调受影响的路由器，避免选择会形成环路的下一跳。[EIGRP 说明](https://www.rfc-editor.org/rfc/rfc7868.html)

理解 DUAL 时常见三个术语：

|术语|含义|
|---|---|
|Successor|当前选用的下一跳；等价路径存在时可能有多个|
|Reported Distance（RD）|某邻居通告的、它自己到目标的距离|
|Feasible Distance（FD）|本机对目标记录的最小已知总距离，用作可行性判断；它不一定等于此刻最优路径的当前度量|

若候选邻居满足 **`RD < 本机 FD`**，它满足可行条件，可作为保证无环的 **Feasible Successor**。例如本机 FD 为 20，某邻居 RD 为 10，则该邻居的通告距离小于 20，可能成为预先准备好的备用下一跳。条件是**充分条件**：不满足它，不代表那条路实际一定有环，只是不能凭这项检查直接证明无环。

当前路径失效时，若有可行备用路径，EIGRP 可较快切换；若没有，就进入主动计算，向相关邻居发送查询并等待回复，再确定新路径。这与 RIP 单纯等待各邻居逐步修正距离不同。[EIGRP DUAL 机制](https://www.rfc-editor.org/rfc/rfc7868.html)

### EIGRP 报文格式与例子

EIGRP 直接使用 **IP 协议号 88**，不经 TCP/UDP。一个报文由 20 字节公共头和后续 TLV 组成：

```text
Header Version(1) | Opcode(1) | Checksum(2)
Flags(4) | Sequence Number(4) | Acknowledgment Number(4)
Virtual Router ID(2) | Autonomous System Number(2)
后续 TLV：Type(2) | Length(2) | Value(可变)
```

`Opcode=1/2/3/4/5` 分别为 Update、Request、Query、Reply、Hello；序列号和确认号服务于可靠传输。TLV 的 `Length` **包含 Type 与 Length 字段本身**。IPv4 内部路由 TLV 的 Type 为 `0x0102`，Value 依次包含 `Next-Hop Forwarding Address(4) + Vector Metric Section(可变) + Destination Section(可变)`。下一跳地址为 `0.0.0.0` 时使用收到的 IP 包的源地址；目标部分先有 1 字节掩码位数，`/24` 再跟 3 字节前缀。度量向量保存带宽、延迟等计算依据，不能把它当作 RIP 的跳数字段。[EIGRP 报文与 TLV](https://www.rfc-editor.org/rfc/rfc7868.html#section-6)

**例子：**R2 用 `Opcode=Update` 报文中的 `0x0102` TLV 通告 `10.3.0.0/24`，并附上自身的度量向量。R1 收到后，结合 R1—R2 链路参数计算这条路径的距离。假设 R1 对 P 记录的 FD 为 `20`，R3 通告的 RD 为 `10`，经 R3 的总距离为 `25`，则 R2 可继续做当前 Successor，而 R3 因 `10 < 20` 满足可行条件，可做 Feasible Successor。这些数字只示意 DUAL 比较；实际度量由向量参数和配置计算，不能从几个数字反推完整报文。

## 七、BGP：按自治系统路径和策略选路

BGP 主要用于**自治系统之间**交换前缀，例如运营商、云服务商和大型组织之间。不同 AS 的邻居交换称为 **eBGP**；同一 AS 内传播 BGP 路由称为 **iBGP**。BGP 邻居通过 TCP 端口 179 建立会话，使用 OPEN 建立会话、KEEPALIVE 维持会话、UPDATE 通告或撤销路由、NOTIFICATION 报告错误。[BGP-4 标准](https://www.rfc-editor.org/rfc/rfc4271.html)

BGP 的 UPDATE 不只说“某前缀可达”，还携带路径属性。常见的通告内容与属性有：

|属性|作用|
|---|---|
|NLRI|被通告的目标前缀|
|AS_PATH|路由经过的自治系统序列；若本 AS 已在路径中，可用于拒绝可能造成 AS 级环路的路径|
|NEXT_HOP|到该路由应到达的下一跳地址；必须能通过本机路由解析|
|LOCAL_PREF|在本 AS 内表达优先选用哪条出口路径的策略|
|MED|向相邻 AS 表达进入本 AS 的路径偏好，是否采用由对方策略决定|

BGP 属于**路径矢量**：路由器看到前缀及其经过的 AS 路径，但通常没有整个互联网的物理链路拓扑。它先按策略接受、过滤和偏好路径，再在合格路径中选出最佳路径。**BGP 的“最佳”不是公里数、跳数或延迟最小**；例如一个 AS 可以因商业或工程策略，优先使用 AS_PATH 更长的路径。具体属性比较顺序还受设备实现和配置影响，不能只记一套固定顺序。[BGP 协议分析](https://www.rfc-editor.org/rfc/rfc4274.html)

例如 AS 65001 到同一前缀有两条候选路由：一条经过 `65002 → 65004`，另一条经过 `65003 → 65005 → 65004`。第二条经过的 AS 更多，但 AS 65001 若按本地策略给它更高的 LOCAL_PREF，仍可选第二条作为出口。若某条新通告的 AS_PATH 已包含本 AS `65001`，该路由又被送回本 AS，接收方就能据此识别 AS 级环路风险。

当路径失效时，BGP 可通过 UPDATE 撤销原先通告的前缀或改报另一条路径。借助多协议扩展（MP-BGP），它也可通告 IPv6 等地址族的路由。[MP-BGP 标准](https://www.rfc-editor.org/rfc/rfc4760.html) 即使 BGP 已选择 NEXT_HOP，本 AS 内仍通常需要 IGP 或静态路由使这个下一跳可达。BGP 解决“采用哪条跨 AS 路径”，IGP 解决“在本 AS 内如何走到下一跳”；两者经常协同工作。

### BGP-4 报文格式与例子

BGP 消息承载于 **TCP 字节流**。每条消息有固定 19 字节头，后跟由 Type 决定的报文体：

```text
Marker(16) | Length(2) | Type(1) | 报文体(可变)

UPDATE 报文体：
Withdrawn Routes Length(2) | Withdrawn Routes(可变)
Total Path Attribute Length(2) | Path Attributes(可变)
NLRI(可变)
```

普通 BGP-4 的 Marker 为 16 字节全 `0xff`；`Length` 是**整条 BGP 消息**的长度，含 19 字节头。`Type=1/2/3/4` 分别表示 OPEN、UPDATE、NOTIFICATION、KEEPALIVE。UPDATE 的撤销路由长度为 `0` 表示没有撤销；每个路径属性编码为 `Flags(1) + Type Code(1) + Length(1 或 2) + Value(可变)`；NLRI 的 IPv4 前缀编码为 `前缀长度(1) + 足够容纳该前缀的字节`。[BGP-4 消息格式](https://www.rfc-editor.org/rfc/rfc4271.html#section-4)

**例子：**假设 AS 65004 发布示例前缀 `203.0.113.0/24`。AS 65003 向 AS 65001 发 `Type=2` 的 UPDATE：`Withdrawn Routes Length=0`，路径属性中有 `AS_PATH=[65003, 65005, 65004]` 和可达的 `NEXT_HOP`，NLRI 为 `203.0.113.0/24`。该 `/24` 在 NLRI 中占 `1+3=4` 字节：1 字节前缀长度 `24`，加 3 字节地址前缀 `203.0.113`。若 AS 65001 同时收到经 `65002, 65004` 的另一条路由，可在本 AS 内给前一条更高的 LOCAL_PREF，使其胜出；**LOCAL_PREF 不通过这条 eBGP UPDATE 从 AS 65003 发给 AS 65001**。撤销时，发送方可在后续 UPDATE 的 `Withdrawn Routes` 中列出该前缀。字段值为教学示意。

## 八、把五种协议放在一起看

|协议|主要应用范围|路由信息的核心内容|选路依据|拓扑变化时|
|---|---|---|---|---|
|RIP|较小的内部网络|邻居通告的前缀与跳数|最少跳数|更新距离，可能出现计数到无穷|
|OSPF|内部网络|区域内的链路状态与前缀|链路 Cost、SPF|泛洪新 LSA，重算 SPF|
|IS-IS|内部网络|Level 1/2 的链路状态与前缀|链路度量、SPF|泛洪新 LSP，重算 SPF|
|EIGRP|内部网络|邻居通告的距离及可行性信息|复合度量、DUAL|切换可行备用路径或查询邻居|
|BGP|跨 AS，也用于 AS 内传播外部路由|前缀与 AS 路径等属性|路由策略与路径属性|撤销或更新路径，再重新选路|

**不要混淆协议的选路与 IP 包的转发。**控制平面可以用不同算法形成路由，但数据平面仍按目标 IP 的最长前缀匹配转发；下一跳的链路层地址由当前链路的机制解析。路由协议也不能替代 ARP/NDP、NAT 或防火墙规则。

## 参考标准

- [RFC 2453：RIP Version 2](https://www.rfc-editor.org/rfc/rfc2453.html)；[RFC 2080：RIPng for IPv6](https://www.rfc-editor.org/rfc/rfc2080.html)
- [RFC 2328：OSPF Version 2](https://www.rfc-editor.org/rfc/rfc2328.html)；[RFC 5340：OSPF for IPv6](https://www.rfc-editor.org/rfc/rfc5340.html)
- [RFC 1195：Use of OSI IS-IS for Routing in TCP/IP and Dual Environments](https://www.rfc-editor.org/rfc/rfc1195.html)
- [RFC 5305：IS-IS Extensions for Traffic Engineering](https://www.rfc-editor.org/rfc/rfc5305.html)
- [RFC 5308：Routing IPv6 with IS-IS](https://www.rfc-editor.org/rfc/rfc5308.html)
- [RFC 7868：Cisco's EIGRP](https://www.rfc-editor.org/rfc/rfc7868.html)
- [RFC 4271：BGP-4](https://www.rfc-editor.org/rfc/rfc4271.html)；[RFC 4274：BGP-4 Protocol Analysis](https://www.rfc-editor.org/rfc/rfc4274.html)
- [RFC 4760：Multiprotocol Extensions for BGP-4](https://www.rfc-editor.org/rfc/rfc4760.html)
