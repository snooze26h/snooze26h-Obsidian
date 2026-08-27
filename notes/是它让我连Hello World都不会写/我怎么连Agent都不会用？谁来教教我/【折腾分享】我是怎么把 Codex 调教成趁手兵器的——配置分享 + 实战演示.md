# 【折腾分享】我是怎么把 Codex 调教成趁手兵器的——配置分享 + 实战演示 

[原帖链接](https://linux.do/t/topic/1413296/1)

**作者：Kilong123**  
**时间：Jan 6, 2026 5:04 am**  

## 前言

最近一直在折腾 OpenAI 的 Codex CLI（就是那个开源的命令行 AI 编程助手），用了一段时间下来，说实话——**体验甚至超越了我之前用的 Claude Code**。

不是说 Claude Code 不好，而是 Codex 有几个点让我特别舒服：

1. **成本可控**：按需付费，用多少算多少，对于日常开发来说很友好
2. **智力在线**：可以畅用 GPT-5.2，推理能力和代码质量都相当能打
3. **可定制性极强**：这是重点！通过自定义配置，可以把它调教成完全适合自己工作流的助手

今天就分享一下我的配置思路和实际使用体验，希望对正在折腾 Codex 的佬友们有帮助。

---

## 配置文件在哪？

Codex 的配置目录在 `~/.codex/`（Windows 就是 `C:\Users\你的用户名\.codex\`），目录结构大概长这样：

```
  .codex/
  ├── AGENTS.md           # 全局指令（核心！）
  ├── AGENTS.override.md  # 覆盖配置（优先级更高）
  ├── prompts/            # 自定义提示词模板
  │   ├── check-fix.md
  │   ├── refactor.md
  │   └── ...
  ├── skills/             # 自定义技能（类似 slash commands）
  │   ├── play/
  │   │   └── SKILL.md
  │   ├── mindmap/
  │   │   └── SKILL.md
  │   ├── plan/
  │   │   └── SKILL.md
  │   └── prompt/
  │       └── SKILL.md
  └── sessions/           # 会话历史
```

**重点来了**：Codex 的配置体系和 Claude Code 类似，但用的是 `.codex` 目录，不是 `.claude`，别搞混了！

---

## 全局指令配置（AGENTS.md）

这是最核心的配置文件，相当于给 AI 立规矩。我的配置主要包括这几块：

### 1. 语言和人格设定

```
## 最重要
- Always reply in Chinese.
- 除非用户明确要求英文，否则所有回复使用简体中文。
- 代码标识符、命令、日志、报错信息保持原始语言；其余解释用中文。
```

**为啥要设这个**：默认情况下 AI 经常用英文回复，每次都要提醒很烦，直接写死在全局配置里一劳永逸。

### 2. 核心原则

```
## 核心原则
- **维持质量与一致性** — 彻底执行自动检查
- **事实确认** — 自行确认信息来源，不将猜测作为事实陈述
- **优先现有文件** — 优先编辑现有文件而非创建新文件
- **任务性质确认** — 确认任务是否需要改动代码，如果是计划或技术文档不要动源代码
```

这几条原则帮我避免了很多坑，比如 AI 动不动就新建文件（明明改现有的就行），或者瞎猜技术细节。

### 3. 对话风格（有点意思）

```
## 对话式人格
### 身份设定
- 行业顶级技术大佬，拥有丰富技术经验和极致的代码质量要求
- 审视用户输入的潜在问题，指出问题并给出框架外的建议
- 如果用户说得太离谱，直接指出帮其清醒

### 性格特征
- 东北人的天生幽默感，豪放不羁，说话随性
- 看到问题就开启吐槽模式，适当嘲讽
- 勇于质疑，敢于反驳，不讨好任何人
```

哈哈没错，我把它设定成了一个"东北技术大佬"的人设。这样交互起来不会太干巴巴的，有时候被它吐槽两句反而挺有意思。

### 4. 技术规范（Windows 用户必看）

因为我在 Windows 上开发，踩过不少坑，所以配置里写了很多 Windows 特有的规范：

```
## ⚠️ 执行环境：Windows（强制规范）

### 工具映射表
| 操作 | 使用工具 | 禁止 |
|------|---------|------|
| 读文件 | Read | cat/head/tail |
| 搜文件 | Glob | find/ls |
| 搜内容 | Grep | grep/rg |
| 编辑 | Edit | sed/awk |
| 创建 | Write | echo > |
| 系统命令 | Bash | PowerShell |
```

**踩坑经验**：Codex 默认会用 Linux 命令，在 Windows 上经常翻车。配置好工具映射后，它就会乖乖用对应的工具了。

---

## Prompts 模块：自定义提示词模板

Prompts 目录下可以放自定义的提示词模板，用法是 `/提示词名称`。我来介绍一个我常用的：

### /check-fix：修复影响检查

这个 prompt 是用来在改完 bug 之后做影响分析的，防止改了一处炸了三处：

```
# 修复影响检查 (Fix Impact Analysis)

对当前修改进行全面影响分析，检查是否对其他逻辑造成破坏。

## 检查维度

### 1️⃣ 直接影响分析
- 修改的函数/方法被哪些地方调用？
- 修改的参数签名是否向后兼容？
- 返回值类型/结构是否发生变化？

### 2️⃣ 间接影响分析
- 调用链上下游的数据流向
- 共享状态/全局变量的修改
- 事件监听器/回调函数的触发时机

### 3️⃣ 数据结构兼容性
- 新增字段: 旧数据读取时是否有默认值？
- 删除字段: 是否有代码仍在访问该字段？
- 类型变更: string→number 等隐式转换
```

**实际效果**：改完代码后跑一下 `/check-fix`，AI 会帮你分析影响范围，生成一份详细的报告：

```
📝 变更摘要
├─ 修改文件: [src/auth/token.ts]
├─ 修改类型: [修复]
├─ 影响范围: [模块级]
└─ 风险等级: 🟡中

🎯 直接影响
├─ 调用方 (3 处):
│  ├─ api.ts:123 - loginHandler() ⚠️ 需要检查
│  └─ middleware.ts:45 - authGuard() ✅ 兼容
```

这玩意儿救了我好几次，强烈推荐。

---

## Skills 模块：自定义技能

Skills 比 Prompts 更强大，可以定义复杂的工作流。

**为什么用 Skill 而不是 Prompt？** 这里有个关键区别：Prompt 会把整个指令文档加载到对话上下文中，**非常占用 token**；而 Skill 是按需触发的，只在调用时才加载，对上下文更友好。所以复杂的工作流建议用 Skill 来实现。

我来介绍几个我常用的：

### $play：Playwright 自动化

这个 skill 用来自动打开浏览器访问链接，还能处理登录：

```
# 打开链接（Playwright 自动化）

**核心任务**：使用 Playwright MCP 工具自动化浏览器操作。

## 执行步骤
1. 打开链接（browser_navigate）
2. 获取页面状态（browser_snapshot）
3. 处理登录逻辑（如果需要）
4. 遇到问题立即停止并告知
```

**使用场景**：

```
$play http://localhost:3000/admin
```

AI 会自动打开浏览器，如果跳转到登录页，会问你要不要用自动填充的账号登录。特别适合测试本地开发环境。

### $mindmap：生成思维导图

这个太实用了！一句话生成思维导图：

```
# 生成思维导图

使用 markmap 工具把描述转换成思维导图，生成后自动打开浏览器预览。
```

**使用示例**：

```
$mindmap React 状态管理方案对比
```

AI 会生成结构化的 Markdown，然后调用 markmap 工具转成可交互的思维导图，还自动在浏览器里打开。写技术文档、做技术调研的时候特别爽。

### $plan：任务规划模式

这个是我用得最多的，用来规划复杂任务：

```
# 任务规划模式

分析任务，生成本地计划文档并同步到 MCP。

**铁律**：
- 计划阶段只读不写——禁止修改项目代码
- 计划必须落地——生成 `.codex/plans/` 目录下的 Markdown 文档
- 执行需要批准——用户确认后才能使用 `$do-plan` 执行
```

**工作流程**：

1. `$plan 实现用户认证功能` → AI 分析需求、探索代码、生成计划文档
2. 审阅计划，提出修改意见
3. `$do-plan` → 按计划逐步执行

**亮点**：

- 计划阶段**绝对不会改代码**，只做分析和规划
- 支持智能更新：`$plan -u 缓存应该用 Redis 而不是内存`
- 自动同步到 MCP 任务管理

### $prompt：提示词优化专家

这个 skill 帮你优化 LLM 提示词：

```
# 提示词优化专家

## 工作流程
1️⃣ 接收提示词 → 识别类型、理解诉求
2️⃣ 问题诊断 → 系统化分析结构、语义、效果
3️⃣ 优化方案 → 提供 3 个层次的改进建议
4️⃣ 效果验证 → 生成对比测试用例
```

把你写的提示词丢给它，会给你一份详细的诊断报告和优化建议。对于经常要写 prompt 的人来说太有用了。

---

## 效率提升有多大？

说点实在的数据：

| 场景 | 没配置前 | 配置后 |
| --- | --- | --- |
| 改 bug 后检查影响 | 手动 grep + 人肉分析，30min | /check-fix，5min |
| 规划复杂任务 | 边想边改，经常返工 | $plan + $do-plan，有条不紊 |
| 测试本地环境 | 手动开浏览器、登录 | $play 一键搞定 |
| 整理技术调研 | 手画思维导图 | $mindmap 秒生成 |

**最大的变化**：从"我指挥 AI 干活"变成了"AI 按我定的规矩干活"。配置好之后，很多重复性的脑力劳动都被自动化了。

---

## 最后

配置这东西，适合自己的才是最好的。我分享的只是我的用法，大家可以根据自己的工作流来调整。

Codex 的可定制性是它最大的优势，花点时间折腾配置，后面能省很多事。

有什么问题欢迎在评论区交流！如果佬友们感兴趣，后续可以分享更多具体的 skill 实现细节。

---

## 附件：配置源文件

文中提到的配置文件我打包放附件了，可以直接下载参考：

- `AGENTS.md` — 全局指令配置
- `AGENTS.override.md` — 覆盖配置（对话风格等）
- `prompts/check-fix.md` — 修复影响检查 prompt
- `skills/play/SKILL.md` — Playwright 自动化 skill
- `skills/mindmap/SKILL.md` — 思维导图生成 skill
- `skills/plan/SKILL.md` — 任务规划 skill
- `skills/prompt/SKILL.md` — 提示词优化 skill

**注意**：这些配置是根据我的工作习惯定制的，建议按自己的需求修改后再用。  

[AGENTS.md.txt](https://linux.do/uploads/short-url/y1DmEZHSdNfXAFddxkej0y0542w.txt) (18.9 KB)  

[AGENTS.override.md.txt](https://linux.do/uploads/short-url/pwmwO9Wfbwp3ZGrRRP92vpqsITb.txt) (4.3 KB)  

[check-fix.md.txt](https://linux.do/uploads/short-url/pItOEMXy43W6iRUGedJMKjdPtIs.txt) (8.0 KB)  

[mindmap-SKILL.md.txt](https://linux.do/uploads/short-url/7xuy766DP9NKbSIABjIR7dg2lQJ.txt) (1.5 KB)  

[plan-SKILL.md.txt](https://linux.do/uploads/short-url/dgrPNGfDcr9TPWrY6JIgDsKb38G.txt) (20.5 KB)  

[play-SKILL.md.txt](https://linux.do/uploads/short-url/kufw4X1sBgPtNu7xCKagNpTPsrL.txt) (5.7 KB)  

[prompt-SKILL.md.txt](https://linux.do/uploads/short-url/cGVtxCWpTbshsQtw0h9tCGCfgqu.txt) (19.2 KB)

补充：审核疏忽，确实错漏了一些细节，plan和mindmap技能都需要调用相应的mcp：mcp-shrimp-task-manager和@jinzcdev/markmap-mcp-server。  

我习惯使用mcp-router来统一管理他们，playwright例外，比如这样：

![image](https://cdn3.ldstatic.com/original/4X/f/8/c/f8cb8b49b56af26bfaf80deefb6fb5518e133512.png)  

[do-plan-SKILL.md.txt](https://linux.do/uploads/short-url/nc0nAkc3rsDE02X7qdeDyzBm8AH.txt) (38.8 KB)
有些佬友建议出demo repo，这个想来实在不便展示，每个人的使用感受是不一样的，有兴趣佬友的可以细细体会。
