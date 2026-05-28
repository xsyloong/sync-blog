>[!INFO] 引言
>	llm大语言模型，对于上下文十分敏感。过长的无关内容和命令会导致Llm出现严重的幻觉，十分影响llm效率和能力。因此为了提高实践的效果，增加llm代码生成的效率，采用 规范驱动开发（SSD）这条路径。
>	规范驱动开发： 主要是将产品功能和代码采用一定的规范联系起来。

## 实践文档

[OpenSpec实践]([OpenSpec实践](https://forceinjection.github.io/OpenSpec-practise/))

## 安装

1. 安装OpenSpec `npm install -g @fission-ai/openspec@latest`
2. 命令速查
``` shell
#初始化项目
openspec init --tools none

# 创建变更目录（仅创建目录，不生成文档）
openspec new change <name>

# 列出所有变更 / 规范
openspec list --changes
openspec list --specs

# 验证变更
openspec validate <name>

# 查看状态
openspec status --change <name>

# 归档变更
openspec archive <name>

# 更新工具文件
openspec update
```

## 目录分析
1. 使用`openspec init` 选择相对应工具后会在项目根目录生成对应文档
```
your-project/
├── openspec/                     # OpenSpec 工作目录
│   ├── config.yaml               # 项目配置（技术栈、约定规则等，注入 AI 请求）
│   ├── changes/                  # 变更提案目录（每个功能/变更一个文件夹）
│   └── specs/                    # 主规范目录（已归档的规范）
├── .claude/                       # cladue 专属目录（示例,根据使用ai工具不同名称不同）
│   ├── commands/opsx/            # /opsx 斜杠命令（供 IDE 直接调用）
│   │   ├── propose.md
│   │   ├── explore.md
│   │   ├── apply.md
│   │   └── archive.md
│   └── skills/                   # Agent Skills（AI 自动检测并加载）
│       ├── openspec-propose/SKILL.md
│       ├── openspec-explore/SKILL.md
│       ├── openspec-apply-change/SKILL.md
│       └── openspec-archive-change/SKILL.md
└── ... (项目其他文件)
```
2. 文件说明

| 文件/目录         | 用途                      | 是否必需 |
| ------------- | ----------------------- | ---- |
| `config.yaml` | 项目背景、技术栈、约束条件、每类文档的规则注入 | 推荐填写 |
| `changes/`    | 存放活跃的变更提案               | 必需   |
| `specs/`      | 存放已归档的规范                | 可选   |

3. 创建变更提案
	1. 使用命令`/opsx:propose <description>
		- 根据用户提供的描述推断出kebab-case变更名
		- 创建 `openspec/changes/<name>/`
		- 依次生成 `proposal.md`、`design.md`、`specs/`、`tasks.md` 所有文档
	2. 
5. 