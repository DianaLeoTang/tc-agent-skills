---
description: 项目 .claude 初始化 — 生成 CLAUDE.md 与 rules/
---

# /tc-init — 项目 .claude 初始化（增量友好）

你是一个项目配置初始化助手。你的任务是为当前项目创建/补全 `.claude/` 文件夹的配置结构。

> 💡 完整工作流：**`/tc-init`（本命令）** → [`/tc-test-setup`](tc-test-setup.md)（项目零测试设施时）→ [`/tc-discuss`](tc-discuss.md)（聊需求）→ [`/tc-prd`](tc-prd.md)（生成 specs）→ [`/tc-ai`](tc-ai.md)（实施）→ [`/tc-test`](tc-test.md)（随时跑测试）

## 🛡 核心原则：增量处理，绝不覆盖

`.claude/` 文件夹是项目级的多方共享空间，可能已经有：
- 其它 Claude Code 插件 / Skill 写入的内容（如 `commands/`, `skills/`, `agents/`）
- 团队成员之前手写的规则
- IDE / 工具集成配置（如 `settings.json`, `settings.local.json`, `launch.json`）

**绝对不允许覆盖任何已存在的文件**。本命令只做：

| 现有状态 | 本命令的行为 |
|---|---|
| 文件**不存在** | ✅ 直接创建（标准目标） |
| 文件**已存在且内容看起来合理** | ⏭ **跳过**，输出"已存在，跳过 {path}" |
| 文件**已存在但内容明显与本命令的目标格式冲突** | ⚠️ 输出 diff 给用户看，**不修改**，让用户自行合并 |
| 不属于本命令产出的文件（commands/、skills/、agents/、*.json）| 🚫 完全不碰 |

任何「**不知道该不该改**」的情况 → **跳过 + 提示用户**，不要自作主张。

## 执行步骤

### 0. 扫描现有 `.claude/` 状态（**必跑，不可跳过**）

在分析项目之前，先检查 `.claude/` 文件夹的状态：

- 如果 `.claude/` **不存在** → 直接创建，走完整初始化流程
- 如果 `.claude/` **已存在** → 执行**增量模式**，只新增缺失的文件，**默认不改**任何已有文件：
  1. 列出所有已有文件 / 子目录（含其它插件可能写入的）
  2. 对每个**本命令计划生成**的目标文件，标记状态：
     - ✅ **缺失** → 待创建
     - ⏭ **已存在** → 跳过（默认行为，除非用户**显式要求**更新）
     - ⚠️ **存在但格式异常** → 仅生成 diff 报告给用户，**不自行修改**
  3. 输出一份「现有状态清单」给用户看，例如：

```text
.claude/ 现有内容扫描：

✅ 待创建：
  - CLAUDE.md
  - rules/coding-style.md

⏭ 已存在，跳过：
  - rules/testing.md（4.2KB，最后修改 3 天前）

🚫 非本命令管辖，完全不碰：
  - commands/ (其它插件)
  - skills/ (其它插件)
  - settings.json
  - settings.local.json

⚠️ 待人工合并（已存在但与目标格式冲突）：
  - 无
```

### 1. 分析项目

在生成任何文件之前，先全面分析当前项目：

- 读取 `package.json`、`Cargo.toml`、`go.mod`、`pyproject.toml`、`pom.xml` 等项目描述文件，判断语言和框架
- 扫描目录结构（重点关注 `src/`、`app/`、`lib/`、`tests/`、`migrations/` 等）
- 读取现有的 README、CI 配置、lint 配置、tsconfig 等，提取构建/测试/运行命令
- 识别项目是否包含前端、后端 API、数据库等模块

### 2. 增量生成文件结构

**只创建 Step 0 标为「✅ 待创建」的文件**，已存在的一律跳过。完整目标结构如下（仅供对比参考）：

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

⚠️ **特殊情况：CLAUDE.md 已存在**

`CLAUDE.md` 是高频被多方修改的文件（IDE 插件 / 团队成员 / 历史 init）。本命令处理逻辑：

1. **默认不改**，输出"已存在，跳过"
2. 如果用户**显式要求**（如"检查 CLAUDE.md 是否完整"、"技术栈变了，更新一下 CLAUDE.md"）：
   - 先生成 **diff 报告**（缺哪些章节 / 建议补充内容）给用户确认
   - 用户确认后才允许在文件内做**增量修改**（追加缺失章节，**不覆盖**已有内容）
