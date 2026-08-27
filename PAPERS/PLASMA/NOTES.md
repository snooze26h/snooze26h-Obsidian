# PLASMA 论文实验

## 一、整体实验架构

整个实验分 4 层

```
第 4 层：评估
    ├── PLASMA（训练过的）           → train.py + evaluate.py
    ├── PLASMA-PF（无训练的）        → evaluate_pf.py
    ├── EBA（另一种对齐方法）        → evaluate_pf.py + alignment_module=eba
    ├── CosineSim（最简单的baseline） → eval_baselines.py
    └── FoldSeek / TM-align           → eval_align_baselines.py

第 3 层：对齐
    ├── η  — 残基间匹配代价函数
    └── Ω  — Sinkhorn

第 2 层：Backbone Embedding
    └── 7 个 backbone 各自把氨基酸序列 → 残基级向量

第 1 层：数据准备
    └── VenusX → 正负蛋白对 → train/val/test/test_hard
```

---

## 二、第 1 层：数据（dataset_prep.py）

### 数据

VenusX 数据集，每行一个蛋白质。以 motif.csv 为例，3401 条记录，核心字段：

- `uid`：蛋白质 ID
- `seq_full`：完整氨基酸序列
- `interpro_id`：功能家族 ID
- 功能残基标注列：哪些位置是功能残基（0/1 向量）

### InterPro 家族

InterPro 是一个蛋白质功能注释数据库，把蛋白质按功能家族分类。
同一个 InterPro ID 下的蛋白质共享某种局部功能结构
它们的整体序列可能完全不同，但在某几个关键残基上有相似的空间排布和化学功能。


前 6 个 backbone（ProtBERT、Ankh、ESM2、ProtT5、ProstT5、TM-Vec）只需要 CSV（序列信息），ProtSSN 需要 PDB（3D 坐标信息）。

### 配对构造

**正样本（label = 1）**：从同一个 InterPro 家族中随机挑两个不同 UID 的蛋白质。

**负样本（label = 0）**：从不同的 InterPro 家族中各挑一个蛋白质。

采样是均匀的：每个 InterPro 家族贡献相同数量的对，避免大家族主导。total_samples=20000，正负各 10000 对。

### test_hard 的构造

1. 把所有 InterPro 家族 ID 列出来（如 100 个家族）
2. 随机抽 10% 完全隔离出来
3. 剩下 90% → 构造 train / val / test（test_frequent）
4. 隔离的 10% → 只用于 test_hard

- **test_frequent（论文叫 test_inter）**：测试集里的功能家族在训练集中出现过
- **test_hard（论文叫 test_extra）**：测试集里的功能家族在训练集中从未出现

### 3-fold 重复

用 3 个不同的 seed（0, 42, 100），各做一次上述全部过程，产生 split_0, split_1, split_2。

---

## 三、第 2 层：Backbone Embedding（embed.py）

对每个蛋白质，用预训练的蛋白质语言模型生成残基级 embedding。

- 输入：一个蛋白质序列 MKALIVLGL...（长度 L）
- 输出：一个矩阵 [L, d]，每行是一个残基的向量表示

7 个 backbone 的关键差异：

| Backbone  | 输入                  | 模型类型                | embedding 维度 | 特点           |
| --------- | ------------------- | ------------------- | ------------ | ------------ |
| ProtBERT  | 序列                  | BERT 架构             | 1024         | 最早期，最弱       |
| Ankh-base | 序列                  | Encoder-only        | 768          | 中等规模         |
| ESM2      | 序列                  | Transformer         | 1280         | Meta 出品，流行   |
| ProtT5    | 序列                  | T5 encoder          | 1024         | 大模型，效果好，内存巨大 |
| ProstT5   | 序列                  | T5 encoder（结构增强预训练） | 1024         | 序列+结构信息      |
| TM-Vec    | ProstT5 的 embedding | Transformer         | 512          | 基于结构相似性训练    |
| ProtSSN   | 序列 + PDB 3D 坐标      | ESM2 + GNN          | 512          | 直接从原子坐标学习    |

ProstT5 和 TM-Vec 是一起生成的（TM-Vec_and_ProstT5 配置），TM-Vec 接收 ProstT5 的输出再做一次编码。

embedding 保存为两种：
- `data/embeddings/{backbone}/AA_embeddings/{uid}.pt`：残基级，[L, d]，PLASMA / PLASMA-PF / EBA 用
- `data/embeddings/{backbone}/PR_embeddings/{uid}.pt`：蛋白级，[d]，CosineSim 用（所有残基 embedding 的平均值）

