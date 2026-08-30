# 《One-to-More: High-Fidelity Training-Free Anomaly Generation with Attention Control》

## 1. 论文总览

### 1.1. O 2 MAG
论文希望建立一个异常生成器。给定：

$$
I_R,
\quad
M_R,
\quad
I_N,
\quad
M_T,
\quad
y,
$$

生成：

$$
\hat I.
$$

1. $I_R$：reference anomaly image，真实参考异常图像。
2. $M_R$：reference anomaly mask，参考图像中真实异常的位置掩码。
3. $I_N$：normal image，目标正常图像。
4. $M_T$：target anomaly mask，希望在正常图像上生成异常的位置和大致形状。
5. $y$：文本提示，描述产品类别和异常类型。
6. $\hat I$：最终生成的异常图像。

论文对 $\hat I$ 有三个主要要求。
第一，目标掩码外的正常区域应尽可能保持 $I_N$ 的外观：

$$
\hat I_{\overline{M_T}}
\approx
(I_N)_{\overline{M_T}}.
$$

第二，目标掩码内应出现来自参考图像的异常属性：

$$
\operatorname{Attr}
\left(
\hat I_{M_T}
\right)
\approx
\operatorname{Attr}
\left(
(I_R)_{M_R}
\right).
$$

$\operatorname{Attr}(\cdot)$ 不是论文定义的具体函数，只是为了表达“颜色、纹理、边缘、局部结构及异常语义等视觉属性”。

第三，实际可见异常区域应尽可能与目标掩码一致：

$$
\operatorname{VisibleAnomalyRegion}
\left(
\hat I
\right)
\approx
M_T.
$$

假设 $M_T$ 标出一大片异常区域，但生成图像中只有一个很小的裂纹，那么生成图像和标签就不一致。使用这样的图像—掩码对训练分割网络，会把大量正常像素错误标成异常。

### 1.3. 概括 O2MAG

O2MAG 同时运行三个 Stable Diffusion 分支：

1. 参考异常分支提供真实异常的视觉特征。
2. 正常图像分支提供正常背景特征。
3. 目标异常分支保留自己的空间查询结构，并在目标异常区域读取参考异常的 Key、Value，在正常区域读取正常图像的 Key、Value。

同时：

1. AGO 优化文本嵌入，使文本语义更接近参考工业异常。
2. DAE 增强异常区域的 self-attention 和 cross-attention，使异常更明显、更完整地填入目标掩码。


### 1.4. 三个核心模块

论文的方法由三个模块

1. Tri-branch Attention Grafting，简称 TriAG。

它是整个系统的主干。其核心是三个并行扩散分支，以及从参考异常和正常图像向目标分支进行的 self-attention Key、Value 移植。

2. Anomaly-Guided Optimization，简称 AGO。

它冻结 Stable Diffusion 参数，仅优化文本 prompt embedding，使文本条件能够更好地解释一张真实参考异常图像。

3. Dual-Attention Enhancement，简称 DAE。

它分别增强：

$$
\text{self-attention 中的真实异常特征迁移}
$$

以及：

$$
\text{cross-attention 中异常 token 对目标掩码的影响}.
$$


### 1.6. 论文真正的新意在哪里

论文没有重新发明 Stable Diffusion、DDIM、attention 或 classifier-free guidance。它的主要贡献是针对工业异常生成重新组织这些已有机制。

可以把贡献边界分为两部分。

已有基础：

1. Latent Diffusion / Stable Diffusion。
2. VAE 潜空间。
3. U-Net。
4. Transformer scaled dot-product attention。
5. DDIM 与 DDIM inversion。
6. Classifier-free guidance。
7. CLIP 文本编码器。
8. 文本嵌入优化的思想。
9. 推理时 self-attention 特征注入的思想。

O2MAG 的方法设计：

1. 将参考异常、正常图像和目标图像组织为三个同步分支。
2. 保留目标 Query，只从参考异常或正常图像读取 Key、Value。
3. 使用参考掩码和目标掩码在 Key 轴与 Query 轴上进行前景—背景解耦。
4. 使用真实参考异常重建损失优化文本条件。
5. 同时增强异常区域的 self-attention 和 anomaly-token cross-attention。

## 2. 研究背景与问题定义

### 2.1. 工业异常检测包含哪些任务

工业异常检测不是一个单一任务，至少包含三个层次。

1. 图像级异常检测。
输入一张产品图像 $I$，输出一个异常分数：
$$
s_{\mathrm{img}}
\in
\mathbb R.
$$
设置阈值 $\delta$ 后：

$$
\hat y
=
\begin{cases}
1,
&
 s_{\mathrm{img}}\geq\delta,
\\
0,
&
 s_{\mathrm{img}}<\delta.
\end{cases}
$$

其中：
1. $\hat y=1$ 表示异常。
2. $\hat y=0$ 表示正常。
它只回答“有没有异常”，不回答异常在哪里。

2. 像素级异常定位。
模型输出一张与图像空间对应的异常概率图：
$$
S
\in
[0,1]^{H\times W}.
$$
其中 $S_{i,j}$ 越大，像素 $(i,j)$ 越可能属于异常。
再通过阈值得到预测掩码：
$$
\hat M_{i,j}
=
\begin{cases}
1,
&
S_{i,j}\geq\delta,
\\
0,
&
S_{i,j}<\delta.
\end{cases}
$$

3. 异常分类。
模型需要区分不同缺陷类型
对于分类任务，仅仅生成“看起来损坏”的图像不够；不同异常类型之间必须具有可辨别的语义差异。

### 2.4. 现有扩散异常生成方法的三条路线

论文在图 1 中把扩散异常生成方法概括为三类。

1. 模型微调。

这类方法修改 U-Net 或附加缺陷分支，使模型学习异常概念。典型思想包括 DreamBooth、defect block、双分支扩散或 inpainting 模型微调。

优点：

1. 可以让模型参数直接适应目标异常。
2. 通常能生成较明显的缺陷。

问题：

1. 每个产品类别可能需要重新训练。
2. 训练耗时大。
3. 需要保存额外模型权重。
4. 少量异常样本容易过拟合。
5. 生成异常时可能破坏正常背景。

2. 嵌入训练。

这类方法冻结扩散主干，学习一个异常 token 或文本 embedding，例如 textual inversion。

优点：

1. 参数量比微调整个 U-Net 小。
2. 可以将某种异常概念绑定到一个文本标识符。

问题：

1. 仍然需要逐类别或逐异常优化。
2. 单个文本 embedding 很难表达细粒度工业纹理。
3. 空间位置和异常形状控制有限。

3. 训练免除的注意力编辑。

例如 AnomalyAny，主要修改 cross-attention，使图像位置更强地关注异常词。

优点：

1. 不需要训练扩散主干。
2. 可以直接利用预训练模型的文本先验。

问题：

1. 通用文本中的 “crack” 不等于特定工业数据集中的真实裂纹。
2. cross-attention 更擅长粗粒度词—区域对应，难以提供细节纹理。
3. 异常类型和位置控制不稳定。
4. 无法稳定获得像素准确的异常掩码。

### 2.5. 为什么论文选择 self-attention 而不是只用 cross-attention

论文的一个基本判断是：

$$
\text{cross-attention}
\neq
\text{完整的局部视觉细节表示}.
$$

Cross-attention 的 Key、Value 来自文本 token。它主要回答：

> 当前图像位置应当关注 “hazelnut”“crack” 还是其他单词？

但文本中通常没有直接写出：

1. 裂纹具体宽度。
2. 破损边缘的细碎纹理。
3. 裂纹内部的深浅变化。
4. 异常与榛子沟槽之间的局部几何关系。
5. 参考异常的高频细节。

Self-attention 的 Key、Value 来自图像特征，包含图像内部位置间的相似性、结构、边界及纹理。因此论文希望直接从真实参考异常图像中提取 self-attention 特征。

### 2.6. 严格的问题形式化

从条件生成角度，O2MAG 实际建立的是条件生成过程：

$$
\hat I
\sim
p_{\mathrm{O2MAG}}
\left(
\hat I
\mid
I_R,M_R,I_N,M_T,y
\right).
$$

它不是无条件地从随机噪声生成任意异常图像，而是在一组非常明确的条件下生成。

目标可以拆成三个条件。

1. 正常区域保持条件：

$$
D_{\mathrm{bg}}
\left(
\hat I_{\overline{M_T}},
(I_N)_{\overline{M_T}}
\right)
\quad
\text{应当较小}.
$$

2. 异常属性迁移条件：

$$
D_{\mathrm{ano}}
\left(
\operatorname{Feat}
\left(
\hat I_{M_T}
\right),
\operatorname{Feat}
\left(
(I_R)_{M_R}
\right)
\right)
\quad
\text{应当较小}.
$$

3. 掩码一致性条件：

$$
D_{\mathrm{mask}}
\left(
\operatorname{AnomalyRegion}
\left(
\hat I
\right),
M_T
\right)
\quad
\text{应当较小}.
$$

