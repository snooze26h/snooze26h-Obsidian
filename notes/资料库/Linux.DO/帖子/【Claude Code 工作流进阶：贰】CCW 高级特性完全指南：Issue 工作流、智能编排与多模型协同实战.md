# 【Claude Code 工作流进阶：贰】CCW 高级特性完全指南：Issue 工作流、智能编排与多模型协同实战 

[原帖链接](https://linux.do/t/topic/1511052/1)

**作者：Prorise**  
**时间：Jan 24, 2026 6:16 pm**  

# 【AI 编程进阶】CCW 高级特性完全指南：Issue 工作流、智能编排与多模型协同实战

笔记上篇地址：[【Claude Code工作流教程】Claude Code Workflow 实战指南：让多智能体 AI 军团为你写代码](https://linux.do/t/topic/1511026)
> **项目地址**：[GitHub - catlog22/Claude-Code-Workflow: JSON-driven multi-agent development framework with intelligent CLI orchestration (Gemini/Qwen/Codex), context-first architecture, and automated workflow execution](https://github.com/catlog22/Claude-Code-Workflow)


## 站内开源项目引用自：[开源] CCW 6.3.X（Claude-Code-Workflow) Issue-loop工作流驾驭你的核动力牛马（Codex）！

## 写在前面

**这篇文章适合谁？**

如果你已经掌握了 CCW 的主干工作流（Level 1-4），想深入了解 Issue 工作流、Level 5 智能编排、多 CLI 协同原理，以及生产环境最佳实践，这篇文章就是为你准备的。

---

## 第一章. Issue 工作流：开发后的维护补充

### 1.1. 为什么需要两套体系

很多人问：“为什么不把 Issue 工作流合并到主干工作流？”

答案是：它们解决的是 **不同阶段** 的问题。

**主干工作流（开发阶段：功能从 0 到 1）**

- 时机：功能开发期
- 目标：实现新功能
- 特点：完整的规划 → 实现 → 测试
- 分支模型：在当前分支工作
- 并行策略：依赖分析 + Agent 并行

**Issue 工作流（维护阶段：功能从 1 到 1.1）**

- 时机：主干开发完成后
- 目标：修复问题、优化功能
- 特点：发现 → 积累 → 批量解决
- 分支模型：可选 Worktree 隔离
- 并行策略：Worktree 隔离（可选）

**生活化类比**：

- 主干工作流是 “盖房子”（建造）
- Issue 工作流是 “装修和维修”（维护）

---

### 1.2. 主干工作流 vs Issue 工作流

| 维度 | 主干工作流 | Issue 工作流 |
| --- | --- | --- |
| 定位 | 主要开发周期 | 开发后的维护补充 |
| 时机 | 功能开发阶段 | 主干开发完成后 |
| 范围 | 完整功能实现 | 针对性修复/增强 |
| 并行策略 | 依赖分析 → Agent 并行 | Worktree 隔离（可选） |
| 分支模型 | 在当前分支工作 | 可使用独立 worktree |

**为什么主干工作流不用 Worktree**：

主干工作流通过 **依赖分析**（分析任务之间的依赖关系，决定哪些可以并行执行）解决并行问题：

1. 规划阶段执行依赖分析
2. 自动识别任务依赖和关键路径
3. 划分 **并行组**（独立任务）和 **串行链**（依赖任务）
4. Agent 并行执行独立任务，无需文件系统隔离

**为什么 Issue 工作流需要 Worktree**：

Issue 工作流作为 **补充机制**，场景不同：

1. 主干开发完成，已合并到 `main`
2. 发现需要修复的问题
3. 需要在不影响当前开发的情况下修复
4. Worktree 隔离让主分支保持稳定

---

### 1.3. Issue 工作流的两阶段生命周期

Issue 工作流分为 **积累阶段** 和 **批量解决阶段**。

**完整流程图**：

  
  
      ​
    

    #mermaid-diagram-1{font-family:"trebuchet ms",verdana,arial,sans-serif;font-size:16px;fill:#ccc;}@keyframes edge-animation-frame{from{stroke-dashoffset:0;}}@keyframes dash{to{stroke-dashoffset:0;}}#mermaid-diagram-1 .edge-animation-slow{stroke-dasharray:9,5!important;stroke-dashoffset:900;animation:dash 50s linear infinite;stroke-linecap:round;}#mermaid-diagram-1 .edge-animation-fast{stroke-dasharray:9,5!important;stroke-dashoffset:900;animation:dash 20s linear infinite;stroke-linecap:round;}#mermaid-diagram-1 .error-icon{fill:#a44141;}#mermaid-diagram-1 .error-text{fill:#ddd;stroke:#ddd;}#mermaid-diagram-1 .edge-thickness-normal{stroke-width:1px;}#mermaid-diagram-1 .edge-thickness-thick{stroke-width:3.5px;}#mermaid-diagram-1 .edge-pattern-solid{stroke-dasharray:0;}#mermaid-diagram-1 .edge-thickness-invisible{stroke-width:0;fill:none;}#mermaid-diagram-1 .edge-pattern-dashed{stroke-dasharray:3;}#mermaid-diagram-1 .edge-pattern-dotted{stroke-dasharray:2;}#mermaid-diagram-1 .marker{fill:lightgrey;stroke:lightgrey;}#mermaid-diagram-1 .marker.cross{stroke:lightgrey;}#mermaid-diagram-1 svg{font-family:"trebuchet ms",verdana,arial,sans-serif;font-size:16px;}#mermaid-diagram-1 p{margin:0;}#mermaid-diagram-1 .label{font-family:"trebuchet ms",verdana,arial,sans-serif;color:#ccc;}#mermaid-diagram-1 .cluster-label text{fill:#F9FFFE;}#mermaid-diagram-1 .cluster-label span{color:#F9FFFE;}#mermaid-diagram-1 .cluster-label span p{background-color:transparent;}#mermaid-diagram-1 .label text,#mermaid-diagram-1 span{fill:#ccc;color:#ccc;}#mermaid-diagram-1 .node rect,#mermaid-diagram-1 .node circle,#mermaid-diagram-1 .node ellipse,#mermaid-diagram-1 .node polygon,#mermaid-diagram-1 .node path{fill:#1f2020;stroke:#ccc;stroke-width:1px;}#mermaid-diagram-1 .rough-node .label text,#mermaid-diagram-1 .node .label text,#mermaid-diagram-1 .image-shape .label,#mermaid-diagram-1 .icon-shape .label{text-anchor:middle;}#mermaid-diagram-1 .node .katex path{fill:#000;stroke:#000;stroke-width:1px;}#mermaid-diagram-1 .rough-node .label,#mermaid-diagram-1 .node .label,#mermaid-diagram-1 .image-shape .label,#mermaid-diagram-1 .icon-shape .label{text-align:center;}#mermaid-diagram-1 .node.clickable{cursor:pointer;}#mermaid-diagram-1 .root .anchor path{fill:lightgrey!important;stroke-width:0;stroke:lightgrey;}#mermaid-diagram-1 .arrowheadPath{fill:lightgrey;}#mermaid-diagram-1 .edgePath .path{stroke:lightgrey;stroke-width:1px;}#mermaid-diagram-1 .flowchart-link{stroke:lightgrey;fill:none;}#mermaid-diagram-1 .edgeLabel{background-color:hsl(0, 0%, 34.4117647059%);text-align:center;}#mermaid-diagram-1 .edgeLabel p{background-color:hsl(0, 0%, 34.4117647059%);}#mermaid-diagram-1 .edgeLabel rect{opacity:0.5;background-color:hsl(0, 0%, 34.4117647059%);fill:hsl(0, 0%, 34.4117647059%);}#mermaid-diagram-1 .labelBkg{background-color:rgba(87.75, 87.75, 87.75, 0.5);}#mermaid-diagram-1 .cluster rect{fill:hsl(180, 1.5873015873%, 28.3529411765%);stroke:rgba(255, 255, 255, 0.25);stroke-width:1px;}#mermaid-diagram-1 .cluster text{fill:#F9FFFE;}#mermaid-diagram-1 .cluster span{color:#F9FFFE;}#mermaid-diagram-1 div.mermaidTooltip{position:absolute;text-align:center;max-width:200px;padding:2px;font-family:"trebuchet ms",verdana,arial,sans-serif;font-size:12px;background:hsl(20, 1.5873015873%, 12.3529411765%);border:1px solid rgba(255, 255, 255, 0.25);border-radius:2px;pointer-events:none;z-index:100;}#mermaid-diagram-1 .flowchartTitleText{text-anchor:middle;font-size:18px;fill:#ccc;}#mermaid-diagram-1 rect.text{fill:none;stroke-width:0;}#mermaid-diagram-1 .icon-shape,#mermaid-diagram-1 .image-shape{background-color:hsl(0, 0%, 34.4117647059%);text-align:center;}#mermaid-diagram-1 .icon-shape p,#mermaid-diagram-1 .image-shape p{background-color:hsl(0, 0%, 34.4117647059%);padding:2px;}#mermaid-diagram-1 .icon-shape .label rect,#mermaid-diagram-1 .image-shape .label rect{opacity:0.5;background-color:hsl(0, 0%, 34.4117647059%);fill:hsl(0, 0%, 34.4117647059%);}#mermaid-diagram-1 .label-icon{display:inline-block;height:1em;overflow:visible;vertical-align:-0.125em;}#mermaid-diagram-1 .node .label-icon path{fill:currentColor;stroke:revert;stroke-width:revert;}#mermaid-diagram-1 .node .neo-node{stroke:#ccc;}#mermaid-diagram-1 [data-look="neo"].node rect,#mermaid-diagram-1 [data-look="neo"].cluster rect,#mermaid-diagram-1 [data-look="neo"].node polygon{stroke:url(#mermaid-diagram-1-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#mermaid-diagram-1 [data-look="neo"].node path{stroke:url(#mermaid-diagram-1-gradient);stroke-width:1px;}#mermaid-diagram-1 [data-look="neo"].node .outer-path{filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#mermaid-diagram-1 [data-look="neo"].node .neo-line path{stroke:#ccc;filter:none;}#mermaid-diagram-1 [data-look="neo"].node circle{stroke:url(#mermaid-diagram-1-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#mermaid-diagram-1 [data-look="neo"].node circle .state-start{fill:#000000;}#mermaid-diagram-1 [data-look="neo"].icon-shape .icon{fill:url(#mermaid-diagram-1-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#mermaid-diagram-1 [data-look="neo"].icon-shape .icon-neo path{stroke:url(#mermaid-diagram-1-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#mermaid-diagram-1 :root{--mermaid-font-family:"trebuchet ms",verdana,arial,sans-serif;}Phase 1: 积累阶段

触发源

任务完成后的 review

代码审查发现

测试失败

discover 自动发现

discover-by-prompt 基于提示

new 手动创建

持续积累 Issue 到待处理队列

积累足够后

Phase 2: 批量解决

plan --all-pending

queue 优化顺序

execute 并行执行

支持 Worktree 隔离

保持主分支稳定

**与主干工作流的协作模式**：

  
  
      ​
    

    #mermaid-diagram-2{font-family:"trebuchet ms",verdana,arial,sans-serif;font-size:16px;fill:#ccc;}@keyframes edge-animation-frame{from{stroke-dashoffset:0;}}@keyframes dash{to{stroke-dashoffset:0;}}#mermaid-diagram-2 .edge-animation-slow{stroke-dasharray:9,5!important;stroke-dashoffset:900;animation:dash 50s linear infinite;stroke-linecap:round;}#mermaid-diagram-2 .edge-animation-fast{stroke-dasharray:9,5!important;stroke-dashoffset:900;animation:dash 20s linear infinite;stroke-linecap:round;}#mermaid-diagram-2 .error-icon{fill:#a44141;}#mermaid-diagram-2 .error-text{fill:#ddd;stroke:#ddd;}#mermaid-diagram-2 .edge-thickness-normal{stroke-width:1px;}#mermaid-diagram-2 .edge-thickness-thick{stroke-width:3.5px;}#mermaid-diagram-2 .edge-pattern-solid{stroke-dasharray:0;}#mermaid-diagram-2 .edge-thickness-invisible{stroke-width:0;fill:none;}#mermaid-diagram-2 .edge-pattern-dashed{stroke-dasharray:3;}#mermaid-diagram-2 .edge-pattern-dotted{stroke-dasharray:2;}#mermaid-diagram-2 .marker{fill:lightgrey;stroke:lightgrey;}#mermaid-diagram-2 .marker.cross{stroke:lightgrey;}#mermaid-diagram-2 svg{font-family:"trebuchet ms",verdana,arial,sans-serif;font-size:16px;}#mermaid-diagram-2 p{margin:0;}#mermaid-diagram-2 .label{font-family:"trebuchet ms",verdana,arial,sans-serif;color:#ccc;}#mermaid-diagram-2 .cluster-label text{fill:#F9FFFE;}#mermaid-diagram-2 .cluster-label span{color:#F9FFFE;}#mermaid-diagram-2 .cluster-label span p{background-color:transparent;}#mermaid-diagram-2 .label text,#mermaid-diagram-2 span{fill:#ccc;color:#ccc;}#mermaid-diagram-2 .node rect,#mermaid-diagram-2 .node circle,#mermaid-diagram-2 .node ellipse,#mermaid-diagram-2 .node polygon,#mermaid-diagram-2 .node path{fill:#1f2020;stroke:#ccc;stroke-width:1px;}#mermaid-diagram-2 .rough-node .label text,#mermaid-diagram-2 .node .label text,#mermaid-diagram-2 .image-shape .label,#mermaid-diagram-2 .icon-shape .label{text-anchor:middle;}#mermaid-diagram-2 .node .katex path{fill:#000;stroke:#000;stroke-width:1px;}#mermaid-diagram-2 .rough-node .label,#mermaid-diagram-2 .node .label,#mermaid-diagram-2 .image-shape .label,#mermaid-diagram-2 .icon-shape .label{text-align:center;}#mermaid-diagram-2 .node.clickable{cursor:pointer;}#mermaid-diagram-2 .root .anchor path{fill:lightgrey!important;stroke-width:0;stroke:lightgrey;}#mermaid-diagram-2 .arrowheadPath{fill:lightgrey;}#mermaid-diagram-2 .edgePath .path{stroke:lightgrey;stroke-width:1px;}#mermaid-diagram-2 .flowchart-link{stroke:lightgrey;fill:none;}#mermaid-diagram-2 .edgeLabel{background-color:hsl(0, 0%, 34.4117647059%);text-align:center;}#mermaid-diagram-2 .edgeLabel p{background-color:hsl(0, 0%, 34.4117647059%);}#mermaid-diagram-2 .edgeLabel rect{opacity:0.5;background-color:hsl(0, 0%, 34.4117647059%);fill:hsl(0, 0%, 34.4117647059%);}#mermaid-diagram-2 .labelBkg{background-color:rgba(87.75, 87.75, 87.75, 0.5);}#mermaid-diagram-2 .cluster rect{fill:hsl(180, 1.5873015873%, 28.3529411765%);stroke:rgba(255, 255, 255, 0.25);stroke-width:1px;}#mermaid-diagram-2 .cluster text{fill:#F9FFFE;}#mermaid-diagram-2 .cluster span{color:#F9FFFE;}#mermaid-diagram-2 div.mermaidTooltip{position:absolute;text-align:center;max-width:200px;padding:2px;font-family:"trebuchet ms",verdana,arial,sans-serif;font-size:12px;background:hsl(20, 1.5873015873%, 12.3529411765%);border:1px solid rgba(255, 255, 255, 0.25);border-radius:2px;pointer-events:none;z-index:100;}#mermaid-diagram-2 .flowchartTitleText{text-anchor:middle;font-size:18px;fill:#ccc;}#mermaid-diagram-2 rect.text{fill:none;stroke-width:0;}#mermaid-diagram-2 .icon-shape,#mermaid-diagram-2 .image-shape{background-color:hsl(0, 0%, 34.4117647059%);text-align:center;}#mermaid-diagram-2 .icon-shape p,#mermaid-diagram-2 .image-shape p{background-color:hsl(0, 0%, 34.4117647059%);padding:2px;}#mermaid-diagram-2 .icon-shape .label rect,#mermaid-diagram-2 .image-shape .label rect{opacity:0.5;background-color:hsl(0, 0%, 34.4117647059%);fill:hsl(0, 0%, 34.4117647059%);}#mermaid-diagram-2 .label-icon{display:inline-block;height:1em;overflow:visible;vertical-align:-0.125em;}#mermaid-diagram-2 .node .label-icon path{fill:currentColor;stroke:revert;stroke-width:revert;}#mermaid-diagram-2 .node .neo-node{stroke:#ccc;}#mermaid-diagram-2 [data-look="neo"].node rect,#mermaid-diagram-2 [data-look="neo"].cluster rect,#mermaid-diagram-2 [data-look="neo"].node polygon{stroke:url(#mermaid-diagram-2-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#mermaid-diagram-2 [data-look="neo"].node path{stroke:url(#mermaid-diagram-2-gradient);stroke-width:1px;}#mermaid-diagram-2 [data-look="neo"].node .outer-path{filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#mermaid-diagram-2 [data-look="neo"].node .neo-line path{stroke:#ccc;filter:none;}#mermaid-diagram-2 [data-look="neo"].node circle{stroke:url(#mermaid-diagram-2-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#mermaid-diagram-2 [data-look="neo"].node circle .state-start{fill:#000000;}#mermaid-diagram-2 [data-look="neo"].icon-shape .icon{fill:url(#mermaid-diagram-2-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#mermaid-diagram-2 [data-look="neo"].icon-shape .icon-neo path{stroke:url(#mermaid-diagram-2-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#mermaid-diagram-2 :root{--mermaid-font-family:"trebuchet ms",verdana,arial,sans-serif;}是

否

Feature Request

Main Workflow Level 1-4

完成

Review

发现问题?

Issue Workflow

Main Branch

修复完成

合并回主干

继续新功能

---

## 第二章. Issue 工作流实战

### 2.1. 积累阶段

积累阶段有 **3 种方式** 发现和创建 Issue。

#### 场景 1：自动发现 Issue

**需求**：项目上线后，想自动扫描潜在问题。

**指令**：

```
/issue:discover
```

**产出**：

- 自动扫描代码和日志
- 识别潜在问题（代码异味、性能瓶颈、安全隐患）
- 生成 Issue 列表

**价值**：多视角自动发现，无需手动排查。

---

#### 场景 2：基于提示发现 Issue

**需求**：根据特定关键词发现问题。

**指令**：

```
/issue:discover-by-prompt "查找所有 TODO 和 FIXME 注释"
```

**产出**：基于提示词的 Issue 列表。

**价值**：针对性发现，聚焦特定问题。

---

#### 场景 3：手动创建 Issue

**需求**：用户反馈 “文件上传失败”。

**指令**：

```
/issue:new "用户上传文件失败，返回 413 错误"
```

**产出**：创建 Issue（如 `ISS-001`）。

**价值**：手动记录，纳入统一管理。

---

### 2.2. 批量解决阶段

积累足够的 Issue 后，进入批量解决阶段。

#### 场景 4：查看 Issue 队列

**需求**：查看所有待处理的 Issue。

**指令**：

```
/issue:queue
```

**产出**：待处理 Issue 列表（按优先级排序）。

**价值**：一目了然，优先级清晰。

---

#### 场景 5：批量规划 Issue

**需求**：为所有待处理 Issue 制定修复计划。

**指令**：

```
/issue:plan --all-pending
```

**产出**：

- 批量规划所有待处理 Issue
- 冲突分析
- 优化执行顺序

**价值**：统一规划，避免冲突。

---

#### 场景 6：执行 Issue 修复

**需求**：执行 Issue 修复。

**指令**：

```
/issue:execute
```

**产出**：并行执行所有 Issue 修复。

**价值**：批量处理，提高效率。

---

#### 场景 7：针对单个 Issue 修复

**需求**：只修复某个特定 Issue。

**指令**：

```
/issue:plan ISS-001
/issue:execute ISS-001
```

**产出**：针对性修复。

**价值**：灵活处理，按需修复。

---

### 2.3. Worktree 隔离机制

Issue 工作流支持 **Git Worktree**（Git 的一个功能，允许同时检出多个分支到不同目录）隔离。

**为什么需要 Worktree**：

- 主干开发完成，已合并到 `main`
- 需要在不影响当前开发的情况下修复
- Worktree 隔离让主分支保持稳定

**使用示例**：

```
# Issue 工作流会自动创建 worktree（如果配置启用）
/issue:execute ISS-001

# 系统会在独立目录中修复
# 修复完成后自动合并回主干
```

**价值**：

- 主分支保持稳定
- 多个 Issue 可以并行修复
- 修复完成后自动合并

---

## 第三章. Level 5 深度解析

### 3.1. 为什么需要 Level 5

**痛点场景**：

```
用户："我想实现一个用户注册功能，但我不知道该用 lite-plan 还是 plan，
      也不知道要不要加测试，要不要审查代码..."

传统方案：用户需要：
1. 阅读文档，理解每个命令的作用
2. 判断任务复杂度
3. 手动选择命令组合
4. 记住命令的执行顺序

Level 5 方案：
/ccw-coordinator "实现用户注册功能"
→ 系统自动分析 → 推荐命令链 → 用户确认 → 自动执行
```

**Level 5 解决的问题**：

- 不知道用什么命令
- 不知道命令的执行顺序
- 不知道任务的复杂度
- 需要端到端自动化

---

### 3.2. 三阶段工作流程

Level 5 智能编排分为 **3 个阶段**。

  
  
      ​
    

    #mermaid-diagram-3{font-family:"trebuchet ms",verdana,arial,sans-serif;font-size:16px;fill:#ccc;}@keyframes edge-animation-frame{from{stroke-dashoffset:0;}}@keyframes dash{to{stroke-dashoffset:0;}}#mermaid-diagram-3 .edge-animation-slow{stroke-dasharray:9,5!important;stroke-dashoffset:900;animation:dash 50s linear infinite;stroke-linecap:round;}#mermaid-diagram-3 .edge-animation-fast{stroke-dasharray:9,5!important;stroke-dashoffset:900;animation:dash 20s linear infinite;stroke-linecap:round;}#mermaid-diagram-3 .error-icon{fill:#a44141;}#mermaid-diagram-3 .error-text{fill:#ddd;stroke:#ddd;}#mermaid-diagram-3 .edge-thickness-normal{stroke-width:1px;}#mermaid-diagram-3 .edge-thickness-thick{stroke-width:3.5px;}#mermaid-diagram-3 .edge-pattern-solid{stroke-dasharray:0;}#mermaid-diagram-3 .edge-thickness-invisible{stroke-width:0;fill:none;}#mermaid-diagram-3 .edge-pattern-dashed{stroke-dasharray:3;}#mermaid-diagram-3 .edge-pattern-dotted{stroke-dasharray:2;}#mermaid-diagram-3 .marker{fill:lightgrey;stroke:lightgrey;}#mermaid-diagram-3 .marker.cross{stroke:lightgrey;}#mermaid-diagram-3 svg{font-family:"trebuchet ms",verdana,arial,sans-serif;font-size:16px;}#mermaid-diagram-3 p{margin:0;}#mermaid-diagram-3 .label{font-family:"trebuchet ms",verdana,arial,sans-serif;color:#ccc;}#mermaid-diagram-3 .cluster-label text{fill:#F9FFFE;}#mermaid-diagram-3 .cluster-label span{color:#F9FFFE;}#mermaid-diagram-3 .cluster-label span p{background-color:transparent;}#mermaid-diagram-3 .label text,#mermaid-diagram-3 span{fill:#ccc;color:#ccc;}#mermaid-diagram-3 .node rect,#mermaid-diagram-3 .node circle,#mermaid-diagram-3 .node ellipse,#mermaid-diagram-3 .node polygon,#mermaid-diagram-3 .node path{fill:#1f2020;stroke:#ccc;stroke-width:1px;}#mermaid-diagram-3 .rough-node .label text,#mermaid-diagram-3 .node .label text,#mermaid-diagram-3 .image-shape .label,#mermaid-diagram-3 .icon-shape .label{text-anchor:middle;}#mermaid-diagram-3 .node .katex path{fill:#000;stroke:#000;stroke-width:1px;}#mermaid-diagram-3 .rough-node .label,#mermaid-diagram-3 .node .label,#mermaid-diagram-3 .image-shape .label,#mermaid-diagram-3 .icon-shape .label{text-align:center;}#mermaid-diagram-3 .node.clickable{cursor:pointer;}#mermaid-diagram-3 .root .anchor path{fill:lightgrey!important;stroke-width:0;stroke:lightgrey;}#mermaid-diagram-3 .arrowheadPath{fill:lightgrey;}#mermaid-diagram-3 .edgePath .path{stroke:lightgrey;stroke-width:1px;}#mermaid-diagram-3 .flowchart-link{stroke:lightgrey;fill:none;}#mermaid-diagram-3 .edgeLabel{background-color:hsl(0, 0%, 34.4117647059%);text-align:center;}#mermaid-diagram-3 .edgeLabel p{background-color:hsl(0, 0%, 34.4117647059%);}#mermaid-diagram-3 .edgeLabel rect{opacity:0.5;background-color:hsl(0, 0%, 34.4117647059%);fill:hsl(0, 0%, 34.4117647059%);}#mermaid-diagram-3 .labelBkg{background-color:rgba(87.75, 87.75, 87.75, 0.5);}#mermaid-diagram-3 .cluster rect{fill:hsl(180, 1.5873015873%, 28.3529411765%);stroke:rgba(255, 255, 255, 0.25);stroke-width:1px;}#mermaid-diagram-3 .cluster text{fill:#F9FFFE;}#mermaid-diagram-3 .cluster span{color:#F9FFFE;}#mermaid-diagram-3 div.mermaidTooltip{position:absolute;text-align:center;max-width:200px;padding:2px;font-family:"trebuchet ms",verdana,arial,sans-serif;font-size:12px;background:hsl(20, 1.5873015873%, 12.3529411765%);border:1px solid rgba(255, 255, 255, 0.25);border-radius:2px;pointer-events:none;z-index:100;}#mermaid-diagram-3 .flowchartTitleText{text-anchor:middle;font-size:18px;fill:#ccc;}#mermaid-diagram-3 rect.text{fill:none;stroke-width:0;}#mermaid-diagram-3 .icon-shape,#mermaid-diagram-3 .image-shape{background-color:hsl(0, 0%, 34.4117647059%);text-align:center;}#mermaid-diagram-3 .icon-shape p,#mermaid-diagram-3 .image-shape p{background-color:hsl(0, 0%, 34.4117647059%);padding:2px;}#mermaid-diagram-3 .icon-shape .label rect,#mermaid-diagram-3 .image-shape .label rect{opacity:0.5;background-color:hsl(0, 0%, 34.4117647059%);fill:hsl(0, 0%, 34.4117647059%);}#mermaid-diagram-3 .label-icon{display:inline-block;height:1em;overflow:visible;vertical-align:-0.125em;}#mermaid-diagram-3 .node .label-icon path{fill:currentColor;stroke:revert;stroke-width:revert;}#mermaid-diagram-3 .node .neo-node{stroke:#ccc;}#mermaid-diagram-3 [data-look="neo"].node rect,#mermaid-diagram-3 [data-look="neo"].cluster rect,#mermaid-diagram-3 [data-look="neo"].node polygon{stroke:url(#mermaid-diagram-3-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#mermaid-diagram-3 [data-look="neo"].node path{stroke:url(#mermaid-diagram-3-gradient);stroke-width:1px;}#mermaid-diagram-3 [data-look="neo"].node .outer-path{filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#mermaid-diagram-3 [data-look="neo"].node .neo-line path{stroke:#ccc;filter:none;}#mermaid-diagram-3 [data-look="neo"].node circle{stroke:url(#mermaid-diagram-3-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#mermaid-diagram-3 [data-look="neo"].node circle .state-start{fill:#000000;}#mermaid-diagram-3 [data-look="neo"].icon-shape .icon{fill:url(#mermaid-diagram-3-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#mermaid-diagram-3 [data-look="neo"].icon-shape .icon-neo path{stroke:url(#mermaid-diagram-3-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#mermaid-diagram-3 :root{--mermaid-font-family:"trebuchet ms",verdana,arial,sans-serif;}Phase 1: 需求分析

Phase 2: 命令发现与推荐

Phase 3: 序列执行

任务类型检测

复杂度评估

端口匹配

生成命令链

用户确认

串行执行

状态持久化

Hook 回调

#### Phase 1：需求分析

**任务类型检测**：

| 任务类型 | 检测关键词 | 示例 |
| --- | --- | --- |
| bugfix | fix, bug, error, crash, fail | “修复登录超时问题” |
| tdd | tdd, test-driven, 先写测试 | “用 TDD 开发支付模块” |
| test-fix | 测试失败, test fail, fix test | “修复失败的集成测试” |
| test-gen | generate test, 写测试, add test | “为认证模块生成测试” |
| review | review, 审查, code review | “审查支付模块代码” |
| brainstorm | 不确定, explore, 研究, what if | “探索缓存方案” |
| multi-cli | 多视角, 比较方案, cross-verify | “比较 OAuth 方案” |
| feature | (默认) | “实现用户注册” |

**复杂度评估**：

| 权重 | 关键词 |
| --- | --- |
| +2 | refactor, 重构, migrate, 迁移, architect, 架构, system, 系统 |
| +2 | multiple, 多个, across, 跨, all, 所有, entire, 整个 |
| +1 | integrate, 集成, api, database, 数据库 |
| +1 | security, 安全, performance, 性能, scale, 扩展 |

- **高复杂度** (≥4)：自动选择复杂工作流
- **中复杂度** (2-3)：自动选择标准工作流
- **低复杂度** (< 2)：自动选择轻量工作流

---

#### Phase 2：命令发现与推荐

**命令端口系统**：

系统通过 **端口匹配** 自动组装命令链。

**任务类型到端口流映射**：

| 任务类型 | 输入端口 | 输出端口 | 示例管道 |
| --- | --- | --- | --- |
| bugfix | bug-report | test-passed | Bug 报告 → lite-fix → 修复 → test-passed |
| tdd | requirement | tdd-verified | 需求 → tdd-plan → execute → tdd-verify |
| test-fix | failing-tests | test-passed | 失败测试 → test-fix-gen → test-cycle-execute |
| feature | requirement | code/test-passed | 需求 → plan → execute → code |

**用户确认界面**：

```
Recommended Command Chain:

Pipeline (管道视图):
需求 → lite-plan → 计划 → lite-execute → 代码 → test-cycle-execute → 测试通过

Commands (命令列表):
1. /workflow:lite-plan
2. /workflow:lite-execute
3. /workflow:test-cycle-execute

Proceed? [Confirm / Show Details / Adjust / Cancel]
```

---

#### Phase 3：序列执行

**串行阻塞模型**：

- 一次执行一个命令
- 通过 Hook 回调延续
- 状态持久化

**执行流程**：

  
  
      ​
    

    #mermaid-diagram-4{font-family:"trebuchet ms",verdana,arial,sans-serif;font-size:16px;fill:#ccc;}@keyframes edge-animation-frame{from{stroke-dashoffset:0;}}@keyframes dash{to{stroke-dashoffset:0;}}#mermaid-diagram-4 .edge-animation-slow{stroke-dasharray:9,5!important;stroke-dashoffset:900;animation:dash 50s linear infinite;stroke-linecap:round;}#mermaid-diagram-4 .edge-animation-fast{stroke-dasharray:9,5!important;stroke-dashoffset:900;animation:dash 20s linear infinite;stroke-linecap:round;}#mermaid-diagram-4 .error-icon{fill:#a44141;}#mermaid-diagram-4 .error-text{fill:#ddd;stroke:#ddd;}#mermaid-diagram-4 .edge-thickness-normal{stroke-width:1px;}#mermaid-diagram-4 .edge-thickness-thick{stroke-width:3.5px;}#mermaid-diagram-4 .edge-pattern-solid{stroke-dasharray:0;}#mermaid-diagram-4 .edge-thickness-invisible{stroke-width:0;fill:none;}#mermaid-diagram-4 .edge-pattern-dashed{stroke-dasharray:3;}#mermaid-diagram-4 .edge-pattern-dotted{stroke-dasharray:2;}#mermaid-diagram-4 .marker{fill:lightgrey;stroke:lightgrey;}#mermaid-diagram-4 .marker.cross{stroke:lightgrey;}#mermaid-diagram-4 svg{font-family:"trebuchet ms",verdana,arial,sans-serif;font-size:16px;}#mermaid-diagram-4 p{margin:0;}#mermaid-diagram-4 defs [id$="-barbEnd"]{fill:lightgrey;stroke:lightgrey;}#mermaid-diagram-4 g.stateGroup text{fill:#ccc;stroke:none;font-size:10px;}#mermaid-diagram-4 g.stateGroup text{fill:#ccc;stroke:none;font-size:10px;}#mermaid-diagram-4 g.stateGroup .state-title{font-weight:bolder;fill:#e0dfdf;}#mermaid-diagram-4 g.stateGroup rect{fill:#1f2020;stroke:#ccc;}#mermaid-diagram-4 g.stateGroup line{stroke:lightgrey;stroke-width:1;}#mermaid-diagram-4 .transition{stroke:lightgrey;stroke-width:1;fill:none;}#mermaid-diagram-4 .stateGroup .composit{fill:#333;border-bottom:1px;}#mermaid-diagram-4 .stateGroup .alt-composit{fill:#e0e0e0;border-bottom:1px;}#mermaid-diagram-4 .state-note{stroke:hsl(180, 0%, 18.3529411765%);fill:hsl(180, 1.5873015873%, 28.3529411765%);}#mermaid-diagram-4 .state-note text{fill:rgb(183.8476190475, 181.5523809523, 181.5523809523);stroke:none;font-size:10px;}#mermaid-diagram-4 .stateLabel .box{stroke:none;stroke-width:0;fill:#1f2020;opacity:0.5;}#mermaid-diagram-4 .edgeLabel .label rect{fill:#1f2020;opacity:0.5;}#mermaid-diagram-4 .edgeLabel{background-color:hsl(0, 0%, 34.4117647059%);text-align:center;}#mermaid-diagram-4 .edgeLabel p{background-color:hsl(0, 0%, 34.4117647059%);}#mermaid-diagram-4 .edgeLabel rect{opacity:0.5;background-color:hsl(0, 0%, 34.4117647059%);fill:hsl(0, 0%, 34.4117647059%);}#mermaid-diagram-4 .edgeLabel .label text{fill:#ccc;}#mermaid-diagram-4 .label div .edgeLabel{color:#ccc;}#mermaid-diagram-4 .stateLabel text{fill:#e0dfdf;font-size:10px;font-weight:bold;}#mermaid-diagram-4 .node circle.state-start{fill:#f4f4f4;stroke:#f4f4f4;}#mermaid-diagram-4 .node .fork-join{fill:#f4f4f4;stroke:#f4f4f4;}#mermaid-diagram-4 .node circle.state-end{fill:#cccccc;stroke:#333;stroke-width:1.5;}#mermaid-diagram-4 .end-state-inner{fill:#333;stroke-width:1.5;}#mermaid-diagram-4 .node rect{fill:#1f2020;stroke:#ccc;stroke-width:1px;}#mermaid-diagram-4 .node polygon{fill:#1f2020;stroke:#ccc;stroke-width:1px;}#mermaid-diagram-4 [id$="-barbEnd"]{fill:lightgrey;}#mermaid-diagram-4 .statediagram-cluster rect{fill:#1f2020;stroke:#ccc;stroke-width:1px;}#mermaid-diagram-4 .cluster-label,#mermaid-diagram-4 .nodeLabel{color:#e0dfdf;}#mermaid-diagram-4 .statediagram-cluster rect.outer{rx:5px;ry:5px;}#mermaid-diagram-4 .statediagram-state .divider{stroke:#ccc;}#mermaid-diagram-4 .statediagram-state .title-state{rx:5px;ry:5px;}#mermaid-diagram-4 .statediagram-cluster.statediagram-cluster .inner{fill:#333;}#mermaid-diagram-4 .statediagram-cluster.statediagram-cluster-alt .inner{fill:#555;}#mermaid-diagram-4 .statediagram-cluster .inner{rx:0;ry:0;}#mermaid-diagram-4 .statediagram-state rect.basic{rx:5px;ry:5px;}#mermaid-diagram-4 .statediagram-state rect.divider{stroke-dasharray:10,10;fill:#555;}#mermaid-diagram-4 .note-edge{stroke-dasharray:5;}#mermaid-diagram-4 .statediagram-note rect{fill:hsl(180, 1.5873015873%, 28.3529411765%);stroke:hsl(180, 0%, 18.3529411765%);stroke-width:1px;rx:0;ry:0;}#mermaid-diagram-4 .statediagram-note rect{fill:hsl(180, 1.5873015873%, 28.3529411765%);stroke:hsl(180, 0%, 18.3529411765%);stroke-width:1px;rx:0;ry:0;}#mermaid-diagram-4 .statediagram-note text{fill:rgb(183.8476190475, 181.5523809523, 181.5523809523);}#mermaid-diagram-4 .statediagram-note .nodeLabel{color:rgb(183.8476190475, 181.5523809523, 181.5523809523);}#mermaid-diagram-4 .statediagram .edgeLabel{color:red;}#mermaid-diagram-4 [id$="-dependencyStart"],#mermaid-diagram-4 [id$="-dependencyEnd"]{fill:lightgrey;stroke:lightgrey;stroke-width:1;}#mermaid-diagram-4 .statediagramTitleText{text-anchor:middle;font-size:18px;fill:#ccc;}#mermaid-diagram-4 [data-look="neo"].statediagram-cluster rect{fill:#1f2020;stroke:url(#mermaid-diagram-4-gradient);stroke-width:1;}#mermaid-diagram-4 [data-look="neo"].statediagram-cluster rect.outer{rx:5px;ry:5px;filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#mermaid-diagram-4 .node .neo-node{stroke:#ccc;}#mermaid-diagram-4 [data-look="neo"].node rect,#mermaid-diagram-4 [data-look="neo"].cluster rect,#mermaid-diagram-4 [data-look="neo"].node polygon{stroke:url(#mermaid-diagram-4-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#mermaid-diagram-4 [data-look="neo"].node path{stroke:url(#mermaid-diagram-4-gradient);stroke-width:1px;}#mermaid-diagram-4 [data-look="neo"].node .outer-path{filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#mermaid-diagram-4 [data-look="neo"].node .neo-line path{stroke:#ccc;filter:none;}#mermaid-diagram-4 [data-look="neo"].node circle{stroke:url(#mermaid-diagram-4-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#mermaid-diagram-4 [data-look="neo"].node circle .state-start{fill:#000000;}#mermaid-diagram-4 [data-look="neo"].icon-shape .icon{fill:url(#mermaid-diagram-4-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#mermaid-diagram-4 [data-look="neo"].icon-shape .icon-neo path{stroke:url(#mermaid-diagram-4-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#mermaid-diagram-4 :root{--mermaid-font-family:"trebuchet ms",verdana,arial,sans-serif;}命令执行完成

Hook 回调触发

所有命令完成

执行失败

running

waiting

completed

failed

**状态值说明**：

- `running`：编排器主动执行（启动 CLI 命令）
- `waiting`：暂停，等待 hook 回调触发继续
- `completed`：所有命令成功完成
- `failed`：用户中止或不可恢复错误

---

### 3.3. 状态文件详解

**状态文件位置**：

```
.workflow/.ccw-coordinator/{session_id}/state.json
```

**状态文件结构**：

```
{
  "session_id": "ccw-coord-20250124-143025",
  "status": "running",
  "created_at": "2025-01-24T14:30:25Z",
  "updated_at": "2025-01-24T14:35:45Z",
  "analysis": {
    "goal": "实现用户注册",
    "scope": ["authentication", "user_management"],
    "constraints": ["no breaking changes"],
    "complexity": "medium",
    "task_type": "feature"
  },
  "command_chain": [
    {
      "index": 0,
      "command": "/workflow:plan",
      "name": "plan",
      "description": "详细规划",
      "status": "completed"
    },
    {
      "index": 1,
      "command": "/workflow:execute",
      "name": "execute",
      "description": "执行实现",
      "status": "running"
    }
  ],
  "execution_results": [
    {
      "index": 0,
      "command": "/workflow:plan",
      "status": "completed",
      "task_id": "task-001",
      "session_id": "WFS-plan-20250124",
      "artifacts": ["IMPL_PLAN.md"],
      "timestamp": "2025-01-24T14:30:25Z",
      "completed_at": "2025-01-24T14:30:45Z"
    }
  ],
  "prompts_used": []
}
```

**中断恢复**：

```
# 查看所有未完成的编排会话
ls .workflow/.ccw-coordinator/

# 查看状态文件
cat .workflow/.ccw-coordinator/ccw-coord-20250124-xxx/state.json

# 恢复执行（功能开发中）
/ccw-coordinator:resume ccw-coord-20250124-xxx
```

---

### 3.4. 典型场景演示

#### 场景 8：简单功能开发的自动编排

**需求**：实现用户头像上传功能。

**指令**：

```
/ccw-coordinator "实现用户头像上传功能"
```

**执行过程**：

1. **Phase 1：分析**   - Goal: 实现用户头像上传 - Complexity: simple - Task Type: feature
2. **Phase 2：推荐命令链**   - Pipeline: 需求 → lite-plan → lite-execute → 代码 → test-fix-gen → test-cycle-execute → 测试通过 - Commands: `lite-plan`, `lite-execute`, `test-fix-gen`, `test-cycle-execute`
3. **Phase 3：用户确认并执行**   - → `lite-plan`: 生成规划（内存） - → `lite-execute`: 实现代码 - → `test-fix-gen`: 生成测试任务 - → `test-cycle-execute`: 测试修复循环

**产出**：

```
.workflow/.ccw-coordinator/ccw-coord-20250124-xxx/state.json
```

**价值**：全自动，无需手动选择命令。

---

#### 场景 9：Bug 修复的自动编排

**需求**：修复支付超时问题。

**指令**：

```
/ccw-coordinator "修复支付超时问题"
```

**执行过程**：

1. **Phase 1：分析**   - Goal: 修复支付超时 - Task Type: bugfix
2. **Phase 2：推荐命令链**   - Pipeline: Bug 报告 → lite-fix → lite-execute → 修复 → test-fix-gen → test-cycle-execute → 测试通过 - Commands: `lite-fix`, `lite-execute`, `test-fix-gen`, `test-cycle-execute`
3. **Phase 3：执行**   - → `lite-fix`: 诊断根因，生成修复计划 - → `lite-execute`: 应用修复 - → `test-fix-gen`: 生成回归测试 - → `test-cycle-execute`: 验证修复

**产出**：

```
.workflow/.ccw-coordinator/ccw-coord-20250124-xxx/state.json
.workflow/.lite-fix/payment-timeout-20250124-xxx/diagnosis.json
```

**价值**：自动识别为 bugfix，推荐正确的命令链。

---

#### 场景 10：复杂功能开发的自动编排

**需求**：实现完整的实时协作编辑系统。

**指令**：

```
/ccw-coordinator "实现完整的实时协作编辑系统"
```

**执行过程**：

1. **Phase 1：分析**   - Goal: 实现实时协作编辑 - Complexity: complex - Task Type: feature
2. **Phase 2：推荐命令链**   - Pipeline: 需求 → plan → plan-verify → execute → 代码 → review-session-cycle → review-fix → 修复 - Commands: `plan`, `plan-verify`, `execute`, `review-session-cycle`, `review-fix`
3. **Phase 3：执行**   - → `plan`: 完整规划（持久化） - → `plan-verify`: 验证计划质量 - → `execute`: 实现功能 - → `review-session-cycle`: 多维度审查 - → `review-fix`: 应用审查修复

**产出**：

```
.workflow/.ccw-coordinator/ccw-coord-20250124-xxx/state.json
.workflow/active/WFS-realtime-collab-xxx/IMPL_PLAN.md
```

**价值**：自动识别为高复杂度，推荐完整的工作流。

---

## 第四章. /ccw vs /ccw-coordinator

CCW 提供 **2 个编排器**，分别适用不同场景。

### 4.1. 两个编排器的区别

| 维度 | /ccw | /ccw-coordinator |
| --- | --- | --- |
| 执行方式 | 主进程（SlashCommand） | 外部 CLI（后台任务） |
| 选择方式 | 自动基于意图识别 | 智能推荐 + 可选调整 |
| 状态管理 | TodoWrite 跟踪 | 持久化 state.json |
| 适用场景 | 通用任务、快速开发 | 复杂链条、可恢复 |
| 用户交互 | 自动执行 | 推荐 → 确认 → 执行 |
| 可调整性 | 无 | 可手动调整命令链 |

---

### 4.2. 如何选择

**使用 /ccw**：

- 通用任务
- 快速开发
- 不需要调整命令链
- 希望自动执行

**使用 /ccw-coordinator**：

- 复杂多步骤工作流
- 需要查看推荐的命令链
- 可能需要调整命令链
- 需要可恢复会话
- 需要完整状态追踪

**示例对比**：

```
# /ccw - 自动工作流选择（主进程）
/ccw "添加用户认证"                    # 自动根据意图选择工作流
/ccw "修复 WebSocket 中的内存泄漏"     # 识别为 bugfix 工作流
/ccw "使用 TDD 方式实现"                # 路由到 TDD 工作流

# /ccw-coordinator - 手动链编排（外部 CLI）
/ccw-coordinator "实现 OAuth2 系统"     # 分析 → 推荐链 → 用户确认 → 执行
```

---

## 第五章. 多 CLI 协同原理

### 5.1. CLI 工具注册

CCW 支持注册多种 CLI 工具。

**已支持的 CLI**：

| CLI 工具 | 擅长领域 |
| --- | --- |
| Gemini | 架构分析、代码理解 |
| Codex | 代码生成、自主编码 |
| Qwen | 中文场景、代码审查 |
| OpenCode | 开源多模型支持 |
| Claude | 通用开发（默认） |

**自定义 CLI 注册**：

通过 Dashboard 注册任意 API：

```
ccw view  # 打开 Dashboard → Status → API Settings
```

| 字段 | 示例 |
| --- | --- |
| 名称 | deepseek |
| 端点 | https://api.deepseek.com/v1/chat |
| API Key | your-api-key |

注册后，可以直接语义调用：

```
"使用 deepseek 分析这段代码"
```

---

### 5.2. 语义化调用机制

CCW 通过 **自然语言解析** 自动调用对应的 CLI。

**单 CLI 调用**：

| 提示词 | 系统动作 |
| --- | --- |
| “使用 Gemini 分析 auth 模块” | 自动调用 gemini CLI |
| “让 Codex 审查这段代码” | 自动调用 codex CLI |
| “问问 Qwen 这个错误怎么解决” | 自动调用 qwen CLI |

**实现原理**：

  
  
      ​
    

    #mermaid-diagram-5{font-family:"trebuchet ms",verdana,arial,sans-serif;font-size:16px;fill:#ccc;}@keyframes edge-animation-frame{from{stroke-dashoffset:0;}}@keyframes dash{to{stroke-dashoffset:0;}}#mermaid-diagram-5 .edge-animation-slow{stroke-dasharray:9,5!important;stroke-dashoffset:900;animation:dash 50s linear infinite;stroke-linecap:round;}#mermaid-diagram-5 .edge-animation-fast{stroke-dasharray:9,5!important;stroke-dashoffset:900;animation:dash 20s linear infinite;stroke-linecap:round;}#mermaid-diagram-5 .error-icon{fill:#a44141;}#mermaid-diagram-5 .error-text{fill:#ddd;stroke:#ddd;}#mermaid-diagram-5 .edge-thickness-normal{stroke-width:1px;}#mermaid-diagram-5 .edge-thickness-thick{stroke-width:3.5px;}#mermaid-diagram-5 .edge-pattern-solid{stroke-dasharray:0;}#mermaid-diagram-5 .edge-thickness-invisible{stroke-width:0;fill:none;}#mermaid-diagram-5 .edge-pattern-dashed{stroke-dasharray:3;}#mermaid-diagram-5 .edge-pattern-dotted{stroke-dasharray:2;}#mermaid-diagram-5 .marker{fill:lightgrey;stroke:lightgrey;}#mermaid-diagram-5 .marker.cross{stroke:lightgrey;}#mermaid-diagram-5 svg{font-family:"trebuchet ms",verdana,arial,sans-serif;font-size:16px;}#mermaid-diagram-5 p{margin:0;}#mermaid-diagram-5 .label{font-family:"trebuchet ms",verdana,arial,sans-serif;color:#ccc;}#mermaid-diagram-5 .cluster-label text{fill:#F9FFFE;}#mermaid-diagram-5 .cluster-label span{color:#F9FFFE;}#mermaid-diagram-5 .cluster-label span p{background-color:transparent;}#mermaid-diagram-5 .label text,#mermaid-diagram-5 span{fill:#ccc;color:#ccc;}#mermaid-diagram-5 .node rect,#mermaid-diagram-5 .node circle,#mermaid-diagram-5 .node ellipse,#mermaid-diagram-5 .node polygon,#mermaid-diagram-5 .node path{fill:#1f2020;stroke:#ccc;stroke-width:1px;}#mermaid-diagram-5 .rough-node .label text,#mermaid-diagram-5 .node .label text,#mermaid-diagram-5 .image-shape .label,#mermaid-diagram-5 .icon-shape .label{text-anchor:middle;}#mermaid-diagram-5 .node .katex path{fill:#000;stroke:#000;stroke-width:1px;}#mermaid-diagram-5 .rough-node .label,#mermaid-diagram-5 .node .label,#mermaid-diagram-5 .image-shape .label,#mermaid-diagram-5 .icon-shape .label{text-align:center;}#mermaid-diagram-5 .node.clickable{cursor:pointer;}#mermaid-diagram-5 .root .anchor path{fill:lightgrey!important;stroke-width:0;stroke:lightgrey;}#mermaid-diagram-5 .arrowheadPath{fill:lightgrey;}#mermaid-diagram-5 .edgePath .path{stroke:lightgrey;stroke-width:1px;}#mermaid-diagram-5 .flowchart-link{stroke:lightgrey;fill:none;}#mermaid-diagram-5 .edgeLabel{background-color:hsl(0, 0%, 34.4117647059%);text-align:center;}#mermaid-diagram-5 .edgeLabel p{background-color:hsl(0, 0%, 34.4117647059%);}#mermaid-diagram-5 .edgeLabel rect{opacity:0.5;background-color:hsl(0, 0%, 34.4117647059%);fill:hsl(0, 0%, 34.4117647059%);}#mermaid-diagram-5 .labelBkg{background-color:rgba(87.75, 87.75, 87.75, 0.5);}#mermaid-diagram-5 .cluster rect{fill:hsl(180, 1.5873015873%, 28.3529411765%);stroke:rgba(255, 255, 255, 0.25);stroke-width:1px;}#mermaid-diagram-5 .cluster text{fill:#F9FFFE;}#mermaid-diagram-5 .cluster span{color:#F9FFFE;}#mermaid-diagram-5 div.mermaidTooltip{position:absolute;text-align:center;max-width:200px;padding:2px;font-family:"trebuchet ms",verdana,arial,sans-serif;font-size:12px;background:hsl(20, 1.5873015873%, 12.3529411765%);border:1px solid rgba(255, 255, 255, 0.25);border-radius:2px;pointer-events:none;z-index:100;}#mermaid-diagram-5 .flowchartTitleText{text-anchor:middle;font-size:18px;fill:#ccc;}#mermaid-diagram-5 rect.text{fill:none;stroke-width:0;}#mermaid-diagram-5 .icon-shape,#mermaid-diagram-5 .image-shape{background-color:hsl(0, 0%, 34.4117647059%);text-align:center;}#mermaid-diagram-5 .icon-shape p,#mermaid-diagram-5 .image-shape p{background-color:hsl(0, 0%, 34.4117647059%);padding:2px;}#mermaid-diagram-5 .icon-shape .label rect,#mermaid-diagram-5 .image-shape .label rect{opacity:0.5;background-color:hsl(0, 0%, 34.4117647059%);fill:hsl(0, 0%, 34.4117647059%);}#mermaid-diagram-5 .label-icon{display:inline-block;height:1em;overflow:visible;vertical-align:-0.125em;}#mermaid-diagram-5 .node .label-icon path{fill:currentColor;stroke:revert;stroke-width:revert;}#mermaid-diagram-5 .node .neo-node{stroke:#ccc;}#mermaid-diagram-5 [data-look="neo"].node rect,#mermaid-diagram-5 [data-look="neo"].cluster rect,#mermaid-diagram-5 [data-look="neo"].node polygon{stroke:url(#mermaid-diagram-5-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#mermaid-diagram-5 [data-look="neo"].node path{stroke:url(#mermaid-diagram-5-gradient);stroke-width:1px;}#mermaid-diagram-5 [data-look="neo"].node .outer-path{filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#mermaid-diagram-5 [data-look="neo"].node .neo-line path{stroke:#ccc;filter:none;}#mermaid-diagram-5 [data-look="neo"].node circle{stroke:url(#mermaid-diagram-5-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#mermaid-diagram-5 [data-look="neo"].node circle .state-start{fill:#000000;}#mermaid-diagram-5 [data-look="neo"].icon-shape .icon{fill:url(#mermaid-diagram-5-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#mermaid-diagram-5 [data-look="neo"].icon-shape .icon-neo path{stroke:url(#mermaid-diagram-5-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#mermaid-diagram-5 :root{--mermaid-font-family:"trebuchet ms",verdana,arial,sans-serif;}用户提示词

解析 CLI 名称

查找 CLI 配置

构造 API 请求

调用 CLI

返回结果

---

### 5.3. 协同模式详解

CCW 支持 **4 种协同模式**。

#### 模式 1：协同分析

**提示词**：

```
"使用 Gemini 和 Codex 协同分析安全漏洞"
```

**执行流程**：

1. Gemini 分析安全漏洞
2. Codex 分析安全漏洞
3. 合并两个分析结果
4. 生成综合报告

**价值**：交叉验证，降低误判。

---

#### 模式 2：并行执行

**提示词**：

```
"让 Gemini、Codex、Qwen 并行分析架构设计"
```

**执行流程**：

  
  
      ​
    

    #mermaid-diagram-6{font-family:"trebuchet ms",verdana,arial,sans-serif;font-size:16px;fill:#ccc;}@keyframes edge-animation-frame{from{stroke-dashoffset:0;}}@keyframes dash{to{stroke-dashoffset:0;}}#mermaid-diagram-6 .edge-animation-slow{stroke-dasharray:9,5!important;stroke-dashoffset:900;animation:dash 50s linear infinite;stroke-linecap:round;}#mermaid-diagram-6 .edge-animation-fast{stroke-dasharray:9,5!important;stroke-dashoffset:900;animation:dash 20s linear infinite;stroke-linecap:round;}#mermaid-diagram-6 .error-icon{fill:#a44141;}#mermaid-diagram-6 .error-text{fill:#ddd;stroke:#ddd;}#mermaid-diagram-6 .edge-thickness-normal{stroke-width:1px;}#mermaid-diagram-6 .edge-thickness-thick{stroke-width:3.5px;}#mermaid-diagram-6 .edge-pattern-solid{stroke-dasharray:0;}#mermaid-diagram-6 .edge-thickness-invisible{stroke-width:0;fill:none;}#mermaid-diagram-6 .edge-pattern-dashed{stroke-dasharray:3;}#mermaid-diagram-6 .edge-pattern-dotted{stroke-dasharray:2;}#mermaid-diagram-6 .marker{fill:lightgrey;stroke:lightgrey;}#mermaid-diagram-6 .marker.cross{stroke:lightgrey;}#mermaid-diagram-6 svg{font-family:"trebuchet ms",verdana,arial,sans-serif;font-size:16px;}#mermaid-diagram-6 p{margin:0;}#mermaid-diagram-6 .label{font-family:"trebuchet ms",verdana,arial,sans-serif;color:#ccc;}#mermaid-diagram-6 .cluster-label text{fill:#F9FFFE;}#mermaid-diagram-6 .cluster-label span{color:#F9FFFE;}#mermaid-diagram-6 .cluster-label span p{background-color:transparent;}#mermaid-diagram-6 .label text,#mermaid-diagram-6 span{fill:#ccc;color:#ccc;}#mermaid-diagram-6 .node rect,#mermaid-diagram-6 .node circle,#mermaid-diagram-6 .node ellipse,#mermaid-diagram-6 .node polygon,#mermaid-diagram-6 .node path{fill:#1f2020;stroke:#ccc;stroke-width:1px;}#mermaid-diagram-6 .rough-node .label text,#mermaid-diagram-6 .node .label text,#mermaid-diagram-6 .image-shape .label,#mermaid-diagram-6 .icon-shape .label{text-anchor:middle;}#mermaid-diagram-6 .node .katex path{fill:#000;stroke:#000;stroke-width:1px;}#mermaid-diagram-6 .rough-node .label,#mermaid-diagram-6 .node .label,#mermaid-diagram-6 .image-shape .label,#mermaid-diagram-6 .icon-shape .label{text-align:center;}#mermaid-diagram-6 .node.clickable{cursor:pointer;}#mermaid-diagram-6 .root .anchor path{fill:lightgrey!important;stroke-width:0;stroke:lightgrey;}#mermaid-diagram-6 .arrowheadPath{fill:lightgrey;}#mermaid-diagram-6 .edgePath .path{stroke:lightgrey;stroke-width:1px;}#mermaid-diagram-6 .flowchart-link{stroke:lightgrey;fill:none;}#mermaid-diagram-6 .edgeLabel{background-color:hsl(0, 0%, 34.4117647059%);text-align:center;}#mermaid-diagram-6 .edgeLabel p{background-color:hsl(0, 0%, 34.4117647059%);}#mermaid-diagram-6 .edgeLabel rect{opacity:0.5;background-color:hsl(0, 0%, 34.4117647059%);fill:hsl(0, 0%, 34.4117647059%);}#mermaid-diagram-6 .labelBkg{background-color:rgba(87.75, 87.75, 87.75, 0.5);}#mermaid-diagram-6 .cluster rect{fill:hsl(180, 1.5873015873%, 28.3529411765%);stroke:rgba(255, 255, 255, 0.25);stroke-width:1px;}#mermaid-diagram-6 .cluster text{fill:#F9FFFE;}#mermaid-diagram-6 .cluster span{color:#F9FFFE;}#mermaid-diagram-6 div.mermaidTooltip{position:absolute;text-align:center;max-width:200px;padding:2px;font-family:"trebuchet ms",verdana,arial,sans-serif;font-size:12px;background:hsl(20, 1.5873015873%, 12.3529411765%);border:1px solid rgba(255, 255, 255, 0.25);border-radius:2px;pointer-events:none;z-index:100;}#mermaid-diagram-6 .flowchartTitleText{text-anchor:middle;font-size:18px;fill:#ccc;}#mermaid-diagram-6 rect.text{fill:none;stroke-width:0;}#mermaid-diagram-6 .icon-shape,#mermaid-diagram-6 .image-shape{background-color:hsl(0, 0%, 34.4117647059%);text-align:center;}#mermaid-diagram-6 .icon-shape p,#mermaid-diagram-6 .image-shape p{background-color:hsl(0, 0%, 34.4117647059%);padding:2px;}#mermaid-diagram-6 .icon-shape .label rect,#mermaid-diagram-6 .image-shape .label rect{opacity:0.5;background-color:hsl(0, 0%, 34.4117647059%);fill:hsl(0, 0%, 34.4117647059%);}#mermaid-diagram-6 .label-icon{display:inline-block;height:1em;overflow:visible;vertical-align:-0.125em;}#mermaid-diagram-6 .node .label-icon path{fill:currentColor;stroke:revert;stroke-width:revert;}#mermaid-diagram-6 .node .neo-node{stroke:#ccc;}#mermaid-diagram-6 [data-look="neo"].node rect,#mermaid-diagram-6 [data-look="neo"].cluster rect,#mermaid-diagram-6 [data-look="neo"].node polygon{stroke:url(#mermaid-diagram-6-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#mermaid-diagram-6 [data-look="neo"].node path{stroke:url(#mermaid-diagram-6-gradient);stroke-width:1px;}#mermaid-diagram-6 [data-look="neo"].node .outer-path{filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#mermaid-diagram-6 [data-look="neo"].node .neo-line path{stroke:#ccc;filter:none;}#mermaid-diagram-6 [data-look="neo"].node circle{stroke:url(#mermaid-diagram-6-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#mermaid-diagram-6 [data-look="neo"].node circle .state-start{fill:#000000;}#mermaid-diagram-6 [data-look="neo"].icon-shape .icon{fill:url(#mermaid-diagram-6-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#mermaid-diagram-6 [data-look="neo"].icon-shape .icon-neo path{stroke:url(#mermaid-diagram-6-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#mermaid-diagram-6 :root{--mermaid-font-family:"trebuchet ms",verdana,arial,sans-serif;}需求

Gemini 分析

Codex 分析

Qwen 分析

合并结果

多视角报告

**价值**：多视角，全面覆盖。

---

#### 模式 3：流水线

**提示词**：

```
"Gemini 设计方案，Codex 实现，Claude 审查"
```

**执行流程**：

  
  
      ​
    

    #mermaid-diagram-7{font-family:"trebuchet ms",verdana,arial,sans-serif;font-size:16px;fill:#ccc;}@keyframes edge-animation-frame{from{stroke-dashoffset:0;}}@keyframes dash{to{stroke-dashoffset:0;}}#mermaid-diagram-7 .edge-animation-slow{stroke-dasharray:9,5!important;stroke-dashoffset:900;animation:dash 50s linear infinite;stroke-linecap:round;}#mermaid-diagram-7 .edge-animation-fast{stroke-dasharray:9,5!important;stroke-dashoffset:900;animation:dash 20s linear infinite;stroke-linecap:round;}#mermaid-diagram-7 .error-icon{fill:#a44141;}#mermaid-diagram-7 .error-text{fill:#ddd;stroke:#ddd;}#mermaid-diagram-7 .edge-thickness-normal{stroke-width:1px;}#mermaid-diagram-7 .edge-thickness-thick{stroke-width:3.5px;}#mermaid-diagram-7 .edge-pattern-solid{stroke-dasharray:0;}#mermaid-diagram-7 .edge-thickness-invisible{stroke-width:0;fill:none;}#mermaid-diagram-7 .edge-pattern-dashed{stroke-dasharray:3;}#mermaid-diagram-7 .edge-pattern-dotted{stroke-dasharray:2;}#mermaid-diagram-7 .marker{fill:lightgrey;stroke:lightgrey;}#mermaid-diagram-7 .marker.cross{stroke:lightgrey;}#mermaid-diagram-7 svg{font-family:"trebuchet ms",verdana,arial,sans-serif;font-size:16px;}#mermaid-diagram-7 p{margin:0;}#mermaid-diagram-7 .label{font-family:"trebuchet ms",verdana,arial,sans-serif;color:#ccc;}#mermaid-diagram-7 .cluster-label text{fill:#F9FFFE;}#mermaid-diagram-7 .cluster-label span{color:#F9FFFE;}#mermaid-diagram-7 .cluster-label span p{background-color:transparent;}#mermaid-diagram-7 .label text,#mermaid-diagram-7 span{fill:#ccc;color:#ccc;}#mermaid-diagram-7 .node rect,#mermaid-diagram-7 .node circle,#mermaid-diagram-7 .node ellipse,#mermaid-diagram-7 .node polygon,#mermaid-diagram-7 .node path{fill:#1f2020;stroke:#ccc;stroke-width:1px;}#mermaid-diagram-7 .rough-node .label text,#mermaid-diagram-7 .node .label text,#mermaid-diagram-7 .image-shape .label,#mermaid-diagram-7 .icon-shape .label{text-anchor:middle;}#mermaid-diagram-7 .node .katex path{fill:#000;stroke:#000;stroke-width:1px;}#mermaid-diagram-7 .rough-node .label,#mermaid-diagram-7 .node .label,#mermaid-diagram-7 .image-shape .label,#mermaid-diagram-7 .icon-shape .label{text-align:center;}#mermaid-diagram-7 .node.clickable{cursor:pointer;}#mermaid-diagram-7 .root .anchor path{fill:lightgrey!important;stroke-width:0;stroke:lightgrey;}#mermaid-diagram-7 .arrowheadPath{fill:lightgrey;}#mermaid-diagram-7 .edgePath .path{stroke:lightgrey;stroke-width:1px;}#mermaid-diagram-7 .flowchart-link{stroke:lightgrey;fill:none;}#mermaid-diagram-7 .edgeLabel{background-color:hsl(0, 0%, 34.4117647059%);text-align:center;}#mermaid-diagram-7 .edgeLabel p{background-color:hsl(0, 0%, 34.4117647059%);}#mermaid-diagram-7 .edgeLabel rect{opacity:0.5;background-color:hsl(0, 0%, 34.4117647059%);fill:hsl(0, 0%, 34.4117647059%);}#mermaid-diagram-7 .labelBkg{background-color:rgba(87.75, 87.75, 87.75, 0.5);}#mermaid-diagram-7 .cluster rect{fill:hsl(180, 1.5873015873%, 28.3529411765%);stroke:rgba(255, 255, 255, 0.25);stroke-width:1px;}#mermaid-diagram-7 .cluster text{fill:#F9FFFE;}#mermaid-diagram-7 .cluster span{color:#F9FFFE;}#mermaid-diagram-7 div.mermaidTooltip{position:absolute;text-align:center;max-width:200px;padding:2px;font-family:"trebuchet ms",verdana,arial,sans-serif;font-size:12px;background:hsl(20, 1.5873015873%, 12.3529411765%);border:1px solid rgba(255, 255, 255, 0.25);border-radius:2px;pointer-events:none;z-index:100;}#mermaid-diagram-7 .flowchartTitleText{text-anchor:middle;font-size:18px;fill:#ccc;}#mermaid-diagram-7 rect.text{fill:none;stroke-width:0;}#mermaid-diagram-7 .icon-shape,#mermaid-diagram-7 .image-shape{background-color:hsl(0, 0%, 34.4117647059%);text-align:center;}#mermaid-diagram-7 .icon-shape p,#mermaid-diagram-7 .image-shape p{background-color:hsl(0, 0%, 34.4117647059%);padding:2px;}#mermaid-diagram-7 .icon-shape .label rect,#mermaid-diagram-7 .image-shape .label rect{opacity:0.5;background-color:hsl(0, 0%, 34.4117647059%);fill:hsl(0, 0%, 34.4117647059%);}#mermaid-diagram-7 .label-icon{display:inline-block;height:1em;overflow:visible;vertical-align:-0.125em;}#mermaid-diagram-7 .node .label-icon path{fill:currentColor;stroke:revert;stroke-width:revert;}#mermaid-diagram-7 .node .neo-node{stroke:#ccc;}#mermaid-diagram-7 [data-look="neo"].node rect,#mermaid-diagram-7 [data-look="neo"].cluster rect,#mermaid-diagram-7 [data-look="neo"].node polygon{stroke:url(#mermaid-diagram-7-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#mermaid-diagram-7 [data-look="neo"].node path{stroke:url(#mermaid-diagram-7-gradient);stroke-width:1px;}#mermaid-diagram-7 [data-look="neo"].node .outer-path{filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#mermaid-diagram-7 [data-look="neo"].node .neo-line path{stroke:#ccc;filter:none;}#mermaid-diagram-7 [data-look="neo"].node circle{stroke:url(#mermaid-diagram-7-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#mermaid-diagram-7 [data-look="neo"].node circle .state-start{fill:#000000;}#mermaid-diagram-7 [data-look="neo"].icon-shape .icon{fill:url(#mermaid-diagram-7-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#mermaid-diagram-7 [data-look="neo"].icon-shape .icon-neo path{stroke:url(#mermaid-diagram-7-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#mermaid-diagram-7 :root{--mermaid-font-family:"trebuchet ms",verdana,arial,sans-serif;}需求

Gemini 设计方案

Codex 实现代码

Claude 审查代码

最终产出

**价值**：流水线作业，各司其职。

---

#### 模式 4：迭代优化

**提示词**：

```
"用 Gemini 诊断问题，然后 Codex 修复，迭代直到解决"
```

**执行流程**：

  
  
      ​
    

    #mermaid-diagram-8{font-family:"trebuchet ms",verdana,arial,sans-serif;font-size:16px;fill:#ccc;}@keyframes edge-animation-frame{from{stroke-dashoffset:0;}}@keyframes dash{to{stroke-dashoffset:0;}}#mermaid-diagram-8 .edge-animation-slow{stroke-dasharray:9,5!important;stroke-dashoffset:900;animation:dash 50s linear infinite;stroke-linecap:round;}#mermaid-diagram-8 .edge-animation-fast{stroke-dasharray:9,5!important;stroke-dashoffset:900;animation:dash 20s linear infinite;stroke-linecap:round;}#mermaid-diagram-8 .error-icon{fill:#a44141;}#mermaid-diagram-8 .error-text{fill:#ddd;stroke:#ddd;}#mermaid-diagram-8 .edge-thickness-normal{stroke-width:1px;}#mermaid-diagram-8 .edge-thickness-thick{stroke-width:3.5px;}#mermaid-diagram-8 .edge-pattern-solid{stroke-dasharray:0;}#mermaid-diagram-8 .edge-thickness-invisible{stroke-width:0;fill:none;}#mermaid-diagram-8 .edge-pattern-dashed{stroke-dasharray:3;}#mermaid-diagram-8 .edge-pattern-dotted{stroke-dasharray:2;}#mermaid-diagram-8 .marker{fill:lightgrey;stroke:lightgrey;}#mermaid-diagram-8 .marker.cross{stroke:lightgrey;}#mermaid-diagram-8 svg{font-family:"trebuchet ms",verdana,arial,sans-serif;font-size:16px;}#mermaid-diagram-8 p{margin:0;}#mermaid-diagram-8 .label{font-family:"trebuchet ms",verdana,arial,sans-serif;color:#ccc;}#mermaid-diagram-8 .cluster-label text{fill:#F9FFFE;}#mermaid-diagram-8 .cluster-label span{color:#F9FFFE;}#mermaid-diagram-8 .cluster-label span p{background-color:transparent;}#mermaid-diagram-8 .label text,#mermaid-diagram-8 span{fill:#ccc;color:#ccc;}#mermaid-diagram-8 .node rect,#mermaid-diagram-8 .node circle,#mermaid-diagram-8 .node ellipse,#mermaid-diagram-8 .node polygon,#mermaid-diagram-8 .node path{fill:#1f2020;stroke:#ccc;stroke-width:1px;}#mermaid-diagram-8 .rough-node .label text,#mermaid-diagram-8 .node .label text,#mermaid-diagram-8 .image-shape .label,#mermaid-diagram-8 .icon-shape .label{text-anchor:middle;}#mermaid-diagram-8 .node .katex path{fill:#000;stroke:#000;stroke-width:1px;}#mermaid-diagram-8 .rough-node .label,#mermaid-diagram-8 .node .label,#mermaid-diagram-8 .image-shape .label,#mermaid-diagram-8 .icon-shape .label{text-align:center;}#mermaid-diagram-8 .node.clickable{cursor:pointer;}#mermaid-diagram-8 .root .anchor path{fill:lightgrey!important;stroke-width:0;stroke:lightgrey;}#mermaid-diagram-8 .arrowheadPath{fill:lightgrey;}#mermaid-diagram-8 .edgePath .path{stroke:lightgrey;stroke-width:1px;}#mermaid-diagram-8 .flowchart-link{stroke:lightgrey;fill:none;}#mermaid-diagram-8 .edgeLabel{background-color:hsl(0, 0%, 34.4117647059%);text-align:center;}#mermaid-diagram-8 .edgeLabel p{background-color:hsl(0, 0%, 34.4117647059%);}#mermaid-diagram-8 .edgeLabel rect{opacity:0.5;background-color:hsl(0, 0%, 34.4117647059%);fill:hsl(0, 0%, 34.4117647059%);}#mermaid-diagram-8 .labelBkg{background-color:rgba(87.75, 87.75, 87.75, 0.5);}#mermaid-diagram-8 .cluster rect{fill:hsl(180, 1.5873015873%, 28.3529411765%);stroke:rgba(255, 255, 255, 0.25);stroke-width:1px;}#mermaid-diagram-8 .cluster text{fill:#F9FFFE;}#mermaid-diagram-8 .cluster span{color:#F9FFFE;}#mermaid-diagram-8 div.mermaidTooltip{position:absolute;text-align:center;max-width:200px;padding:2px;font-family:"trebuchet ms",verdana,arial,sans-serif;font-size:12px;background:hsl(20, 1.5873015873%, 12.3529411765%);border:1px solid rgba(255, 255, 255, 0.25);border-radius:2px;pointer-events:none;z-index:100;}#mermaid-diagram-8 .flowchartTitleText{text-anchor:middle;font-size:18px;fill:#ccc;}#mermaid-diagram-8 rect.text{fill:none;stroke-width:0;}#mermaid-diagram-8 .icon-shape,#mermaid-diagram-8 .image-shape{background-color:hsl(0, 0%, 34.4117647059%);text-align:center;}#mermaid-diagram-8 .icon-shape p,#mermaid-diagram-8 .image-shape p{background-color:hsl(0, 0%, 34.4117647059%);padding:2px;}#mermaid-diagram-8 .icon-shape .label rect,#mermaid-diagram-8 .image-shape .label rect{opacity:0.5;background-color:hsl(0, 0%, 34.4117647059%);fill:hsl(0, 0%, 34.4117647059%);}#mermaid-diagram-8 .label-icon{display:inline-block;height:1em;overflow:visible;vertical-align:-0.125em;}#mermaid-diagram-8 .node .label-icon path{fill:currentColor;stroke:revert;stroke-width:revert;}#mermaid-diagram-8 .node .neo-node{stroke:#ccc;}#mermaid-diagram-8 [data-look="neo"].node rect,#mermaid-diagram-8 [data-look="neo"].cluster rect,#mermaid-diagram-8 [data-look="neo"].node polygon{stroke:url(#mermaid-diagram-8-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#mermaid-diagram-8 [data-look="neo"].node path{stroke:url(#mermaid-diagram-8-gradient);stroke-width:1px;}#mermaid-diagram-8 [data-look="neo"].node .outer-path{filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#mermaid-diagram-8 [data-look="neo"].node .neo-line path{stroke:#ccc;filter:none;}#mermaid-diagram-8 [data-look="neo"].node circle{stroke:url(#mermaid-diagram-8-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#mermaid-diagram-8 [data-look="neo"].node circle .state-start{fill:#000000;}#mermaid-diagram-8 [data-look="neo"].icon-shape .icon{fill:url(#mermaid-diagram-8-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#mermaid-diagram-8 [data-look="neo"].icon-shape .icon-neo path{stroke:url(#mermaid-diagram-8-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#mermaid-diagram-8 :root{--mermaid-font-family:"trebuchet ms",verdana,arial,sans-serif;}否

是

开始

Gemini 诊断问题

Codex 修复问题

问题解决?

完成

**价值**：自动迭代，直到问题解决。

---

### 5.4. multi-cli-plan 深度解析

`multi-cli-plan` 是多 CLI 协同的典型应用。

**5 阶段流程**：

  
  
      ​
    

    #mermaid-diagram-9{font-family:"trebuchet ms",verdana,arial,sans-serif;font-size:16px;fill:#ccc;}@keyframes edge-animation-frame{from{stroke-dashoffset:0;}}@keyframes dash{to{stroke-dashoffset:0;}}#mermaid-diagram-9 .edge-animation-slow{stroke-dasharray:9,5!important;stroke-dashoffset:900;animation:dash 50s linear infinite;stroke-linecap:round;}#mermaid-diagram-9 .edge-animation-fast{stroke-dasharray:9,5!important;stroke-dashoffset:900;animation:dash 20s linear infinite;stroke-linecap:round;}#mermaid-diagram-9 .error-icon{fill:#a44141;}#mermaid-diagram-9 .error-text{fill:#ddd;stroke:#ddd;}#mermaid-diagram-9 .edge-thickness-normal{stroke-width:1px;}#mermaid-diagram-9 .edge-thickness-thick{stroke-width:3.5px;}#mermaid-diagram-9 .edge-pattern-solid{stroke-dasharray:0;}#mermaid-diagram-9 .edge-thickness-invisible{stroke-width:0;fill:none;}#mermaid-diagram-9 .edge-pattern-dashed{stroke-dasharray:3;}#mermaid-diagram-9 .edge-pattern-dotted{stroke-dasharray:2;}#mermaid-diagram-9 .marker{fill:lightgrey;stroke:lightgrey;}#mermaid-diagram-9 .marker.cross{stroke:lightgrey;}#mermaid-diagram-9 svg{font-family:"trebuchet ms",verdana,arial,sans-serif;font-size:16px;}#mermaid-diagram-9 p{margin:0;}#mermaid-diagram-9 .label{font-family:"trebuchet ms",verdana,arial,sans-serif;color:#ccc;}#mermaid-diagram-9 .cluster-label text{fill:#F9FFFE;}#mermaid-diagram-9 .cluster-label span{color:#F9FFFE;}#mermaid-diagram-9 .cluster-label span p{background-color:transparent;}#mermaid-diagram-9 .label text,#mermaid-diagram-9 span{fill:#ccc;color:#ccc;}#mermaid-diagram-9 .node rect,#mermaid-diagram-9 .node circle,#mermaid-diagram-9 .node ellipse,#mermaid-diagram-9 .node polygon,#mermaid-diagram-9 .node path{fill:#1f2020;stroke:#ccc;stroke-width:1px;}#mermaid-diagram-9 .rough-node .label text,#mermaid-diagram-9 .node .label text,#mermaid-diagram-9 .image-shape .label,#mermaid-diagram-9 .icon-shape .label{text-anchor:middle;}#mermaid-diagram-9 .node .katex path{fill:#000;stroke:#000;stroke-width:1px;}#mermaid-diagram-9 .rough-node .label,#mermaid-diagram-9 .node .label,#mermaid-diagram-9 .image-shape .label,#mermaid-diagram-9 .icon-shape .label{text-align:center;}#mermaid-diagram-9 .node.clickable{cursor:pointer;}#mermaid-diagram-9 .root .anchor path{fill:lightgrey!important;stroke-width:0;stroke:lightgrey;}#mermaid-diagram-9 .arrowheadPath{fill:lightgrey;}#mermaid-diagram-9 .edgePath .path{stroke:lightgrey;stroke-width:1px;}#mermaid-diagram-9 .flowchart-link{stroke:lightgrey;fill:none;}#mermaid-diagram-9 .edgeLabel{background-color:hsl(0, 0%, 34.4117647059%);text-align:center;}#mermaid-diagram-9 .edgeLabel p{background-color:hsl(0, 0%, 34.4117647059%);}#mermaid-diagram-9 .edgeLabel rect{opacity:0.5;background-color:hsl(0, 0%, 34.4117647059%);fill:hsl(0, 0%, 34.4117647059%);}#mermaid-diagram-9 .labelBkg{background-color:rgba(87.75, 87.75, 87.75, 0.5);}#mermaid-diagram-9 .cluster rect{fill:hsl(180, 1.5873015873%, 28.3529411765%);stroke:rgba(255, 255, 255, 0.25);stroke-width:1px;}#mermaid-diagram-9 .cluster text{fill:#F9FFFE;}#mermaid-diagram-9 .cluster span{color:#F9FFFE;}#mermaid-diagram-9 div.mermaidTooltip{position:absolute;text-align:center;max-width:200px;padding:2px;font-family:"trebuchet ms",verdana,arial,sans-serif;font-size:12px;background:hsl(20, 1.5873015873%, 12.3529411765%);border:1px solid rgba(255, 255, 255, 0.25);border-radius:2px;pointer-events:none;z-index:100;}#mermaid-diagram-9 .flowchartTitleText{text-anchor:middle;font-size:18px;fill:#ccc;}#mermaid-diagram-9 rect.text{fill:none;stroke-width:0;}#mermaid-diagram-9 .icon-shape,#mermaid-diagram-9 .image-shape{background-color:hsl(0, 0%, 34.4117647059%);text-align:center;}#mermaid-diagram-9 .icon-shape p,#mermaid-diagram-9 .image-shape p{background-color:hsl(0, 0%, 34.4117647059%);padding:2px;}#mermaid-diagram-9 .icon-shape .label rect,#mermaid-diagram-9 .image-shape .label rect{opacity:0.5;background-color:hsl(0, 0%, 34.4117647059%);fill:hsl(0, 0%, 34.4117647059%);}#mermaid-diagram-9 .label-icon{display:inline-block;height:1em;overflow:visible;vertical-align:-0.125em;}#mermaid-diagram-9 .node .label-icon path{fill:currentColor;stroke:revert;stroke-width:revert;}#mermaid-diagram-9 .node .neo-node{stroke:#ccc;}#mermaid-diagram-9 [data-look="neo"].node rect,#mermaid-diagram-9 [data-look="neo"].cluster rect,#mermaid-diagram-9 [data-look="neo"].node polygon{stroke:url(#mermaid-diagram-9-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#mermaid-diagram-9 [data-look="neo"].node path{stroke:url(#mermaid-diagram-9-gradient);stroke-width:1px;}#mermaid-diagram-9 [data-look="neo"].node .outer-path{filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#mermaid-diagram-9 [data-look="neo"].node .neo-line path{stroke:#ccc;filter:none;}#mermaid-diagram-9 [data-look="neo"].node circle{stroke:url(#mermaid-diagram-9-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#mermaid-diagram-9 [data-look="neo"].node circle .state-start{fill:#000000;}#mermaid-diagram-9 [data-look="neo"].icon-shape .icon{fill:url(#mermaid-diagram-9-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#mermaid-diagram-9 [data-look="neo"].icon-shape .icon-neo path{stroke:url(#mermaid-diagram-9-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#mermaid-diagram-9 :root{--mermaid-font-family:"trebuchet ms",verdana,arial,sans-serif;}否

是

Phase 1: Context Gathering

ACE 语义搜索

构建上下文包

Phase 2: Multi-CLI Discussion

cli-discuss-agent 执行

Gemini + Codex + Claude

交叉验证，合成方案

收敛?

Phase 3: Present Options

展示方案及权衡

Phase 4: User Decision

用户选择方案

Phase 5: Plan Generation

cli-lite-planning-agent

```
N --> O[生成计划]
O --> P[lite-execute]
```

```

**产物结构**：

```bash
.workflow/.multi-cli-plan/{MCP-task-slug-date}/
├── rounds/
│   ├── round-1/
│   │   └── synthesis.json
│   ├── round-2/
│   │   └── synthesis.json
│   └── ...
├── context-package.json
├── IMPL_PLAN.md
└── plan.json
```

**multi-cli-plan vs lite-plan 对比**：

| 维度 | multi-cli-plan | lite-plan |
| --- | --- | --- |
| 上下文 | ACE 语义搜索（需提前安装） | 手动文件模式 |
| 分析 | 多 CLI 交叉验证 | 单次规划 |
| 迭代 | 多轮直到收敛 | 单轮 |
| 置信度 | 高（共识驱动） | 中（单一视角） |

---

## 第六章. 生产环境最佳实践

### 6.1. 记忆管理策略

**日常开发**：

```
# 每次提交前更新记忆
git add .
/memory:update-related
git commit -m "feat: 添加用户认证"
```

**新项目初始化**：

```
# 全量构建记忆
/memory:update-full
# 初始化工作流
/workflow:init
```

**定期维护**：

```
# 每周全量更新一次
/memory:update-full
```

**价值**：

- 保持 AI 对项目的最新理解
- 提高代码生成质量
- 减少上下文丢失

---

### 6.2. Session 管理策略

**命名规范**：

```
WFS-{feature-name}-{version}

示例：
WFS-user-auth-v1
WFS-payment-gateway-v2
WFS-realtime-collab-v1
```

**Session 生命周期**：

1. **创建**：`/workflow:plan`
2. **执行**：`/workflow:execute`
3. **暂停**：中断后自动保存
4. **恢复**：`/workflow:session resume WFS-xxx`
5. **完成**：`/workflow:session:complete`

**Session 清理**：

```
# 查看所有 Session
ls .workflow/active/

# 归档已完成的 Session
mv .workflow/active/WFS-xxx .workflow/archived/

# 删除失败的 Session
rm -rf .workflow/active/WFS-failed-xxx
```

---

### 6.3. 工作流选择策略

**决策矩阵**：

| 场景特征 | 推荐工作流 | 理由 |
| --- | --- | --- |
| 需求明确 + 单模块 | lite-plan | 快速迭代 |
| 需求明确 + 多模块 | plan | Session 持久化 |
| Bug 修复 | lite-fix | 5 阶段诊断 |
| 技术选型 | multi-cli-plan | 多视角分析 |
| TDD 开发 | tdd-plan | Red-Green-Refactor |
| 测试失败 | test-fix-gen | 自动修复循环 |
| 新功能设计 | brainstorm | 多角色协同 |
| 不知道用什么 | ccw-coordinator | 自动推荐 |

---

### 6.4. 代码审查策略

**开发阶段**：

```
# 每完成一个功能，立即审查
/workflow:review-fix
```

**提交前**：

```
# 提交前全面审查
/workflow:review
```

**专项审查**：

```
# 安全审查
/workflow:review --type security

# 性能审查
/workflow:review --type performance

# 架构审查
/workflow:review --type architecture
```

**Session 审查**：

```
# 审查整个 Session
/workflow:review-session-cycle
/workflow:review-fix
```

---

### 6.5. 测试策略

**TDD 开发**：

```
# 测试驱动开发
/workflow:tdd-plan "实现用户注册"
/workflow:execute
/workflow:tdd-verify
```

**后置测试**：

```
# 功能开发完成后补充测试
/workflow:test-fix-gen WFS-user-auth-v1
/workflow:test-cycle-execute
```

**测试修复**：

```
# 测试失败时
/workflow:test-fix-gen "修复集成测试失败"
/workflow:test-cycle-execute
```

---

### 6.6. Issue 管理策略

**积累阶段**：

```
# 每天自动发现
/issue:discover

# 手动记录用户反馈
/issue:new "用户反馈的问题"
```

**批量解决**：

```
# 每周批量处理
/issue:plan --all-pending
/issue:queue
/issue:execute
```

**优先级管理**：

- Critical：立即处理（`lite-fix --hotfix`）
- High：当天处理
- Medium：本周处理
- Low：下周处理

---

## 第七章. 常见问题与解决方案

### 7.1. Session 相关

**问题 1：Session 中断后如何恢复？**

```
# 查看所有 Session
ls .workflow/active/

# 恢复 Session
/workflow:session resume WFS-xxx
```

---

**问题 2：Session 文件太大怎么办？**

```
# 清理中间产物
rm -rf .workflow/active/WFS-xxx/.process/

# 只保留核心文件
# - workflow-session.json
# - IMPL_PLAN.md
# - .task/*.json
```

---

### 7.2. 记忆相关

**问题 3：记忆更新失败？**

```
# 检查记忆目录
ls .claude/memory/

# 重建记忆
/memory:update-full
```

---

**问题 4：AI 不理解我的代码？**

```
# 更新记忆
/memory:update-related

# 加载技能包
/memory:load-skill-memory
```

---

### 7.3. 工作流相关

**问题 5：不知道用什么命令？**

```
# 使用智能编排
/ccw-coordinator "你的需求"
```

---

**问题 6：命令执行失败？**

```
# 查看错误日志
cat .workflow/active/WFS-xxx/.process/error.log

# 重新执行
/workflow:execute
```

---

### 7.4. 多 CLI 相关

**问题 7：CLI 调用失败？**

```
# 检查 CLI 配置
ccw view  # Dashboard → Status → API Settings

# 测试 CLI 连接
ccw cli -p "test" --tool gemini
```

---

**问题 8：如何添加自定义 CLI？**

```
# 打开 Dashboard
ccw view

# 进入 API Settings
# 添加自定义 CLI
```

---

## 第八章. 本笔记总结

### 8.1. Issue 工作流速查

| 阶段 | 命令 | 说明 |
| --- | --- | --- |
| 积累 | /issue:discover | 自动发现问题 |
| 积累 | /issue:discover-by-prompt | 基于提示发现 |
| 积累 | /issue:new | 手动创建 Issue |
| 解决 | /issue:queue | 查看待处理队列 |
| 解决 | /issue:plan --all-pending | 批量规划 |
| 解决 | /issue:execute | 并行执行 |

---

### 8.2. Level 5 智能编排速查

**命令**：`/ccw-coordinator`

| 阶段 | 说明 | 产出 |
| --- | --- | --- |
| Phase 1 | 需求分析（任务类型 + 复杂度） | 分析结果 |
| Phase 2 | 命令发现与推荐（端口匹配） | 推荐命令链 |
| Phase 3 | 序列执行（串行阻塞） | 状态文件 |

**使用场景**：

- 不知道用什么命令
- 需要端到端自动化
- 需要完整状态追踪
- 需要可恢复会话

---

### 8.3. 多 CLI 协同速查

| 模式 | 提示词示例 | 说明 |
| --- | --- | --- |
| 协同分析 | “使用 Gemini 和 Codex 协同分析” | 交叉验证 |
| 并行执行 | “让 Gemini、Codex、Qwen 并行分析” | 多视角 |
| 流水线 | “Gemini 设计，Codex 实现，Claude 审查” | 顺序执行 |
| 迭代优化 | “用 Gemini 诊断，Codex 修复，迭代” | 循环优化 |

---

### 8.4. 生产环境最佳实践

**记忆管理**：

- 每次提交前：`/memory:update-related`
- 新项目初始化：`/memory:update-full`
- 定期维护：每周全量更新

**Session 管理**：

- 命名规范：`WFS-{feature-name}-{version}`
- 定期清理：归档已完成，删除失败

**工作流选择**：

- 需求明确 + 单模块 → `lite-plan`
- 需求明确 + 多模块 → `plan`
- Bug 修复 → `lite-fix`
- 不知道用什么 → `ccw-coordinator`

**代码审查**：

- 开发阶段：`/workflow:review-fix`
- 提交前：`/workflow:review`
- 专项审查：`--type security/performance/architecture`

**测试策略**：

- TDD 开发：`tdd-plan` → `execute` → `tdd-verify`
- 后置测试：`test-fix-gen` → `test-cycle-execute`

**Issue 管理**：

- 每天自动发现：`/issue:discover`
- 每周批量处理：`/issue:plan --all-pending` → `/issue:execute`

---

### 8.5. 下一步学习

本笔记覆盖了 Issue 工作流、Level 5 智能编排、多 CLI 协同原理和生产环境最佳实践。

**进阶内容**（第三篇笔记）：

- 完整命令速查手册
- 决策流程图
- 常见问题汇总
- 高级配置技巧

**推荐阅读顺序**：

1. 第一篇：主干工作流 + 日常场景：[【Claude Code工作流教程】Claude Code Workflow 实战指南：让多智能体 AI 军团为你写代码](https://linux.do/t/topic/1511026)
2. 第二篇：Issue 工作流 + 高级特性

---

**项目地址**：[GitHub - catlog22/Claude-Code-Workflow: JSON-driven multi-agent development framework with intelligent CLI orchestration (Gemini/Qwen/Codex), context-first architecture, and automated workflow execution](https://github.com/catlog22/Claude-Code-Workflow)
