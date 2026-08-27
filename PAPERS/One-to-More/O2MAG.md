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


## 3. 理解论文所需的基础概念（妈的不懂得有点多）

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

每个 token 的最终 embedding 不仅包含自身词义，还经过文本 Transformer 融合了上下文。例如 “crack” 在：

> a cracked hazelnut

与：

> a crack in a wall

中的表示可能不同。

### 3.13. 文本提示与文本条件

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

### 3.14. 注意力机制的基本问题

注意力机制要解决的问题是：

> 对当前位置而言，所有可能的信息来源中，哪些最重要？

假设有 $N_q$ 个查询位置和 $N_k$ 个候选来源位置。每个查询都需要对所有来源分配权重。

Attention 可以拆成三步：

1. 计算 Query 与 Key 的匹配分数。
2. 用 Softmax 把分数转成归一化权重。
3. 用权重对 Value 加权求和。

### 3.15. Query、Key、Value 的直观解释

Query、Key、Value 分别记为：

$$
Q,
\quad
K,
\quad
V.
$$

可以类比成图书馆检索。

1. Query：我想找什么。
2. Key：每本书适合回答什么问题。
3. Value：书中真正可读取的内容。

对目标图像某个位置 $i$：

$$
q_i
$$

表示该位置当前需要的信息。

对来源位置 $j$：

$$
k_j
$$

表示来源位置的匹配标签。

点积：

$$
q_i^\top k_j
$$

越大，表示来源位置 $j$ 越适合回答目标位置 $i$ 的需求。

真正被汇总的是：

$$
v_j.
$$

必须特别注意，Value 不是原始 RGB 像素，而是当前 U-Net 层中的学习特征。

### 3.16. Q、K、V 怎样从特征产生

给定隐藏特征：

$$
X
\in
\mathbb R^{B\times N\times C},
$$

通过可学习线性投影：

$$
\begin{aligned}
Q
&=
XW_Q,
\\
K
&=
XW_K,
\\
V
&=
XW_V.
\end{aligned}
$$

其中：

$$
W_Q,
W_K,
W_V
$$

是预训练模型中的参数。O2MAG 不更新这些参数，只改变不同分支中 Q、K、V 的组合方式。

### 3.17. Scaled Dot-Product Attention

论文公式（2）为：

$$
A
=
\operatorname{softmax}
\left(
\frac{QK^\top}{\sqrt{d_k}}
\right).
$$

然后：

$$
\operatorname{Attn}(Q,K,V)
=
AV.
$$

设：

$$
Q
\in
\mathbb R^{N_q\times d_k},
$$

$$
K
\in
\mathbb R^{N_k\times d_k},
$$

$$
V
\in
\mathbb R^{N_k\times d_v}.
$$

则：

$$
QK^\top
\in
\mathbb R^{N_q\times N_k}.
$$

矩阵元素：

$$
(QK^\top)_{i,j}
=
q_i^\top k_j.
$$

每一行对应一个 Query，每一列对应一个 Key。

除以 $\sqrt{d_k}$ 的原因是控制点积数值尺度。如果每个维度近似独立、方差为 1，那么：

$$
\operatorname{Var}
\left(
q_i^\top k_j
\right)
\propto
d_k.
$$

维度越大，点积绝对值越容易变大，Softmax 越容易饱和。缩放后可以使训练和推理更稳定。

### 3.18. Softmax 是什么

对一组 logit：

$$
s_1,s_2,\ldots,s_n,
$$

Softmax 定义为：

$$
p_j
=
\frac{
\exp(s_j)
}{
\sum_{k=1}^{n}
\exp(s_k)
}.
$$

它具有：

$$
p_j>0,
\qquad
\sum_{j=1}^{n}p_j=1.
$$

因此可以把 $p_j$ 理解为归一化注意力权重。

需要注意：

1. Softmax 前的 $s_j$ 叫 logit 或 score。
2. Softmax 后的 $p_j$ 才是归一化权重。
3. 给所有 logit 加同一个常数不会改变 Softmax：

$$
\operatorname{softmax}
\left(
s+c\mathbf 1
\right)
=
\operatorname{softmax}(s).
$$

这个性质对分析论文公式（9）中的 $\log\gamma$ 非常重要。

### 3.19. Self-attention

Self-attention 中，Q、K、V 都来自同一图像分支的空间特征：

$$
\begin{aligned}
Q
&=
XW_Q,
\\
K
&=
XW_K,
\\
V
&=
XW_V.
\end{aligned}
$$

它描述图像内部不同位置之间的关系。

例如，榛子上的某个裂纹像素可能关注：

1. 同一条裂纹的其他位置。
2. 裂纹边缘。
3. 相似的暗色纹理。
4. 榛子外壳结构。
5. 产品轮廓。

论文认为 self-attention 中包含：

1. 全局布局。
2. 物体部分关系。
3. 边界。
4. 局部纹理。
5. 高频 patch 特征。

### 3.20. Cross-attention

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

### 3.21. Self-attention 与 Cross-attention 的分工

可以用下表理解二者。

| 机制 | Query 来源 | Key/Value 来源 | 更擅长表达 | O2MAG 中的作用 |
|---|---|---|---|---|
| Self-attention | 图像特征 | 图像特征 | 空间结构、纹理、边界、位置关系 | 从真实参考图像迁移异常视觉特征 |
| Cross-attention | 图像特征 | 文本 embedding | 图像位置与文本 token 的语义对应 | 强调异常类型及其在掩码内的文本影响 |

O2MAG 不选择二者之一，而是让它们承担不同任务。

### 3.22. 多头注意力

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

### 3.23. Attention mask 与 mask filling

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

代码通常写成：

```python
scores = scores.masked_fill(forbidden_mask, torch.finfo(scores.dtype).min)
weights = torch.softmax(scores, dim=-1)
```

论文将该操作记为：

$$
\mathcal{MF}(\cdot),
$$

即 mask-filling operator。

论文公式（4）和（5）在排版上写成了与 $\mathcal{MF}$ 的逐元素乘法，但从语义上必须理解为 masked fill 或 additive mask，不能真的把一般 logit 与 $-\infty$ 相乘，否则可能产生正无穷或 NaN。

### 3.24. Key 轴掩码与 Query 轴门控的区别

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

### 3.25. Logit bias

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

### 3.26. Softmax temperature

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

例如，两个 logit 为：

$$
[2,1].
$$

普通 Softmax 约为：

$$
[0.731,0.269].
$$

若 $\tau=0.5$：

$$
[2,1]/0.5
=
[4,2],
$$

Softmax 约为：

$$
[0.881,0.119].
$$

因此，小温度使较大分数更占优势。论文在 DAE 中使用：

$$
\tau_{\mathrm{fg}}=0.7.
$$


### 3.27. DDPM、DDIM 与采样步数

DDPM 通常使用带随机性的反向采样。DDIM 提出了一条可以使用更少步骤、并且在特定设置下确定性的采样路径。

需要区分两个时间尺度：

1. 训练扩散时间步。

Stable Diffusion 的训练 scheduler 常包含很多原始时间点，例如 1000 个。

2. 推理采样步数。

论文使用 50 个 denoising steps。Scheduler 会从原始时间轴中选出 50 个时间点进行采样。

所以“第 5 个去噪步骤”和“训练时间编号 $t=5$”不是同一个概念。论文在文字、公式和算法中对 $t$ 的方向存在一定歧义，复现时最好单独维护采样迭代编号：

$$
k
=
1,2,\ldots,50.
$$

### 3.28. DDIM 采样公式

由噪声预测得到干净 latent 估计：

$$
\hat z_0
=
\frac{
z_t
-
\sqrt{1-\bar\alpha_t}
\hat\epsilon_t
}{
\sqrt{\bar\alpha_t}
}.
$$

确定性 DDIM，即 $\eta=0$ 时，可以写为：

$$
\begin{aligned}
z_{t-1}
={}&
\sqrt{\bar\alpha_{t-1}}
\hat z_0
\\
&+
\sqrt{1-\bar\alpha_{t-1}}
\hat\epsilon_t.
\end{aligned}
$$

将 $\hat z_0$ 代入：

$$
\begin{aligned}
z_{t-1}
={}&
\sqrt{\bar\alpha_{t-1}}
\frac{
z_t
-
\sqrt{1-\bar\alpha_t}
\hat\epsilon_t
}{
\sqrt{\bar\alpha_t}
}
\\
&+
\sqrt{1-\bar\alpha_{t-1}}
\hat\epsilon_t.
\end{aligned}
$$

DDIM 的重要特性是，在确定性设置和相同条件下，给定某个高噪声 latent，采样路径可以相对稳定地得到对应图像。

### 3.29. DDIM inversion 是什么

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

### 3.30. “反演噪声”为什么仍然包含图像信息

很多初学者会认为：

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

### 3.31. Classifier-free guidance

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

当 $g>1$ 时，模型不是在两者之间简单插值，而是从无条件预测向条件预测方向外推。

论文设置：

$$
g=7.5.
$$

### 3.32. Negative prompt

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

### 3.33. 参数、梯度与优化变量

神经网络中的参数记为：

$$
\theta.
$$

损失函数：

$$
\mathcal L(\theta).
$$

梯度：

$$
\nabla_\theta\mathcal L
$$

表示损失对参数各维度的变化率。

最简单的梯度下降更新为：

$$
\theta
\leftarrow
\theta
-
\eta
\nabla_\theta\mathcal L,
$$

其中 $\eta$ 是学习率。

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

### 3.34. Adam 优化器

论文在 AGO 中使用 Adam，学习率：

$$
\eta
=
3\times10^{-3}.
$$

Adam 不直接使用当前梯度，而是维护一阶和二阶动量：

$$
\begin{aligned}
g_k
&=
\nabla_e\mathcal L_k,
\\
m_k
&=
\beta_1m_{k-1}
+
(1-\beta_1)g_k,
\\
v_k
&=
\beta_2v_{k-1}
+
(1-\beta_2)g_k^2.
\end{aligned}
$$

偏差修正：

$$
\hat m_k
=
\frac{m_k}{1-\beta_1^k},
$$

$$
\hat v_k
=
\frac{v_k}{1-\beta_2^k}.
$$

更新：

$$
e_{k+1}
=
e_k
-
\eta
\frac{
\hat m_k
}{
\sqrt{\hat v_k}
+
\epsilon_{\mathrm{Adam}}
}.
$$

论文没有给出 Adam 的 $\beta_1,\beta_2$ 等配置。没有作者代码时，常见复现会先采用框架默认值，但这属于复现选择，不是论文明确参数。

### 3.35. PCA 是什么

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

第一主成分：

$$
u_1
=
\arg\max_{\|u\|=1}
\operatorname{Var}
\left(
Xu
\right).
$$

后续主成分在与前面方向正交的条件下继续最大化方差。

论文把 self-attention 高维特征压缩到前三个主成分，再映射到 RGB：

$$
\text{高维 attention 表示}
\longrightarrow
\mathbb R^3
\longrightarrow
\text{RGB 颜色}.
$$

相似颜色表示对应空间位置拥有相似 attention 模式。

### 3.36. 论文中的 self-attention PCA 可视化如何理解

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

论文没有在公式中完整规定 PCA 的数据组织、head 聚合和归一化方式，所以这里是最符合图 3、图 7 和图 14 的常见解释，而不是唯一代码实现。

PCA 图中的“前景与背景可分”是定性观察，不是严格证明。尤其当异常很小时，异常区域贡献的方差远小于背景，前三个主成分可能忽略异常。

### 3.37. 概率分布与数据分布

一张图像可以看作高维随机变量 $X$ 的一次取值：

$$
X=x.
$$

真实数据分布：

$$
p_{\mathrm{real}}(x)
$$

描述真实生产环境中各种图像模式的概率密度。

生成分布：

$$
p_{\mathrm{generated}}(x)
$$

描述反复运行生成器时各种输出模式的概率密度。

在异常生成中，更准确的是条件分布：

$$
p
\left(
x
\mid
c,a,l,s,g,t,b
\right),
$$

其中可以包含：

1. 产品类别 $c$。
2. 异常类型 $a$。
3. 异常位置 $l$。
4. 异常大小 $s$。
5. 几何形状 $g$。
6. 纹理 $t$。
7. 背景与产品外观 $b$。

“分布偏移”可能发生在：

1. 异常纹理。
2. 异常大小。
3. 异常位置。
4. 异常与产品的关系。
5. 正常背景。
6. 图像与掩码的一致性。
7. 异常类型多样性。

### 3.38. 模式与 mode collapse

“模式”可以理解为分布中的一种典型形态。

真实 crack 可能包含：

$$
\left\{
\text{细裂纹},
\text{宽裂纹},
\text{分叉裂纹},
\text{边缘裂纹}
\right\}.
$$

如果生成器只会生成细裂纹，就遗漏了其他模式。

GAN 在少样本环境中可能发生 mode collapse，即大量输入都产生近似输出。扩散模型通常具有更稳定的训练和更好的覆盖能力，但当条件、参考和掩码固定时，确定性 DDIM 也可能产生高度相似的结果。

O2MAG 的多样性主要可能来自：

1. 不同正常图像。
2. 不同目标掩码。
3. 不同参考异常。
4. 图像增强。
5. 采样中的随机性。
6. 扩散模型对局部特征重新组合的能力。

### 3.39. KID

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

### 3.40. LPIPS 与 IC-LPIPS

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

### 3.41. AUROC

对不同阈值计算：

$$
\operatorname{TPR}
=
\frac{TP}{TP+FN},
$$

$$
\operatorname{FPR}
=
\frac{FP}{FP+TN}.
$$

以 FPR 为横轴、TPR 为纵轴得到 ROC 曲线，曲线下面积为 AUROC。

1. AUROC-I：图像级 AUROC。
2. AUROC-P：像素级 AUROC。

AUROC 衡量排序能力，对类别不平衡有一定鲁棒性，但当正常像素极多时，像素 AUROC 可能仍然显得很高，所以还需要 AP 和 F1。

### 3.42. Precision、Recall、AP 与 F1-max

Precision：

$$
\operatorname{Precision}
=
\frac{TP}{TP+FP}.
$$

Recall：

$$
\operatorname{Recall}
=
\frac{TP}{TP+FN}.
$$

AP 是 Precision-Recall 曲线下的面积。

1. AP-I：图像级平均精确率。
2. AP-P：像素级平均精确率。

F1：

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

### 3.43. 分割模型与分类模型在论文中的角色

论文使用合成数据训练：

1. U-Net segmentation model，用于异常检测和像素定位。
2. ResNet-34 classifier，用于异常类型分类。

因此论文的评估逻辑是：

$$
\text{生成数据质量}
\longrightarrow
\text{下游模型训练效果}.
$$

如果合成异常具有真实外观、正确类型和准确掩码，下游模型应当更容易在真实测试集上取得较好性能。

但下游性能受到多个因素共同影响：

1. 合成数据质量。
2. 真实少样本是否混入训练。
3. U-Net 或 ResNet 的训练超参数。
4. 数据增强。
5. 掩码来源。
6. 不同类别的难度。

因此，较好下游结果支持生成数据有用，但不是对每张生成图视觉真实性的严格证明。


## 4. O2MAG 的整体框架

### 4.1. 输入、输出与符号总览

一次生成需要：

$$
\mathcal X
=
\left(
I_R,
M_R,
I_N,
M_T,
y,
y_n
\right).
$$

输出：

$$
\hat I.
$$

各变量含义如下。

| 符号 | 含义 | 主要作用 |
|---|---|---|
| $I_R$ | 参考异常图像 | 提供真实异常外观 |
| $M_R$ | 参考异常掩码 | 指定参考图像中哪些位置是异常来源 |
| $I_N$ | 正常图像 | 提供目标产品、正常纹理与背景 |
| $M_T$ | 目标异常掩码 | 指定最终异常位置和大致形状 |
| $y$ | 正提示 | 指定产品类别和异常类型 |
| $y_n$ | 负提示 | 描述应远离的正常外观 |
| $e_{\mathrm{ori}}$ | 原始文本 embedding | CLIP 对正提示的编码 |
| $e^\star$ | AGO 优化后的 embedding | 更适合参考工业异常的文本条件 |
| $Z_t^{\mathrm{ref}}$ | 参考分支第 $t$ 步 latent | 提供参考异常中间特征 |
| $Z_t^{\mathrm{nor}}$ | 正常分支第 $t$ 步 latent | 提供正常背景中间特征 |
| $Z_t^{\mathrm{tar}}$ | 目标分支第 $t$ 步 latent | 最终被逐步去噪成 $\hat I$ |

### 4.2. 三个扩散分支

图 2 是整篇论文最重要的流程图。它包含三条扩散路径。

1. 参考异常分支。

输入：

$$
I_R.
$$

通过 VAE 编码和 DDIM inversion 得到：

$$
\left\{
Z_T^{\mathrm{ref}},
Z_{T-1}^{\mathrm{ref}},
\ldots,
Z_0^{\mathrm{ref}}
\right\}.
$$

在指定 U-Net self-attention 层中提取：

$$
Q_R,
K_R,
V_R.
$$

O2MAG 最终主要使用参考分支的：

$$
K_R,
V_R.
$$

2. 正常图像分支。

输入：

$$
I_N.
$$

通过 inversion 得到：

$$
\left\{
Z_T^{\mathrm{nor}},
Z_{T-1}^{\mathrm{nor}},
\ldots,
Z_0^{\mathrm{nor}}
\right\}.
$$

提取：

$$
Q_N,
K_N,
V_N.
$$

目标分支主要使用正常分支的：

$$
K_N,
V_N.
$$

3. 目标异常分支。

它不是从随机噪声开始，而是复制正常反演噪声：

$$
Z_T^{\mathrm{tar}}
=
Z_T^{\mathrm{nor}}.
$$

在每个去噪时间步，目标分支产生自己的：

