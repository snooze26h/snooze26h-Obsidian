---
tags:
  - 科研工作流
  - auto-research
  - 审计
status: analysis-only
reviewed: 2026-08-30
sources:
  - "[[【开源推广】作为一名在读博士生，我在日常是如何与 AI 协作的？——ai-collab-playbook（26.6.8版）]]"
  - "[[多agent上下文管理实操手册]]"
---

# 科研 AI 协作流程采用结果

> [!warning] 本页只是流程对照草稿，不代表已经安装帖子项目、启动实验或修改 Auto Research 系统。

## 结论

两篇帖子的核心做法已经映射到现有工具，**不安装 ai-collab-playbook，也不复制另一套多 Agent 项目结构**。当前真正缺的不是更多 Agent，而是可被本机核验的 Auto Research live contract 与状态投影。

## 已落地 / 真缺口 / 不采用

| 主张 | 当前对应物 | 状态 | 处理 |
| --- | --- | --- | --- |
| 按任务重量选择 AI 入口 | 本地任务由 Codex 处理；外部高强度评审按需走 GPT Pro Bridge；研究综述走 `deep-research`。 | 已落地 | 继续按“最窄能力”路由，不新增统一大入口。 |
| 文献“调研 → 筛选 → 精读 → 整合” | `deep-research` 调研，Agentero 归档与检索，`paper-reader` / `equation-annotation` 精读，证据优先报告负责整合。 | 已落地 | 每一步必须留下论文、标注或报告产物；不能只留下聊天。 |
| 人负责问题表述与验收 | Idea 先经 `idea-evaluator`；外部答案与本地 verdict 分开；Auto Research 的科学决策须由本地 owner 接受、修改或拒绝。 | 已落地 | 保留人为终审，不允许外部 Agent 自动改 canonical state。 |
| 定期精简 Skill / MCP | CC Switch 单一真源、13 个有效 Codex 软链、0 断链；库内 6 个科研技能；Obsidian 17 个启用插件。 | 本轮已执行 | 没发现需要新增的 Skill；大包与平行系统不采用。 |
| 唯一 Research Line 与分支身份 | `coordinate-auto-research` 明确要求 `research_line_id`、Seed/fork 边界和一条本地 Chat 对多条 Web lineage。 | 设计已落地，运行态未知 | 当前库和可检索的本机路径中未找到 live registry，不能假装已有绑定。 |
| 唯一状态真源与唯一下一证据 | Skill 要求读取所属仓库的 `AUTO_RESEARCH_SOP.md`、`CONTEXT.md`、`OPERATIONS.md` 和 Current Next Evidence。 | **真缺口 / unknown** | 本机未找到这些 live contract 文件。应在真正拥有 Auto Research 状态的仓库内补，不在笔记库另建平行真源。 |
| Research Contract 与实验前冻结 | 多 Agent 手册有明确约束；现有实验规划与一致性检查 Skill 能支持。 | 机制可用，当前具体项目未知 | 每个项目在实验前冻结假设、数据、指标、预算、kill rule 和预期产物。 |
| Evidence / Run / Decision 分离 | Bridge 要求原始回答与本地 verdict 分离，Auto Research 协调 Skill 要求 Run、Decision、Settlement 分阶段。 | 设计已落地，运行态未知 | 没有具体研究仓库时不宣称已经生成账本或 run 证据。 |
| Handoff 与人为终审 | 已有 `handoff` Skill；Auto Research 结算要求 local verification、mentor acceptance、unique writer。 | 已落地 | 外部回复只作 pressure，不直接成为结论。 |

## 当前工作流

```text
问题与验收标准（人）
  → deep-research 调研
  → Agentero 筛选、归档与可追溯检索
  → paper-reader / equation-annotation 精读
  → 本地 Codex 整合与核验
  → 必要时 GPT Pro 独立压力测试
  → local verdict
  → Handoff / Settlement
```

## 使用纪律

1. 没有明确 Research Line、kill rule 和下一证据时，不创建 Student 或外部对话。
2. 没有 live registry 证据时，一律写 `unknown`，不从目录名或旧聊天推断绑定。
3. 不因某个开源项目角色更多、Skill 更多就整体迁移；只提取能填补已证实缺口的单项设计。
4. 研究结论必须回指论文、实验 artifact 或可复现 run；社区帖子只提供假设和流程灵感。
5. 本轮不启动实验、不调用外部 GPT Pro、不修改研究代码。

## 后续触发条件

只有在一个具体 Auto Research 仓库进入下一轮研究时，才补齐该仓库的 live contract / registry 并验证当前绑定。现在另建一份会制造两个状态真源，所以先不动。
