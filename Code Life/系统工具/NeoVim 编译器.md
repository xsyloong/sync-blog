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
1. 由于刚下载的neovim并没有配置文件目录，因此直接下载lazy.vim
2. 运行`git clone https://github.com/LazyVim/starter ~/.config/nvim` 直接下载lazy.vim的配置文件
3. 删除git文件 `rm -rf ~/.config/nvim/.git` 后续可以添加为自己的git仓库
4. 运行 `:LazyHealth` 可以看到插件运行状况， 如有报错可以根据提示进行修复
5. lazy.vim配置文件目录结构
```txt
.
├── LICENSE
├── README.md
├── init.lua # 初始化配置，启动 lazy.nvim、自定义配置和插件
├── lazy-lock.json
├── lazyvim.json
├── lua
│   ├── config # 配置目录
│   │   ├── autocmds.lua # 每次打开 nvim 自动执行的命令
│   │   ├── keymaps.lua # 快捷键配置
│   │   ├── lazy.lua # lazy.nvim 的配置
│   │   └── options.lua # 个性化配置
│   └── plugins # 插件目录，每个插件对应一个 .lua 文件
│       └── example.lua
├── stylua.toml
└── test.txt
```
6. 可以在 `lua/config/keymaps.lua` 设置快捷键
7. 配置语言环境
	1. 输入 `:LazyExtras` 进入配置界面
	2. 上下选择所需的 `rust, go, json, markdown` 等插件，直接摁 `x` 键可以启用
 
