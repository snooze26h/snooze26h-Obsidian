# NMR 当前阶段工作总结（2026-03-24）

## 1. 当前这批工作主要做了什么

这一阶段主要沿着三条线推进：

1. `ensemble` 结果整理与能力边界分析          
2. `hard residue` 难残基误差分析
3. `feature ablation` 结构特征消融实验

这三条线的目标并不完全一样：
- `ensemble` 线主要回答“简单集成还能不能继续提升”
- `hard residue` 线主要回答“模型到底在哪些化学场景下失败”
- `feature ablation` 线主要回答“哪些额外结构特征真的有用”

目前这三条线已经能拼出一个比较清楚的结论框架。

---

## 2. Ensemble 这条线目前的结果

### 2.1 已有结果

- 最强单模型（旧主线，BEST 8维，`seed_42`）
  - Test MAE = `0.2886`
- simple seed ensemble
  - `0.05(seed42) + 0.05(seed123) + 0.05(seed456)`
  - Test MAE = `0.2742`
- diversity ensemble
  - `0.05(seed42) + 0.10(seed42) + 0.20(seed42)`
  - Test MAE = `0.2759`
- 当前最佳混合 ensemble（Mix A）
  - `0.05(seed42) + 0.05(seed123) + 0.10(seed42)`
  - Test MAE = `0.2735`

- `RING_CURRENT` 3-seed ensemble（当前最好结果）
  - `RING_CURRENT(seed42) + RING_CURRENT(seed123) + RING_CURRENT(seed456)`
  - Test MAE = `0.2728`

- Mix C
  - `0.05(seed42) + 0.05(seed123) + baseline_no_augment`
  - Test MAE = `0.2752`

### 2.2 当前判断

- ensemble 确实有效，但已经接近 simple averaging 的收益上限
- 当前最好的 ensemble 结果已经更新为 `RING_CURRENT` 3-seed：`0.2728`
- 这比此前最好的 `Mix A = 0.2735` 还低 `0.0007`
- diversity 不是完全没用，但弱模型（如 `0.20`）会把平均值拉偏
- 继续盲目试更多拼法，收益大概率会越来越小

### 2.3 ensemble 分析给出的关键发现

- `RING_CURRENT` 3-seed ensemble 把最终主结果推到了 `0.2728`，证明 feature 改进不仅改善单模型，也能改善最终 ensemble
- 但模型间误差相关性依然很高（`0.8576–0.8599`），说明模型之间仍然不够“独立”
- variance 对误差有一定预测能力，但样本级别相关性依然不强（`Spearman = 0.1245`）
- protein 级别 variance 仍有解释力（`Spearman = 0.3737`），更适合做“蛋白整体可靠性判断”
- ensemble 对长尾最差样本帮助依然很有限，甚至更差（top 5% improvement = `-0.1283`）

### 2.4 这条线目前的意义

这条线已经证明：
- 简单 ensemble 能稳定降 MAE
- `RING_CURRENT` 是目前第一条被证明“能同时改善单模型和最终 ensemble”的 feature 主线
- 但 simple averaging 的机制主要还是“平滑随机误差”，不是“逐样本选出最对的模型”
- 如果后面还想继续压 ensemble，方向应该是更强单模型或更强 diversity，而不是继续随便混模型

---

## 3. Hard Residue 这条线目前的结果

### 3.1 这条线是什么时候做的

这一条不是新的训练线，而是在 `ensemble` 跑完之后追加做的误差分析线。

执行顺序是：
1. 先完成 `models/ensemble_H/` 下面的多模型 prediction 和 ensemble 分析
2. `analyze_ensemble.py` 生成合并后的逐样本文件 `models/ensemble_H/analysis/merged_predictions.csv`
3. 再用 `analyze_hard_residues.py` 读取这个 merged 文件，专门做残基层面的误差拆解

对应脚本和输入输出如下：
- 脚本：`SideChain_Base/train_model/analyze_hard_residues.py`
- 输入：`models/ensemble_H/analysis/merged_predictions.csv`
- 输出目录：`models/ensemble_H/hard_residue_analysis/`