$$
Q_T,
K_T,
V_T.
$$

TriAG 保留：

$$
Q_T,
$$

但将目标自注意力中的来源改成：

$$
(K_R,V_R)
$$

或者：

$$
(K_N,V_N).
$$

### 4.3. 为什么需要三个分支，而不是两个分支

假设只保留参考异常分支和目标分支。

目标掩码内可以从参考异常读取特征，但掩码外如果使用目标分支自身的 K、V，目标分支已经受到文本异常语义和前景编辑影响，正常背景可能逐步漂移。

正常图像分支的作用是，在整个去噪过程中持续提供一条接近原正常图像的特征轨迹：

$$
(K_N,V_N).
$$

因此三分支分别承担：

1. Reference：异常内容来源。
2. Normal：正常内容来源。
3. Target：空间结构、接收位置和最终输出载体。

### 4.4. 为什么目标分支必须使用自己的 Query

假设目标位置 $i$ 的 Query 为：

$$
q_{T,i}.
$$

它编码当前目标图像在该位置的结构需求。例如：

1. 该位置位于榛子的左侧还是右侧。
2. 周围是曲面还是边缘。
3. 当前 latent 已形成怎样的局部轮廓。
4. 目标掩码附近的上下文是什么。

参考图像位置 $j$ 提供：

$$
k_{R,j},
\quad
v_{R,j}.
$$

匹配：

$$
q_{T,i}^\top k_{R,j}
$$

表示目标位置 $i$ 与参考异常位置 $j$ 的兼容程度。

如果把 Query 也替换成参考 Query，那么空间组织更容易跟随参考图像，导致：

1. 参考异常原位置被复制。
2. 参考物体轮廓泄漏。
3. 目标掩码的空间控制变弱。
4. 跨类别迁移时来源类别结构混入目标图像。

因此，论文采用：

$$
\boxed{
\text{目标 Query 决定“在目标哪里、需要什么”}
}
$$

$$
\boxed{
\text{来源 Key 决定“从来源哪里取”}
}
$$

$$
\boxed{
\text{来源 Value 决定“实际取回什么内容”}
}.
$$

### 4.5. O2MAG 的完整时间顺序

一次生成可以按以下顺序理解。

1. 准备参考异常图像和掩码：

$$
(I_R,M_R).
$$

2. 准备正常图像：

$$
I_N.
$$

3. 准备目标异常掩码：

$$
M_T.
$$

4. 构造正提示和负提示：

$$
y,
\quad
y_n.
$$

5. 使用 CLIP 得到原始正提示 embedding：

$$
e_{\mathrm{ori}}
=
\tau_\theta(y).
$$

6. 执行 AGO，得到：

$$
e^\star.
$$

7. 对 $I_R$ 执行 DDIM inversion，保存参考轨迹。

8. 对 $I_N$ 执行 DDIM inversion，保存正常轨迹。

9. 复制正常高噪声 latent 初始化目标分支：

$$
Z_T^{\mathrm{tar}}
=
Z_T^{\mathrm{nor}}.
$$

10. 对每个去噪步骤同步运行三条分支。

11. 在指定层和指定时间启用 TriAG。

12. 在指定时间范围启用 self-attention DAE。

13. 在指定时间范围启用 cross-attention DAE。

14. 使用优化后的正提示和负提示执行 CFG 噪声预测。

15. 使用 DDIM 从 $Z_t^{\mathrm{tar}}$ 更新到 $Z_{t-1}^{\mathrm{tar}}$。

16. 最终用 VAE decoder 解码：

$$
\hat I
=
\mathcal D
\left(
Z_0^{\mathrm{tar}}
\right).
$$

17. 将 $(\hat I,M_T)$ 作为下游训练对。

### 4.6. 三个模块分别解决什么问题

| 问题 | 仅使用普通 Stable Diffusion 时的表现 | O2MAG 模块 |
|---|---|---|
| 通用文本异常与真实工业异常不一致 | “crack”可能生成不符合数据集的裂纹 | AGO |
| 文本难以表达真实局部纹理 | 缺陷细节粗糙或错误 | TriAG 的参考 K/V |
| 生成异常时正常背景变化 | 产品颜色、形状或纹理漂移 | 正常分支 + 目标正常初始化 |
| 参考异常和参考背景混在一起 | 目标位置可能读取到参考正常区域 | $M_R$ Key mask |
| 异常扩散到目标掩码外 | 正常区域出现伪缺陷 | $M_T$ 输出门控与背景来源约束 |
| 异常只占目标掩码一小部分 | 图像与标签不一致 | DAE |
| 目标布局被参考图替代 | 像复制粘贴或姿态漂移 | 保留 $Q_T$ |

### 4.7. 图 2 应当怎样逐部分阅读

论文第 3 页图 2 中包含多种箭头和模块，建议按以下顺序阅读。

1. 左上角参考异常图像。

它经过 DDIM inversion，得到参考分支的 latent 序列。该分支在每个采样步骤中提供参考异常的 self-attention Key、Value。

2. 左下角正常图像。

它也经过 DDIM inversion。正常分支一方面提供目标初始化噪声，另一方面在掩码外持续提供背景 Key、Value。

3. 中间目标分支。

目标分支从正常噪声复制开始，通过多步去噪生成最终图像。

4. 中部 self-attention grafting。

目标 Query 分别与参考 Key 和正常 Key 计算注意力，再通过两个掩码决定来源和输出区域。

5. 上方 AGO。

正提示经过文本编码后，不直接固定使用，而是利用参考异常图像重建损失进行 embedding 优化。

6. 右上角 enhanced cross-attention。

目标掩码内部的 anomaly token 权重被显著放大。

7. 中部 DAE self-attention。

参考异常前景注意力的 logit 和温度被调整，使目标异常位置更集中地读取异常特征。

8. 最右侧输出。

最终 latent 经 VAE decoder 得到异常图像。

### 4.8. 三分支是否真的同时“生成三张图”

从概念图看，三个分支都经过 U-Net，但它们的角色不同。

1. 参考和正常分支主要沿预先反演得到的 latent 轨迹，在每个时间点提取中间 Q、K、V。
2. 目标分支的 latent 会根据编辑后的噪声预测逐步更新。
3. 最终只解码目标分支作为输出。

算法 1 保存了参考和正常分支的完整 inversion 序列：

$$
\left\{
Z_T,Z_{T-1},\ldots,Z_0
\right\}.
$$

所以参考和正常分支可以理解为“同步特征提供器”，而不是需要把它们的输出图像作为最终结果。

### 4.9. 为什么必须在相同噪声时间点交换特征

不同时间步的特征语义不同。

1. 高噪声阶段更多决定全局布局和粗结构。
2. 低噪声阶段更多恢复边缘、纹理和细节。

如果目标分支在时间 $t$ 的 Query 与参考分支时间 $s\neq t$ 的 Key 比较，二者的噪声尺度和特征层级可能不匹配。

因此，算法在每个 $t$ 使用：

$$
Q_{T,t,l},
\quad
K_{R,t,l},
\quad
V_{R,t,l},
\quad
K_{N,t,l},
\quad
V_{N,t,l}.
$$

同步时间点有助于保证这些表示处于相近生成阶段。

### 4.10. 为什么还需要文本条件

即使已经从真实异常图像读取 K、V，文本条件仍有四个作用。

1. 指定物体类别。
2. 指定异常类型。
3. 调用 Stable Diffusion 中已有的生成先验。
4. 通过 cross-attention 约束哪些空间位置表达异常 token。

TriAG 提供的是图像级局部信息；文本条件提供的是高层语义。二者并不重复。

### 4.11. 为什么还需要目标掩码

目标掩码同时承担四种功能。

1. 定义输出异常区域。
2. 在正常分支中屏蔽将被异常替换的区域。
3. 在最终 self-attention 输出中选择前景或背景分支。
4. 在 DAE cross-attention 中决定 anomaly token 只在哪些图像位置被增强。

因此：

$$
M_T
$$

不是仅在最后生成完图以后作为标签保存，而是直接参与生成过程。

### 4.12. 目标掩码从哪里来

O2MAG 本身主要研究“给定目标掩码后怎样生成异常”，并不提出一个全新的掩码生成器。

论文实验使用多种掩码来源：

1. MVTec-AD 主实验：AnomalyDiffusion 生成的掩码。
2. VisA 与 Real-IAD：SeaS 生成的掩码。
3. 补充消融：Perlin noise。
4. 补充消融：Nano Banana 生成掩码。
5. 其他方法生成的掩码。

MVTec-AD 中，AnomalyDiffusion 有时产生全零掩码，所以作者生成 1200 个候选掩码，过滤空掩码后保留 1000 个。

这说明 O2MAG 的完整数据生产管线仍依赖一个外部掩码来源。它对不同来源表现出一定鲁棒性，但不是完全自包含的“图像和掩码共同生成器”。

### 4.13. 正常图像的数据增强

论文补充材料说明，MVTec-AD 每类原始正常训练图像约 200–400 张，toothbrush 只有约 60 张。作者通过：

1. 平移。
2. 翻转。
3. 旋转。

把每类正常图像扩充到：

$$
1000
$$

张。

对于 zipper、capsule 等方向敏感类别，只使用翻转，以避免旋转破坏正常方向语义。

这意味着生成数据的背景多样性相当一部分来自正常图像增强，而不是单纯来自扩散随机性。

### 4.14. 一个具体生成例子

假设要生成一张新的 hazelnut-crack 图像。

输入：

1. $I_R$：一张真实裂纹榛子。
2. $M_R$：真实裂纹位置。
3. $I_N$：一张正常榛子。
4. $M_T$：在正常榛子右下区域的一条目标掩码。
5. $y$：A photo of a hazelnut with a crack。
6. $y_n$：no crack, smooth surface。

生成过程：

1. AGO 调整正提示 embedding，使其更适合这张真实榛子裂纹。
2. Reference inversion 建立真实裂纹图像的 latent 轨迹。
3. Normal inversion 建立正常榛子的 latent 轨迹。
4. Target 从正常榛子高噪声 latent 初始化。
5. 前几个去噪步骤先保留正常榛子的整体结构。
6. 开始 TriAG 后，目标右下掩码位置使用自己的 Query 去匹配参考裂纹掩码内的 Key。
7. 对应 Value 把裂纹纹理带入目标位置。
8. 掩码外位置使用正常分支 K、V，尽量保留原榛子表面。
9. DAE 增强掩码内 anomaly token 和参考异常注意力。
10. 多步 DDIM 去噪后得到新裂纹榛子。

最终结果不是把参考裂纹原像素贴到右下角，而是让扩散模型根据目标上下文重新生成一条受参考异常引导的新裂纹。


## 5. Stable Diffusion 基础公式与公式（1）的推导

### 5.1. 论文公式（1）

论文第 3 页给出 Stable Diffusion 的训练目标：

$$
\begin{aligned}
\mathcal L_{\mathrm{SD}}
:=
\mathbb E_{
\mathcal E(x),
y,
\epsilon\sim\mathcal N(0,1),
t
}
\Big[
&
\left\|
\epsilon
-
\epsilon_\theta
\left(
z_t,
t,
\tau_\theta(y)
\right)
\right\|_2^2
\Big].
\end{aligned}
\tag{1}
$$

这不是 O2MAG 提出的新公式，而是 Latent Diffusion / Stable Diffusion 使用的标准噪声预测目标。论文引用的基础工作是 Latent Diffusion Models。

各变量含义：

1. $x$：真实 RGB 图像。
2. $\mathcal E(x)$：VAE encoder 得到的干净 latent，通常记为 $z_0$。
3. $y$：文本条件。
4. $\tau_\theta(y)$：CLIP text encoder 输出的文本 embedding。
5. $t$：随机扩散时间步。
6. $\epsilon$：随机采样的真实高斯噪声。
7. $z_t$：在时间 $t$ 加噪后的 latent。
8. $\epsilon_\theta(\cdot)$：U-Net 预测噪声。
9. $\theta$：模型参数。
10. $\mathbb E$：对图像、文本、时间步和噪声采样取期望。

### 5.2. 为什么前向扩散可以一步写到任意时间

单步过程为：

$$
z_t
=
\sqrt{\alpha_t}
z_{t-1}
+
\sqrt{1-\alpha_t}
\epsilon_t.
$$

将 $z_{t-1}$ 继续展开：

$$
\begin{aligned}
z_t
={}&
\sqrt{\alpha_t}
\left(
\sqrt{\alpha_{t-1}}
z_{t-2}
+
\sqrt{1-\alpha_{t-1}}
\epsilon_{t-1}
\right)
\\
&+
\sqrt{1-\alpha_t}
\epsilon_t.
\end{aligned}
$$

整理：

$$
\begin{aligned}
z_t
={}&
\sqrt{\alpha_t\alpha_{t-1}}
z_{t-2}
\\
&+
\sqrt{\alpha_t(1-\alpha_{t-1})}
\epsilon_{t-1}
\\
&+
\sqrt{1-\alpha_t}
\epsilon_t.
\end{aligned}
$$

多个独立高斯变量的线性组合仍然是高斯变量，所以所有噪声项可以合并成一个新的标准高斯变量 $\epsilon$。不断展开到 $z_0$ 后得到：

$$
z_t
=
\sqrt{\bar\alpha_t}
z_0
+
\sqrt{1-\bar\alpha_t}
\epsilon.
$$

其中：

$$
\bar\alpha_t
=
\prod_{s=1}^{t}
\alpha_s.
$$

这使训练非常方便：不需要真的从 $z_0$ 逐步加噪 $t$ 次，只需随机采样一个 $t$ 并直接构造 $z_t$。

### 5.3. 为什么损失使用平方误差

构造 $z_t$ 时，真实噪声 $\epsilon$ 已知。网络输出：

$$
\hat\epsilon
=
\epsilon_\theta(z_t,t,e).
$$

最小化均方误差：

$$
\begin{aligned}
\mathcal L
&=
\left\|
\epsilon-\hat\epsilon
\right\|_2^2
\\
&=
\sum_r
\left(
\epsilon_r-\hat\epsilon_r
\right)^2.
\end{aligned}
$$

如果把噪声条件分布近似为高斯，最小化平方误差也可以理解为最大化相应高斯似然。

更深层的扩散理论从变分下界开始。生成模型的负对数似然上界可以分解为多个 KL 散度：

$$
\begin{aligned}
\mathcal L_{\mathrm{VLB}}
=
\mathbb E_q
\Big[
&
D_{\mathrm{KL}}
\left(
q(z_T\mid z_0)
\,\|\,
p(z_T)
\right)
\\
&+
\sum_{t=2}^{T}
D_{\mathrm{KL}}
\left(
q(z_{t-1}\mid z_t,z_0)
\,\|\,
p_\theta(z_{t-1}\mid z_t,y)
\right)
\\
&-
\log
p_\theta(z_0\mid z_1)
\Big].
\end{aligned}
$$

在固定反向方差并采用噪声参数化后，中间 KL 项可以化成带权的噪声预测误差。实践中常使用简化目标：

$$
\mathcal L_{\mathrm{simple}}
=
\mathbb E
\left[
\left\|
\epsilon-\epsilon_\theta(z_t,t,e)
\right\|_2^2
\right].
$$

论文公式（1）就是这一简化目标的 latent diffusion 版本。

### 5.4. 条件文本怎样进入噪声预测

文本 embedding：

$$
e
=
\tau_\theta(y)
$$

不会简单拼接到 latent 上。它主要通过 U-Net 内的 cross-attention 注入。

在某个 cross-attention 层：

$$
Q
=
XW_Q,
$$

$$
K
=
eW_K,
$$

$$
V
=
eW_V.
$$

于是不同图像位置根据自己的 Query，对文本 token 分配注意力权重。文本最终影响 U-Net 的中间空间特征，进而影响噪声预测。

### 5.5. 公式（1）与 O2MAG 的关系

公式（1）承担两个作用。

1. 它解释了 Stable Diffusion 原本如何训练。

O2MAG 使用的 U-Net 已经通过这一类目标预训练完成。

2. 它为 AGO 的公式（7）提供形式基础。

Stable Diffusion 训练时优化：

$$
\theta.
$$

AGO 则冻结：

$$
\theta
$$

并优化：

$$
e.
$$

也就是说，AGO 不是发明新的重建原理，而是把同样的噪声预测误差对另一个变量求梯度。

### 5.6. 论文记号中的一个小问题

论文公式（1）使用：

$$
z_t.
$$

公式（7）和公式（8）又使用：

$$
x_t.
$$

但论文明确基于 Stable Diffusion latent space，因此方法公式中的 $x_t$ 应当理解为当前带噪 latent，而不是原始 RGB 像素图。复现时建议统一记为：

$$
z_t.
$$

这样可以避免把 VAE 前后的空间混淆。

## 6. Tri-branch Attention Grafting 超详细解析

### 6.1. TriAG 要解决的核心矛盾

目标图像需要同时满足两个相反要求。

1. 目标掩码内必须改变：

$$
\hat I_{M_T}
\neq
(I_N)_{M_T}.
$$

2. 目标掩码外应尽量不变：

$$
\hat I_{\overline{M_T}}
\approx
(I_N)_{\overline{M_T}}.
$$

如果只使用参考异常特征，背景可能被来源图像污染；如果只从正常图像去噪，异常又不明显。

TriAG 的解决方式是把目标 self-attention 输出分成两个来源：

$$
\text{异常输出}
\quad+
\text{正常输出}.
$$

再通过目标掩码在目标位置上融合。

### 6.2. 论文使用 PCA 观察到了什么

论文第 4 页图 3 展示：

1. 迭代去噪的中间重建结果。
2. 第 30 个采样步骤的 self-attention PCA 可视化。

作者观察到：

1. 物体整体布局在去噪较早阶段已经形成。
2. 异常前景和正常背景在 self-attention 特征空间中往往表现为不同颜色区域。
3. 相似结构区域具有相似的 attention 表示。

