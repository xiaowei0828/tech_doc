# RLHF 详解：基于人类反馈的强化学习全流程

**范围说明：**本文以 InstructGPT 风格的 **SFT → 偏好奖励模型 → PPO** 为主线，再对比 DPO、GRPO、RLAIF 与 RLVR。它是一条经典管线，不是所有“对齐”系统都必须采用的唯一流程。LoRA 只是可选的参数高效实现方式，不能与 RLHF 本身画等号。

## 一、RLHF 概述

### 1\.1 为什么需要人类反馈？

预训练目标学习的是数据分布中的下一个 token，SFT 目标学习的是示范答案的条件似然；二者都不直接等价于“在多种可行回答中更偏好哪一个”。因此模型仍可能出现：

- **数据集偏差：**再高质量的指令数据集，也无法完美涵盖人类偏好的方方面面

- **目标错位：**交叉熵能训练高质量示范，但单个参考答案无法完整表达帮助性、事实性、安全性、简洁度等多目标取舍

- **答非所问：**模型可能生成语法正确但内容空洞、无趣甚至有害的回复

### 1\.2 RLHF 核心目标

把特定标注规范、标注者群体和产品政策下的偏好信号转化为可优化目标，使策略在目标 prompt 分布上更倾向产生被偏好的回答。它优化的是一个有覆盖范围和测量误差的代理目标，不能证明模型已经“对齐全体人类价值观”。

从优化角度看，经典 RLHF 要解决的是一个带参考策略约束的期望奖励最大化问题：

$$
\max_{\pi_\theta}\;
\mathbb{E}_{x\sim\mathcal{D},\,y\sim\pi_\theta(\cdot\mid x)}
\left[r_\phi(x,y)\right]
-\beta\,
\mathbb{E}_{x\sim\mathcal{D}}
\left[D_{\mathrm{KL}}\!\left(
\pi_\theta(\cdot\mid x)\,\|\,\pi_{\mathrm{ref}}(\cdot\mid x)
\right)\right].
$$

其中：

|符号|含义|
|---|---|
|$x$|prompt 或对话上下文|
|$y=(y_1,\ldots,y_T)$|模型生成的完整回答|
|$\pi_\theta$|待优化的策略模型（Actor）|
|$\pi_{\mathrm{ref}}$|冻结的参考策略，通常来自 SFT 模型|
|$r_\phi(x,y)$|由偏好数据训练出的奖励模型分数|
|$\beta$|奖励优化与偏离参考策略之间的权衡系数|

第一项鼓励策略生成高奖励回答；第二项限制策略过快偏离 SFT 分布。这里的 KL 约束既是稳定训练的正则项，也是在奖励模型覆盖有限时降低 reward hacking 风险的保护措施，但它不能替代独立评测与安全约束。

### 1\.3 三阶段管线总览

#### 全流程工程视图

经典 InstructGPT 风格流程分为三个阶段：**SFT → 奖励模型训练 → PPO 强化学习**。后来的 DPO 等方法可以绕过显式奖励模型和在线 PPO，因此“RLHF”在工程语境中有时被宽泛地用于整类偏好后训练，阅读资料时应先确认口径。

|阶段|名称|核心目标|
|---|---|---|
|1|SFT（监督微调）|让预训练模型初步具备指令跟随能力|
|2|Reward Model 训练|训练一个能代替人类打分的神经网络|
|3|PPO 强化学习|用 RM 打分作为奖励信号，微调 LLM 生成高分回答|

更完整的数据流是：

```text
示范数据 ──SFT──> 初始策略
                      │
目标 prompts ──多次采样──> 候选回答 ──人类排序──> 偏好对
                                                   │
                                                   └──训练──> Reward Model
初始策略 ──复制并冻结──> Reference                  │
初始策略 ──初始化──> Actor ──rollout──> 回答 ──RM/KL──> 奖励
                                ↑                         │
                                └──── PPO + Value ────────┘
```

数据收集、奖励建模、策略优化和独立评估应形成迭代闭环；“训练 reward 上升”本身不是完成条件。

---

## 二、实现前置：LoRA 是可选的参数高效技术

LoRA 可以用于 SFT、RM、Actor 或 Value 等可训练模型，但也可以全量微调、只训练 Head，或采用其他 PEFT/分片方案。下面先讲 LoRA，是为了说明本文示例中“冻结底座、只更新 Adapter”的梯度边界。

### 2\.1 为什么需要 LoRA？

#### 显存收益的准确口径

LoRA 的显存收益不能只用“模型参数大小”概括。工程上至少要区分参数、梯度、优化器状态、激活值、通信缓存和并行策略。

|显存项|全量微调|LoRA / QLoRA|
|---|---|---|
|冻结底座参数|需要加载，并通常参与梯度图|需要加载，但权重冻结；QLoRA 可用 NF4/INT4 降低占用|
|可训练参数|全部 Transformer 参数可训练|仅 Adapter 矩阵与少量 Head 可训练|
|梯度|全量参数梯度占用大|只保存 LoRA 参数梯度，占用显著下降|
|优化器状态|AdamW 通常需要 m/v 两份状态|只为 LoRA 参数维护优化器状态|
|激活值|仍是训练显存大头之一|LoRA 不能完全消除激活值，需要 checkpointing、packing、FlashAttention 配合|

注意：“70B FP16/BF16 权重约 140GB”只代表底座权重，不等价于训练总显存。类似“LoRA rank=16 只需约 35GB”的数字只有在 4bit/8bit 量化、较小 batch/sequence length、activation checkpointing、offload 或分布式分片等具体前提下才有意义。普通 FP16/BF16 加载 70B 底座，仅权重本身就约 140GB。

### 2\.2 核心数学原理

对线性层 $h=Wx$，LoRA 冻结 $W$，只学习低秩增量：

$$
h = Wx + \frac{\alpha}{r}BAx,
\qquad
A\in\mathbb{R}^{r\times d_{\mathrm{in}}},
\quad
B\in\mathbb{R}^{d_{\mathrm{out}}\times r}.
$$

当 $r\ll \min(d_{\mathrm{in}},d_{\mathrm{out}})$ 时，可训练参数从 $d_{\mathrm{out}}d_{\mathrm{in}}$ 降为 $r(d_{\mathrm{in}}+d_{\mathrm{out}})$。例如方阵维度 $d=4096$、$r=16$ 时，单层增量参数约为 $13.1$ 万，而原矩阵约为 $1678$ 万；这里的“参数节省”不等于端到端训练显存按相同比例下降。

### 2\.3 前向传播

先计算 $z=Ax$ 将输入投影到 $r$ 维，再计算 $Bz$ 投影回输出维度，最后按 $\alpha/r$ 缩放并与冻结主干 $Wx$ 相加。常见初始化是随机初始化 $A$、将 $B$ 初始化为零，使训练起点的 LoRA 分支输出为零，初始函数与底座一致。

### 2\.4 反向传播

**关键：**梯度只更新 $A$ 和 $B$，原始权重 $W$ 冻结，不分配 $W$ 的参数梯度和优化器状态。

设上游列向量梯度为 $g=\partial\mathcal{L}/\partial h$，则

$$
\frac{\partial\mathcal{L}}{\partial B}
=\frac{\alpha}{r}g(Ax)^\top,
\qquad
\frac{\partial\mathcal{L}}{\partial A}
=\frac{\alpha}{r}B^\top g\,x^\top,
$$

$$
\frac{\partial\mathcal{L}}{\partial x}
=W^\top g+\frac{\alpha}{r}A^\top B^\top g.
$$

$W$ 不保存参数梯度或优化器状态；但为了向更早层传播梯度，反向过程仍需要使用 $W^\top$，并保存或重算相关激活。

### 2\.5 LoRA 具体注入位置与 Head 边界

LoRA 不是加在模型外部，而是插入到 Transformer 内部的线性层旁边，形成一个可训练的低秩旁路。原始权重冻结，只训练 LoRA 的 `A/B` 矩阵：

```text
列向量约定下：
原始线性层：h = W x
加 LoRA 后：h = W x + (alpha / r) B A x
```

#### 2\.5\.1 最常加 LoRA 的模块

|模块位置|常见参数名|作用|是否常用|
|---|---|---|---|
|Attention Q 投影|`q_proj` / `W_Q`|决定当前 token 查询什么信息|高频|
|Attention K 投影|`k_proj` / `W_K`|决定 token 如何被匹配|高频|
|Attention V 投影|`v_proj` / `W_V`|决定被取出的内容表示|高频|
|Attention O 投影|`o_proj` / `W_O`|融合多头注意力输出|高频|
|FFN Gate 投影|`gate_proj`|SwiGLU 门控分支，控制激活强度|常用|
|FFN Up 投影|`up_proj`|将 hidden 扩展到中间维度|常用|
|FFN Down 投影|`down_proj`|将 FFN 中间表示压回 hidden 维度|常用|
|LM Head|`lm_head`|输出词表 logits|可选|
|Embedding|`embed_tokens`|token 查表|通常不加|

> **实践口径：**资源紧张时可以只加 `q_proj/v_proj` 或 `q_proj/k_proj/v_proj/o_proj`；追求更强适配能力时，通常加 `q_proj/k_proj/v_proj/o_proj/gate_proj/up_proj/down_proj`。

#### 2\.5\.2 各 RLHF 阶段的训练边界

|阶段|推荐 LoRA 位置|训练状态|说明|
|---|---|---|---|
|SFT|`Q/K/V/O`，可加 `gate/up/down`|训练 LoRA|学习指令跟随、对话格式和回答风格|
|Reward Model 训练|Backbone 的 `Q/K/V/O`，可加 `gate/up/down`|训练 LoRA|让底座适配偏好判断；Reward Head 通常直接训练|
|PPO Actor|Actor 的 `Q/K/V/O`，常加 `gate/up/down`|训练 LoRA|真正接收策略梯度更新的部分|
|PPO Value|Value Backbone 可加 LoRA；Value Head 直接训练|可选训练|可从策略或 RM backbone 初始化，具体取决于实现|
|PPO Reference|不新增训练 LoRA，或加载 SFT LoRA 后冻结|冻结|只作为 KL 基准，不参与更新|
|PPO Reward|不新增训练 LoRA，或加载 RM LoRA 后冻结|冻结|只负责给完整回答打分|
|GRPO Actor|同 PPO Actor|训练 LoRA|GRPO 去掉 Value Model，但 Actor 仍可用 LoRA 更新|

#### 2\.5\.3 LLaMA / Qwen 类模型常见 `target_modules`

```python
target_modules = [
    "q_proj",
    "k_proj",
    "v_proj",
    "o_proj",
    "gate_proj",
    "up_proj",
    "down_proj",
]
```

如果训练资源更紧，可以先只覆盖 Attention：

```python
target_modules = [
    "q_proj",
    "k_proj",
    "v_proj",
    "o_proj",
]
```

#### 2\.5\.4 Reward Head / Value Head 与 LoRA 的关系

Reward Head 和 Value Head 都是接在 Transformer backbone 后面的轻量线性输出头，通常不需要 LoRA。

|Head|用途|输入位置|输出|训练方式|
|---|---|---|---|---|
|Reward Head|判断完整回答质量|通常取最后 token 或 EOS hidden state|一个 reward 分数|通常直接全量训练|
|Value Head|估计当前状态未来收益|每个 token hidden state|每个 token 一个 value|PPO 中直接训练|

Reward Head 的典型形式：

```text
r(x, y) = w_r^T h_last + b_r
```

Value Head 的典型形式：

```text
V(s_t) = w_v^T h_t + b_v
```

> **边界说明：**LoRA 主要用于 Transformer backbone 的大矩阵；Reward Head / Value Head 参数量很小，通常直接训练即可。PPO 阶段的 Reward Model 和 Reference Model 是冻结模型，即使它们内部带有已训练好的 LoRA，也不会继续更新。

---

## 三、阶段一：监督微调（SFT）详解

### 3\.1 目标与数据格式

**目标：**让预训练模型初步具备指令跟随能力，成为后续 RLHF 的起点。

给定 prompt $x$ 与示范回答 $y=(y_1,\ldots,y_T)$，SFT 用教师强制（teacher forcing）最小化回答 token 的负对数似然：

$$
\mathcal{L}_{\mathrm{SFT}}(\theta)
=-\frac{1}{\sum_t m_t}
\sum_{t=1}^{T}
m_t\log\pi_\theta(y_t\mid x,y_{<t}),
$$

其中 $m_t\in\{0,1\}$ 是 loss mask。这个目标学习的是“模仿数据中的示范回答”，不会自动比较多个合理答案之间的偏好，也不会直接优化部署时整段自由生成的长期结果；这正是后续偏好学习存在的原因。

```text
<|im_start|>system
你是一个有帮助的AI助手。
<|im_end|>
<|im_start|>user
请解释什么是梯度下降？
<|im_end|>
<|im_start|>assistant
梯度下降是一种优化算法，通过计算损失函数对参数的梯度...
<|im_end|>
```

### 3\.2 Label Masking：只对回复计算 Loss

#### Label Masking 与 Label Shift 的边界

Label Masking 只说明哪些位置参与 loss；Label Shift 说明自回归语言模型实际预测的是“下一 token”。