3. 用户未明确授权时，diff 报告仅以 markdown 形式输出给用户复制粘贴，**不直接修改文件**

⚠️ **特殊情况：rules/ 下某些文件已存在但部分缺失**

举例：项目里有 `rules/coding-style.md` 但没有 `rules/testing.md`：

1. 不动 `coding-style.md`
2. 创建 `rules/testing.md`
3. 不修改 `CLAUDE.md` 的 `@rules/...` 引用列表（避免覆盖）；改为输出建议给用户：「请手动在 CLAUDE.md 「## 规则」节加 `@rules/testing.md`」

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
- **testing.md**: **以项目实际使用的测试库为准**（Jest/Vitest/Playwright/Pytest/Go test/Foundry/Hardhat 等），从测试框架配置和现有测试推断测试规范、文件命名、覆盖率要求。**禁止套用通用模板或硬塞项目未使用的测试库**；项目混用多套时，按目录/模块分别说明。
- **security.md**: 列出禁止硬编码密钥、环境变量处理、敏感文件 .gitignore 规则等。
- **git-workflow.md**: 从 git 历史推断 commit 风格（conventional commits?），分支命名规范，PR 流程。
- **frontend.md**: 组件规范、状态管理、路由约定等（仅当项目有前端时创建）。
- **backend-api.md**: API 设计规范、错误处理、中间件约定等（仅当项目有后端 API 时创建）。
- **database.md**: migration 规范、ORM 约定、查询规范等（仅当项目有数据库时创建）。
- **smart-contract.md**: 合约安全规范、常见漏洞防范（重入攻击、整数溢出、权限控制）、审计检查清单、测试要求、部署流程等（仅当项目有智能合约时创建，检测 contracts/、hardhat.config、foundry.toml、truffle-config、anchor.toml 等）。

## 重要约束

### 🛡 增量处理（核心原则，不可绕过）

**一律做增量，不做覆盖** —— 无论是 `.claude/` 整体结构还是单个文件内部。

- ✅ 只创建 **Step 0 扫描后标为「✅ 待创建」** 的文件
- ✅ **用户显式要求**更新某个已有文件（如技术栈变更需更新 `CLAUDE.md`）时，可以单独更新；更新方式仍是**增量**（追加缺失内容，不覆盖已有）
- ❌ **绝对不覆盖**任何已存在文件（即使内容看起来"陈旧"或"过时"）
- ❌ **绝对不删除** `.claude/` 下任何现有文件 / 子目录
- ❌ **绝对不修改配置文件** —— `commands/`、`skills/`、`agents/`、`settings.json`、`settings.local.json`、`launch.json` 等非本命令产物完全不碰
- ⚠️ 已存在但格式异常的文件 → 输出 diff 报告**给用户**，**不自行修改**
- ⚠️ 任何不确定情况 → **跳过 + 输出提示**，不要自作主张
- 👥 多人共用 `.claude/` 文件夹开发不同需求，每个人只新增自己需要的部分

### 内容质量约束

- 所有规则内容必须基于项目实际情况推断，不要生成空洞的通用规则
- CLAUDE.md 严格控制在 150 行以内（仅当 CLAUDE.md 是本次新建时生效）
- 只创建与项目实际相关的 rules 文件，不要创建不适用的文件

### 输出约束

- 生成完成后，**分三类列出**：
  - ✅ 本次新建的文件
  - ⏭ 跳过的已存在文件
  - ⚠️ 建议用户手动处理的项（diff 报告 / 待补 @rules 引用 等）

## 示例输出

```text
🎯 tc-init 完成

✅ 本次新建（3 个）：
  - .claude/CLAUDE.md (148 行)
  - .claude/rules/testing.md
  - .claude/rules/git-workflow.md

⏭ 跳过已存在（2 个）：
  - .claude/rules/coding-style.md
  - .claude/rules/security.md

⚠️ 建议手动处理（1 项）：
  - 现有 .claude/rules/coding-style.md 没有 frontmatter（description / globs），建议补全
    （diff 见下方）

🚫 完全未触碰（4 个）：
  - .claude/commands/   （其它插件）
  - .claude/skills/     （其它插件）
  - .claude/settings.local.json
  - .claude/launch.json

下一步：跑 /tc-discuss 开始第一个 feature 需求澄清
```