这为以下假设提供定性依据：

$$
\text{参考异常的 self-attention K/V 中包含可迁移异常信息}.
$$

但必须注意：

1. PCA 只显示前三个方差方向。
2. 可分颜色不等于严格线性可分。
3. 不同 attention head 可能有不同语义。
4. 很小的异常可能在 PCA 中完全不可见。
5. 这是设计动机，不是理论证明。

### 6.3. 标准目标 self-attention

如果不做 O2MAG 编辑，目标分支在时间 $t$、层 $l$ 的 self-attention 为：

$$
A_{T,t,l}
=
\frac{
Q_{T,t,l}
K_{T,t,l}^\top
}{
\sqrt d
}.
$$

归一化并读取 Value：

$$
O_{T,t,l}
=
\operatorname{softmax}
\left(
A_{T,t,l}
\right)
V_{T,t,l}.
$$

这意味着目标分支只能从自身当前特征中组织信息，不能直接读取真实参考异常的中间视觉特征。

### 6.4. 跨分支组合

TriAG 改成两组 attention。

异常分支组合：

$$
\left(
Q_T,
K_R,
V_R
\right).
$$

正常分支组合：

$$
\left(
Q_T,
K_N,
V_N
\right).
$$

注意：

1. 两组都使用 $Q_T$。
2. 异常组使用参考分支 $K_R,V_R$。
3. 背景组使用正常分支 $K_N,V_N$。

因此，它在 self-attention 模块中实现了跨分支 mutual attention。

### 6.5. 公式（3）

论文公式（3）为：

$$
\begin{aligned}
A_{\mathrm{fg},t,l}
&=
\frac{
Q_{T,t,l}
K_{R,t,l}^{\top}
}{
\sqrt d
},
\\
A_{\mathrm{bg},t,l}
&=
\frac{
Q_{T,t,l}
K_{N,t,l}^{\top}
}{
\sqrt d
}.
\end{aligned}
\tag{3}
$$

其中：

1. $\mathrm{fg}$ 表示 foreground anomaly。
2. $\mathrm{bg}$ 表示 background normal。
3. $t$ 表示去噪时间步。
4. $l$ 表示 U-Net attention 层。
5. $d$ 表示单个 attention head 的维度。

论文排版中的第一项有时省略了 $K_R$ 的时间下标，但从同步三分支逻辑和算法 1 看，应当使用当前时间点的参考 Key：

$$
K_{R,t,l}.
$$

### 6.6. 公式（3）的张量形状

设第 $l$ 层空间分辨率为：

$$
h_l\times w_l.
$$

则：

$$
N_l
=
h_lw_l.
$$

单个 batch、单个 attention head 下：

$$
Q_{T,t,l}
\in
\mathbb R^{N_l\times d},
$$

$$
K_{R,t,l},
K_{N,t,l}
\in
\mathbb R^{N_l\times d}.
$$

所以：

$$
A_{\mathrm{fg},t,l},
A_{\mathrm{bg},t,l}
\in
\mathbb R^{N_l\times N_l}.
$$

矩阵语义：

```text
行 i：目标图像位置 i
列 j：来源图像位置 j
A[i,j]：目标位置 i 对来源位置 j 的匹配分数
```

完整多头形状通常为：

$$
A
\in
\mathbb R^{B\times H_a\times N_l\times N_l}.
$$

### 6.7. 为什么不直接把参考 K、V 拼接到目标 K、V

一种可能方案是：

$$
K
=
[K_T;K_R],
\qquad
V
=
[V_T;V_R].
$$

但这会让目标 Query 在自身内容和参考内容之间自由竞争，容易出现：

1. 异常区域仍主要关注目标正常特征，异常不明显。
2. 正常区域关注参考异常，缺陷泄漏。
3. 参考背景被错误读取。

O2MAG 选择分别计算异常和背景 attention，再用掩码明确分工，比简单拼接更可控。

### 6.8. 公式（4）：参考异常前景读取

论文公式（4）为：

$$
\begin{aligned}
\operatorname{Attn}_{\mathrm{fg}}
=
\operatorname{softmax}
\Big(
&A_{\mathrm{fg},t,l}
\odot
\mathcal{MF}
\left(
M_R=0,-\infty
\right)
\Big)
\\
&\cdot
V_{R,t,l}.
\end{aligned}
\tag{4}
$$

它的真正语义是：

> 目标位置可以查询参考分支，但只能查询参考掩码 $M_R=1$ 的异常位置。

更规范地定义加法掩码：

$$
B^R_j
=
\begin{cases}
0,
&
M_{R,j}=1,
\\
-\infty,
&
M_{R,j}=0.
\end{cases}
$$

广播到所有目标 Query 后：

$$
\tilde A^{R}_{i,j}
=
A_{\mathrm{fg},i,j}
+
B^R_j.
$$

注意力权重：

$$
P^{\mathrm{fg}}_{i,j}
=
\frac{
\exp
\left(
\tilde A^R_{i,j}
\right)
}{
\sum_k
\exp
\left(
\tilde A^R_{i,k}
\right)
}.
$$

因为正常参考位置的 logit 为 $-\infty$，所以：

$$
P^{\mathrm{fg}}_{i,j}
=0
\quad
\text{if}
\quad
M_{R,j}=0.
$$

最终：

$$
O_i^{\mathrm{fg}}
=
\sum_{j:M_{R,j}=1}
P^{\mathrm{fg}}_{i,j}
V_{R,j}.
$$

因此，参考图像的正常背景没有资格成为异常通道的来源。

### 6.9. 为什么公式（4）需要 $M_R$

参考异常图像不是一张纯异常贴片。它同时包含：

1. 异常区域。
2. 正常产品表面。
3. 产品边界。
4. 拍摄背景。

若不使用 $M_R$，目标异常位置可能发现参考图像中的正常区域与自己更相似，于是高权重读取正常 Value，导致：

1. 异常不出现。
2. 异常只出现很小一部分。
3. 参考产品外观被复制。
4. 跨类别时来源材质泄漏。

$M_R$ 的作用就是明确回答：

> 从参考图像的哪里取异常？

### 6.10. 公式（5）：正常背景读取

论文公式（5）为：

$$
\begin{aligned}
\operatorname{Attn}_{\mathrm{bg}}
=
\operatorname{softmax}
\Big(
&A_{\mathrm{bg},t,l}
\odot
\mathcal{MF}
\left(
M_T=1,-\infty
\right)
\Big)
\\
&\cdot
V_{N,t,l}.
\end{aligned}
\tag{5}
$$

其语义是：

> 正常通道只能从目标掩码外的正常图像位置读取背景特征。

定义：

$$
B^N_j
=
\begin{cases}
0,
&
M_{T,j}=0,
\\
-\infty,
&
M_{T,j}=1.
\end{cases}
$$

然后：

$$
P^{\mathrm{bg}}
=
\operatorname{softmax}
\left(
A_{\mathrm{bg}}+B^N
\right).
$$

输出：

$$
O_i^{\mathrm{bg}}
=
\sum_{j:M_{T,j}=0}
P^{\mathrm{bg}}_{i,j}
V_{N,j}.
$$

### 6.11. 为什么正常分支用 $M_T$ 屏蔽来源

正常图像本身没有真实异常掩码，但目标掩码 $M_T$ 标出了即将被异常替换的区域。

如果正常通道仍允许从 $M_T=1$ 的位置读取正常特征，那么目标异常区域可能反复被正常内容拉回，削弱异常。

所以论文把：

$$
M_T=1
$$

区域从正常 Key 来源中移除。

这不是说正常图像对应位置本来有异常，而是说：

> 这些位置在输出中不再承担正常内容来源角色。

### 6.12. 公式（6）：前景与背景融合

论文公式（6）为：

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

逐目标位置写为：

$$
O_i^\star
=
\begin{cases}
O_i^{\mathrm{fg}},
&
M_{T,i}=1,
\\
O_i^{\mathrm{bg}},
&
M_{T,i}=0.
\end{cases}
$$

因此，$M_T$ 在这里不是屏蔽来源列，而是选择目标输出行。

### 6.13. 公式（3）至（6）的完全展开

将整个过程展开：

$$
\begin{aligned}
O_i^\star
={}&
M_{T,i}
\sum_{j:M_{R,j}=1}
\frac{
\exp
\left(
q_{T,i}^\top k_{R,j}/\sqrt d
\right)
}{
\sum_{k:M_{R,k}=1}
\exp
\left(
q_{T,i}^\top k_{R,k}/\sqrt d
\right)
}
V_{R,j}
\\
&+
(1-M_{T,i})
\sum_{j:M_{T,j}=0}
\frac{
\exp
\left(
q_{T,i}^\top k_{N,j}/\sqrt d
\right)
}{
\sum_{k:M_{T,k}=0}
\exp
\left(
q_{T,i}^\top k_{N,k}/\sqrt d
\right)
}
V_{N,j}.
\end{aligned}
$$

这条公式完整表达了 TriAG：

1. 目标掩码内的 Query 只从参考异常位置读取。
2. 目标掩码外的 Query 只从正常背景位置读取。
3. 两种读取都保留目标 Query。

### 6.14. 一个小型数值例子

假设某个目标位置的 Query 对四个参考位置的分数为：

$$
s
=
[2,1,0,-1].
$$

参考掩码为：

$$
M_R
=
[1,0,1,0].
$$

允许位置只有第 1、3 个。Mask fill 后：

$$
\tilde s
=
[2,-\infty,0,-\infty].
$$

Softmax：

$$
\begin{aligned}
p_1
&=
\frac{e^2}{e^2+e^0}
\approx
0.881,
\\
p_3
&=
\frac{e^0}{e^2+e^0}
\approx
0.119.
\end{aligned}
$$

于是：

$$
O_i^{\mathrm{fg}}
=
0.881V_{R,1}
+
0.119V_{R,3}.
$$

第 2、4 个位置虽然原始分数存在，但因为不属于参考异常，权重严格为 0。

若该目标位置满足：

$$
M_{T,i}=1,
$$

最终使用 $O_i^{\mathrm{fg}}$。若：

$$
M_{T,i}=0,
$$

最终使用正常通道 $O_i^{\mathrm{bg}}$。

### 6.15. 前景—背景 Query confusion 是什么

论文使用“foreground-background query confusion”描述一种问题。

目标 Query 对来源图像执行匹配时，并不知道作者主观上希望它读取异常还是背景。它只根据学习到的特征相似度选择 Key。

例如，目标榛子表面某个位置可能与参考图像的大面积正常榛子表面非常相似，而真实裂纹区域只占很小面积。没有掩码时，Softmax 很可能把大部分权重分给正常背景。

因此，所谓 query confusion 不是 Query 本身损坏，而是：

$$
\text{候选 Key 集合中同时存在前景和背景，导致查询来源语义混淆}.
$$

O2MAG 通过 source mask 把候选集合直接切开。

### 6.16. 正常背景为什么不能只靠最终像素混合

一种简单做法是先生成整张异常图，再做：

$$
\hat I
=
M_T\odot I_{\mathrm{gen}}
+
(1-M_T)\odot I_N.
$$

这样虽然能严格保留掩码外像素，但会出现：

1. 掩码边界不连续。
2. 异常与周围材质不能自然融合。
3. 阴影、裂纹延伸和局部结构被硬切断。
4. 掩码内部的内容没有利用正常上下文共同生成。

TriAG 在中间特征层控制来源，让异常区域与正常区域在整个去噪过程中共同形成，理论上更容易得到自然过渡。

### 6.17. 目标正常初始化与正常 K/V 的双重作用

背景保持有两层机制。

第一层，latent 初始化：

$$
Z_T^{\mathrm{tar}}
=
Z_T^{\mathrm{nor}}.
$$

它使目标分支从正常图像反演轨迹出发。

第二层，attention companion branch：

$$
Q_T
\rightarrow
(K_N,V_N).
$$

它在后续每个编辑层持续提供正常背景特征。

只使用第一层时，异常文本和前景注入仍可能逐步改变背景；只使用第二层而从随机噪声开始，则目标整体结构可能不稳定。两者结合更合理。

### 6.18. 为什么选择 decoder/up-block 层

论文正文写：

$$
L_S
\in
\left\{
9,\ldots,16
\right\}.
$$

补充材料写主要使用：

$$
10\text{--}16.
$$

作者的观察是：

1. Down/mid block 中的 self-attention 对本文异常迁移不够有信息量。
2. Up block 后段更包含细粒度、patch 级纹理。
3. 工业异常通常是局部纹理或局部结构变化，所以需要较细的空间特征。

层编号不一致可能来自：

1. 0-based 与 1-based 编号差异。
2. 是否把某些模块计入 attention 层序号。
3. 正文与补充材料的轻微版本差异。

复现时不能只写一个数字范围就假设对应正确模块，必须打印 U-Net attention processor 的执行顺序和模块路径。

### 6.19. 为什么从第 5 个去噪步骤开始

论文观察到 object-centric 异常数据中的产品轮廓较简单，整体结构在去噪早期就形成。作者不在最开始立即注入异常，而是先让目标分支建立正常物体布局，再开始 grafting。

用采样迭代编号表示，更清晰的逻辑是：

$$
\begin{cases}
\text{标准去噪},
&
k\leq5,
\\
\text{启用 TriAG},
&
k>5.
\end{cases}
$$

论文公式（11）写为 $t>T_S$，算法又从 $t=T$ 递减，因此若把 $t$ 直接理解为 scheduler 原始时间编号，会产生方向歧义。复现时应优先遵循文字“from the 5th denoising step”和图 2 的流程，用显式采样步骤计数器实现。

### 6.20. 掩码如何适配不同 attention 分辨率

原始掩码通常为：

$$
M
\in
\left\{0,1\right\}^{512\times512}.
$$

U-Net attention 分辨率可能为：

$$
64\times64,
\quad
32\times32,
\quad
16\times16,
\quad
8\times8.
$$

因此，每层需要得到：

$$
M^{(l)}
=
\operatorname{Resize}
\left(
M,h_l,w_l
\right).
$$

对于二值掩码，最保守的实现是最近邻插值：

$$
M^{(l)}
=
\operatorname{ResizeNearest}
\left(
M,h_l,w_l
\right).
$$

然后展平：

$$
m^{(l)}
\in
\left\{0,1\right\}^{N_l}.
$$

在多头 attention 中：

1. Key mask 可广播为 $[B,1,1,N_l]$。
2. Query gate 可广播为 $[B,1,N_l,1]$。

论文没有明确说明插值方式、阈值、形态学处理和广播实现，这些是复现必须补充的工程细节。

### 6.21. 小掩码为什么会导致 NaN 或异常消失

如果一个很小的异常在下采样后变成全零：

$$
\sum_jM_j^{(l)}
=0,
$$

那么所有参考 Key 都会被屏蔽：

$$
A_{i,:}
=
[-\infty,\ldots,-\infty].
$$

Softmax 的分母为 0，数值实现可能产生 NaN。

即使没有 NaN，小异常在 $8\times8$ 或 $16\times16$ 上也可能只剩一个位置，细节很难表达。

工程上需要：

1. 过滤原始空掩码。
2. 每层缩放后再次检查。
3. 必要时适度膨胀极小掩码。
4. 至少保留一个有效 Key。
5. 使用有限大负数而不是字面 $-\infty$，尤其在 fp16 中。

### 6.22. TriAG 是否能严格保证掩码外完全不变

不能。

原因包括：

1. Attention 只是 U-Net 的一部分。
2. 卷积会混合邻域特征。
3. 残差连接传播全局信息。
4. Cross-attention 仍对整个目标分支生效。
5. CFG 会整体改变噪声预测。
6. VAE 解码器也会产生局部平滑和跨像素影响。

TriAG 的目标是显著提高背景保真度，而不是给出数学上的逐像素不变保证。

### 6.23. TriAG 的核心本质

TriAG 不是简单“复制特征”，而是进行条件化读取：

$$
\text{目标结构提出查询}
\longrightarrow
\text{在允许的来源区域中寻找匹配特征}
\longrightarrow
\text{将来源 Value 融入目标特征}.
$$

它既保留了目标上下文，又利用了真实异常参考，是论文最核心的设计。


## 7. Anomaly-Guided Optimization 超详细解析

### 7.1. AGO 要解决什么问题

正提示模板：

$$
y
=
\text{“A photo of a [cls] with a [anomaly type]”}
$$

很简洁，但存在语义域差异。

Stable Diffusion 的文本—图像知识主要来自通用互联网图文数据。通用语境中的：

$$
\text{crack}
$$

可能指：

1. 墙面裂缝。
2. 玻璃裂纹。
3. 道路裂缝。
4. 皮肤开裂。
5. 破碎物体。

而 MVTec-AD 中的 hazelnut-crack 具有特定：

1. 榛子外壳颜色。
2. 曲面结构。
3. 沟槽方向。
4. 裂纹宽度。
5. 裂纹内部阴影。
6. 与外壳材质的连接方式。

所以：

$$
p_{\mathrm{SD}}
\left(
x
\mid
\text{“hazelnut with crack”}
\right)
$$

不一定与：

$$
p_{\mathrm{real}}
\left(
x
\mid
\text{hazelnut, crack}
\right)
$$

一致。

论文把这一问题称为 Stable Diffusion 训练数据和工业异常数据之间的 distribution discrepancy，以及文本编码语义与真实异常语义之间的 semantic gap。

### 7.2. 原始文本 embedding

先用冻结文本编码器：

$$
e_{\mathrm{ori}}
=
\tau_\theta(y)
\in
\mathbb R^{m\times d_\tau}.
$$

其中：

1. $m$ 是 token 数量。
2. $d_\tau$ 是 token embedding 维度。
3. $e_{\mathrm{ori}}$ 是通用 CLIP 语义表示。

如果直接用它生成，论文图 4 中原始或未充分优化的 embedding 可能无法重建正确物体外观和异常属性。