|概念|含义|工程注意|
|---|---|---|
|Label Masking|system / user / padding 位置通常置为 `-100`|不在这些位置直接优化 next-token loss|
|Label Shift|`logits[t]` 用来预测 `labels[t+1]`|需要确认 tokenizer 模板与 shift 后标签对齐|
|EOS 处理|assistant 回复结束 token 是否参与 loss 需要显式约定|参与 EOS loss 有助于模型学习停止生成|
|多轮对话|通常只对 assistant 回复段计算 loss|需要保留 role 边界，避免跨轮错位|

更准确地说，mask 掉 prompt 位置只是不在这些位置直接计算“预测下一个 prompt token”的交叉熵。prompt 仍参与前向传播，回答位置的损失也会通过注意力路径反传到处理 prompt 表示的可训练参数。因此它不是“模型完全不学习 prompt”，而是“训练信号只从目标回答位置发出”。

```text
Token序列：  [system] 你是... [user] 请解释... [assistant] 梯度下降是...
Label：      [-100]  [-100]  [-100] [-100]     [梯度]     [下降]  [是]...
              ↑ 不计算 loss ↑                      ↑ 只有这部分计算 loss ↑

目的：让损失聚焦于 assistant 应该如何回答，同时保留 prompt 作为条件上下文
```

工程上最容易出错的不是公式，而是序列边界：

- 训练与推理必须使用一致的 chat template、角色 token 和 EOS 约定。
- packing 多条样本时必须阻断跨样本注意力或正确构造边界，不能让后一条样本读取前一条答案。
- 多轮数据要明确是训练所有 assistant turn，还是只训练最后一个 turn；两者的数据权重不同。
- 若示范回答质量不稳定，SFT 会直接模仿其中的事实错误、冗长和拒答偏差，后续 RLHF 不能保证完全纠正。

### 3\.3 一次完整的 SFT 训练迭代

```text
Step 1【前向传播】
  整个序列输入模型（prompt + response）
  每个位置输出下一个 token 的概率分布 logits

Step 2【Loss 计算】
  只对 assistant 回复部分计算交叉熵：
  Loss = masked_mean(CrossEntropy(shifted_logits, shifted_labels))
  system/user 部分的 label=-100，被 ignore_index 自动跳过

Step 3【反向传播】
  Loss 对所有可训练参数求梯度
  如果用 LoRA：只有 A, B 矩阵有梯度，W 冻结
  如果全量微调：所有参数都有梯度

Step 4【参数更新】
  AdamW 优化器更新参数
  如果用 LoRA：只更新 A, B（参数量极小）
```

---

## 四、阶段二：奖励模型（Reward Model）训练详解

### 4\.1 数据标注流程

#### 偏好数据质量控制

偏好标注质量会直接决定 Reward Model 的上限。除 chosen / rejected 本身外，还需要关注标注一致性、样本覆盖和偏置来源。

|环节|建议补充|目的|
|---|---|---|
|标注一致性|双人复核、冲突仲裁、任务分层|降低偏好噪声|
|样本构造|去重、难例采样、领域覆盖|避免 RM 只记住重复模式|
|偏置检查|长度偏置、position bias、格式偏置|防止 RM 学到伪相关信号|
|验证集|OOD 验证集、跨任务验证集|检查 RM 泛化能力|
|质量评审|chosen 胜率、人工复核、错误案例归因|发现 reward hacking 风险|

1. **Prompt 采集：**从真实用户问题中收集 prompts

2. **多样性生成：**对每个 prompt，从一个或多个策略 checkpoint 采样多个回答；温度、采样数 $K$ 和模型混合比例是数据设计变量，而不是固定算法常数

3. **人工排序：**标注者对 K 个回答按质量做相对排序

4. **转化为 Pairwise：**从排序中提取 $(winner, loser)$ 对。可以取全部组合，也可以只采样部分难对；同一 prompt 的多组 pair 高度相关，训练/评估切分必须按 prompt 分组，不能把同一组候选拆到训练集和验证集造成泄漏

若标注允许“并列”或“无法判断”，应保留不确定性，而不是强迫每组都产生 winner。对同一排序展开出的所有 pair 直接等权使用，会让候选数多的 prompt 获得更大训练权重；可以按 prompt 归一化、对子对采样或使用 listwise 目标控制权重。

```text
Prompt: "如何学习编程？"
人工排序 4 个回答：B > D > A > C

转化为 C(4,2)=6 对：
  (B, D), (B, A), (B, C), (D, A), (D, C), (A, C)

每条训练数据：
  {"prompt": "...", "chosen": "B的回答", "rejected": "D的回答"}
```

#### 偏好数据中的结构性偏差

偏好标签不是脱离采集过程的“客观真值”。候选由谁生成、以什么顺序展示、标注者看到什么 rubric，以及哪些 prompt 被送去标注，都会改变最终的数据分布。

|偏差来源|典型表现|识别与缓解|
|---|---|---|
|Prompt 选择偏差|训练集集中在高频、简单或已知领域|按目标流量分层抽样，单列长尾、跨语言和高风险集合|
|候选策略偏差|两个回答都来自同类模型，错误模式高度相似|混合不同 checkpoint、采样参数和模型族；记录生成策略版本|
|位置偏差|标注者或 LLM Judge 更常选择先展示的一项|随机交换 A/B；同一对做反序复核并统计翻转率|
|长度与形式偏差|更长、有标题或更自信的回答即使事实性相同也更易胜出|构造近似等长对；按长度差分桶报告准确率；加入“简洁但完整”的难例|
|标注者异质性|不同语言、文化背景或风险容忍度给出系统性不同判断|保留匿名 labeler/rubric 元数据，分群报告一致性，必要时建模多种偏好而非只取多数票|
|策略自偏好|评审模型偏好与自己风格或措辞相近的回答|使用异构评审、人工校准集和隐藏来源信息，不能只用单一 Judge 自证|

这里有两个容易被忽略的数据边界：

1. **切分单位必须是 prompt 或会话，而不是 pair。**同一 prompt 展开的多对比较共享候选回答，把它们拆到训练集和验证集会造成严重泄漏。
2. **长度控制不是把所有 reward 除以 token 数。**标量 RM 的分数并不是 token reward 的简单求和；机械地除以长度会重新定义目标。更稳妥的做法是控制候选生成、构造等长对，并在评估时把长度差作为分层变量或报告长度控制后的指标。

### 4\.2 网络结构修改

```text
原始 LLM：Input → Transformer Layers → LM Head (vocab_size) → 概率
                                         ↑ 删除

Reward Model：Input → Transformer Layers → Scalar Head → 标量分数
                                            ↑ 新增：Linear(d_model, 1)

具体：取每条序列的 EOS 或最后一个有效 response token 的隐藏状态 h_last
     reward = W_head × h_last + b_head → 输出一个标量
```

如果 batch 使用右侧 padding，不能直接取张量最后一列，因为它可能是 PAD；必须按 attention mask 定位有效结束位置。也有实现对 response token 做 pooling，Head 形式不是标准强制项。

### 4\.3 损失函数：Bradley\-Terry 模型

给定同一 prompt $x$ 下的偏好 $y_w\succ y_l$，Bradley-Terry 模型假设：

$$
P_\phi(y_w\succ y_l\mid x)
=\sigma\!\left(r_\phi(x,y_w)-r_\phi(x,y_l)\right).
$$

对应的负对数似然为：

$$
\mathcal{L}_{\mathrm{RM}}
=-\log\sigma(r_w-r_l).
$$

它鼓励 winner 的分数高于 loser；分差越大，预测偏好概率越接近 $1$，损失越小。若标注有软概率 $q\in[0,1]$ 或平局信息，可以把硬标签交叉熵扩展为软标签交叉熵，避免把不确定样本当作绝对偏好。

Bradley-Terry 损失只依赖分差 $r_w-r_l$，所以给同一 prompt 的所有分数加任意 $c(x)$ 都不会改变损失；奖励的绝对零点没有被偏好对识别。分数尺度也受正则化、数据可分性和模型参数化影响。将 RM 接入 PPO 前通常要在固定 prompt/response 分布上做中心化、缩放或 reward clipping，并持续监控分布漂移；不能把不同 RM checkpoint 的原始分数直接当成同一量纲的“真实效用”。

Bradley-Terry 还隐含了一个重要假设：同一 prompt 下的偏好可以由**单一标量效用差**表示。标准模型采用固定 logistic 噪声尺度；若显式写温度 $\tau$，

$$
P(y_w\succ y_l\mid x)
=\sigma\!\left(\frac{r_w-r_l}{\tau}\right),
$$

那么同时缩放 reward 与 $\tau$ 不改变偏好概率。实践中通常固定 $\tau=1$ 作为尺度约定。即便如此，以下情况仍不能靠一个 BT 标量完整表达：

- 偏好随标注者群体或任务 rubric 改变；
- 多目标之间存在不可统一排序的取舍；
- 比较关系出现上下文依赖或循环偏好；
- 标签有平局、犹豫或系统性噪声，却被强制二值化。

因此，pairwise accuracy 衡量的是模型能否复现某个聚合标签分布，不证明它恢复了唯一的“真实人类效用”。如果产品需要支持不同用户群体，应考虑分群 RM、条件化 rubric 或多目标 Head，而不是默认多数偏好能代表所有人。

### 4\.4 一次完整的 RM 训练迭代（含数值）

```text
输入：(prompt + chosen, prompt + rejected)

Step 1【前向传播×2】
  r_w = RM(prompt + chosen)    = 3.2   （winner 分数）
  r_l = RM(prompt + rejected)  = 1.8   （loser 分数）

Step 2【计算 Loss】
  Loss = -log(sigmoid(3.2 - 1.8))
       = -log(sigmoid(1.4))
       = -log(0.802) = 0.220

Step 3【反向传播】
  dLoss/d(r_w) = sigmoid(1.4) - 1 = -0.198  （鼓励 winner 分更高）
  dLoss/d(r_l) = +0.198                       （鼓励 loser 分更低）

  梯度回传路径：
  → Scalar Head（更新 W_head）
  → Transformer Layers（更新 LoRA 的 A, B 参数）
  → 更早层与输入表示（梯度继续传播，但冻结参数不更新）

Step 4【参数更新】
  仅更新 Scalar Head + LoRA 参数，原始权重冻结
```

这个数值例子采用“LoRA + 可训练 Reward Head”的配置；全量微调 RM 时，backbone 与 embedding 也可更新。无论采用哪种配置，验证 RM 都应至少包含 pairwise accuracy、不同长度区间、跨域 prompt 和对抗样本。训练集准确率很高不代表 RM 能可靠评价当前策略新生成的回答，这正是 PPO 过程中会遇到的分布外问题。

### 4\.5 奖励模型的校准、评估与上线边界

RM 的离线验证至少要回答三个不同问题：它是否排对已知偏好、分数是否能解释为置信度，以及它能否评价训练后策略产生的新回答。三者不能只靠一个 accuracy 概括。

|问题|建议指标或切片|注意事项|
|---|---|---|
|排序能力|pairwise accuracy、log loss、按领域/难度/安全类别分层的准确率|总体准确率可能被大量简单样本掩盖|
|概率校准|Brier score、reliability diagram、平局与低一致性样本上的置信度|原始 reward 分数不是概率；先经过分差与 sigmoid 才得到 BT 概率|
|伪相关|按 A/B 位置、长度差、格式、拒答/非拒答分桶的胜率|交换位置后预测翻转，说明模型利用了展示顺序|
|分布外泛化|不同模型族、较新 checkpoint、对抗改写和真实流量 holdout|只在生成训练偏好数据的旧策略上验证会高估能力|
|不确定性|模型集成分歧、重复标注分歧、低置信样本覆盖率|可用于把高风险或 OOD 样本转交人工，而不是把不确定性抹成一个分数|

置信区间应按 **prompt 聚类重采样**，而不是把同一 prompt 展开的 pair 当作独立样本；后者会虚增有效样本量。RM 上线参与 PPO 后，还要定期从**当前策略**采样并重新做人类盲评，因为策略会主动搜索 RM 的薄弱区域。

更关键的是区分“代理 reward”与目标质量。Reward-model overoptimization 的实验表明，持续优化代理分数时，独立的 gold/人工质量可能先上升、随后停滞甚至下降。因此 checkpoint 选择不能只看 RM score，至少要同时设置 KL/长度预算、独立 Judge 或人工胜率、安全回归集和停止条件。KL 能限制搜索半径，但不能证明该半径内不存在奖励漏洞。

---

## 五、阶段三：PPO 强化学习——完整详解

经典 PPO-RLHF 需要四种**逻辑角色**协同：Policy、Reference、Reward、Value。它们不一定是四份完全独立的全量权重；Actor/Value 可共享 backbone，冻结模型可共享底座并切换 adapter，具体取决于实现。

### 5\.1 四模型体系

|模型|类比|职责|更新？|计算梯度？|
|---|---|---|---|---|
|**Actor**|演员|生成回答，参数在训练中更新|是（LoRA 或全量等）|是|
|**Reference**|档案馆|SFT 模型冻结副本，提供 KL 基准|否|否|
|**Reward**|裁判|对完整回答打分|否|否|
|**Value**|解说员|预测每步的未来期望收益|是|是|

把文本生成写成有限时域 MDP 时：

