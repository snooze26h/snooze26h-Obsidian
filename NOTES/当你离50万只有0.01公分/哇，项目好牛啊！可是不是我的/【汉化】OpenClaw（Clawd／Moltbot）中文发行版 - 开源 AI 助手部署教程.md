# 【汉化】OpenClaw（Clawd/Moltbot）中文发行版 - 开源 AI 助手部署教程 

[原帖链接](https://linux.do/t/topic/1548258/1)

**作者：晴天**  
**时间：Jan 30, 2026 10:59 pm**  

各位佬友好，分享一个开源 AI 助手项目的中文版部署教程。
> 之前发过一帖被删了，回去认真看了下社区规则才发现是违反了发帖规则。这次重新整理成技术教程形式，希望对有需要的佬友有帮助。

> 注：官方上游已经开始支持多语言模块，但是汉化范围有限，如终端命令行窗口，等一些细节部分尚未汉化，此项目则作为**汉化补充**，和**OpenClaw查错合集**，使用原版或汉化版的用户，可以在安装完成后，在 **Overview**页面配置处，设置语言即可看到中文页面！


## 这是什么？

[OpenClaw](https://github.com/openclaw/openclaw)（曾用名 Clawd / Moltbot）是一个开源的个人 AI 助手平台（GitHub 100k+ Stars），可以通过 WhatsApp、Telegram、Discord 等聊天软件与 AI 交互。简单说就是：**在你自己的机器上运行一个 AI 助手，通过常用聊天软件跟它对话**。

原版是全英文的，我们做了一个**中文发行版**：

| 特点 | 说明 |
| --- | --- |
| 开箱即用 | npm 一键安装 / Docker 一键部署，不需要手动打补丁 |
| 实时同步 | 每小时自动从官方仓库拉取最新代码并构建 |
| 双版本 | stable（稳定版）和 nightly（最新版）可选 |
| 深度汉化 | CLI + Dashboard 全中文界面 |

**项目地址**：[GitHub - 1186258278/OpenClawChineseTranslation: 🦞 OpenClaw (Clawdbot/Moltbot) 汉化版 - 开源个人 AI 助手中文版 | Claude/ChatGPT LLM 接入 | WhatsApp/Telegram/Discord 多平台 | 每小时自动同步 | CLI + Dashboard 全中文 | 全流程搭建教程，以及排错指南！](https://github.com/1186258278/OpenClawChineseTranslation)

![image](https://cdn3.ldstatic.com/original/4X/a/b/d/abd3fc599d8a9010a1683a55679edad570607717.png)> 由于项目文档比较长，特意做了一个快速导航，有需要的可以前往项目地址查看，里面的常见问题板块，我觉得是最有帮助的！里面为会不定时更新和补充内容！


---

## 汉化效果预览

先看看效果，Dashboard 界面全中文：

**概览仪表板** - 网关状态、实例监控、快捷操作一目了然：

![概览仪表板](https://cdn3.ldstatic.com/original/4X/e/9/1/e917b2250502c9ac1fc86b41f8ca2dbf01f27e45.png)
**对话界面** - 与 AI 助手实时交互：

![对话界面](https://cdn3.ldstatic.com/original/4X/a/a/2/aa2e645830a6d85004b97f0aa610654a6e1d6d6f.png)
**渠道管理** - WhatsApp、Telegram、Discord 等多平台支持：

![渠道管理](https://cdn3.ldstatic.com/original/4X/2/d/2/2d25a097682172434121f58eacd12c0afc352ad0.png)
**配置中心** - 30+ 配置项完整汉化：

![配置中心](https://cdn3.ldstatic.com/original/4X/8/8/0/880eccff480e684f91b003d705dc41acc95a5fe3.png)
**节点配置** - 执行审批、安全策略管理：

![节点配置](https://cdn3.ldstatic.com/original/4X/9/1/8/918efe13e68e7f522289a1ed216fa7f25118998c.png)
**技能插件** - 1Password、Apple Notes 等丰富扩展：

![技能插件](https://cdn3.ldstatic.com/original/4X/9/5/1/951f0905e09386861dbbdc994f9998dc090cecdd.png)
**安装诊断页** - 全面汉化，有漏掉的可以去提issue，保障佬友的优质体验：

![image](https://cdn3.ldstatic.com/original/4X/4/e/6/4e6d44c073e0e2c8478bf417a778df7042e055bc.png)

---

## 环境要求

| 项目 | 要求 |
| --- | --- |
| Node.js | >= 22.12.0（必须） |
| Docker | 可选，服务器部署推荐 |
| 网络 | 需要能访问 AI 模型 API |

检查 Node.js 版本：

```
node -v
# 输出应该是 v22.x.x 或更高
```

如果版本不够，去 [Node.js 官网](https://nodejs.org/) 下载最新 LTS 版本，或者用 nvm 管理：

```
# Linux/macOS
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash
nvm install 22
nvm use 22

# Windows 用 nvm-windows
# https://github.com/coreybutler/nvm-windows
```

---

## 安装方式

提供三种方式，根据自己情况选择。

### 方式 A：一键脚本（推荐新手）

最简单的方式，下载执行脚本自动完成安装。

**Linux / macOS：**

```
curl -fsSL -o install.sh https://cdn.jsdelivr.net/gh/1186258278/OpenClawChineseTranslation@main/install.sh && bash install.sh
```

**Windows PowerShell：**

```
Invoke-WebRequest -Uri "https://cdn.jsdelivr.net/gh/1186258278/OpenClawChineseTranslation@main/install.ps1" -OutFile "install.ps1"; .\install.ps1
```

脚本会自动：

1. 检查 Node.js 版本
2. 安装中文版 npm 包
3. 尝试运行初始化配置

### 方式 B：npm 手动安装

如果脚本有问题，可以手动安装：

```
# 稳定版（推荐）
npm install -g @qingchencloud/openclaw-zh@latest

# 或者 nightly 版（每小时同步上游最新代码）
npm install -g @qingchencloud/openclaw-zh@nightly
```

验证安装：

```
openclaw --version
openclaw --help
```

如果 `--help` 输出是中文，说明安装成功。

### 方式 C：Docker 部署（服务器推荐）

在服务器上运行，或者不想污染本地环境，用 Docker。

**快速启动（本地访问）：**

```
# 1. 初始化配置
docker run --rm -v openclaw-data:/root/.openclaw \
  ghcr.io/1186258278/openclaw-zh:nightly openclaw setup

docker run --rm -v openclaw-data:/root/.openclaw \
  ghcr.io/1186258278/openclaw-zh:nightly openclaw config set gateway.mode local

# 2. 启动
docker run -d \
  --name openclaw \
  -p 18789:18789 \
  -v openclaw-data:/root/.openclaw \
  ghcr.io/1186258278/openclaw-zh:nightly \
  openclaw gateway run
```

启动后访问 `http://localhost:18789` 打开 Dashboard。

---

## 首次配置

安装完成后，需要进行初始化配置。

### 运行初始化向导

```
openclaw onboard
```

这是一个交互式向导，会引导你完成：

1. **选择 AI 模型**：支持 Claude、GPT、本地模型等
2. **配置 API Key**：根据选择的模型输入对应的 API Key
3. **设置聊天通道**：可以连接 WhatsApp、Telegram 等
4. **创建助手人格**：给你的 AI 起个名字，设置性格

整个过程都是中文界面，跟着提示走就行。

### 安装守护进程（可选）

如果希望 OpenClaw 在后台持续运行：

```
openclaw onboard --install-daemon
```

### 常用命令速查

```
openclaw                    # 启动（交互模式）
openclaw onboard            # 初始化向导
openclaw config             # 查看配置
openclaw config set key val # 修改配置
openclaw skills             # 管理技能插件
openclaw status             # 查看运行状态
openclaw gateway run        # 启动网关（Dashboard）
```

---

## Docker 服务器部署详解

这部分重点讲一下在服务器上部署并远程访问的配置，因为这里坑比较多。

### 本地访问 vs 远程访问

| 场景 | 访问地址 | 配置复杂度 |
| --- | --- | --- |
| 本机运行，本机访问 | http://localhost:18789 | 简单 |
| 服务器运行，远程访问 | http://服务器IP:18789 | 需要额外配置 |

**为什么远程访问需要额外配置？**

OpenClaw 的 Dashboard 使用 Web Crypto API 进行设备身份验证，这个 API 在非 HTTPS 环境下只能在 localhost 使用。简单说就是：**通过 HTTP 远程访问时，浏览器安全策略会阻止认证**。

### 方式 1：一键部署脚本（推荐）

项目提供了一键部署脚本，自动完成环境检测、初始化、配置远程访问：

```
# 自动生成 Token
curl -fsSL https://cdn.jsdelivr.net/gh/1186258278/OpenClawChineseTranslation@main/docker-deploy.sh | bash

# 或者指定 Token
curl -fsSL https://cdn.jsdelivr.net/gh/1186258278/OpenClawChineseTranslation@main/docker-deploy.sh | bash -s -- --token 你的密码

# 仅本地访问（不配置远程）
curl -fsSL https://cdn.jsdelivr.net/gh/1186258278/OpenClawChineseTranslation@main/docker-deploy.sh | bash -s -- --local-only
```

脚本会自动：

1. 检查 Docker 环境
2. 拉取镜像
3. 创建数据卷
4. 初始化配置
5. 配置远程访问（Token 认证）
6. 启动容器

部署完成后会显示访问地址和 Token。

### 方式 2：手动配置步骤

如果想手动控制每一步：

```
# 1. 创建数据卷
docker volume create openclaw-data

# 2. 初始化
docker run --rm -v openclaw-data:/root/.openclaw \
  ghcr.io/1186258278/openclaw-zh:nightly openclaw setup

# 3. 配置网关模式
docker run --rm -v openclaw-data:/root/.openclaw \
  ghcr.io/1186258278/openclaw-zh:nightly openclaw config set gateway.mode local

# 4. 配置远程访问（允许局域网访问）
docker run --rm -v openclaw-data:/root/.openclaw \
  ghcr.io/1186258278/openclaw-zh:nightly openclaw config set gateway.bind lan

# 5. 设置访问令牌（重要！远程访问必须）
docker run --rm -v openclaw-data:/root/.openclaw \
  ghcr.io/1186258278/openclaw-zh:nightly openclaw config set gateway.auth.token 你的密码

# 6. 启动容器
docker run -d \
  --name openclaw \
  -p 18789:18789 \
  -v openclaw-data:/root/.openclaw \
  --restart unless-stopped \
  ghcr.io/1186258278/openclaw-zh:nightly \
  openclaw gateway run
```

访问 `http://服务器IP:18789`，在「网关令牌」输入框填入你设置的 Token，点击连接即可。

### 方式 3：Docker Compose

项目提供了 `docker-compose.yml`：

```
# 下载配置文件
curl -fsSL https://cdn.jsdelivr.net/gh/1186258278/OpenClawChineseTranslation@main/docker-compose.yml -o docker-compose.yml
```

配置文件内容：

```
version: '3.8'
services:
  openclaw:
    image: ghcr.io/1186258278/openclaw-zh:nightly
    container_name: openclaw
    ports:
      - "18789:18789"
    volumes:
      - openclaw-data:/root/.openclaw
    environment:
      - OPENCLAW_GATEWAY_TOKEN=${OPENCLAW_GATEWAY_TOKEN:-}
    restart: unless-stopped
    command: openclaw gateway run --allow-unconfigured

volumes:
  openclaw-data:
    name: openclaw-data
```

首次需要初始化配置：

```
# 启动容器（首次会自动创建卷）
docker-compose up -d

# 初始化配置
docker-compose exec openclaw openclaw setup
docker-compose exec openclaw openclaw config set gateway.mode local

# 远程访问配置（可选）
docker-compose exec openclaw openclaw config set gateway.bind lan
docker-compose exec openclaw openclaw config set gateway.auth.token 你的密码

# 重启生效
docker-compose restart
```

---

## 踩坑记录

分享几个实际踩过的坑：

### 坑 1：挂载路径错误

OpenClaw 容器以 **root** 用户运行，配置文件在 `/root/.openclaw`，不是 `/home/node/.openclaw`。

```
# 错误（配置不会持久化）
-v openclaw-data:/home/node/.openclaw

# 正确
-v openclaw-data:/root/.openclaw
```

### 坑 2：必须先初始化再启动

容器启动前必须先运行 `openclaw setup`，否则会报错：

```
Missing config. Run openclaw setup
```

使用一键脚本或按照上面的步骤顺序执行就不会遇到这个问题。

### 坑 3：远程访问报 1008 错误

如果看到这样的错误：

```
disconnected (1008): control ui requires HTTPS or localhost
disconnected (1008): device identity required
```

这是因为没有配置 Token。浏览器安全策略阻止了非 HTTPS 环境下的设备认证。

**解决方法：设置 gateway.auth.token**

```
# 容器已运行的情况下
docker exec openclaw openclaw config set gateway.auth.token 你的密码
docker restart openclaw
```

然后在 Dashboard 的「网关令牌」输入框填入 Token 连接。

### 坑 4：allowInsecureAuth 配置不生效

[官方文档](https://docs.openclaw.ai/gateway/troubleshooting)提到的 `gateway.controlUi.allowInsecureAuth: true` 配置存在上游 Bug，单独使用不起作用。必须配合 `gateway.auth.token` 使用。

![image](https://cdn3.ldstatic.com/original/4X/e/8/f/e8ffa3730d00979dbfffdb38c46b13a8c372f462.png)

---

## 常见问题

### Q：安装后运行还是英文？

可能安装了原版。先卸载再安装中文版：

```
npm uninstall -g openclaw
npm install -g @qingchencloud/openclaw-zh@latest
```

### Q：Dashboard 打不开？

1. 确认容器在运行：`docker ps`
2. 确认端口没被占用：`netstat -tlnp | grep 18789`
3. 查看容器日志：`docker logs openclaw`

### Q：Docker 重启后配置丢失？

检查挂载路径是否正确（应该是 `/root/.openclaw`），以及是否使用了命名卷而不是匿名卷。

### Q：如何更新到最新版？

```
# npm 安装
npm update -g @qingchencloud/openclaw-zh

# Docker
docker pull ghcr.io/1186258278/openclaw-zh:nightly
docker stop openclaw && docker rm openclaw
# 重新启动（配置保留在数据卷中）
docker run -d \
  --name openclaw \
  -p 18789:18789 \
  -v openclaw-data:/root/.openclaw \
  --restart unless-stopped \
  ghcr.io/1186258278/openclaw-zh:nightly \
  openclaw gateway run
```

### Q：如何彻底卸载？

```
# 卸载 npm 包
npm uninstall -g @qingchencloud/openclaw-zh

# 删除配置文件（可选，会删除所有数据）
rm -rf ~/.openclaw

# Docker 方式
docker stop openclaw && docker rm openclaw
docker volume rm openclaw-data
```

---

## 其他远程访问方案

除了 Token 认证，还有其他方案：

| 方案 | 说明 | 适用场景 |
| --- | --- | --- |
| Token 认证 | 设置 gateway.auth.token，Dashboard 输入连接 | 内网，最简单 |
| SSH 端口转发 | ssh -L 18789:127.0.0.1:18789 user@server | 更安全 |
| Tailscale Serve | 自动提供 HTTPS | 跨网络访问 |
| Nginx 反向代理 + HTTPS | 配置 SSL 证书 | 生产环境 |

---

## 常用 Docker 命令

```
# 查看日志
docker logs -f openclaw

# 重启服务
docker restart openclaw

# 停止服务
docker stop openclaw

# 进入容器
docker exec -it openclaw sh

# 查看当前配置
docker exec openclaw cat /root/.openclaw/openclaw.json
```

---

## 版本说明

中文发行版提供两个版本：

| 版本 | npm 标签 | Docker 标签 | 更新频率 |
| --- | --- | --- | --- |
| 稳定版 | @latest | :latest | 手动发布，经过测试 |
| 最新版 | @nightly | :nightly | 每小时自动同步上游 |

推荐日常使用稳定版，想体验最新功能用 nightly。

官方发布新功能后，中文版最快 1 小时内可用。

---

## 写在最后

这个中文发行版会每小时自动同步上游更新，功能和官方保持一致，界面是中文的，开箱即用。

如果使用过程中遇到问题，可以在 GitHub 仓库提 Issue。也欢迎有兴趣的佬友参与贡献。

## 项目地址：GitHub - 1186258278/OpenClawChineseTranslation: 🦞 OpenClaw (Clawdbot/Moltbot) 汉化版 - 开源个人 AI 助手中文版 | Claude/ChatGPT LLM 接入 | WhatsApp/Telegram/Discord 多平台 | 每小时自动同步 | CLI + Dashboard 全中文 | 全流程搭建教程，以及排错指南！
> **常见问题帮助文档地址**：请点击->([常见问题](https://github.com/1186258278/OpenClawChineseTranslation#-%E5%B8%B8%E8%A7%81%E9%97%AE%E9%A2%98)) ← 查阅问题解决方案！
> ![image](https://cdn3.ldstatic.com/original/4X/e/a/a/eaac1abeceed739b9ee3056d219369fdf5ad95fd.png)


_本文基于实际搭建经验整理，如有错误欢迎指正。_
