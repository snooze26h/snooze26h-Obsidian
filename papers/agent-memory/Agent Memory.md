---
type: topic
status: current
updated: 2026-08-22
---
# Agent Memory

## 本地论文

> [!note]
> 当前共 16 个论文目录，其中 15 篇已有以 `-CN` 结尾的中文翻译；《Memory in the LLM Era》目前只有原文。

### 综述与统一框架

- [[papers/agent-memory/A Survey of Agent Memory in the Second Half - Towards Self-Evolving and Long-Horizon Agents/NOTES|A Survey of Agent Memory in the Second Half]]
- [[papers/agent-memory/A Survey on the Memory Mechanism of Large Language Model based Agents/NOTES|A Survey on the Memory Mechanism of Large Language Model based Agents]]
- [[papers/agent-memory/Memory for Large Language Models/NOTES|Memory for Large Language Models]]
- [[papers/agent-memory/Memory in the LLM Era - Modular Architectures and Strategies in a Unified Framework/NOTES|Memory in the LLM Era]]

### 图记忆、检索与持续学习

- [[papers/agent-memory/AriGraph - Learning Knowledge Graph World Models with Episodic Memory for LLM Agents/NOTES|AriGraph]]
- [[papers/agent-memory/From RAG to Memory - Non-Parametric Continual Learning for Large Language Models/NOTES|From RAG to Memory]]
- [[papers/agent-memory/GFM-RAG - Graph Foundation Model for Retrieval Augmented Generation/GFM-RAG - Graph Foundation Model for Retrieval Augmented Generation-CN|GFM-RAG]]
- [[papers/agent-memory/Graph-based Agent Memory - Taxonomy, Techniques, and Applications/NOTES|Graph-based Agent Memory]]
- [[papers/agent-memory/HippoRAG - Neurobiologically Inspired Long-Term Memory for Large Language Models/NOTES|HippoRAG]]
- [[papers/agent-memory/Memory is Reconstructed, Not Retrieved - Graph Memory for LLM Agents/NOTES|Memory is Reconstructed, Not Retrieved]]
- [[papers/agent-memory/Zep - A Temporal Knowledge Graph Architecture for Agent Memory/NOTES|Zep]]

### 记忆系统与评测

- [[papers/agent-memory/CAM - A Constructivist View of Agentic Memory for LLM-Based Reading Comprehension/NOTES|CAM]]
- [[papers/agent-memory/Evaluating Memory in LLM Agents via Incremental Multi-Turn Interactions/NOTES|Evaluating Memory in LLM Agents via Incremental Multi-Turn Interactions]]
- [[papers/agent-memory/MemEvolve - Meta-Evolution of Agent Memory Systems/NOTES|MemEvolve]]
- [[papers/agent-memory/PlugMem - A Task-Agnostic Plugin Memory Module for LLM Agents/NOTES|PlugMem]]
- [[papers/agent-memory/REMem - Reasoning with Episodic Memory in Language Agents/NOTES|REMem]]

## ICML 2026 研究版图

### A. Graph / Structured Memory