需要注意，这三个距离函数不是论文直接定义并联合优化的损失。它们是为了把论文的文字目标分解清楚。论文实际并没有显式建立一个同时包含背景损失、异常特征损失和掩码损失的目标函数；它主要通过三分支注意力控制、AGO 和 DAE 间接实现这些目标。

### 2.7. “接近真实异常分布”应当怎样理解

论文多次使用“closer to the real anomaly distribution”之类的表述。这个说法不能理解成：

$$
p_{\mathrm{generated}}(x)
=
p_{\mathrm{real}}(x).
$$

论文没有证明两个完整高维分布相等，也没有直接最小化一个已知的分布距离。更准确的理解是：

1. 原始 Stable Diffusion 的通用异常概念与工业异常存在域差异。
2. 真实参考图像为生成过程提供实例级异常外观。
3. AGO 让文本条件更适合重建参考异常。
4. TriAG 将参考异常的图像级特征迁移到目标分支。
5. KID 和下游任务结果显示，有限生成样本在特征统计和实用性上优于若干基线。

一张参考图像只能说明某一种真实异常外观确实存在，不能说明所有异常模式的比例、变化范围和长尾情况。O2MAG 的 One-to-More 能力来自：

$$
\begin{aligned}
&\text{一张或少量真实异常参考}
\\
&+
\text{Stable Diffusion 的预训练先验}
\\
&+
\text{不同正常图像}
\\
&+
\text{不同目标掩码}
\\
&+
\text{扩散生成过程的变化能力}.
\end{aligned}
$$

### 2.8. One-shot、few-shot、zero-shot 与 training-free 的准确含义

1. One-shot。

一次 O2MAG 生成只需要一张参考异常图像—掩码对：

$$
(I_R,M_R).
$$

但 MVTec-AD 主实验并不是每种异常永远只使用一张图。论文将每种异常类型的前 $1/3$ 异常图像作为参考池，剩余 $2/3$ 用于评估。因此需要区分：

$$
\text{单次生成的一张参考}
$$

与：

$$
\text{整个实验可访问的少量参考集合}.
$$

Real-IAD 补充实验才明确采用每个异常类型仅第一张 top-view 异常图像的 one-reference 设置。

2. Few-shot。

模型只允许访问少量真实异常样本，而不是完整异常训练集。

3. Zero-shot cross-class。

目标类别没有该异常类型的真实参考，但使用另一个类别中相同异常类型的参考。例如：

$$
\text{wood-hole}
\longrightarrow
\text{hazelnut-hole}.
$$

4. Training-free。

论文定义下的 training-free 指：

1. 不更新 Stable Diffusion U-Net 参数。
2. 不更新 VAE 参数。
3. 不更新 CLIP 文本编码器参数。
4. 不为每个类别训练额外扩散模型。

但 AGO 仍然执行：

$$
500
$$

步文本 embedding 梯度优化。因此更精确的描述是：

$$
\text{frozen-model inference-time conditioning optimization}.
$$


## 3. 理解论文所需的基础概念（妈的，不懂得有点多）

### 图像

一张 RGB 图像可以表示为：

$$
I
\in
\mathbb R^{H\times W\times 3}.
$$

其中：

1. $H$ 是图像高度。
2. $W$ 是图像宽度。
3. 最后的 3 表示红、绿、蓝三个颜色通道。

### 像素空间与特征空间

像素空间直接表示颜色数值，但神经网络内部通常不一直处理原始 RGB。经过卷积、归一化、激活函数和 attention 后，会得到中间特征图：

$$
X
\in
\mathbb R^{B\times C\times h\times w}.
$$

其中：

1. $C$ 是特征通道数。（特征）（不过为什么是行呢？）
2. $h,w$ 是当前层的空间分辨率。（种类）
3. B 就是批次呗（那就是没什么用）

这些特征通道不再直接对应红、绿、蓝，而是网络学习到的模式。某些通道可能对以下内容敏感：

1. 边缘方向。
2. 粗糙纹理。
3. 物体轮廓。
4. 材质变化。
5. 异常区域。
6. 高层对象语义。

当 attention 处理特征图时，通常把空间维度展平：

$$
N
=
h\times w.
$$

于是：

$$
X
\in
\mathbb R^{B\times N\times C}.
$$

此时：

1. 每一行对应一个空间位置或 patch。
2. 每一列对应一个隐藏特征维度。（666 又变列了）




### U-Net 是什么

它包含：

1. 下采样路径。
2. 中间瓶颈。
3. 上采样路径。
4. 编码器到解码器的跳跃连接。

Stable Diffusion U-Net 的空间分辨率大致经历：

$$
64\times64
\rightarrow
32\times32
\rightarrow
16\times16
\rightarrow
8\times8
\rightarrow
16\times16
\rightarrow
32\times32
\rightarrow
64\times64.
$$

下采样部分逐渐扩大感受野，更容易建立全局结构；上采样部分逐步恢复局部空间细节。

论文补充材料指出：

1. 早期或较浅的 attention 更偏向语义布局和粗结构。
2. 更深的相关层逐渐包含高频、patch 级纹理。
3. 在本文设置中，down block 和 middle block 的 self-attention 对异常迁移帮助有限。
4. 因此主要在 up block 的第 10–16 层执行 self-attention grafting。

这里必须区分两个“早期”：

1. 去噪过程早期：指 50 个采样步骤中的前几步。
2. U-Net 网络早期层：指一次 U-Net 前向中的 down block。

论文说“产品结构在去噪早期形成”，不等于“要在 U-Net 的早期层进行 grafting”。实际上，它在去噪第 5 步后开始编辑，但选的是 U-Net decoder/up-block self-attention。




### CLIP 文本编码器、tokenizer 与 token

文本提示不能直接作为字符串输入 U-Net。

首先，tokenizer 将字符串切成 token。

然后，CLIP 文本编码器将 token 序列转换为：

$$
e
=
\tau_\theta(y)
\in
\mathbb R^{m\times d_\tau}.
$$

其中：

1. $y$ 是文本字符串。
2. $\tau_\theta$ 是冻结的 CLIP 文本编码器。
3. $m$ 是序列长度。
4. $d_\tau$ 是隐藏维度。

每个 token 的最终 embedding 不仅包含自身词义，还经过文本 Transformer 融合了上下文。

### 文本提示与文本条件

论文使用统一正提示模板：

$$
y
=
\text{“A photo of a [cls] with a [anomaly type]”}.
$$

例如：

$$
y
=
\text{“A photo of a hazelnut with a crack”}.
$$

其中：

1. $[cls]$ 表示物体类别。
2. $[anomaly\ type]$ 表示异常类型。

文本条件的作用不是把单词直接画到图像上，而是通过 cross-attention 改变不同空间位置的特征更新方向。

### Cross-attention

Cross-attention 中：

$$
Q
=
XW_Q
$$

来自图像空间特征，而：

$$
\begin{aligned}
K
&=
eW_K,
\\
V
&=
eW_V
\end{aligned}
$$

来自文本 embedding。

如果图像有 $N$ 个空间位置，文本有 $m$ 个 token，那么 cross-attention 矩阵通常具有：

$$
A_c
\in
\mathbb R^{N\times m}.
$$

矩阵含义：

1. 行 $i$：第 $i$ 个图像位置。
2. 列 $j$：第 $j$ 个文本 token。
3. $(A_c)_{i,j}$：图像位置 $i$ 对 token $j$ 的关注程度。

因此，DAE 对 anomaly token 的增强，本质上是在 cross-attention 矩阵中特定列、特定行区域上增加影响。

### Self-attention 与 Cross-attention 的分工

| 机制              | Query 来源 | Key/Value 来源 | 更擅长表达               | O2MAG 中的作用        |
| --------------- | -------- | ------------ | ------------------- | ----------------- |
| Self-attention  | 图像特征     | 图像特征         | 空间结构、纹理、边界、位置关系     | 从真实参考图像迁移异常视觉特征   |
| Cross-attention | 图像特征     | 文本 embedding | 图像位置与文本 token 的语义对应 | 强调异常类型及其在掩码内的文本影响 |


### 多头注意力

实际模型通常使用 multi-head attention。

假设总隐藏维度为 $C$，注意力头数为 $H_a$，则每个头的维度为：

$$
d
=
\frac{C}{H_a}.
$$

Q、K、V 会被重排为：

$$
Q
\in
\mathbb R^{B\times H_a\times N_q\times d},
$$

$$
K,V
\in
\mathbb R^{B\times H_a\times N_k\times d}.
$$

每个头独立计算：

$$
O_h
=
\operatorname{softmax}
\left(
\frac{Q_hK_h^\top}{\sqrt d}
\right)V_h.
$$

最后拼接：

$$
O
=
\operatorname{Concat}
\left(
O_1,
\ldots,
O_{H_a}
\right)W_O.
$$

论文公式为了简洁省略了 batch 和 head 维度。复现时，掩码必须正确广播到每个 batch、每个 head。

