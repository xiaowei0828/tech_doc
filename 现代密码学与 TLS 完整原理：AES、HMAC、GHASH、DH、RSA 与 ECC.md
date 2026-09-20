# 现代密码学与 TLS 完整原理：Hash、AES、HMAC、GHASH、DH、RSA 与 ECC

密码系统不是“选择一个最强算法”就结束了。真实协议必须同时解决：

- **机密性**：旁观者看不懂内容；
- **完整性**：内容被修改时能够发现；
- **身份认证**：确认正在和谁通信；
- **密钥建立**：通信前安全得到会话密钥；
- **密钥派生与隔离**：不同方向、阶段和用途使用不同密钥；
- **重放与降级防护**：旧消息不能被无限重复，协议不能被强制退回弱配置。

本文先解释每类算法的数学原理，再以 TLS 1.3 为主线说明它们如何组合。TLS 1.2 仅用于对比和理解历史方案。TLS 1.3 当前规范是 2026 年发布的 RFC 9846；它向后兼容并取代 RFC 8446，主要收紧实现要求和澄清边界，并没有改变 TLS 1.3 的线上版本号。

---

## 一、先看算法分工，而不是算法名字

### 1.1 能力地图

|能力|代表算法/机制|是否直接加密业务数据|
|---|---|---|
|分组密码|AES|只能变换固定 128 bit 分组|
|对称加密模式|CTR、CBC|把分组密码扩展到任意长度；本身未必认证|
|AEAD|AES-GCM、AES-CCM、ChaCha20-Poly1305|同时提供机密性和完整性|
|密码学哈希|SHA-256、SHA-384|把任意长度消息压缩成固定长度摘要，不使用秘密密钥|
|消息认证码|HMAC、CMAC、Poly1305|认证消息，不加密|
|通用哈希组件|GHASH|GCM 的认证计算组件，不能单独当安全 MAC|
|密钥交换|DH、ECDH|建立共享秘密，不认证身份|
|公钥加密/签名|RSA-OAEP、RSA-PSS|分别用于加密和签名|
|椭圆曲线签名|ECDSA|认证签名，不加密业务数据|
|密钥派生|HKDF|把共享秘密扩展并隔离成多把密钥|
|公钥身份绑定|X.509/PKI|证明某个公钥属于某个主体|

最重要的边界：

1. **加密不自动保证完整性**。CTR 密文可被按位翻转，接收方可能得到被定向修改的明文。
2. **哈希不等于 MAC**。公开的 `Hash(message)` 不能证明消息来自持钥者。
3. **密钥交换不等于认证**。裸 DH/ECDH 会受到中间人攻击。
4. **签名不等于保密**。任何拿到公钥的人都能验证签名并看到被签内容。
5. **算法安全不等于协议安全**。Nonce、编码、错误处理、随机数和状态机同样关键。

### 1.2 TLS 怎样组合这些能力

TLS 1.3 中最常见的“证书认证 + 临时 ECDHE”完整握手可以简化为：

```text
临时 ECDHE
  → 得到临时共享秘密
  → HKDF 派生握手密钥
  → 证书公钥 + 签名认证服务器
  → Finished 认证完整握手记录
  → HKDF 派生双向应用流量密钥
  → AES-GCM / ChaCha20-Poly1305 保护记录
```

AES 不负责协商密钥，ECDHE 不负责证明身份，证书签名也不负责批量加密。TLS 的价值就在于把这些原语按严格状态机组合，并绑定协商参数与完整握手 transcript。

这不是 TLS 1.3 的唯一模式：它还支持有限域 DHE、PSK 与临时密钥建立的组合，以及纯 PSK 模式。RFC 9846 进一步把相关描述泛化为“非对称密钥建立”，为其他规范定义 KEM 或混合密钥交换留出空间；理解经典主线时仍可以先把共享秘密看成 ECDHE 的输出。

---

## 二、共同数学基础

### 2.1 模运算与有限集合

模 $n$ 运算只保留余数：

$$
a\equiv b\pmod n
$$

表示 $n\mid(a-b)$。加法和乘法都可以先正常计算再取模：

$$
(a+b)\bmod n,\qquad (ab)\bmod n
$$

若：

$$
\gcd(a,n)=1
$$

则 $a$ 存在乘法逆元 $a^{-1}$：

$$
aa^{-1}\equiv1\pmod n
$$

RSA、有限域椭圆曲线和 AES 字节运算都依赖“在有限集合中定义可逆运算”，但它们使用的代数结构不同。

### 2.2 群、域与困难问题

- **群**：有封闭运算、单位元、逆元并满足结合律。DH 需要一个离散对数困难的循环子群。
- **域**：加减乘除（除以零除外）都有定义。AES 使用 $GF(2^8)$，GHASH 使用 $GF(2^{128})$，常见 ECC 使用素域 $GF(p)$。
- **困难问题**：正向计算容易，逆向求解在已知最好算法下代价极高。

代表性困难问题：

- 有限域离散对数：已知 $g,g^a$，求 $a$；
- 椭圆曲线离散对数：已知 $G,[d]G$，求 $d$；
- RSA 反演：在不知道私钥的情况下，从 $c=m^e\bmod n$ 恢复 $m$。

RSA 安全常与大整数分解密切相关，但“攻破 RSA 与分解 $n$ 已被证明完全等价”并不是一般结论。

### 2.3 对称密码与公钥密码为何要组合

公钥运算便于在没有预共享密钥时建立信任或共享秘密，但计算昂贵、处理消息长度受限。对称密码适合高速保护大量数据，但通信双方必须先共享密钥。

所以现代协议通常：

1. 用 ECDHE 建立一次会话共享秘密；
2. 用证书签名认证交换过程；
3. 用 HKDF 派生对称密钥；
4. 用 AEAD 保护后续大量数据。

---

## 三、AES：可逆的 128 bit 分组变换

### 3.1 AES 的输入、密钥与轮数

AES 固定处理 128 bit，也就是 16 字节分组。密钥长度和轮数为：

|算法|密钥长度|轮数 $N_r$|
|---|---:|---:|
|AES-128|128 bit|10|
|AES-192|192 bit|12|
|AES-256|256 bit|14|

AES 可以抽象为带密钥的置换：

$$
E_K:\{0,1\}^{128}\rightarrow\{0,1\}^{128}
$$

对固定密钥 $K$，每个 128 bit 输入唯一映射到一个 128 bit 输出，并存在逆变换 $D_K$：

$$
D_K(E_K(P))=P
$$

AES 本身不是任意长度消息加密方案，也不提供 Nonce、Padding、认证或重放保护；这些由工作模式或上层协议定义。

### 3.2 State 的列优先排列

输入 16 字节 $b_0,\ldots,b_{15}$ 按列填入 $4\times4$ State：

$$
\begin{bmatrix}
b_0&b_4&b_8&b_{12}\\
b_1&b_5&b_9&b_{13}\\
b_2&b_6&b_{10}&b_{14}\\
b_3&b_7&b_{11}&b_{15}
\end{bmatrix}
$$

这个排列决定 ShiftRows、MixColumns 和测试向量的解释方式。把输入按行填充会得到完全错误的中间状态。

### 3.3 AES 字节所在的 $GF(2^8)$

一个字节：

$$
b_7b_6\ldots b_0
$$

表示二进制多项式：

$$
b_7x^7+b_6x^6+\cdots+b_1x+b_0
$$

AES 在不可约多项式

$$
m(x)=x^8+x^4+x^3+x+1
$$

对应的有限域中计算。

域加法就是 XOR：

$$
a+b=a\oplus b
$$

乘法是多项式乘法后模 $m(x)$。乘以 $x$，也就是乘常数 `02`，常用 `xtime`：

```text
若最高位为 0：xtime(a) = a << 1
若最高位为 1：xtime(a) = (a << 1) XOR 0x1B
```

结果只保留低 8 位。`0x1B` 是消去 $x^8$ 后的约简常数。

### 3.4 每轮的四个变换

#### SubBytes：非线性

每个字节 $a$ 先在 $GF(2^8)$ 中求乘法逆元（0 作特殊定义），再做仿射变换，得到 S-box 输出。

非线性非常关键：如果所有步骤都是线性变换，多轮可以合并成一次线性变换，攻击者就能用线性代数恢复结构。

S-box 不是随机查表，而是由可逆代数过程构造；解密使用逆 S-box。

#### ShiftRows：跨列扩散

第 $r$ 行循环左移 $r$ 个字节：

```text
第 0 行：不移动
第 1 行：左移 1
第 2 行：左移 2
第 3 行：左移 3
```

它把原本处于同一列的字节分散到不同列，使后续 MixColumns 能把影响传播到整个 State。

#### MixColumns：列内线性混合

每一列乘固定矩阵：

$$
\begin{bmatrix}
02&03&01&01\\
01&02&03&01\\
01&01&02&03\\
03&01&01&02
\end{bmatrix}
$$

所有乘加都在 $GF(2^8)$ 中进行。例如：

$$
s'_0
=(02\cdot s_0)\oplus(03\cdot s_1)\oplus s_2\oplus s_3
$$

MixColumns 提供扩散：一个输入字节的变化会影响该列全部输出，配合 ShiftRows 在多轮后扩散到整个 State。

#### AddRoundKey：注入密钥

$$
\text{State}\leftarrow\text{State}\oplus\text{RoundKey}
$$

XOR 自身可逆：

$$
(x\oplus k)\oplus k=x
$$

前三个变换提供非线性和扩散，AddRoundKey 把这些结构与秘密密钥绑定。

### 3.5 加密轮结构

AES 加密：

```text
初始轮：
  AddRoundKey

中间轮，共 Nr - 1 轮：
  SubBytes
  ShiftRows
  MixColumns
  AddRoundKey

最后一轮：
  SubBytes
  ShiftRows
  AddRoundKey
```

最后一轮省略 MixColumns 是 AES 标准化结构的一部分，安全分析针对的就是这套完整轮函数与轮数。这不是“MixColumns 可有可无”，也不是允许实现者自行删减其他轮步骤的通用理由。

### 3.6 Key Expansion

AES 把原始密钥扩展成 $N_r+1$ 组 128 bit 轮密钥。以 AES-128 为例：

- 原密钥是 4 个 32 bit word；
- 共需要 $4(N_r+1)=44$ 个 word；
- 每 4 个 word 形成一轮密钥。

核心辅助操作：

- `RotWord`：循环移动 4 字节；
- `SubWord`：逐字节应用 S-box；
- `Rcon`：加入轮相关常数，破坏对称。

AES-128 的递推可概括为：

$$
w_i=
\begin{cases}
w_{i-4}\oplus
\operatorname{SubWord}(\operatorname{RotWord}(w_{i-1}))
\oplus\operatorname{Rcon}_{i/4},
&i\bmod4=0\\
w_{i-4}\oplus w_{i-1},
&\text{其他}
\end{cases}
$$

Key Schedule 是专为 AES 轮密钥设计的可逆展开，不是密码学 KDF。知道完整轮密钥状态通常可以反推原始密钥，不能用它代替 HKDF 派生不同用途的密钥。

### 3.7 解密为何能恢复明文

每个加密步骤都有逆变换：

- `AddRoundKey` 的逆仍是 XOR 同一轮密钥；
- `SubBytes` 对应 `InvSubBytes`；
- `ShiftRows` 对应 `InvShiftRows`；
- `MixColumns` 对应 `InvMixColumns`。

解密按相反顺序使用相应轮密钥：

$$
P=D_K(C)
$$

实现必须通过标准测试向量验证。自行实现 AES 适合教学，不适合生产；生产应使用经过审计并具备侧信道防护的密码库。

---

## 四、从分组密码到任意长度消息

### 4.1 ECB 为什么不安全

ECB 对每个分组独立计算：

$$
C_i=E_K(P_i)
$$

相同明文块产生相同密文块，图像纹理、记录类型和重复结构会泄露。即使 AES 本身没有被破解，消息模式也已经暴露。

### 4.2 CBC 与 Padding

CBC：

$$
C_i=E_K(P_i\oplus C_{i-1}),\qquad C_0=IV
$$

明文必须填充到完整分组。CBC 只提供机密性，必须另加 MAC；错误的“先解密、再根据不同错误返回 Padding/MAC 失败”会形成 Padding Oracle。

历史 TLS 的 CBC 套件需要精细处理 MAC、Padding、错误时序和 IV，协议复杂度高，已不属于 TLS 1.3。

### 4.3 CTR：把 AES 变成密钥流

CTR 为每个计数器块生成密钥流：

$$
S_i=E_K(\operatorname{Counter}_i)
$$

$$
C_i=P_i\oplus S_i
$$

解密使用同一操作：

$$
P_i=C_i\oplus S_i
$$

CTR 无需 Padding，可并行，也支持随机访问；但相同密钥下不能重复计数器序列。若重复：

$$
C_1\oplus C_2=P_1\oplus P_2
$$

而且 CTR 本身可篡改：翻转密文某一位会翻转明文对应位。因此实际协议应优先使用 AEAD，而不是“裸 CTR”。

### 4.4 为什么现代协议选择 AEAD

AEAD（Authenticated Encryption with Associated Data）同时保护：

- 明文 $P$：加密且认证；
- 附加认证数据 AAD：不加密但认证；
- Nonce：由调用方按算法约束提供；GCM 等方案要求同一密钥下不重复。

