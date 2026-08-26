# 【AI编程】拒绝上下文过载：如何让Claude Code学会“渐进式阅读” 

[原帖链接](https://linux.do/t/topic/1501356/1)

**作者：cedric chen**  
**时间：Jan 22, 2026 1:41 am**  

被举报AIGC了，hook脚本到github取吧

  

      [github.com](https://github.com/Cedriccmh/cc-read-limit-hook)
  

  
    
  ![](https://cdn3.ldstatic.com/optimized/4X/8/3/a/83a87de3fe85c1fcd436f2a4b56fd84e4ce36718_2_690x344.png)

  ### GitHub - Cedriccmh/cc-read-limit-hook: A hook that limit and guide how claude code read...

    A hook that limit and guide how claude code read files.

  

  
    
    
  

  

---

时间过得真快，距离上次发话题已经过去几个月，成年人的时间真是不经用。马上过年了，想罢年前一定要发点东西出来的。预祝大家新年快乐。

---

场景是这样的：  

当claude code读取代码时，往往倾向于读取整个文件，如果文件非常大（比如5000行+），这样的文本塞进上下文。结果就是：

![image](https://cdn3.ldstatic.com/original/4X/6/0/d/60dca72bdc68777e36f20661073118d1f1d6fd1a.png)

## 什么是“渐进式披露”？

举个例子，这就好比一个人类程序员接手新项目时，不会上来就把 10 万行代码从头到尾读一遍。你会先看目录结构（`ls`），再搜关键字（`grep`），最后只打开相关的那几十行代码（`read`）。

Anthropic 的文档里一直强调这一点：**让模型先通过搜索定位，再通过切片读取。**

但在实际的行时，claude code执“太勤快”，往往是直接 Read 整个文件。所以，我们需要给它装一个“防呆开关”。

## 这个 Hook 是怎么工作的？

这是一个Python 脚本，在 `PreToolUse` 时的 Hook（工具调用前拦截），配合 `CLAUDE.md` 的提示词，组合引导claude code读取精确上下文。

### 核心逻辑

这个方案由两部分组成：

![image](https://cdn3.ldstatic.com/original/4X/1/1/0/11030dcc85b3042ca1910d90af63b4032c9396a9.png)
### 为什么这个方法 Work？

这利用了 LLM 的一个特性：**它们非常听“报错信息”的话。**

当 Tool Use 失败并返回一个明确的“推荐路径”时，claude code会立刻进行自我修正。

![image](https://cdn3.ldstatic.com/original/4X/c/7/5/c753f2d36cf658d54b4337335e2775bef846f0ec.png)

这样就强行把它拽回了“渐进式披露”的最佳实践路径上。

## 如何食用

你需要两个东西：一个是配置在项目根目录的规则文件，一个是实际执行拦截的 Python 脚本。

### 1. 提示词 (CLAUDE.md)

把这段加到你的项目提示词文件(`CLAUDE.md \ AGENTS.md`)中。告诉claude code读取策略。

#### 中文版本

```
### 文件读取策略

**强制规则**：每次调用 Read 工具时**必须**指定 `offset` 和 `limit` 参数，禁止使用默认值。

#### 参数要求

| 参数   | 要求           | 说明                          |
| ------ | -------------- | ----------------------------- |
| `offset` | **必须指定** | 起始行号（从 0 开始）         |
| `limit`  | **必须指定** | 读取行数，单次不超过 500 行   |

#### 读取流程

1. **侦察**：先用 Grep 了解文件结构，或定位目标关键词行号。
2. **精准打击**：使用 offset + limit 精确读取目标区域。
3. **扩展**：如果需要更多上下文，再调整 offset 继续读取。

**目标**：保持上下文精准、最小化。如果不遵守，工具调用将被 Hook 拦截。

```

#### English Version

```
### File Reading Strategy

**MANDATORY RULE**: Every `Read` tool call **MUST** verify `offset` and `limit` parameters. Default full-file reads are prohibited for non-trivial files.

#### Parameter Requirements

| Param    | Requirement    | Description                   |
| -------- | -------------- | ----------------------------- |
| `offset` | **REQUIRED** | Start line number (0-indexed) |
| `limit`  | **REQUIRED** | Max lines to read (Max 500)   |

#### Workflow

1. **Recon**: Use `Grep` first to understand structure or locate keywords.
2. **Surgical Read**: Use `offset` + `limit` to read only the relevant section.
3. **Expand**: Adjust `offset` to read more context only if strictly necessary.

**Goal**: Keep context precise and minimal. Violations will be blocked by the PreToolUse hook.

```

### 2. Hook (Python 脚本)

从上面github仓库获取hook文件，并配置到你的claude code（如果不熟悉可以直接把文件丢给claude code让他代劳）。

_(这个脚本稍微有点长，但逻辑很简单：检查文件大小 → 检查参数 → 决定是放行、自动修正还是报错拦截)_

## 效果

装上这一套之后，你会发现 claude code 的行为模式变了：

![image](https://cdn3.ldstatic.com/original/4X/7/f/3/7f30525269d93fd6d556b367d3405da0f8e3a99d.png)

虽然多了一步交互，但**上下文极其干净**，Token 消耗量大大降低，而且修改的准确率反而提高了。