### Attention mask 与 mask filling

如果某些 Key 不允许被关注，可以在 Softmax 前将对应 logit 设为一个极小值：

$$
\tilde s_{i,j}
=
\begin{cases}
s_{i,j},
&
\text{允许},
\\
-\infty,
&
\text{禁止}.
\end{cases}
$$

因为：

$$
\exp(-\infty)=0,
$$

所以：

$$
\operatorname{softmax}
\left(
\tilde s_i
\right)_j
=0
$$

对被屏蔽位置成立。


论文将该操作记为：

$$
\mathcal{MF}(\cdot),
$$

即 mask-filling operator。

论文公式（4）和（5）在排版上写成了与 $\mathcal{MF}$ 的逐元素乘法，但从语义上必须理解为 masked fill 或 additive mask，不能真的把一般 logit 与 $-\infty$ 相乘，否则可能产生正无穷或 NaN。

### Key 轴掩码与 Query 轴门控的区别

这是理解 TriAG 最关键的形状问题。

假设：

$$
A_{\mathrm{fg}}
\in
\mathbb R^{N_T\times N_R}.
$$

其中：

1. 行对应目标分支 Query 位置。
2. 列对应参考分支 Key 位置。

参考掩码 $M_R$ 应当屏蔽列，也就是来源位置：

$$
A_{\mathrm{fg}}[:,j]
\quad
\text{由}
\quad
M_{R,j}
\quad
\text{决定是否允许}.
$$

而目标掩码 $M_T$ 在公式（6）中门控行，也就是决定每个目标位置最终使用异常输出还是背景输出：

$$
O_i^*
=
M_{T,i}O_i^{\mathrm{fg}}
+
(1-M_{T,i})O_i^{\mathrm{bg}}.
$$

简化为：

```text
A_fg 的列：参考图像中的来源位置，使用 M_R 屏蔽
A_bg 的列：正常图像中的来源位置，使用 M_T 的反区域屏蔽
输出 O 的行：目标图像中的接收位置，使用 M_T 选择前景或背景
```

### Logit bias

给某个 attention logit 加一个偏置：

$$
\hat s_j
=
s_j+b_j.
$$

Softmax 前的未归一化权重变为：

$$
\exp(\hat s_j)
=
\exp(s_j)
\exp(b_j).
$$

如果：

$$
b_j
=
\log\gamma,
$$

则：

$$
\exp(b_j)
=
\gamma.
$$

因此，加 $\log\gamma$ 等价于在未归一化权重层面乘以 $\gamma$，前提是该偏置只加到部分候选上，而不是所有有效候选都加相同常数。

### Softmax temperature

带温度的 Softmax 为：

$$
p_j
=
\frac{
\exp(s_j/\tau)
}{
\sum_k
\exp(s_k/\tau)
}.
$$

当：

$$
0<\tau<1
$$

时，logit 差异被放大，分布更尖锐。


### DDPM、DDIM 与采样步数

DDPM 通常使用带随机性的反向采样。DDIM 提出了一条可以使用更少步骤、并且在特定设置下确定性的采样路径。

需要区分两个时间尺度：

1. 训练扩散时间步。t

Stable Diffusion 的训练 scheduler 常包含很多原始时间点，例如 1000 个。

2. 推理采样步数。k

论文使用 50 个 denoising steps。Scheduler 会从原始时间轴中选出 50 个时间点进行采样。


### DDIM inversion 是什么

普通生成方向是：

$$
z_T
\longrightarrow
z_{T-1}
\longrightarrow
\cdots
\longrightarrow
z_0
\longrightarrow
x.
$$

DDIM inversion 尝试执行相反过程：

$$
x
\longrightarrow
z_0
\longrightarrow
z_1
\longrightarrow
\cdots
\longrightarrow
z_T.
$$

给定真实图像 $I$：

1. 使用 VAE 编码：

$$
z_0
=
\mathcal E(I).
$$

2. 根据 DDIM 的确定性轨迹逐步增加噪声，得到：

$$
\left\{
z_0,z_1,\ldots,z_T
\right\}.
$$

这条轨迹不是随便加入高斯噪声得到的普通噪声序列，而是希望在相同文本条件和去噪模型下能够近似重建原图的轨迹。

O2MAG 分别对参考异常图像和正常图像执行 inversion：

$$
\mathcal Z_R
=
\left\{
Z_0^{\mathrm{ref}},
\ldots,
Z_T^{\mathrm{ref}}
\right\},
$$

$$
\mathcal Z_N
=
\left\{
Z_0^{\mathrm{nor}},
\ldots,
Z_T^{\mathrm{nor}}
\right\}.
$$

在同一采样时间点，参考、正常和目标分支的噪声程度相近，才适合进行 Q、K、V 特征交换。

### “反演噪声”为什么仍然包含图像信息


> 既然 $Z_T$ 看起来像噪声，为什么复制正常图像的 $Z_T^{\mathrm{nor}}$ 能帮助保持正常结构？

关键在于，DDIM inversion 得到的高噪声 latent 不是与图像完全无关的独立随机噪声。它位于一条由该图像反演得到的确定性轨迹上。结合相应文本条件和扩散模型去噪时，它仍然能够趋向原图。

因此：

$$
Z_T^{\mathrm{tar}}
=
Z_T^{\mathrm{nor}}
$$

使目标分支一开始沿着重建正常图像的轨迹运行，而不是从任意随机噪声重新生成整个产品。

但论文也指出，仅共享初始化仍不足以完全保持背景，所以还保留正常图像分支，在后续 attention 中持续提供背景 Key、Value。

### Classifier-free guidance

Classifier-free guidance，简称 CFG，用来增强文本条件。

模型通常计算两种噪声预测：

1. 条件预测：

$$
\epsilon_{\mathrm{cond}}
=
\epsilon_\theta(z_t,t,e_{\mathrm{cond}}).
$$

2. 无条件预测：

$$
\epsilon_{\mathrm{uncond}}
=
\epsilon_\theta(z_t,t,e_{\mathrm{uncond}}).
$$

然后：

$$
\begin{aligned}
\hat\epsilon_t
={}&
\epsilon_{\mathrm{uncond}}
\\
&+
g
\left(
\epsilon_{\mathrm{cond}}
-
\epsilon_{\mathrm{uncond}}
\right).
\end{aligned}
$$

展开：

$$
\hat\epsilon_t
=
(1-g)
\epsilon_{\mathrm{uncond}}
+
g
\epsilon_{\mathrm{cond}}.
$$

当 $g>1$ 时，模型是从无条件预测向条件预测方向外推。

论文设置：

$$
g=7.5.
$$

### Negative prompt

普通 CFG 的无条件 embedding 往往来自空提示。O2MAG 使用描述正常外观的 negative prompt 作为无条件分支，例如：

1. no crack。
2. no scratch。
3. no cut。
4. smooth surface。
5. intact shell。

论文公式（8）为：

$$
\begin{aligned}
\hat\epsilon_t
={}&
\epsilon_\theta(x_t,y_n,t)
\\
&+
g
\left(
\epsilon_\theta(x_t,y,t)
-
\epsilon_\theta(x_t,y_n,t)
\right).
\end{aligned}
$$

其中：

1. $y$ 是正提示。
2. $y_n$ 是负提示。
3. $g$ 是 CFG scale。

直观上，正提示把图像推向“有 crack”，负提示作为对照方向，让结果远离“no crack、smooth surface”。

需要注意，负提示并不是逻辑上的严格否定器。它只是改变扩散条件方向，不能保证某种属性绝对不存在。

不过我觉得这么写有没有什么道理呢？
$$
\begin{aligned}
\hat\epsilon_t
=\epsilon_{uncond}
+g
\left(
\epsilon_\theta(x_t,y,t)
-
\epsilon_\theta(x_t,y_n,t)
\right).
\end{aligned}
$$
### 参数、梯度与优化变量


O2MAG 的 AGO 不更新 $\theta$，而是更新文本 embedding $e$：

$$
e
\leftarrow
e
-
\eta
\nabla_e\mathcal L.
$$

所以必须区分：

1. 模型参数被冻结。
2. 中间输入条件仍可求梯度和优化。


### PCA 是什么

PCA 的全称是 Principal Component Analysis，主成分分析。

假设每个样本是一个高维向量：

$$
x_i
\in
\mathbb R^D.
$$

PCA 希望找到若干正交方向：

$$
u_1,u_2,\ldots,u_k
$$

使数据投影到这些方向后保留尽可能大的方差。


论文把 self-attention 高维特征压缩到前三个主成分，再映射到 RGB：

$$
\text{高维 attention 表示}
\longrightarrow
\mathbb R^3
\longrightarrow
\text{RGB 颜色}.
$$

相似颜色表示对应空间位置拥有相似 attention 模式。

### 论文中的 self-attention PCA 可视化如何理解

Self-attention 矩阵可写为：

$$
A
\in
\mathbb R^{N\times N}.
$$

