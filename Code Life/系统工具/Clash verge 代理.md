# 绕过 DNS 污染

1. 全局覆盖配置文件 json 

```json
# Profile Enhancement Merge Template for Clash Verge

profile:

store-selected: true

  

dns:

enable: true

prefer-h3: true

default-nameserver:

- 114.114.114.114

- 119.29.29.29

nameserver:

- https://doh.pub/dns-query

- https://doh.360.cn/dns-query

proxy-server-nameserver:

- https://doh.pub/dns-query

- https://doh.360.cn/dns-query

fallback:

- tls://1.1.1.1#节点选择

- tls://8.8.8.8#节点选择
```