也就是说，这条线的本质不是“再训一个模型”，而是“把已经得到的 ensemble 结果拆开看，判断模型到底在哪些残基上失败、失败模式是什么”。

### 3.2 这条线具体分析了什么

这条分析线一共做了 6 类内容：

1. 每种残基的误差分布
2. 样本数 vs MAE 的相关性
3. `MET` 的 variance anomaly 深挖
4. hard residue vs easy residue 的误差分布对比
5. 每个 hard residue 的 worst predictions 列表
6. cross-protein consistency，判断“这个残基是普遍难，还是只在特定蛋白里难”

这 6 部分里，对结论最关键的是两段：
- `Per-Residue Error Distribution`
- `Cross-Protein Consistency`

前者告诉我们“谁最难”，后者告诉我们“这种难是系统性的还是环境依赖的”。

### 3.3 当前识别出的 hard residues

从 `Per-Residue Error Distribution` 看，当前主要识别出的 hard residues 包括：
- `HIS`
- `CYS`
- `TRP`
- `ASP`
- `GLY`

当时的代表性数值如下：
- `HIS`: Mean MAE = `0.3254`
- `TRP`: Mean MAE = `0.3122`
- `CYS`: Mean MAE = `0.3010`
- `GLY`: Mean MAE = `0.2903`
- `ASP`: Mean MAE = `0.2883`

这些残基的误差不只是均值偏高，尾部也更重。例如：
- `HIS` 的 `P95 = 1.0165`
- `CYS` 的最大误差达到 `4.7308`
- `TRP` 的 `P90 = 0.6235`

所以这里看到的不是偶然波动，而是明确的“难残基簇”。

### 3.4 关键分析结论

#### 1. 系统性难的残基

- `HIS`
- `ASP`
- `GLY`

这几个残基在 `Cross-Protein Consistency` 里都表现出相对较低的 CV：
- `HIS`: `CV = 0.43`
- `ASP`: `CV = 0.43`
- `GLY`: `CV = 0.43`


- 不同蛋白里，这些残基都偏难
- 问题不像是某几个特殊蛋白把结果拉坏了
- 更像是模型对这类残基缺少关键输入信息或关键建模能力。。。

所以这几个残基更适合被理解为 `residue-specific / systemic bottleneck`。

#### 2. 环境依赖型难残基

- `CYS`
- `TRP`

它们在 `Cross-Protein Consistency` 中的 CV 明显更高：
- `CYS`: `CV = 0.63`
- `TRP`: `CV = 0.68`


- 某些蛋白里预测得还可以
- 但某些蛋白里会明显失败
- 难度高度依赖具体局部环境，而不是残基本身固定难

所以这两个残基更适合被理解为 `context-specific bottleneck`。

#### 3. 样本数只解释了一部分问题

`Sample Count vs MAE` 的结果是：
- `Spearman r = -0.4509`
- `p = 0.0527`

这说明：
- 数据稀缺可能部分解释某些残基难预测
- 但它不是完整答案

也就是说，`HIS/TRP/CYS` 难，并不只是因为样本少；背后更可能有真实的化学或环境建模缺口。

#### 4. MET variance anomaly 是一个“分歧异常”而不是“误差异常”

`MET` 的结果比较特殊：
- `MET mean variance = 0.125914`
- `Others mean variance = 0.019403`
- 比值约 `6.5x`

但 `MET` 的平均 MAE 并不在最难的一档：
- `MET Mean MAE = 0.2651`

这说明：
- 模型在 `MET` 上分歧很大
- 但这种分歧不总是转化为更大的最终误差

因此这里给出的重要提醒是：
- ensemble variance 是一个 disagreement proxy
- 不能直接把它等同于“真实误差”或者“严格 uncertainty”

### 3.5 这些结果能说明什么化学问题

从数据结果出发，可以形成一个比较自然的化学解释框架。

#### 1. `HIS` 难，可能和质子化状态有关

`HIS` 的 `pKa` 接近生理条件，质子化状态容易变化。
如果模型输入只看静态 3D 结构，而没有显式表示 pH / protonation state，那么：
- 同样几何下，真实 chemical shift 可能差很多
- 模型就会把这部分真实化学差异当成噪声

