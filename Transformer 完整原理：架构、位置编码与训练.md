# Transformer 完整原理：架构、位置编码与训练

本文把 Transformer 的三个层面放到一条因果链上：

1. **架构**回答一次前向传播怎样把 Token 变成下一个 Token 的概率。
2. **位置编码**回答没有循环结构的注意力如何感知顺序和距离。
3. **训练**回答损失如何沿输出层、残差、归一化、Attention 和 FFN 反向传播。

全文以 Decoder-only 语言模型为主线，同时指出 Encoder、Encoder-Decoder、训练和自回归推理之间的差别。

---

## 一、先建立完整心智模型

### 1.1 Transformer 学到的是什么

给定 Token 序列

$$
x_1,x_2,\ldots,x_L
$$

自回归语言模型学习条件分布

$$
p(x_{t+1}\mid x_{\le t})
$$

训练时，一次并行计算序列中所有位置的“下一个 Token”预测；推理时，每次从最后一个位置的分布中选择一个 Token，再把它追加到上下文。

Transformer 的主数据流可以概括为：

```text
文本
  → Token ID
  → Token Embedding
  → 注入位置信息
  → N 个 Transformer Block
       ├─ Attention：跨 Token 聚合信息
       └─ FFN：逐 Token 变换特征
  → Final Norm
  → LM Head
  → Logits / Softmax
  → 下一个 Token 的概率
```

设：

- $B$：batch size
- $L$：序列长度
- $d$：模型维度 `d_model`
- $h_q$：Query 头数；讨论标准 MHA 时也简记为 $h$
- $h_{kv}$：Key/Value 头数；MHA 中 $h_{kv}=h_q$，GQA/MQA 中 $h_{kv}<h_q$
- $d_h=d/h_q$：常见实现中每个 Query、Key、Value 头用于点积的维度
- $V$：词表大小

一个批次进入模型后的主体张量形状为

$$
X\in\mathbb{R}^{B\times L\times d}
$$

模型不会让“单个神经元保存一个单词的意义”。语法、实体、距离、风格等信息分布在多层、多头和多个特征方向中。

### 1.2 一个 Pre-Norm Decoder Block

现代大语言模型常采用 Pre-Norm。忽略 Dropout 时，一层可写成：

$$
U=X+\operatorname{Attention}(\operatorname{Norm}(X))
$$

$$
Y=U+\operatorname{FFN}(\operatorname{Norm}(U))
$$

这里有两条必须分开的信息通路：

- **残差通路**直接传递旧表示和梯度。
- **变换通路**通过 Attention 或 FFN 学习增量。

Attention 负责 Token 间通信；FFN 在每个位置独立执行相同的非线性映射。二者交替堆叠后，局部和长距离信息逐层融合。

### 1.3 三类常见架构

|架构|可见范围|典型目标|常见用途|
|---|---|---|---|
|Encoder-only|通常双向可见|掩码预测、表征学习|分类、检索、序列标注|
|Decoder-only|只能看当前位置及之前|下一个 Token 预测|生成式大语言模型|
|Encoder-Decoder|Encoder 双向；Decoder 因果；二者 Cross-Attention|条件序列生成|翻译、摘要、结构化转换|

三者共享 Attention、FFN、残差和归一化等基本部件，主要差别是信息可见范围、是否包含 Cross-Attention，以及输出头和训练目标。

#### 1.3.1 Encoder-only

```text
输入序列
  → Token / Position 表示
  → N × [双向 Self-Attention → FFN]
  → 每个位置的上下文表示
  → Pooling 或任务头
```

Encoder 的 Self-Attention 通常允许每个非 Padding Token 同时读取左右两侧信息，因此擅长对完整输入进行编码。输出仍是长度为 $L$ 的表示序列，可逐位置用于序列标注，也可通过特殊 Token、池化或注意力汇聚成整个序列的表示。预训练常采用掩码恢复或对比式表征目标；它没有内建的从左到右生成约束。

#### 1.3.2 Decoder-only

```text
输入前缀
  → Token / Position 表示
  → N × [因果 Self-Attention → FFN]
  → Final Norm → LM Head
  → 各位置的下一个 Token 分布
```

Decoder-only 的每个位置只能读取自己及之前的位置。训练时用因果 Mask 并行计算所有位置的预测，推理时按顺序生成并用 KV Cache 复用历史 K/V。提示、上下文和待生成内容通常被组织在同一个 Token 序列中，模型通过统一的下一个 Token 目标学习理解与生成。

#### 1.3.3 Encoder-Decoder

```text
源序列 → N × Encoder Block ───────────────┐
                                          ↓ K、V
目标前缀 → N × [因果 Self-Attention → Cross-Attention → FFN]
          → Final Norm → LM Head → 目标 Token 分布
```

Encoder 先双向编码完整源序列；Decoder 的因果 Self-Attention 建模已出现的目标前缀，再以目标表示为 Query、Encoder 输出为 Key/Value，通过 Cross-Attention 读取源信息。训练时通常把右移后的目标序列输入 Decoder 并并行预测目标 Token；推理时 Encoder 只运行一次，Decoder 自回归生成。源序列与目标序列长度可以不同。

这三类结构的核心区别可归结为：Encoder-only 侧重完整输入的双向表征，Decoder-only 用单一因果序列统一条件与生成，Encoder-Decoder 则把源信息编码和目标生成分成两条数据流。

---

## 二、从文本到输入向量

### 2.1 Tokenization

分词器把文本映射为离散 ID：

```text
"位置编码很重要"
    ↓ tokenizer
[3812, 920, 1734, ...]
```

Token 不一定是完整汉字或单词，也可能是子词、字节或特殊控制符。词表大小 $V$ 影响：

- Embedding 和 LM Head 的参数量；
- 平均序列长度；
- 稀有词、代码和多语言文本的切分效果。

分词器属于模型协议的一部分。相同权重配上错误的 tokenizer，输入语义会被彻底破坏。

### 2.2 Token Embedding

Embedding 矩阵为

$$
E\in\mathbb{R}^{V\times d}
$$

Token ID $x_t$ 对应第 $x_t$ 行：

$$
e_t=E[x_t]
$$

把所有位置堆叠后得到

$$
X_{\text{tok}}\in\mathbb{R}^{B\times L\times d}
$$

许多语言模型会让输入 Embedding 与输出 LM Head 共享参数：

$$
W_{\text{lm}}=E^\top
$$

这称为 **weight tying**。它减少参数量，也让“输入词向量空间”和“输出词分类空间”建立直接联系；训练时，同一个参数会同时收到输入侧和输出侧梯度。

### 2.3 N-gram Embedding：用局部 Token 组合查找大容量记忆

N-gram Embedding 的核心作用有三点：

1. **直接记忆局部模式**：为常见短语、代码片段、固定搭配和局部语法提供可学习的专用向量，减少主干反复从单 Token 表示中组合这些模式的负担；
2. **根据上下文区分同一 Token**：普通 Embedding 只看当前 ID，N-gram 地址还包含前一至两个 Token，因此同一个 Token 在不同局部上下文中可以读取不同的附加表示；
3. **以低计算成本扩大参数容量**：每个 Token 只激活巨型表中的少量行，增加大量可寻址知识容量，而不让全部表参数参加矩阵乘法。

它主要补充**短距离词法记忆**，不能取代 Attention、Gated DeltaNet 等长距离信息混合机制；哈希碰撞和有限的 N-gram 长度也决定了它不适合直接保存任意长上下文。

普通 Token Embedding 只由当前位置的一个 Token ID 寻址；N-gram 则表示连续的 $n$ 个 Token。对序列：

```text
我 喜欢 深度 学习
```

可得到：

```text
1-gram：我、喜欢、深度、学习
2-gram：我喜欢、喜欢深度、深度学习
3-gram：我喜欢深度、喜欢深度学习
```

传统统计 N-gram 语言模型使用前 $n-1$ 个 Token 近似完整历史：

$$
P(x_i\mid x_{<i})
\approx
P(x_i\mid x_{i-n+1},\ldots,x_{i-1})
$$

现代 N-gram Embedding 不直接保存条件概率，而是把当前 Token 与若干历史 Token 组成离散地址，从可训练表中读取一个向量：

$$
e_i^{(n)}
=
E_n
\left[
\operatorname{Hash}(x_{i-n+1},\ldots,x_i)
\right]
$$

同一个 Token 在“苹果公司”“苹果手机”“一个苹果”等不同局部上下文中因此可以读取不同的附加表示。它相当于给主干增加一块局部词法与短模式记忆，而不是用 Attention 临时重新推导所有常见短语。

若词表大小为 $V$，完整 Bigram 和 Trigram 空间分别有 $V^2$ 与 $V^3$ 种组合，无法逐项建表。实际方法把组合哈希到固定数量 $M$ 的桶：

$$
h_i^{(n)}
=
\operatorname{Hash}(x_{i-n+1},\ldots,x_i)\mathbin{\operatorname{mod}} M
$$

哈希碰撞意味着不同短语可能共享一行参数。大哈希表、多个独立 Hash Head 和不同模数可降低多个 Head 同时碰撞的概率；代价是表参数占用大，并且随机查表可能受内存带宽限制。

Qwen3.8-Flash-Next 的 N-gram Embedding 是一个具体例子。它使用当前 Token 和过去 Token 构造 2-gram、3-gram，不额外构造 1-gram，因为标准 Token Embedding 已经提供单 Token 表示：

```text
位置 i 的 2-gram：(x[i-1], x[i])
位置 i 的 3-gram：(x[i-2], x[i-1], x[i])
```

这只读取当前及过去 ID，不会泄漏未来标签；遇到 EOS/样本边界时，历史位置以 EOS 填充，避免跨样本拼出虚假的 N-gram。对不同位置使用由固定种子生成的奇数乘数 $a_0,a_1,a_2$，混合值为：

$$
z_i^{(2)}
=
(x_i a_0)
\operatorname{XOR}
(x_{i-1}a_1)
$$

$$
z_i^{(3)}
=
(x_i a_0)
\operatorname{XOR}
(x_{i-1}a_1)
\operatorname{XOR}
(x_{i-2}a_2)
$$

每种 N-gram 使用 8 个 Hash Head。第 $h$ 个 Head 以不同的质数词表大小 $p_{n,h}$ 取模，并加上该 Head 在总表中的行偏移：

$$
\operatorname{id}_{i,n,h}
=
\left(z_i^{(n)}\bmod p_{n,h}\right)
+\operatorname{offset}_{n,h}
$$

因此一个 Token 共得到 $8+8=16$ 个查表地址。每个 Head 的逻辑词表约有 2000 万行，每行向量为 160 维；16 个向量拼接为：

$$
e_i^{\text{ngram}}
=
\operatorname{Concat}_{n\in\{2,3\},\,h=1}^{8}
E_{n,h}[\operatorname{id}_{i,n,h}]
\in\mathbb R^{2560}
$$

对应参数量约为：

$$
16\times20\,000\,000\times160
\approx51.2\text{B}
$$

虽然总表很大，每个 Token 只读取 16 行、共 2560 个表元素，不会让全部 51.2B 参数参加矩阵乘法。地址只依赖 Token ID，可以预先计算；表可放在 Host Memory，由异步预取与前面网络层的计算重叠。因此它扩展的是“可寻址参数容量”，但仍需付出权重存储、随机访问和数据传输成本，不能把它理解为免费的 51B 参数。

Qwen 将这一模块放在靠近输入的第 2 个 Decoder Layer，作为 Per-Layer Embedding（PLE）注入。查表向量先投影成共享 Value 和四条残差流各自的 Key，当前隐藏状态作为 Query 决定每条流接收多少 N-gram 信息：

$$
k_{i,r}^{E}
=
\operatorname{Norm}_r(W_Ke_i^{\text{ngram}}),
\qquad
v_i^{E}=W_Ve_i^{\text{ngram}},
\qquad
q_{i,r}=\operatorname{Norm}_r(R_{i,r})
$$

$$
s_{i,r}
=
\frac{q_{i,r}^{\top}k_{i,r}^{E}}{\sqrt d},
\qquad
\widetilde s_{i,r}
=
\operatorname{sign}(s_{i,r})\sqrt{|s_{i,r}|+\varepsilon}
$$

$$
g_{i,r}=\sigma(\widetilde s_{i,r}),
\qquad
u_{i,r}=g_{i,r}v_i^{E}
$$

门控后的 Value 还会经过 kernel size 为 4、dilation 为 3 的逐通道因果卷积和 SiLU，用于补充更宽的局部词法组合；其输出加入四条残差流。这里的 $g_{i,r}$ 是 PLE 内容注入门，下一节介绍的 Gated Residual 还有独立的子层读门与写门，两者不能混为同一个 Gate。

### 2.4 为什么仅有 Token Embedding 不够

如果没有位置相关信息，自注意力对输入顺序具有置换等变性。设 $P$ 是对序列位置做重排的置换矩阵：

$$
Q=XW_Q,\quad K=XW_K,\quad V=XW_V
$$

对输入重排后：

$$
Q'=PXW_Q=PQ,\quad K'=PK,\quad V'=PV
$$

于是

$$
\operatorname{softmax}\left(\frac{Q'K'^\top}{\sqrt{d_h}}\right)V'
=
P\operatorname{softmax}\left(\frac{QK^\top}{\sqrt{d_h}}\right)V
$$

也就是说，在没有位置项、且可见性关系随输入一起重排的 Self-Attention 中，输入怎么换序，输出只会跟着换序；模块本身没有获得“第几个”“相距多远”的坐标。

严格地说，Decoder 的三角因果 Mask 会破坏任意位置置换的对称性：它暴露了“当前位置能看到多长的前缀”这一边界，所以深层网络可能从边界效应中间接推断部分顺序。可是，对某个 Query 已经可见的多个历史 Token，因果 Mask 只给出“可见/不可见”，不会为它们分别提供明确的绝对位置或相对距离。标准 Decoder 因此仍使用绝对位置、RoPE、相对 Bias 等显式位置机制。

---

## 三、位置编码：给注意力建立坐标系

### 3.1 位置方法真正改变了什么

位置方法大体有三种注入位置：

1. **输入表示**：给 Token Embedding 加绝对位置向量。
2. **Q/K 表示**：旋转或变换 Q、K，使点积显式依赖相对位移。
3. **Attention Logits**：直接给注意力分数增加距离偏置。

它们最终都在改变注意力权重：

$$
A_{ij}
=\operatorname{softmax}_j
\left(
\frac{q_i^\top k_j}{\sqrt{d_h}}+b(i,j)
\right)
$$

位置编码不是独立完成“顺序理解”的模块。它提供坐标或距离信号，模型仍需在训练中学会怎样使用这些信号。

### 3.2 Learned Absolute Position Embedding

为每个位置维护可学习向量：

$$
p_i=P[i],\qquad P\in\mathbb{R}^{L_{\max}\times d}
$$

输入为：

$$
x_i=e_i+p_i
$$

优点是简单、表达自由；边界是：

- 参数表只覆盖训练配置中的最大位置；
- 超过已训练位置时没有天然定义；
- 相对距离必须由后续网络间接学习。

“配置支持 $L_{\max}$”也不代表模型在整个范围内质量一致。有效上下文还取决于训练数据中是否出现过足够多的长序列。

### 3.3 Sinusoidal Position Encoding

原始 Transformer 使用固定正弦余弦函数：

$$
\operatorname{PE}(p,2i)=
\sin\left(p\cdot\omega_i\right)
$$

$$
\operatorname{PE}(p,2i+1)=
\cos\left(p\cdot\omega_i\right)
$$

其中

$$
\omega_i=10000^{-2i/d}
$$

不同维度使用不同频率：高频维度对短距离敏感，低频维度缓慢变化，覆盖更长尺度。这只表示位置编码在输入处提供了不同尺度的坐标基；与 Token Embedding 相加并经过后续投影后，各维度会被重新混合，不能把某个最终隐藏维度固定解释为某种距离或语义。模型使用的是多个频率的组合。

对同一频率，有：

$$
\begin{bmatrix}
\sin((p+\Delta)\omega)\\
\cos((p+\Delta)\omega)
\end{bmatrix}
=
\begin{bmatrix}
\cos(\Delta\omega)&\sin(\Delta\omega)\\
-\sin(\Delta\omega)&\cos(\Delta\omega)
\end{bmatrix}
\begin{bmatrix}
\sin(p\omega)\\
\cos(p\omega)
\end{bmatrix}
$$

因此，位置平移可以由只依赖 $\Delta$ 的线性变换表达。这解释了它为何包含相对位移结构。

但它仍被称为绝对位置编码，因为每个位置 $p$ 先独立生成一个绝对向量，再加到 Token 表示上。函数可计算到训练长度之外，也不等于模型能无损外推；新的相位组合和更长依赖仍可能超出训练分布。

### 3.4 显式相对位置

显式相对位置方法直接按 $i-j$ 修改 Attention。

Shaw 风格的方法可写成：

$$
s_{ij}
=\frac{q_i^\top(k_j+a^K_{i-j})}{\sqrt{d_h}}
$$

其中 $q_i^\top k_j$ 衡量内容匹配，$q_i^\top a^K_{i-j}$ 衡量 Query 与相对位置关系的匹配。$a^K_r$ 按相对距离 $r=i-j$ 查表，与具体词无关；相同距离的 Token 对通常共享它，但因 $q_i$ 不同，实际分数影响仍可不同。参数是否跨层、跨头共享取决于实现。

并可在 Value 聚合时加入相对向量：

$$
o_i=\sum_j \alpha_{ij}(v_j+a^V_{i-j})
$$

$a^K$ 改变注意力分数，$a^V$ 让聚合结果携带信息来源的相对位置；二者通常是两套参数。相对距离一般有方向，$r$ 与 $-r$ 可使用不同表示。

T5 风格相对位置偏置更直接：

$$
s_{ij}
=\frac{q_i^\top k_j}{\sqrt{d_h}}
+b_{\operatorname{bucket}(i-j)}
$$

Bucket 把远距离合并成较粗区间，减少参数量，并让模型把容量集中在短距离的精细区别上。

### 3.5 RoPE：把位置编码成相对相位

RoPE（Rotary Position Embedding）不把位置向量加到 $X$ 上，而是对每个头的 Q、K 成对旋转。

对二维向量 $z=(z_1,z_2)$，位置 $p$ 对应旋转：

$$
R(p\theta)=
\begin{bmatrix}
\cos(p\theta)&-\sin(p\theta)\\
\sin(p\theta)&\cos(p\theta)
\end{bmatrix}
$$

对于更高维向量，RoPE 把旋转维度拆成多个二维对，其整体变换是分块对角矩阵：

$$
R_p=\operatorname{diag}\left(
R(p\theta_0),R(p\theta_1),\ldots,
R\left(p\theta_{d_{\text{rot}}/2-1}\right)
\right)
$$

不同实现可能采用相邻配对或前后半区配对；二者本质上只差维度排列，但推理时必须与训练配置一致。

$$
\widetilde q_p=R(p\theta)q_p,\qquad
\widetilde k_s=R(s\theta)k_s
$$

点积为：

$$
\widetilde q_p^\top\widetilde k_s
=q_p^\top R(p\theta)^\top R(s\theta)k_s
=q_p^\top R((s-p)\theta)k_s
$$

绝对位置 $p,s$ 在点积中合成为相对位移 $s-p$。设实际参与旋转的维度为偶数 $d_{\text{rot}}\le d_h$。这部分维度被分成多个二维对，每一对使用不同频率：

$$
\theta_i=\operatorname{base}^{-2i/d_{\text{rot}}},
\qquad
i=0,\ldots,\frac{d_{\text{rot}}}{2}-1
$$

`base`（配置中也常写作 `rope_theta`）是频率基数，控制各二维对的频率下降速度和波长分布。增大 `base` 会让后面的低频维度旋转得更慢，但它本身不等于最大上下文长度。

位置 $p$ 在第 $i$ 个二维对子上使用角度 $p\theta_i$。全维 RoPE 取 $d_{\text{rot}}=d_h$；部分旋转实现会让剩余的 $d_h-d_{\text{rot}}$ 个通道保持不变。代码中的 `rotary_dim`、`rope_theta` 或 `base` 必须与训练配置一致，不能只看模型总维度 $d$ 来推断频率。

Q、K 在同一个二维对上使用相同的基础频率 $\theta_i$，但其内容方向和实际旋转角度可以不同。若原始内容角度分别为 $\alpha,\beta$，旋转后的夹角为：

$$
(\beta+s\theta_i)-(\alpha+p\theta_i)
=(\beta-\alpha)+(s-p)\theta_i
$$

第一项表示内容关系，第二项表示相对位置。这说明 RoPE 并不要求 Q、K 方向相同，只要求二者使用同一套频率和旋转约定。若 Q、K 使用不同频率，位置项通常不能化为只依赖 $s-p$ 的形式，相对平移性质便会被破坏。

RoPE 的几个关键边界：

- 它让 QK 点积显式依赖相对相位，不代表注意力一定随距离单调衰减。
- 旋转不改变单个 Q/K 向量的范数，但会改变它们之间的夹角和点积。
- 计算到更大位置在数学上可行，不代表模型天然无限外推；长距离会遇到未训练相位、频率分辨率和注意力分布变化。

由于旋转是正交变换，反向传播很简单：

$$
\frac{\partial L}{\partial q_p}
=R(p\theta)^\top
\frac{\partial L}{\partial \widetilde q_p}
$$

K 同理。RoPE 没有大型位置表，也不改变 Attention 的渐近复杂度。

### 3.6 ALiBi：直接加入距离先验

在因果注意力中，ALiBi 给每个头分配斜率 $m_h>0$：

$$
s_{ij}^{(h)}
=\frac{q_i^{(h)\top}k_j^{(h)}}{\sqrt{d_h}}
-m_h(i-j),\qquad j\le i
$$

距离越远，先验惩罚越大。不同头使用不同斜率，使部分头偏向局部，部分头保留更长距离。