---

## 四、第 3 层：PLASMA 对齐模块（alignment/）

### 完整前向传播

```
输入: H_q [Lq, d]（query 蛋白的残基 embedding）
      H_c [Lc, d]（candidate 蛋白的残基 embedding）
            ↓
      η (eta): 计算残基间匹配代价
            ↓
      Ω (omega): Sinkhorn 迭代 → 软对齐矩阵
            ↓
输出: 对齐矩阵 Ω [Lq, Lc]
            ↓
      alignment_score(): 从 Ω 计算总分 κ
```

代码：`alignment/alignment.py` 第 106-122 行
```python
def forward(self, H_q, H_c, batch_q, batch_c):
    return self.omega(self.eta(H_q=H_q, H_c=H_c, batch_q=batch_q, batch_c=batch_c))
```

### η (eta)：残基间匹配代价

代码：`alignment/interaction_nonlinearity.py`

#### Hinge（无参数版，PLASMA-PF 用）

Hinge   :  max(0, x)（取正部分）。

```python
def hinge_non_linearity(H_q, H_c, batch_q, batch_c):
    l1 = torch.cdist(H_q, H_c, p=1)          # L1 距离矩阵
    pos = (l1 + sum_q - sum_c) * 0.5          # hinge 变换
    out = -pos                                  # 取负（越近分数越高）
    return masked_out(batch_c, batch_q, out)  
```


**`torch.cdist(H_q, H_c, p=1)`**：计算两组向量之间所有两两 L1 距离。如果 query 有 m 个残基，candidate 有 n 个残基，结果是一个 m×n 的矩阵，每个格子 l1[i,j] = query 第 i 个残基和 candidate 第 j 个残基的 L1 距离。

`(l1 + sum_q - sum_c) * 0.5` ：这是一个数学恒等式加速。本质上等价于逐维度计算 `Σ max(0, H_q[i,k] - H_c[j,k])`，即只惩罚 query 比 candidate 大的维度。举例：

**Hinge 和普通 L1 距离的关键区别**：L1 距离是双向的（query 比 candidate 大了扣分，小了也扣分），Hinge 是单向的（只在 query > candidate 时扣分）。

不对称性适合蛋白质局部功能匹配。只要求 query 需要的关键特征 candidate 都有。例如：


#### LRL（有参数版，PLASMA 训练版用）

LRL = Linear → ReLU → Linear，三层堆叠。

```python
class LRL(nn.Module):
    def __init__(self, hidden_dim=512):
        self.lrl = nn.Sequential(
            nn.LazyLinear(hidden_dim),# 第 1 层：Linear（输入维度自动推断 → 512）
            nn.ReLU(),                # 第 2 层：ReLU 激活（负值变 0，引入非线性）
            nn.Linear(hidden_dim, hidden_dim)  # 第 3 层：Linear（512 → 512）
        )
        self.norm = nn.LayerNorm(hidden_dim)   # LayerNorm 归一化
    def forward(self, H_q, H_c, batch_q, batch_c):
        H_q = self.norm(self.lrl(H_q))  # 先 MLP 变换，再 LayerNorm
        H_c = self.norm(self.lrl(H_c))  # query 和 candidate 共享同一个网络
        return hinge_non_linearity(H_q, H_c, batch_q, batch_c)
```

**举例（以 ESM2 为例，embedding 维度 1280，hidden_dim=512）：**

```
输入: H_q[i] = [x1, x2, ..., x1280]    ← 一个残基的原始 embedding，1280 维

第 1 步 LazyLinear:  1280 维 → 512 维   ← 线性变换（矩阵乘法 + 偏置）
第 2 步 ReLU:        负值变成 0          ← 引入非线性
第 3 步 Linear:      512 维 → 512 维    ← 再做一次线性变换
第 4 步 LayerNorm:   归一化到均值0方差1   ← 稳定训练

输出: H_q'[i] = [y1, y2, ..., y512]    ← 变换后的 embedding，512 维
```

**LazyLinear**：普通 Linear 提前知道输入维度，但不同 backbone 的 embedding 维度不同LazyLinear 在第一次前向传播时自动确定输入维度。

**LayerNorm**：把一个向量的数值调整到均值为 0、方差为 1。LayerNorm 拉到同一尺度