对每个空间位置 $i$，其整行：

$$
A_{i,:}
\in
\mathbb R^N
$$

描述该位置如何关注所有其他位置。

一种常见可视化方式是：

1. 把每个位置 $i$ 看作一个样本。
2. 把 $A_{i,:}$ 看作该样本的 $N$ 维特征。
3. 对 $N$ 个样本执行 PCA。
4. 取前三个主成分值作为 RGB。
5. 把 $N=h\times w$ 个颜色重新排成 $h\times w$ 图。




### KID

KID 的全称是 Kernel Inception Distance。

首先使用 Inception 网络提取真实图像和生成图像特征：

$$
u_i
=
\phi(x_i^{\mathrm{real}}),
$$

$$
v_j
=
\phi(x_j^{\mathrm{gen}}).
$$

然后使用核函数计算两组样本的 Maximum Mean Discrepancy。无偏估计的一种形式为：

$$
\begin{aligned}
\operatorname{KID}
={}&
\frac{1}{m(m-1)}
\sum_{i\neq j}
k(u_i,u_j)
\\
&+
\frac{1}{n(n-1)}
\sum_{i\neq j}
k(v_i,v_j)
\\
&-
\frac{2}{mn}
\sum_{i=1}^{m}
\sum_{j=1}^{n}
k(u_i,v_j).
\end{aligned}
$$

KID 越低，表示两组 Inception 特征分布越接近。

它并不证明像素级分布完全相同，原因包括：

1. Inception 特征会丢失某些细节。
2. 使用有限样本估计。
3. 工业异常与 ImageNet 特征的匹配并不完美。
4. KID 不直接评价掩码一致性。

论文选择 KID，是因为其无偏估计在小数据集上通常比 FID 更可靠。

### LPIPS 与 IC-LPIPS

LPIPS 使用预训练深度网络多个层的特征差异衡量两张图的感知距离。

简化表达为：

