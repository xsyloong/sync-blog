>[!INFO] 
Vim 众多例如 Neo-Vim,此处选择 Lazy-Vim   Lazy-Vim 是基于 Neo-Vim 的配置工具

# NeoVim 环境配置

 [neovim官网](https://neovim.io/doc/install/)

## 基础下载
1. 在linux系统使用官方安装配置命令
```bash
curl -LO https://github.com/neovim/neovim/releases/latest/download/nvim-linux-x86_64.tar.gz
sudo rm -rf /opt/nvim-linux-x86_64
sudo tar -C /opt -xzf nvim-linux-x86_64.tar.gz
```
2. 添加环境变量 
```bash
# 将改环境变量配置到 ~/.bashrc 或者其他环境变量配置文件
export PATH="$PATH:/opt/nvim-linux-x86_64/bin"
```

## 使用LazyVim配置
1. 

