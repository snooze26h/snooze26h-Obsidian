# 【开源自荐3】Nginx可视化配置编辑器 

[原帖链接](https://linux.do/t/topic/1421387/1)

**作者：anarkh**  
**时间：Jan 8, 2026 8:08 am**  

继《简历优化》和《Docker Compose可视化生成器》后，我又整了一个Nginx可视化配置编辑器。

![1](https://cdn3.ldstatic.com/original/4X/1/b/6/1b65e8c8be519a8dbeabdf6f798b9db60bcbffbb.png)  

可以通过直观的拖拽式画布界面，帮助运维工程师和开发者快速构建、调试和优化 Nginx 配置，减少繁琐的手写配置和语法错误。
主要有以下这些功能：

# 1. 可视化画布编辑 - 拖拽节点构建配置，所见即所得

**支持的节点类型：**

| 节点类型 | 说明 | 对应 Nginx 块 |
| --- | --- | --- |
| Server 节点 | 虚拟主机配置 | server { } |
| Location 节点 | 路径匹配规则 | location { } |
| Upstream 节点 | 负载均衡后端 | upstream { } |

![2](https://cdn3.ldstatic.com/original/4X/8/f/c/8fcb5148cf4b317ff5a63844d40ed49286b650cd.gif)

# 2. 丰富的模板库 - 一键套用常见场景配置模板

内置多种生产级配置模板，覆盖常见应用场景：

| 分类 | 模板名称 | 适用场景 |
| --- | --- | --- |
| 前端 | React/Vue SPA | 单页应用，解决 History 模式刷新 404 |
| 前端 | 静态资源服务器 | CDN 源站，长期缓存 + 防盗链 |
| 后端 | Node.js 反向代理 | Express / NestJS / Fastify |
| 后端 | Python 应用代理 | Django / Flask / FastAPI |
| 后端 | WebSocket 实时通信 | Socket.io / WS 长连接 |
| CMS | WordPress (PHP-FPM) | 博客站点，PHP 处理 |
| CMS | Laravel | PHP 框架，伪静态规则 |
| 高可用 | 多后端负载均衡 | 流量分发，故障转移 |
| 高可用 | 蓝绿部署 | 零宕机发布 |
| 安全 | HTTPS 最佳实践 | TLS 1.2+，HSTS，强制跳转 |
| 安全 | 限流配置 | 防 DDoS，API 限流 |
| 安全 | 隐藏敏感文件 | 禁止访问 .git / .env 等 |

# 3. 流量模拟 - 可视化模拟请求路径匹配

可视化模拟请求路径匹配，帮助理解 Location 匹配优先级。  

比如： 画布中创建以下 **4 个 Location 节点**：

- **节点 A (精确匹配)**: - 规则配置: `User Defined` (或 Exact) - 路径: `= /details`
- **节点 B (正则匹配)**: - 规则配置: `Regex` (区分大小写 `~` 或不区分 `~*`) - 路径: `~ \.png$` _(意思是匹配所有 .png 结尾的)_
- **节点 C (最长前缀匹配 - 高优先级)**: - 规则配置: `Prefix (Important)` (即 `^~`) - 路径: `^~ /static/`
- **节点 D (普通通用匹配)**: - 规则配置: `Prefix` (普通) - 路径: `/`

在“模拟器输入框”中依次输入以下 URL，点击“测试”，可以观察到**光线流向哪个节点**。

#### 测试用例 1：验证“精确匹配”优先级最高

- **输入 URL**: `/details`
- **效果**: 光线流向 **节点 A (`= /details`)**。

#### 测试用例 2：验证 ^~ (非正则前缀) 能够“短路”正则

- **输入 URL**: `/static/icon.png`
- **效果**: 光线流向 **节点 C (`^~ /static/`)**。

#### 测试用例 3：验证“正则匹配”优先级高于“普通前缀”

- **输入 URL**: `/images/logo.png` (注意不要以 /static 开头)
- **效果**: 光线流向 **节点 B (`~ \.png$`)**。

#### 测试用例 4：验证“兜底/回退”逻辑

- **输入 URL**: `/about/us`
- **效果**: 光线流向 **节点 D (`/`)**。    ![3](https://cdn3.ldstatic.com/original/4X/c/6/9/c69a7d68f0e32d2604dfc6b238bad9fe66d0123b.gif)

# 4. 导入/导出 - 支持导入现有配置，导出为配置文件或 Dockerfile

支持导入现有的 Nginx 配置文件进行可视化编辑。

- 自动识别 `server`、`location`、`upstream` 块
- 解析 SSL 配置
- 解析负载均衡配置
- 保留自定义指令

# 5.实时预览 - 实时生成标准 nginx.conf 配置文件

底部预览面板实时生成完整的 `nginx.conf` 配置文件。

- 语法高亮显示
- 一键复制到剪贴板
- 导出为 `.conf` 文件
- 导出为 `Dockerfile`（包含完整部署配置）

# 6. 多语言支持 - 完整的中英文界面支持

完整的中英文界面切换，包括：

- 界面文案
- 配置说明
- 审计报告
- 模板描述

# 7. 属性面板

选中任意节点后，右侧属性面板可编辑该节点的详细配置。

**Server 节点属性：**

- 监听端口 (`listen`)
- 服务器名称 (`server_name`)
- SSL/HTTPS 配置
- 根目录 (`root`) 和索引文件 (`index`)
- HTTP 强制跳转 HTTPS
- 自定义指令

**Location 节点属性：**

- 路径匹配模式 (`=`, `~`, `~*`, `^~`, 无修饰符)
- 代理转发 (`proxy_pass`)
- 代理头设置 (`proxy_set_header`)
- CORS 跨域配置
- WebSocket 支持
- `try_files` 配置
- 重写规则 (`rewrite`)
- 访问控制 (`allow` / `deny`)
- Basic 认证

**Upstream 节点属性：**

- 负载均衡策略 (轮询 / 权重 / IP Hash / 最少连接)
- 后端服务器列表
- 健康检查参数 (`max_fails`, `fail_timeout`)
- 备份服务器 (`backup`)
- 长连接配置 (`keepalive`)

# 8. 智能配置体检并一键修复 - 自动检测并修复安全隐患和性能问题

智能分析配置，检测潜在问题并给出修复建议。

**检测规则分类：**

| 类别 | 规则示例 | 严重程度 |
| --- | --- | --- |
| 安全 | 暴露 Nginx 版本号 (server_tokens on) | 严重 |
| 安全 | 使用 root 用户运行 | 严重 |
| 安全 | 开启目录索引 (autoindex on) | 严重 |
| 安全 | 使用不安全的 SSL 协议 (TLSv1/TLSv1.1) | 严重 |
| 安全 | HTTPS 未强制跳转 | 警告 |
| 安全 | 缺少安全响应头 (X-Frame-Options 等) | 警告 |
| 安全 | 未禁止隐藏文件访问 | 警告 |
| 性能 | 未开启 Gzip 压缩 | 警告 |
| 性能 | 未开启 Sendfile | 警告 |
| 配置 | 存在未使用的 Upstream | 提示 |

**评分机制：**

- 满分 100 分
- 严重问题扣 15 分，警告扣 8 分，提示扣 3 分
- 评级：A (90+) / B (75+) / C (60+) / D (40+) / F (<40)

针对检测到的问题，支持自动修复：

- **单项修复** - 点击问题旁的修复按钮
- **一键全部修复** - 批量修复所有可自动修复的问题

**修复能力：**

- 自动添加 `server_tokens off`
- 自动修正 `user` 为 `nginx`
- 自动移除 `autoindex on`
- 自动升级 SSL 协议到 TLSv1.2+
- 自动配置 HTTP 到 HTTPS 强制跳转（智能处理端口冲突）
- 自动添加安全响应头
- 自动开启 Gzip 和 Sendfile
- 自动添加隐藏文件访问禁止规则

比如，导入以下配置文件：

```

user root;

worker_processes 1;

server_tokens on;

error_log /var/log/nginx/error.log warn;
pid       /var/run/nginx.pid;

events {
    
    worker_connections 1024;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    sendfile        off;
    
    keepalive_timeout  65;

    # gzip  on;

    
    upstream outdated_backend {
        server 127.0.0.1:8080;
    }

    
    server {
        listen       80;
        server_name  example.com;

        # [严重安全问题] 开启目录索引，黑客可以直接浏览文件列表
        autoindex on;

        location / {
            root   /usr/share/nginx/html;
            index  index.html index.htm;
        }

        
    }

    
    server {
        listen       443 ssl;
        server_name  example.com;

        ssl_certificate      /etc/nginx/ssl/server.crt;
        ssl_certificate_key  /etc/nginx/ssl/server.key;
        
        ssl_protocols TLSv1 TLSv1.1 TLSv1.2;

        
        if ($host = 'www.example.com') {
             return 301 https://example.com$request_uri;
        }

        location /api {
            proxy_pass http://outdated_backend;
            
           
        }
    }
}
```

修复后的配置文件：

```
# Generated by Nginx Config Master
# https://nginx-config-master.lovable.app

user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
    worker_connections 1024;
    use epoll;
    multi_accept on;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    log_format main '$remote_addr - $remote_user [$time_local] "$request" $status $body_bytes_sent "$http_referer" "$http_user_agent"';
    access_log /var/log/nginx/access.log main;

    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;

    server_tokens off;
    client_max_body_size 10m;

    # Gzip Settings
    gzip on;
    gzip_comp_level 6;
    gzip_min_length 1024;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml;

    # Upstream: outdated_backend
    upstream outdated_backend {
        server 127.0.0.1:8080 max_fails=3 fail_timeout=30s;
    }

    # Server: example.com (HTTP Redirect)
    server {
        listen 80;
        server_name example.com;

        # Custom server directives
        return 301 https://$host$request_uri;

    }

    # Server: example.com
    server {
        listen 443 ssl;
        server_name example.com;

        # SSL Configuration
        ssl_certificate /etc/nginx/ssl/server.crt;
        ssl_certificate_key /etc/nginx/ssl/server.key;
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384';
        ssl_prefer_server_ciphers on;

        root /var/www/html;
        index index.html index.htm;

        # Custom server directives
        # Block: if
        add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
        add_header X-Frame-Options "SAMEORIGIN" always;
        add_header X-Content-Type-Options "nosniff" always;

        location /api {
            proxy_pass http://outdated_backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

        }

        location ~ /\. {
            return 403;

            # Custom location directives
            # 拦截所有隐藏文件访问（.git, .env 等）
        }

    }

}
```

修复了如下内容：

### 1、 安全性修复

- **权限降权**：将运行用户从 `root` 改为 `nginx`，防止攻击者通过 Nginx 获取系统最高权限。
- **关闭版本泄露**：设置 `server_tokens off`，隐藏 Nginx 版本号，防止针对性漏洞攻击。
- **禁用目录浏览**：删除了 `autoindex on`，禁止黑客直接列出服务器上的所有文件。
- **SSL 协议升级**：禁用了过时的 TLS 1.0/1.1，仅允许更安全的 **TLS 1.2/1.3**。
- **强化加密套件**：配置了强加密算法（Cipher Suites），防止弱加密被破解。
- **引入安全响应头**：添加了 **HSTS**、**防点击劫持（X-Frame-Options）**及内容类型嗅探保护。
- **屏蔽隐藏文件**：新增规则禁止访问 `.git`、`.env` 等以点开头的敏感隐藏文件。

### 2、 性能与稳定性优化

- **多核并行**：将 `worker_processes` 设为 `auto`，自动匹配 CPU 核心数以提升并发处理能力。
- **启用高效 I/O**：开启了 `epoll` 模型、`sendfile` 和 `tcp_nopush`，大幅优化静态资源传输效率。
- **开启 Gzip 压缩**：对文本、JS、CSS 等文件进行压缩，减少网络带宽消耗并加快页面加载。
- **后端健康检查**：在 `upstream` 中增加了超时和重试机制（`max_fails`），防止因个别后端故障导致服务瘫痪。

### 3、 逻辑与路由修正

- **全站 HTTPS 强制跳转**：80 端口不再直接服务，而是通过 `301` 重定向确保用户始终处于加密连接。
- **完善代理透传**：在 `/api` 转发中补全了 `Host` 和 `X-Forwarded-For` 等头部，确保后端应用能获取用户**真实真实 IP**。
- **规范化路径**：将 `root` 目录从系统默认路径改为更标准的 `/var/www/html`。
- **完善日志格式**：增加了自定义 `access_log` 记录，便于监控流量和分析错误。

![4](https://cdn3.ldstatic.com/original/4X/3/d/6/3d6704acb7a29634bc80ce986af0abcadcccc325.gif)

# 地址：

## 工具地址：https://ngxflow.de5.net/

## github地址：GitHub - Anarkh-Lee/nginx-flow: 一款功能强大的 Nginx 配置文件可视化编辑工具

如果佬们觉得有用，希望佬们帮忙给github点点star！  

希望能够对佬们的工作与学习有所帮助！！！

开发的时间比较短，使用过程中有什么bug希望佬友提出来，我下班之后会及时修改！
