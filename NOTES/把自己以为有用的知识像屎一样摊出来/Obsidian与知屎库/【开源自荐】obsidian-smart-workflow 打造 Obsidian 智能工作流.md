# 【开源自荐】obsidian-smart-workflow 打造 Obsidian 智能工作流 

[原帖链接](https://linux.do/t/topic/1398155/1)

**作者：长秋**  
**时间：Jan 2, 2026 3:36 am**  

开源地址：[GitHub - ZyphrZero/obsidian-smart-workflow: AI tools & cross-platform terminal for Obsidian.](https://github.com/ZyphrZero/obsidian-smart-workflow)

目前功能包含自动文件命名/标签生成/自动归档/本地终端/ASR语音/翻译/写作

**终端现在已经拆分到了ZyphrZero/Obsidian-Termy项目**

前端使用的TS，后端是Rust，WebSocket通信，性能优化的还行  

可以直接用BRAT插件下载，添加之后会自动下载Rust服务组件

适配了 Obsidian 1.11.0 版本的共享密钥api  

项目中为了方便开发，我专门构建了一套快速开发的工作流脚本，避免频繁手动 push 构建产物，`pnpm install:dev`就可以直接安装到obsidian插件目录

```
Obsidian 插件 (TypeScript)
    │
    │ WebSocket
    ▼
Smart Workflow Server (Rust)
├── PTY 终端会话
├── 音频录制 & ASR
├── LLM 流式处理
└── 语言检测
```

本地终端：

- 支持Windows/macOS/Linux跨平台终端

语音输入：

- 语音转录：说话直接转文字
- 同声转译：边说边翻译
- 语音润色：说完自动优化文本
- 支持实时流式转录

![QQ_1767352405995](https://cdn3.ldstatic.com/original/4X/5/9/4/5946063432dfa9b4b55db27d0c3041aba067babb.png)  

![image](https://cdn3.ldstatic.com/original/4X/4/c/9/4c9f984dd7b628fda659b154ab0ee629d58f3fe3.png)  

![image](https://cdn3.ldstatic.com/original/4X/d/f/c/dfc08da233abdd4d999109f4af9c7a84cb4a9b17.png)  

![QQ_1767352559918](https://cdn3.ldstatic.com/original/4X/f/5/c/f5c1e1a6c195000cf8de12203e3315a35376d8d3.png)  

![QQ_1767354209740](https://cdn3.ldstatic.com/original/4X/c/8/d/c8d0c57136b5761bc23ba42ff342f8b2184d6f66.png)  

![QQ_1767354472108](https://cdn3.ldstatic.com/original/4X/0/e/f/0ef023c06be1a06385867b93f482871631db74bc.png)
- 写作部分的设计参考了 [【开源自荐】YOLO——可能是目前最棒的 Obsidian AI 笔记插件？](https://linux.do/t/topic/1252853) 大佬的项目
- Voice参考了[【开源】按住说话-Windows平台语音输入转文本小工具（5MB）（qwen-asr-flash/doubao驱动，支持自定义润色）](https://linux.do/t/topic/1295391) 佬友的设计