ALiBi 的偏置只依赖距离，可直接计算到更长序列，因此结构上比绝对位置表更容易外推；但最终质量仍取决于训练长度、数据和模型是否学会长依赖。

### 3.7 位置方法对比

|方法|注入位置|核心信号|额外参数|长度外推边界|
|---|---|---|---|---|
|Learned Absolute|输入 Embedding|绝对位置 ID|有|受位置表与训练分布限制|
|Sinusoidal|输入 Embedding|多频绝对坐标|无|函数可延伸，模型未必适应|
|Shaw/T5 Relative|Logits/Value|显式相对距离|通常有|取决于截断或 Bucket|
|RoPE|Q/K|相对相位|无大型位置表|通常需要缩放和继续训练|
|ALiBi|Logits|线性距离惩罚|通常仅固定斜率|结构上易延伸，能力不保证|

### 3.8 长上下文扩展为何困难

把最大长度从 $L$ 改成 $kL$，会同时改变：

- RoPE 各频率看到的相位范围；
- Attention Logits 和归一化后的分布；
- 训练数据中的依赖长度；
- 显存、计算量和 KV Cache；
- 检索到信息后能否有效利用，而不只是“能塞进去”。

常见 RoPE 扩展方法：

- **Position Interpolation**：把新位置压缩回旧范围，例如 $p'=p/k$。相位不越界，但短距离分辨率下降。
- **NTK-aware scaling**：调整 RoPE 频率基底，使不同频率的缩放更平滑。
- **Dynamic NTK**：在序列超过原长度后，按实际长度动态调整缩放。
- **YaRN**：对不同频率采用差异化插值，并结合 Attention 尺度修正和长文本继续训练。

这些名称代表一类设计思想，不存在脱离具体实现的唯一公式。判断扩展是否成功，应同时测试短文本回退、长文本困惑度、长距离检索、真实任务质量和资源开销。

### 3.9 Position ID、Padding 与 KV Cache 必须使用同一坐标系

公式里的位置 $p$ 是**逻辑位置**，不一定等于张量在当前 batch 中的物理列号。这个区别在左侧 Padding、序列打包和增量推理中非常重要。

假设同一个 batch 中有两条不同长度的序列，并采用左侧 Padding：

```text
[PAD, PAD, A, B, C]
[ D ,  E , F, G, H]
```

对第一条序列，常见逻辑 Position ID 是：

```text
[ignored, ignored, 0, 1, 2]
```

而不是机械使用物理列号 `[0,1,2,3,4]`。Padding 位置必须同时被 Attention Mask 屏蔽。若训练使用一种位置编号，推理却因左 Padding 改成另一种编号，Learned Absolute Position 会直接查到不同向量。纯 RoPE Self-Attention 则需要更细致地区分：只要同一条序列中所有有效 Q/K 都增加同一个常数偏移，虽然各向量的绝对相位改变，QK 点积仍保持不变。

RoPE 的点积对 Q 和 K 的**共同平移**具有相对不变性：

$$
R(p+c)^\top R(s+c)=R(s-p)
$$

但这不表示 Position ID 可以随意修改：

- 同一条注意力边上的 Q 与缓存 K 必须处在一致的坐标系；
- 只有对相关 Q/K 施加相同常数平移时，上述不变性才成立；逐 Token 的非均匀偏移不会抵消；
- 新 Token 的位置应延续此前逻辑序列位置，而不是简单使用“当前缓存数组下标”；
- 滑动窗口淘汰旧 KV 后，缓存槽位可以从零重新编号，但已旋转 K 的逻辑位置不能在没有相应变换的情况下被当作重新从零开始；
- 若模型还使用绝对 Bias、窗口边界或其他位置相关机制，共同平移也未必保持整个网络不变。

还要区分两类经常都被简称为“缓存”的状态：

- **RoPE 系数表**保存或即时生成各位置的 $\cos(p\theta_i)$、$\sin(p\theta_i)$，它是确定性查找表，不是历史 Token 的语义状态；
- **KV Cache**在常见实现中保存已经按其逻辑位置旋转过的 K，以及不需要 RoPE 的 V。Decode 时只旋转新 Token 的 Q/K，并把新 K/V 追加进去；不能再次旋转已有 K。

实现也可以缓存未旋转 K、在读取时统一施加 RoPE，但必须始终遵守同一种状态约定。如果 Dynamic NTK 等方案会随目标长度改变有效频率，不能让旧 K 使用一套频率、新 Q/K 使用另一套频率；应按实现协议固定本次缓存的映射，或在映射变化时重建相关缓存。

序列打包（packing）会把多条样本放进同一个长张量。此时通常需要：

1. 用块对角因果 Mask 阻止不同样本互相读取；
2. 按训练约定为每条样本重置或连续分配 Position ID；
3. 分别处理 EOS、loss mask 和有效长度。

只重置 Position ID、却没有阻断跨样本 Attention，会产生数据泄漏；只做块对角 Mask、却使用与训练不一致的位置编号，也会引入分布偏移。

---

## 四、多头自注意力

### 4.1 从 X 得到 Q、K、V

对归一化后的输入 $H\in\mathbb{R}^{B\times L\times d}$：

$$
Q=HW_Q,\quad K=HW_K,\quad V=HW_V
$$

其中 $W_Q,W_K,W_V$ 是可训练矩阵。直觉上：

- Query 表示当前位置“想找什么”；
- Key 表示每个位置“可以按什么特征被匹配”；
- Value 表示匹配后真正汇入的信息。

在标准 MHA 中，把最后一维拆成 $h=h_q=h_{kv}$ 个头：

$$
Q,K,V\in\mathbb{R}^{B\times h\times L\times d_h}
$$

每个头在不同的投影子空间中建立关系。

### 4.2 缩放点积、Mask 与 Softmax

单头分数：

$$
S=\frac{QK^\top}{\sqrt{d_h}}+M
$$

除以 $\sqrt{d_h}$ 是为了避免维度增大时点积方差过大，使 Softmax 过早饱和。

更一般地，因果性应比较 Query 与 Key 的**逻辑位置**。设第 $i$ 个 Query 的位置为 $p_i^{Q}$，第 $j$ 个 Key 的位置为 $p_j^{K}$：

$$
M_{ij}=
\begin{cases}
0,&p_j^{K}\le p_i^{Q}\ \text{且 Key }j\text{ 有效}\\
-\infty,&\text{其他情况}
\end{cases}
$$

当 Q/K 都是同一条、从位置 0 开始且等长的序列时，它才简化为 $j\le i$。若 K 包含历史前缀、Q 只是末尾长度为 $L_q$ 的新块，且 $L_k\ge L_q$，连续位置下边界为：

$$
j\le L_k-L_q+i
$$

例如 Decode 时 $L_q=1$，唯一 Query 应能读取缓存中的全部有效 Key；若机械套用局部矩阵下标 $j\le i=0$，反而只会留下第一个 Key。Padding Mask 与因果 Mask 都作用于 Attention 可见性；它们与语言模型损失中的 label mask 是不同概念。

数值实现使用稳定 Softmax：

$$
\operatorname{softmax}(s_i)
=\frac{\exp(s_i-\max_j s_j)}
{\sum_k\exp(s_k-\max_j s_j)}
$$

得到权重：

$$
A=\operatorname{softmax}(S)
$$

再聚合 Value：

$$
O=AV
$$

多头结果拼接并投影：

$$
\operatorname{MHA}(H)
=\operatorname{Concat}(O_1,\ldots,O_h)W_O
$$

### 4.3 Attention 到底做了什么

Attention 不是简单寻找“最相似的词”，而是在当前层的特征空间中进行可学习的内容寻址：

1. Q/K 投影定义匹配标准；
2. 位置机制和 Mask 定义可见范围与距离先验；
3. Softmax 把相对分数变成竞争性权重；
4. V 决定被选中后传递什么信息。

同一个 Token 在不同层、不同头中可以承担完全不同的角色。

### 4.4 MHA、GQA、MQA 与 MLA

#### 4.4.1 MHA、GQA 与 MQA

标准 MHA 的 Query、Key、Value 都有 $h$ 个头。自回归推理时，历史 K/V 需要保存在 KV Cache 中，显存随层数和上下文长度近似线性增长。

|方案|Query 头|KV 头|特点|
|---|---:|---:|---|
|MHA|多个|与 Query 头一一对应|表达能力强，KV Cache 最大|
|GQA|多个|少于 Query 头，分组共享|在质量和推理成本之间折中|
|MQA|多个|各 1 个 K/V 头|KV Cache 最小，共享最强|

对 GQA，常见张量形状为：

$$
Q\in\mathbb{R}^{B\times h_q\times L\times d_h},
\qquad
K,V\in\mathbb{R}^{B\times h_{kv}\times L\times d_h}
$$

通常要求 $h_q$ 能被 $h_{kv}$ 整除。令 $g=h_q/h_{kv}$，每个 KV 头服务连续的 $g$ 个 Query 头；这是逻辑共享，不应为了计算方便就在 KV Cache 中物化成 $h_q$ 份副本。MQA 是 $h_{kv}=1$，MHA 是 $h_{kv}=h_q$ 的两个端点。

以每层 K/V Cache 为例，忽略量化和分页管理：

$$
\text{KV bytes}
\approx
2\cdot B\cdot L\cdot h_{kv}\cdot d_h\cdot
\text{bytes\_per\_element}
$$

前面的 2 对应 K 和 V。GQA/MQA 主要降低 $h_{kv}$，不会减少 Query 头数。

#### 4.4.2 MLA：把每个 Token 的 K/V 联合压缩成潜变量

Multi-head Latent Attention（MLA）的直接作用是**显著缩小自回归推理的 KV Cache，并降低每步读取历史缓存的带宽**。MHA、GQA、MQA 是通过减少 KV 头数来节省缓存；MLA 则保留多个 Query 头的表达能力，把所有头的 Key/Value 内容先压缩进每个 Token 共享的低维潜变量，需要使用时再通过各头的投影解释它。

设第 $t$ 个 Token 的层输入为 $h_t\in\mathbb R^d$。MLA 首先对 Key 和 Value 做低秩联合压缩：

$$
c_t^{KV}=W^{DKV}h_t,
\qquad
c_t^{KV}\in\mathbb R^{d_c},
\qquad
d_c\ll h d_h
$$

再由这个共享潜变量产生各个头的内容 Key 和 Value：

$$
k_{t,r}^{C}=W_r^{UK}c_t^{KV},
\qquad
v_{t,r}^{C}=W_r^{UV}c_t^{KV}
$$

这里的“联合”很重要：不是分别缓存一个压缩 Key 和一个压缩 Value，而是让二者共享同一个 $c_t^{KV}$。Query 也可以经过低秩瓶颈，以减少训练激活和投影成本：

$$
c_t^Q=W^{DQ}h_t,
\qquad
q_{t,r}^{C}=W_r^{UQ}c_t^Q
$$

**为什么还要把位置部分拆出来？** 若直接对上投影后的内容 Key 施加 RoPE，位置相关的旋转矩阵会夹在低秩投影和点积之间，使 $W_r^{UK}$ 无法在推理时吸收到 Query 侧；每次解码就可能需要重新展开历史 Key。MLA 因而使用 Decoupled RoPE，为内容和位置保留两条通道：

$$
q_{t,r}^{R}
=
\operatorname{RoPE}(W_r^{QR}c_t^Q,p_t),
\qquad
k_t^{R}
=
\operatorname{RoPE}(W^{KR}h_t,p_t)
$$

$$
q_{t,r}=[q_{t,r}^{C};q_{t,r}^{R}],
\qquad
k_{t,r}=[k_{t,r}^{C};k_t^{R}],
\qquad
v_{t,r}=v_{t,r}^{C}
$$

式中的 $k_t^R=\operatorname{RoPE}(W^{KR}h_t,p_t)$ 可以逐步理解为：$W^{KR}$ 先把当前位置的隐藏状态 $h_t$ 投影到较小的“位置 Key”空间，再由 RoPE 按逻辑位置 $p_t$ 对该向量的二维通道对施加旋转。它不是一个额外拼接的位置 ID，而是带内容条件的、具有相对相位性质的位置 Key；与 $q_{t,r}^R$ 的点积同时编码“当前位置与历史位置相距多远”及其对应的内容匹配。各 Query 头拥有自己的内容和位置 Query，但位置 Key $k_t^R$ 在所有头之间共享，因此它只需每个 Token 缓存一份。

正式 Attention 仍是 Softmax Attention：

$$
o_{t,r}
=
\sum_{s\le t}
\operatorname{softmax}_{s}
\left(
\frac{
(q_{t,r}^{C})^\top k_{s,r}^{C}
+
(q_{t,r}^{R})^\top k_s^{R}
}{\sqrt{d_h+d_h^R}}
\right)
v_{s,r}^{C}
$$

MLA 的高效推理并不要求真的为全部历史 Token 重建每头 $k_{s,r}^{C}$ 和 $v_{s,r}^{C}$。内容分数可以改写为：

$$
(q_{t,r}^{C})^\top W_r^{UK}c_s^{KV}
=
\left((W_r^{UK})^\top q_{t,r}^{C}\right)^\top c_s^{KV}
=
(q_{t,r}^{A})^\top c_s^{KV}
$$

即把 Key 的上投影吸收到 Query 侧。对 Value，也可以先对潜变量加权，再把 $W_r^{UV}$ 与输出投影合并：

$$
\sum_s a_{t,r,s}v_{s,r}^{C}
=
W_r^{UV}\left(\sum_s a_{t,r,s}c_s^{KV}\right)
$$

因此推理时每层、每个历史 Token 主要缓存：

$$
\boxed{c_t^{KV}\ \text{和}\ k_t^R}
$$

而不是完整的所有头 K/V。若只比较元素数，则每层每 Token 的缓存近似为：

$$
\underbrace{2h d_h}_{\text{MHA}}
\qquad\text{对比}\qquad
\underbrace{d_c+d_h^R}_{\text{MLA}}
$$

DeepSeek-V2 的设置为 $d_c=4d_h$、$d_h^R=d_h/2$，所以 MLA 每层每 Token 约缓存 $4.5d_h$ 个元素；它不再随 KV 头数按 $2h d_h$ 增长。

需要明确 MLA 的边界：它主要压缩“历史以什么形式保存和读取”，**不删除 Attention 连接**。若每个 Query 仍访问所有历史潜变量，Prefill 的核心注意力仍具有 $O(L^2)$ 项，Decode 的单步历史扫描仍为 $O(L)$。它与后文 DSA 可以叠加：MLA 先压缩每个 Token 的 KV 表示，DSA 再从这些潜变量中只选择少量相关 Token。

DeepSeek-V4 不再把 MLA 作为主体注意力。V4 的 CSA/HCA 改为在**序列维度**把多个 Token 的 KV 条目合并成一个压缩条目，再使用共享 Key=Value 的 MQA；这与 MLA 在**特征/头维度**把单个 Token 的多头 K/V 压成潜变量是两种不同路线，详见第 4.7.7 节。

### 4.5 训练 Attention 与增量推理

训练时，一次处理整个序列：

$$
Q,K,V\in\mathbb{R}^{L\times d}
$$

因果 Mask 保证位置 $t$ 不读取未来，但所有位置仍可并行计算。

自回归推理到第 $t$ 步时，只为新 Token 计算 $q_t,k_t,v_t$，把 $k_t,v_t$ 追加到缓存，再计算：

$$
o_t
=\operatorname{softmax}
\left(
\frac{q_tK_{\le t}^\top}{\sqrt{d_h}}
\right)V_{\le t}
$$

KV Cache 避免重复计算历史 K/V，但每一步仍要读取越来越长的历史缓存。

实际在线推理通常分成两个阶段：

- **Prefill**：一次并行处理整个 Prompt，把每层所有 Prompt Token 的 K/V 写入缓存。它像训练前向一样能并行处理序列，但不保存反向激活；稠密 Attention 的 Prompt 计算仍具有二次长度关系。
- **Decode**：每一步只处理一个或少量新 Token，Query 读取此前全部有效 K/V。单步矩阵形状较小，却经常受 KV Cache 读取带宽和请求批处理效率限制。

如果初始 Prompt 长度为 $L_p$，随后生成 $T$ 个 Token，忽略常数和投影成本，Decode 期间读取历史的总长度近似为：

$$
\sum_{i=0}^{T-1}(L_p+i)
=
TL_p+\frac{T(T-1)}2
$$

因此 KV Cache 消除了“每步重算整个前缀”的大量重复工作，但没有让无限长生成变成常数成本。分页缓存和连续批处理主要优化内存布局与调度，不改变数学上的注意力连接；滑动窗口则通过丢弃窗口外连接来限制有效历史长度，确实改变了可见范围，不能与前两者混为一类。

### 4.6 复杂度与 FlashAttention

稠密 Attention 的分数矩阵有 $L^2$ 个元素，因此理论计算量通常写为：

$$
O(L^2d)
$$

核心二次项来自 $QK^\top$ 和 $AV$。对 batch size 为 $B$、头数为 $h$ 的实现，分数与注意力权重的形状均为 $B\times h\times L\times L$；朴素实现若显式保存它们，中间激活内存为 $O(BhL^2)$。

FlashAttention 仍然计算 Softmax，且对相同 Mask 和精度约定计算的是稠密的精确 Attention（允许浮点归约顺序造成微小数值差异）。它不物化完整分数矩阵或注意力权重矩阵，而是把 Q、K、V 分块，在片上存储中逐块处理。对某个 Query 行，处理到当前 Key 块时维护已见分数的行最大值、归一化分母和 Value 加权累计量：

$$
m=\max_{j\in\mathrm{seen}} s_j,\qquad
\ell=\sum_{j\in\mathrm{seen}} e^{s_j-m},\qquad
a=\sum_{j\in\mathrm{seen}} e^{s_j-m}v_j
$$

一个具体的分块例子可以看清这个过程。为简化计算，只考虑一个 Query，并把每个 Value 写成标量；设已经包含缩放和 Mask 的 Attention 分数以及对应 Value 为：

$$
s=[1,2,3,4],\qquad v=[10,20,30,40]
$$

若每次只读取两个 Key/Value，第一块包含 $(s_1,s_2)=[1,2]$。以块内最大值 2 为基准，得到：

$$
\begin{aligned}
m_1&=2,\\
\ell_1&=e^{1-2}+e^{2-2}\approx 1.3679,\\
a_1&=e^{1-2}\cdot 10+e^{2-2}\cdot 20\approx 23.6788
\end{aligned}
$$

第二块的分数为 $[3,4]$，新的全局最大值变成 $m_2=4$。此时不能直接把两个块的累计量相加，因为第一块以 2 为指数基准、第二块以 4 为指数基准；需要先用 $e^{m_1-m_2}=e^{-2}$ 把旧累计量换算到新基准：

$$
\begin{aligned}
\ell_2
&=e^{m_1-m_2}\ell_1+e^{3-m_2}+e^{4-m_2}\\
&=e^{-2}\cdot 1.3679+e^{-1}+1
\approx 1.5530,\\[4pt]
a_2
&=e^{m_1-m_2}a_1+e^{3-m_2}\cdot 30+e^{4-m_2}\cdot 40\\
&=e^{-2}\cdot 23.6788+e^{-1}\cdot 30+40
\approx 54.2410
\end{aligned}
$$

于是最终输出为：

$$
o=\frac{a_2}{\ell_2}\approx\frac{54.2410}{1.5530}\approx 34.9265
$$

如果一次性计算完整 Softmax，权重约为 $[0.0321,0.0871,0.2369,0.6439]$，与 $[10,20,30,40]$ 加权后同样得到 $34.9265$。区别在于，分块算法在任何时刻只需保存当前块以及 $(m,\ell,a)$，不必把完整的四个分数和四个权重都作为中间矩阵保存在 HBM 中。真实实现会同时处理一块 Query，且 $a$ 是与 Value 维度相同的向量，但更新原理完全一致。

一般地，每读入一个新块就在线更新这三个量，最终直接输出 $o=a/\ell$。因此它把“$QK^\top\rightarrow$ 完整 Softmax 矩阵 $\rightarrow AV$”改为流式分块累计；反向传播时按需重计算局部块。其收益是减少 HBM 读写和中间激活内存、提高实际吞吐，而不是取消 Softmax 或改变稠密 Attention 的 $O(L^2d)$ 二次计算关系。

滑动窗口、块稀疏、局部加少量全局 Token 等方法，只计算选定连接，可降低长序列成本；代价是被剪掉的连接不能直接传递信息，需要跨层传播、全局通道或检索机制补偿。

### 4.7 Sparse Attention：只计算有价值的 Token 连接

FlashAttention 没有减少 Attention 连接数，只是以更低的 IO 和中间存储成本计算完整的稠密 Attention。Sparse Attention 则直接改变可见连接：每个 Query 不再读取所有历史 Key，而是只读取固定规则或动态检索得到的一个子集。

#### 4.7.1 统一公式与复杂度

对因果 Self-Attention，位置 $i$ 的稠密输出为：

$$
o_i
=
\sum_{j\le i}
\operatorname{softmax}_{j}
\left(
\frac{q_i^\top k_j}{\sqrt{d_h}}
\right)v_j
$$

Sparse Attention 为每个 Query 定义允许访问的位置集合 $\mathcal S_i$，并使用稀疏 Mask：

$$
M_{ij}
=
\begin{cases}
0,&j\in\mathcal S_i\\
-\infty,&j\notin\mathcal S_i
\end{cases}
$$

于是只在集合内部归一化：

$$
o_i
=
\sum_{j\in\mathcal S_i}
\operatorname{softmax}_{j\in\mathcal S_i}
\left(
\frac{q_i^\top k_j}{\sqrt{d_h}}
\right)v_j
$$

总计算量取决于实际保留的边数：

$$
O\left(
d_h\sum_{i=1}^{L}|\mathcal S_i|
\right)
$$

如果每个 Query 最多读取 $K$ 个 Key，则核心 Attention 约为：

$$
O(LKd_h)
$$

