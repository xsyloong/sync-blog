# 绕过 DNS 污染

1. 全局覆盖配置文件 json 

```json
dns:

enable: true # 开启 DNS 处理

prefer-h3: true # (可选, 但推荐) 优先使用 DNS over QUIC (HTTP/3)，可以稍微提升解析速度

respect-rules: true # [核心] 让 DNS 查询本身也遵循分流规则，这是实现DNS防泄露和分流的关键

  

# 基础 DNS (Bootstrap)，用于解析其他 DoH/DoT 服务器的域名。必须是 IP 地址，确保启动时可用。

default-nameserver:

- 223.5.5.5

- 119.29.29.29

- 114.114.114.114

  

# 国外流量 & 默认(MATCH)规则 使用的 DNS。通过代理访问，防止DNS污染和泄露。

nameserver:

- https://cloudflare-dns.com/dns-query

- https://dns.google/dns-query

  

# 国内直连(DIRECT)规则 使用的 DNS。国内服务器，速度快。

direct-nameserver:

- https://dns.alidns.com/dns-query

- https://doh.pub/dns-query

  

# 代理服务器域名 使用的 DNS。国内服务器，确保代理节点地址能被快速稳定解析。

proxy-server-nameserver:

- https://dns.alidns.com/dns-query

- https://doh.pub/dns-query

  

# 后备(Fallback) DNS，当 nameserver 组解析失败时使用。同样需要通过代理访问。

fallback:

- tls://1.1.1.1

- tls://8.8.8.8

  
  

rules:

# ===============================================

# 1. 自定义直连规则 (你想绕过代理的域名放这里)

# ===============================================

# 关键词匹配：包含 "local" 的域名直连

- DOMAIN-KEYWORD,local,DIRECT

# 特定不想走代理的网站

- DOMAIN-SUFFIX,ruankaodaren.com,DIRECT

- DOMAIN-SUFFIX,runkao.org.cn,DIRECT

- DOMAIN-SUFFIX,pan.baidu.com,DIRECT

- DOMAIN-SUFFIX,hybgzs.com,DIRECT

- DOMAIN-SUFFIX,api-flowercloud.com,DIRECT

# 谷歌服务强制走代理

- GEOSITE,google,PROXY

# Telegram 强制走代理

- GEOSITE,telegram,PROXY

# 核心：所有中国大陆网站直连

# (包含百度、淘宝、腾讯等绝大多数国内域名)

- GEOSITE,cn,DIRECT

# ===============================================

# 3. IP 规则 (处理直接访问 IP 或 DNS 解析后的 IP)

# ===============================================

# 局域网 IP 直连

- GEOIP,LAN,DIRECT

# 中国大陆 IP 直连 (防止域名规则漏网)

- GEOIP,CN,DIRECT

# ===============================================

# 4. 兜底规则

# ===============================================

# 剩下的所有流量走代理组

- MATCH,PROXY
```

