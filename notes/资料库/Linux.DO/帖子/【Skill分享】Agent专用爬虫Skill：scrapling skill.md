# 【Skill分享】Agent专用爬虫Skill：scrapling skill 

[原帖链接](https://linux.do/t/topic/1732345/1)

**作者：cedric chen**  
**时间：Mar 11, 2026 4:13 am**  

GitHub:

  

      [github.com](https://github.com/Cedriccmh/claude-code-skill-scrapling)
  

  
    
  ![](//cdn3.ldstatic.com/optimized/4X/1/9/f/19f688517c9ddecf58fdf94b8c0ea9c3e3df8a32_2_690x344.png)

  ### GitHub - Cedriccmh/claude-code-skill-scrapling: Claude Code skill for web scraping with scrapling...

    Claude Code skill for web scraping with scrapling - auto Fetcher selection, Cloudflare bypass, site patterns

  

  
    
    
  

  

## 背景

用 Claude Code 抓网页，每次都得跟它解释一遍：这个站有 Cloudflare 你得用 StealthyFetcher、cookie 格式浏览器端和 HTTP 端不一样……

[scrapling](https://github.com/D4Vinci/Scrapling) 本身已经解决了很多爬虫的硬问题——自动 TLS 指纹、Camoufox 过 Cloudflare、Playwright 渲染 SPA。但它只是个 Python 库，Claude 拿过来还是会写出各种有问题的代码。缺的不是能力，是**怎么正确地用**这部分知识。

所以做了这个 skill，把 scrapling 的最佳实践喂给 Claude Code，让它自己判断用哪个 Fetcher、怎么传 cookie、怎么处理反爬。

## 它干了什么

一个 Fetcher 决策树，Claude 拿到 URL 后进行判断：

```
静态页面，无反爬？ → Fetcher（curl_cffi，最快）
需要登录？         → FetcherSession（自动保持会话）
有 Cloudflare？    → StealthyFetcher（Camoufox，自动过 CF）
SPA / JS 渲染？    → DynamicFetcher（Playwright）
已有 HTML？        → Selector（纯解析，不发请求）
不确定？           → 先 Fetcher 试，403 就升级
```

除了决策树，还有几个比较实用的东西：

- **4 个模板脚本**，覆盖静态抓取、Cloudflare 绕过、登录会话、纯解析，Claude 可以直接套用不用从头写
- **站点模式库**（site-patterns.md），抓过的站点记录下来，下次直接复用，不重复踩坑
- **cookie 管理**，区分了 HTTP Fetcher 用 dict、浏览器 Fetcher 用 list[dict] 这种细节
- **故障排查索引**，避免重复踩坑

## 这个 skill 做了什么不一样的事

**代码式调用，不是 CLI 也不是 REPL**。skill 里所有模板都是完整的 Python 脚本，Claude 读模板、替换参数、直接 `python` 执行拿结果。不需要交互式操作，不需要人在旁边看着。整个流程是：

```
读 site-patterns → 匹配？复用 → 没匹配？走决策树 → 套模板生成 .py → 执行 → 存 pattern
```

全程代码驱动，没有需要人介入的环节。

**经验可沉淀复用**。抓过的站点写进 site-patterns.md，下次再抓同类站点直接复用，不重复踩坑。这个对 agent 很关键——没有 skill 的话，Claude 每个 session 都在重新发现「这个站点需要带 cookie」「那个站点超时要设长一点」。

## 安装

```
pip install "scrapling[fetchers]"
scrapling install
```

然后把 skill 复制到 Claude Code 的 skills 目录：

```
cp -r . ~/.claude/skills/scrapling
```

装完之后直接跟 Claude Code 说「帮我抓 xxx 网页」就行，它会自己选 Fetcher、生成脚本、跑起来。

## 已知限制

- StealthyFetcher 走的是真实浏览器，比 Fetcher 慢很多（秒级 vs 毫秒级）
- 需要登录的站点 cookie 得自己从浏览器 DevTools 里拿，没法自动获取
- `cf_clearance` cookie 绑浏览器指纹，不能跨客户端复用——别手动传，让 StealthyFetcher 自己过

## 目录结构

```
├── SKILL.md                  # 入口，决策树 + 工作流
├── references/
│   ├── api-quick-ref.md      # Scrapling API 速查
│   ├── cookie-vault.md       # Cookie 模板
│   ├── maintenance.md        # 安装/升级
│   ├── site-patterns.md      # 站点模式库
│   └── troubleshooting.md    # 故障排查
└── templates/
    ├── basic_fetch.py        # 静态抓取
    ├── stealth_cloudflare.py # CF 绕过
    ├── session_login.py      # 登录会话
    └── parse_only.py         # 纯解析
```