- 状态 $s_t=(x,y_{<t})$ 是 prompt 与已生成前缀；
- 动作 $a_t=y_t$ 是下一个 token；
- 策略 $\pi_\theta(a_t\mid s_t)$ 是词表上的条件分布；
- 一条轨迹 $\tau=(y_1,\ldots,y_T)$ 在 EOS 或长度上限终止；
- RM 通常只在终局给序列级分数，KL 惩罚则可按 token 分摊。

Value 模型估计的是当前策略下的期望剩余回报 $V_\psi(s_t)$，不是回答“客观质量”的第二个 Reward Model。Reference 也不是旧策略：Reference 在整个 RL 阶段通常固定，而 old policy 是每批 rollout 时行为策略的快照，会随训练迭代更新。

### 5\.2 Phase 1：Rollout（生成采样）

Rollout 是 PPO 的"探索"阶段——Actor 生成文本，同时记录所有中间信息。

```text
输入：一批 prompts（batch_size=64）

对每个 prompt 自回归生成：
for t in range(max_new_tokens):
    logits_t = Actor(prompt + generated_so_far)
    probs_t = Softmax(logits_t / temperature)    # temperature 是 rollout 配置
    token_t = sample(probs_t)                     # 概率采样（非 greedy）
    log_prob_old[t] = log(probs_t[token_t])       # ← 记录！后续算 ratio 用
    generated_so_far.append(token_t)

生成完毕后（4 个模型各司其职）：
  ref_log_probs = Reference(prompt + response)    # Reference 算概率
  reward = RewardModel(prompt + response)          # Reward 打总分
  values = ValueModel(prompt + response)           # Value 逐 token 预测
```

**Old Policy 概念：**Rollout 时 Actor 的参数快照就是旧策略 $\pi_{\mathrm{old}}$。之后 PPO 更新会改变 Actor，得到当前策略 $\pi_\theta$。对实际采样 token 的重要性比率为：

$$
\rho_t(\theta)
=\frac{\pi_\theta(a_t\mid s_t)}
{\pi_{\mathrm{old}}(a_t\mid s_t)}
=\exp\!\left(
\log\pi_\theta(a_t\mid s_t)
-\log\pi_{\mathrm{old}}(a_t\mid s_t)
\right).
$$

#### 5\.2\.1 概率比来自哪里：重要性采样与局部代理目标

概率比不是为了裁剪而临时定义的量。它首先来自重要性采样：rollout 数据由旧策略生成，但优化目标希望描述当前策略下动作的期望效果。固定一个状态 $s$，并把 rollout 阶段计算出的旧策略优势 $\hat A^{\mathrm{old}}(s,a)$ 视为常数，则

$$
\begin{aligned}
\mathbb{E}_{a\sim\pi_\theta(\cdot\mid s)}
\left[\hat A^{\mathrm{old}}(s,a)\right]
&=\sum_a\pi_\theta(a\mid s)\hat A^{\mathrm{old}}(s,a)\\
&=\sum_a\pi_{\mathrm{old}}(a\mid s)
\frac{\pi_\theta(a\mid s)}{\pi_{\mathrm{old}}(a\mid s)}
\hat A^{\mathrm{old}}(s,a)\\
&=\mathbb{E}_{a\sim\pi_{\mathrm{old}}(\cdot\mid s)}
\left[\rho_t(\theta)\hat A^{\mathrm{old}}(s,a)\right].
\end{aligned}
$$

因此，$\rho_t$ 同时有两层含义：

1. 它是在旧策略样本上估计当前策略动作期望的**重要性权重**；
2. 它度量当前策略相对“真正生成这批数据的策略”的累计似然变化，PPO 再利用这个量做裁剪。

这个变换只精确修正了**给定状态下的动作分布**。完整 MDP 中，当前策略与旧策略还会产生不同的状态访问分布 $d^{\pi_\theta}(s)$ 和 $d^{\pi_{\mathrm{old}}}(s)$；PPO/TRPO 的局部代理目标依赖“新旧策略保持接近”的近似，用旧状态分布代替新状态分布。因此，逐 token ratio 不会把任意旧 rollout 变成严格无偏的当前策略数据，PPO 仍是 on-policy/近 on-policy 方法。

若对整条轨迹做精确重要性采样，在环境动力学相同的前提下，轨迹似然比包含 token ratio 的乘积：

$$
\frac{P_\theta(\tau)}{P_{\mathrm{old}}(\tau)}
=\prod_{t=1}^{T}\rho_t.
$$

长序列上的乘积方差通常很大。PPO 使用逐状态、逐动作的局部 surrogate，再配合裁剪和频繁刷新 rollout，避免直接优化高方差的整轨迹比率。

#### 5\.2\.2 普通 Policy Gradient 与 PPO 梯度的关系

普通 on-policy Policy Gradient 的采样估计可以写成：

$$
\mathcal{L}_{\mathrm{PG}}(\theta)
=-\mathbb{E}_t\left[
\hat A_t\log\pi_\theta(a_t\mid s_t)
\right],
$$

$$
\nabla_\theta\mathcal{L}_{\mathrm{PG}}
=-\mathbb{E}_t\left[
\hat A_t\nabla_\theta\log\pi_\theta(a_t\mid s_t)
\right].
$$

暂时不考虑 clipping，PPO 的代理 loss 为

$$
\mathcal{L}_{\mathrm{PPO\text{-}unclip}}(\theta)
=-\mathbb{E}_t\left[\rho_t(\theta)\hat A_t\right].
$$

由于旧策略分母被冻结，

$$
\nabla_\theta\rho_t
=\rho_t\nabla_\theta\log\pi_\theta(a_t\mid s_t),
$$

所以

$$
\nabla_\theta\mathcal{L}_{\mathrm{PPO\text{-}unclip}}
=-\mathbb{E}_t\left[
\rho_t\hat A_t
\nabla_\theta\log\pi_\theta(a_t\mid s_t)
\right].
$$

在一批 PPO 优化刚开始时，$\pi_\theta=\pi_{\mathrm{old}}$，因此 $\rho_t=1$，PPO 第一次更新的 policy 梯度恰好退化为普通 Policy Gradient。这里必须区分：

$$
\rho_t(\theta_{\mathrm{old}})=1
\qquad\text{与}\qquad
\rho_t(\theta)\equiv1.
$$

前者只是函数在当前参数点的**数值**等于 $1$，其导数仍为 $\nabla_\theta\log\pi_\theta$；后者是把 ratio 写死成常数，导数为 $0$，此时 $\min(\rho_t\hat A_t,\operatorname{clip}(\rho_t)\hat A_t)$ 对 Actor 参数不再提供 policy 梯度。类似地，$f(x)=x/2$ 在 $x=2$ 时函数值为 $1$，但导数仍为 $1/2$。

如果不用 ratio，改回 $-\hat A_t\log\pi_\theta$，仍然可以产生普通 PG 梯度；但在同一批旧数据上反复训练多个 epoch 时，不再有重要性权重和 PPO clipping，数据会随当前策略变化而逐渐变旧。因此这种做法不等价于 PPO，普通 on-policy PG 通常在一次或少量更新后重新 rollout。

#### 5\.2\.3 一批 rollout 内的策略生命周期

|策略|生命周期|是否更新|主要用途|
|---|---|---|---|
|$\pi_\theta$（current policy）|每个 optimizer step 后都可能改变|是|重新计算 new logprob，接收 policy gradient|
|$\pi_{\mathrm{old}}$（behavior/rollout policy）|在当前 rollout batch 的所有 epoch 中固定；下一轮 rollout 时刷新|否|提供 PPO ratio 分母；工程上缓存 old logprob 通常就足够|
|$\pi_{\mathrm{ref}}$（reference policy）|经典 PPO-RLHF 中通常跨许多 rollout 迭代长期固定|否|提供相对 SFT 策略的 KL 基准|

一个 PPO epoch 是对固定 rollout buffer 的一次完整遍历，不是一次反向传播。每个 mini-batch 都会 forward、backward 并执行 optimizer step；下一个 mini-batch 和下一个 epoch 都从已经更新后的当前参数继续训练，但 old logprob 始终锚定本批数据的 rollout 策略：

```text
theta_0 rollout，缓存 old_logprob
    │
    ├─ epoch 1: theta_0 → step → theta_1 → step → theta_2 → ...
    ├─ epoch 2: theta_k → step → theta_{k+1} → ...
    └─ epoch K: 得到 theta_final
                         │
                         └─ 用 theta_final 重新 rollout，建立下一批 old_logprob
```

只有开始任何 optimizer step 之前，current 与 old 才完全相同。第一个 mini-batch 更新共享模型参数后，后续 mini-batch 即使还没有被单独训练，其 new logprob 和 ratio 也可能已经改变。

**分布一致性：**`old_log_prob` 必须对应实际采样的行为分布；更新时 `new_log_prob` 也必须定义在同一动作分布上。若 rollout 按温度缩放、top-k 或 top-p 截断后采样，却用另一套未变换 logits 记录概率，$\rho_t$ 就不再是正确的行为策略似然比。截断采样还会改变支持集；工程上应明确优化的是原始模型策略还是变换后的采样策略，并保持 old/new 口径一致。

#### Rollout 张量的 Shift 对齐

自回归模型在“前一个位置”的 logits 上预测当前 action。对 $s_t=(x,y_{<t})$ 与 $a_t=y_t$：

- `logprob[t]` 应来自输入前缀 $s_t$ 的 logits 对标签 $y_t$ 的概率；
- $V(s_t)$ 是发出 $y_t$ **之前**状态的价值；
- $r_t$ 是执行 $y_t$ 后得到的奖励；
- GAE 的下一个状态值是 $V(s_{t+1})$。

若把 `prompt + response` 一次送入模型，response 第一个 token 的 logprob 通常来自最后一个 prompt token 位置的 logits，而不是 response 第一个 token 位置的 logits。实际切片还受 BOS、chat template 和 padding side 影响，必须用一条可手算的小样本核对 token id、logits、value、reward 与 mask，避免整体错一位却仍能正常下降 loss。

### 5\.3 Phase 2：计算 Reward 和 Advantage

#### 5\.3\.1 即时奖励与 KL 惩罚

PPO-RLHF 常把序列奖励写成：

$$
R(x,y)
=r_\phi(x,y)
-\beta\sum_{t=1}^{T}
\left[
\log\pi_{\mathrm{old}}(y_t\mid x,y_{<t})
-\log\pi_{\mathrm{ref}}(y_t\mid x,y_{<t})
\right].
$$

中括号是采样 token 上的 log-probability difference。当 token 确实来自 $\pi_{\mathrm{old}}$ 时，对该随机量取旧策略期望可得到前向 KL：

$$
\mathbb{E}_{a_t\sim\pi_{\mathrm{old}}}
\left[
\log\frac{\pi_{\mathrm{old}}(a_t\mid s_t)}
{\pi_{\mathrm{ref}}(a_t\mid s_t)}
\right]
=D_{\mathrm{KL}}\!\left(
\pi_{\mathrm{old}}(\cdot\mid s_t)
\|\,\pi_{\mathrm{ref}}(\cdot\mid s_t)
\right).
$$

单个采样值可以为负，只有上述期望保证非负；它不是已经枚举全词表得到的精确 KL。

|口径|说明|工程建议|
|---|---|---|
|完整分布 KL|对 actor 策略与 reference 策略的全词表分布差异求期望|适合严谨分析，但计算成本更高|
|采样估计|在 Actor 采样 token 上计算 actor logprob 与 reference logprob 的差值|常用于 token 级 reward shaping；平均后才近似前向 KL|
|自适应 KL|根据平均 KL 与目标 KL 的偏差动态调节 beta|KL 超标提高 beta，KL 过低降低 beta|

当代码里写 `kl = logprob_actor - logprob_ref` 时，建议标注为 sampled KL penalty 或 logprob difference，避免误认为已经计算完整分布 KL。

|时机|常见 shaped reward|含义|
|---|---|---|
|中间每一步 $t<T$|$-\beta k_t$|把偏离 Reference 的代价分摊到 token|
|最后一步 $T$|$r_\phi(x,y)-\beta k_T$|终局 RM 分数加最后一个 token 的 KL 代价|

其中 $k_t=\log\pi_{\mathrm{old}}(a_t\mid s_t)-\log\pi_{\mathrm{ref}}(a_t\mid s_t)$。$\beta$ 的含义是：

- $\beta$ 较小：对偏离 Reference 的约束较弱；
- $\beta$ 较大：约束较强，也可能压制有效策略改进；
- 实际尺度取决于 RM 校准、序列聚合/归一化和 KL 估计方式，不存在跨项目通用区间；
- 可以围绕目标 KL 使用平滑的自适应控制器，“超阈值就翻倍”只适合作为易懂伪代码，不是唯一算法。

序列级目标中的 KL 是 token log-ratio 的**和**。因此在其他条件相同时，较长回答会累积更多 KL 代价；若改成“每 token 平均 KL”，就改变了正则项对长度的约束，而不只是改变日志展示方式。RM 的终局标量、每 token KL 和可选的长度/格式奖励必须分开记录，避免把某种归一化误当成算法默认值。

#### 5\.3\.2 Return（总收益）

从第 $t$ 个 token 开始的折扣回报为：

$$
G_t=\sum_{l=0}^{T-t}\gamma^l r_{t+l}.
$$