接口可抽象为：

$$
(C,T)=\operatorname{Seal}(K,N,P,A)
$$

$$
P=\operatorname{Open}(K,N,C,A,T)
$$

只有 Tag 验证成功，`Open` 才能返回明文。认证失败时释放“暂时解出的明文”会把协议重新暴露给篡改和 Oracle 风险。

### 4.5 ChaCha20：用 ARX 运算生成密钥流

ChaCha20 是流密码，不依赖 AES 分组变换。它把以下内容排成 16 个 32 bit word：

```text
4 个常量 || 8 个 Key word || 1 个 Block Counter || 3 个 Nonce word
```

IETF 版本使用 256 bit Key、32 bit Block Counter 和 96 bit Nonce。所有 word 按小端解释。核心 Quarter Round 只使用三类运算：

- 32 bit 模加法（Add）；
- XOR；
- 固定位数循环左移（Rotate）。

这类结构简称 ARX。对四个 32 bit word $(a,b,c,d)$，Quarter Round 为：

```text
a += b; d ^= a; d <<< 16
c += d; b ^= c; b <<< 12
a += b; d ^= a; d <<<  8
c += d; b ^= c; b <<<  7
```

ChaCha20 对 4×4 状态交替执行列 Quarter Round 和对角 Quarter Round；一组列轮加一组对角轮称为 Double Round。重复 10 个 Double Round 共 20 轮后，把工作状态与初始状态逐 word 模 $2^{32}$ 相加，再按小端序列化，得到 64 字节密钥流块。

第 $i$ 个明文块的加密为：

$$
C_i=P_i\oplus
\operatorname{ChaCha20Block}(K,\operatorname{counter}+i,N)
$$

解密使用同一 XOR。它与 CTR 一样只提供机密性，不认证密文；同一 Key 和 Nonce 重复会生成相同密钥流，使攻击者得到：

$$
C_1\oplus C_2=P_1\oplus P_2
$$

32 bit Block Counter 也限制了单个 Nonce 下可生成的块数，不能让 Counter 回绕。ChaCha20-Poly1305 使用 counter 0 派生一次性 Poly1305 Key，并从 counter 1 开始加密正文；这就是下一章 Poly1305 不能与 ChaCha20 密钥调度割裂理解的原因。

---

## 五、Hash、MAC 与 HMAC：从公开摘要到带密钥认证

### 5.1 密码学 Hash 的接口与三类安全目标

密码学哈希函数把任意长度 bit 串映射为固定长度摘要：

$$
H:\{0,1\}^*\rightarrow\{0,1\}^n
$$

它具有三个基础接口特征：

- **确定性**：同一组输入字节总得到同一摘要；
- **公开计算**：计算 Hash 不需要秘密密钥；
- **压缩**：输入空间无限、输出只有 $2^n$ 种，所以碰撞在数学上必然存在。

“安全 Hash”不是指绝对没有碰撞，而是让攻击者在现实计算资源内难以完成特定任务。常见安全目标必须分开：

|安全目标|给攻击者什么|攻击者要找到什么|理想 $n$ bit Hash 的通用工作量|
|---|---|---|---:|
|原像抗性（preimage resistance）|目标摘要 $y$|任意 $M$，使 $H(M)=y$|约 $2^n$|
|第二原像抗性（second-preimage resistance）|指定消息 $M$|不同的 $M'\ne M$，使 $H(M')=H(M)$|约 $2^n$|
|碰撞抗性（collision resistance）|没有指定消息|任意一对 $M\ne M'$，使摘要相同|约 $2^{n/2}$|

碰撞攻击之所以只需约 $2^{n/2}$，来自生日界。随机抽取 $q$ 个理想摘要时，至少一对碰撞的概率近似为：

$$
1-\exp\left(-\frac{q(q-1)}{2^{n+1}}\right)
$$

当 $q$ 达到 $2^{n/2}$ 量级时，概率就不再可以忽略。因此，SHA-256 的理想碰撞安全强度是约 128 bit，而理想原像安全强度是约 256 bit；“输出 256 bit”不能直接简写成“所有攻击都是 256 bit”。这些是理想模型下的通用量级，不是对任意具体算法的自动证明。

表中的第二原像工作量也是理想 $n$ bit Hash 的常用通用量级。对于极长或具有特殊结构的 Merkle-Damgård 消息，更精细的第二原像分析还与消息分组数有关，不能脱离具体构造和消息长度机械套用 $2^n$。

Hash 还常希望具备良好的扩散：输入改动一个 bit，输出应发生难以预测的大范围变化。但“雪崩效应看起来很强”只是必要的工程现象，不等价于已经证明上述三类抗性。

### 5.2 SHA-256 的 Padding、分块与链式状态

本文以 TLS 和 HMAC 中常见的 SHA-256 为具体例子。SHA-256 属于 SHA-2 家族，接受长度：

$$
0\le \ell<2^{64}
$$

bit 的消息，并输出 256 bit 摘要。它使用 Merkle-Damgård 式迭代结构：先把消息规范填充成 512 bit 分组，再让固定长度压缩函数逐块更新 256 bit 链值。

#### Padding 为什么包含原始长度

对原始长度为 $\ell$ bit 的消息：

1. 追加一个 `1` bit；
2. 追加最少数量 $k$ 个 `0` bit，使：

$$
\ell+1+k\equiv448\pmod{512}
$$

3. 追加 $\ell$ 的 64 bit 大端编码。

最终长度为 512 的整数倍。即使消息原本已经差 64 bit 就对齐，也必须先追加 `1`，必要时新增整个分组；空消息同样会得到一个完整的填充分组。

长度字段和 `1` 分隔位让合法填充具有明确边界，避免简单补零造成：

```text
M
M || 0x00
M || 0x00 || 0x00
```

等不同消息在分块前变成无法区分的尾部形式。Padding 属于算法定义，参与最终摘要计算，不是传输层为了凑整而添加、随后再删除的可选字节。

#### 初始链值与消息分组

填充后的消息被解析为：

$$
M^{(1)},M^{(2)},\ldots,M^{(N)},
\qquad |M^{(i)}|=512\text{ bit}
$$

SHA-256 维护 8 个 32 bit 状态 word：

$$
H_0^{(i)},H_1^{(i)},\ldots,H_7^{(i)}
$$

初始值是规范固定常量：

```text
6a09e667  bb67ae85  3c6ef372  a54ff53a
510e527f  9b05688c  1f83d9ab  5be0cd19
```

它们不是密钥，来源于前 8 个素数平方根小数部分的固定 bit。每个消息块处理完成后产生新的 8-word 链值，作为下一块的输入：

$$
F:\{0,1\}^{256}\times\{0,1\}^{512}
\rightarrow\{0,1\}^{256}
$$

$$
H^{(i)}
=
F\left(H^{(i-1)},M^{(i)}\right)
$$

最后把 $H_0^{(N)}$ 到 $H_7^{(N)}$ 按大端顺序拼接，得到 256 bit 摘要。

### 5.3 SHA-256 的消息扩展与 64 轮压缩

每个 512 bit 消息块先按大端拆成 16 个 32 bit word，再扩展成 64 个 word：

$$
W_t=
\begin{cases}
M_t,&0\le t\le15\\
\sigma_1(W_{t-2})+W_{t-7}
+\sigma_0(W_{t-15})+W_{t-16}\pmod{2^{32}},&16\le t\le63
\end{cases}
$$

其中小 Sigma 函数为：

$$
\sigma_0(x)
=
\operatorname{ROTR}^{7}(x)
\oplus\operatorname{ROTR}^{18}(x)
\oplus\operatorname{SHR}^{3}(x)
$$

$$
\sigma_1(x)
=
\operatorname{ROTR}^{17}(x)
\oplus\operatorname{ROTR}^{19}(x)
\oplus\operatorname{SHR}^{10}(x)
$$

`ROTR` 是循环右移，移出的 bit 会回到左侧；`SHR` 是逻辑右移，左侧补零。二者不能混用。消息扩展让原始一个 word 的影响进入后续许多轮，而不是只处理一次就消失。

#### 轮函数

处理当前消息块前，把上一块链值复制到工作变量：

$$
(a,b,c,d,e,f,g,h)
=
(H_0,H_1,H_2,H_3,H_4,H_5,H_6,H_7)
$$

SHA-256 使用两组大 Sigma 和两个布尔函数：

$$
\Sigma_0(x)
=
\operatorname{ROTR}^{2}(x)
\oplus\operatorname{ROTR}^{13}(x)
\oplus\operatorname{ROTR}^{22}(x)
$$

$$
\Sigma_1(x)
=
\operatorname{ROTR}^{6}(x)
\oplus\operatorname{ROTR}^{11}(x)
\oplus\operatorname{ROTR}^{25}(x)
$$

$$
\operatorname{Ch}(x,y,z)
=(x\land y)\oplus(\neg x\land z)
$$

$$
\operatorname{Maj}(x,y,z)
=(x\land y)\oplus(x\land z)\oplus(y\land z)
$$

`Ch` 可理解为由 $x$ 的每一 bit 在 $y,z$ 中选择；`Maj` 输出三者逐 bit 的多数值。第 $t$ 轮使用固定常量 $K_t$ 和扩展 word $W_t$：

$$
T_1
=h+\Sigma_1(e)+\operatorname{Ch}(e,f,g)+K_t+W_t
\pmod{2^{32}}
$$

$$
T_2
=\Sigma_0(a)+\operatorname{Maj}(a,b,c)
\pmod{2^{32}}
$$

然后更新。下式右侧全部取**本轮开始时的旧值**，逻辑上是同时赋值；若按从上到下的普通赋值语句直接覆盖变量，会实现成另一个错误算法：

$$
\begin{aligned}
h&\leftarrow g,&g&\leftarrow f,&f&\leftarrow e,\\
e&\leftarrow d+T_1,&d&\leftarrow c,&c&\leftarrow b,\\
b&\leftarrow a,&a&\leftarrow T_1+T_2
\end{aligned}
\qquad(\bmod 2^{32})
$$

64 个 $K_t$ 也是公开固定常量，来自前 64 个素数立方根小数部分的 bit。安全性不依赖隐藏常量。

#### Feed-forward 为什么不能省略

完成 64 轮后，不是直接用工作变量替换状态，而是逐 word 加回进入本块前的链值：

$$
\begin{aligned}
H_0'&=H_0+a,&H_1'&=H_1+b,&H_2'&=H_2+c,&H_3'&=H_3+d,\\
H_4'&=H_4+e,&H_5'&=H_5+f,&H_6'&=H_6+g,&H_7'&=H_7+h
\end{aligned}
\qquad(\bmod 2^{32})
$$

这个 Feed-forward 把旧链值与本块压缩结果绑定，再把新链值送入下一消息块。循环移位、XOR、布尔函数和模加法共同扩散消息与状态；它们解释了算法怎样混合 bit，但不能单独构成 SHA-256 安全性的数学证明。生产实现仍应使用经过验证的标准库与测试向量。

### 5.4 Hash 的使用边界与长度扩展

公开 Hash 能检测“拿到的字节是否与某个**可信摘要**一致”，但不能独立证明来源。若攻击者能同时替换文件和旁边公开的 SHA-256 字符串，重新计算摘要即可通过检查。因此：

- 软件分发需要由可信渠道提供摘要，或使用数字签名；
- 协议认证使用 HMAC、CMAC、签名或 AEAD，不能只附加 `Hash(message)`；
- 密码存储应使用带 salt、成本可调的专用 Password KDF，而不是高速 SHA-256；
- 结构化字段必须先做规范、无歧义编码，再 Hash，避免 `a || bc` 与 `ab || c` 产生相同输入字节解释。

SHA-256 的链式结构还带来**长度扩展**。若攻击者知道 $H(M)$ 和 $|M|$，就知道处理完：

$$
M\parallel\operatorname{pad}(M)
$$

后的链值。攻击者可以把这个公开摘要当作继续计算的初始状态，而无需重新处理 $M$ 的内部字节，就能求出：

$$
H\left(
M\parallel\operatorname{pad}(M)\parallel X
\right)
$$

这里的 `pad(M)`（也称 glue padding）会真实出现在伪造消息中；长度扩展得到的不是 $H(M\parallel X)$。在继续处理 $X$ 后，算法还会根据整条扩展消息的累计 bit 长度追加新的最终 Padding。

所以不能用：

$$
\operatorname{SHA256}(K\parallel M)
$$

代替标准 MAC。HMAC 的内外两层构造正是为了避免把可继续扩展的内部链值直接暴露给攻击者。

典型 secret-prefix MAC 攻击中，服务器实际计算 $H(K\parallel M)$。攻击者通常知道公开消息 $M$，猜测秘密前缀 $K$ 的长度，再提交 $M\parallel\operatorname{pad}(K\parallel M)\parallel X$ 与扩展后的摘要；攻击并不要求恢复 $K$。这也是“知道原始消息”与“知道压缩函数内部秘密前缀”必须分开描述的原因。

不要把这个结论错误推广到所有 Hash。SHA-3 使用 Sponge 结构，BLAKE 系列也不是 SHA-256 的同一压缩流程；它们的内部状态、扩展性质和安全边界要按各自规范分析。相反，GHASH 名字里虽然有 “HASH”，却是 GCM 内部的带参数线性通用哈希组件，不具备 SHA-256 这种独立密码学摘要接口。

