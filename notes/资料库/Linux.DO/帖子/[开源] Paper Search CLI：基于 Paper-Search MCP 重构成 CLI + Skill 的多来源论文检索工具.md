# [开源] Paper Search CLI：基于 Paper-Search MCP 重构成 CLI + Skill 的多来源论文检索工具 

[原帖链接](https://linux.do/t/topic/2186734/1)

**作者：饺子**  
**时间：May 15, 2026 11:15 pm**  

# [开源] Paper Search CLI：面向 Agent 的 Skill + CLI 学术文献工具

**本帖使用社区开源推广，符合推广要求。我申明并遵循社区要求的以下内容：**

- **我的帖子已经打上 [开源推广](https://linux.do/tag/2234-tag/2234) 标签：** 是
- **我的开源项目完整开源，无未开源部分：** 是
- **我的开源项目已链接认可 LINUX DO 社区：** 是
- **我帖子内的项目介绍，AI生成、润色内容部分已截图发出：** 是
- **以上选择我承诺是永久有效的，接受社区和佬友监督：** 是

_以下为项目介绍正文内容，AI生成、润色内容已使用截图方式发出_

---

**Paper Search CLI**

一个面向 AI Agent、终端用户和脚本的 Skill + CLI 学术文献工具。

  

      [github.com](https://github.com/dr-dumpling/paper-search-cli)
  

  
    
  ![](https://cdn3.ldstatic.com/optimized/4X/9/8/7/9872776022a891183fb559fd60136c9417a0e063_2_690x344.png)

  ### GitHub - dr-dumpling/paper-search-cli: Agent-friendly CLI for academic paper, journal...

    Agent-friendly CLI for academic paper, journal metric search and paper download

  

  
    
    
  

  

## 为什么做这个

之前一直在用 `paper-search-mcp` 论文检索 MCP。功能本身很完整，在没有接触到cc-switch之前，在多个Ai中切换过程后的MCP都需要单独配置，维护起来比较麻烦。  

前几天刷到站内佬友的建议（[https://linux.do/t/topic/2076559），](https://linux.do/t/topic/2076559%EF%BC%89%EF%BC%8C) 未来的主流可能会逐渐转变为**CLI + Skill**，所以我将原来佬友的项目，加上参考原始作者的来源项目，重新Vibe成了Cli + Skill：

文献检索功能来源主要参考这个项目：

  

      [github.com](https://github.com/openags/paper-search-mcp)
  

  
    
  ![](https://cdn3.ldstatic.com/optimized/4X/5/a/8/5a86d40a648e974aac9b86afb31d808a1a4198ef_2_690x344.png)

  ### GitHub - openags/paper-search-mcp: MCP, CLI, Skills for searching and downloading...

    MCP, CLI, Skills for searching and downloading academic papers from multiple sources like arXiv, PubMed, bioRxiv, etc.

  

  
    
    
  

  

## 核心特性

- **多来源学术检索 + 期刊指标**：覆盖 Crossref、OpenAlex、PubMed、PMC、Europe PMC、arXiv、bioRxiv、medRxiv、Semantic Scholar、CORE、OpenAIRE、DBLP、ACM Digital Library 元数据、USENIX 元数据、OpenReview、Web of Science、Google Scholar、IACR ePrint、Sci-Hub、IEEE Xplore、ScienceDirect、Springer Nature / SpringerLink、Wiley、Scopus、Unpaywall，并支持 EasyScholar 期刊指标。
- **四个主要工作流**：文献元数据检索、期刊指标检索、PDF 获取和下载、正文片段检索分开处理。
- **单一命令入口**：安装后通过 `paper-search` 调用，适合终端、脚本和 agent。
- **漏斗式回退下载链**：`download_with_fallback` 会优先尝试原生下载、PDF URL、PMC / Europe PMC / CORE / OpenAIRE、Unpaywall DOI 解析，最后使用 Sci-Hub 兜底；只有显式传入 `useSciHub=false` 才会关闭 Sci-Hub 兜底。
- **适合 agent 调用**：`doctor`、`smoke`、`skills`、`config`、`tools` 负责健康检查、Skill 同步、配置可见性和工具 schema；`search`、`journal-metrics`、`download`、`run` 负责实际学术任务。

**Paper Search CLI 目前重构并拆成下面四个能力**：

| 功能 | 主要入口 | 适合场景 | 返回内容 |
| --- | --- | --- | --- |
| 文献元数据检索 | paper-search search | 找论文、扩展关键词、做综述初筛、验证 DOI/PMID/PMCID/arXiv ID | 题名、作者、年份、期刊、DOI、PMID、URL、摘要和来源元数据 |
| 期刊指标检索 | paper-search journal-metrics | 查影响因子、JCR/SSCI、中科院分区、JCI、ESI、预警 | 期刊级指标，不是论文搜索 |
| PDF 获取和下载 | paper-search download | 已确认论文身份后获取 PDF | 原生来源、开放获取来源、权限来源和 Sci-Hub fallback 的下载结果 |
| 正文片段检索 | search_semantic_snippets | 查方法、参数、模型、统计写法等正文线索 | Semantic Scholar Open Access snippet |

## 技术路线

这个项目走的是 **CLI + Skill** 路线。

Skill 负责告诉 agent：什么时候该用 `paper-search`、该选哪个平台、什么时候查 DOI、什么时候查正文片段、什么时候下载 PDF。

CLI 负责真正执行：文献元数据检索、去重、期刊指标检索、PDF 获取和下载、健康检查、输出 JSON。

项目的调用框架如下：

  
  
      ​
    

    #mermaid-diagram-1{font-family:"trebuchet ms",verdana,arial,sans-serif;font-size:16px;fill:#ccc;}@keyframes edge-animation-frame{from{stroke-dashoffset:0;}}@keyframes dash{to{stroke-dashoffset:0;}}#mermaid-diagram-1 .edge-animation-slow{stroke-dasharray:9,5!important;stroke-dashoffset:900;animation:dash 50s linear infinite;stroke-linecap:round;}#mermaid-diagram-1 .edge-animation-fast{stroke-dasharray:9,5!important;stroke-dashoffset:900;animation:dash 20s linear infinite;stroke-linecap:round;}#mermaid-diagram-1 .error-icon{fill:#a44141;}#mermaid-diagram-1 .error-text{fill:#ddd;stroke:#ddd;}#mermaid-diagram-1 .edge-thickness-normal{stroke-width:1px;}#mermaid-diagram-1 .edge-thickness-thick{stroke-width:3.5px;}#mermaid-diagram-1 .edge-pattern-solid{stroke-dasharray:0;}#mermaid-diagram-1 .edge-thickness-invisible{stroke-width:0;fill:none;}#mermaid-diagram-1 .edge-pattern-dashed{stroke-dasharray:3;}#mermaid-diagram-1 .edge-pattern-dotted{stroke-dasharray:2;}#mermaid-diagram-1 .marker{fill:lightgrey;stroke:lightgrey;}#mermaid-diagram-1 .marker.cross{stroke:lightgrey;}#mermaid-diagram-1 svg{font-family:"trebuchet ms",verdana,arial,sans-serif;font-size:16px;}#mermaid-diagram-1 p{margin:0;}#mermaid-diagram-1 .label{font-family:"trebuchet ms",verdana,arial,sans-serif;color:#ccc;}#mermaid-diagram-1 .cluster-label text{fill:#F9FFFE;}#mermaid-diagram-1 .cluster-label span{color:#F9FFFE;}#mermaid-diagram-1 .cluster-label span p{background-color:transparent;}#mermaid-diagram-1 .label text,#mermaid-diagram-1 span{fill:#ccc;color:#ccc;}#mermaid-diagram-1 .node rect,#mermaid-diagram-1 .node circle,#mermaid-diagram-1 .node ellipse,#mermaid-diagram-1 .node polygon,#mermaid-diagram-1 .node path{fill:#1f2020;stroke:#ccc;stroke-width:1px;}#mermaid-diagram-1 .rough-node .label text,#mermaid-diagram-1 .node .label text,#mermaid-diagram-1 .image-shape .label,#mermaid-diagram-1 .icon-shape .label{text-anchor:middle;}#mermaid-diagram-1 .node .katex path{fill:#000;stroke:#000;stroke-width:1px;}#mermaid-diagram-1 .rough-node .label,#mermaid-diagram-1 .node .label,#mermaid-diagram-1 .image-shape .label,#mermaid-diagram-1 .icon-shape .label{text-align:center;}#mermaid-diagram-1 .node.clickable{cursor:pointer;}#mermaid-diagram-1 .root .anchor path{fill:lightgrey!important;stroke-width:0;stroke:lightgrey;}#mermaid-diagram-1 .arrowheadPath{fill:lightgrey;}#mermaid-diagram-1 .edgePath .path{stroke:lightgrey;stroke-width:1px;}#mermaid-diagram-1 .flowchart-link{stroke:lightgrey;fill:none;}#mermaid-diagram-1 .edgeLabel{background-color:hsl(0, 0%, 34.4117647059%);text-align:center;}#mermaid-diagram-1 .edgeLabel p{background-color:hsl(0, 0%, 34.4117647059%);}#mermaid-diagram-1 .edgeLabel rect{opacity:0.5;background-color:hsl(0, 0%, 34.4117647059%);fill:hsl(0, 0%, 34.4117647059%);}#mermaid-diagram-1 .labelBkg{background-color:rgba(87.75, 87.75, 87.75, 0.5);}#mermaid-diagram-1 .cluster rect{fill:hsl(180, 1.5873015873%, 28.3529411765%);stroke:rgba(255, 255, 255, 0.25);stroke-width:1px;}#mermaid-diagram-1 .cluster text{fill:#F9FFFE;}#mermaid-diagram-1 .cluster span{color:#F9FFFE;}#mermaid-diagram-1 div.mermaidTooltip{position:absolute;text-align:center;max-width:200px;padding:2px;font-family:"trebuchet ms",verdana,arial,sans-serif;font-size:12px;background:hsl(20, 1.5873015873%, 12.3529411765%);border:1px solid rgba(255, 255, 255, 0.25);border-radius:2px;pointer-events:none;z-index:100;}#mermaid-diagram-1 .flowchartTitleText{text-anchor:middle;font-size:18px;fill:#ccc;}#mermaid-diagram-1 rect.text{fill:none;stroke-width:0;}#mermaid-diagram-1 .icon-shape,#mermaid-diagram-1 .image-shape{background-color:hsl(0, 0%, 34.4117647059%);text-align:center;}#mermaid-diagram-1 .icon-shape p,#mermaid-diagram-1 .image-shape p{background-color:hsl(0, 0%, 34.4117647059%);padding:2px;}#mermaid-diagram-1 .icon-shape .label rect,#mermaid-diagram-1 .image-shape .label rect{opacity:0.5;background-color:hsl(0, 0%, 34.4117647059%);fill:hsl(0, 0%, 34.4117647059%);}#mermaid-diagram-1 .label-icon{display:inline-block;height:1em;overflow:visible;vertical-align:-0.125em;}#mermaid-diagram-1 .node .label-icon path{fill:currentColor;stroke:revert;stroke-width:revert;}#mermaid-diagram-1 .node .neo-node{stroke:#ccc;}#mermaid-diagram-1 [data-look="neo"].node rect,#mermaid-diagram-1 [data-look="neo"].cluster rect,#mermaid-diagram-1 [data-look="neo"].node polygon{stroke:url(#mermaid-diagram-1-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#mermaid-diagram-1 [data-look="neo"].node path{stroke:url(#mermaid-diagram-1-gradient);stroke-width:1px;}#mermaid-diagram-1 [data-look="neo"].node .outer-path{filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#mermaid-diagram-1 [data-look="neo"].node .neo-line path{stroke:#ccc;filter:none;}#mermaid-diagram-1 [data-look="neo"].node circle{stroke:url(#mermaid-diagram-1-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#mermaid-diagram-1 [data-look="neo"].node circle .state-start{fill:#000000;}#mermaid-diagram-1 [data-look="neo"].icon-shape .icon{fill:url(#mermaid-diagram-1-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#mermaid-diagram-1 [data-look="neo"].icon-shape .icon-neo path{stroke:url(#mermaid-diagram-1-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#mermaid-diagram-1 :root{--mermaid-font-family:"trebuchet ms",verdana,arial,sans-serif;}用户问题 / agent 任务

paper-search Skill

Capability Profile

metadata_search 文献元数据检索

journal_metrics 期刊指标检索

download PDF获取和下载

search_semantic_snippets 正文片段检索

Management Layer 管理命令

Crossref / OpenAlex / PubMed / arXiv / Semantic Scholar 等

EasyScholar

原生来源 / OA 来源 / 权限来源 / Sci-Hub final fallback

Semantic Scholar OA snippets

doctor / smoke / skills / config / tools

统一 JSON 输出

agent / 脚本 / 终端继续处理

## 支持平台

| 类型 | 平台 | 适合场景 |
| --- | --- | --- |
| 综合检索 | Crossref、OpenAlex、Semantic Scholar、Google Scholar | 广覆盖发现、DOI 元数据、引用线索、文献初筛 |
| 期刊指标 | EasyScholar | 影响因子、JCR/SSCI 分区、中科院分区、JCI、ESI、预警 |
| 医学/生命科学 | PubMed、PubMed Central、Europe PMC | 临床、生物医学、公卫、生物医学元数据和开放全文 |
| 预印本/会议稿 | arXiv、bioRxiv、medRxiv、OpenReview、IACR ePrint | 跨学科预印本、生命科学/医学预印本、AI/ML 投稿和密码学 ePrint |
| 计算机/工程 | DBLP、ACM Digital Library 元数据、IEEE Xplore、USENIX | CS 文献目录、工程数据库、系统/安全会议论文 |
| 开放全文/仓储 | CORE、OpenAIRE、Unpaywall | 跨学科仓储发现和开放获取 PDF 回退路径 |
| 引文库/出版商 | Web of Science、Scopus、ScienceDirect、Springer Nature/SpringerLink、Wiley | 机构权限型元数据、引文数据库、出版商记录和下载 |
| DOI 定向获取 | Sci-Hub | DOI 定向获取，并作为 PDF 下载漏斗的最后自动兜底；除非传入 useSciHub=false |

下面把完整能力矩阵按类型拆成小表，判断各平台的功能和是否需要配API key。

### 综合检索

| 平台 | 搜索 | 下载 | 全文 | 被引统计 | API Key | 特色功能 |
| --- | --- | --- | --- | --- | --- | --- |
| Crossref |  |  |  |  |  | 默认搜索平台，广泛元数据覆盖 |
| OpenAlex |  | 条件支持 |  |  |  | 广泛免费元数据；记录含开放链接时可用于回退下载 |
| Semantic Scholar |  |  | 正文片段 |  | 可选* | AI 语义检索 + OA 正文片段 |
| Google Scholar |  |  |  |  |  | 广泛学术发现，基于页面解析 |

### 期刊指标

| 平台 | 搜索 | 下载 | 全文 | 被引统计 | API Key | 特色功能 |
| --- | --- | --- | --- | --- | --- | --- |
| EasyScholar | 论文关键词搜索 |  |  |  | 必需 | 影响因子、JCR/SSCI 分区、中科院分区、JCI、ESI、预警等 |

### 医学/生命科学

| 平台 | 搜索 | 下载 | 全文 | 被引统计 | API Key | 特色功能 |
| --- | --- | --- | --- | --- | --- | --- |
| PubMed |  |  |  |  | 可选 | NCBI E-utilities 生物医学文献 |
| PubMed Central |  |  |  |  |  | 生物医学开放全文和 PMC PDF |
| Europe PMC |  |  |  |  |  | 生物医学元数据和开放全文链接 |

### 计算机/工程

| 平台 | 搜索 | 下载 | 全文 | 被引统计 | API Key | 特色功能 |
| --- | --- | --- | --- | --- | --- | --- |
| DBLP |  |  |  |  |  | 官方 DBLP search API，计算机文献目录 |
| ACM Digital Library 元数据 |  |  |  |  |  | 通过 Crossref 的 ACM DOI 前缀检索元数据；不抓取 ACM 页面 |
| USENIX 元数据 |  |  |  |  |  | 基于 DBLP 的 USENIX 会议元数据；不抓取 USENIX 搜索页 |
| IEEE Xplore |  |  |  |  | 必需 | 官方 IEEE Xplore Metadata API，需要 IEEE_API_KEY |

### 开放全文/仓储

| 平台 | 搜索 | 下载 | 全文 | 被引统计 | API Key | 特色功能 |
| --- | --- | --- | --- | --- | --- | --- |
| CORE |  | 条件支持 | 条件支持 |  | 可选 | 记录含 PDF 或全文链接时可下载 |
| OpenAIRE |  | 条件支持 |  |  | 可选 | 记录含开放链接时可用于回退下载 |
| Unpaywall | 条件支持 | 条件支持 |  |  | 邮箱 | 仅支持 DOI 查询；需要 email；发现 OA PDF 时可下载 |

### 预印本/会议稿

| 平台 | 搜索 | 下载 | 全文 | 被引统计 | API Key | 特色功能 |
| --- | --- | --- | --- | --- | --- | --- |
| arXiv |  |  |  |  |  | 物理、计算机、数学等预印本 |
| bioRxiv |  |  |  |  |  | 生物学预印本 |
| medRxiv |  |  |  |  |  | 医学预印本 |
| OpenReview |  |  |  |  |  | 公开 OpenReview notes search，适合 AI/ML 投稿、评审和预印本 |
| IACR ePrint |  |  |  |  |  | 密码学论文 |

### 引文库/出版商

| 平台 | 搜索 | 下载 | 全文 | 被引统计 | API Key | 特色功能 |
| --- | --- | --- | --- | --- | --- | --- |
| Web of Science |  |  |  |  | 必需 | 引文数据库、日期排序、年份范围 |
| ScienceDirect |  |  |  |  | 必需 | Elsevier 元数据和摘要 |
| Springer Nature / SpringerLink |  | 条件支持 |  |  | 必需 | springerlink 是 springer 的别名；开放获取记录可下载 |
| Wiley | 关键词搜索 |  |  |  | 必需 | TDM API，仅支持 DOI 下载 PDF |
| Scopus |  |  |  |  | 必需 | 摘要和引文数据库 |

### DOI 定向获取

| 平台 | 搜索 | 下载 | 全文 | 被引统计 | API Key | 特色功能 |
| --- | --- | --- | --- | --- | --- | --- |
| Sci-Hub |  |  |  |  |  | DOI/URL 定向 PDF fallback；不是文献元数据检索来源 |

## 常用命令

CLI 的输出设计是 **agent-friendly**，优先面向 AI、agent 和脚本解析。默认输出结构化 JSON，完整平台和命令可以看 GitHub 上面。

**安装：**

```
npm install -g paper-search-cli
paper-search setup
```

**搜索论文：可以同时多个来源进行检索**

```
paper-search search "machine learning applications" --platform all --max-results 5 --pretty
```

**检索期刊指标：**

用于查询期刊影响因子、JCR / SSCI 分区、中科院分区、JCI、ESI、预警和等级字段，需要配置 `EASYSCHOLAR_KEY`。

```
paper-search journal-metrics "Nature" "BMJ" --pretty

```

**PDF 下载：**

OA期刊、出版商直连/API、机构访问、CloakBrowser出版商捕获，Sci-Hub兜底。

```
# arXiv 这类有原生 PDF 的来源
paper-search download 2301.12345 --platform arxiv --save-path ./downloads --pretty

# DOI 论文可走统一 PDF discovery fallback
paper-search run download_with_fallback \
  --json-args '{"source":"crossref","paperId":"10.xxxx/xxxxx","doi":"10.xxxx/xxxxx","title":"Paper title","savePath":"./downloads"}' \
  --pretty
```

**正文片段检索：**

用于查论文正文里可能出现的方法描述、参数、模型名、软件名、统计写法等，需要配置`SEMANTIC_SCHOLAR_API_KEY`。

```
paper-search run search_semantic_snippets \
  --arg query="statistical model validation methods" \
  --arg limit=5 \
  --arg fieldsOfStudy=Computer Science \
  --pretty
```

**查看工具列表：**

```
paper-search tools --pretty
```

**查看健康报告和 Capability Profile：**

```
paper-search doctor --pretty
paper-search doctor --format text
```

**查看配置状态和本地自检：**

```
paper-search config list --pretty
paper-search smoke --mock --pretty
```

**同步随包发布的 Skill：**

```
paper-search skills status --pretty
paper-search skills update --targets agents --pretty
```

## API key 说明

不配置 key 也能用像 Crossref、OpenAlex、arXiv、PubMed、DBLP、ACM、USENIX、OpenReview 这些来源的信息，但是还是建议先配置后体验更佳：

| 配置 | 作用 |
| --- | --- |
| SEMANTIC_SCHOLAR_API_KEY | 正文片段检索、提升 Semantic Scholar 稳定性 |
| CORE_API_KEY | CORE 开放仓储检索，减少匿名限流 |
| UNPAYWALL_EMAIL | DOI 开放获取 PDF 解析 |
| CROSSREF_MAILTO | Crossref polite pool |
| EASYSCHOLAR_KEY | EasyScholar 期刊指标检索 |
| IEEE_API_KEY | IEEE Xplore 元数据检索 |

申请入口：

| 配置 | 入口 | 说明 |
| --- | --- | --- |
| EASYSCHOLAR_KEY | EasyScholar Open API | 期刊指标检索需要它。 |
| SEMANTIC_SCHOLAR_API_KEY | Semantic Scholar API | 正文片段检索需要它，好用，用edu邮箱注册申请都可以通过。 |
| CORE_API_KEY | CORE API | 建议配置，匿名访问容易限流。 |
| UNPAYWALL_EMAIL | Unpaywall API | 不需要 API key，只需要邮箱；setup 可自动生成随机邮箱，也可手动填写自己的邮箱。 |
| CROSSREF_MAILTO | Crossref REST API | 不需要 API key，只需要邮箱；setup 可自动生成随机邮箱，也可手动填写自己的邮箱。 |
| PUBMED_API_KEY | NCBI API Keys | 高频使用 PubMed 时再配，一般不需要。 |
| WOS_API_KEY | Clarivate Developer Portal | 需要 Web of Science API 权限。 |
| IEEE_API_KEY | IEEE Xplore Metadata API | IEEE Xplore 元数据检索需要它；通常需要 IEEE API 权限。 |
| ELSEVIER_API_KEY | Elsevier Developer Portal | Scopus / ScienceDirect 需要有权限。 |
| SPRINGER_API_KEY | Springer Nature Developers | Springer 检索和开放获取接口使用。 |
| SPRINGER_OPENACCESS_API_KEY | Springer Nature Developers | Springer OpenAccess 接口使用，和出版商权限型 API 分开。 |
| WILEY_TDM_TOKEN | Wiley Text and Data Mining | Wiley TDM 下载需要对应权限。 |
| OPENAIRE_API_KEY | OpenAIRE APIs | 通常可选，公开检索一般不需要。 |

配置指定平台运行：

```
paper-search setup xx
```

配置全部平台运行：

```
paper-search setup --all
```

## Agent Skill

Skill 用来告诉 agent如何调用 `paper-search`，安装时候会弹出配置Skill的路径：

```
skills/paper-search/SKILL.md
```

后续安装或同步 Skill可以用下面的命令：

```
paper-search setup --install-skills agents
paper-search skills status --pretty
paper-search skills update --targets agents --pretty
```

最后，希望有需要的佬友可以帮我点个star，多多提意见。

后续更新任务：通过 WebVPN/CARSI 下载 学校有权限的PDF。有点麻烦，还在调试。
