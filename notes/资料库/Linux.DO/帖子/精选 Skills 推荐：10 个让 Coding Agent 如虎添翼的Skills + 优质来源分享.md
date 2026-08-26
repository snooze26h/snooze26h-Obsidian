# 精选 Skills 推荐：10 个让 Coding Agent 如虎添翼的Skills + 优质来源分享 

[原帖链接](https://linux.do/t/topic/1802808/1)

**作者：1EchA**  
**时间：Mar 23, 2026 2:43 am**  

# 精选 Skills 推荐：10 个让 Coding Agent 如虎添翼的Skills + 优质来源分享
> 本篇是 **Vibecoding 系列教程** 的工具导向专题篇。
> - 前篇：[Vibecoding 进阶教程总集篇——从能用到可控](https://linux.do/t/topic/1776917)
> - 系列合集仓库：[Vibecoding 教程基础篇+进阶篇](https://github.com/1EchA/how-to-vibecoding)    有下载/保存教程需求的佬有可以点这个系列合集仓库，仓库内映射的链接也均为linuxdo的帖子链接，感谢佬友们一直以来的支持！


---

## 写在前面

[进阶教程（一）](https://linux.do/t/topic/1770399/55)中我们已经了解了 Skills 具体是什么，应该怎么写，怎么让Coding Agent自行调用，当我们发现 skills 可以降低我们重复工作所消耗的时间和精力时，就会开始找更多的 skills ，与此同时问题也来了，社区中的 skills 已经数以十万甚至百万记了 ， 全装下来即是对上下文的挑战也是对钱包的挑战固态也受不了，所以肯定不能全装

这篇直接给佬友们端上来 **10 个我筛选过的 Skills 和 Skills Framework**，从方法论框架到具体场景卡片都有。每个Skills/Skills framework 都有一段分析/说明 ，佬友们可以根据说明来选择自己喜欢的 （还是不推荐一口气都装下来，同一个任务被指向多个 skills coding agent 是不知道该按哪个来工作的）

**文末推荐了skills的组合方法，以及很多Skills整合网站来供佬友们自行选择。**
> 我选择的这些Skills/Skills framework 至少具备怎样的特点呢：
> 1. 社区热度高
> 2. 确实解决了真实痛点
> 3. 彼此不重叠
> 4. 大多数主流工具都能用（Claude Code / Codex / Cursor / Antigravity）


---

## 推荐 Skills/Skills framework 等相关一览

![image](https://cdn3.ldstatic.com/original/4X/2/3/1/2310cfd634280d33f5cf7a43cb95a98132876697.png)

---

## 1. Superpowers（ 90k+）

**仓库**：[obra/superpowers](https://github.com/obra/superpowers)

**作者**：Jesse Vincent

**类型**：Skills Framework，准确的说，这其实是一整套工程方法论系统

**兼容**：Claude Code（Marketplace 已收录）、Codex、Gemini CLI、OpenCode、Cursor

GitHub 上非常非常火的 Coding Agent 框架

但很多佬友以为它是一堆 SKILL.md 卡片，其实它是一套完整的系统：包含Skills + 子智能体 + Slash 命令 + 会话钩子 + 多平台适配器。

### 完整结构

```
superpowers/
├── skills/           ← 14 个核心 Skills（带 references/ 按需加载）
├── agents/           ← 子智能体（code-reviewer）
├── commands/         ← 3 个命令（/brainstorm /write-plan /execute-plan）
├── hooks/            ← 会话钩子（启动时自动注入）
├── .claude-plugin/   ← Claude Code 插件配置
├── .codex/           ← Codex 适配
├── .opencode/        ← OpenCode 适配
├── GEMINI.md         ← Gemini CLI 适配
└── docs/
```

### 14 个 Skills

![image](https://cdn3.ldstatic.com/original/4X/a/b/4/ab4d2581584669228e510f0850e37a32c0ada262.png)
### 为什么把它排第一

它解决了一个让大家头痛很久的问题：coding agent 拿到需求就直接开写，不想不规划不测试。Superpowers 把完整的工程流水线用skills固化了下来：

![image](https://cdn3.ldstatic.com/original/4X/d/e/7/de7d7519417ff36896a2b225774df104d0a5ac07.png)
每个环节都有门控——前一步没过是到不了下一步。装了之后 coding agent 的输出质量是能明显感觉到提升的。

### 安装

支持 Claude Code / Codex / Cursor / Gemini CLI / OpenCode，各平台方式不一样。

**直达仓库 README**：[github.com/obra/superpowers#installation](https://github.com/obra/superpowers#installation)

---

## 2. SuperClaude Framework（ 21.8k）

**仓库**：[SuperClaude-Org/SuperClaude_Framework](https://github.com/SuperClaude-Org/SuperClaude_Framework)

**类型**：Skills 命令框架

**兼容**：Claude Code

这是一个元编程配置框架而不是标准的 skill.md 的格式，但做的事情却和 skills 高度相似，SuperClaude Framework定义了 Coding Agent 的行为模式、工作流程、输出约束，所以我还是把这个项目放到了 skills 推荐专栏，它和 Superpowers 的区别是，Superpowers 管的是“怎么正确地开发”（TDD、代码审查），SuperClaude 管的是怎么高效地指挥 coding agent 干活。两个可以配合着用。

### 30 条命令一览

![image](https://cdn3.ldstatic.com/original/4X/9/7/4/974ca38552ea43c72a2537127043a0979cbfc244.png)
### 安装

**直达仓库 README**：[github.com/SuperClaude-Org/SuperClaude_Framework#-quick-installation](https://github.com/SuperClaude-Org/SuperClaude_Framework#-quick-installation)

---

## 3. MiniMax Skills（ 1.8k）

**仓库**：[MiniMax-AI/skills](https://github.com/MiniMax-AI/skills)

**类型**：Skills 集合包（10 个）

**兼容**：Claude Code / Cursor / Codex / OpenCode

昨天晚上看到佬友分享后马上就去了解了一下，这个 repo 是 MiniMax 官方出的 10 个技能，每个都带完整工作流、设计系统、质量门控，是可以开箱即用的。（最近打算用一下 minimax2.7 ，到时候跟佬友们在说说体验感受)

### 10 个技能

![image](https://cdn3.ldstatic.com/original/4X/4/a/6/4a69eb88ec1b750c1d9530279720b9a0bb244b56.png)
### 安装

支持 Claude Code / Cursor / Codex / OpenCode。

**直达仓库 README**：[github.com/MiniMax-AI/skills#installation](https://github.com/MiniMax-AI/skills#installation)

---

## 4. Anthropic Official Skills（官方出品）

**仓库**：[anthropics/skills](https://github.com/anthropics/skills)

**类型**：官方 Skills

**兼容**：Claude Code（原生支持）

Anthropic 自己出的 Skills，算是 Skills 的标杆了，这个 skill-creator 在站内也有很多佬友推荐， frontend-design 也很不错。

![image](https://cdn3.ldstatic.com/original/4X/6/e/d/6ed7445b7068720f0ba6c6cb74aedfeb990f9e0c.png)
如果佬友们打算自己写 Skills，先装 `skill-creator`。它会引导你按标准格式来，包括 SKILL.md 怎么写、元数据怎么填。

---

## 5. Vercel Agent Skills（ 23.6k）

**仓库**：[vercel-labs/agent-skills](https://github.com/vercel-labs/agent-skills)

**类型**：Skills 集合包（前端最佳实践）

**兼容**：Claude Code / Cursor / Codex / 通用

Vercel 工程团队出品的 5 个 Skills，每个都是 SKILL.md 标准格式。做 React / Next.js / Web 开发的佬友可能会感兴趣，它不教如何写代码，而是教 coding agent 按 Vercel 团队的工程标准来审查和优化代码。

![image](https://cdn3.ldstatic.com/original/4X/3/2/e/32ed96f4ff638d5a7b5e82008cb7607d8a272280.png)
特别推荐 `web-design-guidelines`——100 多条规则覆盖了你能想到的所有 UI 细节，coding agent 做完 UI 后让它跑一遍这个 Skill 的审查，能抓出不少自己根本审不到的问题。

### 安装

**直达仓库 README**：[github.com/vercel-labs/agent-skills#installation](https://github.com/vercel-labs/agent-skills#installation)

---

## 6. Planning with Files

**仓库**：[OthmanAdi/planning-with-files](https://github.com/OthmanAdi/planning-with-files)

**类型**：专项 Skill（上下文工程）

**兼容**：Claude Code / Codex / 通用

做过长任务的佬都知道，coding agent 做到一半就忘了最初的目标，上下文太长前面的指令就被稀释掉了。这个 Skill 用 Markdown 文件给 coding agent 做了一个外挂记忆库：

- `task_plan.md` → 任务规划
- `findings.md` → 调研发现
- `progress.md` → 进度追踪

coding agent 做重大决策会先重读计划，在推进过程中更新进度，十分适合跨多轮对话的复杂任务。

---

## 7. Context Engineering Skills

**仓库**：[muratcankoylan/Agent-Skills-for-Context-Engineering](https://github.com/muratcankoylan/Agent-Skills-for-Context-Engineering)

**类型**：Skills 集合包（上下文治理）

**兼容**：通用

现在社区慢慢意识到了：提示词工程只是冰山一角，左右 coding agent 输出质量的还有很大的一部分是上下文工程，给它看什么、按什么顺序看、什么时候该压缩、什么时候该忘掉是很重要的一环。

![image](https://cdn3.ldstatic.com/original/4X/9/2/7/927c13280559522285a3bcda9ffa2b664f51a9bd.png)

---

## 8. Composio Skills

**仓库**：[ComposioHQ/skills](https://github.com/ComposioHQ/skills)

**类型**：Skills + MCP（双层配合）

**兼容**：通用（`npx skills add composiohq/skills` 一键安装）

日常开发里 coding agent 经常要和外部服务打交道——读写 GitHub Issues、发 Slack 通知、操作 Google Sheets、查 Jira。每个服务都自己写 MCP Server 来对接对很麻烦，Composio 把这些都封装好了。

它比较特殊的一点是：**MCP + Skills 配合**。Composio 本身提供 MCP Server（`composio mcp`），coding agent 通过标准 MCP 协议调用外部服务；同时还有一层 SKILL.md + rules 文件，教 coding agent 怎么正确地使用这些 MCP 工具——session 怎么管、多用户怎么隔离、OAuth 怎么走、各框架怎么集成。光有 MCP 工具 coding agent 只知道能调，加上 Skills 它才知道怎么调才对。

Composio Skills 的模式也是我觉得很有参考价值的模式，用 MCP + Skills 配合来提高agent调用工具的质量和效率。

### 仓库结构

![image](https://cdn3.ldstatic.com/original/4X/9/7/6/97676865cce7a1ff228a7938670ccd57c2a86398.png)
### 四块能力

![image](https://cdn3.ldstatic.com/original/4X/6/d/1/6d1da4d026639e822cf719ce1808bc839b49b002.png)
和其他 Skills 不一样的地方是：Composio 的 rules 文件是按场景拆的小卡片（`tr-session-management.md`、`tr-framework-integration.md`），coding agent 按需读取，不会一次性全加载。

---

## 9. Antfu Skills

**仓库**：[antfu/skills](https://github.com/antfu/skills)

**类型**：个人 Skills

**兼容**：通用

[Anthony Fu](https://github.com/antfu) 的个人 Skills 仓库。他是 Vue / Vite / UnoCSS / Vitest 的核心维护者，代码风格和工程标准就是行业标杆。与其一口气装一堆别人写的 Skills，不如先看看顶级大佬是怎么组织和写 Skills 的，学到手之后自己写的才好用。

---

## 10. Awesome Agent Skills（ 12.4k）

**仓库**：[VoltAgent/awesome-agent-skills](https://github.com/VoltAgent/awesome-agent-skills)

**类型**：Skills 索引

**兼容**：Claude Code / Codex / Antigravity / Gemini CLI / Cursor 等

严格说这不是一个 Skill，而是一个 500+ Skills 的索引仓库，正因为它整合的 skills 实在是全面，所以我决定放进来。里面收录的都是 Anthropic、Google Labs、Vercel、Stripe、Cloudflare、Netlify、Trail of Bits、Sentry、Expo、Hugging Face 这些团队贡献的高质量 skills。

![image](https://cdn3.ldstatic.com/original/4X/e/0/6/e0664d3e728409cd9991b32b92230a59434c5c64.png)
直接把这个项目当成skills索引即可，千万不要一口气都装下来！

---

## 用哪个好呢？（小孩子才做选择，但是我们在skills上最好也是别全都要

![image](https://cdn3.ldstatic.com/original/4X/0/0/0/0003d2a9618288c3ea16122bc531fedcde4bb170.png)

---

## Skills 加载机制

很多佬友以为 Skills 是“coding agent 检测到任务后按需调的”——只对了一半，不同工具之间还是有差异的

### 两种模式

| 模式 | 工作方法 |
| --- | --- |
| 全量加载 | 启动时把 skills 目录里所有 SKILL.md 一股脑塞进 system prompt |
| 按需路由 | 只读 name + description，匹配到了才加载完整内容 |

全量加载的工具，Skills 装多了会吃掉大量 token，最后导致留给实际对话的空间骤减，而按需路由的可以多装一些。

### references/ 目录

好的 Skill 会把详细内容拆到 `references/` 子目录：

```
my-skill/
├── SKILL.md              ← 先读这个（比较短，用来做路由）
└── references/
    ├── detailed-guide.md     ← 需要时才读
    └── anti-patterns.md      ← QA 阶段才读
```

主 SKILL.md 越短越好，这个确实很重要。Superpowers 的 `systematic-debugging` 有 11 个引用文件，但主文件很短，只在任务匹配时才加载详细指南。

## 装之前看这里

### 不要贪多

```
参考分层：
层 1：一个 Framework（Superpowers 或 SuperClaude，二选一）
层 2：1-2 个领域 Skill
层 3：Awesome 系列当skills索引，而不要整个装
```

### 自己写一个

用了别人的 Skills 之后，建议佬友们也自己写一个。回顾一下你和 coding agent 协作最顺畅的那几次，把那套流程固化下来就行。用前文提到的 Anthropic 的 `skill-creator` 来引导编写一下。  

对自己编写 Skills、MCP 感兴趣的佬友可以翻一下我的往期教程。

---

## 附录一：Skills 整合平台

除了上面推荐的 10 个，找更多 Skills 可以去这些地方翻：

| 平台 | 特点 | 链接 |
| --- | --- | --- |
| Skills.sh | Vercel 出的，3.5 万+ 技能包，支持一键安装 | skills.sh |
| SkillsMP | 聚合 GitHub 12.1 万+ 开源技能，AI 语义搜索 | skillsmp.com |
| Agent Skills Me | 人工精选 | agentskills.me |
| agent-skills.md | 6000+ 高频技能 | agent-skills.md |
| Agent Skills Hub | 29,000+ Skills + MCP Servers | agentskillshub.top |
| agent-skills.cc | 从 63,000+ GitHub Skills 里精选 1,000+ | agent-skills.cc |
| Claude Marketplaces | Claude Code 插件市场 | claudemarketplaces.com |
| AI Templates | 1000+ Agents / Commands / Skills / MCP | aitmpl.com |
| SkillStore | 中文友好，有安全审查 | skillstore.io |
| Skills Directory | Reddit 社区推荐的口碑榜 | skillsdirectory.com |
| SkillHub（腾讯） | 2.2 万+ 技能，国内高速镜像 | skillhub.tencent.com |
| ClawHub | OpenClaw 官方技能仓库，像 npm 一样管理 | clawhub.ai |

---

## 附录二：Repo推荐

| 仓库 | 定位 |
| --- | --- |
| anthropics/skills | Anthropic 官方 Skills |
| obra/superpowers | Superpowers 框架 |
| SuperClaude-Org/SuperClaude_Framework | SuperClaude 命令框架 |
| MiniMax-AI/skills | MiniMax 10 个生产级 Skills |
| VoltAgent/awesome-agent-skills | 500+ Skills 索引 |
| ComposioHQ/skills | Composio MCP + Skills |
| vercel-labs/agent-skills | Vercel 官方 |
| antfu/skills | Antfu 的个人实践 |
| ZhanlinCui/Agent-Skills-Hunter | 大杂烩（400+ Skills） |

  

      [github.com](https://github.com/1EchA/how-to-vibecoding)
  

  
    
  ![](https://cdn3.ldstatic.com/optimized/4X/1/7/9/1793de6a38583055e47b52c2886b9f1f56a3df02_2_690x344.png)

  ### GitHub - 1EchA/how-to-vibecoding: Vibecoding 系列教程：从环境搭建到多智能体协作，涵盖 MCP、Skills、Agent...

    Vibecoding 系列教程：从环境搭建到多智能体协作，涵盖 MCP、Skills、Agent 分工治理