当 $K$ 不随 $L$ 增长时，它对序列长度近似线性。不过理论 FLOPs 下降不保证实际延迟同比下降：零散的单 Token 选择会带来不规则访存、Top-K 和小矩阵调度开销，因此工程实现通常使用规则窗口或块对齐的结构化稀疏。

#### 4.7.2 固定稀疏模式：窗口、全局节点与块连接

**滑动窗口注意力（Sliding Window Attention）**让每个因果 Query 只读取最近 $W$ 个位置：

$$
\mathcal S_i
=
\{j\mid \max(1,i-W+1)\le j\le i\}
$$

复杂度从 $O(L^2d_h)$ 降为 $O(LWd_h)$；Decode 时，使用局部窗口的层也只需保留最近 $W$ 个位置的 KV。Mistral 7B 是 Decoder LLM 使用滑动窗口的代表。

**局部窗口加全局 Token**在窗口之外保留少量全局节点：

$$
\mathcal S_i
=
\operatorname{LocalWindow}(i)
\cup
\operatorname{GlobalTokens}
$$

普通 Token 可以通过全局节点跨窗口通信，全局节点则读取整个有效序列。Longformer 使用“局部滑动窗口 + 任务指定的全局注意力”，适合长文档分类、问答和信息抽取。因果生成模型若使用全局节点，仍必须保证这些节点没有汇总未来 Token。

**局部层与全局层交错**不设置特殊全局 Token，而是让大多数层使用滑动窗口，周期性插入稠密全局 Attention。Gemma 3 的重复模式是 5 个窗口大小为 1024 的局部层配 1 个全局层：局部层降低计算和 KV Cache 成本，全局层定期建立任意远距离位置之间的直接连接。

**块稀疏注意力（Block Sparse Attention）**先把 Token 划为连续块，再规定 Query Block 可以访问哪些 Key Block。BigBird 的典型模式组合了：

- 邻近块，建模局部连续性；
- 少量全局 Token，汇总和广播全局信息；
- 少量随机远程块，缩短远距离位置之间的传播路径。

块稀疏比任意 Token 级稀疏更容易转换为规则矩阵乘法，也更适合 GPU；但固定模式无法根据当前问题判断某个远程块是否真正重要。

#### 4.7.3 为什么滑动窗口的跨层视野约为 $nW$

第 $n$ 层读取的是第 $n-1$ 层已经融合过局部上下文的表示，因此信息可以逐层越过窗口边界。假设窗口大小 $W$ 包含当前 Token，第 1 层位置 $i$ 的可见范围为：

$$
[i-W+1,\ i]
$$

第 2 层读取该区间内所有位置在第 1 层的表示；这些表示各自又覆盖一个宽度为 $W$ 的窗口。取所有范围的并集，最远可以追溯到 $i-2(W-1)$。递推到第 $n$ 层：

$$
\operatorname{Range}_n(i)
=
[i-n(W-1),\ i]
$$

所以精确的理论感受野大小为：

$$
R_n=1+n(W-1)\approx nW
$$

例如 $W=3$，考察 Token 8：

```text
第 0 层：{8}
第 1 层：{6, 7, 8}
第 2 层：{4, 5, 6, 7, 8}
第 3 层：{2, 3, 4, 5, 6, 7, 8}
```

这依赖相邻窗口发生重叠。如果每层都使用边界完全相同且互不连通的固定分块，层数增加也不能跨块传播。还要区分“理论上存在传播路径”和“能高质量保留信息”：远程细节经过多层加权、压缩和混合后可能衰减，所以混合架构仍会保留全局层或内容检索通道。

#### 4.7.4 动态稀疏注意力：低成本索引与块摘要

固定窗口容易漏掉关键的远程内容。动态 Sparse Attention 根据当前 Query 选择候选位置：

$$
\mathcal S_i
=
\operatorname{TopK}_{j}
\operatorname{Score}(q_i,k_j)
$$

但若先计算所有 $q_i^\top k_j$ 再做 Top-K，索引阶段本身已经付出 $O(L^2d_h)$，并没有解决问题。实际系统通常采用“低成本粗筛 + 原始 K/V 精算”：

```text
原始 K/V
   ↓ 按连续 Token 分块
低维块摘要
   ↓ 轻量索引器粗粒度打分
Top-K 相关块
   ↓ 展开为原始 Token
只在候选 Token 上计算精确 Softmax Attention
```

设每块有 $B$ 个 Token，第 $b$ 块的摘要可以是平均池化：

$$
s_b
=
\frac1B\sum_{j\in b}k_j
$$

也可以先投影到较低维度 $d_s\ll d_h$，再使用池化或学习式加权：

$$
\widetilde k_j=k_jW_s,
\qquad
s_b=\operatorname{Pool}\left(\{\widetilde k_j:j\in b\}\right)
$$

Query 同样投影到索引空间，用便宜的分数选择块：

$$
\widetilde q_i=q_iW_q,
\qquad
r_{ib}=\widetilde q_i^\top s_b
$$

例如 $L=65536$、$d_h=128$、$B=64$、$d_s=32$：直接让一个 Query 与所有 Key 打分需要约 $65536\times128$ 次乘加；粗筛只需检查 $1024$ 个摘要，即约 $1024\times32$ 次乘加。若再选择 8 个块，正式 Attention 只处理 $8\times64=512$ 个原始 Token。摘要通常只负责导航，最终 Value 加权仍使用候选块里的原始 K/V，因此摘要不必承载全部细节。

扫描全部块摘要仍有 $O(L^2/B)$ 的索引项，只是比逐 Token 索引少约 $B$ 倍。更长上下文还可以使用聚类倒排表、局部敏感哈希或分层摘要树：先定位少量语义簇、章节或上层节点，再检查其中的块。不过近似索引会引入漏召回，动态索引还必须处理自回归过程中不断追加的新 K/V。块摘要也不是唯一方案：后文 DSA 使用低维、少头、可用 FP8 计算的 Token 级索引器，保留更细的选择粒度，但索引阶段仍含 $O(L^2)$ 项。

Native Sparse Attention（NSA）是动态分层稀疏的代表：用压缩路径获得粗粒度全局上下文，用选择路径读取重要远程块，再以局部窗口保留近期细节。三条路径分别弥补纯压缩、纯检索和纯窗口的能力缺口。

#### 4.7.5 Qwen Sparse Attention：微块级动态检索

Qwen3.8-Flash-Next 中的 QSA（Qwen Sparse Attention）属于**学习式索引器驱动的动态微块稀疏检索注意力**，不是固定滑动窗口，也不是 BigBird 式预设随机连接。其基本流程为：

```text
隐藏状态 X
      ↓ Indexer 自己的 Q/K 投影
Token 级 Indexer Query/Key
      ↓ 每 4 个 Indexer Key 聚合
micro-block Indexer Key
      ↓ 轻量 MQA 索引器按当前 Query 打分
选择最多 512 个相关块
      ↓ 展开块索引
从正式 Attention KV Cache 收集最多 2048 个原始 Token
      ↓
计算正式 Softmax Attention
```

QSA 中存在两套用途不同的投影：Indexer 使用独立的 $W_{Q,I},W_{K,I}$ 生成检索表示，只负责决定“读哪里”；正式 Attention 使用自己的 $W_Q,W_K,W_V$ 生成内容表示，负责“读出什么”。Indexer 不需要 Value 投影：

$$
q_{i,h}^{I}=x_iW_{Q,I}^{h},
\qquad
k_i^{I}=x_iW_{K,I}
$$

Qwen3.8-Flash-Next 使用压缩率 $r=4$。对第 $b$ 个完整 micro-block，Indexer 先在 FP32 中平均四个尚未施加位置旋转的 128 维原始 Key：

$$
u_b
=
\frac14\sum_{a=0}^{3}k_{4b+a}^{I}
$$

再进行 L2 归一化：

$$
\widehat u_b
=
\frac{u_b}{\|u_b\|_2+\varepsilon}
$$

最后以块内第一个 Token 的位置 $p_b=4b$ 为锚点施加 MRoPE：

$$
\bar k_b^{I}
=
\operatorname{MRoPE}
\left(
\widehat u_b,p_b
\right)
$$

所以这里压缩的是 Indexer Key 的**序列长度**：$L$ 个 128 维 Key 变成约 $L/4$ 个 128 维 Block Key，向量维度没有从 128 继续降低。先平均内容向量、再统一使用块锚点旋转，也避免四个不同位置的 RoPE/MRoPE 相位在平均时相互抵消。FP32 平均则减小低精度累加误差。

当前位置的四个 Indexer Query Head 与共享 Block Key 打分：

$$
I_{ib}
=
\frac1{\sqrt{128}}
\sum_{h=1}^{4}
\operatorname{ReLU}
\left(
\left\langle q_{i,h}^{I},\bar k_b^{I}\right\rangle
\right)
$$

ReLU 去掉负相关贡献，四个 Query Head 的正相关分数相加，形成该 Query 对块 $b$ 的重要性估计。块摘要只需完成粗粒度排序，不负责承载最终输出所需的完整 Token 细节。

若压缩率为 $r$、Token 预算为 $K$，块预算为：

$$
K_B=\left\lceil\frac Kr\right\rceil
$$

索引器先在满足因果约束的块中计算重要性 $I_{ib}$，再选择：

$$
\beta_i
=
\operatorname{TopK}_{K_B}
\left(\{I_{ib}\}_b\right)
$$

选中的块被展开为原始 Token 索引；若当前 micro-block 尚未填满，还会附加其中 0 至 3 个有效历史 Token，避免最近位置因块未完成而不可见。正式 QSA 随后从未压缩的 Attention KV Cache 中收集这些原始 K/V，再使用高维 Q/K/V 计算缩放点积与 Softmax：

$$
o_i
=
\operatorname{softmax}
\left(
\frac{q_iK_{\beta_i}^{\top}}{\sqrt{d_h}}
\right)V_{\beta_i}
$$

因此，“四合一平均”只发生在检索用的 Indexer Key 上，不会把正式 Attention 的四组 K/V 平均成一组。Qwen3.8-Flash-Next 的公开配置中，正式 QSA 使用 24 个 Query 头、2 个 KV 头和 256 维 Head；轻量索引器使用 4 个 Query 头、1 个共享 Key 头和 128 维 Head。固定预算为 512 个四 Token micro-block，即最多 2048 个原始 Token。

QSA 降低了两部分成本：正式 Attention 从全历史缩减到固定候选预算，索引器则从逐 Token 打分改为微块打分。若每块压缩 $r$ 个 Token，完整扫描块摘要的索引项从 $O(L^2)$ 降为约 $O(L^2/r)$；它仍含二次项，并非单独把全部计算变成严格线性，但索引器本身更小，核心 Sparse Attention 在固定预算下约为 $O(LK)$。

该模型还以 3:1 的层比例混合 Gated DeltaNet 与 QSA：

```text
Gated DeltaNet → Gated DeltaNet → Gated DeltaNet → QSA
```

Gated DeltaNet 把历史持续压缩进固定状态，适合低成本传播“历史大意”；每隔三层出现的 QSA 回到原始 KV 中检索重要 micro-block，补充固定状态难以无损保存的细节。QSA 与 NSA 都属于动态块选择，但 QSA 重点压缩索引过程，并依赖 Gated DeltaNet/QSA 的层间混合；它本身不等同于“压缩 + 选择 + 滑窗”三分支 NSA。

模型名称中的 FP8 只表示该检查点的权重/计算量化格式，不决定 QSA 的连接模式；QSA 的四 Token micro-block 也不能与 FP8 权重的量化 Block Size 混为一谈。

#### 4.7.6 DeepSeek Sparse Attention：Token 级 Lightning Indexer + MLA

DeepSeek Sparse Attention（DSA）的作用是**让每个 Query 只对少量最相关的历史 Token 执行昂贵的正式 Attention**。它在 DeepSeek-V3.2 中由两个组件构成：

1. **Lightning Indexer**：用较少的头和较低维表示，快速估计当前 Query 与每个历史 Token 的相关性；
2. **Fine-grained Token Selection**：选择分数最高的 $k$ 个 Token，再在这些位置上执行正式 MLA。

DSA 的索引器拥有自己的 Query/Key 投影，但不需要 Value。设当前 Token 为 $h_t$、候选历史 Token 为 $h_s$，索引器产生 $H^I$ 个 Query Head、一个共享 Key 和每头权重：

$$
q_{t,j}^{I}=f_j^Q(h_t),
\qquad
k_s^{I}=f^K(h_s),
\qquad
w_{t,j}^{I}=f_j^W(h_t)
$$

其 Token 相关性分数为：

$$
I_{t,s}
=
\sum_{j=1}^{H^I}
w_{t,j}^{I}
\operatorname{ReLU}
\left((q_{t,j}^{I})^\top k_s^{I}\right)
$$

ReLU 丢弃负相关贡献，$w_{t,j}^{I}$ 让当前 Query 动态决定各索引头的重要性。索引器只回答“应该读哪些位置”，不负责生成 Attention 输出；正式内容仍由 MLA 的潜变量和位置分量提供。

在满足因果约束的历史位置中取 Top-$k$：

$$
\mathcal S_t
=
\operatorname{TopK}_{s\le t}
\left(I_{t,s},k\right)
$$

随后只读取被选中的 MLA 条目并计算精确 Softmax：

$$
u_t
=
\operatorname{Attn}_{\mathrm{MLA}}
\left(
h_t,
\{c_s:s\in\mathcal S_t\}
\right)
$$

这里 $c_s$ 表示 MLA 在位置 $s$ 缓存的低维 KV 潜变量以及所需的位置分量。DeepSeek-V3.2 在 MLA 的 MQA 计算模式上实现 DSA：多个 Query Head 仍保留，但被选中的同一份潜变量由所有 Query Head 共享，便于稀疏内核复用数据。

整体数据流可以概括为：

```text
当前隐藏状态 h_t ──→ Lightning Indexer Query + Head Weight ─┐
历史隐藏状态 h_s ──→ Lightning Indexer Key Cache ───────────┤
                                                            ▼
                                          对全部历史 Token 计算轻量分数
                                                            │
                                                      Token 级 Top-k
                                                            │
历史 MLA Cache：{c_s^KV, k_s^R} ────────────────────────────┤
                                                            ▼
                                         只对选中 Token 执行正式 MLA
                                                            │
                                                            ▼
                                                       Attention 输出
```

例如当前 Query 有 8 个可见历史位置，索引分数为：

$$
[0.2,1.8,0.1,2.1,0.6,1.4,0.3,0.5]
$$

若 $k=3$，则选中第 4、2、6 个 Token。正式 MLA 不再读取其余 5 个位置，只在 $\{c_4,c_2,c_6\}$ 上重新计算真正的多头 Attention 分数、Softmax 和 Value 聚合。选择可以跨越很远且不连续，因此 DSA 不是固定窗口。

**Indexer 怎样学会选？** Top-$k$ 排序本身不可微，而且随机初始化的索引器一开始没有可靠召回。DeepSeek-V3.2 采用两阶段继续训练：

1. **Dense Warm-up**：保持主模型的稠密 Attention，冻结主模型，只训练索引器。把正式 Attention 在多个头上的重要性聚合并归一化为教师分布 $p_{t,:}$，让索引分布逼近它：

   $$
   \mathcal L_I
   =
   \sum_t
   D_{\mathrm{KL}}
   \left(
   p_{t,:}
   \middle\|
   \operatorname{softmax}(I_{t,:})
   \right)
   $$

2. **Sparse Training**：启用 Top-$k$ 稀疏正式 Attention，让主模型用语言模型损失适应漏选和稀疏模式；索引器继续在已选集合上用 KL 损失学习。索引器输入从主模型计算图中 detach，使主模型和索引器的优化信号分开。

DeepSeek-V3.2 的公开训练配置为每个 Query 选择 2048 个 KV Token。这个数是模型实例的预算，不是 DSA 原理要求的固定常数。

**复杂度必须分两部分看。** 若正式 MLA 的头数和维度对应成本记为 $d_m$，轻量索引器成本记为 $H^Id_I$：

$$
\underbrace{O(L^2H^Id_I)}_{\text{Lightning Indexer}}
+
\underbrace{O(Lkd_m)}_{\text{正式稀疏 MLA}}
$$

正式 Attention 从 $O(L^2d_m)$ 降为 $O(Lkd_m)$；但 Token 级索引器仍扫描全部历史，理论上保留 $O(L^2)$ 项。它之所以在工程上仍能显著加速，是因为索引头少、维度小、可用 FP8 计算，而正式 MLA 的高维多头计算与大量 Value 读取只发生在 $k\ll L$ 个位置。Decode 时每个新 Token 的索引扫描约为 $O(LH^Id_I)$，正式 Attention 约为 $O(kd_m)$。

DSA 也不会把 KV Cache 的长度从 $O(L)$ 变成常数：为了让未来 Query 可能检索任意历史 Token，MLA 潜变量和索引 Key 仍需保存。它降低的是正式 Attention 的计算量和被选中 KV 的读取带宽，并以额外索引缓存、Top-$k$ 和不规则访存为代价。

DSA 与前文 QSA 很像，但粒度和底层表示不同：

|维度|DSA（DeepSeek-V3.2）|QSA（Qwen3.8-Flash-Next）|
|---|---|---|
|索引粒度|单 Token|每 4 Token 的 micro-block|
|Indexer 聚合|带可学习 Head Weight 的多头 ReLU 分数|4 个 Query Head 的正相关分数求和|
|正式缓存|MLA 的低维潜变量与位置分量|正式 Attention 的原始 K/V|
|候选预算示例|2048 个 Token|512 个块，即最多 2048 个 Token|
|索引复杂度|Token 级扫描，仍含 $O(L^2)$|块压缩后约 $O(L^2/4)$ 个索引项|

一句话区分 MLA 与 DSA：**MLA 决定每个历史 Token“存成什么”，DSA 决定当前 Query“读取其中哪些 Token”**。MLA 可以单独运行稠密 Attention；DeepSeek-V3.2 的 DSA 则建立在 MLA 的压缩缓存之上，同时获得缓存压缩和连接稀疏两类收益。

#### 4.7.7 DeepSeek-V4：CSA 与 HCA 混合压缩注意力

DeepSeek-V4 使用 **Compressed Sparse Attention（CSA）与 Heavily Compressed Attention（HCA）交错的混合注意力**。两者先把多个 Token 的 KV 条目学习式压缩成一个条目，再分别采用两种互补的读取方式：

- **CSA**：低压缩率保留较细粒度信息，再用 DSA 从压缩条目中选择 Top-$k$；
- **HCA**：用很高的压缩率形成短小的全局摘要序列，然后稠密读取全部可见压缩条目；
- **共同的滑动窗口分支**：直接读取最近的未压缩 Token，补回块内和局部细节。

整体结构可以概括为：

```text
                         ┌─ 最近 W 个原始 KV ───────────────┐
原始隐藏状态 H ──────────┤                                  ├─→ Core Attention
                         │                                  │
                         ├─ CSA：m:1 压缩 → DSA Top-k ──────┤
                         │                                  │
                         └─ HCA：m':1 重压缩 → 全部读取 ────┘

不同层只启用 CSA 或 HCA 中的一种长程分支，但每层都保留局部滑窗分支。
```

##### 4.7.7.1 CSA：先压缩 Token，再对压缩条目做 DSA

设当前层输入序列为：

$$
H\in\mathbb R^{n\times d}
$$

其中 $n$ 为序列长度、$d$ 为隐藏维度。CSA 先为每个 Token 产生两组候选 KV 内容和两组通道级压缩 Logit：

$$
C^a=HW^{aKV},
\qquad
C^b=HW^{bKV}
$$

$$
Z^a=HW^{aZ},
\qquad
Z^b=HW^{bZ}
$$

其中 $C^a,C^b,Z^a,Z^b\in\mathbb R^{n\times c}$，$c$ 是共享 KV Head 的维度。对第 $i$ 个压缩条目，CSA 同时查看当前的 $m$ 个 Token 和前一组 $m$ 个 Token，并加入可学习位置偏置：

$$
\begin{aligned}
&[S^a_{mi:m(i+1)-1};S^b_{m(i-1):mi-1}]\\
&\quad=
\operatorname{softmax}_{\mathrm{row}}
\left(
[Z^a_{mi:m(i+1)-1}+B^a;
Z^b_{m(i-1):mi-1}+B^b]
\right)
\end{aligned}
$$

再进行逐通道加权池化：

$$
C_i^{\mathrm{Comp}}
=
\sum_{j=mi}^{m(i+1)-1}S_j^a\odot C_j^a
+
\sum_{j=m(i-1)}^{mi-1}S_j^b\odot C_j^b
$$

这里 $\odot$ 是逐元素乘法；Softmax 对 $2m$ 个候选位置按通道归一化，所以 $S_j^a,S_j^b$ 是 $c$ 维权重向量，而不是整块共用一个标量。相邻压缩条目覆盖的原始区间存在重叠，有利于缓解硬块边界的信息损失；虽然每个条目融合了最多 $2m$ 个 Token，输出步长仍为 $m$，所以条目数约从 $n$ 变成 $n/m$。

CSA 对 Indexer Key 执行相同的序列压缩，得到：

$$
K^{\mathrm{IComp}}
\in
\mathbb R^{(n/m)\times c^I}
$$

当前位置 $t$ 的索引 Query 通过低秩投影产生：

$$
c_t^Q=h_tW^{DQ},
\qquad
[q_{t,1}^I;\ldots;q_{t,n_h^I}^I]
=
c_t^QW^{IUQ}
$$

各索引头的动态权重为：

$$
[w_{t,1}^I;\ldots;w_{t,n_h^I}^I]
=
h_tW^w
$$

然后复用 DSA 的 Lightning Indexer 形式，对已经完成且满足因果约束的压缩块打分：

$$
I_{t,s}
=
\sum_{r=1}^{n_h^I}
w_{t,r}^I
\operatorname{ReLU}
\left(
(q_{t,r}^I)^\top K_s^{\mathrm{IComp}}
\right)
$$