### 7.3. AGO 的优化变量

AGO 固定：

$$
\theta_{\mathrm{UNet}},
\quad
\theta_{\mathrm{VAE}},
\quad
\theta_{\mathrm{CLIP}}.
$$

只令：

$$
e
$$

可训练。

初始化：

$$
e^{(0)}
=
e_{\mathrm{ori}}.
$$

因此，AGO 不是修改模型“会做什么”，而是修改输入条件“怎样调用冻结模型已有能力”。

### 7.4. 公式（7）

论文公式为：

$$
\begin{aligned}
e^\star
=
\arg\min_e
\mathbb E_{t,\epsilon}
\Big[
&
\left\|
\epsilon
-
\epsilon_\theta
\left(
x_t,t,e
\right)
\right\|_2^2
\Big].
\end{aligned}
\tag{7}
$$

统一 latent 记号后，可写成：

$$
\begin{aligned}
e^\star
=
\arg\min_e
\mathbb E_{t,\epsilon}
\Big[
&
\left\|
\epsilon
-
\epsilon_\theta
\left(
z_t^R,t,e
\right)
\right\|_2^2
\Big].
\end{aligned}
$$

这里 $z_t^R$ 来自参考异常图像。

### 7.5. 公式（7）每一步在做什么

一种与公式最一致的标准噪声训练式解释是：

1. VAE 编码参考图像：

$$
z_0^R
=
\mathcal E(I_R).
$$

2. 随机采样时间步：

$$
t
\sim
\operatorname{Uniform}
\left\{
1,\ldots,T_{\mathrm{train}}
\right\}.
$$

3. 采样噪声：

$$
\epsilon
\sim
\mathcal N(0,I).
$$

4. 构造带噪参考 latent：

$$
\begin{aligned}
z_t^R
={}&
\sqrt{\bar\alpha_t}
z_0^R
\\
&+
\sqrt{1-\bar\alpha_t}
\epsilon.
\end{aligned}
$$

5. 冻结 U-Net 预测噪声：

$$
\hat\epsilon
=
\epsilon_\theta
\left(
z_t^R,t,e
\right).
$$

6. 计算损失：

$$
\mathcal L_{\mathrm{AGO}}
=
\frac{1}{D}
\sum_{r=1}^{D}
\left(
\epsilon_r-\hat\epsilon_r
\right)^2.
$$

7. 只对 $e$ 求梯度：

$$
g_e
=
\nabla_e
\mathcal L_{\mathrm{AGO}}.
$$

8. Adam 更新 $e$。

重复 500 步后得到：

$$
e^\star.
$$

### 7.6. 为什么重建噪声会改变文本语义

冻结 U-Net 可以看成一个函数：

$$
F_\theta(z_t,t,e)
=
\hat\epsilon.
$$

对固定参考带噪 latent $z_t^R$ 来说，不同文本条件会让网络给出不同噪声预测。

原始 embedding 如果只表达通用 “crack”，可能无法帮助网络解释参考工业裂纹，于是：

$$
\left\|
\epsilon-F_\theta(z_t^R,t,e_{\mathrm{ori}})
\right\|^2
$$

较大。

梯度会寻找一个 embedding，使冻结网络更容易预测这张参考异常图的真实噪声：

$$
\left\|
\epsilon-F_\theta(z_t^R,t,e^\star)
\right\|^2
$$

更小。

因此，$e^\star$ 会编码与参考图像兼容的信息。

必须谨慎理解论文所谓“从 normal semantics 推向 anomaly space”。它不是显式构造了一个可见的“正常子空间”和“异常子空间”，而是说：

$$
e^\star
$$

在冻结模型中能更好地解释参考异常。

### 7.7. AGO 学到的是整张图还是异常

公式（7）使用整张参考图像重建，没有显式只对 $M_R$ 内计算损失。因此，优化 embedding 可能同时吸收：

1. 物体类别外观。
2. 正常材质。
3. 拍摄背景。
4. 光照。
5. 异常纹理。
6. 异常与物体关系。

论文希望它“preserve object appearance but encode intended anomaly attributes”，但并没有在公式中加入一个明确的前景掩码损失或背景正则来严格解耦。

这也是 TriAG 和目标正常分支仍然必要的原因：

1. AGO 提供文本级异常语义。
2. TriAG 提供掩码约束下的图像级异常特征。
3. 正常分支防止 embedding 中的参考外观覆盖目标背景。

### 7.8. AGO 与 Textual Inversion 的关系

相似点：

1. 冻结扩散模型。
2. 优化文本表示。
3. 让文本条件与给定图像概念对齐。

区别：

1. 标准 Textual Inversion 通常引入一个新占位 token。
2. AGO 从自然语言模板的 embedding 开始。
3. 论文公式表示优化整个 $e$，而不是只优化一个新 token。
4. Textual Inversion 常用多张个性化图像学习可重复调用的概念。
5. AGO 使用一张参考异常，服务于当前异常生成管线。
6. AGO 与三分支图像特征 grafting 联合使用。

因此，AGO 可以理解为受 prompt embedding optimization / textual inversion 思想启发的异常参考驱动条件优化，但不是完全相同的训练协议。

### 7.9. AGO 与 DreamBooth 的区别

DreamBooth 通常更新扩散模型部分或全部参数：

$$
\theta
\leftarrow
\theta-\eta\nabla_\theta\mathcal L.
$$

AGO 更新：

$$
e
\leftarrow
e-\eta\nabla_e\mathcal L.
$$

因此：

1. AGO 优化变量少得多。
2. 不需要保存一套新 U-Net 权重。
3. 不会永久改变模型对其他类别的能力。
4. 但表达能力也受冻结模型限制。

### 7.10. 论文中关于 AGO 优化过程的一个重要歧义

正文写道：

1. 参考异常经过 DDIM inversion 得到 noise map。
2. 反演噪声和文本 embedding 输入冻结 Stable Diffusion。
3. 通过重建损失优化 embedding。

公式（7）却写：

$$
\mathbb E_{t,\epsilon}
\left[
\left\|
\epsilon-\epsilon_\theta(x_t,t,e)
\right\|^2
\right],
$$

这更像标准随机时间步噪声预测训练。

图 4 又展示使用参考图像反演噪声和不同优化阶段 embedding 得到的重建样本。

因此至少有两种可能实现。

第一种，随机噪声式 AGO：

1. 随机采样 $t,\epsilon$。
2. 从参考 $z_0^R$ 直接构造 $z_t^R$。
3. 最小化噪声 MSE。

第二种，固定 inversion trajectory 式 AGO：

1. 先获得 $Z_t^{\mathrm{ref}}$ 轨迹。
2. 让当前 embedding 驱动反向去噪重建 $I_R$。
3. 使用噪声误差、latent 误差或图像重建误差更新 embedding。

Algorithm 1 第一行写成近似：

$$
e^\star
\leftarrow
\operatorname{TextEncoder}(y)
-
\nabla_e
\mathcal L_{\mathrm{recon}}
(I_R,\hat I_R),
$$

又更接近图像重建描述。

论文没有给出足以唯一确定代码的全部细节。严格复现时需要作者实现；在没有代码时，应分别实现并验证这两种解释，而不能把其中一种冒充论文唯一方案。

### 7.11. 500 步优化的含义

论文设置：

$$
N_{\mathrm{AGO}}=500,
$$

$$
\eta=3\times10^{-3}.
$$

补充材料图 9 分析了优化步数：

1. 步数太少：embedding 欠优化，不能正确表达物体和异常。
2. 500 步：在重建质量和计算量之间平衡。
3. 1000 步：可能出现 rigid reconstruction，即过于僵硬地重建参考实例。

“僵硬重建”可以理解为 embedding 过度绑定一张参考图，导致：

1. 多样性下降。
2. 参考姿态或外观被过度复制。
3. 对不同目标正常图像适应变差。

### 7.12. 为什么优化太充分反而可能不好

AGO 的目标是降低参考图像重建误差。如果无限追求该目标，最优 embedding 可能包含过多实例特有信息。

异常生成需要两个相互矛盾的目标：

1. Fidelity：像真实参考异常。
2. Diversity：不是机械复制参考实例。

可抽象写成：

$$
\mathcal J
=
\mathcal L_{\mathrm{fidelity}}
+
\lambda
\mathcal L_{\mathrm{diversity}}.
$$

但 AGO 实际只显式优化重建项，没有显式多样性项。因此，早停和步数选择实际上充当了隐式正则化。

### 7.13. AGO 的数据增强实验

补充材料还尝试在 AGO 阶段对参考异常进行：

1. 随机旋转。
2. 随机平移。

每张参考样本扩充为五个变体，并分别优化对应 embedding。生成时随机选择增强图和相应 embedding。

作者观察到：

1. 多样性没有显著增加。
2. 主要作用是略微提高真实性和与真实分布的一致性。
3. Hazelnut 下游结果变化不大。

这说明简单几何增强不能创造新的异常模式，它只是让 embedding 对小范围位置和姿态变化更稳健。

### 7.14. 公式（8）：使用负提示的 CFG

论文公式（8）：

$$
\begin{aligned}
\hat\epsilon_t
={}&
\epsilon_\theta(x_t,y_n,t)
\\
&+
g
\Big(
\epsilon_\theta(x_t,y,t)
-
\epsilon_\theta(x_t,y_n,t)
\Big).
\end{aligned}
\tag{8}
$$

令：

$$
\epsilon_n
=
\epsilon_\theta(x_t,y_n,t),
$$

$$
\epsilon_p
=
\epsilon_\theta(x_t,y,t).
$$

则：

$$
\hat\epsilon_t
=
\epsilon_n
+
g(\epsilon_p-\epsilon_n).
$$

展开：

$$
\hat\epsilon_t
=
(1-g)
\epsilon_n
+
g
\epsilon_p.
$$

当：

$$
g=7.5
$$

时：

$$
\hat\epsilon_t
=
-6.5
\epsilon_n
+
7.5
\epsilon_p.
$$

所以负提示不是只占很小权重，而是定义了被强烈远离的方向。

### 7.15. 正提示中使用哪个 embedding

论文方法逻辑上，正提示应使用 AGO 优化后的：

$$
e^\star.
$$

负提示使用冻结文本编码器得到：

$$
e_n
=
\tau_\theta(y_n).
$$

因此 CFG 更准确写为：

$$
\begin{aligned}
\hat\epsilon_t
={}&
\epsilon_\theta
\left(
z_t,t,e_n
\right)
\\
&+
g
\Big[
\epsilon_\theta
\left(
z_t,t,e^\star
\right)
-
\epsilon_\theta
\left(
z_t,t,e_n
\right)
\Big].
\end{aligned}
$$

### 7.16. AGO 不能保证什么

AGO 不能保证：

1. 恢复完整真实异常分布。
2. 自动分离异常与背景语义。
3. 对所有目标正常图像都有效。
4. 对逻辑异常具有强关系推理能力。
5. 优化后的 token 仍具有完全可解释的词义。
6. 不会发生参考实例过拟合。

它能直接保证的只是：在所用损失和优化配置下，找到一个使参考异常重建误差更低的文本条件。

## 8. Dual-Attention Enhancement 超详细解析

### 8.1. DAE 为什么必要

即使有 AGO 和 TriAG，论文仍观察到：

$$
\operatorname{VisibleAnomalyRegion}
\left(
\hat I
\right)
\subsetneq
M_T.
$$

也就是说，异常只占目标掩码的一部分。

原因包括：

1. 目标 Query 只对少数参考异常 Key 有高相似度。
2. Softmax 权重过于分散。
3. anomaly token 在 cross-attention 中不是主导 token。
4. Stable Diffusion 的图像先验倾向保持正常表面。
5. 很小的目标掩码在低分辨率特征层中信号弱。
6. AGO 主要优化语义，并不直接强制空间填充。

DAE 的目标是：

$$
\text{让异常视觉特征更强}
+
\text{让异常文本语义更强}.
$$

因此称为 Dual-Attention：同时处理 self-attention 和 cross-attention。

### 8.2. Self-attention enhancement 的目标

TriAG 前景 attention 原本是：

$$
P_{\mathrm{fg}}
=
\operatorname{softmax}
\left(
A_{\mathrm{fg}}+
B^R
\right).
$$

DAE 希望目标位置更集中地关注参考异常特征，因此加入：

1. Logit bias。
2. Temperature scaling。

### 8.3. 公式（9）

论文公式（9）为：

$$
\begin{aligned}
\hat A_{\mathrm{fg},t,l}
&=
A_{\mathrm{fg},t,l}
+
\log(\gamma)
\overline M_R,
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
\gamma=1.1,
$$

$$
\tau_{\mathrm{fg}}=0.7.
$$

### 8.4. $\log\gamma$ 为什么表示倍率

对某个位置：

$$
\hat s
=
s+
\log\gamma.
$$

未归一化权重：

$$
\exp(\hat s)
=
\exp(s)
\gamma.
$$

所以加 $\log\gamma$ 相当于乘 $\gamma$。

加上温度后：

$$
\exp
\left(
\frac{s+
\log\gamma}{\tau}
\right)
=
\exp(s/\tau)
\gamma^{1/\tau}.
$$

本文：

$$
\gamma^{1/\tau}
=
1.1^{1/0.7}
\approx
1.146.
$$

也就是说，被 bias 选中的候选在未归一化权重层面约额外乘 1.146，同时 $\tau=0.7$ 还会整体强化分数差异。

### 8.5. Temperature 如何让 attention 更尖锐

假设两个有效参考异常 Key 的 logit 为：

$$
[2,1].
$$

普通 Softmax：

$$
[0.731,0.269].
$$

使用 $\tau=0.7$：

$$
[2/0.7,1/0.7]
\approx
[2.857,1.429].
$$

Softmax 约为：

$$
[0.807,0.193].
$$

高分 Key 的权重增加，低分 Key 的权重下降。目标位置会更坚定地读取最匹配的参考异常特征。

### 8.6. 公式（9）中 $\overline M_R$ 的歧义

论文没有精确定义：

$$
\overline M_R.
$$

如果它只是把参考前景 Key mask 广播到所有 Query：

$$
\overline M_{R,i,j}
=
M_{R,j},
$$

并且公式后面已经把 $M_R=0$ 的 Key 屏蔽，那么所有剩余有效 Key 都加相同常数：

$$
\log\gamma.
$$

根据 Softmax 平移不变性：

$$
\operatorname{softmax}
\left(
s+c\mathbf 1
\right)
=
\operatorname{softmax}(s),
$$

这时 $\gamma$ 不会改变相对权重，只剩 temperature 有效。

因此，可能的真实实现包括：

1. $\overline M_R$ 不是简单 Key mask，而是 Query-Key 二维掩码。
2. Bias 只加到某些更细的异常关联位置。
3. Bias 在 mask fill 前后采用了不同广播方式。
4. 论文公式进行了简化，代码有额外处理。

没有作者代码时，必须验证 $\gamma$ 是否真正改变 Softmax 输出。可以直接比较加 bias 前后的 attention 权重差异；如果完全相同，说明广播实现使 bias 抵消。

### 8.7. Cross-attention enhancement 的目标

Self-attention 负责从参考图像搬运异常视觉内容，但它不直接表示文本中的 anomaly type。

Cross-attention 矩阵：

$$
A_{c,t}
\in
\mathbb R^{N\times m}
$$

每行表示某个图像 patch 对所有文本 token 的注意力分布。

论文希望目标掩码内的位置强烈关注：

$$
[\mathrm{anomaly\ type}].
$$

例如：

$$
\text{crack token}.
$$

### 8.8. 公式（10）

论文公式（10）为：

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

其中：

1. $i$ 是图像空间位置。
2. $j$ 是文本 token 索引。
3. $j^\star$ 是异常类型 token 的位置。
4. $M_{T,i}$ 表示该图像位置是否在目标掩码内。
5. $C=100$。

### 8.9. 公式（10）逐位置解释

若目标位置在掩码内：

$$
M_{T,i}=1,
$$

则 anomaly token 权重变为：

$$
\tilde A_{i,j^\star}
=
100
A_{i,j^\star}.
$$

若目标位置在掩码外：

$$
M_{T,i}=0,
$$

则：

$$
\tilde A_{i,j^\star}
=0.
$$

其他 token：

$$
\tilde A_{i,j}
=
A_{i,j},
\qquad
j\neq j^\star.
$$

所以它同时完成：

1. 掩码内强化异常词。
2. 掩码外抑制异常词。

### 8.10. Cross-attention 权重乘 100 后是否仍是概率分布

如果 $A_c$ 已经经过 Softmax，那么原本：

$$
\sum_jA_{i,j}=1.
$$

将某一列乘 100 后：

$$
\sum_j\tilde A_{i,j}
\neq
1.
$$

论文公式没有写再次归一化，所以最忠于公式的理解是：

1. 先得到 attention probability。
2. 直接放大 anomaly token 的输出贡献。
3. 不重新 Softmax。

此时它不仅改变 token 之间的相对比例，还放大整个 attention 输出的幅度。

另一种可能实现是在 Softmax 前对 logit 加：

$$
\log C,
$$

这样仍保持归一化。但论文明确写的是对 $A_c$ 乘 $C$，因此不能在没有代码证据时擅自改成 logit bias。

### 8.11. Anomaly type 可能不是单个 token

论文写单一位置：

$$
j^\star.
$$

但 tokenizer 可能把某些异常词拆成多个子词。更一般地应定义：

$$
\mathcal J^\star
=
\left\{
j_1,j_2,\ldots,j_r
\right\}.
$$

然后对所有：

$$
j
\in
\mathcal J^\star
$$

执行增强。

复现时需要：

1. 运行 tokenizer。
2. 打印 token 序列。
3. 找到 anomaly phrase 对应的完整 token 区间。
4. 处理词尾、子词和特殊 token。

只按字符串长度猜 token 位置容易出错。