#### 2. `ASP` 难，也可能和电离状态有关

`ASP` 虽然侧链更小，但其电荷状态会受微环境影响。
如果没有显式建模 protonation / electrostatics，系统性偏差就容易出现。

#### 3. `GLY` 难，可能和构象自由度有关

`GLY` 没有侧链，chemical shift 更依赖骨架构象。
同时它的 Ramachandran 空间更大、构象更灵活，因此模型需要更强的局部构象表示能力。

#### 4. `CYS` 难，可能和二硫键/局部环境有关

`CYS` 是否形成二硫键，会显著影响其局部电子环境。
高 CV 说明：
- 有些蛋白里的 `CYS` 模型能处理
- 有些环境下完全不行

这更像环境依赖的失败，而不是“所有 `CYS` 都难”。

#### 5. `TRP` 难，可能和芳香堆积环境有关

`TRP` 是大芳香侧链，ring current 和邻近芳香残基排布会明显影响 shift。
高 CV 说明模型对一些局部芳香环境能处理，但对另一些环境的表示还不够细。

### 3.6 这条线目前的意义

这条线的价值不在于直接降 MAE，而在于解释误差来源。它至少回答了三个问题：

1. 哪些残基是真的难
2. 这种“难”是系统性的，还是依赖具体蛋白环境
3. 模型失败更像是缺输入信息，还是缺局部环境表示能力

从论文角度看，这条线非常重要，因为它让结果从“报一个整体 MAE”升级成“理解模型为什么失败”。

### 3.7 当前这条线可以沉淀下来的结论

当前可以沉淀成比较稳定的三点：

1. `HIS/ASP/GLY` 更像系统性建模缺口
2. `CYS/TRP` 更像环境依赖型建模缺口
3. ensemble variance 能提示模型分歧，但不能直接等价于误差本身

因此，`hard residue` 这条线给后续工作的启发是：
- 如果想解决 `HIS/ASP`，更可能要补的是 protonation / electrostatics 这类缺失信息
- 如果想解决 `CYS/TRP`，更可能要补的是更细的局部化学环境表示
- 如果只是继续做 simple ensemble，它并不能解决这些系统性问题

---

## 4. Feature Ablation 这条线目前的结果

### 4.1 已完成的最小特征矩阵

当前已经比较过这些特征组：
- `baseline` = `NONE`
- `A_ring_current` = `RING_CURRENT`
- `B_blosum62` = `BLOSUM62`
- `C_ss8` = `SS8`
- `E_best` = `BEST`
- `G_ring_ss8` = `RING_CURRENT + SS8`
- `H_ring_blosum62` = `RING_CURRENT + BLOSUM62`
- `I_ss8_blosum62` = `SS8 + BLOSUM62`
- `D_ring_blosum62_ss8` = `RING_CURRENT + BLOSUM62 + SS8`
- `F_struct67` = `STRUCT67`

### 4.2 当前排序（按 test MAE）

1. `A_ring_current` = `0.2881`（只有一维）
2. `E_best` = `0.2886`
3. `G_ring_ss8` = `0.2906`
4. `C_ss8` = `0.2908`
5. `I_ss8_blosum62` = `0.2931`
6. `D_ring_blosum62_ss8` = `0.2934`
7. `H_ring_blosum62` = `0.2939`
8. `F_struct67` = `0.2944`
9. `baseline` = `0.2946`
10. `B_blosum62` = `0.2983`

### 4.3 核心发现

#### 1. `RING_CURRENT` 是当前唯一明确有效的物理先验

- 单独使用 `RING_CURRENT` 就达到全表最优
- 比 baseline 明显更好
- 比 `BEST` 还略好一点

这说明当前真正有贡献的，很可能就是环电流相关信息。

#### 2. `BEST` 组合并没有超过纯 `RING_CURRENT`

- `BEST = 0.2886`
- `RING_CURRENT = 0.2881`

这说明 `BEST` 中其余特征没有提供额外净收益，甚至可能引入轻微噪声。