文本生成是有限时域 episodic 任务，很多 RLHF 实现取 $\gamma=1$；也可以小于 $1$。该选择会改变早期 token 获得终局奖励的权重，必须与长度分布和 EOS 处理一起验证。终止状态取 $V(s_{T+1})=0$；若回答因长度上限被截断，要明确它是终止还是 bootstrap，否则 return 会产生系统偏差。

#### 5\.3\.3 优势函数（Advantage）

最直接的估计是 $A_t=G_t-V_\psi(s_t)$。实践中常用 GAE（Generalized Advantage Estimation）在偏差与方差之间折中：

$$
\delta_t
=r_t+\gamma V_\psi(s_{t+1})-V_\psi(s_t),
$$

$$
\hat A_t^{\mathrm{GAE}(\gamma,\lambda)}
=\sum_{l=0}^{T-t}(\gamma\lambda)^l\delta_{t+l}.
$$

$\lambda$ 控制 bias-variance 取舍；$0.95$ 是常见示例而非固定要求。$A_t>0$ 表示该动作的结果优于 baseline 预期，$A_t<0$ 表示差于预期。实现中通常只在 response mask 上计算并对有效 token 的 advantage 做标准化；prompt、padding 与越过 EOS 的位置不能混入统计量。

#### 5\.3\.4 聚合口径与长度权重

从 token reward 得到 advantage 后，最终还要决定“哪些 token 或回答在一个 batch 中占多大权重”。下面几种写法并不等价：

|聚合方式|隐含权重|主要边界|
|---|---|---|
|所有有效 token 一起取均值|每个 token 等权，长回答包含更多 token，因而总影响更大|实现简单，但回答长度分布会改变梯度构成|
|先对每条回答取 token 均值，再对回答取均值|每条回答近似等权，短回答的单个 token 权重更大|更接近 sequence-balanced，但会改变原始 token-level 目标|
|按 prompt 先聚合多个回答|每个 prompt 近似等权|适合每个 prompt 采样数不一致的场景|

Advantage whitening 应在 rollout batch 上一次性计算并在 PPO 多个 epoch 中冻结，不能在每个 mini-batch 内反复归一化，否则同一轨迹的权重会随分桶方式变化。类似地，RM score 的中心化/缩放应使用固定校准集或缓慢更新的运行统计；直接按当前小 batch 标准化，会让同一回答的 shaped reward 依赖碰巧与它同批的样本。

“长度归一化”没有跨算法统一答案。选择聚合口径时要同时报告响应长度、EOS/截断率、每序列 KL 和每 token KL，并用长度匹配的盲评判断质量提升是否只是更冗长。

### 5\.4 Phase 3：PPO 参数更新

#### 5\.4\.1 Actor 的 Loss 与反向传播

```text
对每个 token 位置 t：

  # 1. 重新用当前 Actor 算概率（因为参数可能已更新）
  pi_new_t = Actor_current(token_t | context_t)
  ratio_t = pi_new_t / pi_old_t

  # 2. PPO-Clip Loss
  surr1 = ratio_t × Advantage_t
  surr2 = clip(ratio_t, 1-eps, 1+eps) × Advantage_t
  loss_t = -min(surr1, surr2)

  # 3. 总 Loss = mean(所有 token 的 loss_t)
```

```text
Loss_actor
    │
    ▼ dLoss_surrogate/d(log_pi_new) = -m_t × ratio_t × Advantage_t
    │  m_t=0 表示该样本进入裁剪平坦区，否则 m_t=1
    │
    ▼ 穿过 Softmax → dLoss/dLogits
    │
    ▼ 穿过 LM Head → 更新 LM Head 的 LoRA (如有)
    │
    ▼ 穿过每一层 Transformer
    │  ├── 更新该层 Attention 的 LoRA (A_q, B_q, A_k, B_k, ...)
    │  └── 更新该层 FFN 的 LoRA (如有)
    │  原始 W 全部冻结！
    │
    ▼ 直到 Embedding 层

关键区别 vs SFT：
- SFT：梯度方向 = "朝标准答案靠拢"（固定目标）
- PPO：梯度方向 = "Advantage加权的策略梯度"（动态信号）
  正 Advantage → 增大该 token 概率
  负 Advantage → 减小该 token 概率
```

上面的 `min` 是逐 token 的 clipped surrogate。训练时可以加入熵奖励以维持探索，并对有效 response token 做 masked mean：

$$
\mathcal{L}_{\mathrm{policy}}
=-\mathbb{E}_t\!\left[
\min\!\left(
\rho_t\hat A_t,\,
\operatorname{clip}(\rho_t,1-\epsilon,1+\epsilon)\hat A_t
\right)
\right]
-c_{\mathrm{ent}}\mathbb{E}_t[\mathcal{H}(\pi_\theta(\cdot\mid s_t))].
$$

若只看 policy surrogate，并把 $\hat A_t$ 视为 stop-gradient，则远离裁剪边界时可把单样本梯度写成：

$$
\nabla_\theta\mathcal{L}_{\mathrm{clip},t}
=-m_t\rho_t\hat A_t
\nabla_\theta\log\pi_\theta(a_t\mid s_t),
$$

其中

$$
m_t=
\begin{cases}
0,&\hat A_t>0\ \text{且}\ \rho_t>1+\epsilon,\\
0,&\hat A_t<0\ \text{且}\ \rho_t<1-\epsilon,\\
1,&\text{其他情况}.
\end{cases}
$$

因此，裁剪不是在所有区域都把梯度“缩小一点”：当样本已经沿优势方向越过边界时，该样本的 clipped policy surrogate 进入平坦区，梯度为 $0$；在不利方向越界时仍保留未裁剪梯度，把策略推回。恰好位于边界时目标不可微，自动微分框架会按所选分支给出一个次梯度。熵项、显式 KL、辅助 LM loss 和其他样本仍可能贡献额外梯度。

PPO-Clip 抑制部分过大的概率更新，但它不是“每个 token 的概率最多变化 $\epsilon$”的硬约束，也不保证整条序列 KL 一定落在某个区间。因此仍要监控 policy/reference KL、old/new approximate KL、clip fraction 和 early stopping。

#### 5\.4\.2 Value Model 的 Loss 与反向传播

```text
Loss_value = 0.5 × mean((V_pred_t - Return_t)^2)    # MSE

反向传播：
  dLoss/dV_pred = V_pred - Return

  例：V_pred=5.0, Return=7.2
  dLoss/dV = 5.0-7.2 = -2.2 → 梯度方向是增大预测值

梯度路径：
  → Scalar Head
  → Transformer Layers（全量微调或 LoRA）
  → Embedding
```

这里的 `Return`/GAE target 在一次 rollout 的 PPO 多 epoch 更新中应视为固定目标（stop-gradient），不能让梯度穿回 RM、Reference 或旧 Value 预测。一些实现还会像 policy 一样裁剪 Value 的变化，减少多个 epoch 中 baseline 剧烈漂移；是否裁剪是实现选择，不是 PPO-RLHF 定义的必需组件。

#### 5\.4\.3 KL 约束如何影响梯度

必须区分两种实现：

1. **rollout reward shaping：**用 $\pi_{\mathrm{old}}$ 与 $\pi_{\mathrm{ref}}$ 算出 $k_t$，构造 reward/advantage，然后在本批 PPO 多 epoch 中将它们冻结。KL 此时通过 advantage 改变采样动作的权重，**不会**在更新阶段沿 $k_t$ 单独建立一条当前策略梯度路径。
2. **显式 KL 正则：**更新时重新用当前 $\pi_\theta$ 计算 KL 或其估计，并把它直接加进 loss。此时正则项会对 Actor 产生直接梯度。

“KL 像弹簧拉回策略”只是在分布期望层面的直觉，不应解释为每个采样 token 都会被逐点拉回：单个 log-ratio 可以为负，frozen shaping 又与新的 policy 参数断开。真正需要验证的是批次/数据分布上的平均 KL 与生成质量，而不是某一个 token 的符号。

#### 5\.4\.4 InstructGPT 的 PPO-ptx 不是纯 PPO loss

InstructGPT 还实验了把预训练语料上的语言模型梯度混入 PPO，称为 PPO-ptx，用来缓解部分公开 NLP 任务上的能力回退。其概念目标为：

$$
\mathbb{E}_{(x,y)\sim\mathcal{D}_{\pi_\theta}}
\left[
r_\phi(x,y)
-\beta\log\frac{\pi_\theta(y\mid x)}
{\pi_{\mathrm{SFT}}(y\mid x)}
\right]
+\gamma\,
\mathbb{E}_{z\sim\mathcal{D}_{\mathrm{pretrain}}}
\left[\log\pi_\theta(z)\right].
$$

$\gamma=0$ 时退化为不混合预训练梯度的 PPO 版本。这个 auxiliary LM objective 是 InstructGPT 的具体工程选择，不是 PPO 定义的一部分；其他系统也可能混入 SFT loss、蒸馏 loss 或安全约束。

#### 5\.4\.5 四模型显存分工

```text
显存占用分析（以 7B 模型 FP16/BF16 为例，具体数值取决于 batch、seq_len、分片和量化策略）：

Actor：    冻结底座权重 + 可训练 LoRA/Head + 反向传播激活值缓存
Value：    可由 SFT/Actor 或 RM backbone 初始化 + Value Head；
           可选择只训练 Value Head、训练 Value LoRA，或资源充足时训练更多参数
Reference：SFT policy 的冻结副本，只推理，用于提供 KL 基准
Reward：   已训练好的 RM，包含 RM backbone/adapter + Reward Head，只推理打分

共享口径：
- PEFT 场景下，Actor / Reference / Reward / Value 可以共享同一份 frozen base weights 来省显存。
- 但它们的 adapter 与 head 通常不同：Reference 使用 SFT adapter，Reward 使用 RM adapter + Reward Head，Value 使用 Value Head 或 Value adapter。
- 因此不能简单写成“Reference 和 Reward 只有 Head 不同”；如果 RM 训练过 LoRA/backbone adapter，二者不仅 head 不同。
```

### 5\.5 PPO 完整训练循环（伪代码）

#### PPO 训练监控指标

RLHF 是否成功不能只看 reward 分数上涨，必须同时观察 KL、熵、长度、人工胜率和安全指标。

|指标|关注点|异常信号|
|---|---|---|
|reward mean / median|整体奖励是否提升|奖励升高但人工质量下降，可能 reward hacking|
|policy/reference KL|Actor 相对长期 Reference 的偏离|持续暴涨可能说明 KL 正则过弱、奖励尺度异常或策略跑偏|
|old/new approx\_kl|当前 Actor 相对本批 rollout old policy 的更新距离|过高说明本轮 PPO optimizer step 过大，可用于 early stopping|
|entropy|生成分布是否过早塌缩|熵快速下降会导致回答单一化|
|clip fraction|PPO 更新是否频繁触发裁剪|过高说明更新步长过大或 advantage 异常|
|value loss / explained variance|Value baseline 是否可靠|Value 估计不准会放大 advantage 方差|
|response length|回答长度分布是否漂移|异常变长可能是长度偏置或注水|

```text
for iteration in range(num_iterations):

    # ═══ Phase 1: Rollout ═══
    prompts = sample_batch(dataset, batch_size=64)
    with torch.no_grad():
        responses, old_log_probs = Actor.generate_and_log_probs(
            prompts, temperature=1.0, do_sample=True
        )  # 此伪代码直接从 raw policy 采样；old_log_probs 必须对应实际行为分布
        ref_log_probs = Reference.log_probs(prompts + responses)
        old_values = ValueModel.predict(prompts + responses)
        rm_scores = RewardModel.score(prompts + responses)
        rewards = normalize_with_fixed_running_stats(rm_scores)

    # ═══ Phase 2: 计算 Reward 和 Advantage ═══
    for t in valid_response_positions:
        kl_t = old_log_probs[t] - ref_log_probs[t]
        if t == last_token:
            r[t] = rewards - beta * kl_t
        else:
            r[t] = -beta * kl_t

    raw_advantages = compute_gae(
        r, old_values, gamma=1.0, lam=0.95,
        response_mask=response_mask, terminal_mask=eos_mask
    )
    returns = (raw_advantages + old_values).detach()
    advantages = masked_normalize(raw_advantages, response_mask).detach()

    # ═══ Phase 3: PPO 更新（多 epoch × mini-batch）═══
    for epoch in range(ppo_epochs):        # 通常 1~4
        for mini_batch in shuffle_split(data, size=16):

            # Actor 更新
            new_log_probs = Actor.log_probs(mini_batch.seqs)
            ratio = exp(new_log_probs - mini_batch.old_log_probs)
            surr1 = ratio * mini_batch.advantages
            surr2 = clip(ratio, 1-0.2, 1+0.2) * mini_batch.advantages
            actor_loss = -masked_mean(min(surr1, surr2), response_mask)
            actor_loss.backward()
            optimizer_actor.step()
            optimizer_actor.zero_grad()

            # Value 更新
            new_values = ValueModel.predict(mini_batch.seqs)
            value_loss = 0.5 * masked_mean(
                (new_values - mini_batch.returns)^2, response_mask
            )
            value_loss.backward()
            optimizer_value.step()
            optimizer_value.zero_grad()

    # ═══ Phase 4: 自适应 KL 控制 ═══
    rollout_kl_mean = masked_mean(old_log_probs - ref_log_probs, response_mask)
    beta = kl_controller.update(beta, rollout_kl_mean, target_kl)
```

