# 手把手带你搞懂 Clash Meta 的 DNS 分流，从此配置不求人！ 

[原帖链接](https://linux.do/t/topic/2013293/1)

**作者：nothankyou**  
**时间：Apr 20, 2026 4:21 am**  

# Clash Meta DNS 模块：一份真正看得懂的配置指南

如果你曾经从别人的 dotfiles 或论坛帖子里复制过 Clash Meta 的 DNS 配置，你不是一个人。Clash Meta 的 DNS 模块选项繁多，大多数教程要么对原理只字不提，要么还没把基础讲清楚就让读者一头扎进各种细枝末节之中。

因此，这篇文章想要换个思路：当你读完之后，你不仅会得到一份完整可用的 DNS 配置，更重要的是，你会知道每一行在做什么、为什么这样写。

我们的目标很简单：在确保本地服务正在运转的情况下，对于国内域名走国内 DNS 解析，对于国外域名通过代理进行解析。从而达到隐私和速度的完美均衡。

欢迎移步我的英文原版博客，以获取更好的阅读体验：[Clash Meta DNS Module: A Config You Can Actually Understand](https://nothankyouzzz.pages.dev/clash-meta-dns/)

---

## 1. Clash 为什么需要自己的 DNS

在深入配置之前，我们先搞得清楚 Clash Meta 为什么要有自己的 DNS 模块。正常的 DNS 解析流程很简单：应用问操作系统，操作系统问 DNS 服务器，DNS 服务器返回 IP。就这样。

可是一旦引入代理，这个流程就崩溃了，在 TUN 模式下尤为明显。

TUN 模式通过创建虚拟网卡，在 IP 包层面接管所有出站流量。这很强大，它确保了没有流量能够绕过，每个连接都经过 Clash。但它有一个根本问题：**IP 数据包只携带目标 IP 地址，不携带域名**。等数据包到达规则路由时，域名已经消失了。

这很关键，因为你的路由规则是按域名写的：

```
rules:
  - DOMAIN-SUFFIX,google.com,Proxy
  - DOMAIN-SUFFIX,baidu.com,DIRECT
```

如果 Clash 只看到 `142.250.80.46:443`，这些规则形同虚设——它根本不知道这个 IP 属于 Google。

Clash 的 DNS 模块通过拦截 DNS 查询来解决这个问题——在查询**离开本机之前**截住它，由 Clash 自己完成解析，同时保存域名到 IP 的映射，供后续路由决策使用。

这就是 DNS 模块存在的根本原因。它不只是"选一个快的 DNS 服务器"那么简单，而是让域名在路由决策发生之前始终对 Clash 可见。

---

## 2. fake-ip：用一点代价换速度

Clash 有两种 DNS 处理模式，由 `enhanced-mode` 控制。

较简单的是 **redir-host**。应用查询域名时，Clash 正常解析，返回真实 IP，同时记录映射关系——`142.250.80.46 → google.com`——以便连接到来时能还原域名。逻辑清晰，但每个连接都要等完整的 DNS 解析完成后才能开始建连。

**fake-ip** 是更聪明的方案。Clash 不等解析完成，而是立刻从保留地址段里返回一个假 IP——比如 `198.18.0.5`——让应用马上开始连接。Clash 拦截这个连接，识别出 `198.18.0.5` 对应 `google.com`，然后正确路由。真正的 DNS 解析在后台进行，或者直接交给代理处理。

结果就是连接_感觉更快_——DNS 不再阻塞建连。

```
enhanced-mode: fake-ip
fake-ip-range: '198.18.0.1/16'  # Clash 分配假 IP 的地址池
```

### 代价

这个技巧之所以能生效，是因为 Clash 处于中间位置，同时控制着两端。但有些域名完全不符合这个预期：

- **NTP 客户端**连接到 `time.windows.com`，拿到假 IP，然后尝试跟它同步时间——显然会失败。
- **局域网发现**需要真实的本地 IP 才能访问到你的路由器或 NAS。
- **Windows 网络连通性检查**会探测特定的微软端点；假 IP 会让 Windows 误以为没有网络。

对这些域名，fake-ip 会造成静默的、难以排查的故障——TCP 层面看起来连接成功，但应用层面已经失败了。解决方案就是下一个配置项：`fake-ip-filter`。

---

## 3. 保护本地服务：fake-ip-filter

`fake-ip-filter` 是一个例外列表。匹配到的域名会绕过 fake-ip 机制，直接返回真实 IP。

```
fake-ip-filter-mode: blacklist  # 所有域名返回假 IP，除了下面列表里的
fake-ip-filter:
  - '*.lan'                      # 局域网主机名——路由器、NAS、打印机
  - '*.local'                    # mDNS / 本地服务
  - '*.arpa'                     # 反向 DNS 查询——本质上是基于 IP 的
  - 'time.*.com'                 # NTP 时间服务器
  - 'ntp.*.com'                  # NTP 时间服务器
  - '+.market.xiaomi.com'        # 小米应用商店 CDN
  - 'localhost.ptlogin2.qq.com'  # QQ 本地认证
  - '*.msftncsi.com'             # Windows 网络连通性检查
  - 'www.msftconnecttest.com'    # Windows 网络连通性检查
```

`blacklist` 模式的逻辑很直接：**所有域名都返回假 IP，列表里的除外**，列表里的域名返回真实 IP。

### 哪些域名需要真实 IP

需要真实 IP 的域名大致分三类：

**本地网络域名**——任何需要解析到局域网地址的域名。`*.lan`、`*.local`、`*.arpa` 都属于这类。`198.18.x.x` 范围的假 IP 永远到不了你的路由器。

**时间敏感协议**——NTP 客户端拿到 IP 后直接建连，期望对端是时间服务器。这里没有 TLS SNI 那种应用层的主机名协商机制，假 IP 只会走进死胡同。

**网络连通性探测**——Windows、Android 等系统会定期访问特定端点来确认联网状态。这些请求拿到假 IP 后，系统会误判为强制门户或断网，进而出现各种奇怪的行为。
> **实践建议：** 这个列表不需要面面俱到。通配符模式已经覆盖了主要类别，特定应用的条目等遇到实际故障时再补充。把它当作随时间积累的东西，而不是一开始就要配齐的清单。


---

## 4. 还原丢失的域名：Sniffer（域名嗅探）

前面说了 TUN 模式会丢失域名——Clash 只看到 IP 包。DNS 模块通过拦截查询、维护 fake-ip 映射部分解决了这个问题，但还有缺口。

不是所有流量都伴随着 Clash 能拦截的 DNS 查询。有些应用会在重启后继续使用缓存的 DNS 结果，有些则直接用 IP 建连。在 fake-ip 模式下，目标 IP _始终_是假的——对 `198.18.0.5` 做 GeoIP 查询没有任何意义。

Sniffer 的作用是从流量本身把域名挖出来，填补这个缺口。

### 原理

IP 包里没有域名，但包裹在其中的应用层协议几乎都有。浏览器建立 TLS 连接时，第一个包是 ClientHello——这个握手包里以明文形式嵌入了 **SNI（Server Name Indication）** 字段：

```
TLS ClientHello
  └── SNI: google.com   ← 明文可读，Clash 也不例外
```

HTTP 更直接，每个请求头里都有 `Host:` 字段。QUIC 与 TLS 类似，握手时同样暴露 SNI。Sniffer 拦截这些握手包，提取出域名，将其替换回连接的目标地址。此后 Clash 就把这个连接当作目标是 `google.com` 来处理——域名路由规则得以正常生效。

```
sniffer:
  enable: true
  force-dns-mapping: true   # 用嗅探到的 SNI 覆盖过时的 redir-host 映射
  parse-pure-ip: true       # 对完全没有 DNS 映射的连接也尝试嗅探
  override-destination: true
  sniff:
    TLS:
      ports: [443, 8443]
    HTTP:
      ports: [80, 8080-8880]
    QUIC:
      ports: [443, 8443]
```

### force-dns-mapping 和 parse-pure-ip

**`force-dns-mapping`** 针对 redir-host 模式：Clash 已经有了 DNS 拦截得到的域名到 IP 映射，但嗅探结果可能更准确——比如更具体的子域名，或者比缓存更新的版本。开启后，嗅探到的 SNI 会覆盖原有映射。

**`parse-pure-ip`** 针对完全没有 DNS 映射的连接——比如缓存了 IP 的应用，或者直接用 IP 建连的服务。没有这个选项，Clash 就无从获取域名；开启后，Sniffer 仍会尝试从握手包里提取。

### 为什么在 fake-ip 模式下不可或缺

在 fake-ip 模式下，每个目标 IP 都来自 `198.18.0.0/16`。这个地址段不属于任何国家，GeoIP 规则毫无意义；域名规则也无法从裸 IP 触发。Sniffer 是假 IP 替换掉真实目标之后，**唯一能还原域名的手段**。
> 在 fake-ip 模式下不开启 Sniffer，精心编写的域名规则全部失效——所有路由都退化为纯 IP 匹配。


---

## 5. DNS 解析流水线

了解了各个组件之后，来看它们如何协同工作。Clash 收到 DNS 查询后，会经过三层流水线处理，优先级严格递减：

```
收到 DNS 查询
      ↓
1. nameserver-policy   ← 最高优先级，最先检查
      ↓（未命中）
2. nameserver          ← 默认解析器
      ↓（配置了 fallback 时）
3. fallback            ← 与 nameserver 并发查询，用于纠错
```

规则很简单：

- 域名**命中 `nameserver-policy`** 时，直接交给 policy 指定的 DNS，nameserver 和 fallback 完全不参与。
- **没有 policy 命中**时，查询交给 `nameserver`。若配置了 `fallback`，两者并发查询，由 `fallback-filter` 决定最终采用哪个结果。
- **`fallback` 不是故障转移**——它不在 nameserver 宕机或变慢时激活，而是根据 nameserver 返回内容来判断，专门用于识别 DNS 污染。

这个优先级顺序是实现国内/国外 DNS 分流的核心——接下来三节会逐层讲解和配置。

---

## 6. 解决国内/国外分流：nameserver-policy

### 污染问题

你可能会想：为什么不用同一个 DNS 服务器处理所有域名？问题在于，"好用"对国内和国外域名意味着截然不同的东西。

对于国内域名，用 `8.8.8.8` 这样的境外 DNS 技术上能用，但结果往往不理想——你可能被路由到海外 CDN 节点而非国内节点，白白增加延迟。

对于国外域名，国内 DNS 则是**主动有害的**。无论是运营商 DNS 还是 `223.5.5.5` 这样的公共解析器，对被封锁域名返回的都是污染结果——错误 IP、无效 IP，甚至根本连不通的地址。对于任何需要走代理的域名，都不能信任它们的答案。

唯一干净的解法是根据域名**在查询阶段就做路由**——国内查询走国内 DNS，国外查询走境外 DNS。这正是 `nameserver-policy` 的职责。

### nameserver-policy 的工作方式

`nameserver-policy` 是一张域名模式到 DNS 服务器的路由表，在流水线中优先级最高。域名一旦命中，直接交给对应服务器，结果直接使用——不需要 fallback，也不需要污染过滤。

```
nameserver-policy:
  'geosite:cn,private':
    - '223.5.5.5'                        # 阿里 DNS——快，国内
    - '119.29.29.29'                     # DNSPod——快，国内
    - 'https://doh.pub/dns-query'        # DNSPod DoH——加密备份
    - 'https://dns.alidns.com/dns-query' # 阿里 DoH——加密备份
```

### 解读 geosite:cn,private

`geosite:cn,private` 这个键实际上包含两个分组：

**`geosite:cn`** ——社区维护的中国域名列表，涵盖百度、微信、淘宝等数千个域名。命中这个分组的域名走国内 DNS，它了解这些域名的 CDN 拓扑，能给出快速准确的结果。

**`geosite:private`** ——私有和保留网络域名：`*.local`、`*.lan`、`*.arpa`、反向 DNS 等。这些域名本不应该离开局域网，把它们发给境外 DNS，既会查询失败，还会泄露内网信息。

### 一个 policy 条目配置多个服务器

注意这里列了四个服务器。Clash 会并发查询所有服务器，取最快的响应。普通 UDP 服务器响应速度更快；DoH 服务器稍慢但走加密通道，在 UDP 不稳定时充当可靠备份，顺带为国内域名查询提供一层隐私保护。

---

## 7. 国外域名：委托给代理解析

未命中 `nameserver-policy` 的域名落到 `nameserver` 层。能到这里的，几乎全是国外域名——国内域名早已被 `geosite:cn,private` 截走了。

对于国外域名，有一个明确要求：**DNS 查询本身必须通过代理隧道发出**。原因有两个：

**污染**——从中国大陆直连 `8.8.8.8`，好一点是结果不稳定，坏一点是被主动污染。查询必须经由代理隧道出境，才能干净地到达 DNS 服务器。

**隐私**——在 ISP 网络上发出的明文 DNS 查询，会暴露你访问的每一个域名，哪怕实际连接走了代理。让查询也走代理，才能彻底堵住这个泄漏。

### # 代理后缀语法

Clash 允许用 `#` 后缀直接为 DNS 服务器条目指定路由出口：

```
nameserver:
  - 'https://8.8.8.8/dns-query#代理组名'  # 通过指定代理组出境
  - 'https://1.1.1.1/dns-query#RULES'     # 遵循路由规则出境
```

**`#代理组名`** 将 DNS 查询强制通过指定代理组发出，行为明确可预期。把 `代理组名` 替换为你实际的代理组名称，例如 `Proxy`。

**`#RULES`** 让 Clash 按照路由规则决定 DNS 查询的出口，与普通流量的处理方式一致。由于 `8.8.8.8` 和 `1.1.1.1` 是境外 IP，它们自然会命中代理规则走隧道出境。对大多数配置而言，两者效果等价，但 `#RULES` 在代理组名变更时更具弹性。

---

## 8. 引导启动问题

目前的配置里，藏着一个先有鸡还是先有蛋的问题。

我们配置了 `nameserver-policy` 使用 `https://doh.pub/dns-query` 这样的 DoH 服务器——但 DoH 基于 HTTPS，而 HTTPS 需要先把服务器主机名解析为 IP 才能建立连接。_那 doh.pub 这个域名，谁来解析？_

类似地，代理节点通常以 `jp.example.com` 这样的主机名定义。Clash 需要解析这个主机名才能连上代理——可 DNS 配置又要求查询通过代理出境。你没法用代理去解析连接代理所需的地址。

两个专用字段分别破解这两个死结。

### default-nameserver：解析 DNS 服务器主机名

`default-nameserver` 是一个引导解析器，专门负责解析其他 DNS 服务器的主机名。它在流水线其余部分就绪之前运行，因此有一个硬性约束：**必须填纯 IP 地址**，不能是主机名。

```
default-nameserver:
  - '223.5.5.5'    # 纯 IP——无需任何解析即可直达
  - '119.29.29.29' # 纯 IP——同上
```

这些服务器只用于解析 `doh.pub`、`dns.alidns.com` 等 DNS 基础设施的主机名，从不参与实际流量的查询。

### proxy-server-nameserver：解析代理节点主机名

`proxy-server-nameserver` 专门用于解析代理节点的域名，独立于主 DNS 流水线运行，专为打破循环依赖而设。

```
proxy-server-nameserver:
  - '223.5.5.5'
  - '119.29.29.29'
```
> **关键点：** 这两个字段存在的原因是一样的——有些 DNS 解析必须在主流水线建立之前完成。它们是让整套配置得以运转的前提。


---

## 9. 直连出口的重新解析

当一个连接被路由到 **DIRECT**——不走代理——Clash 需要建立真实的 TCP 连接，目标必须是真实 IP。在 fake-ip 模式下，此时持有的是假 IP，必须重新解析域名才能拿到可路由的真实地址。这就是 `direct-nameserver` 的用途。

```
direct-nameserver:
  - '223.5.5.5'
  - '119.29.29.29'
```

### 触发时机

```
应用查询 baidu.com
  → Clash DNS 模块拦截查询
  → nameserver-policy 命中 geosite:cn → 指定 223.5.5.5 解析（后台进行）
  → Clash 立即返回假 IP 198.18.0.5 给应用
  → 应用连接到 198.18.0.5
  → 路由规则匹配：DIRECT
  → Clash 需要真实 IP 才能建连
  → direct-nameserver 重新解析 baidu.com → 获得真实 IP
  → 连接建立 ✓
```

### direct-nameserver-follow-policy

这个选项控制 `direct-nameserver` 在重新解析时是否遵守 `nameserver-policy`：

```
direct-nameserver-follow-policy: true
```

设为 `true` 时，直连连接的重新解析同样遵循 policy 路由——国内域名走国内 DNS，行为前后一致。

实际上，DIRECT 连接几乎全是国内域名，本就被 policy 覆盖，两种设置的实际效果差别不大。开启这个选项更多是**防御性考量**：即便路由规则将来有变，流水线各阶段的行为仍能保持一致。开启它没有任何代价。

---

## 10. Fallback：安全网

我们故意把 `fallback` 留到最后——有了精心配置的 `nameserver-policy` 处理分流，fallback 在这套方案里基本上是多余的。但理解它的作用和局限，仍然有价值。

### Fallback 是做什么的

Fallback 的存在是为了纠正 DNS 污染。配置了 `fallback` 后，Clash 会同时向 `nameserver` 和 `fallback` 发出查询，再由 `fallback-filter` 检查 `nameserver` 的返回结果，决定最终采用哪个：

```
nameserver 和 fallback 并发查询
        ↓
fallback-filter 评估 nameserver 的结果：
  ├── 结果看起来正常（CN 域名返回 CN IP）→ 采用 nameserver 结果
  └── 结果疑似污染（非 CN IP、可疑地址段）→ 采用 fallback 结果
```

### 为什么在这套配置里是多余的

`nameserver-policy` 在查询发生之前就已经把流量路由到了正确的 DNS 服务器——国内域名走国内 DNS，国外域名走代理。污染根本没有机会进入流水线。Fallback 解决的问题，`nameserver-policy` 从源头就阻止了它发生。

### 被 GUI 强制加入的情况

Clash Verge 的图形界面会强制写入某些 `fallback-filter` 字段，即便你不需要它们。让这些字段完全失效的方法：

```
fallback: []        # 空列表——fallback-filter 无从生效
fallback-filter:
  geoip: false      # 禁用 GeoIP 评估
```

`fallback: []` 之后，所有 filter 配置都成了摆设，流水线行为与 fallback 不存在时完全一致。

---

## 11. 完整配置

以下每个选项都在前面的章节里讲解过，注释追溯每一行背后的原因。

```
# ── Sniffer ──────────────────────────────────────────────────────────────────
# 在 TUN 模式下从 TLS/HTTP 握手中还原域名。
# 没有这个，域名路由规则对 TUN 流量形同虚设。
sniffer:
  enable: true
  force-dns-mapping: true    # 用嗅探到的 SNI 覆盖过时的 redir-host 映射
  parse-pure-ip: true        # 对完全没有 DNS 映射的连接也尝试嗅探
  override-destination: true
  sniff:
    TLS:
      ports: [443, 8443]
    HTTP:
      ports: [80, 8080-8880]
    QUIC:
      ports: [443, 8443]

# ── DNS 模块 ─────────────────────────────────────────────────────────────────
dns:
  enable: true
  listen: ':53'

  # fake-ip：立即返回假 IP 以提升速度。
  # Clash 拦截连接并按域名路由；真正的解析在后台或通过代理进行。
  enhanced-mode: fake-ip
  fake-ip-range: '198.18.0.1/16'

  # 黑名单模式：所有域名返回假 IP，以下列表的除外。
  fake-ip-filter-mode: blacklist
  fake-ip-filter:
    - '*.lan'                      # 局域网主机名——路由器、NAS、打印机
    - '*.local'                    # mDNS / 本地服务
    - '*.arpa'                     # 反向 DNS——本质上是基于 IP 的
    - 'time.*.com'                 # NTP 时间服务器
    - 'ntp.*.com'                  # NTP 时间服务器
    - '+.market.xiaomi.com'        # 小米应用商店 CDN
    - 'localhost.ptlogin2.qq.com'  # QQ 本地认证
    - '*.msftncsi.com'             # Windows 网络连通性检查
    - 'www.msftconnecttest.com'    # Windows 网络连通性检查

  # 引导 DNS：在流水线就绪前解析 DoH 服务器主机名。
  # 必须填纯 IP——此阶段没有主机名解析能力。
  default-nameserver:
    - '223.5.5.5'
    - '119.29.29.29'

  # 独立于主流水线，专门解析代理节点主机名。
  # 打破循环依赖：代理需要 DNS，DNS 需要代理。
  proxy-server-nameserver:
    - '223.5.5.5'
    - '119.29.29.29'

  # 最高优先级层：在解析发生前按域名路由查询。
  # 国内和私有域名 → 国内 DNS，不触碰境外 DNS 服务器。
  nameserver-policy:
    'geosite:cn,private':
      - '223.5.5.5'                        # 阿里 DNS——快，国内
      - '119.29.29.29'                     # DNSPod——快，国内
      - 'https://doh.pub/dns-query'        # DNSPod DoH——加密备份
      - 'https://dns.alidns.com/dns-query' # 阿里 DoH——加密备份

  # 默认层：处理所有未被 nameserver-policy 命中的域名。
  # 到这里的几乎全是国外域名。
  # #RULES 让查询通过代理隧道出境——干净，无污染。
  nameserver:
    - 'https://8.8.8.8/dns-query#RULES'
    - 'https://1.1.1.1/dns-query#RULES'

  # 为直连连接重新解析域名——假 IP 无法建立真实连接，
  # 此阶段必须拿到真实 IP。
  direct-nameserver:
    - '223.5.5.5'
    - '119.29.29.29'
  # 重新解析时同样遵守 nameserver-policy，保持端到端行为一致。
  # 防御性选项；实际上 DIRECT 流量几乎都是国内域名。
  direct-nameserver-follow-policy: true

  # 禁用 fallback——nameserver-policy 已在前端干净地完成了分流。
  # fallback: [] 加上 geoip: false，使 GUI 强制写入的 filter 字段完全失效。
  fallback: []
  fallback-filter:
    geoip: false

  prefer-h3: false
  respect-rules: false
  use-hosts: false
  use-system-hosts: false
  ipv6: true
```

### 两处需要自定义的地方

**代理组名**——如果 nameserver 条目里用的是 `#代理组名` 而不是 `#RULES`，把它替换为你实际的代理组名称，例如 `Proxy`。

**运营商 DNS vs 公共国内 DNS**——`223.5.5.5` 和 `119.29.29.29` 是公共国内 DNS，并非你运营商分配的 DNS。如果想用真正的运营商 DNS，改为 `system` 即可。不过公共国内 DNS 通常更稳定，CDN 路由感知也更好，实际上这个区别几乎不影响使用体验。

---

## 12. 每一行都有来由

好的 DNS 配置不是魔法——它是一条流水线，每一层都在回答一个具体的问题：

| 问题 | 答案 |
| --- | --- |
| 如何让 Clash 始终看到域名？ | Sniffer |
| 如何让连接更快？ | fake-ip |
| 哪些域名不能用假 IP？ | fake-ip-filter |
| 如何把国内和国外查询路由到正确的 DNS？ | nameserver-policy |
| 如何不泄漏查询地解析国外域名？ | nameserver 加 #RULES |
| 如何在流水线就绪前完成引导解析？ | default-nameserver、proxy-server-nameserver |
| 如何为直连连接获取真实 IP？ | direct-nameserver |

这样去理解它，配置就不再是一堵照抄的选项墙，而是一组对具体问题的有意识的回答。当网络环境变化——更换代理节点、换了 ISP、新增本地服务——你会清楚地知道该改哪里，以及为什么。