#### 3. `SS8` 有一点帮助，但不是主信号

- `SS8` 单独有轻微增益
- `RING_CURRENT + SS8` 排名第 3

这说明二级结构可能提供弱辅助信息，但并不是主要突破点。

#### 4. `BLOSUM62` 基本无效，甚至有害

- 单独 `BLOSUM62` 比 baseline 更差
- 加入 `BLOSUM62` 的组合整体也不强

这说明序列替换先验在当前 PaiNN + 3D 表示框架里，没有转化成有效增益。

#### 5. `STRUCT67` 不值得

- `STRUCT67` 几乎和 baseline 持平
- 67 维高维特征没有体现出应有收益

这说明“堆更多通用特征”并没有解决问题，很多维度可能是冗余信息。

### 4.4 `RING_CURRENT` 收益来源分析（baseline vs A_ring_current）

在确定 `RING_CURRENT` 是当前最优单特征之后，又额外做了一次 `benefit source analysis`，目的不是再比较总 MAE，而是回答一个更细的问题：

> `RING_CURRENT` 到底帮了谁？

这一步比较的是两个模型：
- baseline：`NONE`
- target：`A_ring_current`

分析脚本：`SideChain_Base/train_model/analyze_benefit_source.py`

输入：
- baseline checkpoint + baseline test cache
- `A_ring_current` checkpoint + `A_ring_current` test cache

输出内容包括：
1. per-residue MAE 对比
2. top improved / degraded proteins
3. aromatic vs non-aromatic summary
4. 误差分位数对比（P50/P90/P95/P99）

#### 1. 整体结果

这次比较的总体结果是：
- Baseline MAE: `0.2940`
- `A_ring_current` MAE: `0.2883`
- 绝对提升：`+0.0057`

同时逐样本统计显示：
- Improved: `8587 (44.9%)`
- Degraded: `8326 (43.6%)`

`RING_CURRENT` 的收益机制不是几乎所有样本都小幅变好，而是：
- 有相当一部分样本明显改善
- 也有一部分样本退步
- 但正收益总体更强，最终把总 MAE 拉低

#### 2. 芳香族是否受益最大

这个问题原本的直觉是：
- 既然这是 `RING_CURRENT` 特征
- 那它应该主要帮助芳香族残基

但这次实际结果比这个直觉更复杂。

芳香族分组结果：
- Aromatic (`PHE/TRP/TYR/HIS`): `0.3191 -> 0.3129`, `Δ=+0.0061`
- Non-aromatic: `0.2908 -> 0.2852`, `Δ=+0.0057`

这两个提升非常接近。
因此当前不能简单写成：
- “`RING_CURRENT` 主要帮助芳香族残基”

更稳的表述应该是：
- `RING_CURRENT` 带来整体增益，但其收益并不只局限于芳香族残基

#### 残基层面的具体表现

这一步最值得看的，是 per-residue 的提升排序。

改善最明显的一些残基包括：
- `HIS`: `0.3510 -> 0.3250`, `Δ=+0.0259`
- `CYS`: `0.3288 -> 0.3167`, `Δ=+0.0121`
- `ASN`: `0.2892 -> 0.2773`, `Δ=+0.0118`
- `GLN`: `0.2892 -> 0.2775`, `Δ=+0.0117`
- `ALA`: `0.2877 -> 0.2771`, `Δ=+0.0105`
- `GLU`: `0.2955 -> 0.2852`, `Δ=+0.0103`
- `MET`: `0.2754 -> 0.2652`, `Δ=+0.0102`
- `PHE`: `0.3063 -> 0.2981`, `Δ=+0.0082`


第一，`HIS` 的提升非常明显。
这很重要，因为 `HIS` 本来就在 hard residue 分析里是最难的一类残基。

第二，很多非芳香族残基也有实质提升。
这说明 `RING_CURRENT` 这个特征在模型里起到的作用，不只是“直接修正芳香族 shift”，还可能提供了一种局部电子环境或空间环境信号，帮助模型整体判断。

#### 4. 芳香族内部并不一致

