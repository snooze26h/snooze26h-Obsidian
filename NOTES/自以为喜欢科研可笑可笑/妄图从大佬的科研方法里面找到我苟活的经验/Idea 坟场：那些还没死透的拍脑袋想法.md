# Idea 坟场：那些还没死透的拍脑袋想法 

[原帖链接](https://linux.do/t/topic/1663523/8)

**作者：elpicio**  
**时间：Feb 26, 2026 11:09 am**  

这是一个长期更新的学术 idea 记录帖，用来持续记录日常研究中随时产生的想法。

<details>
<summary> 写在前面</summary>

正如许多人所说：**idea is cheap**  

未经验证的想法本身价值有限，但我不认为它们毫无意义——很多研究方向最初都只是一个模糊的直觉。问题在于这类想法总是转瞬即逝，而一个人很难有精力把每个拍脑袋的想法都持续推进下去。

所以我决定把这些想法记下来并公开出来，目的很简单：给自己一个可以回看和整理的思维档案，同时给不同知识背景的佬友一个低成本的交流入口。一个想法在原领域里可能走不通，但换一个人看，也许立刻就能想到一篇相关工作、一个反例、或者一个更值得问的问题。我确信这里的大部分idea不会发展成项目，但是只要能触发一点新的思考就是值得的。
</details>

应该会包含一些模糊的想法、失败的方向、不成熟的推测。  

有可能也会对旧条目做补充、修正或标记放弃。  

欢迎任何形式的讨论——质疑、补充、丢一篇相关文献、或者单纯说一句“这个有人做过了”，都很有价值 ---

## 状态标记说明

| 标记 | 含义 |
| --- | --- |
|  | 新想法，尚未深入 |
|  | 正在深入思考或调研 |
|  | 已发展为正式项目/方向 |
|  | 验证后放弃或已有成熟工作 |

---

## 索引

| # | 日期 | 标题 | 状态 |
| --- | --- | --- | --- |
| 001 | 2026-02-27 | LLM记忆系统 |  |
| 002 | 2026-03-04 | 如何通过自然语言获得现实里的颜色 |  |
| 003 | 2026-03-04 | 在auto research中保留多样化的潜在研究轨迹 |  |

## 收藏楼层（#8）

**作者：gwwgwd**  
**时间：May 16, 2026 4:36 am**  
**原帖楼层：[查看 #8](https://linux.do/t/topic/1663523/8)**  

**#003 | 2026-05-16 | :counterclockwise_arrows_button:**

### 在auto research中保留多样化的潜在研究轨迹

**背景**：偶然看到ICML 2025的杰出论文，*Roll the Dice & Look Before You Leap: Going Beyond the Creative Limits of Next-Token Prediction* 。它研究了一个有趣的问题：为什么 LLM 在写谐音梗、出奥数题、研究idea这类开放式任务上经常输出雷同的东西？这篇文章给出的结论是传统的next-token prediction缺乏latent plan，所以会在推理早期走捷径从而降低后面的生成多样性，导致输出分布接近。
事实上这样的情形在复杂的开放性问题上还是很常见的，之前做过很多用LLM辅助idea生成，完善的尝试，经常会发现不同的LLM在给出一些研究方向上的约束之后往往都会收束到近似的idea上。以前有一些论文认为这是由于预训练的数据分布类似造成的，但是现在随着后训练的影响越来越大，事实可能正如ICML这篇文章所表面的那样而不仅仅是输入数据的问题。
其实这反倒是一件好事，因为我们在使用闭源LLM产品的时候可以干涉的基本只有输入prompt，如果仅仅是数据问题反而无可奈何了。

**想法**： 线性 auto-research 通常是：idea → method → experiment → result → paper，muti-agent系统在这里的每个部分内部进行讨论，最后收束出一个统一的结果输入到下一个部分。按照那篇文章的思路，应当要维护多个研究状态的采样来保持latent plan。
一个简单的思路是在每个部分结束的时候让agents互相看各自的输出来建立关联，但是考虑到交互导致共识的坍缩，如果每一层都让节点互相讨论，最后可能全部向最强/最常见节点靠拢，因此可以很自然的想到残差结构，在每次进入下一阶段时重新叠加上初始的输入。

[details="事实上这些想法在我检索相关文献的时候发现已经有人做过了"]
RMoA: Optimizing Mixture-of-Agents through Diversity Maximization and Residual Compensation
Attention-MoA: Enhancing Mixture-of-Agents via Inter-Agent Semantic Attention and Deep Residual Synthesis
AFlow: Automating Agentic Workflow Generation
AOrchestra: Automating Sub-Agent Creation for Agentic Orchestration
[/details]

形式化一些的表述就是一个Transformer式的auto-research结构：
维护一组研究状态：$H_t = \{h_t^{(1)}, h_t^{(2)}, ..., h_t^{(K)}\}$
每个 $h_t^{(k)}$ 都是一条研究分支，比如包含：
- 问题怎么定义
- 核心假设是什么
- 方法大概属于哪一类
- 怎么验证
 
每一层更新时，不是让这些分支各走各的，而是让它们互相检查之后再更新自己：
$h_{t+1}^{(k)} = h_t^{(k)} + g_t^{(k)} \cdot F(h_t^{(k)}, \mathrm{Agg}_{j \neq k}(h_t^{(j)}))$
- $F$ 可以理解为一次 LLM 推理/改写；
- $\mathrm{Agg}_{j \neq k}$ 是其他分支传来的信息；
- $g_t^{(k)}$ 控制这次改多少；
- 残差项 $h_t^{(k)}$ 用来保留原来这条分支的身份，避免大家最后都变成同一个方案。

所以有些类似Transformer的架构：
| Transformer 里 | auto-research 里 |
|---|---|
| token | research state |
| self-attention | 分支之间互相参考、批判、借鉴 |
| residual | 保留每条分支自己的方向 |
| FFN / MLP | LLM 对当前分支做推理和更新 |
| 多层 block | idea → hypothesis → method → experiment |

**难点 / 疑问**：
- 这样做似乎会相当消耗token,至于速度应该影响不大，因为都是并行的，只有融合的时候会增大input tokens
- 到底有没有用，我还没有做过实验，事实上怎么对auto research进行评估确实也是一个悬而未决的问题。
- 显然这是一种充满了自相似性的做法，到底要不要套娃，套几层也是一个问题。

`标签：LLM / 自动化研究 / 图拓扑`