$$
\mathcal C_t^{\mathrm{SprsComp}}
=
\left\{
C_s^{\mathrm{Comp}}
\mid
I_{t,s}\in\operatorname{TopK}(I_{t,:},k)
\right\}
$$

**Indexer 与正式 Attention 哪些共享、哪些分开？** 这里存在两个容易混淆的“共享”。$K^{\mathrm{IComp}}$ 不是直接复用 $C^{\mathrm{Comp}}$；它是由专用的 Indexer Key 投影生成、再采用与 CSA 相同的序列压缩规则得到的轻量检索 Key。$K^{\mathrm{IComp}}$ 只用于为所有候选块快速排序，$C^{\mathrm{Comp}}$ 才会在选中后送入正式 Attention，并同时充当该模块的 Key 和 Value：

$$
o_{t,r}
=
\operatorname{CoreAttn}
\left(
\text{query}=q_{t,r},
\text{key}=\mathcal C_t^{\mathrm{SprsComp}},
\text{value}=\mathcal C_t^{\mathrm{SprsComp}}
\right)
$$

Query 也分为两条最终向量，但共享同一个低维来源：

$$
c_t^Q=h_tW^{DQ},
\qquad
[q_{t,1}^I;\ldots;q_{t,n_h^I}^I]=c_t^QW^{IUQ},
\qquad
[q_{t,1};\ldots;q_{t,n_h}]=c_t^QW^{UQ}
$$

其中 $q^I$ 与 $K^{\mathrm{IComp}}$ 配对，用于 Lightning Indexer 的 Top-$k$ 粗筛；$q$ 与选中的 $C^{\mathrm{Comp}}$ 配对，用于正式多头 Attention。换言之，**Query 共享 Query latent $c_t^Q$，但 Indexer/主注意力的上投影不同；Key 不共享；而主注意力内部的 $C^{\mathrm{Comp}}$ 被所有 Query Head 共享，并且 Key=Value。** 分离检索 Key 与正式 KV，让 Indexer 可采用更小维度和更低精度，而不必把主注意力的内容表达限制为“只适合排序”的表示。

与 DeepSeek-V3.2 的 DSA 相比，关键变化是：V3.2 的索引器搜索原始 Token 对应的 MLA 潜变量，V4 CSA 搜索已经缩短到约 $n/m$ 的压缩条目。因此正式 Attention 和 Indexer 的搜索空间都先缩小了一次。

##### 4.7.7.2 HCA：更重地压缩，然后稠密读取全局摘要

Heavily Compressed Attention（HCA）不做 Top-$k$。它先产生一组共享 KV 内容和压缩 Logit：

$$
C=HW^{KV},
\qquad
Z=HW^Z
$$

对每个互不重叠的 $m'$ Token 块进行通道级 Softmax 加权：

$$
S_{m'i:m'(i+1)-1}
=
\operatorname{softmax}_{\mathrm{row}}
\left(
Z_{m'i:m'(i+1)-1}+B
\right)
$$

$$
C_i^{\mathrm{Comp}}
=
\sum_{j=m'i}^{m'(i+1)-1}
S_j\odot C_j
$$

HCA 把长度从 $n$ 压缩到约 $n/m'$。因为 $m'\gg m$，压缩后的全局序列已经很短，所以每个 Query 可以稠密读取全部可见压缩条目，不再承担 Lightning Indexer 和 Top-$k$ 的成本。它更像“低分辨率全局摘要通道”，而 CSA 更像“较高分辨率的内容检索通道”。

##### 4.7.7.3 Shared K=V MQA、局部窗口与位置处理

CSA 和 HCA 都不沿用 MLA 的“低维潜变量再分别上投影为 Key/Value”。它们直接让同一个压缩向量同时充当 Key 和 Value，并由多个 Query Head 共享：

$$
[q_{t,1};\ldots;q_{t,n_h}]
=
c_t^QW^{UQ}
$$

$$
o_{t,r}
=
\operatorname{CoreAttn}
\left(
\text{query}=q_{t,r},
\text{key}=\mathcal C_t,
\text{value}=\mathcal C_t
\right)
$$

其中 CSA 的 $\mathcal C_t$ 是 Top-$k$ 压缩条目，HCA 的 $\mathcal C_t$ 是全部可见压缩条目。这个 Shared K=V MQA 先让 K/V 共享，再通过序列压缩减少条目数量。

为了弥补压缩造成的细节损失，每个 CSA/HCA 层还把最近 $n_{\mathrm{win}}$ 个未压缩 KV 条目加入同一次 Core Attention。它有两个作用：

1. 当前压缩块尚未闭合时，Query 仍能读取块内更早的 Token；
2. 高频的邻近依赖不必先经过长程压缩再恢复。

此外，V4 的注意力还包含以下配套设计：

- **Query/KV RMSNorm**：在 Core Attention 前分别归一化各 Query Head 和唯一的 KV Head，抑制 Attention Logit 爆炸；
- **Partial RoPE**：只对 Query 和压缩 KV 的最后 64 维应用 RoPE；由于同一条目也充当 Value，聚合输出的最后 64 维再施加位置 $-t$ 的反向 RoPE，把绝对相位还原为相对位置信息；
- **Attention Sink**：每个头增加一个可学习的 Sink Logit，使真实 KV 的 Attention 权重总和可以小于 1，允许当前 Query 选择“少读甚至不读”；
- **Grouped Output Projection**：先把多个 Head 分组降到较小的中间维度，再投影回隐藏维度，降低超宽 Attention 输出投影的计算量。

##### 4.7.7.4 层间混合、复杂度与实际配置

忽略滑动窗口和常数，CSA 与 HCA 的主要长度成本可近似写为：

$$
\begin{aligned}
\text{CSA Indexer}&:\ O(n^2/m),\\
\text{CSA Core Attention}&:\ O(nk),\\
\text{HCA Core Attention}&:\ O(n^2/m')
\end{aligned}
$$

CSA 的 Indexer 仍含二次项，但搜索对象从 $n$ 个 Token 变为约 $n/m$ 个压缩条目，并使用低维多头表示和 FP4 QK 路径；HCA 保留稠密全局连接，但用很大的 $m'$ 降低常数。两者交错后，一类层负责高分辨率稀疏检索，另一类层负责低分辨率全局汇总。

DeepSeek-V4 的公开配置为：

|配置|V4-Flash|V4-Pro|
|---|---:|---:|
|总层数|43|61|
|起始层|前 2 层纯滑窗|前 2 层 HCA|
|后续层|CSA/HCA 交错|CSA/HCA 交错|
|CSA 压缩率 $m$|4|4|
|CSA Top-$k$|512 个压缩条目|1024 个压缩条目|
|HCA 压缩率 $m'$|128|128|
|局部窗口 $n_{\mathrm{win}}$|128 Token|128 Token|
|Query Head 数|64|128|
|共享 KV Head 数|1|1|
|Head 维度 $c$|512|512|

在一百万 Token 场景下，官方报告估计 V4-Pro 的单 Token 推理 FLOPs 和 KV Cache 分别约为 DeepSeek-V3.2 的 27% 和 10%；V4-Flash 分别约为 10% 和 7%。这些是具体模型与精度配置下的工程结果，不应理解为 CSA/HCA 对所有硬件和序列长度都保证相同比例。

三代注意力路线可以概括为：

```text
DeepSeek-V3：   每 Token 的多头 K/V → MLA 特征维压缩 → 稠密读取
DeepSeek-V3.2： MLA 压缩条目 → Token 级 DSA Top-k → 稀疏读取
DeepSeek-V4：   多 Token → CSA/HCA 序列维压缩
                           ├─ CSA：压缩后 DSA Top-k
                           ├─ HCA：压缩后稠密全局读取
                           └─ 两者均叠加未压缩局部滑窗
```

#### 4.7.8 与其他高效 Attention 路线的边界

|方法|是否改变可见连接|历史如何保存|对长度的主要成本|典型用途或代表|
|---|---|---|---|---|
|标准稠密 Attention|否，读取全部允许位置|完整 K/V|$O(L^2d_h)$|标准 Transformer|
|FlashAttention|否，分块流式计算同一结果|完整 K/V|理论仍为 $O(L^2d_h)$，显著降低 IO|稠密 Attention 通用内核|
|滑动窗口|是，只读最近 $W$ 个位置|最近窗口 K/V|$O(LWd_h)$|Mistral 7B|
|局部/全局层交错|部分层稀疏、部分层稠密|局部层保存窗口，全局层保存全历史|由层比例共同决定|Gemma 3|
|固定块稀疏|是，按预设块图连接|被允许块的 K/V|取决于每个 Query Block 的邻接块数|BigBird|
|动态块稀疏|是，按内容选择 Top-K 块|通常仍保存可检索的原始 K/V|核心约 $O(LKd_h)$，另有索引成本|NSA、QSA|
|DSA Token 级动态稀疏|是，按内容选择 Top-$k$ Token|MLA 潜变量、位置分量和索引 Key|正式 Attention 为 $O(Lkd_m)$，轻量索引仍含 $O(L^2)$|DeepSeek-V3.2|
|CSA 压缩稀疏|是，先学习式压缩，再选择 Top-$k$ 条目|约 $L/m$ 个共享 K=V 条目、Indexer Key 和局部窗口|Indexer 约 $O(L^2/m)$，Core 约 $O(Lk)$|DeepSeek-V4|
|HCA 重压缩稠密|压缩后对所有可见摘要连接|约 $L/m'$ 个共享 K=V 条目和局部窗口|约 $O(L^2/m')$|DeepSeek-V4|
|线性注意力|不是从原矩阵删边，而是改写算子|固定大小递归状态|常见形式对 $L$ 线性|DeltaNet、Gated DeltaNet|

PagedAttention 也不属于 Sparse Attention：它改变 KV Cache 的分页存储、复用与调度方式，不改变 Query 数学上可以读取哪些历史 Token。判断一个方法属于哪条路线，可以依次问：它是否仍计算相同的 Softmax 结果、是否真的删除 Token 连接、是否保存原始 K/V，以及加速来自 FLOPs、显存容量还是 HBM 访存。

### 4.8 线性注意力、DeltaNet 与 Gated DeltaNet

FlashAttention 优化的是标准稠密 Attention 的计算与内存访问，仍然得到 Softmax Attention 的结果；线性注意力则直接改变相似度函数和历史信息的表示方式。它不再保留完整的历史 K/V 并计算 $L\times L$ 分数矩阵，而是把历史压缩到固定大小的递归状态中。因此两者不是同一种优化：前者保持数学算子，后者用不同算子换取关于序列长度的线性复杂度。

#### 4.8.1 从 Softmax Attention 到线性注意力

对位置 $t$ 的因果 Attention，可以先写成一般的核加权形式：

$$
o_t
=
\frac{
\sum_{j=1}^{t}\operatorname{sim}(q_t,k_j)v_j
}{
\sum_{j=1}^{t}\operatorname{sim}(q_t,k_j)
}
$$

标准 Softmax Attention 使用：

$$
\operatorname{sim}(q,k)
=
\exp\left(\frac{q^\top k}{\sqrt{d_h}}\right)
$$

这个指数点积通常不能用有限维特征精确分解。线性注意力改用或近似为可分解的核：

$$
\operatorname{sim}(q,k)
=
\phi(q)^\top\phi(k)
$$

其中 $\phi:\mathbb R^{d_k}\rightarrow\mathbb R^r$ 是非负或经过专门设计的特征映射，例如早期 Linear Transformer 使用 $\phi(x)=\operatorname{ELU}(x)+1$；其他方法也可用随机特征近似 Softmax 核。令：

$$
\bar q_t=\phi(q_t),\qquad
\bar k_t=\phi(k_t)
$$

利用矩阵乘法结合律，可维护两个前缀状态：

$$
S_t
=S_{t-1}+v_t\bar k_t^\top,
\qquad
z_t=z_{t-1}+\bar k_t
$$

其中：

$$
S_t\in\mathbb R^{d_v\times r},qquad
z_t\in\mathbb R^r
$$

当前输出直接从状态读取：

$$
o_t
=
\frac{S_t\bar q_t}
{z_t^\top\bar q_t+\varepsilon}
$$

展开分子即可验证：

$$
S_t\bar q_t
=
\sum_{j=1}^{t}
v_j(\bar k_j^\top\bar q_t)
$$

也就是说，$S_t$ 把所有历史 Key-Value 外积压缩到一个矩阵中，$z_t$ 保存归一化所需的 Key 前缀和。模型无需先构造所有 $q_t^\top k_j$，每读入一个 Token 就更新一次状态。

若特征维度为 $r$，单头的递归计算量约为：

$$
O(Lrd_v)
$$

推理状态大小约为 $O(rd_v+r)$，不随历史长度 $L$ 增长；当 $r\approx d_v\approx d_h$ 时，常简写成总计算 $O(Ld_h^2)$、每个新 Token 为 $O(d_h^2)$。相比之下，标准稠密 Attention 为 $O(L^2d_h)$，Decode 时还需要随 $L$ 增长的 KV Cache。

这里的“线性”是指复杂度对序列长度 $L$ 近似线性，不是说整个网络只有线性变换。它也通常不是标准 Softmax Attention 的精确快速实现：有限维 $\phi$ 会改变或近似相似度核，所以输出和标准 Attention 不必相同。

#### 4.8.2 固定状态为什么会发生记忆冲突

许多现代线性注意力省略显式分母 $z_t$，并对 Q/K、状态输出做其他归一化。忽略这些外围细节，其核心可简化为：

$$
S_t=S_{t-1}+v_tk_t^\top,
\qquad
o_t=S_tq_t
$$

此时 $S_t$ 可理解为一张随 Token 快速变化的 Key 到 Value 映射，即 Fast Weight Memory；模型的常规参数负责生成 $q_t,k_t,v_t$，而状态 $S_t$ 在前向传播中临时更新，并不是在推理时用优化器修改模型参数。

纯加法写入有一个明显问题：相同或相似的 Key 再次出现时，新旧 Value 会叠加，而不是覆盖旧关联。若 Key 彼此正交，映射较容易分离；当序列远长于 Key 特征维度时，许多关联只能叠加在固定大小的 $S_t$ 中，碰撞和干扰难以避免。线性状态因此可以持续处理任意长输入，却不等于能无损记住任意长历史。

#### 4.8.3 DeltaNet：按当前 Key 定向擦除并重写

DeltaNet 不再把 $v_tk_t^\top$ 直接加到状态，而是先读取当前 Key 已关联的旧 Value：

$$
\hat v_t=S_{t-1}k_t
$$

再只沿 $k_t$ 对应的方向写入预测误差：

$$
S_t
=S_{t-1}
-\beta_t(\hat v_t-v_t)k_t^\top
$$

等价地：

$$
S_t
=S_{t-1}(I-\beta_tk_tk_t^\top)
+\beta_tv_tk_t^\top
$$

$\beta_t$ 是由模型产生的写入强度，通常限制在 $(0,1)$。该公式可由在线回归目标直接推导：

$$
\mathcal J_t(S)
=\frac12\|Sk_t-v_t\|_2^2
$$

对 $S$ 做一次学习率为 $\beta_t$ 的梯度下降：

$$
S_t
=S_{t-1}
-\beta_t\nabla_S\mathcal J_t(S_{t-1})
$$

便得到上面的 Delta Rule。这里的“梯度下降”只是当前序列内部更新临时状态的数学解释，与训练阶段由 AdamW 更新长期模型参数是两层不同的过程。

一个二维例子能直观看出加法写入和 Delta Rule 的差别。假设旧状态为：

$$
S_{t-1}
=
\begin{bmatrix}
2&0\\
0&3
\end{bmatrix}
$$

它把单位 Key $e_1=[1,0]^\top$ 映射到 $[2,0]^\top$，把 $e_2=[0,1]^\top$ 映射到 $[0,3]^\top$。现在希望把 $e_1$ 的 Value 改为：

$$
k_t=e_1,qquad
v_t=
\begin{bmatrix}
5\\1
\end{bmatrix},qquad
\beta_t=1
$$

纯加法线性注意力得到：

$$
S_t^{\text{add}}
=S_{t-1}+v_tk_t^\top
=
\begin{bmatrix}
7&0\\
1&3
\end{bmatrix}
$$

再次读取 $e_1$ 会得到 $[7,1]^\top$，因为新旧 Value 被累加。DeltaNet 先计算旧值 $\hat v_t=[2,0]^\top$，只写入误差 $v_t-\hat v_t=[3,1]^\top$：

$$
S_t^{\Delta}
=S_{t-1}+(v_t-\hat v_t)k_t^\top
=
\begin{bmatrix}
5&0\\
1&3
\end{bmatrix}
$$

于是：

$$
S_t^{\Delta}e_1=
\begin{bmatrix}5\\1\end{bmatrix},qquad
S_t^{\Delta}e_2=
\begin{bmatrix}0\\3\end{bmatrix}
$$

旧的 $e_1$ 关联被准确替换，而与它正交的 $e_2$ 关联保持不变。这个“精确覆盖”依赖 $\|k_t\|_2=1$、$\beta_t=1$；真实 Key 不完全正交时仍会互相干扰，但 Delta Rule 比无条件累加更善于修改已有映射。

DeltaNet 的递归状态仍为固定大小，推理复杂度仍对 $L$ 线性。难点在训练：$S_t$ 依赖 $S_{t-1}k_t$，不能像纯加法前缀和那样直接并行。现代实现把序列分块，并利用广义 Householder 矩阵乘积的紧凑表示，把块内运算改写为适合 Tensor Core 的矩阵乘法。

#### 4.8.4 Gated DeltaNet：全局遗忘与定向改写结合

只有 Delta Rule 时，模型能沿某个 Key 方向替换内容，却难以在话题切换或文档边界处迅速清空大量无关状态。另一种简单方案是加入全局衰减：

$$
S_t=\alpha_tS_{t-1}+v_tk_t^\top,qquad
\alpha_t\in(0,1)
$$

它能快速遗忘，但每次会把所有旧关联按同一比例缩小，无法只删除某个 Key 对应的内容。通常口语中写成“Gate DeltaNet”的机制，论文正式名称是 **Gated DeltaNet**；其核心 Gated Delta Rule 把两种机制结合：

$$
S_t
=S_{t-1}
\left[
\alpha_t(I-\beta_tk_tk_t^\top)
\right]
+\beta_tv_tk_t^\top
$$

由于 $\alpha_t$ 是标量，也可改写为更直观的形式：

$$
S_t
=\alpha_tS_{t-1}
+\beta_t
\left(v_t-\alpha_tS_{t-1}k_t\right)k_t^\top
$$

可以把一次更新理解为两步：

1. **全局遗忘**：用数据依赖的 $\alpha_t$ 衰减整个旧状态；
2. **定向改写**：用 $\beta_t$ 沿当前 $k_t$ 方向修正关联并写入 $v_t$。

几个边界情况揭示了它的行为：

- $\alpha_t=1$ 时退化为 DeltaNet，只做定向更新；
- $\alpha_t\rightarrow 0$ 时几乎清空旧状态，再写入当前关联；
- $\beta_t=0$ 时不写入当前关联，只对旧状态做全局衰减；
- 若 $\|k_t\|_2=1$ 且 $\beta_t=1$，则 $S_tk_t=v_t$；对任意满足 $k_t^\top u=0$ 的方向 $u$，有 $S_tu=\alpha_tS_{t-1}u$。

因此，$\beta_t$ 解决“当前 Key 应改写多少”，$\alpha_t$ 解决“其余历史整体应保留多少”。两者均由当前输入动态生成。完整 Gated DeltaNet Block 通常还包含 Q/K/V 投影、短卷积、SiLU、Q/K 的 L2 归一化、输出归一化与输出门；这些部件改善局部模式建模和训练稳定性，但固定状态的核心来自上述 Gated Delta Rule。输出门调制当前读出，不能与控制历史状态衰减的 $\alpha_t$ 混为一谈。

#### 4.8.5 训练、推理与能力边界

|机制|历史如何保存|状态更新|主要优势|主要限制|
|---|---|---|---|---|
|标准 Softmax Attention|保存各位置 K/V|历史通常不压缩|可按内容直接访问单独 Token，精确检索能力强|稠密计算对 $L$ 为二次，KV Cache 随 $L$ 增长|
|纯加法线性注意力|固定矩阵状态|$S\leftarrow S+vk^\top$|对 $L$ 线性，递归推理状态固定|难以删除或覆盖，同类 Key 容易累加冲突|
|DeltaNet|固定矩阵状态|按 $v-Sk$ 定向修正|可选择性替换当前 Key 的旧关联|缺少快速清空全部旧内容的机制|
|Gated DeltaNet|固定矩阵状态|全局衰减 + Delta Rule|兼具快速遗忘和定向改写|仍受固定状态容量与关联碰撞限制|

递归形式最适合自回归推理，但逐 Token 依赖不利于训练并行；完全展开成序列矩阵又可能重新产生二次计算。实际内核通常采用 Chunkwise Parallel：块间传递固定状态，块内用并行矩阵乘法。若块大小 $C$ 视为常数，纯线性注意力的典型计算约为 $O(LCd_h+Ld_h^2)$，从而保持对 $L$ 的线性关系并提高 GPU 利用率；DeltaNet 与 Gated DeltaNet 还需要专门处理状态转移矩阵的连乘。

线性复杂度也不保证在所有长度上都比 FlashAttention 快，因为 $O(Ld_h^2)$ 的通道维成本、递归依赖、内核质量和硬件利用率都很重要。固定状态尤其不擅长从极长历史中无损找回任意细节，因此实践中常把 Gated DeltaNet/DeltaNet 层与滑动窗口或全局 Softmax Attention 层混合：线性层负责低成本传播和压缩大部分上下文，少量标准 Attention 层提供更精确的 Token 级访问。

### 4.9 Cross-Attention：Query 与 Key/Value 来自不同序列

Encoder-Decoder Transformer 的 Decoder 通常包含两种 Attention：