虽然 `HIS` 和 `PHE` 确实受益，但芳香族内部并不是全部改善：
- `TYR`: `0.3031 -> 0.3063`, `Δ=-0.0031`
- `TRP`: `0.3466 -> 0.3542`, `Δ=-0.0076`

这说明：
- `RING_CURRENT` 不是简单地“只要是芳香族就收益”
- 它更像是对某些具体局部环境有帮助
- 不同芳香残基、不同局部构型，对这个特征的响应并不一致

所以从论文写法上，当前更好的表述是：
- `RING_CURRENT` 对部分芳香环境有明显帮助
- 但它的收益机制不能被简单概括成“专门帮助芳香族残基”

#### 5. 对蛋白层面的影响

Top improved proteins 里，部分蛋白获得了非常明显的收益，例如：
- `1O82D`: `+0.1661`
- `1KTZB`: `+0.1402`
- `2CG7A`: `+0.0569`
- `7RXN_`: `+0.0562`

但也存在明显退步的蛋白，例如：
- `2GM5D`: `-0.0859`
- `1NBPA`: `-0.0474`
- `1DCDB`: `-0.0471`

这说明：
- `RING_CURRENT` 不是“所有蛋白都统一小幅变好”
- 它对某些蛋白帮助很大，对另一些蛋白则会带来反作用
- 这种模式再次说明，该特征的作用更像是针对某些特定局部环境的补充信息，而不是 universally helpful signal

#### 6. 对误差分布尾部的影响

误差分位数结果如下：
- `P50`: `+0.0045`
- `P90`: `+0.0068`
- `P95`: `+0.0142`
- `P99`: `+0.0007`

这说明：
- `RING_CURRENT` 对中位误差和高分位误差都有帮助
- 特别是 `P95` 的改善比较明显
- 但对最极端的 outlier (`P99`) 帮助很有限

因此它更像是：
- 能改善“比较难但还没到最极端”的样本
- 但不能解决最坏的极端错误

#### 7. 当前可沉淀的结论

从这次收益来源分析中，可以比较稳地沉淀出四点：

1. `RING_CURRENT` 带来了真实且稳定的整体增益
2. 它的收益不只局限于芳香族残基
3. `HIS` 和 `PHE` 明显受益，但 `TRP` 和 `TYR` 并没有一致改善
4. 它对中高误差区间有帮助，但对最极端 outlier 帮助有限

因此，当前对 `RING_CURRENT` 最稳妥的理解是：

> 它不是一个“专门给芳香族加标签”的特征，而更像是向模型提供了一种与局部电子环境相关的补充信号，从而在一部分关键样本上带来明显收益。

### 4.6 `RING_CURRENT` 3-seed ensemble 结果

在确认 `RING_CURRENT` 是当前最优单特征之后，进一步以该配置训练了 3 个 seed：
- `seed_42`: `0.2882`
- `seed_123`: `0.2906`
- `seed_456`: `0.2883`

基于这 3 个 seed 的 mean ensemble，得到：
- Ensemble MAE = `0.2728`
- Best single = `0.2882`
- Improvement = `+0.0154`

这使得 `0.2728` 成为当前整条实验线上的最好结果，并且：
- 优于此前的 simple seed ensemble `0.2742`
- 优于此前最好的混合组合 `Mix A = 0.2735`

#### 1. 这次结果的意义

这一步非常关键，因为它证明了：
- `RING_CURRENT` 不只是把单模型做得更好
- 它还能把最终 ensemble 结果继续往下推

也就是说，这一轮 feature ablation 不是“只找到一个解释性结论”，而是真正导向了新的主结果。

#### 2. 这次 ensemble 的结构特征

虽然结果变好了，但 ensemble 的基本机制并没有改变：
- Prediction correlation: `0.9447–0.9458`
- Error correlation: `0.8576–0.8599`
- Ensemble better ratio: `10.8%`
- Tail (top 5%) improvement: `-0.1283`

这些数字说明：
- 3 个 seed 非常相似，相关性比之前还高
- ensemble 的收益依然主要来自均值平滑，而不是逐样本选择最优模型
- 长尾最差样本上，simple averaging 依旧无能为力，甚至会更差