这段代码只展示数据依赖关系，不是可直接运行的框架实现。实际系统还必须处理变长序列、EOS 与截断、分布式 padding、reward whitening 的统计冻结、梯度累积、mixed precision、异常样本过滤和 old/new 模型同步。尤其不要用当前 mini-batch 最后一次更新后的 `new_log_probs` 代替整批 rollout KL。

### 5\.6 PPO 裁剪机制详解

#### PPO Clip 的梯度平坦区

PPO Clip 不是简单“把更新幅度限制在 1\.2”，而是在部分区域让目标函数进入平坦区，从而抑制继续推大概率的梯度。

|场景|是否触发裁剪|梯度直觉|
|---|---|---|
|Advantage \> 0 且 ratio \> 1 \+ eps|触发上界裁剪|继续增大该 token 概率不再提升目标，梯度趋近 0|
|Advantage \> 0 且 ratio 在区间内|不裁剪|鼓励增大该 token 概率|
|Advantage \< 0 且 ratio \< 1 \- eps|触发下界裁剪|继续减小该 token 概率不再提升目标，梯度趋近 0|
|Advantage \< 0 且 ratio 在区间内|不裁剪|鼓励减小该 token 概率|

公式口径：$\mathcal{L}_{\mathrm{clip}}=-\mathbb{E}[\min(\rho_t\hat A_t, \operatorname{clip}(\rho_t,1-\epsilon,1+\epsilon)\hat A_t)]$。本文统一用 $r_t$ 表示即时奖励、用 $\rho_t$ 表示 PPO 概率比。

```text
ratio = pi_new / pi_old

情况1：Advantage > 0（好动作，要鼓励）
  - ratio > 1.2 时被 clip → 防止过度增大概率（步子太大）
  - ratio < 0.8 时不会选择 clipped 分支（min 选 surr1，保留恢复方向的梯度）

情况2：Advantage < 0（坏动作，要惩罚）
  - ratio < 0.8 时被 clip → 防止过度减小概率
  - ratio > 1.2 时不会选择 clipped 分支（min 选 surr1，保留恢复方向的梯度）

核心效果：降低单批数据上过度更新的激励，减少策略剧烈漂移的风险；但它不是硬 trust region，也不保证策略不会失稳或性能单调提升
```

---

## 六、端到端数值示例：多个 Token 的 Actor/Critic 完整计算

上一节给出了公式，这里用一条包含中间 token 的短回答，把 **sampled KL → token reward → TD error → GAE → Actor loss → Critic loss** 全部串起来。为便于手算，把回答抽象为 3 个生成 token；真实 tokenizer 可能把这些文字切成更多 token，但计算方法完全相同。

### 6\.1 场景、状态与已知量

```text
Prompt: "如何保持健康？"
抽象后的 response token: ["运动", "和", "饮食"]
```

状态和动作依次为：

|时间|状态|动作|
|---|---|---|
|$t=1$|$s_1=(x)$|$a_1=$“运动”|
|$t=2$|$s_2=(x,$“运动”$)$|$a_2=$“和”|
|$t=3$|$s_3=(x,$“运动和”$)$|$a_3=$“饮食”|
|终止|$s_4=(x,$“运动和饮食”$)$|无下一动作，$V(s_4)=0$|

设超参数为：

$$
\beta=0.05,\qquad
\gamma=1,\qquad
\lambda=0.95,\qquad
\epsilon=0.2.
$$

Reward Model 只在终局给出：

$$
r_\phi(x,y)=8.2.
$$

rollout 时保存的旧策略概率、Reference 概率、某次 PPO 更新时的当前策略概率，以及 rollout Value 预测如下：

|$t$|token|$\pi_{\mathrm{old}}(a_t\mid s_t)$|$\pi_{\mathrm{ref}}(a_t\mid s_t)$|$\pi_\theta(a_t\mid s_t)$|$V_{\mathrm{old}}(s_t)$|
|---|---|---:|---:|---:|---:|
|1|“运动”|0.20|0.25|0.22|5.500|
|2|“和”|0.50|0.40|0.35|6.000|
|3|“饮食”|0.15|0.12|0.22|5.889|

为隔离 PPO 的核心计算，本例不加入熵奖励、显式 KL loss、Value clipping 或 PPO-ptx，并且不对 Advantage 做 whitening。实际训练通常会标准化 Actor 使用的 Advantage；Critic 的 target 则必须由未标准化的 raw GAE 构造。

### 6\.2 Step 1：计算每个 token 的 sampled KL 与即时奖励

每个实际采样 token 的 log-probability difference 为：

$$
k_t
=\log\pi_{\mathrm{old}}(a_t\mid s_t)
-\log\pi_{\mathrm{ref}}(a_t\mid s_t)
=\log\frac{\pi_{\mathrm{old}}(a_t\mid s_t)}
{\pi_{\mathrm{ref}}(a_t\mid s_t)}.
$$

中间 token 只有 KL shaping reward，最后一个 token 再加终局 RM 分数：

$$
r_t=
\begin{cases}
-\beta k_t,&t<T,\\
r_\phi(x,y)-\beta k_t,&t=T.
\end{cases}
$$

逐 token 计算：

|$t$|$k_t$|KL reward $-\beta k_t$|RM reward|最终即时奖励 $r_t$|
|---|---:|---:|---:|---:|
|1|$\log(0.20/0.25)=-0.223$|$+0.011$|0|$+0.011$|
|2|$\log(0.50/0.40)=+0.223$|$-0.011$|0|$-0.011$|
|3|$\log(0.15/0.12)=+0.223$|$-0.011$|8.200|$8.189$|

这里 $k_1<0$ 不代表完整 KL 为负；$k_t$ 是单个采样 token 的 log-ratio，可以正也可以负，只有对行为策略分布取期望后的完整 KL 保证非负。KL shaping 使用 rollout 的 $\pi_{\mathrm{old}}$，不是已经经过若干 optimizer step 的 $\pi_\theta$。

### 6\.3 Step 2：计算 TD error

GAE 先计算每一步的 TD error：

$$
\delta_t
=r_t+\gamma V_{\mathrm{old}}(s_{t+1})
-V_{\mathrm{old}}(s_t).
$$

由于 $\gamma=1$ 且终止状态 $V(s_4)=0$：

$$
\delta_1
=0.011+6.000-5.500
=0.511,
$$

$$
\delta_2
=-0.011+5.889-6.000
=-0.122,
$$

$$
\delta_3
=8.189+0-5.889
=2.300.
$$

注意：$\delta_2<0$ 只表示“执行第二个 token 后的一步观测”低于 $V(s_2)$ 的局部预期；它还没有包含后续终局奖励的完整影响。

### 6\.4 Step 3：用 GAE 把终局奖励传给中间 token

GAE 可以从后向前递推：

$$
\hat A_t
=\delta_t+\gamma\lambda\hat A_{t+1},
\qquad
\hat A_{T+1}=0.
$$

所以：

$$
\hat A_3
=\delta_3
=2.300,
$$

$$
\hat A_2
=\delta_2+0.95\hat A_3
=-0.122+0.95\times2.300
=2.063,
$$

$$
\hat A_1
=\delta_1+0.95\hat A_2
=0.511+0.95\times2.063
=2.471.
$$

最终结果为：

|token|局部 TD error $\delta_t$|GAE Advantage $\hat A_t$|解释|
|---|---:|---:|---|
|“运动”|0.511|2.471|同时包含后续“和、饮食”带来的终局收益|
|“和”|-0.122|2.063|局部 TD 为负，但未来终局收益使总优势为正|
|“饮食”|2.300|2.300|直接获得终局 RM reward|

这就是中间 token 没有 RM 分数却仍能得到 Actor 训练信号的原因：GAE 把后续 TD error 以 $(\gamma\lambda)^l$ 权重向前传播。它不是简单把 $8.2$ 平均分给三个 token，而是结合每一步 Value baseline 做信用分配。

### 6\.5 Step 4：构造 Critic 的固定 Value target

用 raw GAE 构造本轮 rollout 的 $\lambda$-return/Value target：

$$
\hat R_t^{\lambda}
=\hat A_t+V_{\mathrm{old}}(s_t).
$$

于是：

|$t$|$\hat A_t$|$V_{\mathrm{old}}(s_t)$|固定 target $\hat R_t^{\lambda}$|
|---|---:|---:|---:|
|1|2.471|5.500|7.971|
|2|2.063|6.000|8.063|
|3|2.300|5.889|8.189|

这些 target、raw Advantage 和经标准化后给 Actor 使用的 Advantage 都在 rollout 后一次性计算，并在本批 PPO 的多个 epoch 中 stop-gradient。后续更新只重新计算当前 Actor logprob 和当前 Value 预测。

### 6\.6 Step 5：逐 token 计算 Actor PPO-Clip Loss

当前策略相对 rollout 策略的概率比为：

$$
\rho_t
=\frac{\pi_\theta(a_t\mid s_t)}
{\pi_{\mathrm{old}}(a_t\mid s_t)}.
$$

逐 token surrogate 为：

$$
s_t^{(1)}=\rho_t\hat A_t,
\qquad
s_t^{(2)}
=\operatorname{clip}(\rho_t,0.8,1.2)\hat A_t,
$$

$$
\mathcal L_{\mathrm{actor},t}
=-\min(s_t^{(1)},s_t^{(2)}).
$$

代入数值：

|$t$|$\rho_t$|$s_t^{(1)}$|$s_t^{(2)}$|Actor loss|是否进入平坦裁剪区|
|---|---:|---:|---:|---:|---|
|1|$0.22/0.20=1.100$|$1.100\times2.471=2.718$|2.718|$-2.718$|否|
|2|$0.35/0.50=0.700$|$0.700\times2.063=1.444$|$0.800\times2.063=1.650$|$-1.444$|否；正优势动作的概率向错误方向降低，保留恢复梯度|
|3|$0.22/0.15=1.467$|$1.467\times2.300=3.373$|$1.200\times2.300=2.760$|$-2.760$|是；正优势动作已提高过多|

对三个有效 response token 做 masked mean：

$$
\mathcal L_{\mathrm{actor}}
=\frac{-2.718-1.444-2.760}{3}
\approx-2.307.
$$

如果先忽略最后的 token mean 系数，在未裁剪区域：

$$
\frac{\partial\mathcal L_{\mathrm{actor},t}}
{\partial\log\pi_\theta(a_t\mid s_t)}
=-\rho_t\hat A_t.
$$

所以本例三个 token 的 policy-surrogate 梯度系数分别约为：

|token|对当前所选 token logprob 的梯度系数|梯度下降效果|
|---|---:|---|
|“运动”|-2.718|提高该 token 在 $s_1$ 下的概率|
|“和”|-1.444|把已经降得过低的概率往回提高|
|“饮食”|0|该样本位于正优势上界的平坦区，不再继续推高概率|

实际反向传播还会把所选 token 的 logprob 梯度穿过 `log_softmax → logits → LM Head → Transformer`。由于不同位置共享模型参数，即使第三个样本自身的 clipped surrogate 梯度为 $0$，其他 token、其他回答、熵项或辅助 loss 仍可能间接改变“饮食”的概率。

### 6\.7 Step 6：逐 token 计算 Critic Loss

Critic 在每个有效状态都有一个预测，不是只对最后一个 token 训练。某次 PPO update 中重新计算当前预测 $V_\psi^{\mathrm{new}}(s_t)$，并与固定 target 比较：

$$
\mathcal L_{\mathrm{value},t}
=\frac12
\left(
V_\psi^{\mathrm{new}}(s_t)-\hat R_t^{\lambda}
\right)^2.
$$

在第一个 PPO optimizer step 之前，假设当前 Value 预测仍等于 rollout 时的旧预测，则：

|$t$|$V_\psi^{\mathrm{new}}(s_t)$|target $\hat R_t^{\lambda}$|误差 $V-\hat R$|单 token Value loss|
|---|---:|---:|---:|---:|
|1|5.500|7.971|-2.471|$0.5\times(-2.471)^2=3.052$|
|2|6.000|8.063|-2.063|$0.5\times(-2.063)^2=2.127$|
|3|5.889|8.189|-2.300|$0.5\times(-2.300)^2=2.645$|

对三个有效 token 取均值：

$$
\mathcal L_{\mathrm{value}}
=\frac{3.052+2.127+2.645}{3}
\approx2.608.
$$

均值之前，单 token 的预测值梯度为：

$$
\frac{\partial\mathcal L_{\mathrm{value},t}}
{\partial V_\psi^{\mathrm{new}}(s_t)}
=V_\psi^{\mathrm{new}}(s_t)-\hat R_t^{\lambda}.
$$

因此三个位置分别得到约 $-2.471、-2.063、-2.300$ 的梯度；若 loss 使用 token mean，回传前还要各除以 $3$。梯度下降会提高这三个状态的 Value 预测，使 Critic 下次看到相似前缀时更接近实际的剩余回报。

### 6\.8 后续 PPO epoch 中什么会变、什么保持固定

同一批 rollout 上继续训练时：