SHA-384 也不是“把 SHA-256 输出加长”。它基于 SHA-512 的 1024 bit 分组、64 bit word 和 80 轮压缩，使用不同初始值，最后截取 512 bit 内部链值的前 384 bit。本文详细展开 SHA-256，是因为二者共享“规范 Padding、消息扩展、迭代压缩、Feed-forward”这一理解框架；具体位宽、旋转常数和初始值必须按各自标准实现。

在 TLS 中，SHA-256/SHA-384 主要承担三类工作：

1. 对规范编码的 Handshake 消息计算 transcript hash；
2. 作为 HMAC 的底层 Hash，进而构成 HKDF、Finished 和 PSK Binder；
3. 在签名方案要求时参与消息摘要，但具体证书签名算法与 TLS Cipher Suite 的 Hash 选择不是同一个协商项。

### 5.5 MAC 的统一接口与安全目标

MAC（Message Authentication Code，消息认证码）不是某一个固定算法，而是一类使用**共享秘密密钥**生成和验证短 Tag 的对称密码机制。一个 MAC 方案可以抽象为三个算法：

1. 密钥生成：

$$
K\leftarrow\operatorname{Gen}(1^\lambda)
$$

2. 对消息生成 Tag：

$$
T\leftarrow\operatorname{Tag}_K(M)
$$

3. 验证消息与 Tag：

$$
b\leftarrow\operatorname{Verify}_K(M,T),
\qquad b\in\{0,1\}
$$

正确性要求诚实生成的 Tag 总能通过验证：

$$
\operatorname{Verify}_K
\left(M,\operatorname{Tag}_K(M)\right)=1
$$

上面是常见的确定性接口。GMAC 等方案还显式接收 Nonce，可写成 $\operatorname{Tag}_K(N,M)$；此时安全定义必须额外约束 Nonce 的生成与复用。不能把需要唯一 Nonce 的方案当成“同一消息永远得到同一 Tag”的普通确定性 MAC。

MAC 的核心安全目标通常称为**选择消息攻击下的存在性不可伪造**（EUF-CMA）。攻击者可以自适应选择许多消息并获得相应 Tag；即便如此，也不应能为一条此前没有查询过的新消息 $M^*$ 生成可通过验证的 $T^*$。

这个定义比“攻击者找不到两个相同 Tag”更准确：

- MAC 主要关心攻击者能否伪造**有效消息与 Tag**，而不是单纯讨论哈希碰撞；
- 碰撞抗性不是所有 MAC 的统一安全定义，也不能直接替代 HMAC、CMAC 或 Poly1305 各自的安全分析；
- 某些协议还要求**强不可伪造性**：即使消息已经查询过，也不能为同一消息制造一个新的有效 Tag。是否需要这一性质取决于上层协议。

若一个理想化 MAC 输出 $t$ bit Tag，攻击者进行 $q$ 次近似独立的在线猜测，累计猜中概率约为：

$$
1-(1-2^{-t})^q
\approx
\frac{q}{2^t},
\qquad q\ll2^t
$$

这只是理想化近似；真实安全界还受构造方式、认证总数据量、Nonce 规则和密钥生命周期影响。因此，Tag 截断、失败重试次数和限流策略必须一起评估，不能只看单次猜测概率。

MAC 还明确**不自动提供**以下能力：

- **不提供保密性**：消息仍是明文；
- **不提供不可否认性**：通信双方都持有同一密钥，双方都能生成 Tag；
- **不自动防重放**：旧的 $(M,T)$ 原样再次提交，仍会通过纯 MAC 验证。

若协议需要防重放，必须把序号、Nonce、时间窗、方向和上下文等字段以无歧义编码纳入认证输入，并由验证端维护“哪些值已经使用”的状态。例如：

$$
M_{\text{auth}}
=
\operatorname{Encode}
(\text{protocol},\text{direction},\text{sequence},\text{payload})
$$

只把序号放在未认证的旁路字段中没有意义，攻击者可以连同旁路字段一起修改。

### 5.6 哈希、MAC 与签名的区别

|机制|需要秘密|谁能验证|主要目标|
|---|---|---|---|
|Hash(message)|否|任何人|内容指纹，不证明来源|
|HMAC(key, message)|共享密钥|持有同一密钥的人|完整性与共享密钥持有者认证|
|Signature(private_key, message)|私钥签名|任何持公钥者|公开可验证的身份认证|

HMAC 是 MAC 家族的一种具体构造。它不加密消息，也不提供公钥意义上的不可否认性：因为通信双方都持有同一把 MAC 密钥，任何一方都能生成合法 Tag。

### 5.7 HMAC 公式

设哈希函数为 $H$，内部块大小为 $B$，输出长度为 $L$。HMAC 定义：

$$
\operatorname{HMAC}_K(M)
=
H\left(
(K_0\oplus\operatorname{opad})
\parallel
H((K_0\oplus\operatorname{ipad})\parallel M)
\right)
$$

其中：

- `ipad` 是重复到 $B$ 字节的 `0x36`；
- `opad` 是重复到 $B$ 字节的 `0x5c`；
- $K_0$ 是规范化到一个哈希块的密钥。

对 SHA-256：

$$
B=64\text{ bytes},\qquad L=32\text{ bytes}
$$

### 5.8 密钥规范化

$$
K_0=
\begin{cases}
H(K)\parallel 0^{B-L},&|K|>B\\
K\parallel0^{B-|K|},&|K|\le B
\end{cases}
$$

之后：

1. 计算内层：

$$
\text{inner}
=H((K_0\oplus\operatorname{ipad})\parallel M)
$$

2. 计算外层：

$$
\text{tag}
=H((K_0\oplus\operatorname{opad})\parallel\text{inner})
$$

验证端重新计算 Tag，并使用常量时间比较，避免比较过程泄露第一个不相等字节的位置。

### 5.9 为什么不是简单的 `Hash(key || message)`

5.4 节已经说明，直接公开 $H(K\parallel M)$ 会把一个可继续计算的 Merkle-Damgård 链值暴露给攻击者。HMAC 使用不同的内外密钥填充，并对内层摘要再次哈希。攻击者只看到：

$$
H\left((K_0\oplus\operatorname{opad})\parallel\text{inner}\right)
$$

即使对这个外层摘要做长度扩展，得到的也是“外层输入后再附加数据”的 Hash；它不具备合法 HMAC 所要求的：

