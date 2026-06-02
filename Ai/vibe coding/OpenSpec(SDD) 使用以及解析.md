>[!INFO] 引言
>	llm大语言模型，对于上下文十分敏感。过长的无关内容和命令会导致Llm出现严重的幻觉，十分影响llm效率和能力。因此为了提高实践的效果，增加llm代码生成的效率，采用 规范驱动开发（SSD）这条路径。
>	规范驱动开发： 主要是将产品功能和代码采用一定的规范联系起来。

**基础使用文档：** [OpenSpec实践](https://forceinjection.github.io/OpenSpec-practise/)
**实践：** [实践](https://forceinjection.github.io/OpenSpec-practise/docs/openspec-practical-guide.html)
**最佳实践：** [深度实践指南](https://forceinjection.github.io/OpenSpec-practise/docs/openspec-ai-workflow-analysis.html)
# OpenSpec 实践

## 安装

1. 安装OpenSpec `npm install -g @fission-ai/openspec@latest`
2. 命令速查

| 命令                                | 说明                                                    | 示例                                             |
| --------------------------------- | ----------------------------------------------------- | ---------------------------------------------- |
| `openspec init`                   | 初始化 OpenSpec 项目                                       | `openspec init --tools qoder`                  |
| `openspec new change <name>`      | 仅创建变更目录结构                                             | `openspec new change add-user-auth`            |
| `openspec update`                 | 更新 AI 技能和命令文件                                         | `openspec update`                              |
| `openspec view`                   | 打开终端交互界面                                              | `openspec view`                                |
| `openspec status --change <name>` | 查看变更状态                                                | `openspec status --change user-auth`           |
| `openspec validate <name>`        | 验证变更文档格式                                              | `openspec validate user-auth`                  |
| `openspec list --changes`         | 列出所有变更                                                | `openspec list --changes`                      |
| `openspec list --specs`           | 列出所有规范                                                | `openspec list --specs`                        |
| `openspec show <name>`            | 显示变更详情                                                | `openspec show user-auth --json --deltas-only` |
| `openspec archive <name>`         | 归档已完成的变更（将 Delta 合并至 `specs/` 主目录并清理 `changes/` 临时目录） | `openspec archive user-auth`                   |
| `openspec config list`            | 查看当前配置                                                | `openspec config list`                         |
| `openspec config profile`         | 设置工作流 Profile                                         | `openspec config profile`                      |
| `openspec templates`              | 查看内置文档模板的绝对路径                                         | `openspec templates`                           |
| `openspec schemas`                | 列出可用 Schema                                           | `openspec schemas`                             |
| `openspec --version`              | 查看版本号                                                 | `openspec --version`                           |
| `openspec --help`                 | 查看帮助信息                                                | `openspec --help`                              |


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

``` yaml
schema: spec-driven

context: |
  Tech stack: TypeScript, React, Node.js
  Testing: Jest with React Testing Library
  API: RESTful, documented in docs/api.md
  We maintain backwards compatibility for all public APIs

rules:
  proposal:
    - Include rollback plan for risky changes
  specs:
    - Use Given/When/Then format for scenarios
  design:
    - Include sequence diagrams for complex flows
  tasks:
    - Break tasks into max 2-hour chunks
```

## 使用流程

### 创建变更提案

1. 使用命令 `/opsx:propose <description>`
	- 根据用户提供的描述推断出kebab-case变更名
	- 创建 `openspec/changes/<name>/`
	- 依次生成 `proposal.md`、`design.md`、`specs/`、`tasks.md` 所有文档
2. 创建变更目录 `/opsx:new <change-name>`
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

> [!TIP] 规约
> 
> 能力包含一个功能闭包（区别于DDD的业务闭包domain，业务闭包比如订单领域可能包含多个能力：创建订单，取消订单等等）。
> 
> 能力应该包含：**业务行为, 业务规则, 状态流转, scenario**
> 能力不应该包含： **实现层的任何细节，比如说创建redis-util等等**
> 
> 划分能力也是关键的一环。
> 
> **一个好的能力具备如下条件**：
> - 是一个 AI 能独立理解、独立推理、独立修改、独立测试的业务行为闭环。
> - 太大的能力会导致一点点小改动就需要计算大量token，太小的能力会导致对全局把控不足。
> - 一个功能提案不应该涉及过多的能力改造，一般3个能力改造为区分界限。最好每次就改一个能力。
> 
> ***能力划分方法论：***
> 1. 正交分解法
> 	- 确保能力之间是“正交”的，也就是说，在**业务纵向**和**技术横向**上不要产生交叉
> 	- **业务轴（纵向）**：商品、购物车、订单
> 	- **技术轴（横向）**：全局日志、错误处理、基础鉴权
> 	- **OpenSpec 规约**：Capability 目录应该只留给**纵向业务**，横向技术规范全部剥离去全局配置文件或底层的 `project.md` 中
> 2. 状态机法
> 	- 该能力应该有独立的生命周期
> 	- 比如说订单有 订单创建，订单支付，订单完成，订单取消等状态，那么订单管理就可以是一个能力
> 3. 用户行为闭环法
> 	- 用户 `输入 -> 状态变化 -> 输出` 这个链条是完整的
> 	- 比如说点击结算： → 校验库存  → 计算价格  → 创建订单  → 返回支付页 这个流程可以看作一个能力
> 
> ***总结***
> 	能力可以分为***原子能力***（比如订单，库存） 和  ***流程能力***（比如下单）这两类。作为能力划分的基本原则。
> 	***原子能力***负责“单个领域对象的稳定规则”，***流程能力***负责“跨对象协作的业务编排”。
> 



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

1. 格式要点

| 元素           | 格式                                       | 示例                             |
| ------------ | ---------------------------------------- | ------------------------------ |
| Delta Header | `## ADDED/MODIFIED/REMOVED Requirements` | `## ADDED Requirements`        |
| 需求标题         | `### Requirement: <标题>`                  | `### Requirement: GPU 自动发现`    |
| 场景标题         | `#### Scenario: <标题>`                    | `#### Scenario: NVIDIA GPU 发现` |
| 场景内容         | Gherkin 格式                               | `Given/When/Then`              |
**Delta Header 选择说明**：

|Delta Header|适用场景|
|---|---|
|`## ADDED Requirements`|本次变更新增的能力或需求|
|`## MODIFIED Requirements`|对已有规范中某个 Requirement 的修改|
|`## REMOVED Requirements`|明确废弃或删除的需求|

2. 完整格式模板
```
spec.md 结构：
├── # 能力名称
├── ## Overview（概述，推荐）
│   - 能力简介
│   - 解决的问题
└── ## ADDED/MODIFIED/REMOVED Requirements 【必需】
    ├── ### Requirement: <标题>
    │   ├── **Priority**: P0/P1/P2
    │   ├── **Rationale**: ...
    │   └── #### Scenario: <标题>
    │       └── Given/When/Then
```

3. 示例
> 以下示例展示核心 Requirement + Scenario 结构。完整示例（含 `## Overview` 段落）参见 `examples/openspec/changes/v1-mvp/specs/domain-model/spec.md`（电商领域模型规范）
```
## ADDED Requirements

### Requirement: 商品实体定义

系统 SHALL 定义商品实体，包含唯一标识、名称、价格和库存。

**Priority**: P0 (Critical)

**Rationale**: 商品是电商系统的核心实体，是所有交易的基础。

#### Scenario: 创建有效商品

Given 需要创建新商品
When 提供商品信息 { id, name, priceCents, stock }
Then 商品实体创建成功
And id 格式为 prod_xxxx
And priceCents >= 0
```

### design.md - 技术设计

技术设计文档没有严格的格式要求，但建议包含以下章节。

| 章节名称                  | 建议内容                                   |
| --------------------- | -------------------------------------- |
| Architecture Overview | 系统整体架构图（建议使用 Mermaid 或 ASCII 图）及层次关系说明 |
| Core Components       | 核心模块列表，每个模块的职责、边界和内部实现要点               |
| Data Model            | 关键实体的字段定义、类型、约束及实体间关系                  |
| API Design            | 接口路由、请求/响应格式、错误码规范                     |
| Integration Patterns  | 与外部系统/模块的集成方式，包括事件、队列、同步调用等            |
| Technology Stack      | 所选技术及库、选型理由和备选方案对比                     |
| Security              | 身份认证、权限控制、数据加密、输入校验等安全设计要点             |
| Deployment            | 环境要求、部署步骤、回滚方案                         |


### tasks.md - 任务清单

**建议章节结构**：

- **Milestone**：按里程碑对实现步骤分组（如 M1 基础层、M2 API 层、M3 测试）。每个任务拆小，确保单个任务可在 2 小时内完成。
- **Definition of Done**：列出此里程碑的完成标准，如代码通过 CI、测试覆盖率达标、spec validate 通过等。
- **Progress Tracking**：利用 `- [x]` / `- [ ]` 标记完成进度，方便 IDE 内直观查看。

**示例：**
```
## Milestone 1 - Domain Model

### Definition of Done

- 完成所有 P0 Requirement 的实现
- `openspec validate v1-mvp` 验证通过
- 单元测试覆盖所有领域实体

### Tasks

- [x] 定义 Product 实体类型（id、name、priceCents、stock）
- [x] 定义 Cart / CartItem 实体类型
- [ ] 定义 Order / OrderItem 实体类型
- [ ] 实现领域实体的编排验证逻辑

## Milestone 2 - Service Layer

### Definition of Done

- 所有服务方法均有对应集成测试

### Tasks

- [ ] 实现 CatalogService.getProduct / listProducts
- [ ] 实现 CartService.addItem / removeItem
- [ ] 实现 OrderService.checkout
```

## 推荐做法

### 推荐实践
- **一个能力一个文件夹**：按功能领域划分能力
- **需求粒度适中**：每个需求应该是可测试的单一功能点
- **场景具体化**：使用具体的 Gherkin 场景描述行为
- **优先级标注**：为每个需求标注 P0/P1/P2 优先级
- **添加 Rationale**：说明为什么需要这个需求

### 场景编写最佳实践
场景描述是连接业务语言与技术验证的桥梁。采用标准的 Gherkin 语法能够消除歧义，确保每个场景都能转化为明确的可执行测试。

#### Gherkin 格式要点

|关键字|用途|示例|
|---|---|---|
|`Given`|前置条件，描述系统初始状态|`Given 用户已登录系统`|
|`When`|触发动作|`When 用户点击"提交订单"按钮`|
|`Then`|预期结果|`Then 订单状态变为"待支付"`|
|`And`|连接多个条件或结果|`And 用户收到订单确认邮件`|

#### 好的场景示例

```
Scenario: 使用信用卡支付订单

Given 用户已登录系统
And 购物车中有 2 件商品，总价 299 元
And 用户已绑定信用卡
When 用户选择"信用卡支付"并确认
Then 订单创建成功
And 从信用卡扣除 299 元
And 用户收到支付成功通知
And 库存减少 2 件
```

#### 不好的场景示例

```
Scenario: 支付

Given 系统
When 支付
Then 成功
```

**问题**：

- 太模糊，无法验证
- 缺少具体的前置条件
- 没有明确的预期结果

### 迭代开发最佳实践

规范驱动并非僵化的瀑布流，而是拥抱变化的增量过程。在迭代开发中，保持文档与代码的同步更新，是维持系统一致性的关键。

- **增量添加**：可以随时添加新的需求到变更中
- **频繁验证**：使用 `openspec validate` 确保格式正确
- **版本控制**：将 OpenSpec 文档纳入 Git 管理
- **及时归档**：完成开发后使用 `openspec archive` 归档变更
- **存量项目（Brownfield）优先从小处入手**：对于已有历史代码的项目，建议从一个小的、相对独立的功能开始创建第一个 Change，逐步建立规范体系，不要试图一次性为所有旧代码补规范

### 与 AI 协作最佳实践

充分发挥大模型潜力的关键在于合理利用工作流指令和上下文管理。通过结构化的交互模式，可以有效降低 AI 的幻觉并提升代码生成质量。

#### OPSX 斜杠命令（Slash Commands，推荐）

OpenSpec 1.0+ 引入了全新的 OPSX 工作流，替换了旧版的阶段锁定模式。所有命令均通过 `openspec init` 安装到 AI 工具对应目录。

**默认 Core 配置（常用 4 个命令）**:

|命令|作用|
|---|---|
|`/opsx:propose <description>`|一步创建变更并**智能生成**所有规划文档（AI 基于描述自动推断 kebab-case 目录名并填充 proposal/design/specs/tasks）|
|`/opsx:explore`|进入探索模式，思考问题、调查代码库，不写代码|
|`/opsx:apply`|按照 tasks.md 实现任务|
|`/opsx:archive`|完成并归档当前变更|

**扩展工作流命令（通过 `openspec config profile` 开启）**

|命令|作用|
|---|---|
|`/opsx:new`|仅初始化变更目录结构，不创建文档|
|`/opsx:continue`|按依赖顺序创建下一个文档（逐步模式）|
|`/opsx:ff`|快进生成所有规划文档（一步到位）|
|`/opsx:verify`|验证实现是否与规范一致|
|`/opsx:sync`|将 Delta Spec 合并到主规范（不归档）|
|`/opsx:bulk-archive`|批量归档多个已完成的变更|
|`/opsx:onboard`|带教 15 分钟全流程引导，适合新手上手|

#### 与 AI 协作的技巧

1. **先探索后提案**：不确定时先用 `/opsx:explore` 思考，明确后再 `/opsx:propose`
2. **支持流动迭代**：实现过程发现设计错误？直接编辑对应文档即可，无阶段锁定
3. **定期清理对话上下文**：开始实现任务前，建议清空当前对话上下文，确保高质量的指令注入效果
4. **增量迭代**：完成一个需求后验证，再进行下一个

### 团队协作最佳实践

将 OpenSpec 融入团队现有的研发流程中，需要建立配套的审查机制与文档维护习惯，从而确保规范体系在长期协作中不被破坏。

#### 代码审查清单

在 PR 审查时，检查 OpenSpec 文档：

- [ ] proposal.md 有清晰的 Why 和 What
- [ ] 每个 Requirement 都有至少一个 Scenario
- [ ] Scenario 使用标准的 Gherkin 格式
- [ ] 优先级标注合理
- [ ] 没有遗漏重要的边界场景

#### 文档维护

- **保持更新**：实现过程中如果发现规范需要调整，及时更新文档
- **同步修改**：如果需求变更，先更新 spec.md 再修改代码
- **归档记录**：归档的变更应保留历史记录，便于追溯

# OpenSpec 拆解

>[!tip] Think
> 我在使用ai开发的时候，发现ai工具无法将**需求和代码很好的结合起来**。
> 导致每次我询问相关需求实现，他就会提取出需求的关键词之后在全局模糊搜索，这样的不仅效率很低，而且很多时候会加载无关信息导致出现严重的幻觉。
> 了解到OpenSpec工具可以将需求和代码结合起来形成规范，还可以随着需求的演进不断更新文档，看起来可以解决这个问题。
> OpenSpec 及觉得

>[!question] 工具局限性
>OpenSpec 确实可以解决上述问题，但是他对项目的架构分析等等文档都需要用户自己维护，关于这个方面应该引入ai互动模式和用户不断交互完善该文档。