### 8.12. AGO 后语义为什么仍局限在原 token 位置

论文称，优化后 anomaly token 的属性仍位于原 token 位置，没有明显 semantic leakage，因此可以直接增强 $j^\star$ 列。

这句话意味着作者观察到：

1. 即使优化的是文本 embedding，异常语义主要仍由原 anomaly token 位置承载。
2. 其他 token 没有完全吸收异常属性。

但论文没有给出定量证明或 token 级消融。若实际优化整个 $m\times d_\tau$ embedding，语义完全不泄漏并非理论保证。复现时可以通过：

1. 可视化各 token cross-attention。
2. 分别屏蔽 token。
3. 比较生成结果。

验证这一假设。

### 8.13. DAE 的时间调度

补充材料给出：

$$
\tau_s
\in
(5,50)
$$

用于 self-attention enhancement。

$$
\tau_c
\in
(20,40)
$$

用于 cross-attention enhancement。

这里仍有时间编号方向问题。更安全的实现是用采样迭代编号 $k$：

1. Self-attention DAE 在第 5–50 个采样步骤间启用。
2. Cross-attention DAE 在中间第 20–40 个步骤间启用。

直觉上：

1. Self-attention 异常特征迁移覆盖较长阶段。
2. Cross-attention 强增强只在中间阶段，避免过早破坏布局或过晚产生尖锐伪影。

论文称这些超参数通过经验验证获得，并在 MVTec-AD、VisA、Real-IAD 中统一使用。

### 8.14. 为什么 Cross-attention 不从第一步一直乘 100

如果最早高噪声阶段就极强地强调 anomaly token，可能：

1. 干扰目标产品整体结构形成。
2. 使异常概念扩散到过大区域。
3. 产生与掩码不协调的全局变化。

如果最后阶段才增强：

1. 留给模型形成异常结构的步骤太少。
2. 只能产生表面颜色变化。
3. 难以改变局部几何。

因此论文选择中间时间窗口。

### 8.15. $C=100$ 是否过大

论文补充材料图 10 对 $C$ 做了分析。作者称：

1. 小掩码需要足够大的 $C$ 才能显现异常。
2. 在较宽范围内方法稳定。
3. 直到 $C>10000$ 才明显出现伪影。

这不代表 100 对所有模型、scheduler、精度和 prompt 都普遍安全。该结论是在论文实现和数据集下得到。若更换模型版本、attention processor 或归一化方式，需要重新验证。

### 8.16. 公式（11）：Attention EDIT 的整体调度，以及 TriAG 与 DAE 到底是串联还是互斥

论文将去噪过程中采用哪一种 attention 操作概括为：

$$
\operatorname{EDIT}
:=
\begin{cases}
\operatorname{TriAG},
&
\begin{aligned}
&t>T_S\\
&\text{且 }\ell\in L_S,
\end{aligned}
\\[8pt]
\operatorname{DAE},
&
\begin{aligned}
&t\in\tau_s\\
&\text{或 }t\in\tau_c,
\end{aligned}
\\[8pt]
\operatorname{SelfAttention}
\left(
Q_T,K_T,V_T
\right),
&
\text{其他情况}.
\end{cases}
\tag{11}
$$

其中：

1. $T_S$ 是开始执行 self-attention grafting 的去噪时刻。
2. $L_S$ 是被编辑的 U-Net self-attention 层集合。
3. $\tau_s$ 是 self-attention enhancement 生效的时间区间。
4. $\tau_c$ 是 cross-attention enhancement 生效的时间区间。
5. $\operatorname{SelfAttention}(Q_T,K_T,V_T)$ 表示不进行跨分支特征移植，仍然使用目标分支自身的 Query、Key 和 Value。

乍看公式（11），TriAG 与 DAE 好像是两个互斥选项：一次只执行其中一个。但这种字面理解会产生逻辑冲突，因为公式（9）明确是把 TriAG 的前景公式（4）改成增强版本。因此，self-attention DAE 必须建立在 TriAG 前景 attention 上，而不可能完全脱离 TriAG 单独运行。

更合理的执行逻辑是：

1. 先判断该层是否启用 TriAG。
2. 若启用，计算参考前景和正常背景 attention。
3. 若时间落入 $\tau_s$，用增强版公式（9）计算前景。
4. 若 cross-attention 时间落入 $\tau_c$，独立修改 anomaly token 权重。

所以实际应是：

$$
\text{TriAG}
+
\text{optional self-attention DAE}
+
\text{optional cross-attention DAE}.
$$

Algorithm 1 也把 EDIT 描述为同时包含 DAE 和 TriAG。这是论文公式排版不够严谨的地方。

### 8.17. 一个 Cross-attention 数值例子

假设某个目标 patch 对四个 token 的权重为：

$$
A_i
=
[0.2,0.5,0.1,0.2].
$$

第三个 token 是 crack：

$$
j^\star=3.
$$

若该位置在目标掩码内，$C=100$：

$$
\tilde A_i
=
[0.2,0.5,10,0.2].
$$

不重新归一化时，cross-attention 输出：

$$
\begin{aligned}
\tilde O_i
={}&
0.2V_1
+
0.5V_2
\\
&+
10V_3
+
0.2V_4.
\end{aligned}
$$

异常 token Value 成为绝对主导。

若重新归一化，则：

$$
\frac{1}{10.9}
[0.2,0.5,10,0.2]
\approx
[0.018,0.046,0.917,0.018].
$$

两种实现都让 crack 占主导，但输出幅度不同。论文没有明确再次归一化，因此必须依据作者代码确认。

### 8.18. DAE 的本质

DAE 的两条增强可以分别概括为：

$$
\boxed{
\text{Self-attention DAE：更强地读取真实异常图像特征}
}
$$

$$
\boxed{
\text{Cross-attention DAE：更强地表达异常文本 token}
}.
$$

两者结合，试图同时保证：

1. 异常看起来像真实参考。
2. 异常类型符合文本。
3. 异常在目标掩码中足够明显。
4. 图像与掩码标签更加一致。


## 9. Algorithm 1 与完整生成流程

### 9.1. Algorithm 1 的输入输出

补充材料第 12 页给出 Algorithm 1。

输入：

$$
(I_R,M_R),
\quad
I_N,
\quad
M_T,
\quad
y.
$$

输出：

$$
\hat I.
$$

算法可以拆为四大阶段：

1. 优化文本 embedding。
2. 对参考与正常图像做 inversion。
3. 逐时间步运行三分支并编辑 attention。
4. 解码目标 latent。

### 9.2. Algorithm 1 第 1 行：AGO

算法近似写为：

$$
e^\star
\leftarrow
\operatorname{TextEncoder}(y)
-
\nabla_e
\mathcal L_{\mathrm{recon}}
(I_R,\hat I_R).
$$

这不是一次减法就完成，而是对 embedding 执行多次迭代优化。

更完整的伪过程：

```text
初始化 e = TextEncoder(y)
冻结 VAE、U-Net、TextEncoder
重复 500 次：
    依据参考异常图像构造带噪 latent 或使用反演轨迹
    U-Net 根据 e 预测噪声或重建参考图
    计算重建损失
    只更新 e
返回 e*
```

### 9.3. Algorithm 1 第 2、3 行：两次 inversion

参考分支：

$$
\left\{
Z_T^{\mathrm{ref}},
Z_{T-1}^{\mathrm{ref}},
\ldots,
Z_0^{\mathrm{ref}}
\right\}
\leftarrow
\operatorname{Inversion}(I_R).
$$

正常分支：

$$
\left\{
Z_T^{\mathrm{nor}},
Z_{T-1}^{\mathrm{nor}},
\ldots,
Z_0^{\mathrm{nor}}
\right\}
\leftarrow
\operatorname{Inversion}(I_N).
$$

必须保存整个采样时间点序列，而不只是最终 $Z_T$，因为每个时间步都要从对应来源 latent 提取 K、V。

### 9.4. Algorithm 1 第 4 行：目标初始化

$$
Z_T^{\mathrm{tar}}
\leftarrow
Z_T^{\mathrm{nor}}.
$$

其作用是让目标分支继承：

1. 正常产品整体轮廓。
2. 摄影背景。
3. 产品姿态。
4. 正常材质的大尺度结构。

### 9.5. Algorithm 1 第 5 行：逐步去噪循环

论文写：

$$
t=T,T-1,\ldots,1.
$$

每一次循环都对应一个采样时间点。

实现时 scheduler 的 `timesteps` 可能类似：

```python
[981, 961, 941, ..., 21, 1]
```

而不是简单的 50 到 1。建议同时记录：

1. `step_index = 0,1,...,49`。
2. `scheduler_timestep`。

所有“第 5 步”“第 20–40 步”调度用 `step_index` 判断，U-Net 时间嵌入仍使用 scheduler timestep。

### 9.6. Algorithm 1 第 6 行：参考分支前向

论文写为：

$$
\left\{
Q_R,K_R,V_R
\right\}
\leftarrow
\epsilon_\theta
\left(
Z_t^{\mathrm{ref}},t
\right).
$$

这不是说 U-Net 通常只输出 Q、K、V。实际 U-Net 输出噪声预测；要获得 Q、K、V，需要：

1. 给指定 attention 模块安装 hook 或自定义 attention processor。
2. 在 forward 过程中截获线性投影后的 Q、K、V。
3. 按层和 attention 类型缓存。

参考分支的噪声预测结果本身未必需要保存，关键是中间 K、V。

### 9.7. Algorithm 1 第 7 行：正常分支前向

同理：

$$
\left\{
Q_N,K_N,V_N
\right\}
\leftarrow
\epsilon_\theta
\left(
Z_t^{\mathrm{nor}},t
\right).
$$

缓存选定 self-attention 层中的：

$$
K_N,V_N.
$$

### 9.8. Algorithm 1 第 8 行：目标分支前向

论文写：

$$
\left\{
Q_T,K_T,V_T
\right\}
\leftarrow
\epsilon_\theta
\left(
Z_t^{\mathrm{tar}},t
\right).
$$

实际实现有一个循环依赖问题：

1. 目标 U-Net 前向需要在 attention 层内部使用编辑结果。
2. 不能先完整跑完目标 U-Net，再把 attention 输出替换后重新使用。

因此，更现实的实现是：

1. 先运行参考、正常分支，缓存来源 K、V。
2. 运行目标分支时，自定义 attention processor 在每层即时计算 $Q_T$。
3. Processor 直接读取缓存的 $K_R,V_R,K_N,V_N$。
4. 在目标 forward 内返回编辑后的 attention 输出。

Algorithm 1 把 Q、K、V 提取和噪声预测拆成两行，是概念描述，不应机械理解为目标 U-Net 必须前向两次。

### 9.9. Algorithm 1 第 9 行：EDIT

算法写为：

$$
\operatorname{Attn}^\star
\leftarrow
\operatorname{EDIT}
\left(
\left\{
Q_T,K_R,V_R
\right\},
\left\{
Q_T,K_N,V_N
\right\},
M_R,M_T
\right).
$$

EDIT 实际包含：

1. TriAG 前景 attention。
2. TriAG 背景 attention。
3. 目标掩码融合。
4. 指定时间的 self-attention DAE。
5. 指定时间的 cross-attention DAE。

算法注释将其指向 Eq. (10)，但 Eq. (10) 只描述 cross-attention enhancement；整体调度实际上对应正文 Eq. (11)。这很可能是补充材料的公式编号引用错误。

### 9.10. Algorithm 1 第 10 行：噪声预测

论文写：

$$
\epsilon
\leftarrow
\epsilon_\theta
\left(
Z_t^{\mathrm{tar}},
t,
\operatorname{Attn}^\star,
e^\star
\right).
$$

更准确地说，自定义 attention 输出已经嵌入 U-Net forward，U-Net 最终返回正条件噪声预测。

论文算法没有显式写 negative prompt CFG。结合公式（8），目标分支还应计算负条件预测，并组合：

$$
\begin{aligned}
\epsilon_t^{+}
&=
\epsilon_\theta
\left(
Z_t^{\mathrm{tar}},t,e^\star;
\operatorname{EDIT}
\right),
\\
\epsilon_t^{-}
&=
\epsilon_\theta
\left(
Z_t^{\mathrm{tar}},t,e_n;
\operatorname{EDIT}^{-}
\right),
\\
\hat\epsilon_t
&=
\epsilon_t^{-}
+
g
\left(
\epsilon_t^{+}
-
\epsilon_t^{-}
\right).
\end{aligned}
$$

这里又有一个论文未说明的细节：负条件 forward 是否也使用相同 TriAG self-attention，以及 cross-attention DAE 是否只对正分支使用。最合理的实现是：

1. TriAG self-attention 对正、负 CFG batch 都用于保持图像结构和来源控制。
2. Anomaly-token cross-attention enhancement 只对正提示对应的条件样本使用。
3. 负提示的 token 序列不同，不能沿用正提示的 $j^\star$。

### 9.11. Algorithm 1 第 11 行：DDIM 更新

$$
Z_{t-1}^{\mathrm{tar}}
\leftarrow
\operatorname{SampleDDIM}
\left(
Z_t^{\mathrm{tar}},
\hat\epsilon_t,
t
\right).
$$

这一步把编辑后的 attention 影响转化为新的目标 latent。

需要理解：attention 编辑不是直接改最终图像。它先改变 U-Net 中间特征，再改变噪声预测，噪声预测再改变下一步 latent，经过多次累积后最终影响图像。

### 9.12. Algorithm 1 第 13 行：解码

循环结束得到：

$$
Z_0^{\mathrm{tar}}.
$$

VAE 解码：

$$
\hat I
=
\operatorname{Decode}
\left(
Z_0^{\mathrm{tar}}
\right).
$$

图像通常还需要：

1. VAE scaling factor 还原。
2. 从模型值域映射到 $[0,1]$ 或 $[0,255]$。
3. 截断数值范围。
4. 转成 RGB 图像格式。

### 9.13. 更严格的完整数学流程

AGO：

$$
e^\star
=
\operatorname{AGO}
\left(
I_R,y
\right).
$$

反演：

$$
\mathcal Z_R
=
\operatorname{Inv}
\left(
I_R,e_R
\right),
$$

$$
\mathcal Z_N
=
\operatorname{Inv}
\left(
I_N,e_N
\right).
$$

初始化：

$$
Z_T^{\mathrm{tar}}
=
Z_T^{\mathrm{nor}}.
$$

对每个采样步骤 $k$：

$$
\left(
K_{R,k,l},V_{R,k,l}
\right)
=
\operatorname{ExtractKV}
\left(
Z_k^{\mathrm{ref}},l
\right),
$$

$$
\left(
K_{N,k,l},V_{N,k,l}
\right)
=
\operatorname{ExtractKV}
\left(
Z_k^{\mathrm{nor}},l
\right).
$$

目标层中：

$$
Q_{T,k,l}
=
X_{T,k,l}W_Q.
$$

前景：

$$
O_{\mathrm{fg},k,l}
=
\operatorname{ForegroundAttn}
\left(
Q_{T,k,l},
K_{R,k,l},
V_{R,k,l},
M_R
\right).
$$

背景：

$$
O_{\mathrm{bg},k,l}
=
\operatorname{BackgroundAttn}
\left(
Q_{T,k,l},
K_{N,k,l},
V_{N,k,l},
M_T
\right).
$$

融合：

$$
O_{T,k,l}^\star
=
M_T^{(l)}
\odot
O_{\mathrm{fg},k,l}
+
\left(
1-M_T^{(l)}
\right)
\odot
O_{\mathrm{bg},k,l}.
$$

正负噪声预测：

$$
\epsilon_k^+,
\quad
\epsilon_k^-.
$$

CFG：

$$
\hat\epsilon_k
=
\epsilon_k^-
+
g
\left(
\epsilon_k^+
-
\epsilon_k^-
\right).
$$

更新：

$$
Z_{k+1}^{\mathrm{tar}}
=
\operatorname{DDIMStep}
\left(
Z_k^{\mathrm{tar}},
\hat\epsilon_k
\right).
$$

最终：

$$
\hat I
=
\mathcal D
\left(
Z_0^{\mathrm{tar}}
\right).
$$

### 9.14. 高层伪代码

```python
# 所有模型参数冻结
text_emb = text_encoder(positive_prompt)
optimized_emb = optimize_embedding_with_reference(
    reference_image,
    text_emb,
    steps=500,
    lr=3e-3,
)
negative_emb = text_encoder(negative_prompt)

ref_trajectory = ddim_inversion(reference_image)
nor_trajectory = ddim_inversion(normal_image)

target_latent = nor_trajectory.highest_noise_latent.clone()

for step_index, timestep in enumerate(scheduler.timesteps):
    ref_latent = ref_trajectory[timestep]
    nor_latent = nor_trajectory[timestep]

    # 两个来源分支前向并缓存选定层 K/V
    ref_cache = run_source_branch_and_cache_kv(
        ref_latent, timestep, branch="reference"
    )
    nor_cache = run_source_branch_and_cache_kv(
        nor_latent, timestep, branch="normal"
    )

    # 目标正条件前向；目标 self-attention processor 内部执行 TriAG/DAE
    eps_pos = run_target_branch(
        target_latent,
        timestep,
        optimized_emb,
        ref_cache,
        nor_cache,
        reference_mask,
        target_mask,
        step_index,
        apply_positive_cross_dae=True,
    )

    # 目标负条件前向
    eps_neg = run_target_branch(
        target_latent,
        timestep,
        negative_emb,
        ref_cache,
        nor_cache,
        reference_mask,
        target_mask,
        step_index,
        apply_positive_cross_dae=False,
    )

    eps = eps_neg + 7.5 * (eps_pos - eps_neg)
    target_latent = scheduler.step(eps, timestep, target_latent)

generated_image = vae_decode(target_latent)
```

这段伪代码只是把论文逻辑翻译成工程结构，不代表作者官方实现。

## 10. 从代码层面复现 O2MAG