$$
H\left(
(K_0\oplus\operatorname{opad})
\parallel
H((K_0\oplus\operatorname{ipad})\parallel M')
\right)
$$

结构。攻击者既不知道两个密钥化前缀，也不能把公开 Tag 当作新的 HMAC 内层摘要。

安全性不应简化为“哈希抗碰撞，所以 HMAC 一定安全”。HMAC 有专门的 PRF/MAC 安全分析；底层哈希的压缩函数性质、密钥质量和 Tag 长度都会影响实际安全。

### 5.10 HMAC 的使用边界

- 密钥必须来自安全随机数或 KDF，不能直接使用低熵口令；口令应先经过专用 Password KDF。
- 同一密钥不要跨协议、跨方向复用；使用 HKDF 的 `info`/label 做用途隔离。
- 截断 Tag 会降低伪造难度；只能按协议规定截断并评估尝试次数。
- MAC 输入必须采用无歧义编码。`a || bc` 与 `ab || c` 不能产生相同字节串解释。
- HMAC 适合 API 请求签名、Webhook、HKDF、TLS 1.2 PRF 和 TLS 1.3 Finished 等场景。
- 生产代码应直接使用标准库 HMAC 接口，不要手工实现 SHA-256 常量、Padding 或 `ipad/opad` 拼接。

### 5.11 CBC-MAC 与 CMAC：怎样用分组密码认证消息

HMAC 从哈希函数构造 MAC；CBC-MAC 和 CMAC 则从分组密码构造 MAC。设分组密码的分组长度为 $n$，把消息分成完整分组 $M_1,\ldots,M_m$。最基本的 CBC-MAC 从全零状态开始：

$$
X_0=0^n
$$

$$
X_i=E_K(X_{i-1}\oplus M_i)
$$

最终：

$$
T=X_m
$$

这个结构看起来与 CBC 加密相似，但目的不同：

- CBC 加密需要按具体协议满足不可预测性等 IV 要求，并输出所有密文块；
- 基础 CBC-MAC 固定使用零初始状态，只输出最后一个链值；
- CBC-MAC 不加密消息，不能把“CBC 加密模式”和“CBC-MAC”当成同一个接口。

**裸 CBC-MAC 只适合协议预先固定且区分消息长度的输入域。**若允许攻击者混用不同长度消息，就可能发生拼接伪造。设：

$$
T=\operatorname{CBCMAC}_K(M)
$$

攻击者再查询一个单块消息 $X$，得到：

$$
T_X=E_K(X)
$$

则消息：

$$
M\parallel(X\oplus T)
$$

从状态 $T$ 继续计算时，下一轮输入变成：

$$
T\oplus(X\oplus T)=X
$$

所以其最终 Tag 正好是已知的 $T_X$。这说明“简单补零后直接做 CBC-MAC”并不能安全认证任意变长消息。

CMAC 修正了末块和长度边界。以 AES-CMAC 为例，先计算：

$$
L=E_K(0^{128})
$$

再在 $GF(2^{128})$ 中连续倍增得到子密钥 $K_1,K_2$。若最后一块恰好完整，就与 $K_1$ 异或；若不完整，则按 `10*` 规则填充后与 $K_2$ 异或。经过处理的最后一块再进入 CBC 链，最终链值作为 Tag。

直觉上，$K_1/K_2$ 把“完整末块”和“填充末块”放进不同的认证域，攻击者不能再把一个消息的末尾链值无缝解释成另一条变长消息的中间状态。生产实现必须直接使用标准 CMAC API，并遵守协议规定的 Tag 长度，不能自行实现倍增常量或末块规则。

### 5.12 Poly1305：一次性多项式认证器

Poly1305 使用 256 bit 一次性密钥：

$$
(r,s)
$$

其中 $r$ 和 $s$ 各 128 bit；$r$ 还要按规范清除特定位，这一步通常称为 `clamp`。令：

$$
p=2^{130}-5
$$

把消息切成最多 16 字节的小端分组。每个分组在有效字节之后附加一个 `1` bit，解释为整数 $n_i$。从：

$$
a_0=0
$$

开始迭代：

$$
a_i=(a_{i-1}+n_i)r\bmod p
$$

最终 Tag 为：

$$
T=(a_m+s)\bmod2^{128}
$$

可以把 $r$ 理解为多项式求值点，把 $s$ 理解为隐藏最终代数结果的一次性掩码。Poly1305 的安全边界与 HMAC 不同：**同一个 $(r,s)$ 绝不能用于两条不同消息**。重复一次性密钥会暴露关于 $r$ 的代数关系，从而破坏认证安全。

在 ChaCha20-Poly1305 AEAD 中，调用方并不直接管理 $(r,s)$：

1. 用会话密钥、当前 Nonce 和 ChaCha20 的 counter 0 生成密钥流块；
2. 取前 32 字节作为本条消息的一次性 Poly1305 密钥；
3. 从 counter 1 开始生成真正用于加密明文的密钥流；
4. 对以下规范编码进行 Poly1305 认证：

```text
AAD || pad16(AAD) ||
Ciphertext || pad16(Ciphertext) ||
le64(len(AAD)) || le64(len(Ciphertext))
```

因此，同一 ChaCha20-Poly1305 会话密钥下重复 Nonce，不仅会重复加密密钥流，也会重复 Poly1305 一次性密钥。唯一 Nonce 是整个 AEAD 构造的安全前提。

### 5.13 CCM：CBC-MAC 与 CTR 的组合

CCM（Counter with CBC-MAC）也是 AEAD，但内部结构与 GCM 不同：

1. 把 Nonce、Tag 长度、消息长度、AAD 是否存在等信息编码进首块 $B_0$；
2. 对格式化后的 $B_0$、AAD 和**明文**运行 CBC-MAC；
3. 用 CTR 模式加密明文；
4. 用 counter 0 产生的密钥流块 $S_0$ 掩码 CBC-MAC 结果，形成最终 Tag；
5. 接收端先完成解密与认证计算，但只有 Tag 验证成功才能向应用释放明文。

可以简化表示为：

$$
T_{\text{raw}}
=
\operatorname{CBCMAC}_K
(\operatorname{Format}(N,A,P))
$$

$$
C=\operatorname{CTR}_K(N,P)
$$

$$
T=\operatorname{MSB}_t(T_{\text{raw}}\oplus S_0)
$$

CCM 通过严格格式化解决裸 CBC-MAC 的变长消息问题，并把 Nonce、AAD、消息长度和 Tag 长度绑定进认证输入。它不是“历史 TLS CBC 加密再附加 HMAC”，也不是 CMAC：三者的输入格式、密钥使用和安全证明都不同。

TLS 1.3 定义了 AES-CCM 和 AES-CCM-8 套件。后者使用 64 bit Tag，带宽更省，但在相同攻击尝试次数下伪造余量显著低于 128 bit Tag，因此必须严格遵守协议的数据量和失败尝试限制。

---

## 六、GHASH 与 AES-GCM

### 6.1 GHASH 不是独立 MAC

GHASH 是由 128 bit 子密钥 $H$ 参数化的通用哈希：

$$
\operatorname{GHASH}_H(X_1,\ldots,X_m)
$$

它是线性的、几乎通用的哈希族，不具备 HMAC 那种独立 PRF/MAC 接口的安全语义。在 GCM 中，GHASH 的结果还要与 AES 生成的秘密掩码结合，才形成最终 Tag。

如果把同一个 $H$ 直接暴露给攻击者并把裸 GHASH 当 MAC，线性结构会允许构造伪造关系。

GMAC 也不能与裸 GHASH 混为一谈。GMAC 是 GCM 的“只认证、不加密”特例：明文为空，AAD 进入 GHASH，最终结果仍由唯一 Nonce 和 $E_K(J_0)$ 掩码保护。它继承 GCM 的 Nonce 唯一性与密钥使用量约束。

### 6.2 $GF(2^{128})$

GHASH 把 128 bit 串解释为次数小于 128 的二进制多项式。加法是 XOR：

$$
A+B=A\oplus B
$$

乘法是无进位多项式乘法，再模：

$$
P(x)=x^{128}+x^7+x^2+x+1
$$

与普通整数乘法不同，这里的系数在 $GF(2)$ 中：

$$
1+1=0
$$

因此部分积相加不产生进位，只做 XOR。

实现中最容易错的是“bit 串与多项式系数的映射方向”。GCM 规范定义了固定的位序和约简方法；不能仅凭普通大端整数乘法直觉实现。现代 CPU 常用 CLMUL/PMULL 指令加速无进位乘法。

### 6.3 Horner 累加

令：

$$
Y_0=0
$$

对每个 128 bit 输入块：

$$
Y_i=(Y_{i-1}\oplus X_i)\cdot H
$$

最终：

$$
\operatorname{GHASH}_H(X_1,\ldots,X_m)=Y_m
$$

展开可得：

$$
Y_m=X_1H^m\oplus X_2H^{m-1}\oplus\cdots\oplus X_mH
$$

这个多项式结构让不同消息在随机未知 $H$ 下发生碰撞的概率可控，但它仍是线性的，所以必须放在 GCM 的完整构造中理解。

### 6.4 GCM 的完整组成

GCM 可以概括为：

```text
AES-CTR              → 加密明文
GHASH(AAD, Ciphertext) → 认证 AAD 与密文
AES_K(J0)            → 掩码 GHASH 结果
```

#### 第一步：哈希子密钥

$$
H=E_K(0^{128})
$$

$H$ 必须保密，并且只与当前 AES 密钥一起使用。

#### 第二步：构造初始计数器 $J_0$

若 Nonce $N$ 恰好为 96 bit：

$$
J_0=N\parallel 0^{31}\parallel1
$$

这是 GCM 的高效标准路径。

若 Nonce 长度不是 96 bit，则按规范对 Nonce 做 GHASH，并编码长度得到 $J_0$。这不是简单补零。

#### 第三步：CTR 加密

对连续计数器：

$$
C=\operatorname{GCTR}_K(\operatorname{inc}_{32}(J_0),P)
$$

更直观地：

$$
C_i=P_i\oplus E_K(\operatorname{inc}_{32}^{\,i}(J_0))
$$

计数器只递增低 32 bit，并受单次调用块数限制。

#### 第四步：认证 AAD 与密文

GHASH 输入为：

$$
A\parallel\operatorname{pad}(A)
\parallel C\parallel\operatorname{pad}(C)
\parallel[|A|]_{64}\parallel[|C|]_{64}
$$

长度按 bit 编码。长度块防止不同分段方式产生相同多项式解释。

令：

$$
S=\operatorname{GHASH}_H(A,C)
$$

最终 Tag：

$$
T=\operatorname{MSB}_t(E_K(J_0)\oplus S)
$$

其中 $t$ 是协议选定的 Tag bit 长度。`AES_K(J0)` 隐藏 GHASH 的线性输出。

### 6.5 AAD 为什么重要

AAD 不需要保密，但必须与密文一起认证。适合放入 AAD 的信息包括：

- 协议版本和内容类型；
- 记录序号；
- 路由头、消息类型或不可篡改元数据。

如果攻击者能修改“如何解释密文”的头部字段，即使正文密文没有变化，也可能改变业务语义。AAD 把这些上下文绑定到同一个 Tag。

### 6.6 Nonce 重复为何是灾难

同一密钥和 Nonce 会复用 CTR 密钥流：

$$
C_1\oplus C_2=P_1\oplus P_2
$$

同时，两个 Tag 之间的关系会消去相同的 $E_K(J_0)$ 掩码，泄露关于 GHASH 子密钥 $H$ 的代数约束；在足够条件下攻击者可恢复认证结构并伪造消息。

因此，GCM 最重要的规则不是“Nonce 必须随机”，而是：

> 同一 AES-GCM 密钥下，Nonce 必须唯一。

计数器、序列号或固定 IV 与序列号组合通常比“每次随机 96 bit 并祈祷不碰撞”更容易给出确定性唯一保证。

### 6.7 HMAC、CMAC、Poly1305、GHASH 与 GMAC 的边界

|机制|底层原语|定位|能否直接作为协议 MAC|关键红线|
|---|---|---|---|---|
|HMAC-SHA256|哈希函数|通用独立 MAC|可以|密钥质量、用途隔离、Tag 长度、无歧义编码|
|AES-CMAC|分组密码|通用独立 MAC|可以|标准末块处理、Tag 长度和密钥隔离|
|Poly1305|模 $2^{130}-5$ 的多项式认证|一次性密钥 MAC|可以，但每条消息必须使用新的 $(r,s)$|一次性密钥绝不复用|
|GHASH| $GF(2^{128})$ 多项式累加|GCM 内部通用哈希组件|不应脱离完整 GCM 构造|线性输出不能裸露，输入必须包含长度编码|
|GMAC|GCM 的只认证模式|带 Nonce 的独立认证接口|可以，必须作为完整 GMAC 使用|同一 AES Key 下 Nonce 唯一，并遵守数据量限制|

“都使用多项式运算”不代表 Poly1305 和 GHASH 可以互换。Poly1305 依赖一次性秘密 $(r,s)$；GHASH 的 $H=E_K(0^{128})$ 在同一 GCM 密钥期间保持不变，安全性依赖每条消息使用不同的 $E_K(J_0)$ 掩码。二者的域、编码、密钥生命周期和安全证明均不同。

在经分析的 Encrypt-then-MAC 协议中，可以用独立 HMAC 或 CMAC 认证密文；现代 TLS 1.3 则直接使用标准 AEAD 套件，避免应用自行决定加密、Padding 与 MAC 的组合顺序。

---

## 七、DH：在公开信道上建立共享秘密

### 7.1 有限域 DH 的数学结构

选择一个循环群 $G=\langle g\rangle$，其阶为大素数 $q$。经典有限域 DH 常在模素数 $p$ 的乘法群或其大素数阶子群中工作。

公开参数：

$$
p,\ q,\ g
$$

其中 $g$ 生成选定子群。安全参数不应由应用临时自创，应使用标准命名组并遵守其公钥验证规则。

Alice 随机选择私钥：

$$
a\in\{1,\ldots,q-1\}
$$

计算公钥：

$$
A=g^a\bmod p
$$

Bob 选择：

$$
b\in\{1,\ldots,q-1\}
$$

计算：

$$
B=g^b\bmod p
$$

交换 $A,B$ 后：

$$
Z_A=B^a\bmod p
=(g^b)^a
=g^{ab}\bmod p
$$

$$
Z_B=A^b\bmod p
=(g^a)^b
=g^{ab}\bmod p
$$

双方得到相同共享秘密 $Z$。

### 7.2 为什么旁观者不能直接得到共享秘密

旁观者知道：

$$
p,g,A=g^a,B=g^b
$$

但不知道 $a,b$。从 $g^a$ 求 $a$ 是离散对数问题；直接从 $g^a,g^b$ 求 $g^{ab}$ 对应计算 Diffie-Hellman 问题。

“困难”依赖群参数、子群阶、实现和攻击模型。太小的参数、非标准群、未验证公钥或侧信道泄露都会绕过数学困难性。

对使用已知素数阶子群的有限域 DH，验证通常至少包括范围检查，并按组规范确认公钥属于目标子群，例如检查：

$$
2\le Y\le p-2,\qquad Y^q\equiv1\pmod p
$$

具体标准组可能通过参数结构简化部分检查，必须遵循协议规范，不能机械套用一条通用条件。

### 7.3 DH 不提供身份认证

裸 DH 无法区分“对方”和中间人。Mallory 可以分别与 Alice、Bob 做两次 DH：

```text
Alice  ← DH →  Mallory  ← DH →  Bob
```

Alice 和 Bob 各自得到的密钥都与 Mallory 共享，而不是彼此共享。

解决办法是把 DH 公钥和握手 transcript 绑定到认证机制：

- 证书私钥签名；
- 预共享密钥；
- 已认证的公钥；
- 密码认证密钥交换协议。

TLS 1.3 服务器通常用证书对应私钥签署握手上下文，从而认证临时 ECDHE 交换。

### 7.4 从共享秘密到会话密钥

不能直接把整数 $Z$ 截成 AES Key。应把规范编码后的共享秘密送入 KDF：

$$
\text{PRK}=\operatorname{HKDF\text{-}Extract}(\text{salt},Z)
$$

$$
\text{keying material}
=\operatorname{HKDF\text{-}Expand}(\text{PRK},\text{context},L)
$$

KDF 的作用包括：

- 提取接近均匀的密钥材料；
- 绑定协议 transcript 和上下文；
- 派生客户端/服务器、加密/IV/Finished 等不同用途的密钥；
- 防止一个用途的密钥被直接复用到另一个用途。

### 7.5 Ephemeral DH 与前向安全

若每次会话生成新的临时私钥 $a,b$，并在会话后安全销毁，即使未来证书长期私钥泄露，攻击者也不能仅凭历史录包恢复过去的 $g^{ab}$。这就是前向安全的核心。

它依赖：

- 临时私钥确实随机且没有复用；
- 会话密钥和临时秘密已销毁；
- 当时握手未被实时中间人攻破；
- 端点没有同时记录明文或会话密钥。

前向安全不是“长期私钥泄露后系统仍然一切安全”；攻击者仍可冒充未来会话，除非证书被撤销和替换。

---

## 八、RSA：模幂的陷门置换

### 8.1 密钥生成

1. 随机选择大素数 $p,q$，且 $p\ne q$。
2. 计算：

$$
n=pq
$$

3. 计算 Carmichael 函数：

$$
\lambda(n)=\operatorname{lcm}(p-1,q-1)
$$

4. 选择公开指数 $e$，满足：

$$
1<e<\lambda(n),\qquad \gcd(e,\lambda(n))=1
$$

工程中常用：

$$
e=65537
$$

5. 计算私有指数：

$$
d\equiv e^{-1}\pmod{\lambda(n)}
$$

于是：

$$
ed=1+k\lambda(n)
$$

公钥是 $(n,e)$，私钥至少包含 $d$，高性能实现还保存 $p,q$ 和 CRT 参数。

### 8.2 为什么可以解密

教科书 RSA：

$$
c=m^e\bmod n
$$

$$
m'=c^d\bmod n=m^{ed}\bmod n
$$

因为：

$$
ed=1+k\lambda(n)
$$

对任意 $m$，可分别模 $p$ 和模 $q$ 证明：

$$
m^{ed}\equiv m\pmod p
$$

$$
m^{ed}\equiv m\pmod q
$$

即使 $m$ 与 $n$ 不互素，分别考虑 $m\equiv0$ 和 $m\not\equiv0$ 也成立。由中国剩余定理：

$$
m^{ed}\equiv m\pmod n
$$

所以解密恢复原消息代表元。

### 8.3 CRT 为什么加速私钥运算

直接模 $n$ 做大指数运算很昂贵。私钥端可计算：

$$
d_p=d\bmod(p-1),\qquad
d_q=d\bmod(q-1)
$$

$$
m_p=c^{d_p}\bmod p,\qquad
m_q=c^{d_q}\bmod q
$$

再用 CRT 合并为模 $n$ 的结果。两个约一半位宽的模幂通常比一个完整位宽模幂快得多。

CRT 实现必须防故障攻击：若攻击者诱导一次模 $p$ 或模 $q$ 的错误结果，可能通过正确/错误签名之差分解 $n$。工程上会做结果校验、指数盲化和常量时间实现。

### 8.4 “加密”和“签名”不能只说成指数方向相反

教科书公式看似对称：

$$
\text{公钥指数 }e,\qquad \text{私钥指数 }d
$$

但安全编码完全不同：

- RSA 加密使用 RSA-OAEP；
- RSA 签名使用 RSA-PSS；
- 旧的 PKCS #1 v1.5 编码只应在协议兼容和严格实现条件下使用；
- 裸 $m^e\bmod n$ 或裸 $h^d\bmod n$ 是确定性、可塑且不安全的。

RSA-OAEP 对消息做随机化编码，提供公钥加密所需的语义安全性质。RSA-PSS 把消息哈希、随机 salt 和结构化编码结合，提供签名安全。

不能拿“私钥加密、公钥解密”定义数字签名：签名要绑定消息编码、哈希算法和验证规则，并且不以保密为目标。

### 8.5 RSA-OAEP：加密前怎样编码消息

设：

- RSA 模数编码长度为 $k$ 字节；
- 哈希输出长度为 $hLen$ 字节；
- $L$ 是可选 Label，通常为空但仍要参与哈希；
- `MGF1` 是基于同一哈希的掩码生成函数。

OAEP 能编码的消息长度满足：

$$
|M|\le k-2hLen-2
$$

编码过程如下。

1. 计算 Label 哈希：

$$
lHash=H(L)
$$

2. 构造数据块：

$$
DB=lHash\parallel PS\parallel\texttt{0x01}\parallel M
$$

其中 $PS$ 是长度恰好使 $DB$ 达到 $k-hLen-1$ 字节的全零串。

3. 随机生成 $hLen$ 字节的：

$$
seed
$$

4. 用 `MGF1` 交叉掩码：

$$
dbMask=\operatorname{MGF1}(seed,k-hLen-1)
$$

$$
maskedDB=DB\oplus dbMask
$$

$$
seedMask=\operatorname{MGF1}(maskedDB,hLen)
$$

$$
maskedSeed=seed\oplus seedMask
$$

5. 得到编码消息：

$$
EM=\texttt{0x00}\parallel maskedSeed\parallel maskedDB
$$

把 $EM$ 按大端解释为整数 $m$，再执行 RSA 公钥运算：

$$
c=m^e\bmod n
$$

随机 `seed` 使同一个明文在同一公钥下多次加密得到不同密文；`lHash` 把 Label 绑定进编码；两次 `MGF1` 让种子和数据块彼此掩码，避免暴露确定性结构。

解密端先执行私钥运算恢复 $EM$，再反向解除掩码，并统一检查：

- 首字节是否为 `0x00`；
- 恢复的 `lHash` 是否等于 $H(L)$；
- 零填充后是否存在规范的 `0x01` 分隔符；
- 消息长度和整数范围是否合法。

所有失败都必须以不可区分的方式返回，不能泄露“哪一项检查失败”或明显的时序差异，否则会重新形成解密 Oracle。OAEP 是完整编码方案，不能只做其中一次哈希或只添加随机前缀。

### 8.6 RSA-PSS：签名前怎样编码摘要

PSS 是概率化签名编码。设消息摘要为：

$$
mHash=H(M)
$$

若 RSA 模数长度为 `modBits`，PSS 使用：

$$
emBits=modBits-1,\qquad
emLen=\left\lceil\frac{emBits}{8}\right\rceil
$$

签名端生成协议规定长度的随机 `salt`，构造：

$$
M'=
\underbrace{\texttt{0x00}\ldots\texttt{0x00}}_{8\text{ bytes}}
\parallel mHash
\parallel salt
$$

并计算：

$$
H'=H(M')
$$

