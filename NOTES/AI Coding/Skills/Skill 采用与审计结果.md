---
tags:
  - skills
  - 审计
status: analysis-only
reviewed: 2026-08-30
source: "[[如何写一个好的skill 让你的效率加倍!]]"
---

# Skill 采用与审计结果

> [!warning] 本页只是只读分析草稿，不代表已经安装、修改或运行任何 Skill。是否进一步采用，必须先由用户确认。

## 结论

这篇帖子的有效部分已经用来审计现有 Skill；**不安装它推荐的 meta-skill，也不增加 Skill 管理器或 Hook**。

当前技能管理没有失控：`/Users/snooze26h/.cc-switch/skills` 是用户技能真源，Codex 侧有 13 个有效软链，断链为 0；库内另有 6 个研究技能。Obsidian 已启用 17 个插件，继续批量安装只会扩大触发冲突、权限面和维护成本。

## 三项审计标准

1. description 能否准确说明“什么时候必须触发、什么时候不要触发”。
2. 主文件是否只放路由与硬约束，细节按条件加载。
3. 是否有可判定的完成标准，而不是“尽量做完”。

## 抽查结果

| Skill | 触发边界 | 渐进披露 | 验收门槛 | 决策 |
| --- | --- | --- | --- | --- |
| `coordinate-auto-research` | 好：只覆盖 Research Line、分支、Chat 绑定、WIP、Handoff、Settlement 等 Auto Research 协调问题。 | 好：authority、chat lineage、concurrency 只在跨角色、建/续对话或异步结算时读取。 | 好：每个阶段均有独立 completion criterion。 | 保留。当前真实缺口不在 Skill，而是本机未找到所属研究仓库的 `AUTO_RESEARCH_SOP.md`、状态注册表或运行投影，因此实时绑定只能标为 unknown。 |
| `gpt-pro-question-window` | 好：明确只用于需要外部 GPT Pro 推理的任务，并排除纯本地问题。 | 合格：Bridge 协议是强制真源，浏览器 adapter 和普通问题 prompt 按操作类型加载。 | 强：原始回复、Codex verdict、线程校验和 Project 绑定必须分别可验证。 | 保留，不简化。它的 14 步较重，但对应真实的浏览器、会话与证据风险。 |
| `neat-freak` | **过宽**：裸“整理一下”“梳理一下”就强制触发，容易把笔记整理误判成开发文档收尾。 | 合格：sync matrix、agent paths 仅在需要时读；主文件本身偏长。 | 强但越权风险高：要求真的改文档、删废弃项和同步记忆。 | 暂不改 Skill；本轮仅用其文件盘点和文档一致性检查。记忆没有得到用户明确授权，因此不修改。后续若修订，应把触发词收窄到“开发里程碑/文档同步”。 |

## 采用规则

- 新 Skill 必须对应一个当前工具确实完成不了的任务，并先写验收样例。
- 同一能力只保留一个默认入口；新方案只能替换旧方案，不能长期并行。
- 社区 Skill 先读 `SKILL.md`、脚本、网络请求与写权限，再决定是否试用。
- 批量技能包、第三方管理器、全局 Hook 和要求复制到 `~/.claude/skills` 的教程，默认不采用。
- 未来审计时记录：用途、唯一增量、重叠对象、最近一次真实使用、保留理由。

## 本轮实际变化

- 安装 Skill：0
- 修改 Skill：0
- 新增 Skill 管理器：0
- 删除断链：0（没有断链）

帖子已经产生了审计结果，不再只是收藏；原帖保留作方法来源。
