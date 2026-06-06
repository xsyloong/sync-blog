>[!INFO] 引言
>为了更好的使用代理工具，因此直接使用mihomo内核是最合适的方式。
>配置完成后记得删除所有 http , https , ftp , ssh 的代理配置，直接使用 tun 模式即可代替所有

> 引用文章 ： [直接运行 mihomo 内核代理(跑裸核)](https://www.qichiyu.com/282.html)

## 安装流程

### 一键开启透明Ip转发

>[!question] 注意
>这一步根据系统不同，注意命令格式
>
>**最重要核心配置**
> net.ipv4.ip_forward=1 
> net.ipv6.conf.all.forwarding=1

1. 原文是： `sudo sed -i '/net.ipv4.ip_forward/s/^#//;/net.ipv6.conf.all.forwarding/s/^#//' /etc/sysctl.conf && sudo sysctl -p` 命令实现，在ubuntu26无法使用
2. 可行的命令是： `echo -e "net.ipv4.ip_forward=1\nnet.ipv6.conf.all.forwarding=1" | sudo tee /etc/sysctl.d/99-mihomo-forward.conf`


### 下载 mihomo 内核二进制文件并配置服务

1. 下载mihomo内核二进制文件 [mihomo 内核下载地址](github.com/MetaCubeX/mihomo/releases) ，选择版本为 `mihomo-linux-amd64-v3-v1.19.27.gz` (根据系统进行选择)
2. 执行解压命令 `gunzip mihomo-linux-amd64-v3-v1.19.27.gz` 
3. 将解压的二进制文件放到可执行路径中 `sudo mv mihomo-linux-amd64-v3-v1.19.27 /usr/local/bin/mihomo`
4. 赋予运行权限 `sudo chmod 755 /usr/local/bin/mihomo`
5. 制作运行服务目录 `sudo mkdir -p /etc/mihomo`
6. 制作运行服务配置文件 `sudo nano /etc/mihomo/config.yaml`  (注意修改订阅连接)
```yaml
# =================================================================
# 1. 基础与入站配置（已为裸核网关深度优化）
# =================================================================
proxy-providers:
  Airport1:
    url: "切换成相对应的订阅连接1"
    type: http
    interval: 86400
    health-check:
      enable: true
      url: https://www.gstatic.com/generate_204
      interval: 300
    proxy: 直连
    
  Airport2:
    url: "切换成相对应的订阅连接2"
    type: http
    interval: 86400
    health-check:
      enable: true
      url: https://www.gstatic.com/generate_204
      interval: 300
    proxy: 直连
   

proxies:
  - {name: 直连, type: direct}

# 全局基础
mixed-port: 7890
allow-lan: true
bind-address: "*"
ipv6: false
unified-delay: true
tcp-concurrent: true
log-level: warning
#global-client-fingerprint: chrome
keep-alive-idle: 600
keep-alive-interval: 15
profile:
  store-selected: true
  store-fake-ip: true

geox-url:
  geoip: "https://testingcf.jsdelivr.net/gh/MetaCubeX/meta-rules-dat@release/geoip.dat"
  geosite: "https://testingcf.jsdelivr.net/gh/MetaCubeX/meta-rules-dat@release/geosite.dat"
  mmdb: "https://testingcf.jsdelivr.net/gh/MetaCubeX/meta-rules-dat@release/country.mmdb"
# Web 控制面板模块（已解除注释，裸核可用）
external-controller: 0.0.0.0:9090
secret: ""
external-ui: ui
external-ui-name: zashboard
external-ui-url: "https://github.com/Zephyruso/zashboard/archive/refs/heads/gh-pages.zip"

# 流量嗅探
sniffer:
  enable: true
  sniff:
    HTTP:
      ports: [80, 8080-8880]
      override-destination: true
    TLS:
      ports: [443, 8443]
    QUIC:
      ports: [443, 8443]
  skip-domain:
    - "Mijia Cloud"
    - "+.push.apple.com"

# TUN 入站网关模式（已开启三项 true，实现全自动系统路由托管）
tun:
  enable: true
  stack: mixed
  dns-hijack: ["any:53", "tcp://any:53"]
  auto-route: true
  auto-redirect: true
  auto-detect-interface: true

# =================================================================
# 2. DNS 模块（高级防泄露、防污染方案）
# =================================================================
dns:
  enable: true
  prefer-h3: true                
  respect-rules: true            # 让 DNS 查询遵循分流规则，彻底防止 DNS 泄露
  enhanced-mode: fake-ip
  fake-ip-range: 198.18.0.1/16
  cache-algorithm: arc
  listen: 0.0.0.0:1053
  ipv6: false
  fake-ip-filter-mode: blacklist
  fake-ip-filter:
    - "rule-set:fakeipfilter_domain"

  # 基础 DNS，用于解析阿里/腾讯/谷歌等 DoH 域名
  default-nameserver:
    - https://223.5.5.5/dns-query
    - 119.29.29.29
    - 114.114.114.114

  # 默认境外流量/ MATCH 规则使用的 DNS（通过代理组路由，防污染）
  nameserver:
    - https://cloudflare-dns.com/dns-query
    - https://dns.google/dns-query

  # 国内直连流量使用的 DNS
  direct-nameserver:
    - https://dns.alidns.com/dns-query
    - https://doh.pub/dns-query

  # 代理节点服务器域名解析专用的 DNS
  proxy-server-nameserver:
    - https://dns.alidns.com/dns-query
    - https://doh.pub/dns-query

  # 后备 DNS
  fallback:
    - tls://1.1.1.1
    - tls://8.8.8.8
 
# =================================================================
# 3. 出站策略组（现代无锚点架构，原生集成港/日/美/狮自动测速与故转）
# =================================================================
proxy-groups:
  - {name: 🚀 默认代理, type: select, proxies: [🔯 香港故转, 🔯 日本故转, 🔯 狮城故转, 🔯 美国故转, ♻️ 香港自动, ♻️ 日本自动, ♻️ 狮城自动, ♻️ 美国自动, ♻️ 自动选择, 🇭🇰 香港节点, 🇯🇵 日本节点, 🇸🇬 狮城节点, 🇺🇲 美国节点, 🌐 全部节点, 直连]}
  - {name: 📹 YouTube, type: select, proxies: [🔯 美国故转, 🔯 香港故转, 🔯 日本故转, 🔯 狮城故转, ♻️ 美国自动, ♻️ 香港自动, ♻️ 日本自动, ♻️ 狮城自动, ♻️ 自动选择, 🇭🇰 香港节点, 🇯🇵 日本节点, 🇸🇬 狮城节点, 🇺🇲 美国节点, 🌐 全部节点, 直连]}
  - {name: 🍀 Google, type: select, proxies: [🔯 香港故转, 🔯 日本故转, 🔯 狮城故转, 🔯 美国故转, ♻️ 香港自动, ♻️ 日本自动, ♻️ 狮城自动, ♻️ 美国自动, ♻️ 自动选择, 🇭🇰 香港节点, 🇯🇵 日本节点, 🇸🇬 狮城节点, 🇺🇲 美国节点, 🌐 全部节点, 直连]}
  - {name: 🤖 ChatGPT, type: select, proxies: [🔯 日本故转, 🔯 狮城故转, 🔯 美国故转, ♻️ 香港自动, ♻️ 日本自动, ♻️ 狮城自动, ♻️ 美国自动, ♻️ 自动选择, 🇭🇰 香港节点, 🇯🇵 日本节点, 🇸🇬 狮城节点, 🇺🇲 美国节点, 🌐 全部节点, 直连]}
  - {name: 👨🏿‍💻 GitHub, type: select, proxies: [🔯 香港故转, 🔯 日本故转, 🔯 狮城故转, 🔯 美国故转, ♻️ 香港自动, ♻️ 日本自动, ♻️ 狮城自动, ♻️ 美国自动, ♻️ 自动选择, 🇭🇰 香港节点, 🇯🇵 日本节点, 🇸🇬 狮城节点, 🇺🇲 美国节点, 🌐 全部节点, 直连]}
  - {name: 🐬 OneDrive, type: select, proxies: [🔯 日本故转, 🔯 狮城故转, 🔯 美国故转, ♻️ 香港自动, ♻️ 日本自动, ♻️ 狮城自动, ♻️ 美国自动, ♻️ 自动选择, 🇭🇰 香港节点, 🇯🇵 日本节点, 🇸🇬 狮城节点, 🇺🇲 美国节点, 🌐 全部节点, 直连]}
  - {name: 🪟 Microsoft, type: select, proxies: [🔯 日本故转, 🔯 狮城故转, 🔯 美国故转, ♻️ 香港自动, ♻️ 日本自动, ♻️ 狮城自动, ♻️ 美国自动, ♻️ 自动选择, 🇭🇰 香港节点, 🇯🇵 日本节点, 🇸🇬 狮城节点, 🇺🇲 美国节点, 🌐 全部节点, 直连]}
  - {name: 🎵 TikTok, type: select, proxies: [🔯 日本故转, 🔯 狮城故转, 🔯 美国故转, ♻️ 香港自动, ♻️ 日本自动, ♻️ 狮城自动, ♻️ 美国自动, ♻️ 自动选择, 🇭🇰 香港节点, 🇯🇵 日本节点, 🇸🇬 狮城节点, 🇺🇲 美国节点, 🌐 全部节点, 直连]}
  - {name: 📲 Telegram, type: select, proxies: [🔯 香港故转, 🔯 日本故转, 🔯 狮城故转, 🔯 美国故转, ♻️ 香港自动, ♻️ 日本自动, ♻️ 狮城自动, ♻️ 美国自动, ♻️ 自动选择, 🇭🇰 香港节点, 🇯🇵 日本节点, 🇸🇬 狮城节点, 🇺🇲 美国节点, 🌐 全部节点, 直连]}
  - {name: 🎥 NETFLIX, type: select, proxies: [🔯 狮城故转, 🔯 香港故转, 🔯 日本故转, 🔯 美国故转, ♻️ 香港自动, ♻️ 日本自动, ♻️ 狮城自动, ♻️ 美国自动, ♻️ 自动选择, 🇭🇰 香港节点, 🇯🇵 日本节点, 🇸🇬 狮城节点, 🇺🇲 美国节点, 🌐 全部节点, 直连]}
  - {name: 💶 PayPal, type: select, proxies: [🔯 日本故转, 🔯 香港故转, 🔯 狮城故转, 🔯 美国故转, ♻️ 香港自动, ♻️ 日本自动, ♻️ 狮城自动, ♻️ 美国自动, ♻️ 自动选择, 🇭🇰 香港节点, 🇯🇵 日本节点, 🇸🇬 狮城节点, 🇺🇲 美国节点, 🌐 全部节点, 直连]}
  - {name: 🐟 漏网之鱼, type: select, proxies: [🚀 默认代理, 🔯 香港故转, 🔯 日本故转, 🔯 狮城故转, 🔯 美国故转, ♻️ 香港自动, ♻️ 日本自动, ♻️ 狮城自动, ♻️ 美国自动, ♻️ 自动选择, 🇭🇰 香港节点, 🇯🇵 日本节点, 🇸🇬 狮城节点, 🇺🇲 美国节点, 🌐 全部节点, 直连]}
  
  # 正则节点筛选
  - {name: 🇭🇰 香港节点, type: select, include-all: true, filter: "(?=.*(港|HK|(?i)Hong))^((?!(台|日|韩|新|深|美)).)*$"}
  - {name: 🇯🇵 日本节点, type: select, include-all: true, filter: "(?=.*(日|JP|(?i)Japan))^((?!(港|台|韩|新|美)).)*$"}
  - {name: 🇸🇬 狮城节点, type: select, include-all: true, filter: "(?=.*(新加坡|坡|狮城|SG|Singapore))^((?!(台|日|韩|深|美)).)*$"}
  - {name: 🇺🇲 美国节点, type: select, include-all: true, filter: "(?=.*(美|US|(?i)States|America))^((?!(港|台|韩|新|日)).)*$"}
  
  # 故障转移（Fallback）
  - {name: 🔯 香港故转, type: fallback, include-all: true, interval: 300, filter: "(?=.*(港|HK|(?i)Hong))^((?!(台|日|韩|新|深|美)).)*$"}
  - {name: 🔯 日本故转, type: fallback, include-all: true, interval: 300, filter: "(?=.*(日|JP|(?i)Japan))^((?!(港|台|韩|新|美)).)*$"}
  - {name: 🔯 狮城故转, type: fallback, include-all: true, interval: 300, filter: "(?=.*(新加坡|坡|狮城|SG|Singapore))^((?!(台|日|韩|深|美)).)*$"}
  - {name: 🔯 美国故转, type: fallback, include-all: true, interval: 300, filter: "(?=.*(美|US|(?i)States|America))^((?!(港|台|韩|新|日)).)*$"}
  
  # 自动测速（URL-Test）
  - {name: ♻️ 香港自动, type: url-test, include-all: true, tolerance: 20, interval: 300, filter: "(?=.*(港|HK|(?i)Hong))^((?!(台|日|韩|新|深|美)).)*$"}
  - {name: ♻️ 日本自动, type: url-test, include-all: true, tolerance: 20, interval: 300, filter: "(?=.*(日|JP|(?i)Japan))^((?!(港|台|韩|新|美)).)*$"}
  - {name: ♻️ 狮城自动, type: url-test, include-all: true, tolerance: 20, interval: 300, filter: "(?=.*(新加坡|坡|狮城|SG|Singapore))^((?!(港|台|韩|日|美)).)*$"}
  - {name: ♻️ 美国自动, type: url-test, include-all: true, tolerance: 20, interval: 300, filter: "(?=.*(美|US|(?i)States|America))^((?!(港|台|日|韩|新)).)*$"}
  - {name: ♻️ 自动选择, type: url-test, include-all: true, tolerance: 20, interval: 300, filter: "^((?!(直连)).)*$"}
  - {name: 🌐 全部节点, type: select, include-all: true}

# =================================================================
# 4. 规则匹配（已完美融入你的自定义前置规则）
# =================================================================
rules:
  # ① 你的专属前置直连域名
  - DOMAIN-SUFFIX,hybgzs.com,直连
  - DOMAIN-SUFFIX,ai.hybgzs.com,直连
  - DOMAIN-SUFFIX,ruankaodaren.com,直连
  - DOMAIN-SUFFIX,runkao.org.cn,直连
  - DOMAIN-SUFFIX,pan.baidu.com,直连
  - DOMAIN-SUFFIX,api-flowercloud.com,直连
  - DOMAIN-KEYWORD,local,直连

  # ② 广告拦截（Loyalsoldier 规则集请求拦截）
  - RULE-SET,reject,REJECT

  # ③ 系统基础局域网与私有网络直连
  - RULE-SET,private_ip,直连,no-resolve
  - RULE-SET,private_domain,直连

  # ④ 你的专属前置强制代理域名
  - DOMAIN-SUFFIX,code.claudex.us.ci,🚀 默认代理

  # ⑤ 模板级高效二进制大规则集调用
  - DOMAIN-SUFFIX,qichiyu.com,🚀 默认代理
  - RULE-SET,proxylite,🚀 默认代理
  - RULE-SET,ai,🤖 ChatGPT
  - RULE-SET,github_domain,👨🏿‍💻 GitHub
  - RULE-SET,youtube_domain,📹 YouTube
  - RULE-SET,google_domain,🍀 Google
  - RULE-SET,onedrive_domain,🐬 OneDrive
  - RULE-SET,microsoft_domain,🪟 Microsoft
  - RULE-SET,apple_domain,直连
  - RULE-SET,tiktok_domain,🎵 TikTok
  - RULE-SET,telegram_domain,📲 Telegram
  - RULE-SET,netflix_domain,🎥 NETFLIX
  - RULE-SET,paypal_domain,💶 PayPal
  - RULE-SET,apple_ip,直连
  - RULE-SET,google_ip,🍀 Google
  - RULE-SET,netflix_ip,🎥 NETFLIX
  - RULE-SET,telegram_ip,📲 Telegram
  - RULE-SET,geolocation-!cn,🚀 默认代理
  - RULE-SET,cn_domain,直连
  - RULE-SET,cn_ip,直连
  
  # 漏网之鱼兜底
  - MATCH,🐟 漏网之鱼

# =================================================================
# 5. 远程规则集（高效率二进制 .mrs 与常规规则混合集）
# =================================================================
rule-anchor:
  ip: &ip {type: http, interval: 86400, behavior: ipcidr, format: mrs}
  domain: &domain {type: http, interval: 86400, behavior: domain, format: mrs}
  class: &class {type: http, interval: 86400, behavior: classical, format: text}

rule-providers: 
  # 额外扩展：Loyalsoldier 的经典广告拦截集（指定为 text 格式调用）
  reject: { <<: *domain, format: text, url: "https://cdn.jsdelivr.net/gh/Loyalsoldier/clash-rules@release/reject.txt" }

  # 模板原厂 MRS 二进制超高效率分流集
  fakeipfilter_domain: {<<: *domain, url: "https://raw.githubusercontent.com/wwqgtxx/clash-rules/release/fakeip-filter.mrs"}
  private_domain: { <<: *domain, url: "https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/meta/geo/geosite/private.mrs"}
  proxylite: { <<: *class, url: "https://raw.githubusercontent.com/qichiyuhub/rule/refs/heads/main/rules/proxy.list"}
  ai: { <<: *domain, url: "https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/meta/geo/geosite/category-ai-!cn.mrs" }
  youtube_domain: { <<: *domain, url: "https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/meta/geo/geosite/youtube.mrs"}
  google_domain: { <<: *domain, url: "https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/meta/geo/geosite/google.mrs"}
  github_domain: { <<: *domain, url: "https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/meta/geo/geosite/github.mrs"}
  telegram_domain: { <<: *domain, url: "https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/meta/geo/geosite/telegram.mrs"}
  netflix_domain: { <<: *domain, url: "https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/meta/geo/geosite/netflix.mrs"}
  paypal_domain: { <<: *domain, url: "https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/meta/geo/geosite/paypal.mrs"}
  onedrive_domain: { <<: *domain, url: "https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/meta/geo/geosite/onedrive.mrs"}
  microsoft_domain: { <<: *domain, url: "https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/meta/geo/geosite/microsoft.mrs"}
  apple_domain: { <<: *domain, url: "https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/meta/geo/geosite/apple.mrs"}
  speedtest_domain: { <<: *domain, url: "https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/meta/geo/geosite/ookla-speedtest.mrs"}
  tiktok_domain: { <<: *domain, url: "https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/meta/geo/geosite/tiktok.mrs"}
  geolocation-!cn: { <<: *domain, url: "https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/meta/geo/geosite/geolocation-!cn.mrs"}
  cn_domain: { <<: *domain, url: "https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/meta/geo/geosite/cn.mrs"}
  
  private_ip: {<<: *ip, url: "https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/meta/geo/geoip/private.mrs"}
  cn_ip: { <<: *ip, url: "https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/meta/geo/geoip/cn.mrs"}
  google_ip: { <<: *ip, url: "https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/meta/geo/geoip/google.mrs"}
  telegram_ip: { <<: *ip, url: "https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/meta/geoip/telegram.mrs"}
  netflix_ip: { <<: *ip, url: "https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/meta/geo/geoip/netflix.mrs"}
  apple_ip: {<<: *ip, url: "https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/meta/geo-lite/geoip/apple.mrs"}
```
1. 制作守护进程文件 `sudo nano /etc/systemd/system/mihomo.service` , 写入守护进程脚本
```toml
[Unit]
Description=mihomo Daemon, Another Clash Kernel.
After=network.target NetworkManager.service systemd-networkd.service iwd.service

[Service]
Type=simple
User=root
LimitNPROC=500
LimitNOFILE=1000000
CapabilityBoundingSet=CAP_NET_ADMIN CAP_NET_RAW CAP_NET_BIND_SERVICE CAP_SYS_TIME CAP_SYS_PTRACE CAP_DAC_READ_SEARCH CAP_DAC_OVERRIDE
AmbientCapabilities=CAP_NET_ADMIN CAP_NET_RAW CAP_NET_BIND_SERVICE CAP_SYS_TIME CAP_SYS_PTRACE CAP_DAC_READ_SEARCH CAP_DAC_OVERRIDE
Restart=always
ExecStartPre=/usr/bin/sleep 1s
ExecStart=/usr/local/bin/mihomo -d /etc/mihomo
ExecReload=/bin/kill -HUP $MAINPID

[Install]
WantedBy=multi-user.target
```
8. 让 systemd 重新扫描刚刚创建的服务文件 `sudo systemctl daemon-reload` 
9. 赋予开机自启动权限 `sudo systemctl enable mihomo`
10. 立即启动内核服务 `sudo systemctl start mihomo`
11. 常用日常维护命令
	- 重载配置（修改 `config.yaml` 规则或更换节点后无缝生效） `sudo systemctl reload mihomo`
	- 查看当前内核服务运行健康状态 `sudo systemctl status mihomo`
	- 实时查看内核数据转发滚动日志 `journalctl -u mihomo -o cat -f`
	- 网关物理网卡重启（更改静态 IP 或上游网关后备用）(*注意：* 目前该配置在ubuntu26上未生效) `sudo systemctl restart networking`
### 安装后网页配置 `http://127.0.0.1:9090/ui` 