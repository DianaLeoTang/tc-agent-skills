---
description: 项目 .claude 初始化 — 生成 CLAUDE.md 与 rules/
---

# /tc-init — 项目 .claude 初始化

你是一个项目配置初始化助手。你的任务是在当前项目目录中创建 `.claude/` 文件夹及其完整配置结构。

> 💡 完整工作流：**`/tc-init`（本命令，一次性）** → [`/tc-discuss`](tc-discuss.md)（聊需求）→ [`/tc-prd`](tc-prd.md)（生成 specs）→ [`/tc-ai`](tc-ai.md)（实施）

## 执行步骤

### 0. 增量检查

在分析项目之前，先检查 `.claude/` 文件夹的状态：

- 如果 `.claude/` **不存在**：直接创建，走完整初始化流程
- 如果 `.claude/` **已存在**：执行**增量模式**，只新增缺失的文件，不覆盖已有文件
  - 列出已存在的文件（`CLAUDE.md`、`rules/*.md`）
  - 识别缺失的文件，仅创建这些
  - 如果所有文件都已存在，提示用户无需操作，可选择仅更新 `CLAUDE.md`（如项目技术栈发生变化）

### 1. 分析项目

在生成任何文件之前，先全面分析当前项目：

- 读取 `package.json`、`Cargo.toml`、`go.mod`、`pyproject.toml`、`pom.xml` 等项目描述文件，判断语言和框架
- 扫描目录结构（重点关注 `src/`、`app/`、`lib/`、`tests/`、`migrations/` 等）
- 读取现有的 README、CI 配置、lint 配置、tsconfig 等，提取构建/测试/运行命令
- 识别项目是否包含前端、后端 API、数据库等模块

### 2. 生成文件结构

根据分析结果，生成以下结构（只创建与项目相关的文件）：

```
.claude/
├── CLAUDE.md                    # 项目门面，≤150 行
├── rules/
│   ├── coding-style.md          # 命名/缩进/import/注释规范
│   ├── testing.md               # 测试约定、覆盖率要求
│   ├── security.md              # 禁止事项、密钥处理
│   ├── git-workflow.md          # 分支/commit/PR 规范
│   ├── frontend.md              # (如有前端) paths: src/web/**
│   ├── backend-api.md           # (如有后端 API) paths: src/api/**
│   ├── database.md              # (如有数据库) paths: src/db/**, migrations/**
│   └── smart-contract.md        # (如有合约) paths: contracts/**, src/contracts/**
```

### 3. CLAUDE.md 模板

CLAUDE.md 必须包含以下部分，控制在 150 行以内：

```markdown
# {项目名}

{一句话简介}

## 技术栈

- 语言: {lang}
- 框架: {framework}
- 包管理: {pkg manager}

## 常用命令

- 安装依赖: `{install cmd}`
- 开发运行: `{dev cmd}`
- 构建: `{build cmd}`
- 测试: `{test cmd}`
- Lint: `{lint cmd}`

## 目录结构

{树形结构速览，只列关键目录，不超过 20 行}

## 规则

@rules/coding-style.md
@rules/testing.md
@rules/security.md
@rules/git-workflow.md
{以下按需引入}
@rules/frontend.md
@rules/backend-api.md
@rules/database.md
@rules/smart-contract.md
```

### 4. rules 文件格式

每个 rules 文件使用以下格式：

```markdown
---
description: {规则一句话描述}
globs: {可选，如 "src/web/**"}
---

# {规则标题}

{具体规则内容，从项目实际配置中推断，简洁明了}
```

### 5. 规则内容指引

- **coding-style.md**: 从 eslint/prettier/editorconfig/rustfmt 等配置推断命名风格、缩进、import 排序、注释规范。如无配置则根据语言社区惯例设定。
- **testing.md**: 从测试框架配置和现有测试推断测试规范、文件命名、覆盖率要求。
- **security.md**: 列出禁止硬编码密钥、环境变量处理、敏感文件 .gitignore 规则等。
- **git-workflow.md**: 从 git 历史推断 commit 风格（conventional commits?），分支命名规范，PR 流程。
- **frontend.md**: 组件规范、状态管理、路由约定等（仅当项目有前端时创建）。
- **backend-api.md**: API 设计规范、错误处理、中间件约定等（仅当项目有后端 API 时创建）。
- **database.md**: migration 规范、ORM 约定、查询规范等（仅当项目有数据库时创建）。
- **smart-contract.md**: 合约安全规范、常见漏洞防范（重入攻击、整数溢出、权限控制）、审计检查清单、测试要求、部署流程等（仅当项目有智能合约时创建，检测 contracts/、hardhat.config、foundry.toml、truffle-config、anchor.toml 等）。

## 重要约束

- **增量模式**：`.claude/` 已存在时，只新增缺失的文件，**不覆盖**任何已有文件
  - 如果用户明确要求更新某个已有文件（如技术栈变更需更新 `CLAUDE.md`），可以单独更新
  - 多人共用 `.claude/` 文件夹开发不同需求，每个人只新增自己需要的部分
- 所有规则内容必须基于项目实际情况推断，不要生成空洞的通用规则
- CLAUDE.md 严格控制在 150 行以内
- 只创建与项目实际相关的 rules 文件，不要创建不适用的文件
- 生成完成后，列出所有创建的文件并给出简要说明
