# 绕过 DNS 污染

1. 全局覆盖配置文件 json (防止dns泄漏)
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
```
2. 全局覆盖脚本（js脚本）
```js
  
// ===============================================

// 远程规则集 (Rule Providers)

// ===============================================

const ruleProviders = {

reject: {

type: "http",

behavior: "domain",

url: "https://cdn.jsdelivr.net/gh/Loyalsoldier/clash-rules@release/reject.txt",

path: "./ruleset/reject.yaml",

interval: 86400,

},

icloud: {

type: "http",

behavior: "domain",

url: "https://cdn.jsdelivr.net/gh/Loyalsoldier/clash-rules@release/icloud.txt",

path: "./ruleset/icloud.yaml",

interval: 86400,

},

apple: {

type: "http",

behavior: "domain",

url: "https://cdn.jsdelivr.net/gh/Loyalsoldier/clash-rules@release/apple.txt",

path: "./ruleset/apple.yaml",

interval: 86400,

},

google: {

type: "http",

behavior: "domain",

url: "https://cdn.jsdelivr.net/gh/Loyalsoldier/clash-rules@release/google.txt",

path: "./ruleset/google.yaml",

interval: 86400,

},

proxy: {

type: "http",

behavior: "domain",

url: "https://cdn.jsdelivr.net/gh/Loyalsoldier/clash-rules@release/proxy.txt",

path: "./ruleset/proxy.yaml",

interval: 86400,

},

direct: {

type: "http",

behavior: "domain",

url: "https://cdn.jsdelivr.net/gh/Loyalsoldier/clash-rules@release/direct.txt",

path: "./ruleset/direct.yaml",

interval: 86400,

},

private: {

type: "http",

behavior: "domain",

url: "https://cdn.jsdelivr.net/gh/Loyalsoldier/clash-rules@release/private.txt",

path: "./ruleset/private.yaml",

interval: 86400,

},

gfw: {

type: "http",

behavior: "domain",

url: "https://cdn.jsdelivr.net/gh/Loyalsoldier/clash-rules@release/gfw.txt",

path: "./ruleset/gfw.yaml",

interval: 86400,

},

"tld-not-cn": {

type: "http",

behavior: "domain",

url: "https://cdn.jsdelivr.net/gh/Loyalsoldier/clash-rules@release/tld-not-cn.txt",

path: "./ruleset/tld-not-cn.yaml",

interval: 86400,

},

telegramcidr: {

type: "http",

behavior: "ipcidr",

url: "https://cdn.jsdelivr.net/gh/Loyalsoldier/clash-rules@release/telegramcidr.txt",

path: "./ruleset/telegramcidr.yaml",

interval: 86400,

},

cncidr: {

type: "http",

behavior: "ipcidr",

url: "https://cdn.jsdelivr.net/gh/Loyalsoldier/clash-rules@release/cncidr.txt",

path: "./ruleset/cncidr.yaml",

interval: 86400,

},

lancidr: {

type: "http",

behavior: "ipcidr",

url: "https://cdn.jsdelivr.net/gh/Loyalsoldier/clash-rules@release/lancidr.txt",

path: "./ruleset/lancidr.yaml",

interval: 86400,

},

applications: {

type: "http",

behavior: "classical",

url: "https://cdn.jsdelivr.net/gh/Loyalsoldier/clash-rules@release/applications.txt",

path: "./ruleset/applications.yaml",

interval: 86400,

},

};

  

// ===============================================

// 前置分流规则（优先级高于订阅自带规则）

// 顺序：自定义直连 → 广告拦截 → 强制代理 → 国内直连 → IP规则

// ===============================================

const prependRules = [

// -----------------------------------------------

// 1. 自定义直连域名（最高优先级）

// -----------------------------------------------

"DOMAIN-SUFFIX,hybgzs.com,DIRECT",

"DOMAIN-SUFFIX,ai.hybgzs.com,DIRECT",

"DOMAIN-SUFFIX,ruankaodaren.com,DIRECT",

"DOMAIN-SUFFIX,runkao.org.cn,DIRECT",

"DOMAIN-SUFFIX,pan.baidu.com,DIRECT",

"DOMAIN-SUFFIX,api-flowercloud.com,DIRECT",

"DOMAIN-KEYWORD,local,DIRECT", // 含 local 关键词直连

  

// -----------------------------------------------

// 2. 广告/恶意域名拦截

// -----------------------------------------------

"RULE-SET,reject,REJECT",

  

// -----------------------------------------------

// 3. 私有/局域网域名直连

// -----------------------------------------------

"RULE-SET,private,DIRECT",

  
  

// -----------------------------------------------

// 5. 国内服务直连

// -----------------------------------------------

"RULE-SET,icloud,DIRECT",

"RULE-SET,apple,DIRECT",

"RULE-SET,direct,DIRECT",

"RULE-SET,applications,DIRECT",

  

// -----------------------------------------------

// 6. IP 规则（处理直接 IP 访问或 DNS 解析后的 IP）

// -----------------------------------------------

"GEOIP,LAN,DIRECT,no-resolve", // 局域网 IP 直连

"RULE-SET,lancidr,DIRECT,no-resolve", // 局域网 CIDR 直连

"GEOIP,CN,DIRECT,no-resolve", // 中国大陆 IP 直连

"RULE-SET,cncidr,DIRECT,no-resolve", // 中国大陆 CIDR 直连

];

  

// ===============================================

// 主函数入口

// ===============================================

function main(config) {

// 2. 注入 rule-providers（合并订阅自带的 + 脚本定义的）

config["rule-providers"] = Object.assign(

{},

config["rule-providers"] || {}, // 保留订阅自带的规则集

ruleProviders // 注入脚本定义的规则集

);

  

// 3. 前置规则注入（优先于订阅自带规则）

// 同时保留订阅末尾的 MATCH 兜底规则

config["rules"] = [

...prependRules,

...(config["rules"] || []),

];

  

return config;

}
```
3. clash关闭dns覆写设置