随后构造：

$$
DB=PS\parallel\texttt{0x01}\parallel salt
$$

用 `MGF1` 掩码：

$$
dbMask=\operatorname{MGF1}(H',emLen-hLen-1)
$$

$$
maskedDB=DB\oplus dbMask
$$

还要按模数 bit 长度清零 `maskedDB` 最左侧未使用的高位，最后得到：

$$
EM=maskedDB\parallel H'\parallel\texttt{0xbc}
$$

将 $EM$ 转为整数 $m=\operatorname{OS2IP}(EM)$ 后执行 RSA 私钥运算：

$$
s=m^d\bmod n
$$

验证端不是“用公钥把签名解密后看起来像哈希就通过”，而是：

1. 检查签名整数范围；
2. 计算 $m=s^e\bmod n$，再用固定长度 `I2OSP` 恢复 $EM$；
3. 检查末尾 `0xbc`、未使用高位和 `DB` 格式；
4. 恢复 `salt`；
5. 用收到的消息重新计算 $mHash$ 和 $H'$；
6. 常量时间比较编码中的 $H'$ 与重新计算值。

PSS 的随机 `salt` 与结构化编码阻止教科书 RSA 签名的确定性和代数可塑问题。`salt` 长度、哈希和 `MGF1` 参数都是签名方案的一部分；验证端必须按协议和证书算法标识执行，不能自行猜测参数。某些标准允许确定性或特定长度的 salt，但通信双方必须使用同一套明确规则。

OAEP 与 PSS 都使用 `MGF1`，但二者不能互换：

- OAEP 服务于**公钥加密**，核心目标是隐藏明文；
- PSS 服务于**数字签名**，核心目标是公开验证来源和完整性；
- 两者的编码布局、随机量、安全目标和错误处理完全不同。

### 8.7 RSA 在 TLS 中的角色变化

历史 TLS 可使用静态 RSA 密钥传输：客户端生成 Pre-Master Secret，用服务器 RSA 公钥加密。这种方式：

- 没有前向安全；
- 解密错误处理曾导致 Bleichenbacher 类 Oracle；
- 证书私钥泄露后，历史录包可能被解密。

TLS 1.3 删除了 RSA 密钥传输。RSA 证书仍可用于 RSA-PSS 签名，认证 ECDHE 握手，但不再负责加密会话秘密。

---

## 九、ECC：用点的标量乘法构造公钥密码

### 9.1 素域曲线

常见短 Weierstrass 曲线定义在素域 $GF(p)$：

$$
y^2\equiv x^3+ax+b\pmod p
$$

并要求：

$$
4a^3+27b^2\not\equiv0\pmod p
$$

以避免奇异曲线。曲线点集合再加无穷远点 $\mathcal O$，配合点加法形成有限阿贝尔群。

这里有两个不同的模数：

- $p$：坐标所在有限域的模数；
- $n$：基点 $G$ 的阶，私钥和 ECDSA 标量通常模 $n$。

把 $p$ 和 $n$ 混用会直接破坏算法。

### 9.2 域中的加减乘除

所有坐标运算都模 $p$：

$$
(a+b)\bmod p,\quad
(a-b)\bmod p,\quad
(ab)\bmod p
$$

除法通过逆元：

$$
\frac ab\bmod p
=a\cdot b^{-1}\bmod p
$$

其中 $b\ne0$。因为 $p$ 为素数，每个非零元素都有逆元。

实际高性能实现常用射影坐标避免每次点加法都做昂贵的模逆，只在最后转回仿射坐标。

### 9.3 点加与倍点

设：

$$
P=(x_1,y_1),\qquad Q=(x_2,y_2)
$$

若 $P\ne Q$，斜率：

$$
\lambda=(y_2-y_1)(x_2-x_1)^{-1}\pmod p
$$

若 $P=Q$，倍点斜率：

$$
\lambda=(3x_1^2+a)(2y_1)^{-1}\pmod p
$$

结果 $R=P+Q=(x_3,y_3)$：

$$
x_3=\lambda^2-x_1-x_2\pmod p
$$

$$
y_3=\lambda(x_1-x_3)-y_1\pmod p
$$

特殊情况包括：

- $P+\mathcal O=P$；
- $P+(-P)=\mathcal O$；
- 分母为零时必须按群规则处理，不能直接求逆。

几何“直线与曲线第三交点再关于 x 轴对称”只是实数曲线上的直觉；密码学计算实际发生在有限域离散点集上。

### 9.4 标量乘法

ECC 的核心操作：

$$
Q=[d]G
$$

表示把 $G$ 加 $d$ 次。实际使用 double-and-add、窗口法或固定基点预计算，不会循环执行 $d$ 次。

正向从 $d,G$ 计算 $Q$ 高效；已知 $G,Q$ 求 $d$ 是椭圆曲线离散对数问题。ECC 在相近经典安全级别下通常使用比有限域 DH/RSA 更短的密钥。

实现必须抵抗侧信道：

- 标量乘法流程不能随私钥 bit 明显分支；
- 查表要常量时间；
- 私钥和随机 nonce 不能泄露；
- 接收点必须按协议验证。

### 9.5 ECDH

Alice：

$$
Q_A=[d_A]G
$$

Bob：

$$
Q_B=[d_B]G
$$

Alice 计算：

$$
Z_A=[d_A]Q_B=[d_Ad_B]G
$$

Bob 计算：

$$
Z_B=[d_B]Q_A=[d_Bd_A]G
$$

因此得到同一个曲线点。

协议通常提取共享点坐标的规范编码，再送入 KDF，而不是把点直接当 AES Key。还必须执行具体曲线/协议要求的公钥解析、曲线成员、无穷远点、小子群或低阶点检查。

X25519 使用 Montgomery 曲线和专门的标量/编码规则，不能把短 Weierstrass 曲线公式机械套用；它也有必须检查全零共享结果等协议边界。

### 9.6 ECDSA 签名

设基点 $G$ 的阶为素数 $n$，私钥：

$$
d\in[1,n-1]
$$

公钥：

$$
Q=[d]G
$$

对消息 $M$：

1. 计算消息哈希，并按标准规则取与 $n$ bit 长度匹配的左侧部分，得到整数 $z$。
2. 生成每条消息唯一且不可预测的 nonce：

$$
k\in[1,n-1]
$$

3. 计算：

$$
R=[k]G
$$

4. 取：

$$
r=x_R\bmod n
$$

若 $r=0$，重新选择 $k$。

5. 计算：

$$
s=k^{-1}(z+rd)\bmod n
$$

若 $s=0$，重新选择 $k$。

签名为 $(r,s)$。

### 9.7 ECDSA 验签为何成立

验证者先检查：

$$
1\le r,s\le n-1
$$

再计算：

$$
w=s^{-1}\bmod n
$$

$$
u_1=zw\bmod n,\qquad
u_2=rw\bmod n
$$

$$
X=[u_1]G+[u_2]Q
$$

因为 $Q=[d]G$：

$$
X=[w(z+rd)]G
$$

而签名公式给出：

$$
s=k^{-1}(z+rd)\pmod n
$$

所以：

$$
w(z+rd)=s^{-1}(z+rd)\equiv k\pmod n
$$

因此：

$$
X=[k]G=R
$$

若：

$$
x_X\bmod n=r
$$

则签名通过。

验签还必须拒绝无穷远点、非法编码和不满足曲线/子群规则的公钥。

### 9.8 ECDSA 的 nonce 红线

由：

$$
s=k^{-1}(z+rd)\pmod n
$$

可得：

$$
d=(sk-z)r^{-1}\pmod n
$$

所以知道 $k$ 就知道私钥。

如果同一私钥对两条消息重复使用 $k$：

$$
s_1=k^{-1}(z_1+rd)
$$

$$
s_2=k^{-1}(z_2+rd)
$$

相减：

$$
k=(z_1-z_2)(s_1-s_2)^{-1}\pmod n
$$

随后即可恢复 $d$。

生产实现应使用可靠 CSPRNG，或使用 RFC 6979 等确定性 nonce 生成方法；确定性不代表使用常数 nonce，而是用私钥和消息哈希通过安全过程为每条消息生成不同的 $k$。

### 9.9 ECDSA 的可塑性与编码

若 $(r,s)$ 是有效 ECDSA 签名，则通常：

$$
(r,n-s)
$$

也能通过同一消息和公钥的验证，因为两个 $s$ 互为模 $n$ 的相反数。这称为**签名可塑性**：攻击者不需要私钥，也可能把一个有效签名改写成另一个有效签名。

它不会直接泄露私钥，但会破坏“签名字节串就是唯一交易标识”之类的上层假设。某些协议因此要求规范化为：

$$
s\le\frac n2
$$

也就是 low-$s$ 形式。low-$s$ 不是 ECDSA 数学验签的普遍要求，是否强制必须由具体协议规定。

编码同样属于安全边界。ECDSA 常把 $(r,s)$ 编码为 ASN.1 DER 序列，而某些协议使用两个固定宽度整数的直接拼接。验证端必须只接受协议指定的规范编码，不能把“同一个整数的多种字节表示”都宽松接受。

### 9.10 EdDSA 不是“换一条曲线的 ECDSA”

Ed25519/Ed448 属于 EdDSA，使用 Edwards 曲线和与 ECDSA 不同的签名方程、点编码及 nonce 派生方法。EdDSA 通常从私钥材料和消息确定性地产生每条消息的 nonce，因此避免了“外部随机数生成器直接输出重复 $k$”这一类故障，但仍要求：

- 正确处理 Ed25519、Ed25519ctx、Ed25519ph 等不同模式；
- 按规范验证点编码、标量范围和小阶点相关条件；
- 不跨模式、跨协议误用同一签名输入；
- 保护私钥和确定性 nonce 计算免受侧信道与故障攻击。

因此，不能把前面的 ECDSA 公式套到 EdDSA，也不能因为 nonce 是确定性的就忽略实现安全。

### 9.11 DH、RSA 与 ECC 对比

|维度|有限域 DH|RSA|ECC|
|---|---|---|---|
|核心运算|模幂|模幂|点标量乘法|
|困难问题|离散对数/CDH|RSA 反演，与分解密切相关|椭圆曲线离散对数|
|密钥交换|DH/DHE|历史上可做密钥传输，不是 DH|ECDH/ECDHE|
|签名|通常配合 DSA 类算法|RSA-PSS|ECDSA/EdDSA|
|前向安全|临时 DHE 可提供|静态 RSA 密钥传输不提供|临时 ECDHE 可提供|
|工程重点|标准组、公钥验证|OAEP/PSS、CRT/盲化|曲线选择、点验证、nonce/侧信道|

这些算法的安全级别不能只比较“密钥 bit 数”。不同数学问题的攻击复杂度不同。

---

## 十、证书与 PKI：把公钥绑定到身份

### 10.1 为什么仅收到公钥还不够

攻击者可以替换网络中传输的公钥。证书要证明：

> 某个域名或主体的公钥，得到受信任签发者的签名背书。

X.509 证书主要包含：

- Subject / Subject Alternative Name；
- Subject Public Key Info；
- Issuer；
- Serial Number；
- Validity；
- Key Usage / Extended Key Usage；
- Basic Constraints；
- 签名算法与 CA 签名。

服务身份必须按应用协议从连接目标独立构造，再与 `subjectAltName` 中对应类型的标识匹配：域名使用 `dNSName`，直接连接固定 IP 时使用 `iPAddress`。RFC 9525 已明确要求不要用 Subject Common Name 中“看起来像域名”的自由文本识别服务。

### 10.2 证书链

典型链：

```text
服务器叶子证书
  ← 中间 CA 签名
中间 CA 证书
  ← 根 CA 签名
根 CA
  ← 预装在客户端信任库
```

根证书是本地信任锚，不是因为“自签名本身可信”，而是因为操作系统、浏览器或组织策略预先信任它。

服务器通常发送叶子证书和必要中间证书，不发送或不依赖网络中收到的根证书来建立信任。

### 10.3 客户端验证步骤

客户端至少要检查：

1. 证书链签名能否逐级验证到本地信任锚；
2. 当前时间是否在有效期内；
3. 独立得到的 DNS/IP 等 reference identifier 是否与 SAN 中同类型标识匹配；
4. Basic Constraints 是否允许中间证书作为 CA；
5. Key Usage / Extended Key Usage 是否允许当前用途；
6. 签名算法、密钥强度和策略是否可接受；
7. 是否存在未知 critical extension；
8. 吊销信息是否按客户端策略处理。

OCSP、CRL、OCSP Stapling、浏览器厂商维护的吊销列表等机制各有覆盖范围和可用性权衡。不能把“网络 OCSP 请求失败”简单等同为一定拒绝，也不能宣称证书吊销在所有客户端都实时、完整生效。

### 10.4 证书如何阻止 ECDHE 中间人

服务器不是只发送 ECDHE 公钥，而是用证书对应私钥签署包含握手上下文和 transcript hash 的结构。客户端：

1. 验证证书链和主机名；
2. 从证书取得服务器认证公钥；
3. 验证 `CertificateVerify`；
4. 验证 `Finished`。

攻击者即使能替换 ECDHE 公钥，也无法为被篡改的握手生成有效证书签名和 Finished。

证书公钥算法与密钥交换组可以不同。例如，含 RSA 公钥的服务器证书可配合 RSA-PSS `CertificateVerify`，认证本次使用 X25519 的临时密钥交换；证书链自身的签名算法还可以与叶子证书公钥算法不同。TLS 1.3 Cipher Suite 不负责选择这些证书或签名算法。

---

## 十一、TLS 1.3 握手：把交换、认证和密钥派生绑定起来

### 11.1 ClientHello

客户端发送的关键信息包括：

- 支持的 TLS 版本；
- AEAD/Hash Cipher Suites；
- 支持的密钥建立组；
- 一个或多个 `key_share`；经典组中可以是 ECDHE 或 FFDHE 公钥；
- 支持的签名算法；
- Server Name、ALPN 等扩展；
- 恢复会话时的 PSK identity、binder；
- 请求 0-RTT 时的 early data 指示。

客户端随机数不是临时私钥，也不是 AES Key。它通过 ClientHello 进入 transcript，从而影响派生结果；真正的秘密输入来自 PSK 和/或非对称密钥建立。

### 11.2 ServerHello 与共享秘密

服务器选择：

- TLS 1.3；
- Cipher Suite；
- 密钥交换组；
- 自己的 `key_share`。

在经典 ECDHE 路径中，双方计算：

$$
Z=\operatorname{ECDH}(d_{\text{local}},Q_{\text{peer}})
$$

在 FFDHE、KEM 或混合组中，共享秘密的计算与编码按该组规范执行。TLS Key Schedule 只接收规范化后的非对称共享秘密，不应假设它永远是某个椭圆曲线点的坐标。

客户端和服务器**不得跨连接复用同一个 KeyShare 私有值/公钥值**。即使某些经典 ECDHE 场景看似仍有数学安全余量，复用会损害前向安全、可关联性以及新型组的安全假设；RFC 9846 已明确禁止发送端这样做。

双方随后通过 TLS 1.3 Key Schedule 派生握手流量密钥。从 `ServerHello` 之后，常规握手消息已受握手密钥保护。

如果客户端没有提供服务器可接受的 `key_share`，服务器可发送 HelloRetryRequest，让客户端为指定组重发；该过程也被 transcript 特殊规则绑定，不能被当作握手外的无认证重试。

### 11.3 服务器认证消息

在基于证书的完整握手中，服务器在加密握手通道中发送：

1. `EncryptedExtensions`：确认 ALPN 等协商结果；
2. `Certificate`：证书链；
3. `CertificateVerify`：用证书私钥签署握手上下文；
4. `Finished`：用从握手密钥派生的 Finished Key 对 transcript hash 做 HMAC。

`CertificateVerify` 证明服务器持有证书私钥；`Finished` 证明发送方掌握本次握手秘密，并把之前全部握手消息绑定在一起。

客户端完成证书和签名验证后，也发送自己的 `Finished`。若要求双向 TLS，还会发送客户端证书与 `CertificateVerify`。

`CertificateVerify` 不是简单地“签 transcript hash”。其签名输入为：

```text
0x20 重复 64 次
|| context_string
|| 0x00
|| Transcript-Hash(截至本端 Certificate)
```

服务器与客户端分别使用：

```text
"TLS 1.3, server CertificateVerify"
"TLS 1.3, client CertificateVerify"
```

前导空格、角色字符串和分隔字节共同提供域分离，避免同一签名被解释到别的角色、协议或消息结构中。

### 11.4 Transcript Hash

TLS 不只认证单个公钥或单条消息，而是不断哈希按协议编码的握手消息：

$$
\operatorname{TranscriptHash}
=H(\text{Handshake}_1\parallel\cdots\parallel\text{Handshake}_n)
$$

Key Schedule、CertificateVerify、Finished 都在不同阶段绑定这个摘要。这防止攻击者无声修改：

- 版本；
- Cipher Suite；
- 密钥交换组与公钥；
- 扩展；
- 证书；
- 协议协商结果。

协议实现必须哈希准确的握手编码字节和正确阶段，不能用“解析后重新序列化的近似内容”代替。

这里每个 $\text{Handshake}_i$ 都包含完整的握手层编码：

```text
msg_type（1 byte） || length（uint24，3 bytes） || body
```

TLS 记录层的类型、版本、记录长度等 Record Header 不属于 transcript。因而，同一条握手消息被拆到多个 Record 中，或多条握手消息合并进一个 Record，都不应改变 Transcript Hash；真正参与哈希的是重组后的握手消息及其握手头。不同派生步骤还必须严格使用规范要求的截止位置，不能多算或少算一条握手消息。

若发生 HelloRetryRequest，第一次 ClientHello 的普通 transcript 状态会被一个合成的 `message_hash` 握手消息替换后再继续累计。这样既避免保存任意长的首个 ClientHello，又把重试前后的参数绑定到同一条经过认证的 transcript；不能把第二个 ClientHello 当作一次全新的握手。

### 11.5 握手完成后的状态

双方验证 Finished 后，派生独立的客户端和服务器应用流量秘密：

```text
client_application_traffic_secret_0
server_application_traffic_secret_0
```

再从每个 traffic secret 派生各自的 AEAD Key 和 IV。客户端写密钥对应服务器读密钥，反方向使用另一组材料。

这保证：

- 双向密钥隔离；
- 握手与应用数据密钥隔离；
- Key、IV、Finished、Exporter、Resumption 各用途隔离。

### 11.6 版本协商与降级保护

TLS 1.3 的真实版本通过 `supported_versions` 扩展协商，ClientHello 和 ServerHello 中的 `legacy_version` 为兼容旧中间盒固定使用 `0x0303`，不能据此判断线上实际版本。

防降级不是只靠某一个字段：

1. 客户端只接受自己在 `supported_versions` 中提供过的版本；
2. 版本、套件、组和扩展都进入 transcript，并由 `CertificateVerify`/`Finished` 认证；
3. 能支持 TLS 1.3 的服务器若实际协商到 TLS 1.2，会把 ServerHello.random 最后 8 字节设置为 `DOWNGRD` 加 `0x01`；TLS 1.3 客户端在旧版本 ServerHello 中看到该标记必须中止。

RFC 9846 已禁止协商 TLS 1.0 和 TLS 1.1。旧式“连接失败后悄悄再试更低版本”会绕开上述安全边界，不应由应用自行实现。

---

## 十二、TLS 1.3 Key Schedule 与 HKDF

### 12.1 HKDF-Extract

$$
\operatorname{PRK}
=\operatorname{HMAC}_{\text{salt}}(\operatorname{IKM})
$$

Extract 把可能分布不规则的输入密钥材料 IKM 提取为固定长度伪随机密钥 PRK。salt 不要求保密，但应与 IKM 独立；若未提供，HKDF 按一个 `Hash.length` 长度的全零 salt 处理。HKDF 不是 Password KDF，不能靠它把低熵口令“变成”高熵密钥。

### 12.2 HKDF-Expand

标准 HKDF Expand：

$$
T(0)=\epsilon
$$

$$
T(i)
=\operatorname{HMAC}_{\text{PRK}}
(T(i-1)\parallel\text{info}\parallel i)
$$

$$
\operatorname{OKM}
=T(1)\parallel T(2)\parallel\cdots
$$

截取所需长度。

TLS 1.3 使用：

$$
\operatorname{HKDF\text{-}Expand\text{-}Label}
(\text{Secret},\text{Label},\text{Context},L)
$$

它把长度、`"tls13 "` 前缀、Label 和 Context 编码到 `info`，防止不同协议或用途生成相同输出。

TLS 还定义：

$$
\operatorname{Derive\text{-}Secret}
(S,\text{Label},\text{Messages})
=
\operatorname{HKDF\text{-}Expand\text{-}Label}
(S,\text{Label},H(\text{Messages}),\operatorname{Hash.length})
$$

所以 `Derive-Secret(..., "derived", "")` 的 Context 是**空握手消息串的哈希** $H(\epsilon)$，不是零长度 Context。只有直接调用 `HKDF-Expand-Label(..., "", ...)` 时，`""` 才表示零长度 Context。混淆两者会得到完全不同的密钥。

### 12.3 三阶段秘密

#### Early Secret

$$
\text{early\_secret}
=\operatorname{HKDF\text{-}Extract}(0^{h},\text{PSK})
$$

其中 $h=\operatorname{Hash.length}$（按字节计）。没有 PSK 时，PSK 也取 $0^h$，因此仍执行：

$$
\operatorname{HKDF\text{-}Extract}(0^h,0^h)
$$

而不是跳过 Early Secret 阶段。

Early Secret 可派生：

- PSK binder key；
- 0-RTT 客户端 early traffic secret；
- early exporter secret。

#### Handshake Secret

先派生过渡秘密：

$$
\text{derived}
=\operatorname{Derive\text{-}Secret}
(\text{early\_secret},"derived","")
$$

再混入规范编码的非对称共享秘密：

$$
\text{handshake\_secret}
=\operatorname{HKDF\text{-}Extract}
(\text{derived},Z_{\text{asym}})
$$

在经典路径中，$Z_{\text{asym}}$ 可以来自 ECDHE 或 FFDHE；其他组按各自规范生成固定、无歧义的字节串。随后绑定从 ClientHello 到 ServerHello 的 transcript：

$$
\text{c\_hs\_traffic}
=\operatorname{Derive\text{-}Secret}
(\text{handshake\_secret},"c hs traffic",
\text{ClientHello}\ldots\text{ServerHello})
$$

$$
\text{s\_hs\_traffic}
=\operatorname{Derive\text{-}Secret}
(\text{handshake\_secret},"s hs traffic",
\text{ClientHello}\ldots\text{ServerHello})
$$

#### Main Secret（旧称 Master Secret）

再次通过 `"derived"` 隔离阶段：

$$
\text{derived}_2
=\operatorname{Derive\text{-}Secret}
(\text{handshake\_secret},"derived","")
$$

$$
\text{main\_secret}
=\operatorname{HKDF\text{-}Extract}(\text{derived}_2,0^h)
$$

RFC 9846 将正文术语改为 Main Secret，但为保持兼容，HKDF label 中仍保留 `"exp master"`、`"res master"` 等既有字节串。不能因为文档术语变化而修改线上 label。

不同叶子秘密绑定的 transcript 截止点不同：

$$
\begin{aligned}
\text{c\_ap\_traffic}_0
&=\operatorname{Derive\text{-}Secret}
(\text{main\_secret},"c ap traffic",
\text{CH}\ldots\text{server Finished})\\
\text{s\_ap\_traffic}_0
&=\operatorname{Derive\text{-}Secret}
(\text{main\_secret},"s ap traffic",
\text{CH}\ldots\text{server Finished})\\
\text{exporter\_secret}
&=\operatorname{Derive\text{-}Secret}
(\text{main\_secret},"exp master",
\text{CH}\ldots\text{server Finished})\\
\text{resumption\_secret}
&=\operatorname{Derive\text{-}Secret}
(\text{main\_secret},"res master",
\text{CH}\ldots\text{client Finished})
\end{aligned}
$$

也就是说，应用流量秘密和 exporter secret 截止到服务器 Finished；恢复秘密还要绑定客户端 Finished。把它们笼统写成同一个“完整 transcript”会掩盖重要的状态机边界。

### 12.4 Finished

在主握手中，`base_key` 是发送方对应方向的 handshake traffic secret：

$$
\text{finished\_key}
=\operatorname{HKDF\text{-}Expand\text{-}Label}
(\text{base\_key},"finished","",\operatorname{Hash.length})
$$

$$
\text{verify\_data}
=\operatorname{HMAC}_{\text{finished\_key}}
(\operatorname{TranscriptHash}(\text{截至本条 Finished 之前}))
$$

Finished 本身不进入自己的输入，但会进入后续派生或对端 Finished 所使用的 transcript。PSK binder 复用相同的“两步 Finished 结构”，只是其 base key 是从 PSK 派生的 binder key。

因此 HMAC 在 TLS 1.3 中仍然重要：Cipher Suite 中的 Hash 既服务于 HKDF，也服务于 transcript 和 Finished。

### 12.5 为什么分阶段、分方向

如果所有用途都直接截取同一个共享秘密：

- 一个方向的密钥泄露会直接影响另一个方向；
- 握手密钥泄露可能等同于应用密钥泄露；
- 不同协议字段可能意外使用相同字节；
- KeyUpdate 和恢复会话难以建立清晰边界。

TLS 1.3 用 label、transcript 和阶段性 Extract/Expand 做域分离，使每把密钥都绑定到明确用途和握手上下文。

---

## 十三、TLS 1.3 记录层如何使用 AES-GCM

### 13.1 从 Traffic Secret 派生 Key 和 IV

对每个方向：

$$
\text{write\_key}
=\operatorname{HKDF\text{-}Expand\text{-}Label}
(\text{traffic\_secret},"key","",\text{key\_length})
$$

$$
\text{write\_iv}
=\operatorname{HKDF\text{-}Expand\text{-}Label}
(\text{traffic\_secret},"iv","",\text{iv\_length})
$$

AES-128-GCM 的 key 长度为 16 字节，GCM TLS Nonce 长度为 12 字节。

### 13.2 每条记录的 Nonce

每个方向维护从 0 开始的 64 bit 记录序号 `seq`。把序号左侧补零到 IV 长度：

$$
\text{padded\_seq}
=0^{\text{ivlen}-8}\parallel\text{seq}_{64}
$$

每条记录的 Nonce：

$$
N=\text{write\_iv}\oplus\text{padded\_seq}
$$

因此，同一 traffic key 下只要序号不重复，Nonce 就不重复。更换 traffic key 后序号重新从 0 开始。

记录序号不在网络中显式发送。每个方向分别维护读、写序号，某组 traffic key 下的第一条记录使用 0，读或写完一条记录后加 1；认证失败会直接终止连接，因此不会尝试“跳过坏记录后继续同步”。实现必须遵守记录数和 AEAD 使用上限，在序号耗尽前更新密钥或关闭连接。

### 13.3 TLSInnerPlaintext 与 AAD

TLS 1.3 加密的内部明文：

```text
content || inner_content_type || zero_padding
```

外层 `TLSCiphertext` 的 content type 固定表现为 application_data，以减少握手/应用类型泄露。

AAD 是外层记录头的编码：

```text
opaque_type || legacy_record_version || length
```

其中加密记录的 `opaque_type` 固定为 `application_data(23)`，`legacy_record_version` 固定为 `0x0303`，`length` 是后续 `encrypted_record` 的字节数，包含 AEAD Tag 等扩张。它们虽然以明文发送，却都受 Tag 认证；攻击者不能在不触发验证失败的情况下修改这些字段。

### 13.4 接收端顺序

1. 根据当前读 IV 和接收序号构造 Nonce；
2. 以外层记录头为 AAD；
3. 调用 AEAD `Open`；
4. Tag 验证失败则终止相应连接处理，绝不释放未认证明文；
5. 验证成功后从末尾向前跳过零字节，把第一个非零字节解释为内部 content type；若整个结果全为零或类型非法，则报错；
6. 更新序号并把内容交给对应协议层。

不要把 GCM 拆成“先 CTR 解密，再让应用看看明文，最后验证 GHASH”。AEAD 接口必须把认证成功作为释放明文的前置条件。

### 13.5 为什么 64 bit 序号不是实际使用上限

Nonce 的 64 bit 序号尚未回绕，并不代表一把 key 可以安全处理接近 $2^{64}$ 条记录。AEAD 的安全界会随加密数据量和攻击者提交的伪造次数下降。

RFC 9846 给 AES-GCM 的发送端参考界限是每组 key 最多约：

$$
2^{24.5}
$$

条满长记录，约 2400 万条，以保留约 $2^{-57}$ 的认证加密安全裕量。实现必须在达到规范界限前执行 KeyUpdate 或关闭连接。0-RTT 没有可用的 KeyUpdate，因此发送端必须保证 early data 本身不越界。

这类界限属于协议和 AEAD 的组合性质，不能用“AES-128 穷举要 $2^{128}$ 次”来替代评估。

---

## 十四、会话恢复、0-RTT、KeyUpdate 与前向安全

### 14.1 PSK 恢复

完成一次 TLS 1.3 握手后，服务器可发送 NewSessionTicket。客户端以后用票据标识的 PSK 恢复会话。

PSK 不直接取代 Key Schedule，而是进入 Early Secret，再与新握手上下文、可选的新非对称共享秘密共同派生密钥。

`psk_dhe_ke` 中的名称是历史命名；该模式把 PSK 与新的非对称密钥建立结果结合，在经典部署中通常是 ECDHE，因此可保留该次恢复会话的前向安全。纯 `psk_ke` 不混入新共享秘密，其安全性质不同，且并非所有实现都允许。

### 14.2 Binder

客户端先按 PSK 类型派生 binder key：

$$
\text{binder\_key}
=\operatorname{Derive\text{-}Secret}
(\text{early\_secret},
"ext binder"\ \text{或}\ "res binder","")
$$

再像 Finished 一样先派生：

$$
\text{binder\_finished\_key}
=\operatorname{HKDF\text{-}Expand\text{-}Label}
(\text{binder\_key},"finished","",h)
$$

最后计算：

$$
\text{binder}
=\operatorname{HMAC}_{\text{binder\_finished\_key}}
(\operatorname{TranscriptHash}(\operatorname{Truncate}(\text{ClientHello})))
$$

`Truncate(ClientHello)` 保留到 `pre_shared_key.identities` 为止，不包含 binders 列表本身，但各级长度字段按“完整 binder 已存在”编码。发生 HelloRetryRequest 时，前一条 ClientHello 与重试消息也按 transcript 的 `message_hash` 规则纳入。

因此 binder 不是直接用 `binder_key` 对一个随意裁剪的 ClientHello 做 HMAC。它证明客户端知道 PSK，并把 PSK identity 与当前握手绑定，防止攻击者仅复制票据标识就声称持有恢复密钥。

### 14.3 0-RTT 为什么可重放

0-RTT Early Data 在客户端收到本次服务器新鲜随机量和 ECDHE 公钥之前发送。密钥主要来自旧会话 PSK 与当前 ClientHello。

攻击者可复制合法的 ClientHello 和 Early Data，让一个或多个服务器实例再次接受。AEAD 只能证明密文由持有 early key 的一方生成且未被修改，不能自动证明“服务器从未处理过这条业务请求”。

因此 0-RTT 只适合可安全重放或具有应用层幂等/防重放机制的操作，例如某些读取请求；转账、创建订单、发送消息等有副作用操作不应默认允许。

0-RTT 还不具备相对于该 PSK 的前向安全：若恢复 PSK 后来泄露，已捕获的 early data 可能被解密。应用必须显式选择是否启用 0-RTT，并定义服务器拒绝 early data 后的重试语义；TLS 库不应在应用不知情时自动重发。

### 14.4 KeyUpdate

TLS 1.3 可从当前 application traffic secret 派生下一代：

$$
\text{traffic\_secret}_{N+1}
=
\operatorname{HKDF\text{-}Expand\text{-}Label}
(\text{traffic\_secret}_N,"traffic upd","",\operatorname{Hash.length})
$$

再派生新的 Key 和 IV。

KeyUpdate：

- 限制单把 AEAD Key 的使用量；
- 提供阶段隔离；
- 在旧密钥已安全擦除时，保护旧记录不被仅凭新密钥解密。

状态机细节同样是安全的一部分：

- KeyUpdate 消息本身用旧发送 key 加密，随后发送端才切到下一代 key；
- 接收端必须先在旧 key 下收到 KeyUpdate，才能接受新 key 下的记录，以防截断攻击；
- 每次换 key 后，该方向的记录序号重新从 0 开始；
- `update_requested` 只请求对端更新其发送方向，不能把两个方向当成同一组 key；
- RFC 9846 把单连接发送端 epoch 上限设为 $2^{48}-1$，并限制重复请求更新的状态机行为。

但它不是新的 DH 交换。若攻击者获得当前 traffic secret，通常也能计算未来链上的 traffic secrets；单靠 KeyUpdate 不能在攻击者持续掌握当前状态时“自愈”。

### 14.5 前向安全的精确定义

使用临时非对称密钥建立（经典场景通常是 ECDHE）并销毁临时秘密后，未来长期认证私钥泄露不应使历史会话密钥可恢复。

它不保证：

- 端点当时被攻陷时历史明文仍安全；
- 会话密钥被日志记录后仍安全；
- 恶意随机数或复用临时私钥仍安全；
- 未来会话不受已泄露认证私钥冒充。

---

## 十五、TLS 1.2 与 TLS 1.3 的关键差异

### 15.1 Cipher Suite 语义

TLS 1.2 套件名称常同时编码：

```text
密钥交换 + 身份认证 + 对称加密 + MAC/PRF Hash
```

例如：

```text
TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
```

TLS 1.3 套件只选择：

```text
AEAD + HKDF/Transcript Hash
```

例如：

```text
TLS_AES_128_GCM_SHA256
TLS_AES_256_GCM_SHA384
TLS_CHACHA20_POLY1305_SHA256
TLS_AES_128_CCM_SHA256
TLS_AES_128_CCM_8_SHA256
```

密钥交换组和签名算法分别通过扩展协商。所以 `TLS_AES_128_GCM_SHA256` 不代表证书一定使用 RSA、ECDSA 或 SHA-256 签名。

### 15.2 TLS 1.2 PRF

TLS 1.2 典型派生：

$$
\text{master\_secret}
=\operatorname{PRF}
(\text{pre\_master\_secret},
"master secret",
\text{ClientRandom}\parallel\text{ServerRandom})
$$

再派生：

$$
\text{key\_block}
=\operatorname{PRF}
(\text{master\_secret},
"key expansion",
\text{ServerRandom}\parallel\text{ClientRandom})
$$

Extended Master Secret 扩展把 master secret 绑定到会话握手摘要，以抵御三次握手等跨会话绑定问题。现代 TLS 1.2 实现应启用该扩展并只保留 ECDHE + AEAD 等安全配置。

### 15.3 TLS 1.2 记录 MAC：从 MAC-then-Encrypt 到 AEAD

TLS 1.2 并不只有一种记录保护路径。使用传统流密码或 CBC 套件时，每个方向从 `key_block` 中获得独立的 `write_MAC_key` 和加密密钥。对发送方向的第 `seq_num` 条记录，标准 MAC 输入为：

$$
\operatorname{HMAC}_{\text{write\_MAC\_key}}
\left(
\begin{aligned}
&\text{seq\_num}
\parallel\text{type}
\parallel\text{version}\\
&\parallel\text{length}
\parallel\text{fragment}
\end{aligned}
\right)
$$

其中：

- `seq_num` 是每个连接状态、每个方向独立维护的 64 bit 序号，不在线上传输；
- `type`、`version` 和 `length` 来自待保护记录头；
- `fragment` 是压缩层输出；现代配置通常不使用 TLS 压缩；
- 序号纳入 MAC 后，删除、重复或重排记录会在有状态接收端被发现；
- 客户端写密钥与服务器写密钥相互独立，不能让一个方向的记录被原样插入另一个方向。

TLS 1.2 CBC 的默认组合是 **MAC-then-Encrypt**：

```text
MAC = HMAC(seq_num || header || plaintext_fragment)
encrypted = CBC_Encrypt(plaintext_fragment || MAC || padding)
```

MAC 位于加密内容内部，Padding 位于 MAC 之后。接收端必须在不泄露“Padding 是否正确”“MAC 是否正确”及处理时间差异的条件下完成解密和验证；历史上这一复杂组合导致了多类 Padding/时序 Oracle。

RFC 7366 定义了可协商的 **Encrypt-then-MAC** 扩展：

```text
encrypted = CBC_Encrypt(plaintext_fragment || padding)
MAC = HMAC(seq_num || record_header || explicit_IV || encrypted)
record = explicit_IV || encrypted || MAC
```

它先认证密文，再在 MAC 验证通过后进入 CBC 解密和 Padding 处理，安全边界更清晰。但这是需要双方协商的扩展，不能看到“TLS 1.2 CBC”就假定一定使用 Encrypt-then-MAC。

TLS 1.2 的 AEAD 套件是第三条路径。AES-GCM、AES-CCM 等套件直接使用 AEAD Tag，不再为记录正文派生独立 HMAC Key。套件名末尾的 `SHA256` 或 `SHA384` 仍可用于 TLS 1.2 PRF，不能机械理解为“记录又额外做了一次 HMAC”。

TLS 1.3 进一步删除了 CBC 和所有“加密 + 独立记录 MAC”套件：记录正文统一由 AEAD 保护。HMAC 仍存在，但用于 HKDF、Finished 和 PSK Binder，而不是作为应用记录末尾的独立 MAC。

### 15.4 被 TLS 1.3 删除的历史方案

- 静态 RSA 密钥传输；
- 静态 DH；
- CBC 套件；
- RC4、3DES 等旧加密；
- 非 AEAD 的“加密 + 独立 MAC”记录保护；
- DSA，以及 CertificateVerify 中的 SHA-1 等弱签名组合；
- 复杂的重新协商机制。

TLS 1.3 的 `CertificateVerify` 禁止 SHA-1；旧证书链中的 SHA-1 仅存在非常受限的兼容例外，不应把它理解为新握手签名仍可选择 SHA-1。删除历史组合减少了状态机和 Oracle 攻击面。TLS 1.3 不是“把 TLS 1.2 默认参数改强”，而是重新简化了可组合的密码学路径。

---

## 十六、安全边界与工程选型

### 16.1 AES-128、AES-256 与 ChaCha20-Poly1305

- **AES-128-GCM**：128 bit 密钥，拥有广泛硬件加速，通常是高性能通用选择。
- **AES-256-GCM**：更长密钥，轮数更多；适用于明确要求 256 bit 对称密钥的策略，但不会修复 Nonce 重复、侧信道或协议错误。
- **ChaCha20-Poly1305**：在缺少 AES 硬件加速的设备上常有稳定的软件性能，也避免基于查表 AES 的部分实现风险。

AES-256 不意味着整个 TLS 连接拥有“256 bit 综合安全”：

- 证书、密钥建立、Hash 和实现分别有自己的安全级别；
- GCM Tag 通常仍为 128 bit；
- 量子攻击对对称和公钥密码的影响不同；
- X25519、RSA、ECDSA 等传统公钥算法不是后量子安全算法。

RFC 9846 已把 TLS 1.3 的相关接口描述泛化为“非对称密钥建立”；RFC 9954 进一步给出把多个固定长度共享秘密拼接为一个混合组输入的通用构造。但 RFC 9954 是 Informational，且不指定某个具体后量子算法或部署策略。是否启用某个经典 + 后量子混合组，应遵循相应组规范、实现支持和组织迁移策略，不能只看到“hybrid”名称就假定全部安全属性自动成立。

应优先使用平台 TLS 库的安全默认值，而不是仅根据一个算法名字手工排列套件。

### 16.2 密钥和 Nonce 生命周期

对每把密钥都应明确：

- 如何生成；
- 属于哪个协议、方向和阶段；
- 允许处理多少消息/字节；
- Nonce 如何唯一；
- 何时轮换；
- 何时从内存销毁；
- 是否允许持久化或日志记录。

“同一 AES Key 在多个服务共享，但各服务自行从 0 计数”会造成 Nonce 冲突。解决方案需要全局分区的 Nonce 空间、不同子密钥，或由协议统一管理序号。

### 16.3 随机数

以下值必须来自密码学安全随机源或标准确定性构造：

- AES/HMAC 长期密钥；
- DH/ECDH 临时私钥；
- RSA 素数；
- ECDSA nonce；
- OAEP/PSS 随机量；
- 协议挑战值。

普通伪随机函数、时间戳、进程 ID 和自增整数不能直接充当秘密随机数。Nonce 可以是计数器，但前提是协议保证同一密钥下唯一；“Nonce 不保密”不代表它可以重复。

### 16.4 编码与输入验证

大量密码漏洞发生在数学运算之前：

- 公钥点或整数没有按规范验证；
- 可变长字段拼接存在歧义；
- DER/ASN.1 接受非规范或重复字段；
- RSA/ECDSA 签名解析宽松；
- Tag 比较提前退出；
- 解密错误被区分成可观察的 Oracle；
- 证书主机名、用途或 critical extension 被忽略。

应让成熟库同时负责原语、编码和协议状态机。不要把多个“安全函数”随意拼成自定义协议。

### 16.5 常见误解

#### 误解一：TLS 就是 RSA 加密 AES 密钥

这是对部分历史 TLS 1.2 静态 RSA 套件的简化描述。TLS 1.3 使用 PSK 和/或临时非对称密钥建立结果，再经 HKDF 派生密钥；RSA 若出现通常用于签名认证。

#### 误解二：用了 AES 就同时有完整性

AES 是分组密码。ECB、CBC、CTR 本身都不提供完整认证；应使用 AES-GCM 等 AEAD，或经分析的 Encrypt-then-MAC 协议。

#### 误解三：GHASH 和 HMAC 可以互换

HMAC 是独立 MAC 构造；GHASH 是 GCM 内部的线性通用哈希组件，必须与 AES 掩码和唯一 Nonce 一起使用。

#### 误解四：GCM Nonce 越随机越安全

核心要求是同一密钥下唯一。若随机生成，必须评估生日碰撞和消息总量；协议序号常能提供更强的确定性保证。

#### 误解五：证书签名会加密业务数据

证书签名证明公钥归属并认证握手。业务数据由握手派生的对称 AEAD 密钥保护。

#### 误解六：KeyUpdate 能让已被持续控制的连接自愈

KeyUpdate 只沿当前秘密单向派生，不引入新的外部熵或 DH 共享秘密。攻击者掌握当前 secret 时通常能推导未来 secret。

#### 误解七：0-RTT 有 AEAD，所以不能重放

AEAD 防篡改，不自动提供全局唯一处理语义。0-RTT 的重放风险必须由服务器和应用层管理。

#### 误解八：MAC 验证成功就证明消息是“新的”

MAC 只证明认证输入与共享密钥一致。攻击者原样重放一个旧的合法 $(M,T)$ 时，纯 MAC 验证仍会成功。防重放必须把序号、Nonce、时间窗或请求唯一标识纳入认证输入，并由接收端维护和执行相应状态。

---

## 十七、把整套原理串起来

### 17.1 一条 HTTPS 连接发生了什么

以下仍以最常见的证书认证 ECDHE 路径为例：

1. 客户端发送支持的版本、套件、组、签名算法和临时 ECDHE KeyShare。
2. 服务器选择参数并返回自己的临时 ECDHE KeyShare。
3. 双方独立计算相同 ECDHE 共享秘密，且不跨连接复用 KeyShare。
4. HKDF 把 PSK（若有）、共享秘密和 transcript 逐阶段混合。
5. 服务器证书把认证公钥绑定到域名。
6. CertificateVerify 用证书私钥认证本次握手。
7. Finished 用 HMAC 证明握手秘密持有者看到了相同 transcript。
8. 双方派生不同方向的 application traffic secret。
9. 每个方向再派生 AES-GCM/ChaCha20-Poly1305 Key 和 IV。
10. 记录序号与 IV 组合成唯一 Nonce，记录头作为 AAD。
11. 接收端只有在 Tag 验证成功后才释放明文。

### 17.2 每个算法只回答一个核心问题

- **SHA-256/SHA-384**：怎样把任意长度公开消息压缩成固定长度摘要，并为 transcript、签名以及 HMAC/HKDF 提供底层 Hash？
- **AES**：怎样用同一把秘密密钥高效、可逆地变换固定分组？
- **MAC**：怎样让共享密钥持有者检测伪造，并把重放防护所需状态绑定进认证输入？
- **HMAC**：怎样用哈希函数构造通用 MAC？
- **CMAC/CCM**：怎样用分组密码完成消息认证，并进一步与 CTR 组合成 AEAD？
- **Poly1305**：怎样用每条消息唯一的一次性密钥进行高效多项式认证？
- **GHASH**：怎样高效累加 GCM 的 AAD 和密文认证多项式？
- **DH/ECDH**：怎样在公开信道上得到相同共享秘密？
- **RSA/ECDSA/EdDSA**：怎样用私钥生成公开可验证的身份签名？
- **HKDF**：怎样提取共享秘密并派生成上下文隔离的密钥？
- **X.509**：怎样把认证公钥绑定到域名和信任链？
- **TLS**：怎样把以上能力按状态机组合，并处理降级、重放、错误和密钥生命周期？

### 17.3 最小安全检查表

- 使用受支持的 TLS 库和 TLS 1.3 安全默认配置；
- 不自定义 AES 模式、MAC 拼接或证书验证；
- GCM 同一密钥下绝不重复 Nonce；
- ChaCha20-Poly1305 同一密钥下绝不重复 Nonce，不直接复用 Poly1305 一次性密钥；
- MAC 输入使用无歧义编码；需要防重放时，把序号或 Nonce 纳入认证并维护接收状态；
- ECDSA nonce 不重复、不泄露；
- DH/ECDH 使用标准组并验证对端输入，TLS KeyShare 不跨连接复用；
- RSA 使用 OAEP/PSS，不使用裸 RSA；
- HMAC、CMAC 等 MAC 密钥按协议和用途隔离，所有 Tag 使用常量时间验证；
- AEAD 失败绝不释放明文；
- 在 AEAD 单 key 使用上限前 KeyUpdate 或关闭连接；
- 0-RTT 只承载可重放安全的业务；
- 密钥、随机数和错误日志不泄露敏感材料。

---

## 参考标准

1. NIST, [FIPS 197: Advanced Encryption Standard](https://csrc.nist.gov/pubs/fips/197/final).
2. NIST, [SP 800-38A: Block Cipher Modes of Operation](https://csrc.nist.gov/pubs/sp/800/38/a/final).
3. NIST, [SP 800-38D: Galois/Counter Mode and GMAC](https://csrc.nist.gov/pubs/sp/800/38/d/final).
4. NIST, [FIPS 180-4: Secure Hash Standard](https://csrc.nist.gov/pubs/fips/180-4/upd1/final).
5. IETF, [RFC 2104: HMAC](https://www.rfc-editor.org/rfc/rfc2104).
6. IETF, [RFC 4231: HMAC-SHA Test Cases](https://www.rfc-editor.org/rfc/rfc4231).
7. IETF, [RFC 5869: HKDF](https://www.rfc-editor.org/rfc/rfc5869).
8. IETF, [RFC 8017: PKCS #1 v2.2](https://www.rfc-editor.org/rfc/rfc8017).
9. NIST, [SP 800-56A Rev. 3: Pair-Wise Key-Establishment Schemes Using Discrete Logarithm Cryptography](https://csrc.nist.gov/pubs/sp/800/56/a/r3/final).
10. NIST, [FIPS 186-5: Digital Signature Standard](https://csrc.nist.gov/pubs/fips/186-5/final).
11. IETF, [RFC 6090: Fundamental Elliptic Curve Cryptography Algorithms](https://www.rfc-editor.org/rfc/rfc6090).
12. IETF, [RFC 7748: Elliptic Curves for Security](https://www.rfc-editor.org/rfc/rfc7748).
13. IETF, [RFC 6979: Deterministic DSA and ECDSA](https://www.rfc-editor.org/rfc/rfc6979).
14. IETF, [RFC 8032: Edwards-Curve Digital Signature Algorithm](https://www.rfc-editor.org/rfc/rfc8032).
15. IETF, [RFC 5280: Internet X.509 Public Key Infrastructure Certificate and CRL Profile](https://www.rfc-editor.org/rfc/rfc5280).
16. IETF, [RFC 9846: TLS 1.3](https://www.rfc-editor.org/rfc/rfc9846).
17. IETF, [RFC 5246: TLS 1.2（历史规范，已被 RFC 9846 取代）](https://www.rfc-editor.org/rfc/rfc5246).
18. IETF, [RFC 7627: TLS Extended Master Secret（历史扩展，已被 RFC 9846 取代）](https://www.rfc-editor.org/rfc/rfc7627).
19. IETF, [RFC 8996: Deprecating TLS 1.0 and TLS 1.1](https://www.rfc-editor.org/rfc/rfc8996).
20. IETF, [RFC 9325: Recommendations for Secure Use of TLS and DTLS](https://www.rfc-editor.org/rfc/rfc9325).
21. IETF, [RFC 9525: Service Identity in TLS](https://www.rfc-editor.org/rfc/rfc9525).
22. IETF, [RFC 9954: Hybrid Key Exchange in TLS 1.3](https://www.rfc-editor.org/rfc/rfc9954).
23. NIST, [SP 800-38B: CMAC Mode for Authentication](https://csrc.nist.gov/pubs/sp/800/38/b/upd1/final).
24. NIST, [SP 800-38C: CCM Mode for Authentication and Confidentiality](https://csrc.nist.gov/pubs/sp/800/38/c/upd1/final).
25. IRTF, [RFC 8439: ChaCha20 and Poly1305 for IETF Protocols](https://www.rfc-editor.org/rfc/rfc8439).
26. IETF, [RFC 7366: Encrypt-then-MAC for TLS and DTLS](https://www.rfc-editor.org/rfc/rfc7366).
27. IETF, [RFC 6234: US Secure Hash Algorithms, HMAC and HKDF](https://www.rfc-editor.org/rfc/rfc6234).