**Siamese（共享参数）**：query 和 candidate 用的是同一个 LRL 网络。


**LRL 和 Hinge 的关系**：

```
PLASMA:    embedding → [LRL 变换] → 新 embedding → [Hinge 计算距离]
PLASMA-PF: embedding → [Hinge 直接计算距离]
```


### Ω (omega)：Sinkhorn 最优传输

代码：`alignment/interaction_structures.py`

```python
class Sinkhorn(nn.Module):
    def __init__(self, n_iters=20, temperature=0.1):
        ...
    def forward(self, eta_result):
        K = (eta_result[0] * self.temperature).exp_()
        for _ in range(self.n_iters):
            u = 1.0 / (K @ v + eps)    # 归一化行
            v = 1.0 / (K.T @ u + eps)  # 归一化列
        return diag(u) @ K @ diag(v)   # 双随机矩阵
```


#### 为什么需要 Sinkhorn

η 输出的代价矩阵取 exp 后变成核矩阵 K，但 K 有问题：多个 query 残基可能都想对齐到同一个 candidate 残基，导致某些 candidate 残基被"抢"，其他被冷落。

Sinkhorn 把 K 变成双随机矩阵（行列之和都为 1），保证对齐关系是均衡的——每个 query 残基公平分配注意力，每个 candidate 残基也公平被关注。

#### temperature 的作用

temperature=0.1 让 exp 之前的值乘以 0.1，效果是让分布更尖锐：

### alignment_score()：从 Ω 算总分 κ

代码：`utils/alignment_utils.py` 第 46-56 行

```
κ = 语义相似度 s × 对角线置信度 α
```

**语义相似度 s**：
1. 找出被对齐上的残基（对齐矩阵列/行最大值 > threshold=0.5）
2. 用对齐矩阵"搬运" query 的 embedding 到 candidate 空间
3. 算搬运后的 query 表示和 candidate 表示的余弦相似度

**余弦相似度**：衡量两个向量方向是否一致，不关心长度。范围 -1（完全相反）到 1（完全一致）。

**对角线置信度 α**（max_main_diag_score）：
- 用 K×K（默认10×10）的单位矩阵做卷积核在 Ω 上滑动
- 取最大值 / K
- 连续斜对角线 → α ≈ 1.0（真正的局部结构相似）
- 散乱的高分点 → α ≈ 0.3（碰巧像，不是真相似）

最终：κ = s × α。

**κ  意义**：训练前 κ 是随机的，正负样本分数混在一起。训练过程中 BCE loss 逼迫正样本的 κ 趋近 1、负样本趋近 0。反复训练后 κ 就能把正负样本分开。

---

## 五、第 4 层：各种方法的评估

### 方法 1：PLASMA（训练版）— train.py

**流程**：加载蛋白对 + embedding → η(LRL) → Ω(Sinkhorn) → alignment_score → κ → 两个 loss 反向传播

**训练的两个 loss**（train.py 第 374-397 行）：

```python
# Loss 1: BCE — 蛋白对级别的分类
alignment_loss = BCEWithLogitsLoss(κ, label)
# 正样本 label=1 → κ 趋近 1
# 负样本 label=0 → κ 趋近 0

# Loss 2: LML — 残基级别的定位
label_loss = label_match_loss(query_targets, candidate_targets, Ω)
# 检查标注的功能残基是否真的通过 Ω 被对齐上了

total_loss = alignment_loss + label_loss
```

**LML (Label Match Loss) 的具体计算**：

```python
def label_match_loss(l1, l2, alignment, is_match=True):
    if is_match:
        return torch.clamp(l2 - l1 @ alignment, min=0).sum() / l2.sum()
    else:
        return 0.0  # 负样本不算
```

- `l1`：query 的功能残基标签，如 [1, 0, 1, 0]
- `l2`：candidate 的功能残基标签，如 [0, 1, 0, 1]
- `l1 @ Ω`：把 query 的功能标签通过对齐矩阵"搬运"到 candidate 空间（`@` 是矩阵乘法）
- `l2 - l1 @ Ω`：candidate 功能位置减去搬运过来的信号
- `clamp(min=0)`：只保留正的部分（只惩罚"candidate 功能残基没被覆盖"的情况）
- 除以 `l2.sum()` 归一化：不管功能残基有多少个，loss 都在 0~1 之间

举例：如果对齐正确，搬运过来的信号恰好覆盖 candidate 的功能位置 → 差值为 0 → loss 为 0。如果对齐错误，搬运到了错误位置 → 差值大于 0 → loss 大。