|量|后续 mini-batch/epoch 是否重算|原因|
|---|---|---|
|当前 $\log\pi_\theta(a_t\mid s_t)$|是|Actor 参数每次 optimizer step 后都会变化|
|PPO ratio $\rho_t$|是|分子使用最新 Actor，分母仍是固定 old logprob|
|当前 $V_\psi^{\mathrm{new}}(s_t)$|是|Critic 参数持续更新|
|old logprob、Reference logprob|否|它们描述生成本批 rollout 时的固定基准|
|token reward、raw GAE、Value target|否|它们是本批 rollout 的 stop-gradient 训练目标|
|用于 Actor 的 Advantage|否|可以在 rollout batch 上一次性 whitening，但多个 epoch 中不能反复改变|

因此，中间 token 的完整训练路径是：

```text
终局 RM reward + 每 token KL shaping
                    │
                    ▼
          r_1, r_2, ..., r_T
                    │
                    ▼
             TD error + GAE
                    │
          ┌─────────┴─────────┐
          ▼                   ▼
  每 token Advantage   每 token Value target
          │                   │
          ▼                   ▼
  PPO-Clip Actor loss   Critic MSE loss
          │                   │
          └─────────┬─────────┘
                    ▼
             各自反向传播更新
```

---

## 七、DPO：直接偏好优化

DPO（Direct Preference Optimization）可以理解为“把偏好数据直接变成 Policy 的训练目标”：不单独训练 Reward Model，不跑 PPO，也不需要 Value Model。它和 GRPO 都是在降低传统 PPO\-RLHF 的复杂度，但 DPO 不是 GRPO 的子机制，而是一条独立的离线偏好优化路线。

### 7\.1 DPO 与 Reward Model 训练的区别

DPO 和 RM 训练都使用 `chosen / rejected` 偏好对，因此形式上看起来很像。但二者训练对象不同：

|对比项|Reward Model 训练|DPO|
|---|---|---|
|训练对象|Reward Model / 打分器|Policy Model / 生成模型|
|输出|标量 reward score|token 概率分布|
|偏好数据用途|训练“裁判”学会打分|直接训练“演员”更偏向 chosen|
|后续流程|还需要 PPO / GRPO 更新 Actor|训练后 Policy 已完成偏好对齐|
|是否需要 Value Model|PPO 阶段通常需要|不需要|

Reward Model 训练显式学习 $r_\phi$，目标为

$$
\mathcal{L}_{\mathrm{RM}}
=-\log\sigma\!\left(r_\phi(x,y_w)-r_\phi(x,y_l)\right).
$$

DPO 的关键不是简单“把 $r$ 换成 logprob”，而是从 KL 正则化奖励最大化的闭式最优策略出发。对固定 prompt $x$，考虑：

$$
\max_{\pi}\;
\mathbb{E}_{y\sim\pi(\cdot\mid x)}[r(x,y)]
-\beta D_{\mathrm{KL}}\!\left(
\pi(\cdot\mid x)\,\|\,\pi_{\mathrm{ref}}(\cdot\mid x)
\right).
$$

在分布空间中求最优解可得：

$$
\pi^*(y\mid x)
=\frac{1}{Z(x)}
\pi_{\mathrm{ref}}(y\mid x)
\exp\!\left(\frac{r(x,y)}{\beta}\right).
$$

反解奖励：

$$
r(x,y)
=\beta\log\frac{\pi^*(y\mid x)}
{\pi_{\mathrm{ref}}(y\mid x)}
+\beta\log Z(x).
$$

将它代入 Bradley-Terry 偏好概率时，同一 prompt 的 $\beta\log Z(x)$ 在奖励差中抵消。用待训练策略 $\pi_\theta$ 参数化 $\pi^*$，得到：

$$
\mathcal{L}_{\mathrm{DPO}}(\theta)
=-\log\sigma\!\left(
\beta\left[
\log\frac{\pi_\theta(y_w\mid x)}
{\pi_{\mathrm{ref}}(y_w\mid x)}
-\log\frac{\pi_\theta(y_l\mid x)}
{\pi_{\mathrm{ref}}(y_l\mid x)}
\right]
\right).
$$

这就是 DPO 能跳过显式 RM 与在线 PPO 的原因：Policy 相对 Reference 的序列 log-ratio 同时扮演隐式 reward。$\beta$ 在原始 RL 目标中是 KL 正则强度，在 DPO loss 中又缩放分类 logit；实际训练行为还与学习率、长度分布、标签噪声和数据可分性耦合，不能只靠单变量直觉调参。

令括号内乘 $\beta$ 后的 margin 为 $z_\theta$，单个偏好对的梯度为：

$$
\nabla_\theta\mathcal{L}_{\mathrm{DPO}}
=-\beta\sigma(-z_\theta)
\left[
\nabla_\theta\log\pi_\theta(y_w\mid x)
-\nabla_\theta\log\pi_\theta(y_l\mid x)
\right].
$$

$\sigma(-z_\theta)$ 会给当前仍错排或 margin 较小的样本更大权重；已经大幅正确排序的样本权重趋小。Reference logprob 只进入 $z_\theta$，必须冻结或 `stop-gradient`，不能让 DPO loss 同时更新 Reference。由于两个序列共享参数，这个梯度表示两条序列 score function 的差，不保证 chosen 的每个 token 都单独上升。

### 7\.2 为什么 DPO 不需要 Value Model

Value Model 的作用是估计每个生成状态的未来期望收益，用于 PPO 中计算 token 级优势：

$$
A_t=G_t-V_\psi(s_t).
$$

DPO 不做训练期间的在线 rollout，也不计算 token 级 return / advantage。它读取预先收集的离线偏好对：

```text
prompt x
chosen response y_w
rejected response y_l
```

DPO 判断的是“整段 chosen response 比整段 rejected response 更好”，而不是判断“某个 token 是好还是坏”。

### 7\.3 DPO 如何把序列偏好传到 token

虽然 DPO 不显式判断单个 token 的好坏，但一段 response 的 log probability 是所有 token logprob 的和：

$$
\log\pi_\theta(y\mid x)
=\sum_{t=1}^{T}
\log\pi_\theta(y_t\mid x,y_{<t}).
$$

因此，当 DPO 提高 chosen 序列的相对 logprob、降低 rejected 序列的相对 logprob 时，梯度会沿着整段序列的每个 token logprob 反传：

|序列|DPO 目标|token 层面的结果|
|---|---|---|
|chosen response|提高相对 Reference 的序列 log-ratio|梯度沿 chosen 的 token logprob 路径反传|
|rejected response|降低相对 Reference 的序列 log-ratio|梯度沿 rejected 的 token logprob 路径反向作用|

这是一种“序列级偏好学习”的隐式信用分配。由于参数共享、Softmax 归一化以及 chosen/rejected 可能共享前缀，不能保证一次更新后 chosen 中每个 token 的概率都单调上升。标准 DPO 使用回答 token logprob 之和，因此响应长度、EOS 概率与截断方式都会影响 margin；改成长度平均会得到另一个目标，不能当作等价的纯数值稳定技巧。

工程实现必须只累计 **response token** 的条件 logprob，并让 Policy 与 Reference 使用完全相同的 tokenizer、chat template、BOS/EOS、截断和 loss mask。把 prompt token 也计入联合似然在理想数学中会因 chosen/rejected 共享 prompt 而抵消，但有限精度、不同拼接边界或错误 mask 会破坏抵消并引入无关梯度。Padding 位置必须排除，EOS 是否计入则要固定一致。

标准 DPO 通常把 Policy 初始化为 SFT 模型，并冻结同一起点的副本作为 $\pi_{\mathrm{ref}}$。Reference 不应跟随每一步 Policy 更新；若动态刷新 Reference，就已经改变了原始 DPO 目标。复用外部偏好集而拿不到生成它的 SFT checkpoint 时，原论文建议先对 chosen 做最大似然拟合以构造较匹配的 Reference，但这只能缓解、不能消除离线数据的行为策略偏移。

### 7\.4 DPO 主要使用场景

DPO 相比“训练 RM 再跑在线 PPO”少了 rollout、Value Model 和显式 RM 训练环节，工程链路更短；但训练稳定性与效果仍取决于 reference、beta、数据噪声和长度偏置。它本质上适合已有偏好对的离线优化，而不是在线探索算法。

|使用场景|为什么适合 DPO|典型数据|
|---|---|---|
|聊天模型偏好对齐|直接学习 chosen 比 rejected 更好的回答风格|人类/AI 标注的回答偏好对|
|安全拒答优化|学会何时拒答、如何拒答得更有帮助|安全场景 chosen/rejected|
|风格与语气调整|控制回答更礼貌、更简洁、更符合产品调性|品牌风格偏好对|
|SFT 后二阶段对齐|在已有 SFT 模型上低成本提升偏好质量|SFT 模型生成候选 \+ 偏好筛选|
|LoRA / QLoRA 轻量对齐|不需要 PPO rollout 和多模型在线训练，资源压力小|中小规模偏好数据|
|Rejection Sampling 后训练|用筛出的好/坏样本对继续拉开偏好差距|采样候选中的 winner/loser|

不建议只靠 DPO 的场景：

|场景|原因|更适合的方法|
|---|---|---|
|希望模型在训练中主动探索新解法|固定偏好对不能随当前策略持续扩展搜索覆盖|在线 RLVR / GRPO 等|
|需要 token/action 级 reward shaping|DPO 没有显式 advantage 和 Value baseline|PPO / GRPO|
|偏好数据噪声很大|DPO 会直接学习偏好数据中的偏差|数据清洗、RM 校准、RLAIF|
|需要持续在线改进策略|DPO 是离线训练，不做在线 rollout|PPO / GRPO / 在线 RL|

### 7\.5 DPO 的适用边界

- **没有显式 RM，不等于没有奖励假设。**标准 DPO 仍采用 Bradley-Terry/logistic 偏好模型，只是把 reward 重参数化为 policy/reference log-ratio。
- **没有训练期 rollout，不等于没有生成数据。**chosen/rejected 仍来自某个行为策略；若它们与当前策略差异很大，离线覆盖不足会限制可学行为。
- **不需要 Value，不等于有更细粒度信用分配。**DPO 对整段序列做对比，不能直接接收每一步 verifier reward。
- **Reference 通常仍然存在。**标准 DPO 需要冻结的 $\pi_{\mathrm{ref}}$；所谓 reference-free 实现对应不同近似或目标，应单独说明。
- **分类准确率不是最终指标。**偏好 loss 下降可能伴随长度漂移、模式塌缩或对 rejected 过度降概率，仍需独立生成评测。

一句话总结：DPO 不是 PPO 里的 Reward Model 训练，而是把“偏好比较 + KL 正则化最优策略”化成一个直接优化 Policy 的离线分类损失；它最适合已有高质量偏好对、且不依赖训练期在线探索的场景。

---

## 八、DeepSeek\-R1：从 RLHF 到 RLVR/GRPO 的工程实践

DeepSeek\-R1 不是传统“人类偏好 RM \+ PPO”的单一路线，而是把冷启动 SFT、规则奖励 RL、GRPO、拒绝采样、再 SFT、最终安全/通用对齐组合起来。GRPO 不再单独成章，而应作为 R1 推理 RL 阶段的核心算法来理解。

### 8\.1 R1\-Zero：不用人工 CoT 的纯 RL 探索

R1\-Zero 的关键实验是：不使用人工标注的长思维链 SFT 数据，直接从基础模型出发，用规则奖励和 GRPO 训练推理能力。

|维度|R1\-Zero 做法|关键含义|
|---|---|---|
|初始化模型|从基础模型开始做 RL|不先教模型固定 CoT 模板|
|训练算法|GRPO|去掉 Value Model，用组内相对优势降低显存压力|
|奖励来源|规则奖励为主|更接近 RLVR，而不是传统人类偏好 RM|
|任务范围|数学、代码等可验证任务|可以自动判断答案是否正确|
|人工推理数据|不依赖人工 CoT 标注|观察模型是否能自发形成长推理|

#### 8\.1\.1 GRPO 在 R1\-Zero 中解决什么问题

GRPO（Group Relative Policy Optimization）的核心是：不再训练单独的 Value Model，而是让 Actor 对同一个 prompt 多生成几个回答，用组内平均分和标准差构造相对优势。

对同一问题 $q$ 采样 $G$ 个输出 $\{o_i\}_{i=1}^{G}$，得到终局奖励 $\{r_i\}_{i=1}^{G}$。Outcome-supervision GRPO 常令一条回答的所有 token 共享：

$$
\hat A_{i,t}=\widetilde r_i
=\frac{r_i-\operatorname{mean}(\mathbf r)}
{\operatorname{std}(\mathbf r)+\varepsilon}.
$$

例如同一题生成 4 个回答：

```text
分数：[10, 8, 2, 0]
均值 = 5，population std = sqrt(17) ≈ 4.123

标准化优势：
A: (10-5)/4.123 = +1.213  → 鼓励
B: (8-5)/4.123  = +0.728  → 鼓励
C: (2-5)/4.123  = -0.728  → 惩罚
D: (0-5)/4.123  = -1.213  → 惩罚
```

这意味着 R1 的推理 RL 阶段不需要单独维护 `Value Model`，但仍然需要奖励信号。奖励可以来自规则、验证器、Reward Model 或混合信号；在 R1\-Zero 这类可验证任务中，主要使用规则奖励。

DeepSeekMath 原始 GRPO 的策略项仍采用 PPO 风格的 token ratio 与裁剪。记

