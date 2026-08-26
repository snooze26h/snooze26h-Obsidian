# Sample Views 阶段总结

## 1. 研究目标
这一阶段的目标，是验证“样本异质性”是否能作为比 `seed diversity` 更 fundamental 的 diversity 来源。

与之前通过修改特征或融合方式制造差异不同，这一阶段固定了模型结构和训练配置，只改变每个 view 所看到的训练样本子集。核心问题是：

- 在固定 backbone 的前提下，单纯通过不同训练样本子集构造的 sample views，是否能带来足够的互补性？
- 这种 sample diversity，能否超过已有的 full-data seed ensemble？

## 2. 实验设置
这一阶段统一采用如下设置：

- backbone 固定为 `M family = RING_CURRENT late`
- model seed 固定为 `42`
- train/val/test 划分固定
- 采样方式为无放回采样
- view 之间独立采样，允许自然重叠
- ensemble 方式为 simple average

对照组使用已有的 full-data seed ensemble：

- `M0 = 0.2864`
- `M1 = 0.2910`
- `M2 = 0.2923`
- `M0 + M1 + M2 = 0.2735`

这个 `0.2735` 是本阶段 sample-view 实验最重要的 baseline。

##  ratio = 0.5 的实验结果
第一轮 sample-view 采用 `50%` 的训练样本比例，每个 view 独立采样，训练了三个 sample views：`SV1 / SV2 / SV3`。

单模型 test MAE 为：

- `SV1 = 0.3078`
- `SV2 = 0.3112`
- `SV3 = 0.3124`

从 pairwise 结果看，sample views 之间存在互补性：

- `SV2 + SV3 = 0.2988`
- `SV1 + SV2 = 0.2971`
- `SV1 + SV3 = 0.2980`


但关键在于和 full-data seed ensemble 对比时，最优结果仍然是：

- `M0 + M1 + M2 = 0.2735`

而将 sample views 加入该组合后，结果反而变差：

- `M0 + M1 + M2 + SV1 = 0.2741`
- `M0 + M1 + M2 + SV1 + SV2 + SV3 = 0.2770`

### 结论
`ratio = 0.5` 的实验说明：

- 样本异质性确实存在
- 但训练信息损失过大
- ensemble 增益不足以弥补单模型退化

因此，`50%` 采样比例对于当前任务来说过于激进，不适合作为 sample-view 设置。

## 4. ratio = 0.7 的实验结果
第二轮将采样比例提高到 `70%`，同样训练三个独立 sample views：`SV1 / SV2 / SV3`。

单模型 test MAE 为：

- `SV1 = 0.3015`
- `SV2 = 0.3029`
- `SV3 = 0.3034`


- `SV1 + SV3 = 0.2888`
- `SV2 + SV3 = 0.2898`
- `SV1 + SV2 = 0.2893`


- `M0 + M1 + M2 = 0.2735`

最好的情况只是追平：

- `M0 + M1 + M2 + SV1 = 0.2735`
- `M0 + M1 + M2 + SV1 + SV3 = 0.2735`

### 结论
`ratio = 0.7` 相比 `0.5` 明显更合理，但目前只能说明：

- sample-view ensemble 已经接近可用
- 能与 full-data 3-seed ensemble 持平
- 但还没有稳定超过 seed ensemble baseline

## 5. 0.5 与 0.7 的对比
将两个采样比例并列比较，可以看到非常清晰的趋势。

### 单模型层面
- `0.5`：`0.3078 ~ 0.3124`
- `0.7`：`0.3015 ~ 0.3034`

说明 `0.7` 单模型质量明显更高，`0.5` 的信息损失过大。

### pairwise 层面
- `0.5` 最好 pair：`0.2988`
- `0.7` 最好 pair：`0.2888`

说明 `0.7` 不仅单模型更强，view 之间的互补性也更有效。

### 与 seed ensemble 的对比
- full-data 3-seed ensemble：`0.2735`
- `0.5`：明显打不过
- `0.7`：能追平，但尚未超过

### 结论
`0.7` 明显优于 `0.5`，是当前 sample-view 路线中更合理的采样比例；但现阶段仍不足以证明 sample diversity 比 seed diversity 更强。

## 6. 阶段性结论
这一阶段关于 sample views，可以总结为以下几点：

1. 样本异质性确实能够产生有效互补。无论 `0.5` 还是 `0.7`，sample views 之间的 pairwise ensemble 都明显优于各自单模型。
2. 采样比例过低会带来过大的训练信息损失。`ratio = 0.5` 下，单模型能力下降过于明显，ensemble 无法补回损失。
3. `ratio = 0.7` 明显优于 `ratio = 0.5`。它在保留更多训练信息的同时，仍然保留了有效的样本异质性。
4. 当前 sample-view 还没有超过 full-data seed ensemble。最好的 `0.7` 结果只能与 `M0 + M1 + M2 = 0.2735` 持平，尚不能证明 sample diversity 优于 seed diversity。

## 7. 一句话总结
这一阶段的 sample-view 实验表明：

> 样本异质性确实能带来互补，但 `50%` 采样过于激进；`70%` 采样已经接近 full-data 3-seed ensemble，但暂时还没有稳定超越它。