1. 因果 Self-Attention：目标序列内部只能读取已经出现的目标 Token；
2. Cross-Attention：目标表示作为 Query，读取 Encoder 产生的源序列表示。

设：

$$
H_{\text{dec}}\in\mathbb{R}^{B\times L_t\times d},
\qquad
H_{\text{enc}}\in\mathbb{R}^{B\times L_s\times d}
$$

则：

$$
Q=H_{\text{dec}}W_Q
$$

$$
K=H_{\text{enc}}W_K,\qquad
V=H_{\text{enc}}W_V
$$

Cross-Attention 为：

$$
\operatorname{CrossAttn}
=
\operatorname{softmax}
\left(
\frac{QK^\top}{\sqrt{d_h}}+M_{\text{src}}
\right)V
$$

其分数矩阵形状为：

$$
B\times h\times L_t\times L_s
$$

所以 Query 长度 $L_t$ 与 Key/Value 长度 $L_s$ 不需要相等；投影后的每头通道维 $d_h$ 必须匹配。$M_{\text{src}}$ 通常屏蔽源序列 Padding，而不是对源序列施加目标侧三角因果 Mask。

Cross-Attention 不会让目标 Token 绕过因果约束读取未来目标：未来目标根本没有进入 Encoder K/V；目标侧未来信息仍由 Decoder Self-Attention Mask 阻断。

推理时，Encoder 输出以及由它投影得到的 Cross-Attention K/V 可以只计算一次并在所有解码步复用；Decoder Self-Attention 的 KV Cache 则随目标序列逐步增长。这两类缓存来源和生命周期不同，不能混成同一个“KV Cache”概念。

---

## 五、FFN、门控与 MoE

### 5.1 FFN 为什么逐 Token 仍然重要

Attention 完成位置之间的信息混合，FFN 对每个位置独立执行相同变换：

$$
\operatorname{FFN}(x)
=W_2\,\phi(W_1x+b_1)+b_2
$$

典型形状是：

$$
d\rightarrow d_{\text{ff}}\rightarrow d
$$

其中 $d_{\text{ff}}>d$。第一层把表示投影到更宽空间，激活函数选择和组合特征，第二层再投回残差维度。

可把 FFN 直观理解为逐位置的特征存储和非线性计算器；Attention 决定“从哪里取信息”，FFN 决定“怎样加工已取得的信息”。

### 5.2 激活函数与 GLU/SwiGLU

如果两层线性变换之间没有激活函数，那么忽略 Bias 时：

$$
W_2(W_1x)=(W_2W_1)x
$$

两层仍等价于一层线性变换，无法表达更复杂的非线性关系。ReLU、GELU 和 SiLU 都是逐元素激活；GLU 则是带两条投影分支的门控结构，严格来说不只是一个单输入标量函数。

#### 5.2.1 ReLU

$$
\operatorname{ReLU}(x)=\max(0,x)
$$

ReLU 把负数直接置零、正数保持不变。除 $x=0$ 外，其导数为：

$$
\operatorname{ReLU}'(x)=
\begin{cases}
0,&x<0\\
1,&x>0
\end{cases}
$$

它计算简单，并能产生精确的零激活；但负半轴梯度恒为零，神经元若长期落在该区域可能难以恢复，即所谓“死亡 ReLU”。$x=0$ 处不可导，实际实现会约定一个次梯度。早期 Transformer 的原始 FFN 使用 ReLU。

#### 5.2.2 GELU

$$
\operatorname{GELU}(x)=x\Phi(x)
$$

其中 $\Phi(x)$ 是标准正态分布的累积分布函数。它可理解为用一个随输入平滑变化的权重 $\Phi(x)\in(0,1)$ 调制输入：较大的正值几乎完整通过，较大的负值被强烈抑制，零附近的小负值不会像 ReLU 那样被立即截断。常用近似为：

$$
\operatorname{GELU}(x)
\approx
\frac{x}{2}
\left[
1+\tanh\left(
\sqrt{\frac{2}{\pi}}(x+0.044715x^3)
\right)
\right]
$$

GELU 处处平滑，在 BERT、GPT-2 等 Transformer 中被广泛使用；相对 ReLU，它保留了更柔和的负值响应，但计算也更复杂，工程实现通常使用近似或融合算子。

#### 5.2.3 SiLU

$$
\sigma(x)=\frac{1}{1+e^{-x}},\qquad
\operatorname{SiLU}(x)=x\sigma(x)
$$

SiLU 也称 Swish（当 Swish 的参数 $\beta=1$ 时）。它可以看成输入用自身产生的 Sigmoid 门进行调制：正值越大，通过比例越接近 1；负值越小，通过比例越接近 0。SiLU 平滑且在负半轴不恒为零，并会保留少量负输出；它在零点附近略微非单调。其导数为：

$$
\operatorname{SiLU}'(x)
=\sigma(x)+x\sigma(x)(1-\sigma(x))
$$

SiLU 可以单独作为普通 FFN 的激活，也常作为 SwiGLU 门控分支的激活。

#### 5.2.4 GLU 与门控变体

GLU（Gated Linear Unit）先从同一输入生成内容分支和门控分支：

$$
u=xW_{\text{up}}+b_{\text{up}},\qquad
g=xW_{\text{gate}}+b_{\text{gate}}
$$

原始 GLU 使用 Sigmoid 门：

$$
h=\sigma(g)\odot u,qquad
y=hW_{\text{down}}+b_{\text{down}}
$$

$u$ 提供候选内容，$\sigma(g)$ 为每个通道生成 $0$ 到 $1$ 之间的通过比例，$\odot$ 表示逐元素乘法。门不是人工开关，而是由模型参数根据当前 Token 的表示学习出来的。

把 Sigmoid 换成一般激活 $\phi$，可写成统一形式：

$$
h=\phi(g)\odot u
$$

由此得到常见门控变体：

|结构|门控分支 $\phi(g)$|核心特点|
|---|---|---|
|GLU|$\sigma(g)$|门值限制在 $(0,1)$，主要控制通过比例|
|ReGLU|$\operatorname{ReLU}(g)$|门可为精确零，形式简单但负区间无梯度|
|GEGLU|$\operatorname{GELU}(g)$|使用平滑的 GELU 门控|
|SwiGLU|$\operatorname{SiLU}(g)$|使用平滑、允许少量负值的 SiLU 门控，现代 LLM 中很常见|

因此 SwiGLU 的完整计算为：

$$
u=xW_{\text{up}},\qquad
g=xW_{\text{gate}},\qquad
h=\operatorname{SiLU}(g)\odot u,qquad
y=hW_{\text{down}}
$$

普通 ReLU/GELU/SiLU FFN 主要有 up、down 两组投影，而 GLU 系列增加了 gate 投影，共有三组主要投影。若保持相同的中间宽度，门控 FFN 的参数量和计算量会更大；忽略 Bias，为匹配普通 FFN 的参数预算，可令门控 FFN 的中间宽度约为普通 FFN 的 $2/3$，实际模型还会按硬件对齐要求取整。

|方案|类型|负输入行为|主要权衡|
|---|---|---|---|
|ReLU|逐元素激活|全部截为 0|最简单，可产生稀疏激活，但存在死亡区间|
|GELU|逐元素激活|平滑抑制，保留少量负值|响应柔和，计算比 ReLU 复杂|
|SiLU|逐元素、自门控激活|平滑抑制，保留少量负值|平滑且非单调，需要计算 Sigmoid|
|GLU 系列|双投影乘法门控|取决于门控激活|通道选择能力更强，但多一组投影|

### 5.3 从 Dense FFN 到 MoE

Dense FFN 对每个 Token 使用同一组参数。Mixture of Experts 准备多个 FFN 专家，并由 Router 为每个 Token 选择少量专家：

$$
r(x)=W_rx
$$

$$
\mathcal{T}(x)=\operatorname{TopK}(r(x))
$$

$$
y=\sum_{e\in\mathcal{T}(x)}g_e(x)E_e(x)
$$

这里的 $x$ 是当前 Token 的输入表示（在 Pre-Norm Block 中通常为子层输入的归一化结果）。Router 与被选专家都读取同一个 $x$：Router 决定分发和权重，专家以各自独立的 FFN 参数处理该 Token；最后只对 Top-$K$ 专家的输出加权求和。MoE 子层外仍有残差连接，因此它通常嵌入为 $u+\operatorname{MoE}(\operatorname{Norm}(u))$。含共享专家的实现会让共享专家额外处理 Token，再按其具体规则与路由专家结果合并。

Router 参数通过主任务损失训练。损失从 $y$ 同时反传到被选专家的 $E_e$ 和连续门控权重 $g_e$，进而更新 $W_r$；因此 Router 会学习“哪类 Token 送往哪个专家更能降低任务损失”。但 Top-$K$ 的集合选择是离散的，通常不能通过“专家是否入选”本身获得普通梯度，训练主要依赖已选专家的门控权重梯度，并配合探索或平衡机制避免过早塌缩到少数专家。

设 $f_e$ 是一个 batch 中实际分配给专家 $e$ 的 Token 比例，$P_e$ 是 Router 对该专家的平均概率。常见辅助负载均衡项可写为：

$$
L_{\text{balance}}
=\alpha E\sum_{e=1}^{E}f_eP_e
$$

总损失为 $L=L_{\text{task}}+L_{\text{balance}}$。它鼓励 Router 降低过载专家的概率、提高闲置专家的使用率；具体实现也可使用 importance/load loss、Router logits 正则、加噪声探索，或基于近期负载动态调整专家偏置的无辅助损失策略。每个专家还常有容量上限，超出的 Token 需丢弃、重路由或排队；容量限制控制拥塞，但不等于自动实现负载均衡。

MoE 的目标是增加总参数容量，而不让每个 Token 激活全部参数。它并非“免费扩容”：

- Router 可能把大量 Token 集中到少数专家；
- 分布式训练会产生 All-to-All 通信；
- 推理部署要同时考虑专家权重内存与每 Token 激活计算。

Router 的 Top-K、归一化和负载均衡形式由具体实现决定，不能一概写成“总是 Softmax 后选两个专家”。

---

## 六、残差连接与归一化

### 6.1 残差为何改善深层训练

残差结构：

$$
y=x+F(x)
$$

其雅可比矩阵为：

$$
\frac{\partial y}{\partial x}
=I+\frac{\partial F}{\partial x}
$$

反向传播因此存在一条乘以单位矩阵的直接路径。它不能保证梯度永不爆炸或消失，但显著降低了深层网络必须让全部梯度穿过复杂非线性变换的难度。

### 6.2 LayerNorm

对单个 Token 的 $d$ 个特征：

$$
\mu=\frac1d\sum_{i=1}^d x_i
$$

$$
\sigma^2=\frac1d\sum_{i=1}^d(x_i-\mu)^2
$$

$$
\hat x_i=\frac{x_i-\mu}{\sqrt{\sigma^2+\epsilon}}
$$

$$
y_i=\gamma_i\hat x_i+\beta_i
$$

LayerNorm 在特征维归一化，不依赖 batch 中其他样本，因此适合可变长度序列和自回归推理。

对单个 Token，设上游梯度为 $g_i=\partial L/\partial y_i$，则参数梯度贡献为：

$$
\frac{\partial L}{\partial\beta_i}=g_i
$$

$$
\frac{\partial L}{\partial\gamma_i}=g_i\hat x_i
$$

令：

$$
\widetilde g=g\odot\gamma,\qquad
r=\sqrt{\sigma^2+\epsilon}
$$

则精确的输入梯度为：

$$
\frac{\partial L}{\partial x_i}
=
\frac1r
\left[
\widetilde g_i-\operatorname{mean}(\widetilde g)
-\hat x_i\operatorname{mean}(\widetilde g\odot\hat x)
\right]
$$

共享的 $\gamma,\beta$ 会对 batch 和 Token 维的所有贡献求和。

### 6.3 RMSNorm

RMSNorm 不减均值：

$$
r=\sqrt{\frac1d\sum_i x_i^2+\epsilon}
$$

$$
y_i=\gamma_i\frac{x_i}{r}
$$

它保留均值方向，计算更简单。令 $\tilde g=g\odot\gamma$，输入梯度为：

$$
\frac{\partial L}{\partial x}
=\frac{\tilde g}{r}
-\frac{x}{d\,r^3}
\sum_i \tilde g_ix_i
$$

LayerNorm 和 RMSNorm 都稳定特征尺度，但不是“把梯度固定在某个范围”的硬约束。

### 6.4 Pre-Norm 与 Post-Norm

Post-Norm：

$$
y=\operatorname{Norm}(x+F(x))
$$

Pre-Norm：

$$
y=x+F(\operatorname{Norm}(x))
$$

Pre-Norm 的残差主干更接近恒等映射，通常更容易训练很深的网络。Post-Norm 在一些配置中可获得不同的表示性质，但往往更依赖初始化、学习率与训练技巧。

在第二个子层中：

$$
u=x+\operatorname{Attention}(\operatorname{Norm}(x))
$$

$$
y=u+\operatorname{FFN}(\operatorname{Norm}(u))
$$

FFN 残差里的“旧值”是 $u$，不是层最初的 $x$。这对画计算图和写反向传播都很重要。

### 6.5 Gated Residual：在多条残差流之间动态读写

Gated Residual 的核心作用是扩展并管理**网络深度方向的信息通道**：

1. **提高残差带宽**：用多条并行残差流保存不同类型或不同时间尺度的特征，缓解所有信息都挤在单一 $d$ 维主干中的竞争；
2. **动态选择子层输入**：逐通道读门根据当前 Token 决定 Attention、Gated DeltaNet 或 FFN 应从每条流读取哪些特征；
3. **控制信息写回路径**：逐分支写门决定本次子层输出应更新哪些残差流、更新多强，使某些分支可以少受中间层扰动并形成更直接的深层传播路径；
4. **保留稳定的恒等路径**：门控只调节读取和新增写入，每条旧残差流仍直接进入下一状态，有利于梯度传播和深层训练稳定性。

它不是用来检索历史 Token，也不是序列方向的记忆压缩器；它解决的是跨层信息怎样保存、读取和更新。序列历史仍由 QSA、Attention 或 Gated DeltaNet 处理。

标准 Transformer 只有一条宽度为 $d$ 的残差流：

$$
y=x+F(x)
$$

每个子层固定读取整条 $x$，输出也以系数 1 写回同一条流。Gated Residual 将残差状态扩展为 $N$ 条并行流：

$$
R
=
[R_1,\ldots,R_N],
\qquad
R_r\in\mathbb R^d
$$

子层不会分别执行 $N$ 次，而是先从多条流动态读出一个宽度仍为 $d$ 的输入，运行一次 Attention、Gated DeltaNet 或 FFN，再以不同强度把同一个子层输出写回各条流：

```text
R1 ─┐                         ┌─→ R1 + s1·y
R2 ─┼─→ 逐通道读门 → x → F → y ├─→ R2 + s2·y
R3 ─┼                         ├─→ R3 + s3·y
R4 ─┘                         └─→ R4 + s4·y
```

以 Qwen3.8-Flash-Next 为例，$N=4$。先分别对四条流做 RMSNorm：

$$
\widehat R_r=\operatorname{RMSNorm}(R_r)
$$

把它们拼接后，通过秩为 320 的低秩门控网络生成逐元素读门：

$$
Z
=
\operatorname{Concat}(\widehat R_1,\ldots,\widehat R_N)
\in\mathbb R^{Nd}
$$

$$
A
=
\sigma
\left(
W_{\text{up}}
\operatorname{SiLU}
\left(
\frac{W_{\text{down}}Z}{N}
\right)
\right),
\qquad
A\in(0,1)^{N\times d}
$$

子层输入是各分支归一化表示的门控平均：

$$
x
=
\frac1N
\sum_{r=1}^{N}
A_r\odot\widehat R_r
$$

$A_r$ 对每个通道都可以不同，因此模型能够从某条流读取语法特征、从另一条流读取远程信息，而不必把所有历史表示持续挤在一条残差主干中。低秩瓶颈把门控参数和计算控制在较低水平。

子层只对混合结果执行一次：

$$
y=F(x)
$$

写门由同一个多流状态产生，但每条流只使用一个标量：

$$
s
=
2\sigma
\left(
\frac{W_{\text{write}}Z}{N}
\right),
\qquad
s\in(0,2)^N
$$

最后逐流更新：

$$
R_r'
=
R_r+s_ry
$$

所以它具有不对称的读写粒度：

- **读门是逐元素的**：每条流的每个特征通道可以有不同权重；
- **写门是逐分支标量的**：同一子层输出以不同整体强度写入四条流；
- **恒等路径始终保留**：每条 $R_r$ 都直接出现在 $R_r'$ 中，仍具有普通残差连接的梯度直通路径。

系数 $2\sigma(\cdot)$ 以 1 为自然中心并限制在 $(0,2)$，既能减弱某条流的本次写入，也能把写入放大到普通残差的两倍以内。实现不再对残差分支做昂贵的全连接跨分支混合：表达能力主要来自数据依赖的读门、写门以及多层累积，从而降低内存访问和训练不稳定性。

在完整 Decoder 中，输入 Token Embedding 最初复制到四条残差流；每个 Attention/Gated DeltaNet 子层和每个 MoE/FFN 子层都有独立的 Gated Residual 读写过程。最后一层之后再使用只有读操作的 Gated Residual Mixer，把四条流合成为一个 $d$ 维表示，送入 Final Norm 和 LM Head。

N-gram PLE 也按当前四流状态计算门，并把局部短语 Value 注入四条残差流；随后 Attention 与 FFN 再通过各自的 Gated Residual 读写。因此两者的协作关系是：

```text
N-gram PLE：根据局部 Token 组合补充词法记忆
Gated Residual：决定各子层怎样从四条深度状态读取、又写回哪些流
```

Gated Residual 还不能与 Gated DeltaNet 混淆。前者管理的是**网络深度方向**的多条残差流，状态大小随层数固定；后者管理的是**序列时间方向**的递归记忆，用遗忘门和 Delta Rule 更新历史 Key-Value 关联。它们可以同时存在于同一个 Block 中，但控制的是两种不同的信息流。

---

## 七、输出层与语言模型损失

### 7.1 从隐藏状态到 Logits

最后一层输出经 Final Norm 后得到：

$$
H\in\mathbb{R}^{B\times L\times d}
$$

LM Head 投影到词表：

$$
Z=HW_{\text{lm}}+b
$$

$$
Z\in\mathbb{R}^{B\times L\times V}
$$

Logit 是未归一化分数，不是概率。对位置 $t$：

$$
p_{t,j}
=\frac{\exp(z_{t,j})}
{\sum_{k=1}^V\exp(z_{t,k})}
$$

### 7.2 Label Shift

输入：

```text
[BOS, 我, 爱, 苹果]
```

目标：

```text
[我, 爱, 苹果, EOS]
```

也就是用位置 $t$ 的输出预测 $x_{t+1}$。若有效目标位置集合为 $\mathcal V$，数量为 $M$，平均交叉熵为：

$$
L=-\frac1M\sum_{t\in\mathcal V}\log p_{t,y_t}
$$

Padding 或不参与监督的 Prompt Token 应通过 label/loss mask 排除。Attention Padding Mask 决定“能否读取”，label mask 决定“是否计入损失”，两者不能混为一个 Mask。

### 7.3 Softmax 与交叉熵的联合梯度

对没有 label smoothing 的单个有效位置：

$$
\frac{\partial L}{\partial z_j}
=p_j-\mathbf 1[j=y]
$$

若损失对 $M$ 个有效目标取平均：

$$
\frac{\partial L}{\partial z_{t,j}}
=\frac{p_{t,j}-\mathbf 1[j=y_t]}{M}
$$

这条公式解释了训练的起点：

- 正确 Token 的 Logit 被向上推动；
- 其他 Token 按当前概率被向下推动；
- 模型越自信地预测错误，梯度通常越大。

若使用 label smoothing，目标不再是严格 one-hot，梯度相应变为 $p-q$，其中 $q$ 是平滑后的目标分布。

工程实现常直接使用 `log_softmax`/`cross_entropy` 的融合内核，避免先显式计算概率再取对数造成数值不稳定。

### 7.4 MTP：让同一位置学习预测多个未来 Token

MTP（Multi-Token Prediction，多 Token 预测）把普通的“预测下一个 Token”扩展为“从同一个上下文预测多个不同距离的未来 Token”。给定上下文 $x_{\le t}$，普通下一 Token 预测只有：

$$
p_t^{(1)}=P(x_{t+1}\mid x_{\le t})
$$

若预测未来 $n$ 个 Token，第 $k$ 个预测头负责偏移量 $k$：

$$
p_t^{(k)}=P(x_{t+k}\mid x_{\le t}),\qquad k=1,2,\ldots,n
$$

每个偏移量都有自己的有效位置集合 $\mathcal V_k$。越靠近序列末尾，可用的远期标签越少，因此必须分别做边界和 Padding/Loss Mask：

$$
L_k
=-
\frac{1}{|\mathcal V_k|}
\sum_{t\in\mathcal V_k}
\log p_t^{(k)}[x_{t+k}]
$$

一种常见总损失写法是保留下一 Token 损失为主目标，把其余深度作为加权辅助目标：

$$
L_{\text{total}}
=L_1+
\sum_{k=2}^{n}\alpha_kL_k
$$

不同实现也可能对所有预测深度取平均。权重 $\alpha_k$ 控制远期监督对共享主干梯度的影响。

例如训练序列为：

```text
[BOS, 我, 爱, 吃, 红, 苹果]
```

在“爱”所在位置，普通训练只要求隐藏状态 $h_t$ 预测“吃”；三 Token MTP 还会要求它通过不同预测分支预测更远的“红”和“苹果”：

```text
上下文 [BOS, 我, 爱] → 共享主干隐藏状态 h_t
                              ├─ 第 1 头 → 目标 x_(t+1) = 吃
                              ├─ 第 2 头 → 目标 x_(t+2) = 红
                              └─ 第 3 头 → 目标 x_(t+3) = 苹果
```