$$
\rho_{i,t}(\theta)
=\frac{\pi_\theta(o_{i,t}\mid q,o_{i,<t})}
{\pi_{\mathrm{old}}(o_{i,t}\mid q,o_{i,<t})},
$$

则其核心目标可概括为：

$$
\begin{aligned}
\mathcal{J}_{\mathrm{GRPO}}(\theta)
=\mathbb{E}\Bigg[
\frac{1}{G}\sum_{i=1}^{G}\frac{1}{|o_i|}
\sum_{t=1}^{|o_i|}
\Big(&
\min\big(
\rho_{i,t}\hat A_{i,t},
\operatorname{clip}(\rho_{i,t},1-\epsilon,1+\epsilon)\hat A_{i,t}
\big)\\
&-\beta D_{\mathrm{KL}}(\pi_\theta\|\pi_{\mathrm{ref}})
\Big)\Bigg].
\end{aligned}
$$

与本文前面介绍的 PPO rollout reward shaping 不同，DeepSeekMath 的原始写法把 KL 直接放进 GRPO 目标。论文在采样 token 上使用的非负估计量为：

$$
\widehat D_{\mathrm{KL}}
=\frac{\pi_{\mathrm{ref}}(a_t\mid s_t)}
{\pi_\theta(a_t\mid s_t)}
-\log\frac{\pi_{\mathrm{ref}}(a_t\mid s_t)}
{\pi_\theta(a_t\mid s_t)}
-1.
$$

令 $u=\pi_{\mathrm{ref}}(a_t\mid s_t)/\pi_\theta(a_t\mid s_t)>0$，由 $u-\log u-1\geq 0$ 可知该样本值逐点非负；当 action 按**当前** $\pi_\theta$ 采样时，

$$
\mathbb{E}_{a_t\sim\pi_\theta}
\left[\widehat D_{\mathrm{KL}}\right]
=D_{\mathrm{KL}}\!\left(\pi_\theta\|\pi_{\mathrm{ref}}\right).
$$

但 GRPO rollout 实际来自 $\pi_{\mathrm{old}}$。多轮更新后若 $\pi_\theta\neq\pi_{\mathrm{old}}$，直接对旧样本平均不再严格是当前策略 KL 的无偏估计，除非做相应的重要性修正；实践依赖小步更新、裁剪或频繁刷新 rollout 控制误差。这里要把“估计量的数学性质”和“旧样本上的工程近似”分开。

后续工程实现可能采用不同 KL 估计、token/sequence 聚合或 clipping 细节，使用“GRPO”名称时应同时说明具体目标。

原始 DeepSeekMath 公式中的 $\frac{1}{|o_i|}\sum_t$ 先对每条回答做 token 平均，再在组内对回答平均，因此每条回答近似等权，而不是整个 batch 的每个 token 等权。这个选择会让短回答中单个 token 的权重更大；去掉 $1/|o_i|$、改成全局 token mean 或只在特定 token 上施加 advantage 都会得到不同的长度动力学，不能统称为同一个实现细节。

还要区分三种快照：

- $\pi_{\mathrm{old}}$ 是生成当前组样本的行为策略，用于 importance ratio；
- $\pi_\theta$ 是正在更新的当前策略；
- $\pi_{\mathrm{ref}}$ 用于 KL 正则。

在 DeepSeekMath 的迭代式 GRPO 算法中，Reference 会在外层迭代开始时刷新为当时的 Policy，而 $\pi_{\mathrm{old}}$ 在采样前更新；这与经典 PPO-RLHF 全程固定 SFT Reference 的做法不同。其他实现可能固定 Reference 或把 KL 系数设为零，因此看到“GRPO”不能自动推断 Reference 的生命周期和 KL 实现。

对于只有终局分数的写法，同一条回答的多个生成 token 共享组相对优势。这比 Value Model 的逐状态估计更省一套模型，但信用分配更粗：它知道“整条回答相对组内更好”，不直接知道哪一步推理最关键。若使用过程奖励，可以把未来步骤奖励累计到对应 token，得到更细粒度的 advantage，但这需要可靠的过程监督。若同组奖励几乎相同，标准差很小，应使用 $\varepsilon$、裁剪或跳过等策略避免数值不稳定；即使数值稳定，组内也几乎没有可用的相对排序信号。

|对比项|PPO\-RLHF|GRPO / R1 推理 RL|
|---|---|---|
|是否需要 Value Model|通常需要|不需要|
|baseline 来源|Value Model 预测 `V(s_t)`|同组多个回答的均值/标准差|
|奖励来源|通常是人类偏好 RM|规则奖励、验证器或混合奖励|
|是否在线采样|需要|需要，同一 prompt 多采样|
|主要代价|多模型显存和训练复杂度高|多次 rollout 推理成本高|

#### 8\.1\.2 奖励设计

R1\-Zero 的奖励不是“回答看起来更礼貌”这种主观偏好，而是更硬的规则信号：

|奖励项|作用|示例|
|---|---|---|
|准确性奖励|判断最终答案是否正确|数学题结果可比对标准答案，代码题可跑测试|
|格式奖励|约束模型把思考过程放到指定格式中|要求 reasoning 与 final answer 可解析|

这种奖励设计能减少对人工偏好标注的依赖，但也限制了适用范围：它最适合有 verifier 的任务，不适合开放式写作、审美、语气等主观任务。

#### 8\.1\.3 Aha Moment 与副作用

DeepSeek-R1 报告把训练中某个中间 checkpoint 的自我复核式输出称为 “Aha Moment”。这是定性案例：它说明策略在奖励驱动下出现了延长推理、回看与修正的行为，但不能仅凭一个示例证明模型形成了人类式“顿悟”或稳定的元认知能力。

|现象|正面意义|副作用|
|---|---|---|
|推理变长|有更多中间步骤可用于搜索答案|token 成本上升|
|自我反思|能修正一部分错误假设|可能出现重复检查或绕圈|
|格式约束|便于抽取 reasoning 与 final answer|格式奖励可能被模型投机利用|
|无人工 CoT|证明 RL 可以诱发推理行为|可读性、语言一致性和通用对话质量不足|

### 8\.2 正式版 R1：四阶段后训练流程

根据 DeepSeek-R1 技术报告，正式版 R1 没有直接沿用 R1-Zero 的纯 RL 结果，而是采用多阶段流程：先用冷启动数据做 SFT，再做推理 RL，再用拒绝采样扩充训练集并继续 SFT，最后做覆盖推理、帮助性与无害性的 RL。

|阶段|训练方式|数据 / 奖励|解决的问题|
|---|---|---|---|
|阶段 1：冷启动 SFT|少量高质量长 CoT 数据做 SFT|人工整理或高质量推理样本|提升可读性，减少 R1\-Zero 的混乱格式和语言混杂|
|阶段 2：推理 RL|GRPO / RLVR|可验证准确性奖励 + 语言一致性奖励|强化复杂推理，同时缓解语言混杂；报告指出一致性约束会带来轻微性能取舍|
|阶段 3：拒绝采样 \+ 通用 SFT|生成候选、过滤高质量样本，再 SFT|报告给出的规模约为 60 万推理样本 + 20 万非推理样本|把 RL 采样中的高质量轨迹沉淀为监督数据|
|阶段 4：最终 RL 对齐|混合奖励继续训练|推理任务用规则奖励，通用任务用偏好/安全奖励|补齐 helpfulness、harmlessness、通用对话和安全拒答|

#### 8\.2\.1 冷启动 SFT

冷启动不是为了把所有推理能力都教给模型，而是为了给 RL 一个更好的起点：

|目标|说明|
|---|---|
|规范输出格式|让模型知道 reasoning 与 final answer 应该如何组织|
|提升可读性|避免 R1\-Zero 中可能出现的语言混杂、格式混乱|
|降低 RL 难度|让后续 GRPO 不必从完全无结构的输出中探索|
|保留探索空间|数据量相对少，不把模型完全锁死在人工 CoT 风格里|

#### 8\.2\.2 推理 RL

推理 RL 是 R1 能力提升的核心阶段。它主要依赖可验证任务的规则奖励判断“答案是否正确”，并加入目标语言一致性奖励以改善可读性；因此不能简化成只有一个二值正确性分数。

|组件|R1 中的作用|
|---|---|
|Actor|生成多个候选推理路径和答案|
|Reward / Verifier|对最终答案正确性与语言一致性进行打分；具体可验证任务可使用答案检查或测试执行|
|GRPO|对同一 prompt 的多个回答做组内相对比较|
|Reference / KL|约束模型不要偏离初始策略过远|
|Value Model|不需要，GRPO 用组内均值/标准差替代 baseline|

如果同一题生成多个回答，奖励更高的回答获得正的组相对优势，奖励更低的回答获得负优势。这样不需要单独训练 Value Model，也能知道组内哪个回答相对更值得强化；但当整组全对或全错且其他奖励相同，组内比较信号会消失。

#### 8\.2\.3 拒绝采样与再 SFT

推理 RL 后，模型已经能生成更强的 reasoning 轨迹，但直接使用 RL checkpoint 作为最终模型并不一定最稳。R1 使用拒绝采样把模型生成的大量候选答案过滤成高质量训练集，再做一次监督微调。

|步骤|作用|
|---|---|
|多采样生成|对同一 prompt 生成多条 reasoning path|
|规则验证|保留答案正确、格式合规、推理较清晰的样本|
|质量过滤|去掉重复、混乱、语言不一致或低可读性样本|
|再 SFT|让模型以更稳定的监督学习方式吸收 RL 发现的高质量模式|

这里的关键不是“再做一遍普通 SFT”，而是把 RL 探索出来的好轨迹沉淀成训练数据，使模型在推理能力和生成稳定性之间取得平衡。

#### 8\.2\.4 最终通用与安全对齐

R1 的强推理能力主要来自规则奖励 RL，但一个可用的聊天模型还需要处理开放式问答、写作、角色约束、安全拒答、无害性等任务。因此最终阶段需要混合对齐：

|任务类型|更适合的奖励 / 训练信号|
|---|---|
|数学、代码、逻辑题|规则奖励、单元测试、答案验证器|
|通用问答、写作、摘要|偏好模型、人类/AI 反馈、拒绝采样|
|安全与无害性|安全 RM、红队数据、拒答质量评估|
|多语言与表达质量|SFT 数据、偏好数据、语言一致性奖励|

这说明 R1 并不是“只靠 RLVR 就够了”。可验证奖励解决推理正确性，偏好/安全对齐解决真实用户场景中的可用性和风险边界。

### 8\.3 R1、GRPO 与 DPO 的关系

|对比项|GRPO|DPO|R1 中的位置|
|---|---|---|---|
|方法类型|在线 RL / policy optimization|离线偏好优化|R1 核心使用 GRPO，而不是 DPO|
|数据来源|模型自己对同一 prompt 多采样|已有 chosen/rejected 偏好对|R1 推理阶段主要依赖多采样和规则奖励|
|是否需要 Value Model|不需要|不需要|R1 通过 GRPO 去掉 Value Model|
|是否需要奖励|需要规则奖励、RM 或 verifier|不需要显式 RM，但需要偏好对|R1 使用规则奖励和验证器强化推理|
|适合任务|数学、代码、可验证推理|聊天偏好、风格、安全拒答|R1 的强推理更匹配 GRPO/RLVR|

因此，GRPO 是 R1 推理 RL 阶段的核心算法；DPO 是另一条偏好优化路线，适合 SFT 后的离线偏好对齐，但不是 R1 推理能力形成的核心机制。

### 8\.4 R1 与传统 RLHF 的关系

|对比项|传统 PPO\-RLHF|DeepSeek\-R1 路线|
|---|---|---|
|核心目标|对齐人类偏好、提升帮助性和安全性|强化复杂推理，同时补齐通用对齐|
|奖励来源|Reward Model 学习人类偏好|规则奖励、验证器、偏好/安全奖励混合|
|主要算法|PPO \+ Value Model|GRPO 为核心，去掉 Value Model|
|数据来源|SFT 数据 \+ 偏好对 \+ rollout|冷启动 CoT、RL 采样、拒绝采样、通用数据|
|最适合任务|开放式聊天、主观偏好|数学、代码、可验证推理 \+ 通用聊天补齐|
|主要风险|Reward hacking、PPO 不稳定|格式投机、推理冗长、规则奖励覆盖不足|

更准确的表述是：R1 把传统 RLHF 中“人类偏好奖励”这一部分，扩展为“规则奖励 \+ 偏好奖励 \+ 安全奖励”的混合后训练体系。它不是传统 RLHF 的简单替代，而是后训练范式从 RLHF 走向 RLVR/GRPO/合成数据闭环的代表案例。

### 8\.5 R1 的蒸馏路线

R1 还展示了一个重要工程思路：不一定让小模型自己经历完整 RL 过程，而是用强模型生成高质量推理数据，再蒸馏到较小模型。

|路线|做法|优点|
|---|---|---|
|直接 RL 小模型|小模型自己做 GRPO/RLVR|能探索自身策略，但成本和稳定性压力大|
|R1 蒸馏小模型|用 R1 生成 reasoning 数据，小模型做 SFT|成本更低，训练更稳定，能快速获得推理风格|

因此，R1 的价值不只在一个最终模型，还在于提供了“强模型 RL 探索 → 高质量推理数据 → 小模型蒸馏”的可复用工程路径。