$$
\operatorname{LPIPS}(x,x')
=
\sum_l
\frac{1}{H_lW_l}
\sum_{h,w}
\left\|
w_l
\odot
\left(
\hat f_l(x)_{h,w}
-
\hat f_l(x')_{h,w}
\right)
\right\|_2^2.
$$

其中：

1. $f_l$ 是第 $l$ 层特征。
2. $\hat f_l$ 是归一化特征。
3. $w_l$ 是学习或设定的通道权重。

IC-LPIPS 是同一类别或同一 cluster 内样本两两 LPIPS 的平均，用于衡量多样性。

IC-LPIPS 越高通常意味着生成样本差异更大，但不能单独理解为质量更高。假设一个方法随机破坏正常背景，它也会增大样本间差异。论文指出，DualAnoDiff 等方法较高的 IC-LPIPS 部分来自背景损坏，因此需要结合 KID 和视觉结果判断。

**LPIPS：两张图片“看起来”有多不一样。**  
**IC-LPIPS：一批生成图片彼此有多不一样，也就是生成结果的多样性。**

### 3.41. AUROC
| 真实情况 | 模型判断 | 名称        |
| ---- | ---- | --------- |
| 异常   | 异常   | TP，正确找到异常 |
| 异常   | 正常   | FN，漏检异常   |
| 正常   | 异常   | FP，正常被误报  |
| 正常   | 正常   | TN，正确识别正常 |
对不同阈值计算：

所有真正异常里面，我成功找出了多少：
$$
\operatorname{TPR}
=
\frac{TP}{TP+FN},
$$
所有正常样本里面，有多少被我误判成异常：
$$
\operatorname{FPR}
=
\frac{FP}{FP+TN}.
$$

以 FPR 为横轴、TPR 为纵轴得到 ROC 曲线，曲线下面积为 AUROC。

1. AUROC-I：图像级 AUROC。
2. AUROC-P：像素级 AUROC。

AUROC 衡量排序能力，对类别不平衡有一定鲁棒性，但当正常像素极多时，像素 AUROC 可能仍然显得很高，所以还需要 AP 和 F1。

### Precision、Recall、AP 与 F1-max

Precision：报出来的异常有多准

$$
\operatorname{Precision}
=
\frac{TP}{TP+FP}.
$$

Recall：真正的异常找全了吗

$$
\operatorname{Recall}
=
\frac{TP}{TP+FN}.
$$
![[Pasted image 20260828132511.png|422]]


1. AP-I：图像级平均精确率。
2. AP-P：像素级平均精确率。

F1：模型能不能同时做到“报得准”和“找得全”

$$
F_1
=
\frac{
2
\cdot
\operatorname{Precision}
\cdot
\operatorname{Recall}
}{
\operatorname{Precision}
+
\operatorname{Recall}
}.
$$

F1-max 表示遍历不同阈值后取得的最大 F1：

$$
F_{1,\max}
=
\max_\delta
F_1(\delta).
$$

论文表格中的 F1-I 和 F1-P 分别对应图像级和像素级最佳 F1。

## 4. O2MAG 方法：先把整篇论文串起来

### 4.1. 先看整个方法到底在干什么

前面已经知道，O2MAG 的任务不是直接判断一张图有没有异常，而是利用少量真实异常样本生成更多“异常图像 + 异常掩码”，再用这些数据训练后续的异常检测、定位或分类模型。

一次生成时，给 O2MAG：

$$
(I_R,M_R),\quad I_N,\quad M_T,\quad y
$$

其中：

1. $I_R$：真实参考异常图像，告诉模型“这种异常真实长什么样”。
2. $M_R$：参考异常掩码，告诉模型“参考图里哪一部分才是真正的异常”。
3. $I_N$：正常图像，告诉模型“最终产品整体应该保持什么样”。
4. $M_T$：目标异常掩码，告诉模型“希望把异常生成到哪里”。
5. $y$：文本提示，例如 `A photo of a hazelnut with a crack`，告诉模型“物体类别和异常类型是什么”。

最终生成：

$$
\hat I
$$

希望得到：

$$
(\hat I,M_T)
$$

作为新的异常训练样本。

整套方法实际可以按照下面的顺序理解：

1. 先用 AGO 根据真实参考异常图 $I_R$ 优化文本 embedding，让普通的“crack”文本语义更接近当前工业数据中的真实 crack。
2. 对参考异常图 $I_R$ 和正常图 $I_N$ 分别做 DDIM inversion，得到两条对应的扩散轨迹。
3. 建立三个扩散分支：Reference、Normal、Target。
4. Target 分支从 Normal 分支的高噪声 latent 开始，因此一开始就尽量继承正常图像的整体结构。
5. 在每一个去噪步骤中，Reference 分支提供真实异常的 $K_R,V_R$，Normal 分支提供正常背景的 $K_N,V_N$。
6. Target 分支保留自己的 $Q_T$，然后使用 TriAG：
   - 目标掩码内部，用 $Q_T$ 去查询参考异常中的 $K_R,V_R$；
   - 目标掩码外部，用 $Q_T$ 去查询正常图像中的 $K_N,V_N$。
7. 如果异常在目标掩码中表现得还不够明显，再用 DAE：
   - 增强 self-attention 中对真实异常特征的读取；
   - 增强 cross-attention 中 anomaly token 对目标区域的影响。
8. U-Net 根据这些被修改过的 attention 预测噪声。
9. DDIM 根据预测噪声继续执行下一步去噪。
10. 最后把 Target 分支的 latent 通过 VAE decoder 解码成最终异常图像 $\hat I$。

因此，这篇论文其实就是在回答一个问题：

> 如何在不重新训练 Stable Diffusion 的情况下，把一张真实异常图中的“异常内容”迁移到另一张正常图指定的位置，同时尽量保持正常图的其他区域不变？

它的答案是：

$$
\begin{aligned}
\text{O2MAG}
={}&
\text{AGO：先修正文本异常语义}
\\
&+
\text{TriAG：迁移真实异常并保持正常背景}
\\
&+
\text{DAE：让异常在目标掩码中真正显现出来}.
\end{aligned}
$$

### 4.2. 三个模块分别负责什么

| 模块    | 解决的问题                             | 核心做法                                                          |
| ----- | --------------------------------- | ------------------------------------------------------------- |
| TriAG | 怎么把真实异常迁移到目标图，同时保持背景              | Target 保留自己的 Query，异常区域读取 Reference 的 K/V，正常区域读取 Normal 的 K/V |
| AGO   | 普通文本中的 crack、hole 等语义与工业真实异常不完全一致 | 冻结 Stable Diffusion，只优化 prompt embedding，使它更适合重建参考异常图         |
| DAE   | TriAG 生成的异常有时太弱、没有填满目标 mask       | 同时增强 self-attention 的异常特征和 cross-attention 的 anomaly token    |

这三个模块不是三个互不相关的方法。

它们在同一条生成流程中分工：

$$
\text{AGO}
\longrightarrow
\text{准备更准确的文本条件}
$$

$$
\text{TriAG}
\longrightarrow
\text{完成主要的异常迁移和背景保持}
$$

$$
\text{DAE}
\longrightarrow
\text{在 TriAG 基础上进一步强化异常}
$$

### 4.3. 三个扩散分支的作用

TriAG 中有三个分支。

1. Reference-anomaly branch

输入真实异常图：

$$
I_R
$$

DDIM inversion 后得到：

$$
Z_T^{\mathrm{ref}},
Z_{T-1}^{\mathrm{ref}},
\dots,
Z_0^{\mathrm{ref}}
$$

这个分支最重要的作用是提供：

$$
K_R,\quad V_R
$$

也就是“真实异常图像中的中间视觉特征”。

2. Normal-image branch

输入正常图：

$$
I_N
$$

得到：

$$
Z_T^{\mathrm{nor}},
Z_{T-1}^{\mathrm{nor}},
\dots,
Z_0^{\mathrm{nor}}
$$

这个分支最重要的作用是提供：

$$
K_N,\quad V_N
$$

也就是“正常图像中的背景和产品外观特征”。

3. Target-anomaly branch

这是最终真正生成图像的分支。

它初始化为：

$$
Z_T^{\mathrm{tar}}
=
Z_T^{\mathrm{nor}}
$$

因此 Target 一开始沿着“重建正常图”的轨迹走。

之后每一步再通过 TriAG 把目标掩码中的局部内容逐渐改造成异常。

最终只有 Target 分支会被解码成：

$$
\hat I
$$

所以可以把三个分支简单记成：

$$
\boxed{
\text{Reference：提供异常}
}
$$

$$
\boxed{
\text{Normal：提供正常背景}
}
$$

$$
\boxed{
\text{Target：接收两边的信息并生成最终图像}
}
$$

---

## 5. Tri-branch Attention Grafting：论文最核心的主干

### 5.1. TriAG 的基本思路

普通 self-attention 中，Target 分支自己产生：

$$
Q_T,\quad K_T,\quad V_T
$$

然后计算：

$$
\operatorname{Attn}(Q_T,K_T,V_T)
=
\operatorname{softmax}
\left(
\frac{Q_TK_T^{\mathrm T}}{\sqrt d}
\right)V_T
$$

这表示：

> Target 图像中的每一个位置，只能从 Target 自己当前的图像特征中读取信息。

但 O2MAG 希望目标图像获得两种外部信息：

1. 目标异常区域需要真实参考异常的纹理和结构。
2. 正常区域需要原正常图像的背景和产品特征。

所以 TriAG 不再让 Target 只看自己的 $K_T,V_T$，而是改成：

$$
Q_T
+
(K_R,V_R)
$$

以及：

$$
Q_T
+
(K_N,V_N)
$$

这里最关键的是：

$$
\boxed{Q_T\text{ 保持不变}}
$$

因为 $Q_T$ 来自当前目标图像，它决定：

> 目标图像当前位置到底需要什么信息。

而 $K_R,V_R$ 和 $K_N,V_N$ 决定：

> 去哪里找，以及真正拿回什么内容。

因此可以简单记成：

$$
\boxed{
Q_T：目标图决定“我要什么”
}
$$

$$
\boxed{
K_R,V_R：参考异常图提供“异常内容”
}
$$

$$
\boxed{
K_N,V_N：正常图提供“正常内容”
}
$$

### 5.2. 从普通 Self-attention 推到公式（3）

普通 Target self-attention 的匹配分数是：

$$
A_T
=
\frac{
Q_TK_T^{\mathrm T}
}{
\sqrt d
}
$$

TriAG 做的第一步非常直接：

异常区域不再使用 $K_T$，改用参考异常的 $K_R$：

$$
A_{\mathrm{fg},t,l}
=
\frac{
Q_{T,t,l}K_{R,t,l}^{\mathrm T}
}{
\sqrt d
}
$$

正常区域则使用正常图像的 $K_N$：

$$
A_{\mathrm{bg},t,l}
=
\frac{
Q_{T,t,l}K_{N,t,l}^{\mathrm T}
}{
\sqrt d
}
$$

合起来就是论文公式（3）：

$$
\begin{aligned}
A_{\mathrm{fg},t,l}
&=
\frac{
Q_{T,t,l}K_{R,t,l}^{\mathrm T}
}{
\sqrt d
},
\\
A_{\mathrm{bg},t,l}
&=
\frac{
Q_{T,t,l}K_{N,t,l}^{\mathrm T}
}{
\sqrt d
}.
\end{aligned}
\tag{3}
$$

其中：

1. $t$：当前去噪步骤。
2. $l$：当前 U-Net self-attention 层。
3. $\mathrm{fg}$：foreground，异常前景。
4. $\mathrm{bg}$：background，正常背景。
5. $d$：单个 attention head 的维度。

这一步只完成了“Target 去查询 Reference / Normal”。

但还有一个问题：

> Reference 图中不只有异常，Normal 图中也有一部分位置准备被目标异常覆盖，怎么保证 Target 不会读错地方？

所以接下来需要两个 mask。

### 5.3. 公式（4）：只允许读取参考图中的真实异常

参考图 $I_R$ 中同时包含：

1. 真实异常区域。
2. 正常产品区域。
3. 可能还有背景。

所以不能让 Target 在整张 Reference 上随便找。

论文使用参考掩码：

$$
M_R
$$

规定：

$$
M_R=1
$$

的位置才允许作为异常来源。

可以把它理解成：

$$
\tilde A_{\mathrm{fg},i,j}
=
\begin{cases}
A_{\mathrm{fg},i,j},
&
M_{R,j}=1,
\\
-\infty,
&
M_{R,j}=0.
\end{cases}
$$

为什么设成 $-\infty$？

因为 Softmax 中：

$$
e^{-\infty}=0
$$

所以这些位置最终 attention 权重就是 0。

于是前景 attention 为：

$$
\operatorname{Attn}_{\mathrm{fg}}
=
\operatorname{softmax}
\left(
\tilde A_{\mathrm{fg}}
\right)
V_R
\tag{4}
$$

它表示：

> Target 可以从 Reference 读取信息，但只能从 $M_R=1$ 的真实异常区域读取。

论文原式使用 $\mathcal{MF}(M_R=0,-\infty)$ 表示这一 mask-filling 操作。学习时把它理解成“把参考正常区域屏蔽掉”即可，不需要按论文排版中的逐元素乘法字面实现。

### 5.4. 公式（5）：只允许读取正常图中的正常区域

接着计算背景分支。

Target 的正常区域需要读取：

$$
K_N,\quad V_N
$$

但目标掩码：

$$
M_T=1
$$

的区域之后准备生成异常，所以不希望这部分正常特征继续参与背景来源。

因此将这些位置屏蔽：

$$
\tilde A_{\mathrm{bg},i,j}
=
\begin{cases}
A_{\mathrm{bg},i,j},
&
M_{T,j}=0,
\\
-\infty,
&
M_{T,j}=1.
\end{cases}
$$

再计算：

$$
\operatorname{Attn}_{\mathrm{bg}}
=
\operatorname{softmax}
\left(
\tilde A_{\mathrm{bg}}
\right)
V_N
\tag{5}
$$

它表示：

> Target 可以从 Normal 分支读取信息，但只读取目标掩码外的正常内容。

所以两个 mask 的作用一定要区分：

$$
\boxed{
M_R：告诉模型“从参考图哪里拿异常”
}
$$

$$
\boxed{
M_T：告诉模型“目标图哪里要放异常”
}
$$

### 5.5. 公式（6）：把异常输出和正常输出合起来

现在已经得到两份结果：

$$
\operatorname{Attn}_{\mathrm{fg}}
$$

表示从真实异常中读取出来的内容。

$$
\operatorname{Attn}_{\mathrm{bg}}
$$

表示从正常图中读取出来的内容。

最终根据目标掩码 $M_T$ 选择：

$$
\begin{aligned}
\operatorname{Attn}^{\star}_{T,t,l}
={}&
M_T
\odot
\operatorname{Attn}_{\mathrm{fg}}
\\
&+
(1-M_T)
\odot
\operatorname{Attn}_{\mathrm{bg}}.
\end{aligned}
\tag{6}
$$

这个公式其实非常直观。

如果某个目标位置满足：

$$
M_{T,i}=1
$$

那么：

$$
O_i^\star
=
O_i^{\mathrm{fg}}
$$

也就是使用真实异常特征。

如果：

$$
M_{T,i}=0
$$

那么：

$$
O_i^\star
=
O_i^{\mathrm{bg}}
$$

也就是使用正常背景特征。

所以公式（3）到公式（6）合起来就是整个 TriAG：

$$
\boxed{
\text{目标区域内：}
Q_T
\rightarrow
K_R,V_R
}
$$

$$
\boxed{
\text{目标区域外：}
Q_T
\rightarrow
K_N,V_N
}
$$

### 5.6. 保留 Target Query 的作用

假设要把一条参考裂纹迁移到另一张正常榛子的右下方。

如果直接把 Reference 的 $Q_R,K_R,V_R$ 全部搬过去，Target 很容易一起继承：

1. Reference 原来的空间布局。
2. Reference 产品局部形状。
3. Reference 裂纹原来的位置关系。

而 O2MAG 保留：

$$
Q_T
$$

相当于让目标图自己问：

> “我这个位置现在需要什么？”

再用：

$$
Q_TK_R^{\mathrm T}
$$

去寻找最匹配的参考异常特征。

所以它不是把参考裂纹像像素贴片一样直接复制过去，而是：

> 用目标图自己的结构去重新读取和组织参考异常特征。

### 5.7. TriAG 的层与时间设置

论文使用 50 个 denoising steps。

作者观察到产品的大体结构在去噪早期已经形成，因此不是第一步就开始 grafting，而是大约从第 5 个去噪步骤开始。

论文正文写选择 U-Net decoder self-attention 层：

$$
L_S\in\{9,\dots,16\}
$$

补充材料则主要写成 layers 10–16。

这里不用过度纠结编号差 1 的问题，核心思想是：

> 主要修改 U-Net 的 up-block / decoder 中较后面的 self-attention，因为这些层包含更多局部、patch 级纹理信息，更适合迁移裂纹、孔洞等工业异常细节。

---

## 6. Anomaly-Guided Optimization：让文本真正表示这类工业异常

### 6.1. AGO 的问题与目标

TriAG 已经能够从真实参考图中迁移异常特征，但 Stable Diffusion 仍然受到文本条件控制。

论文使用的提示非常简单：

$$
y=
\text{“A photo of a [cls] with a [anomaly type]”}
$$

例如：

$$
y=
\text{“A photo of a hazelnut with a crack”}
$$

问题是 Stable Diffusion 在大规模通用互联网图文数据上训练。

它理解的：

$$
\text{crack}
$$

可能是：

1. 墙上的裂缝。
2. 道路裂缝。
3. 玻璃裂纹。
4. 一般物体破裂。

但 MVTec 中 hazelnut-crack 有自己非常具体的：

1. 裂纹颜色。
2. 裂纹宽度。
3. 外壳纹理。
4. 阴影。
5. 裂纹和榛子表面的结合方式。

所以普通 prompt 得到的文本 embedding：

$$
e_{\mathrm{ori}}
=
\tau_\theta(y)
$$

并不一定适合当前真实工业异常。

AGO 的目标就是：

> 不改 Stable Diffusion，而是把这个文本 embedding 调整到更适合参考异常图的位置。

### 6.2. 从 Stable Diffusion 的普通训练目标理解 AGO

这里不需要推导 Stable Diffusion 的完整扩散公式，只要知道它原本训练时会最小化噪声预测误差：

$$
\mathcal L_{\mathrm{SD}}
=
\mathbb E
\left[
\left\|
\epsilon-
\epsilon_\theta(z_t,t,e)
\right\|_2^2
\right]
$$

其中：

1. $z_t$：带噪 latent。
2. $e$：文本 embedding。
3. $\epsilon$：真实加入的噪声。
4. $\epsilon_\theta$：U-Net 预测出的噪声。

普通 Stable Diffusion 训练时，优化的是模型参数：

$$
\theta
$$

也就是：

$$
\min_\theta
\mathcal L_{\mathrm{SD}}
$$

AGO 做了一个非常关键的变化：

$$
\boxed{
\theta\text{ 完全冻结}
}
$$

然后把优化变量换成：

$$
e
$$

于是得到：

$$
\boxed{
\min_e
\mathcal L_{\mathrm{recon}}
}
$$

这就是公式（7）的核心来源。

### 6.3. 公式（7）到底在做什么

论文写：

$$
e^\star
=
\arg\min_e
\mathbb E_{t,\epsilon}
\left[
\left\|
\epsilon
-
\epsilon_\theta
\left(
x_t,t,e
\right)
\right\|_2^2
\right].
\tag{7}
$$

这里论文写成 $x_t$，理解时仍然把它看成 Stable Diffusion 中的带噪 latent 即可。

整个过程可以这样理解：

1. 从真实参考异常图 $I_R$ 出发。
2. 得到它在扩散过程中的带噪表示。
3. 把当前文本 embedding $e$ 交给冻结的 U-Net。
4. U-Net 根据 $e$ 预测噪声。
5. 如果预测得不好，说明当前文本 embedding 还不能很好地解释这张真实异常图。
6. 计算误差并对 $e$ 求梯度。
7. 只更新 $e$，Stable Diffusion 的 U-Net、VAE、CLIP 全部不动。
8. 重复 500 步，最终得到：

$$
e^\star
$$

所以 AGO 最重要的逻辑是：

$$
\boxed{
\text{让一个冻结的 Stable Diffusion 更容易重建真实参考异常}
}
$$

由于模型本身不能改变，只能修改文本 embedding，于是 $e$ 会逐渐包含更多与这张真实工业异常一致的信息。

论文设置：

$$
N_{\mathrm{AGO}}=500
$$

学习率：

$$
\eta=3\times10^{-3}
$$

优化器使用 Adam。

### 6.4. 重建损失如何完成语义对齐

可以把冻结的 U-Net 写成：

$$
\hat\epsilon
=
F_\theta(z_t,t,e)
$$

其中 $\theta$ 不变。

原始 embedding：

$$
e_{\mathrm{ori}}
$$

如果只表示一个很泛化的“crack”，那么：

$$
F_\theta(z_t,t,e_{\mathrm{ori}})
$$

可能无法准确解释参考异常图。

所以噪声误差较大。

优化后找到：

$$
e^\star
$$

使：

$$
\left\|
\epsilon-
F_\theta(z_t,t,e^\star)
\right\|_2^2
$$

更小。

因此最准确的理解不是：

> AGO 数学上学出了完整“真实异常分布”。

而是：

> AGO 找到了一个更适合让冻结 Stable Diffusion 表达当前真实异常参考的文本条件。

这也是为什么 AGO 必须和 TriAG 一起使用：

1. AGO 提供文本级异常语义。
2. TriAG 直接提供图像级真实异常特征。

### 6.5. 负提示与公式（8）

除了正提示，论文还构造负提示，例如：

1. `no crack`
2. `no scratch`
3. `smooth surface`
4. `intact shell`

本文把 negative prompt 的 embedding 当作 CFG 中的“无条件方向”。

因此可以记成：

$$
\epsilon_n
=
\epsilon_\theta(z_t,t,e_n)
$$

$$
\epsilon_p
=
\epsilon_\theta(z_t,t,e^\star)
$$

最终：

$$
\hat\epsilon_t
=
\epsilon_n
+
g
\left(
\epsilon_p-\epsilon_n
\right)
\tag{8}
$$

其中：

$$
g=7.5
$$

直观上：

1. $e^\star$ 把生成结果推向“真实的目标异常”。
2. $e_n$ 表示“无裂纹、表面完整”等正常方向。
3. CFG 进一步把生成过程从正常方向推向异常方向。

所以如果你想写成：

$$
\hat\epsilon_t
=
\epsilon_{\mathrm{uncond}}
+
g
\left(
\epsilon_p-\epsilon_n
\right)
$$

只有在这里明确：

$$
\epsilon_{\mathrm{uncond}}
=
\epsilon_n
$$

也就是本文把 negative prompt 当成 unconditional embedding 时才完全等价。

### 6.6. AGO 需要注意的一点

论文关于 AGO 的文字、公式（7）和 Algorithm 1 对“具体怎样使用 DDIM inversion 轨迹进行优化”写得并不完全统一。

学习这篇论文时先抓住不变的核心：

$$
\boxed{
\text{冻结模型，只优化 prompt embedding，使其更好重建参考异常}
}
$$

至于严格代码复现时到底采用随机 $t,\epsilon$ 还是固定 inversion trajectory，需要结合作者代码进一步确认。

---

## 7. Dual-Attention Enhancement：让异常真正填进目标掩码

### 7.1. DAE 要解决的问题

做到 AGO + TriAG 后，论文发现仍然会出现一种情况：

目标掩码：

$$
M_T
$$

明明很大，但真正生成出来的异常只占其中一小部分。

也就是：

$$
\operatorname{VisibleAnomalyRegion}(\hat I)
\subsetneq
M_T
$$

这样会带来一个很严重的问题：

> 最终把 $(\hat I,M_T)$ 当成分割训练数据时，mask 说这些位置都是异常，但图像里实际上很多位置还是正常的。

所以 DAE 的目的不是重新设计一个新的生成器，而是在 TriAG 基础上把异常信号进一步强化。

它同时增强：

1. Self-attention：让 Target 更强地读取真实异常特征。
2. Cross-attention：让 Target mask 内的位置更强地关注 anomaly token。

### 7.2. Self-attention Enhancement：公式（9）

TriAG 中已经有：

$$
A_{\mathrm{fg},t,l}
=
\frac{
Q_{T,t,l}K_{R,t,l}^{\mathrm T}
}{
\sqrt d
}
$$

DAE 对它做两件事：

1. 给异常相关位置增加 logit bias。
2. 使用更低的 Softmax temperature。

论文写成：

$$
\begin{aligned}
\hat A_{\mathrm{fg},t,l}
&=
A_{\mathrm{fg},t,l}
+
\log(\gamma)\overline M_R,
\\
\widehat{\operatorname{Attn}}_{\mathrm{fg}}
&=
\operatorname{softmax}
\left(
\frac{
\hat A_{\mathrm{fg},t,l}
}{
\tau_{\mathrm{fg}}
}
+
\mathcal{MF}
\left(
M_R=0,-\infty
\right)
\right)
V_{R,t,l}.
\end{aligned}
\tag{9}
$$

论文设置：

$$
\gamma=1.1
$$

$$
\tau_{\mathrm{fg}}=0.7
$$

先看 $\log(\gamma)$。

如果某个 attention logit 原来为：

$$
s
$$

现在变成：

$$
s+\log\gamma
$$

那么 Softmax 前对应的指数项：

$$
e^{s+\log\gamma}
$$

可以拆成：

$$
e^s e^{\log\gamma}
=
\gamma e^s
$$

因此：

$$
\boxed{
+\log\gamma
\Longleftrightarrow
\text{把对应的未归一化 attention 权重乘上 }\gamma
}
$$

再看 temperature。

普通 Softmax：

$$
\operatorname{softmax}(s)
$$

现在变成：

$$
\operatorname{softmax}
\left(
\frac{s}{\tau_{\mathrm{fg}}}
\right)
$$

因为：

$$
\tau_{\mathrm{fg}}=0.7<1
$$

所以原本不同 logit 之间的差距会被放大。

但有一个工程上的维度问题：

MRM_R 原本可能只是：

MR∈RNRM_R\in\mathbb R^{N_R}

即参考图的 NRN_R 个位置。

而 attention score 是：

Sfg∈RNT×NRS_{\mathrm{fg}} \in \mathbb R^{N_T\times N_R}

它包含：

> 每个 Target Query × 每个 Reference Key。

所以真正计算时，MRM_R 需要先被扩展、广播成和 SS 能对应的形状。

作者写一个：

M‾R\overline M_R

**可能**是想表示“经过某种扩展/处理后的 mask”。


### 7.3. Cross-attention Enhancement：公式（10）

Self-attention 负责真实视觉特征。

Cross-attention 则负责：

> 图像的每个位置到底应该关注哪个文本 token。

例如 prompt：

`A photo of a hazelnut with a crack`

其中 `crack` 对应 anomaly token。

设：

$$
j^\star
$$

表示 anomaly token 的位置。

Cross-attention 中：

$$
(A_{c,t})_{i,j}
$$

表示：

> 第 $i$ 个图像位置对第 $j$ 个文本 token 的注意力。

论文只对 anomaly token 那一列做增强：

$$
\mathcal F
\left(
A_{c,t},M_T
\right)_{i,j}
=
\begin{cases}
C
\cdot
M_{T,i}
\cdot
(A_{c,t})_{i,j},
&
j=j^\star,
\\
(A_{c,t})_{i,j},
&
\text{otherwise}.
\end{cases}
\tag{10}
$$

论文设置：

$$
C=100
$$

如果：

$$
M_{T,i}=1
$$

说明位置 $i$ 在目标异常区域内，那么：

$$
(A_{c,t})_{i,j^\star}
\longrightarrow
100(A_{c,t})_{i,j^\star}
$$

也就是强烈放大 anomaly token。

如果：

$$
M_{T,i}=0
$$

那么异常 token 对该位置的增强被关闭。

所以公式（10）最简单的理解是：

$$
\boxed{
\text{目标 mask 内：强烈告诉模型“这里必须表现 anomaly type”}
}
$$

$$
\boxed{
\text{目标 mask 外：不要增强 anomaly token}
}
$$

### 7.4. Self-attention 与 Cross-attention 的互补作用

如果只增强 self-attention：

> 模型会更强地读取参考图中的局部纹理，但不一定明确知道这些纹理应该表现成哪个异常类型。

如果只增强 cross-attention：

> 模型知道这里应该是 crack，但生成的可能只是 Stable Diffusion 自己理解的普通 crack，而不一定像真实参考异常。

所以两者分别提供：

$$
\text{Self-attention}
\rightarrow
\text{真实异常外观}
$$

$$
\text{Cross-attention}
\rightarrow
\text{异常类型语义}
$$

组合后才是：

$$
\boxed{
\text{既像真实参考异常，又明确表达目标 anomaly type}
}
$$

### 7.5. DAE 与 TriAG 的时间调度

论文总共使用：

$$
50
$$

个 denoising steps。

主要设置为：

$$
T_S=5
$$

即大约从第 5 个去噪步骤开始做 self-attention grafting。

Self-attention enhancement 的时间范围：

$$
\tau_s\in(5,50)
$$

Cross-attention enhancement 的时间范围：

$$
\tau_c\in(20,40)
$$

论文用公式（11）总结 attention editing 的调度：

$$
\operatorname{EDIT}
=
\begin{cases}
\operatorname{TriAG},
&
t>T_S
\text{ 且 }
l\in L_S,
\\
\operatorname{DAE},
&
t\in\tau_s
\text{ 或 }
t\in\tau_c,
\\
\operatorname{SelfAttention}(Q_T,K_T,V_T),
&
\text{其他情况}.
\end{cases}
\tag{11}
$$

这个公式主要是在说明“哪些步骤、哪些层需要编辑 attention”，不用再单独推导。

不过从公式（9）和整个方法流程看，学习时不要把 TriAG 和 DAE 理解成完全互斥的两个方法。更合适的理解是：

1. TriAG 是主干。
2. 到指定时间后，在 TriAG 的前景 self-attention 上叠加 self-attention DAE。
3. 到 $\tau_c$ 时，再额外增强 cross-attention 中的 anomaly token。
4. 其他不需要编辑的位置或步骤使用普通 attention。

---

## 8. 把三个模块重新串成一次完整生成

### 8.1. 一次 O2MAG 生成的完整过程

现在把前面所有内容重新走一遍。

1. 准备输入

真实参考异常：

$$
(I_R,M_R)
$$

正常目标图：

$$
I_N
$$

目标异常区域：

$$
M_T
$$

正提示：

$$
y=
\text{“A photo of a [cls] with a [anomaly type]”}
$$

以及负提示 $y_n$。

2. AGO 优化文本条件

先得到原始 embedding：

$$
e_{\mathrm{ori}}
=
\tau_\theta(y)
$$

冻结整个 Stable Diffusion，仅优化：

$$
e
$$

得到：

$$
e^\star
$$

因此接下来的正条件不再只是普通文本语义，而是更接近参考工业异常。

3. 对 Reference 和 Normal 做 DDIM inversion

得到：

$$
\left\{
Z_T^{\mathrm{ref}},
\dots,
Z_0^{\mathrm{ref}}
\right\}
$$

以及：

$$
\left\{
Z_T^{\mathrm{nor}},
\dots,
Z_0^{\mathrm{nor}}
\right\}
$$

4. 初始化 Target

$$
Z_T^{\mathrm{tar}}
=
Z_T^{\mathrm{nor}}
$$

所以 Target 首先继承正常图的整体结构。

5. 开始逐步去噪

在每一个时间点 $t$：

Reference 分支提供：

$$
K_{R,t,l},\quad V_{R,t,l}
$$

Normal 分支提供：

$$
K_{N,t,l},\quad V_{N,t,l}
$$

Target 分支自己产生：

$$
Q_{T,t,l}
$$

6. TriAG 决定“哪里读异常、哪里读正常”

目标 mask 内：

$$
Q_T
\rightarrow
K_R,V_R
$$

并通过 $M_R$ 保证只能读取真实异常区域。

目标 mask 外：

$$
Q_T
\rightarrow
K_N,V_N
$$

保持正常背景。

两部分再通过 $M_T$ 融合。

7. DAE 进一步增强

Self-attention DAE：

> 让 Target 更集中地读取参考异常特征。

Cross-attention DAE：

> 让目标 mask 内更强地关注 anomaly token。

8. U-Net 输出噪声预测

正条件使用：

$$
e^\star
$$

负条件使用：

$$
e_n
$$

通过 CFG 得到：

$$
\hat\epsilon_t
=
\epsilon_n
+
7.5
\left(
\epsilon_p-\epsilon_n
\right)
$$

9. DDIM 更新 Target latent

$$
Z_{t-1}^{\mathrm{tar}}
=
\operatorname{SampleDDIM}
\left(
Z_t^{\mathrm{tar}},
\hat\epsilon_t,
t
\right)
$$

然后继续下一步。

10. 最终解码

经过所有去噪步骤后得到：

$$
Z_0^{\mathrm{tar}}
$$

VAE decoder：

$$
\hat I
=
\mathcal D
\left(
Z_0^{\mathrm{tar}}
\right)
$$

最终得到：

$$
(\hat I,M_T)
$$

### 8.2. 整篇论文最应该记住的一条信息流

如果把所有细节都先放掉，O2MAG 就是下面这条信息流：

$$
\boxed{
I_R
\rightarrow
\text{提供真实异常}
}
$$

$$
\boxed{
I_N
\rightarrow
\text{提供正常产品和背景}
}
$$

$$
\boxed{
M_R
\rightarrow
\text{告诉模型从 Reference 哪里拿异常}
}
$$

$$
\boxed{
M_T
\rightarrow
\text{告诉模型把异常放到 Target 哪里}
}
$$

$$
\boxed{
y
\xrightarrow{\mathrm{AGO}}
e^\star
\rightarrow
\text{提供更准确的异常文本语义}
}
$$

然后：

$$
\boxed{
Q_T
+
(K_R,V_R)
+
(K_N,V_N)
+
(M_R,M_T)
\rightarrow
\text{TriAG}
}
$$

再加：

$$
\boxed{
\text{DAE}
\rightarrow
\text{把异常进一步强化}
}
$$

最后：

$$
\boxed{
\text{DDIM 多步去噪}
\rightarrow
\hat I
}
$$

这就是整篇论文的方法主线。

---

## 9. 实验：作者怎么证明这套方法有效

### 9.1. 实验设置

论文主要使用三个数据集：

1. MVTec-AD。
2. VisA。
3. Real-IAD。

MVTec-AD 主实验采用 few-shot 设置：

1. 每种异常类型前 $1/3$ 的真实异常图作为参考数据。
2. 剩余 $2/3$ 用于测试。
3. 每类正常图通过简单增强扩充到约 1000 张。
4. 每类生成约 1000 个异常图像—掩码对。
5. 这些合成异常与少量真实异常一起训练下游 U-Net。

主要生成参数：

$$
\text{Stable Diffusion v1.5}
$$

$$
T=50
$$

$$
g=7.5
$$

$$
N_{\mathrm{AGO}}=500
$$

$$
\eta_{\mathrm{AGO}}=3\times10^{-3}
$$

$$
\gamma=1.1,
\qquad
\tau_{\mathrm{fg}}=0.7,
\qquad
C=100
$$

### 9.2. 生成质量

论文使用：

1. KID：越低越好，主要看生成图与真实图特征分布是否接近。
2. IC-LPIPS：越高通常表示生成样本越多样。

MVTec-AD 平均结果：

| 方法 | KID ↓ | IC-LPIPS ↑ |
|---|---:|---:|
| DFMGAN | 62.85 | 0.20 |
| AnomalyDiffusion | 102.67 | 0.29 |
| DualAnoDiff | 103.87 | 0.38 |
| SeaS | 125.67 | 0.35 |
| O2MAG | 45.55 | 0.30 |

O2MAG 的 KID 最低，说明生成结果在 Inception 特征统计上与真实异常更接近。

IC-LPIPS 不是最高，论文认为部分方法的高多样性来自正常背景被破坏，因此不能只看 IC-LPIPS 判断生成质量。

### 9.3. 下游异常检测和定位

作者用各方法生成的数据训练 U-Net。

MVTec-AD 平均结果：

| 方法 | AP-I ↑ | AUROC-P ↑ | AP-P ↑ | F1-P ↑ |
|---|---:|---:|---:|---:|
| DFMGAN | 94.8 | 90.0 | 62.7 | 62.1 |
| AnomalyDiffusion | 99.7 | 99.1 | 81.4 | 76.3 |
| DualAnoDiff | 98.9 | 99.1 | 84.5 | 78.8 |
| SeaS | 99.6 | 98.7 | 83.1 | 78.1 |
| O2MAG | 99.8 | 99.2 | 86.3 | 80.8 |

O2MAG 的主要优势集中在像素级：

$$
\mathrm{AP-P}=86.3
$$

$$
\mathrm{F1-P}=80.8
$$

这与它的方法设计很一致，因为 TriAG、$M_R$、$M_T$ 和 DAE 都主要在解决：

> 异常到底生成在哪里，以及生成区域是否与 mask 一致。

### 9.4. 异常分类

作者还使用生成图训练 ResNet-34 做异常类型分类。

平均准确率：

$$
82.35\%
$$

高于 AnomalyDiffusion 的：

$$
70.25\%
$$

论文认为这说明 AGO + TriAG 生成的异常类型语义更清晰，更接近真实异常。

### 9.5. 消融实验最能说明三个模块的作用

| TriAG | DAE | AGO | AUROC ↑ | AP ↑ | F1-max ↑ |
|---|---|---|---:|---:|---:|
| ✓ |  |  | 99.0 | 81.9 | 77.6 |
| ✓ | ✓ |  | 99.0 | 83.3 | 77.9 |
| ✓ |  | ✓ | 99.1 | 82.7 | 77.7 |
| ✓ | ✓ | ✓ | 99.2 | 86.3 | 80.8 |

这个表最适合用来理解模块分工：

1. TriAG 单独已经能完成基本异常迁移，是整个方法的主干。
2. 加 DAE 后，异常更完整地填入目标区域，因此 AP 提升。
3. 加 AGO 后，异常文本语义和真实感进一步改善。
4. 三者一起效果最好，说明文本语义增强和空间异常增强存在协同作用。

### 9.6. Zero-shot cross-class

论文还测试：

$$
\text{wood-hole}
\rightarrow
\text{hazelnut-hole}
$$

也就是：

> 没有 hazelnut-hole 真实异常参考，只拿 wood-hole 的异常特征去指导正常 hazelnut 生成 hole。

O2MAG-zero 在 hazelnut-hole 上得到：

$$
\mathrm{AP-P}=86.8
$$

$$
\mathrm{F1-P}=80.2
$$

说明同一种异常类型的一部分 self-attention 特征确实可以跨产品迁移。

但它仍低于使用 hazelnut-hole 自身异常作为参考的 O2MAG-origin，因此跨类别并不是没有损失。

---

## 10. 论文的局限与最终理解

### 10.1. 小异常

Self-attention 运行在：

$$
64\times64,
\quad
32\times32,
\quad
16\times16,
\quad
8\times8
$$

等中间特征分辨率上。

如果原图中的异常非常小，下采样以后可能只剩极少数位置，甚至几乎消失。

因此：

1. $M_R$ 在低分辨率下可能变得非常小。
2. Reference K/V 中异常和背景更难分开。
3. PCA 可视化也可能看不到异常。
4. TriAG 很难稳定迁移这种 tiny anomaly。

这是论文明确承认的主要局限之一。

### 10.2. 逻辑异常

O2MAG 更擅长：

1. crack。
2. hole。
3. scratch。
4. stain。

这类局部外观异常。

但对于：

1. transistor misplaced。
2. missing transistor。
3. cable swap。

这类逻辑异常，模型需要理解：

$$
\text{部件}
+
\text{位置}
+
\text{连接关系}
+
\text{数量约束}
$$

而 O2MAG 主要是在迁移 attention 特征，并没有显式的关系推理模块，所以表现较弱。

### 10.3. 跨类别外观泄漏

Reference 的 $V_R$ 不只包含“异常”。

它还可能包含：

1. 来源产品的材质。
2. 来源颜色。
3. 来源局部形状。

所以：

$$
\text{wood-hole}
\rightarrow
\text{hazelnut-hole}
$$

时，有可能把部分木纹一起迁移到榛子上。

这就是论文补充材料讨论的 appearance leakage。

### 10.4. Training-free 应该怎样准确理解

论文把 O2MAG 称为 training-free。

准确地说：

1. 不微调 U-Net。
2. 不训练 VAE。
3. 不训练 CLIP。
4. 不为每个类别保存新的扩散模型。

但 AGO 仍然会对：

$$
e
$$

执行 500 步梯度优化。

所以更准确地理解为：

$$
\boxed{
\text{不训练生成模型参数，但允许推理阶段优化文本条件}
}
$$

### 10.5. 最后把整篇论文压成一句话

O2MAG 的核心不是重新训练 Stable Diffusion 学会某种工业缺陷，而是在冻结模型中重新安排信息流：

> Target 图像用自己的 Query 决定“当前目标位置需要什么”；Reference 图像提供真实异常的 Key/Value；Normal 图像提供正常背景的 Key/Value；$M_R$ 限制“从哪里拿异常”，$M_T$ 限制“把异常放在哪里”；AGO 让文本条件更符合真实工业异常；DAE 再把异常视觉特征和异常 token 同时强化，最终通过多步 DDIM 去噪得到新的异常图像。

最核心的关系可以记成：

$$
\boxed{
Q_T
+
(K_R,V_R)
+
(K_N,V_N)
+
(M_R,M_T)
}
$$

再加：

$$
\boxed{
\text{AGO}
+
\text{DAE}
}
$$

最终得到：

$$
\boxed{
\hat I
}
$$