因此，这个 `0.2728` 的提升是真实的，但也已经再次提示：
- 这条 simple averaging 路线正在接近极限

#### 3. 对 hard residues 的影响

在 `RING_CURRENT` 3-seed ensemble 中，hard residues 仍然主要集中在：
- `TRP`: `0.3242`
- `HIS`: `0.3135`
- `CYS`: `0.2905`
- `TYR`: `0.2904`
- `ASP`: `0.2886`
- `GLY`: `0.2854`

但与各单模型均值相比，它们大多仍然从 ensemble 中获益：
- `HIS`: `+0.0113`
- `CYS`: `+0.0166`
- `TYR`: `+0.0148`
- `PHE`: `+0.0158`
- `GLY`: `+0.0114`

唯一比较顽固的是 `TRP`：
- `vs_best = -0.0021`

这说明：
- `RING_CURRENT` 主线能够改善大多数 hard residue 的平均表现
- 但像 `TRP` 这样的困难点仍然没有被根本解决

#### 4. 当前可沉淀下来的结论

当前关于这一步可以沉淀成三点：

1. `RING_CURRENT` 已经同时证明了单模型价值和最终 ensemble 价值
2. 当前最好结果可以更新为：`0.2728`
3. simple averaging 的瓶颈依然存在，因此后续如果还想继续提升，重点应该转向更强 base learner 或更高质量 diversity，而不是继续大范围试拼法

---

## 5. 目前好的结论和不好的结论

### 5.1 效果好的结论

- simple seed ensemble 有稳定收益
- 当前最好的 ensemble 结果已经更新为 `RING_CURRENT` 3-seed：`0.2728`
- `RING_CURRENT` 是当前最有价值的单特征，而且已经被证明能改善最终 ensemble
- hard residue 分析已经能解释模型失败的主要方向
- 这批结果已经足够支撑论文中的一部分 Results + Discussion

### 5.2 效果差或应该先停的方向

- `BLOSUM62` 暂时不值得继续投入
- `STRUCT67` 暂时不值得继续投入
- simple averaging 的大范围乱拼收益已经很有限
- 长尾 worst-case 样本上，ensemble 不能解决根本问题

---

## 6. 当前最清楚的总判断

到目前为止，可以把整体结论概括成三句话：

1. `ensemble` 有用，但 simple averaging 已接近天花板
2. `hard residue` 问题说明模型仍存在 residue-specific 和 context-specific 的建模缺口
3. `feature ablation` 说明当前真正有效的额外结构信息，核心是 `RING_CURRENT`

目前这个判断已经被进一步验证：
- 用 `RING_CURRENT` 这个更强的单模型配置重新做 multi-seed ensemble 后，结果达到了新的最好值 `0.2728`
- 这说明 `RING_CURRENT` 的价值不只体现在单模型，也体现在最终主结果上
- 但 ensemble 的结构性瓶颈并没有改变，高误差相关性和长尾失败仍然存在

---

## 7. 当前最务实的下一步

当前最推荐的直接下一步原本是：

1. 用 `RING_CURRENT` 作为唯一结构特征
2. 跑 `seed_42 / 123 / 456`
3. 导出三份 prediction csv
4. 做新的 multi-seed mean ensemble
5. 和当前最优 `Mix A = 0.2735` 正面对比

而这一步现在已经完成，结果是：
- `RING_CURRENT` 3-seed ensemble = `0.2728`
- 成功超过此前的 `Mix A = 0.2735`

因此下一步不再是“验证这条线是否成立”，而是：
- 接受 `0.2728` 为当前主结果
- 把对应分析整理进论文结果与讨论部分
- 后续只做少量高价值补充实验，而不是继续大范围扫 ensemble 组合

---

## 8. 当前一句话总结

这阶段工作的核心收获是：

> 先通过 ensemble 和 hard residue 分析看清模型边界，再通过 feature ablation 找到真正有效的结构先验；目前这条主线已经落到新的最好结果：`RING_CURRENT` 3-seed ensemble = `0.2728`。