---

## 九、相关方法的边界：反馈来源不等于优化算法

很多术语并不在同一个分类维度。把它们都说成“PPO 的替代方案”会造成错误比较。

|维度|代表术语|它回答的问题|
|---|---|---|
|反馈由谁提供|RLHF、RLAIF|偏好判断来自人类，还是来自 AI/规则辅助？|
|奖励如何表示|显式 RM、Outcome RM、Process RM、verifier|信号是整段标量、过程分数，还是可执行验证结果？|
|策略如何更新|PPO、GRPO、DPO、KTO、SFT/RFT|拿到反馈后，用在线策略梯度还是离线损失更新？|
|数据如何产生|在线 rollout、固定偏好集、拒绝采样、蒸馏|训练样本是否随当前策略更新？|

### 9\.1 RLAIF：改变反馈来源

RLAIF（Reinforcement Learning from AI Feedback）通常让 AI 按原则或 rubric 比较候选回答，再用这些偏好训练模型。它不是一个固定 loss：

- AI 偏好可以训练显式 RM，再用 PPO/GRPO 更新策略；
- 也可以直接形成 chosen/rejected 对，用 DPO 等离线目标训练；
- Constitutional AI 的经典流程还包含“模型自我批评与修订”的监督阶段，以及用 AI 偏好训练 preference model 后进行 RL 的阶段。

RLAIF 降低人工标注成本，但会继承评审模型的偏差、盲点和提示敏感性。少量人类校准、位置随机化、多评审一致性和独立红队评测仍然重要。

### 9\.2 RLVR：改变奖励可验证性

RLVR（Reinforcement Learning with Verifiable Rewards）强调奖励能由答案检查器、编译器、单元测试、定理证明器或环境反馈验证。它同样不指定优化器：可以配 PPO、GRPO 或其他在线 RL 方法。

可验证不等于无漏洞。若 verifier 只检查最终字符串，策略可能利用解析器缺陷、测试覆盖不足或格式捷径；奖励设计必须验证“判分正确”与“任务真正完成”是否一致。开放式写作、价值判断和帮助性通常也不存在单一可靠 verifier，因此 RLVR 不能覆盖所有对齐目标。

### 9\.3 GRPO、DPO、KTO 与拒绝采样分别解决什么

|方法|在线采样|输入信号|显式 Value|核心边界|
|---|---|---|---|---|
|PPO|是|标量/逐步 reward|通常需要|能利用当前策略探索，但系统复杂|
|GRPO|是，同 prompt 多采样|组内可比较 reward|不需要|省去 critic，但 rollout 成本高，组内同分时信号弱|
|DPO|否|成对偏好|不需要|标准形式需要 Reference，依赖离线覆盖与 Bradley-Terry 假设|
|KTO|否|单条 desirable/undesirable 标签|不需要|不要求成对数据，目标与 DPO 不同，不能把二者 loss 混用|
|拒绝采样微调（RFT）|采样与训练分阶段|筛选后的正样本|不需要|本质是“生成→筛选→SFT”；筛选后不会对未选样本直接做策略梯度|

所以，一条实际训练管线可以同时是“RLAIF + DPO”，也可以是“RLVR + GRPO”。前者描述反馈来源与离线优化的组合，后者描述奖励性质与在线优化的组合；这些标签并不互斥。

---

## 十、评估、安全边界与调试

### 10\.1 端到端评估协议

RLHF 的调试不能只盯训练 loss 或 reward 曲线。一个可复现的评估协议需要固定模型 checkpoint、chat template、解码参数、最大长度、工具权限和随机种子，并把能力、偏好、安全与线上效果分开报告。

|层级|回答的问题|建议做法|
|---|---|---|
|可执行任务|数学、代码或结构化任务是否真的完成？|使用隐藏答案、单元测试或隔离环境；报告 pass rate 和超时/解析失败|
|离线生成质量|相对基线的回答是否更受偏好？|同 prompt、同采样预算盲评；随机 A/B 顺序；允许 tie/无法判断|
|RM 与 Judge 交叉验证|自动评审是否与人工一致？|保留独立人工校准集，按领域、长度差、位置和拒答类型报告一致率|
|安全评估|有害服从与过度拒答如何变化？|固定分类集 + 多轮/越狱红队 + 人工复核，不能只看一个拒答率|
|线上实验|离线改进是否转化为用户价值？|随机 A/B；同时观察满意度、投诉、留存、延迟、token 成本和故障率|

若把平局计半分，成对胜率可以写成：

$$
\operatorname{WinRate}
=\frac{N_{\mathrm{win}}+0.5N_{\mathrm{tie}}}
{N_{\mathrm{win}}+N_{\mathrm{loss}}+N_{\mathrm{tie}}}.
$$

也有评测选择丢弃 tie 或单独报告，必须写明口径。置信区间要以 prompt 为重采样单位；对同一 prompt 的多个采样或多名标注者判断并非相互独立。模型选择过程中反复查看的验证集不能再充当最终测试集，还要检查 benchmark 污染和 prompt 模板泄漏。

LLM-as-a-Judge 可以扩大评测规模，但已知会受到位置、冗长度、自相似风格和自身推理能力限制的影响。最低限度应交换 A/B 顺序、隐藏模型身份、固定 rubric、记录理由，并用人工样本校准；“Judge 胜率提高”不能单独证明真实用户偏好提高。

### 10\.2 跨算法的长度效应

长度既可能是质量的一部分，也可能成为伪相关信号。不同阶段的“归一化”作用位置不同，不能使用一个开关统一处理：

|阶段|标准量|长度如何进入|合理检查|
|---|---|---|---|
|RM|整段回答的标量分数|模型可从内容与长度相关性中学习偏好；分数本身不是 token 和|等长偏好对、长度差分桶、反事实删减/扩写|
|PPO-RLHF|终局 RM reward + token KL/advantage|token KL 通常按序列累加；loss 的 token/sequence 聚合又决定样本权重|同时报告总 KL、每 token KL、长度和截断率|
|DPO|chosen/rejected 的 response logprob 和|序列 logprob 天然随 token 数累加；长度平均会改变原始目标|按长度差切片，固定 EOS/mask；比较质量而非只看 margin|
|原始 DeepSeekMath GRPO|每序列 token 平均后做组平均|每条回答近似等权，短回答单 token 权重更大|报告组内长度—reward—advantage 相关性|
|自动 Judge|pairwise 选择|Judge 可能把信息量或冗长误当质量|交换顺序、长度控制指标、人工等长对校准|

因此，发现输出变长时不应立刻“所有分数除以长度”。先判断变化来自 RM、KL 累加、loss 聚合、EOS 学习还是 Judge 偏置，再选择对应修正；否则可能把确实需要详细说明的任务也错误压短。

### 10\.3 安全评估的边界

对齐训练不能证明模型在未测试输入上安全。安全评估至少要覆盖：

- **有害服从：**是否提供了会实质提高伤害能力的内容；
- **过度拒答：**对无害、教育性或危机求助问题是否错误拒绝；
- **多轮与越狱：**风险是否在上下文积累、角色扮演或编码改写后出现；
- **多语言与长尾：**安全策略是否只在高资源语言和常见模板上有效；
- **工具与环境：**文本看似安全不代表工具调用、代码执行或外部副作用安全；
- **分布漂移：**新产品功能、系统提示或工具权限变化后，旧安全分数不再自动有效。

安全集需要与训练数据、RM 数据和 checkpoint 选择集隔离，并同时保留“应拒绝”与“应回答”样本。高风险类别要由具备领域能力的评审或经过验证的规则判断；单一通用 Judge 不能充当最终安全证明。红队只能发现漏洞，未发现问题不等于不存在问题。

### 10\.4 常见误区与故障定位

|常见误区|更准确的说法|
|---|---|
|Reward 越高，模型越好|Reward 是代理指标；过度优化时独立质量可能停滞或下降|
|GRPO 完全不需要奖励模型|GRPO 不需要 Value Model；奖励可来自规则、RM 或混合信号|
|DPO 是 PPO 的简单替代|DPO 是离线偏好目标，不进行训练期 rollout，覆盖与探索能力不同|
|`KL_t` 就是完整 KL 散度|常见的 `KL_t` 是采样 token 的 logprob difference，平均后才估计前向 KL|
|LoRA 后显存只剩 Adapter|底座权重、激活值和并行通信缓存仍然存在|
|RLAIF 是一种新优化器|RLAIF 描述 AI 反馈来源，仍需搭配 RM+RL 或直接偏好目标|
|RLVR 就等于 GRPO|RLVR 描述奖励可验证，GRPO 是可使用这种奖励的一种在线优化器|
|一个总胜率足以选模型|平均值会隐藏领域退化、长度偏置、过度拒答与高风险尾部|

|问题|联合表现|优先排查|
|---|---|---|
|**Reward Hacking**|RM 分数上升，独立盲评下降，长度/格式异常|奖励漏洞、训练分布外样本、Judge 偏置；补充对抗样本并降低优化预算|
|**训练不稳定**|policy loss、clip fraction、entropy 同时剧烈震荡|学习率、batch、PPO epochs、advantage/reward 标准化与数值溢出|
|**KL 暴涨**|policy/reference KL 与响应风格突变|beta 控制器、学习率、old-policy 同步、采样与 logprob 分布是否一致|
|**Value 估计不准**|value loss 高、explained variance 低、advantage 方差大|终止/截断 mask、return target、Value 初始化和更新次数|
|**DPO 只会压低 rejected**|loss 下降但 chosen 生成质量不升|response mask、Reference、beta、长度差和数据覆盖；监控 chosen/rejected reward margin 与生成评测|
|**GRPO 组内无信号**|整组全对/全错，std 接近 0|任务难度、组大小、采样温度、奖励分辨率；不要只靠减小 $\varepsilon$ 制造假信号|

---

## 十一、总结

### 技术演进脉络

#### 现代对齐方法对比

从工程演进看，后续方法主要围绕“减少模型数量、降低 rollout 成本、提升奖励可控性”展开。

|方法|数据来源|是否需要 RM|是否需要 Value|适合场景|
|---|---|---|---|---|
|PPO\-RLHF|人类偏好 \+ 在线采样|经典管线需要|通常需要|可用在线奖励塑形，但实现复杂、调参敏感、资源开销高|
|GRPO|规则奖励 / RM / 混合奖励|可选|不需要|数学、代码、可组内比较的任务|
|DPO|偏好对|不单独训练显式 RM|不需要|离线偏好优化，流程简单|
|RLAIF|AI 反馈或 AI 辅助标注|取决于算法|取决于算法|降低人工标注成本|
|RLVR|可验证规则信号|通常不依赖人工 RM|取决于算法|数学、代码、逻辑推理等可验证任务|

总结起来，PPO 支持在线采样与细粒度 reward shaping；GRPO 省去 Value Model，但增加同 prompt 多采样成本并采用更粗的组相对 baseline；DPO 把固定偏好对变成离线分类式目标；RLAIF 改变反馈来源，RLVR 强调可验证奖励。它们不是按时间依次“全面替代”的单一路线。

### 关键启示

- **偏好是代理信号：**RLHF 让策略更符合特定数据与标注规范，不能证明模型“理解”或覆盖了全部人类价值

- **LoRA 只降低可训练参数成本：**它不消除底座加载、rollout、多模型推理和激活显存，是否能在有限硬件完成全流程仍取决于模型规模与系统方案

- **奖励与数据共同决定上限：**奖励必须可泛化、难被投机且覆盖目标行为；再好的公式也无法弥补系统性缺失或错误的反馈数据

- **反向传播是核心引擎：**无论哪个阶段，参数更新的本质都是梯度下降——从 Loss 出发，沿计算图回传，精确计算每个参数的贡献

### 参考资料

- [Training language models to follow instructions with human feedback（InstructGPT）](https://arxiv.org/abs/2203.02155)
- [Proximal Policy Optimization Algorithms](https://arxiv.org/abs/1707.06347)
- [Trust Region Policy Optimization](https://arxiv.org/abs/1502.05477)
- [High-Dimensional Continuous Control Using Generalized Advantage Estimation](https://arxiv.org/abs/1506.02438)
- [Direct Preference Optimization](https://arxiv.org/abs/2305.18290)
- [LoRA: Low-Rank Adaptation of Large Language Models](https://arxiv.org/abs/2106.09685)
- [DeepSeekMath: Pushing the Limits of Mathematical Reasoning in Open Language Models](https://arxiv.org/abs/2402.03300)
- [DeepSeek-R1](https://arxiv.org/abs/2501.12948)
- [Training a Helpful and Harmless Assistant with Reinforcement Learning from Human Feedback](https://arxiv.org/abs/2204.05862)
- [Scaling Laws for Reward Model Overoptimization](https://arxiv.org/abs/2210.10760)
- [RewardBench: Evaluating Reward Models for Language Modeling](https://arxiv.org/abs/2403.13787)
- [Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena](https://arxiv.org/abs/2306.05685)
- [Length-Controlled AlpacaEval: A Simple Way to Debias Automatic Evaluators](https://arxiv.org/abs/2404.04475)
- [Constitutional AI: Harmlessness from AI Feedback](https://arxiv.org/abs/2212.08073)
- [KTO: Model Alignment as Prospect Theoretic Optimization](https://arxiv.org/abs/2402.01306)
