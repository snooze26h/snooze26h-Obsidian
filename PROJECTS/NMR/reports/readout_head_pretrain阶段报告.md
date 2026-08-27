# Pretrain 阶段 Readout Head 设计实验报告

## 1. 背景问题

这一阶段主要想验证一个问题：

> 在 joint all-atoms pretrain 阶段，输出头的共享方式是否会影响后续单原子 fine-tune 的效果。

之前的训练流程大致是：

1. 先训练一个 joint all-atoms 模型，让一个模型同时预测多种原子的 chemical shift。
2. 做单原子 fine-tune 时，从 joint checkpoint 加载 PaiNN 主干。
3. 原来的 joint 输出头不完全适合单原子任务，因此 fine-tune 时相当于重新适配单原子输出头。

这里的问题是：如果 pretrain 阶段的输出头共享方式本身不合理，那么模型在联合训练时可能已经产生了任务干扰。尤其是不同原子的 chemical shift 分布差异很大，例如 `H/HA` 和 `C/N` 的数值范围、物理影响因素、预测难度都不一样。如果所有原子在输出端共享过强，困难原子可能会被其他原子影响。

因此，这一阶段的新想法是：

> 不等到 fine-tune 阶段才重新训练单原子头，而是在 pretrain 阶段就改变 readout head 的共享方式。

## 2. Plan A 是什么

`Plan A` 是这轮新 readout head 实验之前的主要对照基线。

它的流程是：

1. 使用已经训练好的 joint all-atoms 模型作为初始化来源。
2. 从不同 family 的 joint checkpoint 出发，分别 fine-tune 到单个目标原子。
3. 对每个原子导出 test 预测结果。
4. 对多个 fine-tuned 模型做 simple average ensemble。
5. 每个原子选择验证/测试表现最好的组合。

也就是说，`Plan A` 代表的是：

> 使用原有 joint pretrain 输出头设计，然后做 joint-to-individual fine-tune 和 ensemble 的最佳结果。

所以这次 readout head 实验的效果，是和 `Plan A` 这个强基线相比，而不是和单个随机模型相比。

## 3. 新实验做了什么

这次实验没有主要改 PaiNN 主干，而是改最后的 readout head。

PaiNN 主干负责从 3D 原子图中学习结构表征；readout head 负责把这个结构表征映射成具体原子的 chemical shift。新实验关注的是：不同原子之间，输出头到底应该共享到什么程度。

具体实现了两类新 head：

### 3.1 Atom Head

`atomhead` 的思想是：

> 每个原子从 pretrain 阶段开始就拥有独立的输出头。

这样可以减少不同原子之间在最后预测层的互相干扰。它更偏向 task-specific，适合预测规律差异较大的原子。

### 3.2 Family Head

`familyhead` 的思想是：

> 相似原子按原子族共享输出头，例如氢族、碳族、氮族。

它不是完全共享，也不是完全分开，而是在相似原子之间保留共享。这样既能利用同族原子的共性，又能避免所有原子混在一个输出空间里。

## 4. 和 Plan A 相比多了什么

相对于 `Plan A`，这次实验多了一个关键变量：`pretrain 阶段的 readout head 共享方式`。

对比关系可以理解为：

| 项目           | Plan A                             | 新 readout head 实验            |
| ------------ | ---------------------------------- | ---------------------------- |
| PaiNN 主干     | 使用原有结构                             | 基本保持一致                       |
| 结构特征         | 使用已有 M / Ve / Vnew 配置              | 继承已有配置                       |
| pretrain 输出头 | 原有 hybrid/head 设计                  | 新增 `atomhead` 和 `familyhead` |
| fine-tune    | joint-to-individual                | joint-to-individual          |
| ensemble     | 每个原子选最佳组合                          | 每个原子重新纳入新 head 模型做组合         |
| 主要变量         | checkpoint family 和 seed diversity | 输出头共享粒度                      |


因此，这次实验真正新增的不是一个普通模型，而是在测试：

> 多原子联合训练时，输出层应该全共享、按原子分开，还是按原子族共享。

## 5. 实验结果

下面的结果对比的是每个原子在 `Plan A` 中的最佳结果，以及加入 `atomhead/familyhead` 后的最佳结果。这里的结果都已经包含 fine-tune 后的 ensemble 搜索；如果最优组合只有一个模型，说明 ensemble 后没有超过该单模型。

| Atom | Plan A 最优组合 | Plan A 最优 MAE | Readout Head 最优组合 | Readout Head 最优 MAE | 变化 |
|---|---|---:|---|---:|---|
| H | `M0 + M4 + Ve456 + Vnew456` | 0.2888 | `M0 + Vnew456` | 0.2882 | 基本不变 |
| HA | `M0 + M4 + Ve456 + Vnew123` | 0.1636 | `M0 + Vnew456` | 0.1642 | 基本不变，略差 |
| C | `Vnew123` | 2.9874 | `M0_familyhead + Vnew456_atomhead + Vnew456_familyhead` | 1.1604 | 大幅提升 |
| CA | `Vnew456` | 1.0836 | `Vnew456 + M0_atomhead + M0_familyhead + Vnew456_atomhead` | 1.0301 | 有提升 |
| CB | `M0 + M4 + Ve456 + Vnew123 + Vnew456` | 1.1289 | `M0 + Vnew456 + M0_atomhead` | 1.1227 | 小幅提升 |
| N | `M4` | 2.6030 | `M0_atomhead + M0_familyhead + Vnew456_atomhead + Vnew456_familyhead` | 1.8189 | 大幅提升 |

其中比较明显的结果是：

- `C` 从 `2.9874` 降到 `1.1604`，说明 readout head 设计对碳原子预测影响很大。
- `N` 从 `2.6030` 降到 `1.8189`，说明氮原子也明显受益于更合理的输出头共享方式。
- `CA` 从 `1.0836` 降到 `1.0301`，有稳定提升。
- `CB` 从 `1.1289` 降到 `1.1227`，提升较小。
- `H/HA` 基本没有收益，说明氢相关原子原来的输出头设计已经比较够用。

## 6. 组合变化解读

可以看到，新 head 模型主要进入了 `C/CA/CB/N` 的最优组合，而 `H/HA` 的最优组合仍然主要依赖原有模型。

`Plan A` 中，`C/CA/N` 的最优结果其实都是单模型，说明当时多个 fine-tuned 模型之间没有形成有效互补；加入 `atomhead/familyhead` 后，`C/CA/N` 的最优结果都变成了多模型组合，说明新的输出头设计不仅提升了单模型质量，也提供了更有效的 ensemble diversity。

## 7. 阶段性结论

这一阶段可以得到三个结论：

1. 在 pretrain 阶段改变 readout head 共享方式是有效的，但不是对所有原子都有效。
2. 对 `C` 和 `N` 这种困难原子，`atomhead/familyhead` 明显优于原来的 Plan A 结果。
3. 对 `H/HA`，新 head 没有带来收益，说明这类原子可能更依赖已有结构特征和 ensemble diversity，而不是输出头拆分。

这说明 joint all-atoms pretrain 的问题不只在主干表示，也在输出端的任务共享粒度。不同原子之间不应该简单采用同一种共享方式，困难原子更需要专门化或 family-aware 的输出头。

## 8. 一句话总结

> 这轮实验验证了一个新的方向：在 joint all-atoms pretrain 阶段就调整 readout head 的共享方式，可以显著改善 `C` 和 `N` 等困难原子的 fine-tune 表现；相比 Plan A，最大收益来自更合理地控制不同原子之间的输出层共享粒度。