早停：验证集 loss 连续 3 轮不改善就停止。训练完后自动在 test 和 test_hard 上评估。

### 方法 2：PLASMA-PF（无训练版）— evaluate_pf.py

和 PLASMA 的唯一区别：η 用 Hinge 而不是 LRL。

```
PLASMA:    embedding → LRL(变换) → Hinge(算距离) → Sinkhorn
PLASMA-PF: embedding → Hinge(直接算距离)          → Sinkhorn
```


不需要训练，直接在测试集上推理。

**意义**：PLASMA 比 PLASMA-PF 好多少 = 训练 LRL 带来的提升。


### 方法 3：EBA — evaluate_pf.py + alignment_module=eba

EBA (Embedding-Based Alignment) 是另一篇论文提出的方法。


**流程**：

第 1 步：对每对残基算余弦相似度，得到相似度矩阵。

第 2 步：直接在矩阵上统计打分（不经过 Sinkhorn）：
- `EBA_raw`：矩阵中所有值的平均
- `EBA_max`：每行取最大值，再求平均
- `EBA_min`：每行取最小值，再求平均

   例子：
```
        c0    c1    c2    每行最大值
q0    [ 0.9   0.2   0.1 ]  → 0.9
q1    [ 0.3   0.8   0.1 ]  → 0.8
q2    [ 0.1   0.2   0.7 ]  → 0.7

EBA_max = (0.9 + 0.8 + 0.7) / 3 = 0.8
```

意思是：每个 query 残基找到和它最像的 candidate 残基，取那个最高分，再求平均。（论文默认用
EBA_max）

**EBA 和 PLASMA-PF 的区别**：

|       | PLASMA-PF             | EBA                           |
| ----- | --------------------- | ----------------------------- |
| 相似度计算 | Hinge（单向 L1）          | 余弦相似度（对称）                     |
| 对齐方式  | Sinkhorn（行列约束，趋近一对一）  | 无对齐（每行独立取 max）                |
| 列约束   | 有（每个 candidate 不会被独占） | 无（多个 query 可以都选同一个 candidate） |

**EBA 的问题**：没有列约束，如果 candidate 有一个"万能"残基和谁都像，所有 query 残基的 max 都指向它，其他 candidate 残基被忽略。可能导致假阳性（把不相似的判为相似）。PLASMA-PF 通过 Sinkhorn 的列约束避免了这个问题。


### 方法 4：CosineSim — eval_baselines.py

最简单的 baseline。不做任何对齐，直接比较 **蛋白级 embedding** 的余弦相似度。

用的是蛋白级（PR_embeddings）embedding——整个蛋白压缩成一个向量（所有残基 embedding 的平均值），不是残基级的。

然后两个蛋白各一个向量，直接算余弦相似度得到一个分数。

**意义**：证明残基级对齐是必要的。如果 CosineSim 就能做得很好，PLASMA 就没有存在价值。

### 方法 5：FoldSeek / TM-align — eval_align_baselines.py

传统的 3D 结构比对工具，完全不同的路线：

- **TM-align**：把两个蛋白的 3D 原子坐标做刚体叠合（旋转+平移），尽量重叠，算 TM-score。优点是物理意义最明确，缺点是每一对都要做优化计算，很慢。
- **FoldSeek**：先把每个残基的局部 3D 几何特征编码成"结构字母"，把 3D 结构变成一维序列，再用快速的序列比对算法。比 TM-align 快，但编码过程丢失一些 3D 信息。

**意义**：证明 embedding 路线比 3D 路线更快更准。

---

## 六、各方法横向对比

| 方法        | 需要训练？      | 用什么 embedding？ | 对齐方式             | 代码入口                    |
| --------- | ---------- | -------------- | ---------------- | ----------------------- |
| PLASMA    | 需要（LRL 参数） | AA 级（残基）       | LRL → Sinkhorn   | train.py                |
| PLASMA-PF | 不需要        | AA 级（残基）       | Hinge → Sinkhorn | evaluate_pf.py          |
| EBA       | 不需要        | AA 级（残基）       | 余弦矩阵 → EBA 统计    | evaluate_pf.py + eba    |
| CosineSim | 不需要        | PR 级（蛋白整体）     | 无对齐，直接余弦         | eval_baselines.py       |
| TM-align  | 不需要        | 不用 embedding   | 3D 坐标刚体叠合        | eval_align_baselines.py |
| FoldSeek  | 不需要        | 不用 embedding   | 结构字母表比对          | eval_align_baselines.py |