这些标签在训练数据中本来就已知，所有位置和预测头仍可并行训练。MTP 增加的是“每个位置所承担的预测距离”，不是把自回归训练改成逐 Token 循环，也不会移除因果 Mask。

#### 7.4.1 并行预测头与顺序预测模块

最直接的 MTP 使用共享 Transformer 主干和 $n$ 个独立输出头。各头同时读取 $h_t$，分别估计不同偏移量的条件分布。若把它们写成未来片段的近似联合分布，相当于：

$$
P(x_{t+1:t+n}\mid x_{\le t})
\approx
\prod_{k=1}^{n}p_t^{(k)}(x_{t+k})
$$

这种设计简单且并行度高，但较远预测头没有显式条件于前面预测出的 Token。例如，第 3 头预测 $x_{t+3}$ 时并未真正读取第 1、2 头选择的 $x_{t+1},x_{t+2}$，所以多个头各自合理不代表组合出的片段一定连贯。

另一类实现用顺序 MTP 模块保留完整因果链。以 DeepSeek-V3 的通用形式为例，主模型表示 $h_t^{(0)}$ 先预测 $x_{t+1}$；第 $k$ 个附加模块把上一深度表示与 $x_{t+k}$ 的 Embedding 拼接并投影：

$$
\tilde h_t^{(k)}
=M_k
\left[
\operatorname{RMSNorm}(h_t^{(k-1)});
\operatorname{RMSNorm}(E(x_{t+k}))
\right]
$$

把各位置的 $\tilde h_t^{(k)}$ 组成序列后，再以因果 Mask 送入一个附加 Transformer Block（相对完整主干更轻量），产生新表示并预测下一个深度的 Token。简写为：

$$
h_t^{(k)}=\operatorname{TRM}_k(\tilde h_t^{(k)}),\qquad
q_t^{(k)}=\operatorname{softmax}(W_{\text{lm}}h_t^{(k)})
$$

这里用 $q_t^{(k)}$ 区分前面并行头的记号，其监督目标是 $x_{t+k+1}$。训练时使用真实的 $x_{t+k}$，属于 Teacher Forcing；第一个附加模块看见真实 $x_{t+1}$ 后预测 $x_{t+2}$，第二个再看见真实 $x_{t+2}$ 后预测 $x_{t+3}$：

```text
h_t^(0) ──主 LM Head────────────────────────→ 预测 x_(t+1)
   │
   └─与 E(x_(t+1)) 融合→ MTP 模块 1 → h_t^(1) → 预测 x_(t+2)
                                      │
                                      └─与 E(x_(t+2)) 融合
                                           → MTP 模块 2 → 预测 x_(t+3)
```

这不是未来信息泄漏：主干 $h_t^{(0)}$ 仍只能读取 $x_{\le t}$；附加模块虽然读取训练期真实的 $x_{t+k}$，但其标签是尚未读取的 $x_{t+k+1}$。共享主干会收到所有深度损失的梯度，因此被推动去产生对较远未来也有用的表示。DeepSeek-V3 的实际配置深度 $D=1$，即主模型执行常规下一 Token 预测，一个附加模块再预测第二个未来 Token；Embedding 和 LM Head 与主模型共享。

#### 7.4.2 MTP 为什么可能改善训练

- **监督更密集**：普通训练中每个位置只对应一个偏移量；MTP 让同一主干表示同时收到近、远多个目标的梯度，在固定语料上提供更多训练信号。
- **鼓励提前规划**：只依赖局部模式可能足以猜中下一个 Token，预测更远 Token 则要求表示保留对后续结构、语义和程序状态有用的信息。
- **可能提高数据效率**：同一段语料被用于学习多个预测跨度，尤其可能帮助需要维持长程一致性的生成任务。

这些是训练目标带来的归纳偏置，不保证所有模型和任务都会同等受益。远期目标的不确定性更大，过多预测深度或过大的辅助损失权重可能使优化变难；额外预测头、Transformer 模块和词表 Logits 也会增加训练计算与显存。共享 Embedding/LM Head、逐个计算并释放各头 Logits、激活重计算等技术可以降低额外内存。

#### 7.4.3 MTP 如何用于推测式解码

MTP 模块在普通推理时可以直接丢弃，主模型仍按标准方式一次生成一个 Token。若要加速，可把附加头或模块当作轻量 Draft，形成自推测解码：

1. 主头产生第一个 Token，MTP 分支继续草拟后续若干 Token；
2. 把整段草稿送给主模型，借助因果 Mask 在一次批量前向中验证各位置；
3. 接受通过验证的最长前缀；遇到第一个不接受的 Token 时，由主模型结果纠正并重新起草。

Greedy 解码可按草稿是否与主模型选择一致来验收；随机采样需要使用带接受/拒绝与概率校正的推测采样算法，才能保持主模型的目标分布。不能未经验证就把所有 MTP 输出直接追加到序列，因为并行头的远期输出尤其可能与实际已选前缀不一致。

加速来自“用较便宜的 Draft 加一次并行验证，替代多次昂贵的主模型串行 Decode”。实际收益取决于草稿接受率、一次起草长度、验证成本和 batch 规模；MTP 训练本身不自动保证推理更快。

---

## 八、反向传播：梯度怎样穿过整层

### 8.1 两条矩阵求导规则

对线性变换：

$$
Y=XW+b
$$

若上游梯度为 $G_Y=\partial L/\partial Y$，则：

$$
\frac{\partial L}{\partial W}=X^\top G_Y
$$

$$
\frac{\partial L}{\partial X}=G_YW^\top
$$

$$
\frac{\partial L}{\partial b}
=\sum_{\text{batch/token}}G_Y
$$

判断形状是排查反向传播错误的最快方法：

$$
X:(N,d_{\text{in}}),\quad
W:(d_{\text{in}},d_{\text{out}}),\quad
G_Y:(N,d_{\text{out}})
$$

因此 $X^\top G_Y$ 恰好与 $W$ 同形。

另一条规则是分支梯度相加。如果同一张量被两条路径使用：

$$
L=L_1(x)+L_2(x)
$$

那么：

$$
\frac{\partial L}{\partial x}
=\frac{\partial L_1}{\partial x}
+\frac{\partial L_2}{\partial x}
$$

残差、Q/K/V 三分支和 weight tying 都依赖这条规则。

### 8.2 从 LM Head 回到隐藏状态

把 batch 和序列位置合并为 $N=B\cdot L$：

$$
Z=HW_{\text{lm}}
$$

$$
G_H^{(\text{out})}=G_ZW_{\text{lm}}^\top
$$

$$
G_{W_{\text{lm}}}=H^\top G_Z
$$

若 LM Head 与 Embedding 共享权重，Embedding 参数的总梯度是：

1. 输出分类路径产生的稠密梯度；
2. 输入 lookup 路径按 Token ID 累积的梯度。

### 8.3 穿过残差

对：

$$
Y=X+F(X)
$$

上游梯度 $G_Y$ 分成：

$$
G_X^{(\text{skip})}=G_Y
$$

以及经过 $F$ 的：

$$
G_X^{(F)}
=G_Y\frac{\partial F}{\partial X}
$$

最终：

$$
G_X=G_X^{(\text{skip})}+G_X^{(F)}
$$

这也是为什么画 Transformer 反向图时不能只沿模块顺序画一条线。

### 8.4 标准 Softmax Attention 的完整梯度

先看单头：

$$
S=\frac{QK^\top}{\sqrt{d_h}}+M,\qquad
A=\operatorname{softmax}(S),\qquad
O=AV
$$

设上游梯度为 $G_O$。

经过 $O=AV$：

$$
G_A=G_OV^\top
$$

$$
G_V=A^\top G_O
$$

对 Softmax 的每一行，令：

$$
c_i=\sum_j G_{A,ij}A_{ij}
$$

则：

$$
G_{S,ij}=A_{ij}(G_{A,ij}-c_i)
$$

被 Mask 为不可见的位置应保持零梯度。实现中不能让有限精度下的“大负数”产生意外的非零概率，也要正确处理整行都被 Mask 的异常输入。

经过缩放点积：

$$
G_Q=\frac{G_SK}{\sqrt{d_h}}
$$

$$
G_K=\frac{G_S^\top Q}{\sqrt{d_h}}
$$

再经过 Q/K/V 投影。若：

$$
Q=HW_Q,\quad K=HW_K,\quad V=HW_V
$$

则：

$$
G_{W_Q}=H^\top G_Q,\quad
G_{W_K}=H^\top G_K,\quad
G_{W_V}=H^\top G_V
$$

$$
G_H
=G_QW_Q^\top+G_KW_K^\top+G_VW_V^\top
$$

多头 Attention 还要先穿过 $W_O$，再把拼接梯度拆回各头。GQA/MQA 中，多个 Query 头共享同一 KV 头，因此共享 K/V 分支的梯度需要对使用它的 Query 组求和。

如果 Q/K 经过 RoPE，应先用旋转矩阵的转置把梯度转回未旋转空间，然后再计算 $W_Q,W_K$ 的梯度。

以上推导只适用于标准 Softmax Attention。线性注意力、DeltaNet 和 Gated DeltaNet 没有完整的 $A=\operatorname{softmax}(S)$，反向传播要沿 $S_t$ 的状态递推展开：$S_t$ 的梯度既来自当前输出 $o_t$，也来自后续状态 $S_{t+1}$；$\alpha_t$、$\beta_t$ 还会收到状态保留与写入路径的梯度。工程实现通常以分块扫描和状态重计算避免保存每一步的完整状态，不能直接套用本节的 Softmax Jacobian。

### 8.5 FFN 与 SwiGLU 的梯度

教学版 FFN：

$$
u=xW_1+b_1,\quad h=\phi(u),\quad y=hW_2+b_2
$$

反向：

$$
G_{W_2}=h^\top G_y,\quad
G_h=G_yW_2^\top
$$

$$
G_u=G_h\odot\phi'(u)
$$

$$
G_{W_1}=x^\top G_u,\quad
G_x=G_uW_1^\top
$$

SwiGLU 有两条输入分支：

$$
u=xW_{\text{up}},\quad
g=xW_{\text{gate}},\quad
h=\operatorname{SiLU}(g)\odot u
$$

设 $G_h$ 已知：

$$
G_u=G_h\odot\operatorname{SiLU}(g)
$$

$$
G_g=G_h\odot u\odot\operatorname{SiLU}'(g)
$$

$$
G_x
=G_uW_{\text{up}}^\top
+G_gW_{\text{gate}}^\top
$$

输入梯度必须把门控和内容两条路径相加。

### 8.6 一层梯度流全景

```text
Cross-Entropy
  ↓
Logits
  ↓
LM Head / tied Embedding
  ↓
Final Norm
  ↓
第 N 层残差分叉
  ├─ FFN → down / gate / up
  └─ skip
  ↓ 相加
Norm
  ↓
Attention 残差分叉
  ├─ W_O
  │   → O = AV
  │   → Softmax
  │   → QKᵀ / √d_h
  │   → RoPE（若有）
  │   → W_Q / W_K / W_V
  └─ skip
  ↓ 相加
第 N-1 层
  ↓
Embedding / Position 参数
```

反向传播负责计算梯度；优化器负责根据梯度更新参数。二者不是同一个步骤。

---

## 九、优化器与完整训练循环

### 9.1 SGD：从梯度得到最基本的参数更新

设当前参数为 $\theta_t$，损失对参数的梯度为：

$$
g_t=\nabla_{\theta}L(\theta_t)
$$

最基本的随机梯度下降（Stochastic Gradient Descent，SGD）更新为：

$$
\theta_{t+1}=\theta_t-\eta g_t
$$

其中 $\eta$ 是学习率。梯度给出损失上升最快的方向，因此沿负梯度方向移动。名称中的“随机”来自每次通常只用一个 mini-batch 估计完整训练集梯度；该估计含噪声，但计算成本远低于每步遍历全部数据。

SGD 对所有参数坐标使用同一个学习率。为了减少 mini-batch 梯度抖动，可以对历史梯度做动量累计：

$$
u_t=\mu u_{t-1}+g_t,\qquad
\theta_{t+1}=\theta_t-\eta u_t
$$

动量保留方向持续一致的更新、抵消频繁改变方向的噪声。Adam 的一阶矩可以看作与这一思想相关的指数滑动平均。

### 9.2 Weight Decay：在更新时主动收缩权重

Weight decay 的目标是让权重在每一步都略微向零收缩。将它直接施加到 SGD 更新，可写为：

$$
\theta_{t+1}
=(1-\eta\lambda)\theta_t-\eta g_t
$$

其中 $\lambda$ 是衰减系数。它由两部分组成：

$$
\underbrace{-\eta g_t}_{\text{任务损失推动参数学习}}
\qquad+
\underbrace{-\eta\lambda\theta_t}_{\text{按当前权重大小进行收缩}}
$$

另一种常见写法是在损失中加入 L2 正则：

$$
L_{\text{total}}
=L_{\text{task}}+\frac{\lambda}{2}\lVert\theta\rVert_2^2
$$

其梯度为：

$$
\nabla_\theta L_{\text{total}}
=g_t+\lambda\theta_t
$$

代入普通 SGD 后：

$$
\theta_{t+1}
=\theta_t-\eta(g_t+\lambda\theta_t)
=(1-\eta\lambda)\theta_t-\eta g_t
$$

因此，在普通 SGD 中，“把 L2 项加入损失”和“直接做 weight decay”更新形式等价。这个等价关系到了自适应优化器中会被破坏。

### 9.3 Adam：为不同参数坐标自适应调整步长

SGD 直接使用当前梯度；Adam 同时维护梯度的一阶矩和平方梯度的二阶矩：

$$
m_t=\beta_1m_{t-1}+(1-\beta_1)g_t
$$

$$
v_t=\beta_2v_{t-1}+(1-\beta_2)g_t^2
$$

这里的平方和运算都按参数坐标逐元素执行。$m_t$ 平滑梯度方向，$v_t$ 估计各坐标近期的梯度尺度。由于 $m_0=v_0=0$，训练早期估计偏向零，需要做偏差修正：

$$
\hat m_t=\frac{m_t}{1-\beta_1^t},\qquad
\hat v_t=\frac{v_t}{1-\beta_2^t}
$$

Adam 更新为：

$$
\theta_{t+1}
=\theta_t
-\eta\frac{\hat m_t}{\sqrt{\hat v_t}+\epsilon}
$$

梯度长期较大的坐标会被较大的 $\sqrt{\hat v_t}$ 缩小，梯度较小的坐标则得到相对更大的有效步长。因此 Adam 不再像 SGD 那样对所有坐标采用同一缩放。

如果直接把 L2 梯度混入 Adam：

$$
g_t^{\prime}=g_t+\lambda\theta_t
$$

那么 $\lambda\theta_t$ 也会进入 $m_t$、$v_t$，并被每个坐标不同的 $1/(\sqrt{\hat v_t}+\epsilon)$ 自适应缩放。此时它不再等价于独立地按固定比例收缩参数。

### 9.4 AdamW：把自适应梯度与权重衰减解耦

AdamW 让 Adam 只处理任务梯度，再把 weight decay 作为独立更新施加：

$$
\theta_{t+1}
=
\theta_t
-\eta\frac{\hat m_t}{\sqrt{\hat v_t}+\epsilon}
-\eta\lambda\theta_t
$$

等价地看：

$$
\theta_{t+1}
=(1-\eta\lambda)\theta_t
-\eta\frac{\hat m_t}{\sqrt{\hat v_t}+\epsilon}
$$

其两条通路彼此分开：

```text
任务梯度 g_t → Adam 一阶/二阶矩 → 自适应更新

当前参数 θ_t → 按 ηλ 独立收缩 → Weight Decay
```

因此逻辑顺序是：SGD 给出基础梯度更新；weight decay 引入参数收缩；Adam 引入逐坐标自适应缩放；AdamW 再把参数收缩从 Adam 的梯度统计中分离。Bias、Norm scale 等一维参数是否衰减是训练配置选择，常见做法是将它们放入不衰减参数组。

### 9.5 一次参数更新的工程顺序

典型混合精度、梯度累积和 DDP 训练循环：

```python
optimizer.zero_grad(set_to_none=True)

# 这里假设 micro_batches 已准备好，且 label_mask 中 1 表示有效目标。
local_valid_tokens = sum(batch.label_mask.sum() for batch in micro_batches)
global_valid_tokens_in_update = local_valid_tokens.detach().clone()
dist.all_reduce(global_valid_tokens_in_update, op=dist.ReduceOp.SUM)

for micro_step, batch in enumerate(micro_batches):
    should_sync = micro_step == len(micro_batches) - 1

    # 非最后一个 micro-batch 可使用 DDP no_sync，避免每次都通信。
    sync_context = nullcontext() if should_sync else model.no_sync()

    with sync_context:
        with autocast():
            logits = model(batch.input_ids, attention_mask=batch.attention_mask)
            loss_sum = token_cross_entropy_sum(
                logits,
                batch.labels,
                label_mask=batch.label_mask,
            )
            # DDP 默认对各 rank 的梯度取平均。若按全局有效 Token
            # 数归一化，应补偿 world_size；单卡时 world_size=1。
            loss = (
                loss_sum
                * ddp_world_size
                / global_valid_tokens_in_update
            )

        scaler.scale(loss).backward()

# 最后一次 backward 后，DDP 梯度已完成同步。
scaler.unscale_(optimizer)
clip_grad_norm_(model.parameters(), max_norm)
scaler.step(optimizer)
scaler.update()

# 概念判断：若 AMP 没有因 overflow 跳过 optimizer step，
# 才推进按 update 计数的 scheduler。
if optimizer_update_was_applied:
    scheduler.step()

optimizer.zero_grad(set_to_none=True)
```

重要边界：

若各 micro-batch 或各 rank 的有效 Token 数不同，简单平均“每批平均 Loss”会给 Token 较少的批次过高权重。设默认 DDP 的 world size 为 $W$，rank $r$ 在本次更新上的有效 Token 损失和为 $L_r^\Sigma$，全局有效 Token 数为 $G$。每个 rank 应反向传播：

$$
\widetilde L_r=\frac{W}{G}L_r^\Sigma
$$

DDP 再对 rank 梯度取平均，得到：

$$
\frac1W\sum_r\nabla\widetilde L_r
=
\frac1G\sum_r\nabla L_r^\Sigma
$$

这才等价于对全局有效 Token 求平均。这里假设使用默认 DDP 平均语义；若注册了自定义通信 Hook、使用其他并行损失归约，缩放因子必须按其语义重新推导。若 $G=0$，应跳过该次更新，不能除以零。

此外还要注意：

- 使用 DDP `no_sync()` 时，必须让最后一个 micro-batch 触发同步。
- 梯度裁剪应在 AMP `unscale_` 之后，否则裁剪的是放大后的梯度。
- 学习率调度通常按实际 optimizer update 计步，而不是每个 micro-batch 计步；若 AMP 因溢出跳过更新，也不应把它误算为一次成功更新。

### 9.6 混合精度、检查点与并行

- **混合精度**：FP16/BF16 降低显存和计算成本；归一化、Softmax、Loss reduction 等敏感操作常保留更高精度累加。
- **Gradient Scaling**：主要用于 FP16，避免很小的梯度下溢；BF16 指数范围更大，通常不需要相同形式的动态缩放。
- **Gradient Checkpointing**：前向不保存部分激活，反向时重新计算，以更多计算换更少显存。
- **数据并行**：不同设备处理不同样本，并对参数梯度求和/平均。
- **张量并行**：把大矩阵的行或列拆到多个设备。
- **流水线并行**：把不同层放到不同设备。
- **专家并行**：把 MoE 专家分散到设备，并为 Token 路由产生通信。

这些策略改变执行方式，不应改变理想数学结果；实际仍会受到浮点归约顺序、随机数和异步执行影响。

### 9.7 常见训练故障如何定位

|现象|优先检查|
|---|---|
|Loss 不下降|Label Shift、loss mask、学习率、Tokenizer 与权重是否匹配|
|出现 NaN/Inf|Softmax Mask、AMP overflow、Norm epsilon、过大学习率|
|多卡结果偏差很大|有效 Token 归一化、DDP 同步时机、随机种子与数据切分|
|长文本突然退化|位置缩放配置、训练长度分布、KV Cache/Mask 边界|
|MoE 专家拥塞|Router 分布、容量因子、负载均衡和通信瓶颈|
|显存超预期|Attention 激活、KV Cache、优化器状态、未启用 checkpoint|

梯度检查可对小模型、短序列使用有限差分：

$$
\frac{\partial L}{\partial\theta_i}
\approx
\frac{L(\theta_i+\varepsilon)-L(\theta_i-\varepsilon)}
{2\varepsilon}
$$

它适合验证自定义算子，不适合直接检查完整大模型。

---

## 十、参数量、训练与推理

### 10.1 一层参数量的主要来源

对标准 MHA，Q/K/V/O 投影合计约：

$$
4d^2
$$

两层普通 FFN 约：

$$
2dd_{\text{ff}}
$$

SwiGLU 约：

$$
3dd_{\text{ff}}
$$

Embedding/LM Head 若共享权重约为：

$$
Vd
$$

GQA 会减少 K/V 投影参数，但更显著的收益通常是降低推理 KV Cache。MoE 增加总专家参数，但每个 Token 只激活 Top-K 专家。

### 10.2 大模型量化：用数值精度换取存储与吞吐

量化（Quantization）的核心是：用更少的 bit 近似表示权重、激活或 KV Cache，在允许一定数值误差的前提下降低存储、内存带宽和计算成本。它不会改变参数的逻辑数量。例如，一个 7B 模型量化后仍有约 70 亿个参数，只是每个参数可能从 FP16 的 16 bit 变为 INT4 的 4 bit。

量化与剪枝、蒸馏也不同：剪枝删除或置零部分参数，蒸馏训练一个新的较小模型，量化主要改变数值表示格式及执行内核。

