# 【开源自荐】oh-my-paper，规范化科研工作流，claudecode内插件增强cc能力，cc与codex协作科研 

[原帖链接](https://linux.do/t/topic/1863510/1)

**作者：donk666**  
**时间：Mar 31, 2026 1:41 am**  

#### 本帖使用社区开源推广，符合推广要求。我申明并遵循社区要求的以下内容：

- **我的帖子已经打上 [开源推广](https://linux.do/tag/2234-tag/2234) 标签：** 是
- **我的开源项目完整开源，无未开源部分：** 是
- **我的开源项目已链接认可 LINUX DO 社区：** 是
- **我帖子内的项目介绍，AI生成、润色内容部分已截图发出：** 是
- **以上选择我承诺是永久有效的，接受社区和佬友监督：** 是

_以下为项目介绍正文内容，AI生成、润色内容已使用截图方式发出_

---

  

      [github.com](https://github.com/LigphiDonk/Oh-my--paper)
  

  
    
  ![](https://cdn3.ldstatic.com/optimized/4X/1/5/7/15749e83643e48b7f49b88078a64390a212251ba_2_690x344.png)

  ### GitHub - LigphiDonk/Oh-my--paper: auto...

    auto research的入口，claudecode插件，规范agent科研流程，无感接入科研工作流，科研harness。

  

  
    
    
  

  

## 为什么做 Oh My Paper？

科研工作流是碎片化的：从文献调研，创新点的选择，到实验进行，最终论文撰写，这是一个复杂宏大的过程，单纯的claudecode 可能会出现记忆混乱，流程不清晰，以及无法在正确的位置使用恰当的skill等问题

**Oh My Paper 是自主科研的harness。**  

它把从文献调研到最终发表的整个生命周期整合进一个理解项目状态的harness中 在恰当的地点触发正确的流程化pipeline

---

## 核心亮点

### AI Agent Harness — 多角色协作的自主科研系统 集成在claudecode中，一句命令即可安装，内置5种专属科研agent，40个精选科研skill，和分层记忆架构

**这是 Oh My Paper 的核心创新。** 不是简单的 AI 对话，而是一套完整的 Agent  

编排框架：

#### 五个专业 Agent 角色

打开 Claude Code 后，系统自动检测项目状态，弹出角色选择器：

| Agent 角色 | 职责 | 读取的记忆文件 |
| --- | --- | --- |
| Conductor（统筹者） | 全局规划、评审产出、派遣任务 | project_truth +orchestrator_state + tasks.json + `review_log`` |
| Literature Scout（文献侦察兵） | 搜索论文、整理文献库 | project_truth + execution_context + paper_bank.json |
| Experiment Driver（实验驾驶员） | 设计实验、编写代码、运行评估 | execution_context + experiment_ledger+research_brief.json |
| Paper Writer（论文作者） | 撰写章节、生成图表、审查引用 | execution_context + result_summary + paper_bank.json |
| Reviewer（评审者） | 同行评审、质量门控 | execution_context +project_truth + result_summary |

#### 工作流程

用户选择角色 → Agent 读取对应记忆 → 以该身份工作 → 更新共享状态  

↓  

写入 tasks.json / paper_bank.json  

↓

一句命令就将这套hardness安装进自己的claudecode，更加轻便可靠

![截屏2026-03-31 14.39.49](https://cdn3.ldstatic.com/original/4X/2/a/9/2a9d35dfa7e3c100d27633b1e876a08afbfb8586.jpeg)  

欢迎体验，重塑科研流程，求佬友的star