---

## 七、评估指标

### ROCAUC

遍历所有可能的阈值，看每个阈值下模型能不能把正负样本分开，画 ROC 曲线，算面积。

- ROCAUC = 1.0：所有正样本的分数都高于所有负样本，存在一个阈值能完美分开
- ROCAUC = 0.5：正负样本的分数完全混在一起，约等于瞎猜

### F1-max

先解释 F1。给定一个阈值，分数高于阈值的判为正：

```
精确率 Precision = 判对的正样本数 / 判为正的总数（我说"相似"的里面有多少真的相似）
召回率 Recall    = 判对的正样本数 / 真正的正样本总数（所有真正相似的里面我找到了多少）
F1 = 2 × P × R / (P + R)（精确率和召回率的调和平均）
```

阈值不同 F1 就不同。F1-max = 遍历所有阈值，取 F1 最大的那个。意义：模型在最佳条件下能达到的 F1。

### PR-AUC

和 ROCAUC 类似，但画的是 Precision-Recall 曲线。每个阈值算一组 (Recall, Precision)，Recall 横轴 Precision 纵轴，连成曲线，算面积。

好模型的曲线贴近左上角（R=1, P=1 即零误报零漏报），面积大。差模型的曲线往右下塌，面积小。

PLASMA 的数据正负 1:1 平衡，所以 PR-AUC 和 ROCAUC 差别不大。

### Label Match Score

前三个指标评估蛋白对级别判断对不对，Label Match Score 评估残基级别对齐准不准。

只对正样本计算，因为负样本没有共享功能结构。

---

## 八、三类任务

| 任务           | 含义                     | 难度  |
| ------------ | ---------------------- | --- |
| motif        | 序列/结构模体，如锌指结构中结合锌离子的残基 | 最难  |
| binding_site | 蛋白质和配体结合的口袋里的残基        | 较易  |
| active_site  | 酶催化反应时直接参与化学反应的残基      | 较易  |

---

## 九、代码文件地图

```
dataset_prep.py          → 数据下载、配对、划分
embed.py                 → backbone embedding 生成
train.py                 → PLASMA 训练（LRL + Sinkhorn + BCE + LML）
evaluate.py              → 对已训练的 PLASMA 模型做测试集评估
evaluate_pf.py           → PLASMA-PF 和 EBA 评估（无训练）
eval_baselines.py        → CosineSim baseline（蛋白级余弦相似度）
eval_align_baselines.py  → FoldSeek / TM-align（3D 结构比对）

alignment/
├── alignment.py                → Alignment 类：组合 η + Ω
├── interaction_nonlinearity.py → η 的实现：Hinge / LRL / DotProduct
├── interaction_structures.py   → Ω 的实现：Sinkhorn
└── utils.py                    → LayerNorm 等辅助

utils/
├── alignment_utils.py          → alignment_score()、label_match_loss()
├── data_utils.py               → 数据加载、pair→dataset 转换
└── time_utils.py               → 计时工具

configs/
├── train.yaml                  → 训练配置（lr、epoch、patience 等）
├── evaluate_pf.yaml            → PLASMA-PF / EBA 评估配置
├── baselines.yaml              → CosineSim baseline 配置
├── alignment_module/
│   ├── plasma.yaml             → eta=lrl, omega=sinkhorn（训练版）
│   ├── plasma_pf.yaml          → eta=hinge, omega=sinkhorn（无训练版）
│   └── eba.yaml                → EBA 配置
└── backbone/                   → 7 个 backbone 的模型配置
```

---

### 超参（config 中手动设置）

| 超参数           | 默认值  | 在哪改        | 影响               |
| ------------- | ---- | ---------- | ---------------- |
| hidden_dim    | 512  | eta 配置     | LRL 的隐藏层宽度       |
| temperature   | 0.1  | omega 配置   | Sinkhorn 对齐的尖锐程度 |
| n_iters       | 20   | omega 配置   | Sinkhorn 迭代次数    |
| K             | 10   | score 配置   | 对角线卷积核大小         |
| threshold     | 0.5  | score 配置   | 判定"匹配上"的阈值       |
| learning_rate | 1e-4 | train.yaml | 步长               |
| patience      | 3    | train.yaml | 早停耐心值            |
