# Clash 小白的 VPS 落地 + 机场中转的链式代理配置过程 

[原帖链接](https://linux.do/t/topic/1238749/1)

**作者：Dionysun**  
**时间：Nov 29, 2025 11:26 pm**  

## 前言：为什么折腾这个？

1. 手头有**机场订阅**，线路快，但 IP 是 NAT 的广播 IP，经常跳Cloudflare盾，并且有潜在的降智、风控风险。
2. 自建可能有频繁封IP、端口的情况。
3. 黑五闲得无聊想买个国外鸡折腾一下，家宽又有点贵。

**目标**：利用机场节点做**前置中转**，流量路径为 `我 -> 机场(香港/日本) -> 落地鸡 -> Google`，也就是 **“链式代理” (Chain Proxy)**。既享受了机场的速度，并且在网络出入口由机场进行代理，避免频繁更换VPS的IP，又拥有了落地鸡的干净 IP。

---

## 一、落地机的设置

由于现有的机场协议为SS，并且该VPS的流量不会直接从国内直连，因此落地机采取 **SS** 协议作为转发。这里，我采用了233boy的一键配置，并且在 VPS 的防火墙中开放相应端口。

  

      [github.com](https://github.com/233boy/sing-box)
  

  
    
  ![](https://cdn3.ldstatic.com/optimized/4X/1/3/3/13301e3a9d9815c1e1d726b68bbf39458a12a666_2_690x344.png)

  ### GitHub - 233boy/sing-box: 最好用的 sing-box 一键安装脚本 & 管理脚本，自动创建 REALITY 协议；支持...

    最好用的 sing-box 一键安装脚本 & 管理脚本，自动创建 REALITY 协议；支持 TUIC，Trojan，Hysteria2 等所有常见的协议

  

  
    
    
  

  

---

## 二、Yaml 配置文件

  

      [github.com](https://github.com/Lanlan13-14/Rules)
  

  
    
  ![](https://cdn3.ldstatic.com/optimized/4X/4/8/0/48002ace1eca6783352d547df4a7ad390049ef6b_2_690x344.png)

  ### GitHub - Lanlan13-14/Rules: 适用于Mihomo/Stash客户端的Yaml配置文件

    适用于Mihomo/Stash客户端的Yaml配置文件

  

  
    
    
  

  

配置文件参考了上述仓库中的标准版[configfull.yaml](https://raw.githubusercontent.com/Lanlan13-14/Rules/refs/heads/main/configfull.yaml)  

并且做出以下改动：

- 在`proxy-providers`中填写自己的机场
- 在`proxy-groups`中填写实际需要中转流量的节点 - 这里，我选择了`url-test`的策略，因为我的落地机在美西，将对google的测速认为是对我的落地机的测速，进行前置节点的选择。 - 并且，在测试中发现，当 Clash 重载配置或机场节点全部超时的瞬间，Clash 会**降级为直连**，尝试直接连接我的 VPS。因此，我做了以下三个步骤： 1. 不直接用`url-test`作为前置，而是外套一个`fallback` 2. 在`hosts`里把落地机的域名解析到`127.0.0.1`，这样在走代理时：域名发给机场，远程解析真实 IP，而在机场中转组（自动前置）未测速完，自动降级为直连时，无法解析，拒绝访问，防止国内 IP 直连。   ``` # 在全局设置置顶中填写 hosts:   "域名": 127.0.0.1 # 填写proxy-groups - {name: 自动竞速, type: url-test, proxies: [香港自动, 日本自动, 新加坡自动, 美国自动, 台湾自动], url: 'http://www.gstatic.com/generate_204', interval: 300, tolerance: 50, hidden: false} - {name: 自动前置, type: fallback, proxies: [自动竞速, 🚫 拒绝], url: 'http://www.gstatic.com/generate_204', interval: 300, hidden: false} ```
- 在`proxies`中填写自己的落地机配置，并且配置`dialer-proxy`，其中为你需要的前置中转组，即第二步中所填写部分``` - name: 美国落地鸡   type: ss   server: 域名   port: 端口   cipher: 你的设置   password: 脚本中的密码   udp: true   dialer-proxy: 自动前置 ```

---
