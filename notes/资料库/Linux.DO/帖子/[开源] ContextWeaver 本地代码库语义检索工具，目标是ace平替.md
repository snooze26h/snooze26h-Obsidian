# [开源] ContextWeaver 本地代码库语义检索工具，目标是ace平替 

[原帖链接](https://linux.do/t/topic/1373618/1)

**作者：刘作老虎**  
**时间：Dec 28, 2025 10:12 pm**  

项目链接：[GitHub - hsingjui/ContextWeaver: ContextWeaver 是一个基于 MCP 协议、利用 Tree-sitter 和向量搜索为大语言模型提供本地代码库智能上下文编织与检索的工具。](https://github.com/hsingjui/ContextWeaver)

`augment code`的账号太难注册了，注册了就封，从海鲜市场买别人注册好的号也是几天就封了，`augment context engine`又很好用，只能试试看能不能做一个平替了

项目使用方式

1. 安装

```
# 全局安装
npm install -g @hsingjui/contextweaver

# 或使用 pnpm
pnpm add -g @hsingjui/contextweaver
```

1. 配置

```
# 初始化
contextweaver init
# 或者简写
cw init

#修改 `~/.contextweaver/.env`

EMBEDDINGS_API_KEY=your-api-key-here
EMBEDDINGS_BASE_URL=https://api.siliconflow.cn/v1/embeddings
EMBEDDINGS_MODEL=BAAI/bge-m3
EMBEDDINGS_MAX_CONCURRENCY=10
EMBEDDINGS_DIMENSIONS=1024

RERANK_API_KEY=your-api-key-here
RERANK_BASE_URL=https://api.siliconflow.cn/v1/rerank
RERANK_MODEL=BAAI/bge-reranker-v2-m3
RERANK_TOP_N=20
```

这里`Embedding`和`Reranker`模型用的硅基流动免费的模型,用`Qwen/Qwen3-Embedding-8B`和`Qwen/Qwen3-Reranker-8B`，效果好一些，但是速度会慢一点

1. 索引代码库

```
#这一步不是必须的，使用mcp搜索的时候，如果没有索引代码库会自动索引
# 在代码库根目录执行
contextweaver index

# 指定路径
contextweaver index /path/to/your/project

# 强制重新索引
contextweaver index --force
```

1. 使用mcp

```
{
  "mcpServers": {
    "contextweaver": {
      "command": "contextweaver",
      "args": ["mcp"]
    }
  }
}
```

`claude code`使用

```
claude mcp add contextweaver -- command contextweaver mcp
```

这是项目的工作流程

![16B06869-EB7F-42D8-B7F1-329722D4C821](https://cdn3.ldstatic.com/original/4X/0/d/4/0d414039b364de00b80f2c18ce8c4d6991de93b7.png)
下面是图一乐的用`claude code`在`new-api`的仓库对比`ace`和`ContextWeaver`的结果，仅供参考  

使用的prompt

```
任务：对比 Ace 和 ContextWeaver 在当前项目中的 Codebase Retrieval (代码库检索) 效果。
请执行以下步骤进行 A/B 测试：
设定三个测试问题：使用问题 "[请在此处填入具体的复杂技术问题，例如：如何修改鉴权逻辑以支持JWT？]" 作为基准。
分别检索：
场景 A (Ace)：调用 Ace 的检索能力，列出其提取的关键文件和代码片段。
场景 B (ContextWeaver)：调用 ContextWeaver 的检索能力，列出其提取的关键文件和代码片段。
对比分析：请基于以下维度创建一个对比表格：
相关性 (Precision)：检索到的文件是否直接解决了问题？是否有核心文件遗漏？
噪音干扰 (Noise)：是否包含了大量无关的测试文件或通用配置？
上下文完整度 (Context)：是否提供了足够的上下文（如引用链路、类型定义）来理解代码？
结论：基于当前项目的代码结构，通过三个测试问题，指出哪一个工具的检索策略更优。
```

![4EA639AD-A788-4172-87AE-BFA66263970B](https://cdn3.ldstatic.com/original/4X/6/c/c/6ccf604f0e4918c3d742651adb26588fd120c741.png)
