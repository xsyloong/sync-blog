>[!INFO] 引言
>为了更好的使用代理工具，因此直接使用mihomo内核是最合适的方式。
>配置完成后记得删除所有 http , https , ftp , ssh 的代理配置，直接使用 tun 模式即可代替所有

> 引用文章 ： [直接运行 mihomo 内核代理(跑裸核)](https://www.qichiyu.com/282.html)

## 安装流程

### 一键开启透明Ip转发

>[!question] 注意
>这一步根据系统不同，注意命令格式

1. 


### 下载 mihomo 内核二进制文件并配置服务

1. 下载mihomo内核二进制文件 [mihomo 内核下载地址](github.com/MetaCubeX/mihomo/releases) ，选择版本为 `mihomo-linux-amd64-v3-v1.19.27.gz` (根据系统进行选择)
2. 执行解压命令 `gunzip mihomo-linux-amd64-v3-v1.19.27.gz` 
3. 将解压的二进制文件放到可执行路径中 `sudo mv mihomo-linux-amd64-v3-v1.19.27 /usr/local/bin/mihomo`
4. 赋予运行权限 `sudo chmod 755 /usr/local/bin/mihomo`
5. 制作运行服务目录 `sudo mkdir -p /etc/mihomo`
6. 制作运行服务配置文件 `sudo nano /etc/mihomo/config.yaml` 
7. 创建服务文件 `sudo nano /etc/systemd/system/mihomo.service` 
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