### 10.1. 建议的软件组件

一个常见复现环境需要：

1. PyTorch。
2. Hugging Face Diffusers。
3. Transformers。
4. Stable Diffusion v1.5 权重。
5. DDIMScheduler 或支持 inversion 的自定义 scheduler。
6. NumPy、Pillow、OpenCV。
7. scikit-learn，用于 PCA。
8. 评价指标实现，如 KID、LPIPS。

论文只明确模型和主要超参数，不规定必须使用 Diffusers。其他 Stable Diffusion 框架也可以实现，但 attention hook 接口不同。

### 10.2. 冻结模型

应设置：

```python
vae.requires_grad_(False)
unet.requires_grad_(False)
text_encoder.requires_grad_(False)
```

AGO 中只让文本 embedding 可训练：

```python
optimized_emb = original_emb.detach().clone()
optimized_emb.requires_grad_(True)
```

还需要确保优化器只接收：

```python
[optimized_emb]
```

否则可能意外更新模型参数。

### 10.3. 图像预处理

需要与 Stable Diffusion VAE 预期一致：

1. 调整到 $512\times512$。
2. 转为 RGB。
3. 归一化到模型需要的值域，常见为 $[-1,1]$。
4. 形成 $[B,3,H,W]$ 张量。

参考、正常图像必须使用同样的预处理。

### 10.4. 掩码预处理

原始掩码通常是：

$$
M
\in
\left\{0,1\right\}^{H\times W}.
$$

处理步骤：

1. 与图像做完全相同的几何变换。
2. 使用最近邻插值缩放，避免产生模糊概率边界。
3. 转为 bool 或 0/1 浮点。
4. 为每个 attention 分辨率预缓存一个版本。
5. 检查每个版本是否为空。

示例：

```python
mask_64 = interpolate(mask, size=(64, 64), mode="nearest")
mask_32 = interpolate(mask, size=(32, 32), mode="nearest")
mask_16 = interpolate(mask, size=(16, 16), mode="nearest")
mask_8 = interpolate(mask, size=(8, 8), mode="nearest")
```

### 10.5. Attention processor 需要知道哪些信息

目标 self-attention processor 至少需要访问：

1. 当前模块路径或层编号 $l$。
2. 当前采样步骤 $k$。
3. 当前目标 hidden states。
4. 对应层参考 $K_R,V_R$。
5. 对应层正常 $K_N,V_N$。
6. 对应分辨率的 $M_R,M_T$。
7. 是否启用 TriAG。
8. 是否启用 self-attention DAE。

目标 cross-attention processor 还需要：

1. 当前 token 序列。
2. anomaly token 索引集合。
3. 是否为正 CFG 分支。
4. 是否处于 $\tau_c$ 时间窗口。

### 10.6. 怎样区分 self-attention 与 cross-attention

在 Diffusers 常见 attention processor 中：

1. `encoder_hidden_states is None` 通常表示 self-attention。
2. `encoder_hidden_states is not None` 通常表示 cross-attention。

但具体模型实现可能先把 self hidden states 赋给 `encoder_hidden_states`，因此应检查版本代码，而不能只凭名称猜测。

### 10.7. 来源分支缓存什么

最省信息的缓存是：

```text
cache[layer_name]["key"]
cache[layer_name]["value"]
```

无需缓存完整 attention 矩阵，因为目标 Query 不同，必须在目标分支内重新计算：

$$
Q_TK_R^\top
$$

和：

$$
Q_TK_N^\top.
$$

也不需要缓存参考 Query，除非用于可视化或调试。

### 10.8. 来源 K/V 必须使用相同投影层

目标 Query 和来源 Key 必须位于兼容表示空间。

设目标层投影为：

$$
W_Q^{(l)},
$$

参考层 Key 投影为：

$$
W_K^{(l)}.
$$

由于三个分支共享同一冻结 U-Net，它们使用相同层参数，所以：

$$
q_{T,i}^\top k_{R,j}
$$

具有可比性。

如果来源和目标使用不同模型或不同层，特征空间可能不对齐，点积不再有合理意义。

### 10.9. CFG batch 的工程处理

Diffusers 常把无条件和条件样本拼成一个 batch：

$$
B_{\mathrm{cfg}}
=2B.
$$

然后一次 U-Net forward 同时得到：

$$
\epsilon_{\mathrm{uncond}},
\quad
\epsilon_{\mathrm{cond}}.
$$

此时自定义 attention 需要正确扩展：

1. 来源 K/V 到 2B。
2. 掩码到 2B。
3. Cross-attention DAE 只修改条件 batch 的 anomaly token。

如果错误地对整个 2B 都增强 anomaly token，负提示分支也会被污染。

### 10.10. 数值稳定性

注意力分数可能使用 fp16。建议：

1. 计算 Softmax 前临时转 fp32。
2. 屏蔽值使用 `torch.finfo(dtype).min` 或足够大的负数。
3. 检查全屏蔽行。
4. 检查 Softmax 后是否含 NaN、Inf。
5. 公式（10）乘 100 后监控输出幅度。
6. 必要时对 attention output 做 dtype 恢复。

### 10.11. 层编号映射

论文写 9–16 或 10–16，但 Diffusers 中模块名可能类似：

```text
up_blocks.1.attentions.0.transformer_blocks.0.attn1
up_blocks.1.attentions.1.transformer_blocks.0.attn1
up_blocks.2.attentions.0.transformer_blocks.0.attn1
...
```

建议：

1. 按实际 forward 顺序给所有 `attn1` 编号。
2. 记录每层空间分辨率。
3. 用 PCA 或 attention 可视化验证选定层确实含异常纹理。
4. 把 9–16 和 10–16 都做消融。

### 10.12. Inversion 文本条件的缺失细节

DDIM inversion 通常也需要文本条件。论文没有明确说明：

1. 参考异常 inversion 使用原始 embedding 还是 $e^\star$。
2. 正常图像 inversion 使用正提示、class-only prompt 还是空提示。
3. 两条来源分支前向时分别用什么 cross-attention 文本条件。

一种合理但非论文唯一答案的方案：

1. 参考分支使用 $e^\star$。
2. 正常分支使用 “A photo of a [cls]” 或正常负提示。
3. 目标正分支使用 $e^\star$。
4. 目标负分支使用 $e_n$。

必须把这些方案作为复现超参数记录，而不是隐藏。

### 10.13. AGO 优化整个 embedding 还是只优化 anomaly token

论文公式优化：

$$
e
\in
\mathbb R^{m\times d_\tau},
$$

看起来是整个 prompt embedding。

但可能的工程选择有：

1. 优化全部 token。
2. 只优化 anomaly token。
3. 优化 class token 与 anomaly token。
4. 只引入一个新的可训练 token。

优化全部 token 表达能力更强，但更容易绑定参考物体和背景；只优化 anomaly token 更利于语义解耦，但可能重建能力不足。

论文没有给出 token 级参数冻结细节。主复现应先按照公式优化整个 embedding，再把 token-restricted 版本作为消融。

### 10.14. 生成多样性怎样产生

若以下全部固定：

1. $I_R$。
2. $M_R$。
3. $I_N$。
4. $M_T$。
5. $e^\star$。
6. 确定性 DDIM。
7. 随机种子。

则输出可能几乎确定。

要生成 1000 个样本，需要改变：

1. 正常图像或其增强版本。
2. 目标掩码。
3. 参考异常图像。
4. 参考增强版本。
5. 随机采样设置。
6. 可能的 scheduler stochasticity。

论文没有逐项给出每个生成样本的随机配对策略。复现时应记录：

```text
sample_id -> normal_id, target_mask_id, reference_id, seed, prompt, embedding_id
```

否则难以复现实验数据集。

### 10.15. 资源开销

论文报告：

1. 使用 NVIDIA RTX 5880 Ada 48GB GPU。
2. O2MAG 约使用 30GB；Real-IAD 表格报告约 32GB。
3. 单张图推理约 28 秒。
4. AnomalyAny 约 120 秒。
5. O2MAG 不需要每类训练扩散模型。

为什么显存高：

1. 三个分支。
2. 多层 K/V 缓存。
3. 多头高分辨率 attention。
4. CFG 条件与负条件。
5. 512×512 生成。

“0 training hours”只计算不微调生成模型的训练时间，并不表示 AGO 的 500 步优化和每张图 28 秒推理没有计算成本。

### 10.16. 最小复现验证顺序

不应一开始就完整生成 1000 张。建议按以下顺序逐步验证。

1. 验证 VAE 编解码。
2. 验证 DDIM inversion 能近似重建输入正常图和参考图。
3. 验证能截获指定层 Q、K、V。
4. 验证标准目标分支可重建正常图。
5. 只启用 TriAG，不启用 AGO、DAE。
6. 检查掩码内异常迁移与掩码外背景。
7. 加 AGO，检查异常语义变化。
8. 加 self-attention DAE。
9. 加 cross-attention DAE。
10. 检查 NaN、掩码空化和 token 索引。
11. 对少量类别生成可视化结果。
12. 最后批量生成数据并训练下游模型。

### 10.17. 复现时应保存的中间结果

建议保存：

1. 原始和优化后的 prompt embedding 距离。
2. AGO 每步 loss 曲线。
3. 不同步数的参考重建图。
4. 参考与正常 inversion 重建误差。
5. 每层 Q/K/V 范数。
6. TriAG 前后的 attention map。
7. DAE 前后的 attention entropy。
8. 目标掩码在每层分辨率的可视化。
9. 生成图与正常图掩码外差异图。
10. 生成可见异常区域与目标掩码的重叠。

这些中间诊断比只看最终图片更容易定位实现错误。


## 11. 实验设置与结果的完整解读

### 11.1. MVTec-AD 数据设置

论文主实验使用 MVTec-AD。该数据集包含 15 个产品类别，每个类别最多包含 8 种异常类型。

论文采用 few-shot 划分：

1. 每种异常类型的前 $1/3$ 图像作为参考异常数据。
2. 剩余 $2/3$ 异常图像用于评估。

这里需要注意：

1. 单次 O2MAG 生成使用一张参考图。
2. 整个 MVTec 实验可以访问前 $1/3$ 的参考池。
3. 因此主实验不是严格的“每种异常类型全程只有一张参考”。

### 11.2. 正常数据与目标掩码准备

论文补充材料说明：

1. 每类正常图像通过翻转、平移和旋转扩充到 1000 张。
2. 方向敏感类别只使用翻转。
3. 目标掩码主要来自 AnomalyDiffusion。
4. 先生成 1200 个候选掩码。
5. 过滤全零掩码后保留 1000 个。

因此每个类别最终可以合成约：

$$
1000
$$

个图像—掩码对。

### 11.3. 生成模型超参数

论文明确给出：

$$
\text{Backbone}
=
\text{Stable Diffusion v1.5},
$$

$$
T_{\mathrm{denoise}}
=
50,
$$

$$
g
=
7.5,
$$

$$
N_{\mathrm{AGO}}
=
500,
$$

$$
\eta_{\mathrm{AGO}}
=
3\times10^{-3},
$$

$$
\gamma
=
1.1,
$$

$$
\tau_{\mathrm{fg}}
=
0.7,
$$

$$
C
=
100.
$$

时间窗口：

$$
\tau_s
\in
(5,50),
$$

$$
\tau_c
\in
(20,40).
$$

### 11.4. 对比方法

论文主要比较：

1. DFMGAN：GAN 异常生成方法。
2. AnomalyDiffusion：冻结扩散主干并学习异常 embedding。
3. DualAnoDiff：双分支或缺陷分支的训练式扩散方法。
4. SeaS：训练式异常生成方法。
5. AnomalyAny：training-free cross-attention 异常生成。
6. TF2：training-free 方法，但公开结果和代码条件有限。

论文说明：

1. TF2 没有公开代码，只在分类比较中使用论文报告结果。
2. AnomalyAny 没有公开完整 prompt 和掩码生成流程，所以部分实验采用论文作者自己构造的提示或引用其报告结果。

这会影响严格的公平比较，后文会单独分析。

### 11.5. 生成质量指标

论文使用：

1. KID，越低越好。
2. IC-LPIPS，越高通常表示越多样。

MVTec-AD 平均结果：

| 方法 | KID ↓ | IC-LPIPS ↑ |
|---|---:|---:|
| DFMGAN | 62.85 | 0.20 |
| AnomalyDiffusion | 102.67 | 0.29 |
| DualAnoDiff | 103.87 | 0.38 |
| SeaS | 125.67 | 0.35 |
| O2MAG | 45.55 | 0.30 |

O2MAG 的平均 KID 最低，说明在 Inception 特征统计上与真实异常更接近。

但 O2MAG 的 IC-LPIPS 不是最高。论文解释：

1. 它较好保持正常背景。
2. 正常图像主要来自有限训练集的简单增强。
3. 某些基线的高 IC-LPIPS 部分来自背景破坏，而不是纯异常多样性。

因此，不能把：

$$
\text{更高 IC-LPIPS}
$$

直接等同为：

$$
\text{更高质量异常多样性}.
$$

### 11.6. 不同类别 KID 结果的差异

O2MAG 在许多类别上 KID 很低，例如：

1. screw：1.48。
2. grid：7.04。
3. hazelnut：9.54。
4. toothbrush：10.75。
5. capsule：22.71。

但 leather 的 KID 为：

$$
168.22,
$$

明显较高。

这说明方法并非所有类别都同样有效。纹理类、颜色类或异常非常细微时，Inception 特征和 self-attention 迁移可能面临困难。平均结果很好，但阅读表格时应关注类别级失败，而不是只看平均值。

### 11.7. 下游异常检测和定位协议

论文为每种方法生成：

$$
1000
$$

个异常图像—掩码对，并训练 U-Net 分割模型。

正文第 7 页写“synthesize 1,000 image-mask pairs to train a U-Net”。补充材料第 16 页进一步说明：

> 生成 1000 张异常图后，将它们与前 $1/3$ 的真实异常图像结合，训练 U-Net。

因此下游训练并非只使用合成数据，而是：

$$
\text{少量真实异常}
+
\text{1000 张合成异常}.
$$

这个细节很重要。实验评估的是：

> 在少量真实异常基础上，哪种生成方法提供的扩充数据更有用？

而不是：

> 完全没有任何真实异常训练样本时，生成数据能否独立训练检测器？

### 11.8. MVTec-AD 下游平均结果

论文表 1 的平均结果：

| 方法 | AP-I ↑ | AUROC-P ↑ | AP-P ↑ | F1-P ↑ |
|---|---:|---:|---:|---:|
| DFMGAN | 94.8 | 90.0 | 62.7 | 62.1 |
| AnomalyDiffusion | 99.7 | 99.1 | 81.4 | 76.3 |
| DualAnoDiff | 98.9 | 99.1 | 84.5 | 78.8 |
| SeaS | 99.6 | 98.7 | 83.1 | 78.1 |
| O2MAG | 99.8 | 99.2 | 86.3 | 80.8 |

相对之前较好结果：

$$
\Delta\mathrm{AP-P}
=
+1.8,
$$

$$
\Delta\mathrm{F1-P}
=
+2.0.
$$

像素 AP 和 F1 的提升尤其重要，因为它们比接近饱和的图像级指标更能反映异常掩码质量。

### 11.9. 为什么图像级结果容易饱和

MVTec-AD 中很多类别只要存在明显异常，图像级检测就相对容易。表中大量方法的 AP-I 接近：

$$
100.
$$

此时，0.1 或 0.2 的平均差异可能不具有强解释力。

像素级定位更困难，因为模型必须：

1. 找到准确边界。
2. 避免把正常纹理误报为异常。
3. 识别小异常。
4. 处理异常面积不平衡。

所以 O2MAG 的主要证据应更多看：

$$
\mathrm{AP-P},
\quad
\mathrm{F1-P}.
$$

### 11.10. 类别级下游结果

O2MAG 在一些类别上表现突出：

1. transistor：

$$
\mathrm{AP-P}=98.2,
\qquad
\mathrm{F1-P}=93.2.
$$

2. screw：

$$
\mathrm{AP-P}=68.2,
\qquad
\mathrm{F1-P}=64.4,
$$

明显优于多种基线，但绝对值仍不高。

3. tile：

$$
\mathrm{AP-P}=98.2,
\qquad
\mathrm{F1-P}=92.7.
$$

也有类别不是最佳：

1. hazelnut 的 AP-P 低于 DualAnoDiff。
2. toothbrush 的像素指标明显不如部分基线。
3. capsule 的 AP-P 为 60.6，低于 DualAnoDiff 73.2。

因此，“平均最优”不意味着所有产品和异常类型均最优。

### 11.11. 与 AnomalyAny 的 training-free 比较

论文表 3：

| 方法 | AUC-I | F1-I | AUC-P | F1-P |
|---|---:|---:|---:|---:|
| AnomalyAny | 98.4 | 96.9 | 97.4 | 65.1 |
| O2MAG | 99.6 | 99.2 | 99.2 | 80.6 |

像素 F1 提升：

$$
80.6-65.1
=
15.5.
$$

论文据此强调，只有 cross-attention 文本控制难以产生与真实异常和掩码一致的局部缺陷；参考图像 self-attention 特征和 DAE 对定位有显著帮助。

但需要注意 AnomalyAny 公开 prompt 和 mask 生成流程不完整，比较协议可能不是完全同源官方实现。

### 11.12. 异常分类实验

论文使用各方法生成的图像训练 ResNet-34，平均分类准确率：

| 方法 | Accuracy ↑ |
|---|---:|
| DFMGAN | 49.61 |
| AnomalyDiffusion | 70.25 |
| TF2 | 61.88 |
| DualAnoDiff | 65.55 |
| SeaS | 56.70 |
| O2MAG | 82.35 |

O2MAG 相对 AnomalyDiffusion 提升：

$$
82.35-70.25
=
12.10
$$

个百分点。

作者认为这是因为：

