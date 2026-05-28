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

## 使用流程

### 创建变更提案

1. 使用命令`/opsx:propose <description>
	- 根据用户提供的描述推断出kebab-case变更名
	- 创建 `openspec/changes/<name>/`
	- 依次生成 `proposal.md`、`design.md`、`specs/`、`tasks.md` 所有文档
2. 创建变更目录 `/opsx:new <change-name>
		1. 只生成目录不创建任何文档
		2. 配合 `/opsx:continue` 逐步手动生成文档时使用。
```
# 好的命名示例
add-user-authentication
add-payment-module
fix-login-timeout

# 不好的命名示例
feature1           # 太模糊
addUserAuth        # 应使用 kebab-case
```

### 变更文件说明

#### 文件说明

```
openspec/changes/<change-name>/
├── .openspec.yaml     # 变更元数据（ID、状态、创建时间等，由 CLI 自动管理）
├── proposal.md        # 提案文档【必填】描述 Why 和 What
├── design.md          # 技术设计文档（架构、数据模型、API 设计等）
├── tasks.md           # 实现任务清单（按里程碑组织的待办事项）
└── specs/             # 规范目录（存放能力规范文件）
    ├── <capability-1>/
    │   └── spec.md    # 能力规范（使用 Requirement + Scenario 格式）
    ├── <capability-2>/
    │   └── spec.md
```

|文件|作用|是否必需|格式要求|
|---|---|---|---|
|`proposal.md`|说明“为什么做”和“做什么”|**必需**|必须包含 `## Why` 和 `## What Changes`（验证器强制检查）；推荐包含 `## Capabilities`（AI 工作流所需）|
|`specs/<capability>/spec.md`|详细的需求和验收场景|**必需**|必须使用 Delta Header + Requirement + Scenario 格式|
|`design.md`|技术实现方案|推荐|无严格格式要求|
|`tasks.md`|实现任务清单|推荐|无严格格式要求|
#### 变更的生命周期

```
提案 (斜杠命令) → 编写规范 → 验证 (validate) → 实现 (apply) → 归档 (archive)
```

1. **提案**：`/opsx:propose <description>`（一步生成所有规划文档）
2. **编写规范**：编辑 proposal.md 和 specs/
3. **验证**：`openspec validate <name>`
4. **实现**：`/opsx:apply` 按照 tasks.md 执行开发
5. **归档**：`/opsx:archive` 将变更中的规范增量（Delta）合并回 `openspec/specs/` 主规范目录，并清理 `openspec/changes/` 下的临时目录，标志着该功能规范已正式「上线」

## 文档结构规范

OpenSpec 强调文档的结构化和规范化，通过明确定义 `proposal.md` 的提案架构和 `spec.md` 的能力场景契约，确保 AI 助手和开发人员能够无歧义地解析需求并生成可靠的代码。**请务必遵循这些格式，否则 `openspec validate` 会失败。**

> **模板文件**：OpenSpec 内置了所有文档模板，可通过 `openspec templates` 命令查看各模板路径，或直接使用 `/opsx:propose` / `/opsx:new` 斜杠命令自动生成完整文档。

### proposal.md - 提案文档 

**核心要求：** proposal.md 必须包含 `## Why` 和 `## What Changes` 两个验证器强制检查的必需章节；推荐包含 `## Capabilities` 章节，作为 AI 自动生成 `specs/<name>/spec.md` 文件的关键输入。

#### 章节要求
OpenSpec 的设计理念是“先想清楚为什么做，再决定做什么，再明确影响哪些能力”：
- `## Why` - 说明变更的背景、问题和动机（**验证器强制检查**）
- `## What Changes` - 说明具体要添加、修改或删除什么（**验证器强制检查**）
- `## Capabilities` - 列出 New / Modified Capabilities，驱动 `specs/<name>/spec.md` 文件的生成（**推荐，AI 工作流所需**）

#### 完整格式模板
> 内置模板路径可通过 `openspec templates` 命令查看；`/opsx:propose` 斜杠命令会自动生成填充好的完整提案。

章节必须结构
```
proposal.md 结构：
├── ## Why 【必需 - 验证器强制检查】
│   ├── ### Background（背景）
│   ├── ### Problem Statement（问题描述）
│   └── ### Alternatives Considered（备选方案）
├── ## What Changes 【必需 - 验证器强制检查】
│   ├── ### New Resources Added（新增资源）
│   └── ### New Capabilities（功能点简述，自然语言概括即可）
├── ## Capabilities 【推荐 - AI 工作流所需，驱动 spec 文件生成】
│   ├── ### New Capabilities（kebab-case 标识符列表，每项对应 specs/<name>/ 目录）
│   └── ### Modified Capabilities（已有能力的 requirement 变更）
├── ## Impact（影响范围）
├── ## Scope（范围，可选）
│   ├── ### In Scope
│   └── ### Out of Scope
├── ## Goals（成功标准，可选）
└── ## References（参考链接，可选）
```
**注意**：章节标题必须完全匹配 `## Why` 和 `## What Changes`（区分大小写）。

### specs/ 目录 - 能力规范

**核心要求：** specs/ 必须使用能力文件夹（capability folders），每个能力一个文件夹。

目录结构
```
specs/
├── accelerator-management/     # 能力一：加速器管理
│   └── spec.md
├── training-job-lifecycle/     # 能力二：训练任务生命周期
│   └── spec.md
├── inference-service/          # 能力三：推理服务
│   └── spec.md
└── relationship-management/    # 能力四：关系管理
    └── spec.md
```

重要规则：
- 不要在 specs/ 根目录直接放置 spec.md 文件
- 每个能力文件夹名称使用 kebab-case
- 文件夹名称应体现能力领域

### spec.md - 能力规范格式

**核心要求：** 必须使用 Delta Header + Requirement + Scenario 格式。

1. 