#### 10.2.1 均匀整数量化的基本公式

设浮点数为 $x$，用 $b$ bit 整数 $q$ 近似保存。最常见的仿射量化为：

$$
q
=
\operatorname{clip}
\left(
\operatorname{round}\left(\frac{x}{s}\right)+z,
q_{\min},q_{\max}
\right)
$$

其中：

- $s>0$ 是 Scale，决定相邻量化点在浮点域中的间隔；
- $z$ 是 Zero Point，使整数 $q=z$ 对应浮点零；
- $q_{\min},q_{\max}$ 由位宽和有符号/无符号格式决定；
- `round` 产生舍入误差，`clip` 产生范围外截断误差。

使用时做反量化：

$$
\hat x=s(q-z)
$$

量化误差为：

$$
e=x-\hat x
$$

例如 $x=0.73$、$s=0.1$、$z=0$：

$$
q=\operatorname{round}(0.73/0.1)=7,
\qquad
\hat x=7\times0.1=0.7
$$

因此误差为 $0.03$。对没有被截断的数，使用最近舍入时误差绝对值通常不超过半个量化步长：

$$
|x-\hat x|\leq \frac{s}{2}
$$

#### 10.2.2 对称量化、非对称量化与量化粒度

权重通常大致以零为中心，适合对称量化。令 $z=0$，对于有符号 $b$ bit 表示，可取：

$$
q_{\max}=2^{b-1}-1,
\qquad
s=\frac{\max|x|}{q_{\max}}
$$

例如对称 INT8 常使用 $[-127,127]$，避免正负范围不完全对称。某些内核也会利用完整的 $[-128,127]$，具体约定必须与模型文件一致。

若数据明显偏向一侧，可使用非对称量化：

$$
s=\frac{x_{\max}-x_{\min}}{q_{\max}-q_{\min}}
$$

$$
z
=
\operatorname{clip}
\left(
\operatorname{round}
\left(q_{\min}-\frac{x_{\min}}s\right),
q_{\min},q_{\max}
\right)
$$

Scale/Zero Point 可以按不同粒度计算：

|粒度|共享范围|特点|
|---|---|---|
|Per-Tensor|整个张量共用一组参数|元数据和计算开销最低，但容易被少数离群值拉大量程|
|Per-Channel|每个输入或输出通道一组|能适应通道间尺度差异，权重量化常用|
|Per-Group|连续 32、64、128 等个权重一组|INT4 常用，在精度、Scale 元数据和内核效率之间折中|
|Per-Token|每个 Token 的激活一组|能适应不同 Token 的动态范围，但需要运行时计算 Scale|
|Per-Block|二维小块共用一组|适合某些 FP8/低比特矩阵乘法硬件|

组越小，Scale 越能贴合局部分布，量化误差通常越低；代价是 Scale/Zero Point 元数据更多，并可能降低内核吞吐。所谓“4 bit 模型”实际平均位宽通常略高于 4 bit，因为还需要保存这些元数据和对齐填充。

#### 10.2.3 低比特线性层怎样计算

原始线性层为：

$$
y=Wx
$$

若权重和激活分别量化：

$$
W\approx s_W(Q_W-z_W),
\qquad
x\approx s_x(Q_x-z_x)
$$

在 Per-Tensor Scale 的简化情况下：

$$
Wx
\approx
s_Ws_x
(Q_W-z_W)(Q_x-z_x)
$$

对称量化取 $z_W=z_x=0$ 时，中间乘加可以由整数矩阵乘法完成，再乘 $s_Ws_x$ 恢复尺度。Per-Channel/Per-Group Scale 则需要在相应维度广播或融合到内核中。

常用记号 `WmAn` 表示权重为 $m$ bit、激活为 $n$ bit：

|方案|权重|激活|典型执行方式|
|---|---|---|---|
|W8A8|INT8/FP8|INT8/FP8|使用硬件低精度矩阵乘法，再以较高精度累加|
|W4A16|INT4|FP16/BF16|权重压缩存储，内核即时解包/反量化后参与高精度乘加|
|W4A8|INT4|INT8/FP8|同时降低权重带宽和激活计算位宽，但校准与内核更复杂|

因此“权重是 INT4”不代表所有乘法和累加都在 INT4 中进行。许多 Weight-Only 内核只是用 INT4 减少显存读取，在寄存器或片上存储中解包为 FP16/BF16；只有硬件和内核真正支持相应低精度 GEMM 时，低位宽才会同时减少算术成本。

Decode 通常更受权重读取带宽限制，W4A16 可能很有价值；大 batch 的 Prefill 更可能受矩阵计算限制，W8A8、FP8 或高效 W4A8 更容易发挥低精度计算单元的吞吐。量化能否加速不能只看模型文件大小，还取决于硬件指令、解包开销、矩阵形状、batch size 和融合内核质量。

#### 10.2.4 误差、离群值与校准

均匀量化要在两类误差之间折中：

- **量程过大**：极端值不被截断，但 $s$ 变大，普通值之间的量化网格太粗；
- **量程过小**：多数普通值表示更精细，但离群值会被 `clip` 截断。

少数权重或激活离群值可能占据大部分动态范围，使大量普通值集中到少数量化格点。误差还会经过多层残差、Attention 和 FFN 传播，因此不同层、通道和张量对量化的敏感度并不相同。Embedding、LM Head、归一化统计或少数异常通道有时会保留较高精度，这称为混合精度量化。

校准（Calibration）使用一小批具有代表性的输入，收集权重或激活的最大值、百分位数、直方图、均方误差等统计量，再选择 Scale、裁剪阈值和量化粒度。校准集与实际部署数据差异过大时，量化模型可能在校准测试正常、上线后明显退化。

#### 10.2.5 PTQ、QAT 与代表性方法

**训练后量化（Post-Training Quantization, PTQ）** 在浮点模型训练完成后进行：

```text
浮点模型
  → 收集权重/激活统计
  → 选择 Scale、Zero Point、分组与裁剪范围
  → 量化并做误差补偿
  → 在目标推理内核上验证精度和速度
```

PTQ 不需要重新进行完整预训练，成本较低。典型思路包括：

- **GPTQ**：使用近似二阶信息，按层或按块量化权重，并补偿已量化权重造成的输出误差；
- **AWQ**：根据激活分布识别更重要的权重通道，通过等价缩放降低这些通道的量化误差；
- **SmoothQuant**：通过数学等价的通道缩放，把难量化的激活离群程度迁移到相对容易量化的权重上，以支持 W8A8。

**量化感知训练（Quantization-Aware Training, QAT）** 在训练前向中插入伪量化：

$$
x
\rightarrow
\operatorname{Quantize}(x)
\rightarrow
\operatorname{Dequantize}(x)
$$

模型在训练时便看见舍入和截断误差。由于 `round` 几乎处处导数为零，反向传播通常使用 Straight-Through Estimator，在未截断范围内近似令：

$$
\frac{\partial\operatorname{round}(x)}{\partial x}
\approx 1
$$

QAT 通常比 PTQ 更能适应低 bit 表示，但需要训练数据、算力和更复杂的数值稳定性控制。

QLoRA 属于另一种常见场景：预训练基座以 4 bit 冻结保存，前向时反量化参与计算，梯度穿过基座流向可训练的 LoRA 参数；它并不是直接用普通优化器更新离散的 4 bit 基座权重。

#### 10.2.6 INT、NF4 与浮点量化

不同低精度格式的量化点分布不同：

- **INT8/INT4**：量化点通常均匀分布，Scale 决定网格间隔，适合硬件整数乘加；
- **NF4（NormalFloat 4）**：使用非均匀 4 bit Codebook，使量化点更贴近近似正态分布的预训练权重，常用于 QLoRA；计算前通常需要查表/反量化到计算类型；
- **FP8**：仍保留符号、指数和尾数，量化间隔会随数值指数变化，动态范围通常大于同位宽整数，更适合权重、激活和训练张量具有较大尺度变化的场景。

NF4 的“4 bit”主要描述存储编码，不意味着硬件直接执行 NF4 乘法。QLoRA 还可对 Scale 等量化常数再次量化，即 Double Quantization，以进一步减少元数据开销。

#### 10.2.7 FP8 E4M3 与 E5M2

FP8 常见两种编码为：

```text
E4M3：1 个符号位 + 4 个指数位 + 3 个尾数位
E5M2：1 个符号位 + 5 个指数位 + 2 个尾数位
```

E4M3 用一个指数位换取一个尾数位，因此精度相对较高、动态范围较小；E5M2 的指数更多，动态范围更大、尾数精度更低。

对普通规格化 E4M3 数，使用常见指数偏置 $\mathrm{bias}=7$：

$$
x
=(-1)^s
2^{E-7}
\left(1+\frac{M}{2^3}\right)
$$

其中 $s$ 是符号位，$E$ 是 4 bit 指数字段，$M$ 是 3 bit 尾数字段。隐含的前导 1 使规格化数具有 4 个二进制有效位。

例如：

$$
1.375
=1.011_2\times2^0
$$

因此：

```text
符号位：0
指数：0 + 7 = 7 = 0111₂
尾数：011
编码：0 0111 011
```

FP8 论文提出的 E4M3 扩展了有限数范围：不表示正负无穷，并只保留一个尾数模式表示 NaN。其典型数值范围为：

|格式|最大有限值|最小正规格化正数|最小正非规格化数|特殊值|
|---|---:|---:|---:|---|
|E4M3|448|$2^{-6}=0.015625$|$2^{-9}=0.001953125$|NaN，不表示 $\pm\infty$|
|E5M2|57344|$2^{-14}\approx6.10\times10^{-5}$|$2^{-16}\approx1.53\times10^{-5}$|NaN 和 $\pm\infty$|

具体硬件或软件还可能使用 E4M3FN、E4M3FNUZ 等变体，它们对负零、NaN、无穷和最大有限值的约定可能不同；模型格式、Scale 和内核必须采用同一规范。

E4M3 在 $[1,2)$ 区间的相邻规格化数间隔为：

$$
2^{-3}=0.125
$$

例如可以表示 $1.000,1.125,1.250,1.375,\ldots$。这种精度远低于 FP16/BF16，所以 FP8 几乎总要配合缩放：

$$
x_{\text{FP8}}
=
\operatorname{cast}_{\text{FP8}}
\left(\frac{x}{s}\right),
\qquad
\hat x=sx_{\text{FP8}}
$$

Scale 可以由当前张量的绝对最大值、历史 Amax，或更细粒度的 Block 统计产生。常见训练方案倾向于让 E4M3 承担需要较高精度的前向权重/激活，让动态范围更大的 E5M2 承担部分梯度；这不是固定规则，现代硬件和训练框架也可能对不同张量统一使用 E4M3 或采用更细粒度缩放。

低精度乘法通常以 FP16、BF16 或 FP32 累加。即使前向使用 FP8，训练仍可能保存 BF16/FP32 主权重、梯度副本以及 FP32 优化器状态；因此“参数计算为 FP8”不等于训练总显存自动缩小到 BF16 的一半。

#### 10.2.8 显存收益与三个量化对象

忽略 Scale、Zero Point 和对齐元数据，$N$ 个参数、每个参数 $b$ bit 的理论权重大小为：

$$
\text{Weight Memory}
\approx
N\frac{b}{8}\ \text{bytes}
$$

以 7B 参数为例：

|权重格式|理论权重大小|
|---|---:|
|FP32|约 28 GB|
|FP16/BF16|约 14 GB|
|INT8/FP8|约 7 GB|
|INT4/NF4|约 3.5 GB|

实际显存还包括量化 Scale、Zero Point、未量化层、运行时 Workspace 和其他状态。训练还要加上激活、梯度、主权重和优化器状态；推理则要加上 KV Cache，因此不能只用模型文件大小估计总显存。

需要分别说明量化对象：

|对象|主要收益|主要难点|
|---|---|---|
|参数/权重量化|缩小模型文件与权重读取带宽|不同层和通道敏感度不同，低 bit 下误差明显|
|激活量化|减少激活带宽并启用低精度 GEMM|激活随 Token 动态变化，离群值更难校准|
|KV Cache 量化|降低长上下文 Decode 的缓存容量和读取带宽|误差会影响每一层对历史信息的读取|

例如一个模型可以采用：

```text
权重 INT4 + 激活 FP16 + KV Cache INT8
```

最终部署应同时检查任务精度、长上下文质量、首 Token 延迟、单 Token Decode 延迟、吞吐和峰值显存。量化率更高不一定整体更快，也不保证对所有任务同样稳定。

### 10.3 训练和推理的根本差别

|维度|训练|自回归推理|
|---|---|---|
|序列计算|整段并行，使用因果 Mask|逐 Token 生成|
|保存内容|为反向传播保存或重算激活|保存历史 K/V|
|主要内存|参数、梯度、优化器状态、激活|参数、KV Cache|
|低精度侧重点|混合精度计算，并可能保留高精度主权重/优化器状态|权重、激活和 KV Cache 可按部署内核分别量化|
|输出使用|所有有效位置参与 Loss|通常只使用最后位置|
|随机性|数据顺序、Dropout 等|采样温度、Top-k/Top-p 等|

Softmax 只给出概率分布。推理时如何选 Token 是解码策略：

- Greedy：取最大概率；
- Temperature：缩放 Logits；
- Top-k：只保留概率最高的 k 个；
- Top-p：保留累计概率达到阈值的最小集合。

这些超参数影响生成风格和随机性，不改变模型权重。

### 10.4 现代 Decoder-only 架构的常见组合

一个常见现代配置是：

```text
Tokenizer
  → Token Embedding
  → N × [
       RMSNorm
       → GQA/MHA + RoPE
       → Residual
       → RMSNorm
       → SwiGLU FFN 或 MoE
       → Residual
     ]
  → Final RMSNorm
  → tied LM Head
```

采用 N-gram PLE 与 Gated Residual 的混合架构可写成另一条具体数据流：

```text
Token IDs ──→ Token Embedding ──→ 复制为 4 条残差流
    │                                  │
    └─→ 2/3-gram 多头哈希查表 ─→ PLE 门控注入（早期层）
                                       │
       12 × [                         ▼
           3 × (GR 读 → Gated DeltaNet → GR 写 → GR 读 → MoE → GR 写)
           1 × (GR 读 → QSA            → GR 写 → GR 读 → MoE → GR 写)
       ]
                                       │
                              最终 GR 只读合并
                                       ↓
                              Final Norm → LM Head
```

这里 N-gram Embedding 增加可寻址的局部模式容量，Gated DeltaNet 以固定状态传播大部分历史，QSA 从原始 KV 中稀疏检索细节，MoE 增加每 Token 条件计算容量，Gated Residual 则管理这些子层沿网络深度读写哪条残差流。它们分别优化不同资源，不能仅因为都含有“稀疏、门控或缓存”就视为同一种机制。

这不是唯一正确架构。不同模型会选择绝对/相对位置、稠密/稀疏 Attention、Dense/MoE FFN、不同 Norm 和 Bias 配置。理解这些变化时，应先问它改变了哪一种资源或归纳偏置：

- 参数容量；
- 每 Token 计算量；
- KV Cache；
- 长上下文连接；
- 训练稳定性；
- 分布式通信。

---

## 十一、把整个原理串起来

### 11.1 前向传播

1. Tokenizer 把文本变为 ID。
2. Embedding 查表得到语义向量。
3. 绝对位置向量、RoPE 或 Attention Bias 注入顺序信息。
4. Attention 用 Q/K 决定从哪些位置读取，用 V 传递内容。
5. FFN 在每个位置做非线性特征变换。
6. 残差保存旧信息，Norm 稳定各层输入尺度。
7. 重复多层后，LM Head 产生词表 Logits。
8. Softmax/交叉熵把 Logits 与下一个 Token 标签比较。

### 11.2 反向传播

1. 从 $p-y$ 得到 Logit 梯度。
2. 穿过 LM Head 和 Final Norm。
3. 在每个残差处把梯度分成 skip 与变换两路。
4. FFN 梯度穿过 down、激活、gate/up。
5. Attention 梯度按 $O=AV$、Softmax、$QK^\top$、RoPE、QKV 投影逆向展开。
6. 所有到达同一张量或共享参数的梯度求和。
7. 优化器用最终梯度更新参数。

### 11.3 最容易混淆的十一件事

1. **因果 Mask 不等于位置编码**：前者限制可见性，后者提供坐标和距离。
2. **数学可计算到长位置不等于模型会长上下文**：还受训练分布和 Attention 行为限制。
3. **FlashAttention 不等于线性注意力**：前者优化稠密 Softmax Attention 的 IO 和中间存储，后者改变算子并把历史压缩为固定状态。
4. **Sparse Attention 不等于 FlashAttention 或 PagedAttention**：Sparse Attention 删除部分数学连接；FlashAttention 流式计算同一 Attention；PagedAttention 优化 KV Cache 的分页存储和调度。
5. **KV Cache 不减少历史读取长度**：它避免重算历史 K/V。
6. **反向传播不等于梯度下降**：前者算梯度，后者/优化器用梯度更新。
7. **Attention Mask 不等于 Loss Mask**：一个控制读取，一个控制监督。
8. **总参数量不等于每 Token 激活参数量**：MoE 中二者尤其不同。
9. **4 bit 权重不等于所有计算都用 4 bit**：低 bit 可能只用于存储，反量化、乘法、累加和训练状态可以使用不同精度。
10. **MLA 不等于 DSA**：MLA 用低秩潜变量压缩每个 Token 的 KV Cache；DSA 用索引器选择当前 Query 要读取的 Token。前者改变缓存表示，后者改变可见连接，二者可以叠加。
11. **MLA 压缩不等于 CSA/HCA 压缩**：MLA 把一个 Token 的多头 K/V 沿特征维压成一份潜变量，缓存条目数仍约为 $L$；CSA/HCA 把多个 Token 沿序列维合成一个共享 K=V 条目，缓存条目数约变为 $L/m$ 或 $L/m'$。

---

## 十二、核心公式速查

$$
\operatorname{Attention}(Q,K,V)
=
\operatorname{softmax}
\left(
\frac{QK^\top}{\sqrt{d_h}}+M
\right)V
$$

$$
\text{Sparse Attention:}\qquad
o_i
=
\sum_{j\in\mathcal S_i}
\operatorname{softmax}_{j\in\mathcal S_i}
\left(
\frac{q_i^\top k_j}{\sqrt{d_h}}
\right)v_j
$$

$$
\text{MLA:}\qquad
c_t^{KV}=W^{DKV}h_t,
\qquad
q_{t,r}^{A}=(W_r^{UK})^\top q_{t,r}^{C},
\qquad
\text{Cache}_t=\{c_t^{KV},k_t^R\}
$$

$$
\text{DSA Indexer:}\qquad
I_{t,s}
=
\sum_{j=1}^{H^I}
w_{t,j}^{I}
\operatorname{ReLU}
\left((q_{t,j}^{I})^\top k_s^{I}\right),
\qquad
\mathcal S_t=\operatorname{TopK}_{s\le t}(I_{t,s},k)
$$

$$
\text{CSA:}\qquad
C^{\mathrm{Comp}}=\operatorname{Compress}_m(H),
\qquad
\mathcal S_t=\operatorname{TopK}_s(I_{t,s},k),
\qquad
o_t=\operatorname{Attn}(q_t,C_{\mathcal S_t}^{\mathrm{Comp}},C_{\mathcal S_t}^{\mathrm{Comp}})
$$