1. AGO 提供更准确的异常类型语义。
2. TriAG 提供真实异常视觉特征。
3. 生成类别之间更可区分。

但分类准确率高也可能受到以下因素影响：

1. 合成异常类型是否过于明显。
2. 背景是否带有类别相关偏差。
3. 参考池和训练协议。
4. 不同方法生成数量和分辨率。

因此它支持“合成数据对分类有用”，不等价于全面证明视觉真实度。

### 11.13. Zero-shot cross-class 实验

论文主要例子：

$$
\text{wood-hole}
\longrightarrow
\text{hazelnut-hole}.
$$

生成时：

1. 只访问 wood-hole 异常图像。
2. 只访问 hazelnut 正常图像。
3. 不访问 hazelnut-hole 真实异常。

目标是把“hole”的异常属性跨类别迁移，同时保留 hazelnut 正常外观。

论文还测试：

1. hazelnut-crack $\rightarrow$ tile-crack。
2. pill-scratch $\rightarrow$ metal-nut-scratch。
3. leather-color $\rightarrow$ wood-color。

### 11.14. Zero-shot 定量结果

Hazelnut-hole 子集表 5：

| 方法 | AUC-P | AP-P | F1-P |
|---|---:|---:|---:|
| DRAEM | 99.3 | 74.8 | 73.3 |
| AnomalyAny | 98.6 | 78.8 | 75.0 |
| SeaS-zero | 97.5 | 78.6 | 73.7 |
| O2MAG-zero | 99.1 | 86.8 | 80.2 |
| SeaS-origin | 99.2 | 87.3 | 80.0 |
| O2MAG-origin | 99.1 | 90.4 | 84.8 |

O2MAG-zero 的 AP-P 和 F1-P 高于其他 zero-shot 对比，并接近使用目标类别真实异常的原始设置。

但 zero-shot 仍低于 O2MAG-origin：

$$
86.8<90.4,
$$

$$
80.2<84.8.
$$

说明同类异常词可以迁移一部分视觉属性，但来源类别差异仍造成损失。

### 11.15. Cross-class appearance leakage

补充材料明确指出，self-attention 特征同时包含：

1. 异常纹理。
2. 来源物体外观。
3. 来源材质。
4. 局部形状。

跨类别迁移时可能发生：

$$
\text{source appearance leakage}.
$$

例如：

1. 木纹随 hole 一起迁移到榛子。
2. 皮革颜色变化携带皮革表面纹理到木板。
3. 生成结果像 copy-paste，而不是自然融合。

这说明 K/V 中的“异常”与“产品外观”不是完全可分离的。

### 11.16. 消融实验

表 6：

| TriAG | DAE | AGO | AUROC ↑ | AP ↑ | F1-max ↑ |
|---|---|---|---:|---:|---:|
| ✓ |  |  | 99.0 | 81.9 | 77.6 |
| ✓ | ✓ |  | 99.0 | 83.3 | 77.9 |
| ✓ |  | ✓ | 99.1 | 82.7 | 77.7 |
| ✓ | ✓ | ✓ | 99.2 | 86.3 | 80.8 |

解读：

1. TriAG 单独已经是强主干。
2. 加 DAE：AP 从 81.9 提到 83.3，说明掩码填充和定位改善。
3. 加 AGO：AP 从 81.9 提到 82.7，说明文本异常语义和真实度有所帮助。
4. 两者同时加入：AP 提到 86.3，联合提升明显大于简单单模块增量。

这可能说明 AGO 与 DAE 有协同：

1. AGO 让异常 token 和生成语义更正确。
2. DAE 再把这个更正确的异常语义强制作用于目标区域。

### 11.17. 视觉消融结果

补充材料图 12 展示：

1. Only TriAG：能迁移真实异常，但异常可能较弱或只占掩码一部分。
2. +AGO：异常纹理和类型更真实，但空间覆盖仍可能不足。
3. +DAE：异常更明显、掩码填充更完整。
4. Full：结合图像级异常特征、文本级异常语义和空间增强。

视觉消融比平均数值更直接地对应三个模块的设计目标。

### 11.18. VisA 实验

VisA 包含 12 个类别，异常范围包括：

1. scratches。
2. dents。
3. stains。
4. cracks。
5. misalignment。
6. missing parts。

论文使用：

1. 前 $1/3$ 异常作为参考。
2. 后 $2/3$ 评估。
3. SeaS 生成目标掩码，因为 AnomalyDiffusion 在 VisA 上常产生空掩码。
4. 每种方法生成 1000 个图像—掩码对，并与前 $1/3$ 真实异常结合训练 U-Net。

### 11.19. VisA 生成质量

平均结果：

| 方法 | KID ↓ | IC-LPIPS ↑ |
|---|---:|---:|
| AnomalyDiffusion | 108.04 | 0.30 |
| DualAnoDiff | 96.99 | 0.43 |
| SeaS | 63.22 | 0.29 |
| O2MAG | 23.10 | 0.27 |

O2MAG KID 明显最低，但 IC-LPIPS 较低，仍需结合背景保持解释。

### 11.20. VisA 下游结果

图像级平均：

| 方法 | AUC-I | AP-I | F1-I |
|---|---:|---:|---:|
| AnomalyDiffusion | 90.1 | 88.9 | 82.1 |
| DualAnoDiff | 95.1 | 95.4 | 89.9 |
| SeaS | 92.8 | 93.3 | 87.5 |
| O2MAG | 93.6 | 94.0 | 88.4 |

O2MAG 不是图像级最佳，DualAnoDiff 更高。

像素级平均：

| 方法 | AUC-P | AP-P | F1-P |
|---|---:|---:|---:|
| AnomalyDiffusion | 97.0 | 42.9 | 45.8 |
| DualAnoDiff | 98.7 | 61.3 | 62.1 |
| SeaS | 98.8 | 65.1 | 64.1 |
| O2MAG | 98.9 | 67.2 | 64.6 |

O2MAG 像素指标最佳，但优势不如 MVTec-AD 明显。

论文认为：

1. DualAnoDiff 有专门异常分支，图像级异常可见性强。
2. O2MAG 的掩码一致性使像素定位更好。
3. VisA 中小异常较多，限制 self-attention grafting。

### 11.21. 与 AnomalyAny 的 VisA 比较

表 11 平均：

| 方法 | I-AUC | I-F1 | P-AUC | P-F1 |
|---|---:|---:|---:|---:|
| AnomalyAny | 95.8 | 91.9 | 98.7 | 50.4 |
| O2MAG | 93.6 | 88.4 | 98.9 | 64.6 |

O2MAG 图像级较低，但像素 F1 高：

$$
64.6-50.4
=
14.2.
$$

这再次支持 O2MAG 的主要优势在局部掩码一致性和定位，而不是每个数据集上都具有最高图像级可分性。

### 11.22. Real-IAD one-reference 实验

Real-IAD 包含：

1. 30 个类别。
2. 111 种异常类型。
3. 超过 15 万张图像。
4. 五个视角。
5. 每种异常约 120 张图。

论文严格使用：

$$
1
$$

张 top-view 异常图作为每种异常类型的参考，并使用 SeaS 掩码控制空间变化。

结果：

| 方法 | AUROC-I | AP-I | F1-I | AUROC-P | AP-P | F1-P | PRO |
|---|---:|---:|---:|---:|---:|---:|---:|
| DualAnoDiff | 72.5 | 77.8 | 75.8 | 91.3 | 25.7 | 31.9 | 75.7 |
| SeaS | 76.7 | 80.4 | 77.2 | 87.7 | 31.1 | 36.5 | 76.9 |
| O2MAG | 79.3 | 84.0 | 80.4 | 93.6 | 37.3 | 41.9 | 82.9 |

这组结果更直接支持“一张参考”的实用性，但绝对像素 AP、F1 仍不高，说明大规模多类别真实场景非常困难。

### 11.23. 运行时间与显存

补充材料表格给出：

| 方法 | 生成模型训练时间 | 单图推理 | GPU 显存或说明 |
|---|---:|---:|---:|
| DFMGAN | 464 h | 0.9 s | 训练式 |
| AnomalyDiffusion | 310 h | 7.4 s | 训练式 |
| DualAnoDiff | 197 h | 32.4 s | 训练式 |
| SeaS | 124 h | 10.7 s | 训练式 |
| AnomalyAny | 0 h | 120 s | training-free |
| O2MAG | 0 h | 28 s | 约 30–32GB |

这里的比较需要谨慎：

1. 训练式方法的训练时间可能是多类别累计或每类训练策略。
2. O2MAG 的 AGO 优化是否计入“推理时间”需要看具体计时实现。
3. 不同方法生成分辨率不同；论文指出 DFMGAN、AnomalyDiffusion 为 256×256，而部分其他方法为 512×512。
4. 单图推理快不等于总数据生产成本低，因为 O2MAG 每类要生成大量图像。
5. O2MAG 的显存明显高于部分训练式方法。

### 11.24. 掩码来源鲁棒性

Hazelnut-crack 上，论文比较：

1. Perlin noise。
2. Nano Banana。
3. AnomalyDiffusion。
4. SeaS。

像素 AP 大约在：

$$
94.8
\text{ 到 }
97.3
$$

之间，说明方法不只适用于一种目标掩码来源。

但不同掩码分布会改变：

1. 异常大小。
2. 异常形状。
3. 异常位置。
4. 与产品前景的重叠。

所以“鲁棒”不代表掩码质量不重要。一个完全不合理的掩码仍会让模型生成不自然异常。

### 11.25. 实验结果可以支持哪些结论

较强支持：

1. 三分支 self-attention grafting 对背景保持和异常迁移有效。
2. DAE 提高像素定位和掩码一致性。
3. AGO 提高异常语义和分类实用性。
4. O2MAG 在多个数据集上生成数据对下游 U-Net 有帮助。
5. 方法可以进行一定程度跨类别同异常类型迁移。
6. 不微调扩散主干也能达到强结果。

不能由实验严格证明：

1. 完整生成分布等于真实分布。
2. 每种异常类别都优于所有基线。
3. 一张参考足以覆盖所有真实异常模式。
4. Training-free 一定比训练式方法总成本更低。
5. 所有提升都只来自生成图，而与真实少样本混合、掩码来源和下游训练无关。
6. 逻辑异常已经得到可靠解决。


## 12. 核心难点的直白解释

### 12.1. “移植 Key、Value”到底移植了什么

Key、Value 不是一张可直接观看的异常贴片，也不是参考图像像素。

在某个 U-Net attention 层，参考隐藏特征：

$$
X_R
$$

经过投影得到：

$$
K_R
=
X_RW_K,
$$

$$
V_R
=
X_RW_V.
$$

其中包含模型在当前噪声时间和当前网络层对参考图像的中间理解。

移植 $K_R,V_R$ 的意思是：

1. 用参考位置的 Key 参与目标位置匹配。
2. 用匹配权重读取参考位置的 Value。
3. 把读取结果作为目标 attention 输出的一部分。

最终变化还要经过：

1. 输出投影。
2. 残差连接。
3. 后续卷积和 attention 层。
4. 噪声预测头。
5. DDIM 更新。
6. 多个采样步骤。

所以它是中间特征层面的引导，而不是直接粘贴。

### 12.2. 为什么 Query 能代表目标布局

Query 来自目标分支当前隐藏特征：

$$
Q_T
=
X_TW_Q.
$$

而 $X_T$ 已经受到：

1. 正常 inversion 初始化。
2. 前几个标准去噪步骤。
3. 目标产品结构。
4. 目标文本条件。
5. 之前所有层和时间步的更新。

因此，Query 不是一个纯位置编号，而是带有目标上下文的表示。

保留 Query 不意味着目标布局绝对不会变化，但比直接用参考 Query 更有利于保持目标产品结构。

### 12.3. Key 与 Value 为什么要同时来自同一来源

Key 决定匹配，Value 决定内容。

如果使用：

$$
K_R
\quad\text{但}\quad
V_T,
$$

那么目标 Query 会根据参考异常位置建立权重，却读取目标自身内容，异常迁移会非常弱或语义错位。

如果使用：

$$
K_T
\quad\text{但}\quad
V_R,
$$

权重根据目标自身位置关系产生，却去读取参考 Value，来源位置对应可能混乱。

使用配对的：

$$
(K_R,V_R)
$$

能让“匹配坐标”和“被读取内容”处于同一参考特征空间。

### 12.4. 两个掩码分别控制什么

最简单记忆方式：

$$
M_R
=
\text{从哪里拿异常},
$$

$$
M_T
=
\text{把异常放到哪里}.
$$

更严格地说：

1. $M_R$ 主要作用于参考 Key 轴。
2. $M_T$ 的反区域作用于正常 Key 轴。
3. $M_T$ 还作用于目标 Query 输出门控。
4. $M_T$ 还作用于 cross-attention anomaly token 的空间增强。

### 12.5. 为什么目标掩码不能直接等于参考掩码

如果总是：

$$
M_T=M_R,
$$

异常形状和位置会高度固定，One-to-More 的多样性有限。

独立目标掩码允许改变：

1. 异常位置。
2. 异常面积。
3. 异常粗略形状。
4. 与目标产品的空间关系。

但若 $M_T$ 与参考异常性质完全不兼容，例如把细裂纹掩码变成巨大圆形区域，模型可能：

1. 只在局部生成裂纹。
2. 产生不自然填充。
3. 被 DAE 强行放大成伪影。

目标掩码不是任意形状都合理，它最好与异常类型的几何先验相符。

### 12.6. 为什么文本 embedding 优化不等于训练模型

训练模型改变的是函数本身：

$$
F_\theta.
$$

Embedding 优化改变的是函数输入：

$$
F_\theta(\cdot,e).
$$

类比：

1. 微调模型像改造一台机器内部零件。
2. 优化 embedding 像寻找更有效的控制指令。

两者都用梯度，但影响范围、参数量和存储成本不同。

### 12.7. 为什么 AGO 会过拟合一张参考图

损失只要求重建参考图：

$$
\min_e
\mathcal L(I_R,e).
$$

它没有要求：

$$
\text{对其他正常图和其他异常形态也保持多样性}.
$$

所以 embedding 可能把：

1. 参考物体形状。
2. 特定光照。
3. 参考背景。
4. 异常位置。
5. 特定纹理。

一并编码进去。

500 步和早停是在经验上平衡这个问题，而不是从目标函数中彻底解决。

### 12.8. 为什么 DAE 不是简单把掩码区域变黑

DAE 没有直接修改 RGB。

Self-attention DAE 改变：

$$
\text{目标位置从哪些参考异常 Value 读取信息}.
$$

Cross-attention DAE 改变：

$$
\text{目标位置对 anomaly token 的语义响应强度}.
$$

最终颜色和结构由冻结扩散模型根据这些中间条件生成。因此 DAE 能产生不同颜色、形状和材质的异常，而不是固定像素操作。

### 12.9. 为什么异常可能仍然不填满掩码

即使 $C=100$，仍不能保证每个掩码像素可见异常，原因包括：

1. Attention 分辨率低于原图。
2. VAE 和 U-Net 特征会平滑边界。
3. Cross-attention 增强的是语义，不是硬像素约束。
4. Self-attention 匹配可能只找到少数参考位置。
5. 目标掩码形状与异常类型不兼容。
6. 后续网络层可能削弱增强。

所以“full-mask filling”是方法目标和经验效果，不是严格约束。

### 12.10. 为什么小异常特别困难

假设原图中的异常只有：

$$
4\times4
$$

像素。

在 VAE 8 倍下采样后，理论尺度约为：

$$
0.5\times0.5
$$

个 latent cell。

在 $32\times32$、$16\times16$、$8\times8$ attention 层中更难保留。

此外，PCA 选择方差最大方向。背景占据绝大多数位置，小异常带来的方差很低，前三主成分容易忽略它。

所以小异常问题不是简单把 $C$ 再增大就完全解决，而是受到潜空间分辨率和特征表示能力限制。

### 12.11. 为什么逻辑异常困难

纹理异常通常可以描述为局部外观改变：

$$
\text{正常表面}
\rightarrow
\text{裂纹、孔洞、污渍}.
$$

逻辑异常依赖部件关系：

1. 晶体管引脚是否连接焊盘。
2. 部件是否缺失。
3. 线缆颜色和位置是否交换。
4. 多个组件数量是否正确。

这需要模型表示：

$$
\text{实体}
+
\text{关系}
+
\text{约束}.
$$

O2MAG 主要编辑局部 attention 特征和文本 embedding，没有显式图结构、部件检测或关系推理模块。

例如 missing-transistor：前五步不编辑 attention，目标分支可能已经形成晶体管轮廓，后面再移植“缺失”特征很难把整个对象移除。

### 12.12. 为什么跨类别会像复制粘贴

参考 Value 同时编码：

$$
\text{异常}
+
\text{来源材质}
+
\text{来源物体局部外观}.
$$

$M_R$ 只能在空间上选择异常区域，不能保证选中 Value 内部只包含异常因素。

因此，wood-hole 的 Value 可能包含孔洞边缘和木纹。迁移到 hazelnut 时，木纹也被读出，结果像把木板贴到榛子。

这属于表示层面的 factor disentanglement 不充分。

### 12.13. 为什么 KID 低仍可能有失败样本

KID 是分布级平均统计。一个方法可能：

1. 大多数图像非常接近真实。
2. 少数图像存在严重伪影。

平均 KID 仍可能很好。

另一方面，一个方法可能准确覆盖罕见模式，但样本数少、统计波动大，KID 不一定体现其价值。

因此需要共同查看：

1. KID。
2. IC-LPIPS。
3. 类别级结果。
4. 视觉样本。
5. 下游定位。
6. 失败案例。

### 12.14. 为什么像素 AP 比 AUROC 更值得关注

像素异常通常占图像很小比例：

$$
N_{\mathrm{normal-pixel}}
\gg
N_{\mathrm{anomaly-pixel}}.
$$