| #     | 论文                                                                                                                                    | 主要研究问题                                                                                                                                                                                                                                                                             | ICML 2026 OpenReview                                                                     | arXiv                                                                |
| ----- | ------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------- | -------------------------------------------------------------------- |
| **1** | **MRAgent** — _Memory is Reconstructed, Not Retrieved: Graph Memory for LLM Agents_                                                   | 一次性检索结束后，推理过程中发现缺证据怎么办？把静态检索改成沿图不断探索和重构证据。 ([OpenReview](https://openreview.net/forum?id=xRVWftS3ES&utm_source=chatgpt.com "Memory is Reconstructed, Not Retrieved: Graph ..."))                                                                                                   | [https://openreview.net/forum?id=xRVWftS3ES](https://openreview.net/forum?id=xRVWftS3ES) | [https://arxiv.org/abs/2606.06036](https://arxiv.org/abs/2606.06036) |
| **2** | **PlugMem** — _A Task-Agnostic Plugin Memory Module for LLM Agents_                                                                   | 图里到底应该把什么作为“记忆单元”？提出把命题性知识和指导性知识作为核心，而不是直接存实体或原始文本。 ([OpenReview](https://openreview.net/forum?id=NWKaQIKoGp&referrer=%5Bthe+profile+of+Jianfeng+Gao%5D%28%2Fprofile%3Fid%3D~Jianfeng_Gao1%29&utm_source=chatgpt.com "A Task-Agnostic Plugin Memory Module for LLM Agents"))       | [https://openreview.net/forum?id=NWKaQIKoGp](https://openreview.net/forum?id=NWKaQIKoGp) | [https://arxiv.org/abs/2603.03296](https://arxiv.org/abs/2603.03296) |
| **3** | **HGMem** — _Hypergraph-based Working Memory to Improve Multi-step RAG for Long-Context Complex Relational Modeling_                  | 普通图只能方便地表达二元关系，如何表示多个事实共同构成的高阶关系？使用超图作为工作记忆。 ([OpenReview](https://openreview.net/forum?id=dvrq3GX7qL&referrer=%5Bthe+profile+of+Wai+Lam%5D%28%2Fprofile%3Fid%3D~Wai_Lam1%29&utm_source=chatgpt.com "Hypergraph-based Working Memory to Improve Multi-step ..."))                  | [https://openreview.net/forum?id=dvrq3GX7qL](https://openreview.net/forum?id=dvrq3GX7qL) | [https://arxiv.org/abs/2512.23959](https://arxiv.org/abs/2512.23959) |
| **4** | **Memora** — _A Harmonic Memory Representation Balancing Abstraction and Specificity_                                                 | 长期记忆怎样既压缩抽象、又不丢失具体细节？同时组织抽象索引和具体记忆。 ([OpenReview](https://openreview.net/forum?id=zSrvkj0ers&utm_source=chatgpt.com "Memora: A Harmonic Memory Representation Balancing ..."))                                                                                                     | [https://openreview.net/forum?id=zSrvkj0ers](https://openreview.net/forum?id=zSrvkj0ers) | [https://arxiv.org/abs/2602.03315](https://arxiv.org/abs/2602.03315) |
| **5** | **Executable Agentic Memory** — _Executable Agentic Memory for GUI Agent_                                                             | 图形界面智能体为什么每次都要重新规划？把成功操作保存成可执行的结构化记忆，供之后直接检索和执行。 ([OpenReview](https://openreview.net/forum?id=8metOwZHjD&utm_source=chatgpt.com "Executable Agentic Memory for GUI Agent"))                                                                                                       | [https://openreview.net/forum?id=8metOwZHjD](https://openreview.net/forum?id=8metOwZHjD) | [https://arxiv.org/abs/2605.12294](https://arxiv.org/abs/2605.12294) |
| **6** | **GiG / Graph-Informed Action Generation** — _Embodied Task Planning via Graph-Informed Action Generation with Large Language Models_ | 具身智能体做长程规划时容易目标漂移、产生不符合环境约束的动作，如何利用图结构保存环境状态和历史经验帮助规划？ ([OpenReview](https://openreview.net/forum?id=qBuZDNyQCM&utm_source=chatgpt.com "Embodied Task Planning via Graph-Informed Action ..."))                                                                                    | [https://openreview.net/forum?id=qBuZDNyQCM](https://openreview.net/forum?id=qBuZDNyQCM) | [https://arxiv.org/abs/2601.21841](https://arxiv.org/abs/2601.21841) |
| **7** | **DualGraph / A Tale of Two Graphs** — _Separating Knowledge Exploration from Outline Structure for Open-Ended Deep Research_         | 深度研究中“已经知道什么”和“报告应该怎么组织”容易混在一起，如何分别建模？使用知识图和提纲图两张共同演化的图。 ([OpenReview](https://openreview.net/forum?id=SDJWCc0Cml&utm_source=chatgpt.com "A Tale of Two Graphs: Separating Knowledge Exploration ..."))                                                                            | [https://openreview.net/forum?id=SDJWCc0Cml](https://openreview.net/forum?id=SDJWCc0Cml) | [https://arxiv.org/abs/2602.13830](https://arxiv.org/abs/2602.13830) |
| **8** | **H-EPM** — _Experience-Evolving Multi-Turn Tool-Use Agent with Hybrid Episodic-Procedural Memory_                                    | 完整工具轨迹太具体，而单纯工具图又缺少上下文，如何兼顾可复用性和具体情境？结合情景记忆与程序性记忆。 ([OpenReview](https://openreview.net/forum?id=PJ0GpmFYrR&referrer=%5Bthe+profile+of+Jingjing+Fu%5D%28%2Fprofile%3Fid%3D~Jingjing_Fu1%29&utm_source=chatgpt.com "Experience-Evolving Multi-Turn Tool-Use Agent with Hybrid...")) | [https://openreview.net/forum?id=PJ0GpmFYrR](https://openreview.net/forum?id=PJ0GpmFYrR) | [https://arxiv.org/abs/2512.07287](https://arxiv.org/abs/2512.07287) |

---

### B. Memory Management / Evolution

|#|论文|主要研究问题|ICML 2026 OpenReview|arXiv|
|---|---|---|---|---|
|**9**|**GAM-RAG** — _Gain-Adaptive Memory for Evolving Retrieval in Retrieval-Augmented Generation_|静态检索索引不会从过去成功的检索中学习，如何让检索系统“越用越会找”？把成功检索经验持续写回记忆。 ([OpenReview](https://openreview.net/forum?id=9YC7mbloXl&noteId=ukxFkR8jtt&utm_source=chatgpt.com "GAM-RAG: Gain-Adaptive Memory for Evolving Retrieval in..."))| [https://openreview.net/forum?id=9YC7mbloXl](https://openreview.net/forum?id=9YC7mbloXl)|[https://arxiv.org/abs/2603.01783](https://arxiv.org/abs/2603.01783) |
|**10**|**RGMem** — _Renormalization Group-inspired Memory Evolution for Language Agents_|怎么从不断变化、甚至互相矛盾的短期对话中逐渐形成稳定的长期用户知识？做多尺度、分层的记忆演化。 ([OpenReview](https://openreview.net/forum?id=FtvNZxhW9W&utm_source=chatgpt.com "RGMem: Renormalization Group–inspired Memory Evolution for ..."))| [https://openreview.net/forum?id=FtvNZxhW9W](https://openreview.net/forum?id=FtvNZxhW9W)|[https://arxiv.org/abs/2510.16392](https://arxiv.org/abs/2510.16392) |
|**11**|**UMEM** — _Unified Memory Extraction and Management Framework for Generalizable Memory_|只训练记忆管理、却固定记忆抽取过程，会产生很多只针对单一样本的噪声；如何联合优化抽取和管理？ ([OpenReview](https://openreview.net/forum?id=BoiXvrwtdi&referrer=%5Bthe+profile+of+Longyue+Wang%5D%28%2Fprofile%3Fid%3D~Longyue_Wang3%29&utm_source=chatgpt.com "Unified Memory Extraction and Management Framework for..."))| [https://openreview.net/forum?id=BoiXvrwtdi](https://openreview.net/forum?id=BoiXvrwtdi)|[https://arxiv.org/abs/2602.10652](https://arxiv.org/abs/2602.10652) |
|**12**|**SimpleMem** — _Efficient Lifelong Memory for LLM Agents_|完整保存历史太冗余，查询时再过滤又太贵，能不能在写入阶段就压缩和合并记忆？ ([OpenReview](https://openreview.net/forum?id=oBgLvd5YC6&referrer=%5Bthe+profile+of+Mingyu+Ding%5D%28%2Fprofile%3Fid%3D~Mingyu_Ding1%29&utm_source=chatgpt.com "SimpleMem: Efficient Lifelong Memory for LLM Agents"))| [https://openreview.net/forum?id=oBgLvd5YC6](https://openreview.net/forum?id=oBgLvd5YC6)|[https://arxiv.org/abs/2601.02553](https://arxiv.org/abs/2601.02553) |
|**13**|**E-mem** — _Multi-Agent Based Episodic Context Reconstruction for LLM Agent Memory_|把连续经验过早压成向量或图，会不会破坏时间顺序和上下文？主张保留更完整的情景上下文，再按需重构。 ([OpenReview](https://openreview.net/forum?id=FAjA0snAYq&referrer=%5Bthe+profile+of+Chentao+Wu%5D%28%2Fprofile%3Fid%3D~Chentao_Wu1%29&utm_source=chatgpt.com "E-mem: Multi-Agent Based Episodic Context ..."))| [https://openreview.net/forum?id=FAjA0snAYq](https://openreview.net/forum?id=FAjA0snAYq)|[https://arxiv.org/abs/2601.21714](https://arxiv.org/abs/2601.21714) |
|**14**|**Darwinian Memory** — _A Training-Free Self-Regulating Memory System for GUI Agent Evolution_|如果什么记忆都保留，过时和失败经验会污染后续决策，怎样自动强化有用经验、淘汰无用经验？ ([OpenReview](https://openreview.net/forum?id=tNn9bikyXZ&noteId=R9rR8uBq1l&utm_source=chatgpt.com "A Training-Free Self-Regulating Memory System for GUI ..."))| [https://openreview.net/forum?id=tNn9bikyXZ](https://openreview.net/forum?id=tNn9bikyXZ)|[https://arxiv.org/abs/2601.22528](https://arxiv.org/abs/2601.22528) |
|**15**|**Mem-T** — _Densifying Rewards for Long-Horizon Memory Agents_|智能体要连续做很多次写入、更新和检索，但只有任务最后才有奖励，如何判断中间哪些记忆操作是正确的？提供更密集的过程级反馈。 ([OpenReview](https://openreview.net/forum?id=8ppVmLtA2V&referrer=%5Bthe+profile+of+Yan+Zhang%5D%28%2Fprofile%3Fid%3D~Yan_Zhang14%29&utm_source=chatgpt.com "Densifying Rewards for Long-Horizon Memory Agents"))| [https://openreview.net/forum?id=8ppVmLtA2V](https://openreview.net/forum?id=8ppVmLtA2V)|[https://arxiv.org/abs/2601.23014](https://arxiv.org/abs/2601.23014) |
|**16**|**MemoPilot** — _From Player to Master: Enhancing Test-Time Learning of LLM Agents via Reinforcement Learning over Memory_|记忆更新通常依赖人工提示和规则，如何直接根据长期任务效果学习“怎样更新记忆”？使用强化学习训练记忆更新策略。 ([OpenReview](https://openreview.net/forum?id=gNWNtstp3r&referrer=%5Bthe+profile+of+Jinhua+Du%5D%28%2Fprofile%3Fid%3D~Jinhua_Du2%29&utm_source=chatgpt.com "Enhancing Test-Time Learning of LLM Agents via ..."))| [https://openreview.net/forum?id=gNWNtstp3r](https://openreview.net/forum?id=gNWNtstp3r)|[https://arxiv.org/abs/2606.08656](https://arxiv.org/abs/2606.08656) |
|**17**|**AdaMEM** — _Test-Time Adaptive Memory for Language Agents_|只在任务开始时检索一次记忆，随着任务状态改变，最初建议会失效；如何在执行过程中持续调整记忆指导？ ([OpenReview](https://openreview.net/forum?id=RzlmkviaNy&utm_source=chatgpt.com "AdaMEM: Test-Time Adaptive Memory for Language Agents"))| [https://openreview.net/forum?id=RzlmkviaNy](https://openreview.net/forum?id=RzlmkviaNy)|[https://arxiv.org/abs/2606.05684](https://arxiv.org/abs/2606.05684) |
|**18**|**BudgetMem** — _Learning Query-Aware Budget-Tier Routing for Runtime Agent Memory_|简单问题和复杂问题为什么使用同样昂贵的记忆流程？根据问题动态选择记忆模块以及计算预算。 ([OpenReview](https://openreview.net/forum?id=5G9puwbCsu&referrer=%5Bthe+profile+of+Weizhi+Zhang%5D%28%2Fprofile%3Fid%3D~Weizhi_Zhang1%29&utm_source=chatgpt.com "Learning Query-Aware Budget-Tier Routing for Runtime ..."))| [https://openreview.net/forum?id=5G9puwbCsu](https://openreview.net/forum?id=5G9puwbCsu)|[https://arxiv.org/abs/2602.06025](https://arxiv.org/abs/2602.06025) |
|**19**|**MemEvolve** — _Meta-Evolution of Agent Memory Systems_|以前只是让记忆内容变化，但编码、存储、检索和管理架构仍由人固定；能不能连记忆系统架构本身一起演化？ ([OpenReview](https://openreview.net/forum?id=qpkG0eKx4v&referrer=%5Bthe+profile+of+He+Zhu%5D%28%2Fprofile%3Fid%3D~He_Zhu8%29&utm_source=chatgpt.com "MemEvolve: Meta-Evolution of Agent Memory Systems"))| [https://openreview.net/forum?id=qpkG0eKx4v](https://openreview.net/forum?id=qpkG0eKx4v)|[https://arxiv.org/abs/2512.18746](https://arxiv.org/abs/2512.18746) |