$$
\text{HCA:}\qquad
C^{\mathrm{Comp}}=\operatorname{Compress}_{m'}(H),
\qquad
o_t=\operatorname{Attn}(q_t,C_{<t/m'}^{\mathrm{Comp}},C_{<t/m'}^{\mathrm{Comp}})
$$

$$
\text{Linear Attention:}\qquad
S_t=S_{t-1}+v_tk_t^\top,qquad
o_t=S_tq_t
$$

$$
\text{DeltaNet:}\qquad
S_t
=S_{t-1}
-\beta_t(S_{t-1}k_t-v_t)k_t^\top
$$

$$
\text{Gated DeltaNet:}\qquad
S_t
=S_{t-1}\left[\alpha_t(I-\beta_tk_tk_t^\top)\right]
+\beta_tv_tk_t^\top
$$

$$
\operatorname{RoPE}(q_p)^\top\operatorname{RoPE}(k_s)
=q_p^\top R((s-p)\theta)k_s
$$

$$
\operatorname{SwiGLU}(x)
=
\left[\operatorname{SiLU}(xW_{\text{gate}})
\odot(xW_{\text{up}})\right]W_{\text{down}}
$$

$$
L=-\frac1M\sum_{t\in\mathcal V}\log p_{t,y_t}
$$

$$
\frac{\partial L}{\partial z_{t,j}}
=
\frac{p_{t,j}-\mathbf1[j=y_t]}{M}
$$

$$
\frac{\partial L}{\partial W}=X^\top G_Y,\qquad
\frac{\partial L}{\partial X}=G_YW^\top
$$

$$
q
=
\operatorname{clip}
\left(
\operatorname{round}\left(\frac{x}{s}\right)+z,
q_{\min},q_{\max}
\right),
\qquad
\hat x=s(q-z)
$$

$$
\theta_{t+1}
=
\theta_t
-\eta\frac{\hat m_t}{\sqrt{\hat v_t}+\epsilon}
-\eta\lambda\theta_t
$$

---

## 十三、业界代表性 Transformer 架构图

本章按公开技术报告画出代表模型的逻辑结构。图中省略 Dropout、张量并行等实现细节；同一模型家族的层数和维度可能随规格变化。

### 13.1 四种架构先放在同一张设计地图中

```text
Transformer 设计地图
│
├─ DeepSeek-V3 / V3.2 / V4
│  ├─ Decoder-only
│  ├─ MLA：V3/V3.2 压缩单 Token 的多头 KV
│  ├─ DSA：V3.2 用 Lightning Indexer 选择 Token
│  ├─ CSA/HCA：V4 压缩 Token 序列并混合稀疏/稠密读取
│  ├─ DeepSeekMoE：共享专家 + 稀疏路由专家
│  └─ MTP：增加未来 Token 辅助预测
│
├─ Qwen3
│  ├─ Decoder-only
│  ├─ GQA + RoPE
│  ├─ Dense / MoE 两类模型规格
│  └─ 同一模型支持思考与非思考模式
│
├─ Google T5
│  ├─ Encoder-Decoder
│  ├─ Cross-Attention
│  ├─ 相对位置 Bucket
│  └─ Text-to-Text 任务统一
│
└─ Google Gemma 3
   ├─ 多模态 Decoder-only
   ├─ SigLIP → 视觉 Token
   ├─ 5 个局部层 : 1 个全局层
   └─ 面向长上下文的位置配置
```

### 13.2 DeepSeek-V3 / V3.2 / V4：从潜变量缓存到混合压缩注意力

#### 13.2.1 DeepSeek-V3 整体骨干

```text
Token
  │
  ▼
Embedding
  │
  ▼
前 3 个 Dense Block
  ┌────────────────────────────────────┐
  │ RMSNorm → MLA → 残差相加           │
  │     ↓                              │
  │ RMSNorm → Dense FFN → 残差相加     │
  └────────────────────────────────────┘
  │
  ▼
后 58 个 MoE Block
  ┌────────────────────────────────────┐
  │ RMSNorm → MLA → 残差相加           │
  │     ↓                              │
  │ RMSNorm → DeepSeekMoE → 残差相加   │
  └────────────────────────────────────┘
  │                    └──训练阶段──→ MTP → 更远未来 Token 的辅助损失
  ▼
Final RMSNorm → LM Head → 下一 Token Logits
```

上图的 DeepSeek-V3 主模型约 671B 总参数、每 Token 激活约 37B。MLA 替换普通 Attention，DeepSeekMoE 替换大多数 Dense FFN；MTP 位于训练侧，不是常规逐 Token 解码必须经过的主路径。DeepSeek-V3.2 的注意力变化在第 13.2.5 节单独说明。

#### 13.2.2 MLA：缓存低维潜变量，而不是完整每头 K/V

```text
当前 Token 隐藏状态 h
│
├─→ Q 内容分量 ─────────────────────┐
│                                  ├─→ 拼接 Q ──┐
├─→ Q 位置分量 → RoPE ─────────────┘            │
│                                               ├─→ QK 点积 → Softmax ─┐
├─→ 低秩压缩 → KV 潜变量 c_KV                   │                     │
│       ├─→ 上投影为各头 K 内容分量 ──┐         │                     │
│       │                            ├─→ 拼接 K ─┘                     │
│       └─→ 上投影为各头 V ──────────┼────────────────────────────────┤
│                                    │                                ▼
└─→ K 位置分量 → RoPE ───────────────┘                         加权聚合 V

推理时主要缓存：c_KV + K 的位置分量
而不是为每个历史 Token 保存完整的每头 K、V
```

```text
标准 MHA/GQA：历史 Token → 缓存每个 KV 头的完整 K、V
MLA：         历史 Token → 主要缓存低维 c_KV 与位置分量 → 使用时再上投影
```

MLA 用额外的压缩/上投影换取更小的 KV Cache 和更低的读取带宽。位置分量单独施加 RoPE，避免把不能直接吸收到低秩缓存中的位置旋转与内容压缩混在一起。低秩联合压缩、Decoupled RoPE 和投影吸收的公式见第 4.4.2 节。

#### 13.2.3 DeepSeekMoE：共享知识与路由专长并行

```text
同一个 Token 表示 x
│
├─→ 共享专家（始终执行）──────────────────────────┐
│                                                │
└─→ Router → 从 256 个路由专家中选择 Top-8        │
             ├─→ 路由专家 e1 ─┐                  │
             ├─→ 路由专家 e2 ─┤                  │
             ├─→ ...          ├─→ 按门控权重加权 ─┤
             └─→ 路由专家 e8 ─┘                  │
                                                  ▼
                                    合并共享专家与路由专家
                                                  │
                                                  ▼
                                              MoE 输出
```

共享专家吸收跨领域的公共模式，细粒度路由专家学习不同 Token/领域的专长。每个 Token 只激活少数专家，因此总参数容量很大而活跃计算受控；代价是 Router 负载均衡和跨设备 All-to-All 通信。

#### 13.2.4 MTP 与无辅助损失负载均衡

```text
训练目标：

主干隐藏状态 h_t^(0) ──→ 共享 LM Head ─────────────→ 预测 x_(t+1)
          │
          └─与真实 E(x_(t+1)) 融合→ MTP 模块→ h_t^(1)
                                                  │
                                                  └─→ 预测 x_(t+2)

负载均衡：

各专家近期实际负载
       │
       ▼
动态调整路由偏置 → Router 选择 → 更均衡地分发到各专家
```

DeepSeek-V3 使用第 7.4 节所述的顺序式 MTP，并把深度设为 $D=1$：主模型预测 $x_{t+1}$，附加模块在 Teacher Forcing 下读取真实 $x_{t+1}$ 后预测 $x_{t+2}$。该模块主要提供训练辅助损失；常规推理可移除，也可保留为推测式解码的 Draft。无辅助损失负载均衡则是另一项独立机制，它通过动态路由偏置调节专家流量，尽量避免额外平衡损失干扰主任务目标。

#### 13.2.5 DeepSeek-V3.2：在 MLA 上增加 DSA

DeepSeek-V3 / V3.1 的 MLA 仍对全部历史潜变量执行稠密 Attention。DeepSeek-V3.2 保留 MLA 缓存形式，并在正式 Attention 前增加 Token 级 Lightning Indexer：

```text
DeepSeek-V3 / V3.1：
Query ─────────────────────────────→ 全部历史 MLA 潜变量 → Dense MLA

DeepSeek-V3.2：
Query → Lightning Indexer → Top-2048 Token ─┐
历史 MLA 潜变量 ────────────────────────────┴→ Sparse MLA
```

因此两代架构的关键变化不是把 MLA 换掉，而是把“MLA 压缩缓存”进一步变为“MLA 压缩缓存 + DSA 稀疏读取”。训练时先让索引器模仿稠密 Attention 的位置重要性，再启用 Top-$k$ 继续训练全模型；详细公式、具体示例和复杂度边界见第 4.7.6 节。

#### 13.2.6 DeepSeek-V4：交错使用 CSA 与 HCA

DeepSeek-V4 用序列维压缩取代 V3/V3.2 的 MLA 主体注意力。CSA 和 HCA 层共享同一套核心思想：低秩产生多头 Query，把压缩条目同时作为共享 Key 和 Value，并在长程压缩分支旁保留 128 Token 的未压缩滑动窗口。

```text
每个 V4 Attention Block
│
├─ 局部分支：最近 128 个原始 KV ─────────────────────┐
│                                                    │
└─ 长程分支（按层选择一种）                           │
   ├─ CSA：每 4 Token 产生约 1 个压缩条目             │
   │       → Lightning Indexer → Top-k                │
   │                                                  ├─→ Shared K=V MQA
   └─ HCA：每 128 Token 产生 1 个压缩条目              │
           → 稠密读取所有可见压缩条目                  │
                                                      ▼
                                            Grouped Output Projection
```

层间安排为：

```text
V4-Flash：前 2 层纯滑窗 → CSA ↔ HCA 交错，CSA Top-512
V4-Pro：  前 2 层 HCA   → CSA ↔ HCA 交错，CSA Top-1024
```

CSA 提供较高分辨率的远程内容检索，HCA 提供极低分辨率但覆盖完整历史的全局摘要，局部滑窗负责精确的近期依赖。完整压缩公式、Partial RoPE、Attention Sink、复杂度和配置对比见第 4.7.7 节。

### 13.3 Qwen3：同一 Decoder 骨干覆盖 Dense 与 MoE

#### 13.3.1 整体结构

```text
Token → Embedding
          │
          ▼
Qwen3 Decoder Block × N
┌───────────────────────────────────────────────┐
│ RMSNorm                                       │
│    ↓                                          │
│ GQA + RoPE + 因果 Mask                        │
│    ↓                                          │
│ 残差相加                                      │
│    ↓                                          │
│ RMSNorm                                       │
│    ↓                                          │
│    ├─ Dense 型号：SwiGLU FFN                  │
│    └─ MoE 型号：Router → Top-K 专家           │
│    ↓                                          │
│ 残差相加                                      │
└───────────────────────────────────────────────┘
          │
          ▼
Final RMSNorm → LM Head → 下一 Token Logits
                               │
                               └─→ 追加 Token 后继续自回归生成
```

Dense 与 MoE 不是同一次前向中的动态二选一，而是不同 Qwen3 型号在 FFN 子层采用的不同配置；Attention 和 Decoder-only 数据流保持一致。

#### 13.3.2 GQA：多组 Query 共享较少的 KV 头

```text
隐藏状态
│
├─→ 64 个 Query 头
│    ├─ Q 头  1～16 ──→ 共享 KV 头 1
│    ├─ Q 头 17～32 ──→ 共享 KV 头 2
│    ├─ Q 头 33～48 ──→ 共享 KV 头 3
│    └─ Q 头 49～64 ──→ 共享 KV 头 4
│
└─→ 4 个 KV 头
     ├─ KV 头 1 ──服务── Q 头  1～16
     ├─ KV 头 2 ──服务── Q 头 17～32
     ├─ KV 头 3 ──服务── Q 头 33～48
     └─ KV 头 4 ──服务── Q 头 49～64

MHA：64 个 Q 头 + 64 个 KV 头
GQA：64 个 Q 头 +  4 个 KV 头    ← Qwen3-235B-A22B
MQA：64 个 Q 头 +  1 个 KV 头
```

上图对应 Qwen3-235B-A22B 的 64 个 Query 头、4 个 KV 头：每 16 个 Query 头共享一组 K/V。与同样拥有 64 个 KV 头的 MHA 相比，KV 头数降为 $1/16$，主要节省生成阶段的 KV Cache 和带宽。

#### 13.3.3 Qwen3 MoE 与思考模式的边界

```text
MoE 子层：

Token 表示 x
    │
    ▼
Router → 从 128 个专家中选择 8 个
    │
    ▼
8 个专家分别处理同一个 x
    │
    ▼
按门控权重加权求和 → MoE 子层输出

思考模式：

                   ┌─ enable_thinking=true  → 生成 think 区段 → 最终回答
同一个 Qwen3 骨干 ─┤
                   └─ enable_thinking=false → 直接生成回答

注意：这里只改变聊天模板和生成行为，不更换模型骨干。
```

Qwen3-235B-A22B 总参数约 235B、每 Token 活跃约 22B；稀疏专家提升容量但不要求每次激活全部参数。思考开关控制聊天模板和生成行为，不会切换成另一套 Attention 或 MoE 网络。

### 13.4 Google T5：Encoder 负责理解，Decoder 负责条件生成

#### 13.4.1 完整 Encoder-Decoder 信息流

```text
源序列路径                                         目标序列路径

任务前缀 + 输入文本                                右移后的目标文本
        │                                                   │
        ▼                                                   ▼
共享 Token Embedding                              共享 Token Embedding
        │                                                   │
        ▼                                                   ▼
T5 Encoder × N                                    T5 Decoder × N
┌──────────────────────────┐                      ┌──────────────────────────┐
│ Pre-Norm                 │                      │ Pre-Norm                 │
│    ↓                     │                      │    ↓                     │
│ 双向 Self-Attention      │                      │ 因果 Self-Attention      │
│ + 相对位置 Bucket Bias   │                      │ + 相对位置 Bucket Bias   │
│    ↓                     │                      │    ↓                     │
│ 残差相加                 │                      │ 残差相加                 │
│    ↓                     │                      │    ↓                     │
│ Pre-Norm + FFN           │                      │ Cross-Attention          │
│    ↓                     │                      │    ↑                     │
│ 残差相加                 │────完整源表示 K、V──→│ Encoder 输出             │
└──────────────────────────┘                      │    ↓                     │
                                                  │ 残差 → FFN → 残差       │
                                                  └──────────────────────────┘
                                                               │
                                                               ▼
                                                    Final Norm + LM Head
                                                               │
                                                               ▼
                                                         目标文本 Token
```

Encoder 的每个位置能双向读取整个源序列；Decoder 的 Self-Attention 只能读取已有目标前缀，但 Cross-Attention 可以读取全部 Encoder 输出。源序列与目标序列因此具有独立长度和独立表示空间。

#### 13.4.2 Text-to-Text 与 span corruption

```text
Span corruption 预训练：

原始文本
  │
  ▼
遮盖若干连续片段
  │
  ├─ Encoder 输入：原文保留部分 + sentinel Token
  │
  └─ Decoder 目标：sentinel Token + 被遮盖片段

Text-to-Text 任务统一：

分类 ──→ “任务前缀 + 输入文本” ──→ T5 ──→ “类别文本”
翻译 ──→ “任务前缀 + 源语言”   ──→ T5 ──→ “目标语言文本”
摘要 ──→ “任务前缀 + 长文本”   ──→ T5 ──→ “摘要文本”
```

T5 的创新不只在 Block：它把分类、翻译、摘要、问答都统一成“文本输入 → 文本输出”，并用连续片段恢复进行预训练。相对位置 Bucket 让近距离保持精细、远距离逐渐合并，从而以有限参数覆盖较长距离。

### 13.5 Google Gemma 3：把视觉 Token 与长文本放入同一 Decoder

#### 13.5.1 多模态输入与语言主干

```text
图像路径                                      文本路径

输入图像                                      输入文本
   │                                             │
   ▼                                             ▼
可选 Pan & Scan 多裁剪                       Token Embedding
   │                                             │
   ▼                                             │
SigLIP 视觉编码器                                │
   │                                             │
   ▼                                             │
池化 / 压缩                                      │
   │                                             │
   ▼                                             │
固定 256 个视觉 soft token ────────────────┐     │
                                           ▼     ▼
                                      图文 Token 序列
                                             │
                                             ▼
Gemma 3 Decoder Block × N
┌─────────────────────────────────────────────────────┐
│ Pre/Post RMSNorm                                    │
│      ↓                                              │
│ GQA：当前层选择局部滑动窗口或全局注意力             │
│      ↓                                              │
│ 残差相加 → Gated FFN → 残差相加                    │
└─────────────────────────────────────────────────────┘
                                             │
                                             ▼
                               Final Norm + LM Head
                                             │
                                             ▼
                                      下一 Token Logits
```

视觉表示不通过每层独立的 Cross-Attention 注入，而是转成 soft token，与文本 Token 一起进入同一个因果 Decoder。固定 256 个视觉 Token 控制语言模型实际处理的图像序列长度。

#### 13.5.2 五个局部层配一个全局层

```text
输入表示
   │
   ▼
局部 GQA 1 ─→ 局部 GQA 2 ─→ 局部 GQA 3
窗口 1024      窗口 1024      窗口 1024
   │
   ▼
局部 GQA 4 ─→ 局部 GQA 5 ─→ 全局 GQA
窗口 1024      窗口 1024      完整上下文
                                  │
                                  └─→ 进入下一组“5 局部 + 1 全局”层

位置频率配置：

局部层：RoPE base = 10K  → 关注短窗口内的相对位置
全局层：RoPE base = 1M   → 适配更长尺度的位置变化
```

```text
局部层：低成本传播邻近信息，KV Cache 只需覆盖窗口
全局层：周期性建立跨窗口连接，维持长距离信息整合
```

这种 5:1 层序把“每层都做全局 Attention”改为“多数局部、少数全局”，显著降低长上下文 KV Cache 成本；代价是远距离 Token 不能在每一层直接交互。Gemma 3 再通过全局层更大的 RoPE base、训练后期缩放和 Pan & Scan 支持长上下文与高分辨率图像。

---

## 参考资料

1. Vaswani et al., [Attention Is All You Need](https://arxiv.org/abs/1706.03762).
2. Shaw et al., [Self-Attention with Relative Position Representations](https://arxiv.org/abs/1803.02155).
3. Raffel et al., [Exploring the Limits of Transfer Learning with a Unified Text-to-Text Transformer](https://jmlr.org/papers/v21/20-074.html).
4. Su et al., [RoFormer: Enhanced Transformer with Rotary Position Embedding](https://arxiv.org/abs/2104.09864).
5. Press et al., [Train Short, Test Long: Attention with Linear Biases](https://arxiv.org/abs/2108.12409).
6. Chen et al., [Extending Context Window of Large Language Models via Positional Interpolation](https://arxiv.org/abs/2306.15595).
7. Peng et al., [YaRN: Efficient Context Window Extension of Large Language Models](https://arxiv.org/abs/2309.00071).
8. Dao et al., [FlashAttention: Fast and Memory-Efficient Exact Attention](https://arxiv.org/abs/2205.14135).
9. Shazeer, [GLU Variants Improve Transformer](https://arxiv.org/abs/2002.05202).
10. Shazeer, [Fast Transformer Decoding: One Write-Head is All You Need](https://arxiv.org/abs/1911.02150).
11. Ainslie et al., [GQA: Training Generalized Multi-Query Transformer Models from Multi-Head Checkpoints](https://arxiv.org/abs/2305.13245).
12. Zhang and Sennrich, [Root Mean Square Layer Normalization](https://arxiv.org/abs/1910.07467).
13. Loshchilov and Hutter, [Decoupled Weight Decay Regularization](https://arxiv.org/abs/1711.05101).
14. PyTorch, [DistributedDataParallel](https://pytorch.org/docs/stable/generated/torch.nn.parallel.DistributedDataParallel.html).
15. DeepSeek-AI, [DeepSeek-V3 Technical Report](https://arxiv.org/abs/2412.19437).
16. Yang et al., [Qwen3 Technical Report](https://arxiv.org/abs/2505.09388).
17. Gemma Team, [Gemma 3 Technical Report](https://arxiv.org/abs/2503.19786).
18. Qwen Team, [Qwen3-235B-A22B Model Card](https://huggingface.co/Qwen/Qwen3-235B-A22B).
19. Gloeckle et al., [Better & Faster Large Language Models via Multi-token Prediction](https://arxiv.org/abs/2404.19737).
20. Katharopoulos et al., [Transformers are RNNs: Fast Autoregressive Transformers with Linear Attention](https://arxiv.org/abs/2006.16236).
21. Schlag et al., [Linear Transformers Are Secretly Fast Weight Programmers](https://arxiv.org/abs/2102.11174).
22. Yang et al., [Parallelizing Linear Transformers with the Delta Rule over Sequence Length](https://arxiv.org/abs/2406.06484).
23. Yang et al., [Gated Delta Networks: Improving Mamba2 with Delta Rule](https://arxiv.org/abs/2412.06464).
24. Micikevicius et al., [FP8 Formats for Deep Learning](https://arxiv.org/abs/2209.05433).
25. Frantar et al., [GPTQ: Accurate Post-Training Quantization for Generative Pre-trained Transformers](https://arxiv.org/abs/2210.17323).
26. Xiao et al., [SmoothQuant: Accurate and Efficient Post-Training Quantization for Large Language Models](https://arxiv.org/abs/2211.10438).
27. Lin et al., [AWQ: Activation-aware Weight Quantization for LLM Compression and Acceleration](https://arxiv.org/abs/2306.00978).
28. Dettmers et al., [QLoRA: Efficient Finetuning of Quantized LLMs](https://arxiv.org/abs/2305.14314).
29. Beltagy et al., [Longformer: The Long-Document Transformer](https://arxiv.org/abs/2004.05150).
30. Zaheer et al., [Big Bird: Transformers for Longer Sequences](https://arxiv.org/abs/2007.14062).
31. Jiang et al., [Mistral 7B](https://arxiv.org/abs/2310.06825).
32. Yuan et al., [Native Sparse Attention: Hardware-Aligned and Natively Trainable Sparse Attention](https://arxiv.org/abs/2502.11089).
33. Qwen Team, [Qwen3.8-Flash-Next Model Card](https://huggingface.co/Qwen/Qwen3.8-Flash-Next-FP8).
34. Qwen Team, [Qwen3.8-Flash-Next Technical Report and Repository](https://github.com/QwenLM/Qwen3.8-Flash-Next).
35. SGLang Team, [Qwen3.8-Flash-Next: Day-0 Support in SGLang](https://www.lmsys.org/blog/2026-08-26-qwen-flash-next/).
36. Qwen Team, [Qwen3.8-Flash-Next: A New Architecture, Towards Ultimate Cost-Efficiency](https://qwen.ai/blog?id=qwen3.8-flash-next).
37. Hugging Face Transformers, [Qwen4-Exp Architecture and Configuration](https://huggingface.co/docs/transformers/main/en/model_doc/qwen4_exp).
38. Hugging Face Transformers, [Qwen4-Exp Reference Implementation](https://github.com/huggingface/transformers/blob/main/src/transformers/models/qwen4_exp/modular_qwen4_exp.py).
39. DeepSeek-AI, [DeepSeek-V2: A Strong, Economical, and Efficient Mixture-of-Experts Language Model](https://arxiv.org/abs/2405.04434).
40. DeepSeek-AI, [DeepSeek-V3.2: Pushing the Frontier of Open Large Language Models](https://huggingface.co/deepseek-ai/DeepSeek-V3.2/resolve/main/assets/paper.pdf).
41. DeepSeek-AI, [DeepSeek-V3.2-Exp Repository and Reference Implementation](https://github.com/deepseek-ai/DeepSeek-V3.2-Exp).
42. DeepSeek-AI, [DeepSeek-V4: Towards Highly Efficient Million-Token Context Intelligence](https://arxiv.org/abs/2606.19348).
43. DeepSeek-AI, [DeepSeek-V4-Pro Model Card](https://huggingface.co/deepseek-ai/DeepSeek-V4-Pro).
44. DeepSeek-AI, [DeepSeek-V4 Reference Inference Implementation](https://huggingface.co/deepseek-ai/DeepSeek-V4-Pro/tree/main/inference).