AUROC 的 FPR 分母包含大量正常像素，即使有不少假阳性，FPR 仍可能很低。

Precision 会直接受到假阳性影响：

$$
\operatorname{Precision}
=
\frac{TP}{TP+FP}.
$$

因此 AP-P 和 F1-P 往往更敏感地反映定位质量。

### 12.15. 为什么 O2MAG 的主要提升集中在像素级

三个核心设计都与空间控制相关：

1. $M_R$ 限制异常来源。
2. $M_T$ 限制异常目标位置。
3. DAE 强制掩码内异常语义。
4. 正常分支保护掩码外背景。

所以它最直接改善的是：

$$
\text{异常在哪里、是否与掩码一致}.
$$

这与实验中像素 F1 的显著提升相吻合。

## 13. 论文中的实现歧义与批判性分析

### 13.1. “Training-free”术语存在边界问题

论文将 O2MAG 称为 training-free，但 AGO 有：

$$
500
$$

步 Adam 优化。

支持论文叫法的理由：

1. 不更新生成模型参数。
2. 不保存每类 U-Net checkpoint。
3. 优化变量只是文本条件。

需要保留的批判：

1. 仍然发生梯度反向传播。
2. 每张参考图或每个 embedding 需要优化成本。
3. 从广义角度它不是“零优化”。

更精确术语应是：

$$
\text{model-finetuning-free}
$$

或：

$$
\text{inference-time prompt optimization}.
$$

### 13.2. MVTec 主实验不是严格每类一张参考

论文方法宣传“One reference”，但 MVTec 主协议使用每种异常类型前 $1/3$ 图像作为参考数据。

这并非方法逻辑矛盾，因为一次生成仍可使用一张图，但会影响读者对数据可用量的理解。

最严格的一张参考证据来自 Real-IAD one-reference 实验，而不是 MVTec 主表。

### 13.3. 下游 U-Net 训练混入真实异常

补充材料说明生成图与前 $1/3$ 真实异常结合训练。

因此表 1 的提升不能解释成“纯合成数据完全替代真实异常”。正确解释是：

$$
\text{少量真实异常}
+
\text{O2MAG 增强}
$$

优于：

$$
\text{少量真实异常}
+
\text{其他生成增强}.
$$

### 13.4. 公式（4）、（5）的 mask 运算不严谨

论文写：

$$
A
\odot
\mathcal{MF}(M=0,-\infty).
$$

如果按字面逐元素乘：

1. 正数乘 $-\infty$ 得 $-\infty$。
2. 负数乘 $-\infty$ 可能得 $+\infty$。
3. 零乘无穷可能是 NaN。

正确语义应为：

$$
A.
\operatorname{masked\_fill}
(M=0,-\infty)
$$

或：

$$
A+B_{\mathrm{mask}}.
$$

### 13.5. 公式（9）的 $\overline M_R$ 未定义清楚

如前所述，如果所有有效 Key 都加相同 $\log\gamma$，Softmax 不变。

论文需要进一步说明：

1. $\overline M_R$ 的具体维度。
2. 它沿 Query 还是 Key 广播。
3. 是否是 $M_T\otimes M_R$ 的二维组合。
4. 是否包含连续权重而非二值值。

缺少这些信息会影响 DAE self-attention 的忠实复现。

### 13.6. 公式（10）是在 Softmax 前还是后操作

论文称 cross-attention map 是 token 概率分布，并直接乘：

$$
C=100.
$$

这暗示在 Softmax 后修改。

但很多 attention editing 方法在 logit 前加 bias。两者效果不同：

1. Softmax 前加 $\log C$：仍归一化。
2. Softmax 后乘 $C$：输出总幅度增大。

论文未说明是否重新归一化，作者代码是决定性依据。

### 13.7. 时间步方向不一致

论文同时出现：

1. 从第 5 个 denoising step 开始。
2. Algorithm 1 中 $t=T$ 递减到 1。
3. 公式（11）中 $t>T_S$。
4. $\tau_s\in(5,50)$、$\tau_c\in(20,40)$。

若 $t$ 是递减 scheduler 编号，$t>T_S$ 会在大多数早期步骤成立；若 $t$ 是正向采样计数，则含义不同。

复现必须用 `step_index` 明确重写。

### 13.8. 层编号 9–16 与 10–16 不一致

正文：

$$
L_S
\in
\left\{
9,\ldots,16
\right\}.
$$

补充材料：

$$
10\text{--}16.
$$

可能是索引起点差异，但论文没有说明。应把具体模块路径而非抽象编号作为最终配置。

### 13.9. Algorithm 1 的 EDIT 公式编号疑似错误

Algorithm 1 注释把 EDIT 指向 Eq. (10)，但：

1. Eq. (10) 只定义 cross-attention enhancement。
2. 整体 EDIT 调度是 Eq. (11)。
3. TriAG 是 Eq. (3)–(6)。
4. Self-attention DAE 是 Eq. (9)。

所以该注释不能直接作为实现依据。

### 13.10. AGO 使用随机噪声还是 inversion trajectory 不清楚

公式、文字和图示没有完全统一，前文已详细分析。

这会影响：

1. 优化目标。
2. 每步计算成本。
3. embedding 对不同时间步的泛化。
4. 是否容易重建参考实例。
5. 500 步的具体含义。

### 13.11. 参考和正常分支的文本条件未说明

Self-attention 特征虽然来自图像 hidden states，但 hidden states 已经过前面 cross-attention，受文本条件影响。

因此：

$$
K_R,V_R,K_N,V_N
$$

不是与 prompt 无关。

论文算法省略 source branch 文本条件，使以下细节不清楚：

1. Reference branch 使用哪个 prompt。
2. Normal branch 是否使用 class-only prompt。
3. Inversion 与生成时 prompt 是否相同。
4. CFG 是否用于 source branch。

### 13.12. AGO 优化哪些 token 未说明

若优化全部 token，异常语义可能泄漏到 class、介词、特殊 token；若只优化 anomaly token，公式应明确选择矩阵行。

论文称 anomaly attributes remain localized at original token position，但没有说明优化约束如何保证这一点。

### 13.13. 生成样本配对策略未完整公开

生成 1000 张时需要决定：

1. 正常图像和掩码如何配对。
2. 参考异常如何采样。
3. 每个参考使用多少次。
4. 是否使用固定随机种子。
5. 是否使用增强参考 embedding。

这些策略会显著影响多样性和类别覆盖，但论文没有提供完整数据生成清单。

### 13.14. 下游训练超参数不足

论文给出使用 U-Net 和 ResNet-34，但正文中没有完整列出：

1. 训练 epoch。
2. batch size。
3. 学习率。
4. 损失函数。
5. 数据增强。
6. 类别平衡策略。
7. 验证和阈值选择。

若严格复现表格，需要参考基线协议、作者代码或额外配置。

### 13.15. KID 的“真实分布”表述需要保守

KID 低说明：

$$
\text{有限样本的 Inception 特征统计更接近}.
$$

它不直接说明：

$$
p_{\mathrm{gen}}(x)
=
p_{\mathrm{real}}(x).
$$

论文中的“distribution-faithful”是经验性结论，而非概率论意义上的完整等价证明。

### 13.16. 基线公平性问题

论文面临实际困难：

1. TF2 无公开代码。
2. AnomalyAny 无完整 prompt 和 mask 生成细节。
3. 不同方法输出分辨率不同。
4. 有的方法每类训练，有的方法共享模型。
5. 掩码来源可能不同。

作者说明了部分限制，但这意味着比较结果应理解为现有可复现协议下的证据，而非完全控制变量的理想实验。

### 13.17. “背景保持”没有直接量化指标

论文主要通过：

1. KID。
2. IC-LPIPS 解释。
3. 视觉结果。
4. 下游性能。

论证背景保持。

但可以进一步增加：

$$
\operatorname{LPIPS}
\left(
\hat I_{\overline{M_T}},
(I_N)_{\overline{M_T}}
\right),
$$

$$
\operatorname{PSNR}
\left(
\hat I_{\overline{M_T}},
(I_N)_{\overline{M_T}}
\right),
$$

或背景特征距离，直接评价掩码外保持程度。

### 13.18. 掩码一致性也可以更直接评价

论文通过下游像素指标说明 DAE 有效，但可以额外测量：

1. 生成图异常显著图与 $M_T$ 的 IoU。
2. 异常可见面积占掩码面积比例。
3. 掩码外异常泄漏率。
4. 边界一致性。

这样能更直接验证“full-mask filling”。

## 14. 论文局限性与可行改进方向

### 14.1. 论文明确承认的两个主要局限

论文补充材料第 19 页列出：

1. 对逻辑异常控制有限。
2. 当参考异常很小时，self-attention grafting 表现不佳。

### 14.2. 逻辑异常的具体失败

论文讨论：

1. cable-cable swap。
2. transistor-misplaced。
3. missing transistor。

原因：

1. Mask 与具体线缆或部件对应关系必须准确。
2. 早期步骤已经形成正常部件轮廓，后期很难删除。
3. Attention 特征容易把正常和参考部件内容纠缠。
4. 简单文本模板不能完整表达连接关系。

论文提出未来可结合 MLLM，提供更丰富逻辑提示。

### 14.3. 小异常问题

Self-attention 运行在：

$$
64\times64,
32\times32,
16\times16,
8\times8.
$$

小异常下采样后边界模糊。图 14 显示真实小异常在前三 PCA 分量中几乎不可见，导致生成结果难以在小掩码中形成清晰缺陷。

### 14.4. 可以怎样改善小异常

以下属于方法扩展建议，不是论文已实现内容。

1. 使用更高分辨率 latent 或更浅高分辨率特征。
2. 引入像素空间局部 refinement network。
3. 在掩码区域做局部裁剪放大后生成，再融合回原图。
4. 使用多尺度异常特征金字塔。
5. 对小异常使用 mask-aware feature pooling，而不是全图 PCA/attention。
6. 在高分辨率卷积层注入异常特征。
7. 使用边界损失或局部对比损失。
8. 对极小掩码采用专门的 dilation 和尺度适配。

### 14.5. 可以怎样改善逻辑异常

1. 引入部件检测器，明确实体位置。
2. 构建部件关系图。
3. 使用 scene graph 或结构化 prompt。
4. 在最早去噪阶段就进行布局约束，而不是固定跳过前五步。
5. 使用 ControlNet、布局条件或关键点条件。
6. 使用 MLLM 将“misplaced”分解成可执行几何约束。
7. 对 missing-part 使用显式删除或 inpainting 分支。
8. 为线缆交换建立实例级 mask 对应，而不是单个粗掩码。

### 14.6. 改善跨类别 appearance leakage

1. 对参考异常区域做异常—材质解耦表示。
2. 使用局部归一化去除来源颜色统计。
3. 仅迁移高频残差而非完整 Value。
4. 将参考异常表示成相对正常背景的差分：

$$
\Delta V_R
=
V_R^{\mathrm{anomaly}}
-
V_R^{\mathrm{normal-context}}.
$$

5. 使用 AdaIN 或 feature whitening/coloring 对齐目标材质。
6. 在目标类别正常特征条件下重新编码异常。
7. 对来源外观泄漏加入对比或背景一致性损失。

### 14.7. 改善 AGO 的语义解耦

1. 只优化 anomaly token。
2. 对 class token 添加保持正则：

$$
\mathcal L_{\mathrm{class}}
=
\left\|
e_{\mathrm{class}}
-
e_{\mathrm{class}}^{\mathrm{ori}}
\right\|_2^2.
$$

3. 对非异常 token 添加 embedding drift 正则：

$$
\mathcal L_{\mathrm{drift}}
=
\sum_{j\notin\mathcal J^\star}
\left\|
e_j-e_j^{\mathrm{ori}}
\right\|_2^2.
$$

4. 使用掩码加权噪声损失，使优化更关注异常区域。
5. 使用多参考异常共同优化，降低单实例过拟合。
6. 加入多样性或正交约束。

### 14.8. 改善背景保持

可以显式加入背景重建约束：

$$
\mathcal L_{\mathrm{bg}}
=
\left\|
(1-M_T)
\odot
\left(
\hat I-I_N
\right)
\right\|_1.
$$

或在 latent / feature 空间：

$$
\mathcal L_{\mathrm{bg-feat}}
=
\left\|
(1-M_T^{(l)})
\odot
\left(
F_l(\hat I)
-
F_l(I_N)
\right)
\right\|_2^2.
$$

但一旦通过梯度反向优化生成过程，方法是否仍被称为 training-free 需要重新定义。

### 14.9. 计算效率改进

O2MAG 每步至少需要来源和目标多次前向，显存约 30–32GB。可考虑：

1. 只在启用 grafting 的步骤运行来源分支。
2. 预计算并压缩来源 K/V。
3. 只缓存选定层和选定 head。
4. 使用低秩或量化 K/V。
5. 将参考、正常分支合并 batch 前向。
6. CFG 使用单次拼 batch。
7. 对 AGO embedding 结果做参考级缓存。
8. 使用较少 DDIM 步数并重新调时间窗口。
9. 使用 xFormers 或 FlashAttention，但自定义掩码和跨分支 K/V 需要兼容。

### 14.10. 更全面的评价设计

一个更完整的评价体系可以包括：

1. 异常真实性：KID、人工评分、领域模型特征距离。
2. 异常多样性：IC-LPIPS、模式覆盖、参考距离分布。
3. 背景保持：掩码外 LPIPS、PSNR、SSIM。
4. 掩码一致性：异常显著图与 $M_T$ 的 IoU。
5. 异常泄漏：掩码外异常响应比例。
6. 类型一致性：真实异常分类器准确率。
7. 下游实用性：检测、定位、分类。
8. 小异常分桶：按异常面积分别统计。
9. 逻辑与纹理异常分开统计。
10. 跨类别来源泄漏评分。


## 17. 从输入到输出的完整故事

### 17.1. 生成开始前

系统拥有一台已经预训练好的 Stable Diffusion。它知道一般图像、物体、纹理和 “crack” 等概念，但不知道 MVTec 中某一种裂纹到底长什么样。

现在给它：

1. 一张真实裂纹产品图。
2. 裂纹的真实掩码。
3. 一张正常目标产品图。
4. 一个新的目标异常掩码。
5. 产品和异常类型的文本提示。

### 17.2. AGO 阶段

原始文本 embedding 只能表达通用异常。系统把参考异常图加入噪声，并要求冻结 U-Net 根据当前 embedding 预测噪声。

预测不准时，只修改 embedding。经过 500 步，embedding 变成更适合这张工业异常的条件。

### 17.3. Inversion 阶段

参考异常图和正常图分别被映射到各自 DDIM 噪声轨迹。

参考轨迹保留“怎样从噪声重建真实异常图”的中间状态；正常轨迹保留“怎样从噪声重建正常目标图”的中间状态。

目标分支复制正常图最高噪声 latent，因此起点与正常图一致。

### 17.4. 前几个去噪步骤

系统先不进行主要 self-attention grafting，让目标分支形成正常产品的大体轮廓和布局。

### 17.5. TriAG 开始

每个目标位置产生自己的 Query。

如果该位置在目标异常掩码内：

1. 它只能查看参考异常掩码内的 Key。
2. 根据相似度选择参考异常 Value。
3. 把真实异常纹理和结构带入目标位置。

如果该位置在目标掩码外：

1. 它只能查看正常图像的正常区域 Key。
2. 读取正常 Value。
3. 尽量保持原背景和产品外观。

### 17.6. DAE 增强

如果异常信号太弱：

1. Self-attention DAE 用温度和 bias 强化参考异常特征选择。
2. Cross-attention DAE 在目标掩码内把 anomaly token 影响放大 100 倍。
3. 掩码外 anomaly token 被抑制。

### 17.7. CFG 与 DDIM 更新

正提示预测使图像朝目标异常移动，负提示预测表示正常、无缺陷方向。CFG 从负方向向正方向外推。

编辑后的噪声预测用于 DDIM 更新，目标 latent 逐步变清晰。

### 17.8. 最终输出

最后 VAE decoder 将目标 latent 解码为 RGB 图像。

生成图与目标掩码组成：

$$
(\hat I,M_T).
$$

它们与少量真实异常一起用于训练分割或分类网络。

## 18. 最终核心结论

### 18.1. 方法核心

O2MAG 的最核心公式思想可以压缩成：

$$
\boxed{
Q_T
+
(K_R,V_R)
+
(K_N,V_N)
+
(M_R,M_T)
}.
$$

它保留目标 Query 的结构需求，按掩码从两个来源读取内容。

### 18.2. 三个模块的本质分工

$$
\boxed{
\text{TriAG：图像级异常特征迁移与背景保持}
}
$$

$$
\boxed{
\text{AGO：文本条件与参考工业异常语义对齐}
}
$$

$$
\boxed{
\text{DAE：增强异常可见性与掩码一致性}
}.
$$

### 18.3. 最重要的理解边界

1. O2MAG 是异常数据生成器，不是最终检测器。
2. Training-free 指不微调扩散主干，不是完全没有优化。
3. 一张参考提供一个真实异常实例，不提供完整真实分布。
4. 多样性还来自正常图、目标掩码、参考池和扩散先验。
5. 背景保持和掩码填充是经验性控制，不是严格像素保证。
6. 小异常、逻辑异常和跨类别外观泄漏仍是明显局限。
7. 论文公式与算法存在若干实现歧义，完全复现需要作者代码或额外实验核对。

### 18.4. 最直白的一句话

O2MAG 不是重新训练 Stable Diffusion 学会一种缺陷，而是在冻结模型内部重新安排信息流：

> 目标图像负责决定“哪里需要什么”，真实异常图负责提供“缺陷内容”，正常图负责提供“背景内容”，文本优化负责告诉模型“这是哪一种真实工业异常”，双注意力增强负责让这种异常在目标掩码中真正显现出来。

