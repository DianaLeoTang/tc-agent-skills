---
description: 跑测试 — 自动识别项目测试栈，跑现有测试套件，输出结构化报告（不设计新用例）
---

# /tc-test — 跑测试

随时跑项目的现有测试套件。**不设计新用例、不修改业务代码**——只是「按下按钮、跑、报告结果」。

## 核心原则（铁律）

1. **不硬编码项目形态**：项目可能是纯前端 / 纯后端 / 全栈 / 纯 AI / 合约 / 库 / CLI。**探测出什么就说什么**，输出里只出现项目里真实存在的栈名（如 "Python 后端"、"React 前端"、"Go 服务"、"Solidity 合约"、"AI Agent (Python + Anthropic SDK)"）。**禁止**在没探测前就预设"先跑后端再跑前端"。
2. **环境不就绪必须强制拦截**：依赖未装、虚拟环境未激活、解释器路径错、关键 binary 缺失 —— 任一项立即停止并给修复命令。**绝不允许**"先试一下""容错跑""跳过"环境问题。环境失败后跑出来的测试是无用测试，会让上层（如 `tc-qa-engineer`）基于错误信号写出错误用例 —— 这种污染绝不容忍。
3. **只跑不改**：不动业务代码、不动测试代码、不装依赖、不创建配置、不写示例。

> 💡 与 [`tc-qa-engineer`](../skills/tc-qa-engineer/SKILL.md) skill 的区别：
> - `tc-qa-engineer`：**设计** 新的测试用例（基于 requirements.md AC-XXX）+ 跑测 + 形成 bug 闭环 — 由 `/tc-ai` 的 N6 节点自动触发
> - `/tc-test`（本命令）：**只跑** 现有测试、结构化报告 — 由你随时手动调用

## 何时用

- 改完代码想验证「没破现有测试」
- PR 提交前最后一道闸
- 上线前跑完整联调
- 开发中想 watch 模式实时反馈

## 何时不用

- 想**新增**测试用例 → 用 `tc-qa-engineer` skill（在 `/tc-ai` 流程里自动调）
- 测试基础设施还没搭 → 跑 `/tc-test-setup`（按项目栈推荐方案 + 用户确认 + 装依赖 + hello world）

## 「零测试设施」项目的正确路径

如果项目**完全没有测试框架 / 没有测试代码**，本命令**不会自动安装依赖、不会写示例测试**——这是有意的设计（避免污染你的项目结构）。

按项目状态分两档：

### 档位 1：完全没有测试基础设施（无框架、无配置、无测试代码）

```bash
/tc-test-setup              # 按项目实际栈推荐方案 → 用户确认 → 装依赖 + 写配置 + 建目录 + hello world
   ↓
/tc-test                    # 现在能跑了（hello world 通过）
   ↓
# 接着开发新需求时，业务测试由 /tc-ai 的 N6 节点调 tc-qa-engineer 设计
/tc-discuss → /tc-prd → /tc-ai
```

### 档位 2：测试框架装了，只是没写测试代码

```bash
# 直接用 /tc-ai，N6 节点会调 tc-qa-engineer 基于已有 specs 的 AC-XXX 设计测试：
/tc-ai <项目路径>
   ├─ N3 走任务（如果没新代码要写，可空 task）
   └─ N6 必触发 tc-qa-engineer（feature 完成 = must-trigger 条件）
        └─ 自动设计单元 + 集成 + 端到端用例 → 落到 *.test.* 文件
   
/tc-test                    # 然后跑
```

> 💡 **历史项目**（已有大量历史代码、无测试覆盖、想补回归测试）→ 使用 `/tc-test-legacy`（独立命令，与本流程无关）。

### 反模式（**禁止**自动做的事）

- ❌ 不要自己 `npm install` / `pip install` / `cargo build` / `forge install` —— 会改变项目依赖图，违反"只跑不修"原则
- ❌ 不要自己创建 `*.test.*` / `test_*.*` 示例文件 —— 测试用例必须基于 specs 设计
- ❌ 不要自己创建测试配置文件（`vitest.config.ts` / `pytest.ini` / `playwright.config.ts` 等）—— 配置选项是技术决策，应在 `/tc-prd` 的 specs 步骤里让用户确认
- ❌ 不要"跳过"缺失层后假装成功 —— 必须明确输出"跳过 X 因为 Y"
- ❌ **环境未就绪时不要"试一下"** —— 见 Step 0 的强制拦截清单

## 输入参数

`$ARGUMENTS` 灵活，下面格式都接受。**注意：模式语义按项目实际栈解析**——例如 `online` 在没有真后端的纯前端库项目里不适用，会直接报错而不是硬跑。

```bash
# 通用模式（按项目栈映射到具体测试层）
/tc-test                    # 默认 = 跑所有"快测层"（单元 + 组件 + 离线集成；不跑 e2e/contract 等需要真服务的层）
/tc-test fast               # = 默认（明示）
/tc-test full / all         # 全量 = 快测层 + 联调层（contract / e2e / 真服务集成）

# 按层名跑（如果该层在项目里存在）
/tc-test unit               # 仅单元测试
/tc-test integration        # 仅集成测试
/tc-test contract           # 仅契约测试（需要真服务）
/tc-test e2e                # 仅端到端（需要真服务 + 真客户端）

# 按"项目组件"跑（按 Step 1 探测出的组件名）
/tc-test backend            # 仅"后端组件"（如有）—— 由探测结果定义，不预设
/tc-test frontend           # 仅"前端组件"（如有）
/tc-test agent              # 仅"AI agent 组件"（如有）
/tc-test contracts          # 仅"合约组件"（如有）
# ↑ 不存在的组件名直接报错"探测中未识别到该组件"

# 修饰符
/tc-test --watch            # watch 模式（仅在支持 watch 的栈上有效，如 vitest/jest/pytest-watch）
/tc-test --coverage         # 强制带覆盖率报告
/tc-test feature 2          # 只跑与 feature 2 相关的测试（grep [F-XXX] 或 [AC-XXX] 标签）+ 报告归属 feature 2
/tc-test --no-fail-fast     # 不要在某层失败后停止，全跑完一起报
/tc-test --report-dir <path> # 强制指定报告落盘目录（逃生舱：仅在项目无 specs 目录时使用）

# 组合
/tc-test full --coverage
/tc-test backend feature 2
```

## 流程

### Step 0: 环境健康检查（强制拦截）

**这一步是铁门**。任何一项失败，立即停止 + 给具体修复命令 + **不允许继续到 Step 1**。

> 💡 为什么放在 Step 1 之前：探测项目结构本身依赖工具可用（`node` / `python` / `go` / `forge` 等都得能跑）。环境烂掉的项目里，探测结果也不可信。

#### 0.0 前置豁免：纯 prompt / 规则 / 文档仓库

某些仓库**根本没有运行时代码**，只承载 prompt / 配置 / 文档（如 Claude Code 插件、Cursor rules、agent 提示词集合、纯 markdown 知识库）。这类仓库**不该走 `/tc-test` 通用流程，也不该被引导到 `/tc-test-setup`**——它们没有可执行代码可测，搭测试基建毫无意义。

**判定为豁免对象**（满足以下**任一**且根目录**没有任何运行时项目描述文件** —— 即 Step 0.1 表里的所有 `package.json` / `pyproject.toml` / `go.mod` / `Cargo.toml` / `foundry.toml` / `pubspec.yaml` / `Gemfile` / `composer.json` / `pom.xml` / `build.gradle` 都不存在）：

| 标志 | 仓库类型 |
|---|---|
| `.claude-plugin/plugin.json` 存在（根目录或子目录） | Claude Code 插件仓库 |
| `commands/*.md` + `skills/*/SKILL.md` 是主要内容 | Claude Code 插件仓库（无 manifest 但目录约定齐） |
| 根目录 `.cursorrules` / `.cursor/rules/` | Cursor 规则仓库 |
| `.continue/` 配置目录 | Continue 规则仓库 |
| `.github/copilot-instructions.md` 是主要内容 | Copilot 规则仓库 |
| 根目录 `mcp.json` / `mcp_config.json` 但**无任何业务代码** | 纯 MCP 配置仓库 |
| `**/*.md` 占源文件 >80% 且无运行时项目描述 | 纯 markdown 知识库 |

> ⚠️ **同时拥有运行时代码** 的混合仓库（如插件 manifest + TypeScript 源码 + `package.json`）**不豁免**，正常走 Step 0.1。

**豁免输出**（不进入 0.1，直接结束）：

```text
🔍 检测到这是 Prompt / 规则 / 文档仓库（无运行时代码）：
   类型: Claude Code 插件仓库
   入口标志: .claude-plugin/plugin.json
   主要内容: commands/*.md + skills/*/SKILL.md
   
ℹ️ /tc-test 不适用 —— 项目没有可执行代码可测。
ℹ️ /tc-test-setup 也不适用 —— 没有运行时栈可搭测试基建。

🤔 这类仓库的"测试"应是验证**内容质量**，推荐工具（**本命令不会自动跑**）：
   ✓ markdown-link-check / lychee     验证内部 / 外部链接能跳到
   ✓ markdownlint                     markdown 格式规范
   ✓ prettier --check "**/*.md"       格式一致性
   ✓ 自定义脚本                       校验 frontmatter 字段（name/description/type 等）完整性、
                                      命令与 Skill 互相引用的链接是否还能 resolve

📌 真正的"端到端验证"是**人工演练**：
   在 Claude Code 里实际跑一遍你写的 /tc-init / /tc-test-setup / /tc-ai 等命令，
   看 Claude 是否按 prompt 的预期行为响应。这不是 /tc-test 的职责范围。

⛔ /tc-test 在此结束，无需任何后续步骤。
```

满足豁免 → **STOP**，不进入 0.1。不满足 → 继续 0.1。

#### 0.1 项目语言/栈识别（粗判）

扫描根目录与一级子目录，识别有哪些"项目描述文件"：

| 描述文件 | 暗示的栈 |
|---|---|
| `package.json` | Node.js / 前端 / TypeScript |
| `requirements.txt` / `pyproject.toml` / `Pipfile` / `setup.py` | Python |
| `go.mod` | Go |
| `Cargo.toml` | Rust |
| `foundry.toml` / `hardhat.config.*` | Solidity |
| `pubspec.yaml` | Dart / Flutter |
| `Gemfile` | Ruby |
| `composer.json` | PHP |
| `pom.xml` / `build.gradle` | Java / Kotlin |
| `mcp_config.json` / `agent_*.yml` / `dify_*.yml` | AI Agent / LLM 工作流 |

允许多栈共存（全栈项目、polyrepo monorepo）。**至少识别到一个**才能进入 0.2。

#### 0.2 环境健康检查（按识别出的栈逐项检）

对每个识别出的栈，按对应清单检查。**任一项 ❌ 即拦截**：

##### Node / 前端栈
- [ ] `node` 可执行 + 版本满足 `package.json` engines（如有）
- [ ] `node_modules/` 存在且非空
- [ ] `package.json` 与 `package-lock.json` / `yarn.lock` / `pnpm-lock.yaml` 一致（不强制对比 hash，但要有 lock 文件且 mtime 不早于 package.json）
- [ ] `package.json` scripts 里有目标 test 命令（如要跑 `npm test`，对应 script 必须存在）
- [ ] 测试框架的 binary 能 resolve（`./node_modules/.bin/vitest` 或 `./node_modules/.bin/jest` 或 `./node_modules/.bin/playwright` 等存在）

##### Python 栈
- [ ] **虚拟环境已激活**（`which python` 指向项目 `.venv` / `venv` / conda env，**不是系统 python**）
- [ ] `python --version` 满足项目要求（如 `pyproject.toml` 的 `requires-python`）
- [ ] **关键测试包能 import**：`python -c "import pytest"`（或 unittest / nose / 项目实际用的）退出码 0
- [ ] 业务关键包能 import（粗筛：试 import `requirements.txt` 头几个非测试包，证明依赖已装齐 —— 至少 `fastapi` / `django` / `anthropic` / 项目主框架要 import 得到）
- [ ] 测试配置存在（`pytest.ini` / `pyproject.toml [tool.pytest]` / `setup.cfg` 之一）

##### Go 栈
- [ ] `go version` 可用
- [ ] `go mod download` 或 `go mod verify` 通过
- [ ] `go test` 能找到测试包（`go list ./...` 不报错）

##### Rust 栈
- [ ] `cargo --version` 可用
- [ ] `Cargo.lock` 存在
- [ ] `cargo test --no-run` 能编译通过

##### Solidity 栈
- [ ] `forge --version` / `npx hardhat --version` 可用
- [ ] 依赖已下载（`lib/` for foundry / `node_modules/` for hardhat）

##### AI Agent / LLM 栈（特殊）
- [ ] **API key 已配置**（环境变量 `ANTHROPIC_API_KEY` / `OPENAI_API_KEY` 等已设置；**不要打印 key 值**，只检查存在性）
- [ ] 测试模式是否走真 API：如果是，必须明确告知用户"将消耗真实 token 调用"并询问继续；如果是 mock 模式，验证 mock 框架到位
- [ ] MCP 配置可加载（如有）

##### 跨栈通用
- [ ] 测试要用的端口未被占用（如 e2e 需要 backend 在 8000 端口，先 `lsof -i :8000` 确认）
- [ ] 测试数据库 / fixture 文件可访问（如 `test.db` 路径写、tmp 目录可用）

#### 0.3 拦截输出格式（强制）

任一检查失败，按下例格式输出并 **STOP**：

```text
⛔ 环境检查未通过，停止跑测试

❌ Python 后端 (backend/)
   - 虚拟环境未激活：which python 指向 /usr/bin/python（系统 python），
     而不是项目的 backend/.venv/bin/python
   
🔧 修复命令：
   cd backend && source .venv/bin/activate
   # 或如果 venv 不存在：
   cd backend && python -m venv .venv && source .venv/bin/activate && pip install -r requirements.txt

❌ React 前端 (frontend/)
   - node_modules/ 不存在
   
🔧 修复命令：
   cd frontend && npm install

📌 修好上面所有问题后，重新跑 /tc-test

⚠️ 重要提示：
   - 我**不会**自动帮你装依赖 / 激活 venv —— 这些是项目状态，由你决定
   - 我**不会**在环境烂掉的情况下硬跑测试 —— 跑出来的"通过 / 失败"都不可信，
     会污染后续 tc-qa-engineer 写出无用测试。
```

#### 0.4 环境检查通过的输出

```text
✅ 环境健康检查通过：
   - Python 后端: venv 已激活, pytest 可用, fastapi 可 import
   - React 前端: node_modules 已装, vitest/playwright 可用
   
进入 Step 1 探测 →
```

### Step 1: 探测项目结构与测试栈

**核心：按识别出的「项目组件」描述，不硬编码"前端/后端"**。

#### 1.1 识别项目组件

根据 Step 0.1 识别的栈 + 实际目录结构，列出本项目的"组件清单"。组件命名按实际功能描述，例如：

- 单一栈项目：组件 = 项目本身（如 `Python CLI 工具` / `React 组件库` / `Go 微服务`）
- 多栈项目：每个独立栈 = 一个组件（如 `Python 后端` / `Vue 前端` / `Solidity 合约` / `AI Agent`）
- 在子目录的项目：用目录名 + 栈描述（如 `backend/ (Python)` / `frontend/ (React + TS)`）

#### 1.2 对每个组件做三层判断

| 层级 | 判断方法 | 状态 |
|---|---|---|
| L1 测试**框架**装了吗？ | 看依赖：`vitest`/`jest`/`pytest`/`go test` 等是否在 `package.json` / `requirements.txt` / `go.mod` | 装了 ✓ / 未装 ✗ |
| L2 测试**脚本**配了吗？ | 看 `package.json` scripts、`pytest.ini`、`Makefile`、`justfile` 等是否有 test 命令 | 配了 ✓ / 未配 ✗ |
| L3 测试**代码**有吗？ | grep 测试文件：`**/*.test.{ts,tsx,js}` / `**/test_*.py` / `**/*_test.go` / `**/*.t.sol` 等 | 有 ✓ / 无 ✗ |

#### 1.3 输出探测结果（按状态分类，**所有示例的组件名都按实际项目替换**）

##### 情形 A：完整有 — 所有层都 ✓

```text
🔍 项目探测：
   按实际识别出的组件列出，例：
   
   - Python 后端 (backend/)：✓ pytest + 67 个测试用例
   - React 前端 (frontend/)：✓ vitest + playwright + 96 个单元/组件 + 7 contract + 5 e2e
   
   或者纯单一栈项目：
   - Solidity 合约 (./)：✓ foundry + 23 个测试
   
   或者 AI 项目：
   - AI Agent (agent/)：✓ pytest + 12 个 prompt 测试 + 3 个 MCP 集成测试
   
   测试入口齐全，进入 Step 2 →
```

##### 情形 B：测试框架装了但没测试代码（L1 ✓ / L2 ✓ / L3 ✗）

```text
🔍 项目探测：
   - <按实际栈名描述>: ⚠ 框架已装, 但找不到测试文件

⚠ 测试框架已就绪，但**没有任何测试代码**。

🤔 推荐路径（你选）：
   1) 跑 /tc-ai 让 N6 节点自动调 tc-qa-engineer 基于现有 specs 的 AC-XXX 设计测试
   2) 历史项目想补历史回归测试 → /tc-test-legacy（独立命令）
   3) 如果你已知道要测什么，手动写测试文件后再跑 /tc-test

⛔ 本命令不写新测试，**也不写任何报告文件**（避免空报告污染 specs 目录），停止于此。
```

> 🚫 **绝不在情形 B 下写 test-report.md / coverage-report.md** — 没有真实跑过用例的报告就是噪音，会让用户/CI 误以为"测过了"。

##### 情形 C：完全没有测试框架（L1 ✗）

```text
🔍 项目探测：
   - <按实际栈名描述>: ❌ 未发现测试框架

❌ 项目尚未建立测试基础设施。

🤔 推荐路径（按项目实际栈给推荐）：
   - JS/TS 前端 → 推荐 Vitest + Testing Library + MSW
   - JS/TS 后端 → 推荐 Vitest + supertest
   - Python → 推荐 pytest + httpx
   - Go → go test + testify
   - Rust → cargo test
   - Solidity → Foundry / Hardhat
   - AI Agent → pytest + 真 API 慎用 / promptfoo / deepeval

✋ 建议工作流：
   1) /tc-test-setup           ← 按项目栈推荐方案 + 用户确认 + 装依赖 + hello world
   2) /tc-test                 ← 设施搭好后再回来跑

⛔ 本命令不引入新依赖，**也不写任何报告文件**（避免空报告污染 specs 目录），停止于此。
```

> 🚫 **绝不在情形 C 下写 test-report.md / coverage-report.md** — 业务项目什么测试都没跑，写报告等于伪造证据。

##### 情形 D：部分有部分没（混合）

```text
🔍 项目探测（多组件，部分缺失）：
   - <组件 X>: ✓ 测试齐全
   - <组件 Y>: ❌ 无测试框架

⚠ 部分测试设施缺失。本命令只跑能跑的层，跳过缺失部分。

🎯 实际可跑：
   - <组件 X> 的所有测试

🚫 跳过：<组件 Y>（请按情形 C 路径补全）
```

### Step 1.5: 确定本次跑测对应的 **feature 目录**（决定报告落盘位置）

**核心原则**：测试报告 + 覆盖率报告**收敛到对应需求文件夹下**（`{SPECS_DIR}/N.{feature}/`），不再散落到项目根。归属哪一个 feature 由用户**从列表选**（按编号），不让用户手动打 feature 名。

#### 1.5.1 解析 `SPECS_DIR`

按 `/tc-prd` 的约定查找：

1. `{PROJECT_DIR}/docs/tc-spec/` 优先
2. `{PROJECT_DIR}/doc/tc-spec/` 兼容
3. `{PROJECT_DIR}/specs/` 兼容（旧版本约定）
4. 都没有 → 见 1.5.5 兜底

记为 `SPECS_DIR`。然后扫描 `SPECS_DIR/` 下所有形如 `N.{name}/`（且含 `tasks.md` 或 `requirements.md`）的目录，按编号升序记为 `FEATURE_LIST`。

#### 1.5.2 选定 `FEATURE_DIR`（**始终给用户选择权**）

按以下决策树（**绝不直接默认开跑**）：

##### 情况 A：用户显式传了 `/tc-test feature N`
直接取 `FEATURE_LIST` 里编号为 `N` 的项 → 不再问，直接跑：

```text
🎯 选定 feature: 2.user-login（来自命令参数 feature 2）
   报告路径: docs/tc-spec/2.user-login/test-report.md
```

匹配不到 → 报错列出可用编号，停止。

##### 情况 B：`FEATURE_LIST` 只有 1 个 feature
唯一项，仍然显示给用户确认（**不打断式**：默认就是它，回车继续）：

```text
🎯 项目下只有一个 feature:
   1) 2.user-login

报告将写入: docs/tc-spec/2.user-login/test-report.md
回复 1 或直接回车确认，回复 q 取消
```

##### 情况 C：`FEATURE_LIST` 有多个 feature（典型场景）
**必须列出来让用户按编号选**，不允许默认开跑：

```text
🎯 项目下检测到 5 个 feature，请选择本次跑测对应的 feature:

   1) 1.jiyuan-channel       (tasks.md 改于 12 天前)
   2) 2.user-login           (tasks.md 改于 5 天前)
   3) 3.payment              (tasks.md 改于 2 天前)
   4) 4.dashboard            (tasks.md 改于 1 天前)  ← 推荐 (与当前 git 分支匹配)
   5) 5.notification         (tasks.md 改于刚刚)

请回复编号（1-5），或回复 q 取消。

ℹ 提示: 多 feature 并行开发是反模式。如有多个 feature 都需要跑测，
        建议按编号顺序逐个跑（先选 1，跑完再回来选 2…），
        而不是一次跑所有 feature。
```

**推荐项**的判定（仅作为视觉提示用 ←，**不预选、不默认**）：

1. 当前 git 分支名能抽出 N（如 `feature-4-dashboard` → 4）→ 标推荐
2. 否则取 `tasks.md` mtime 最新的那个 → 标推荐
3. 推荐只是提示，用户**仍然必须显式回复编号**

#### 1.5.3 接受用户回复

| 回复 | 行为 |
|---|---|
| `1` ~ `N`（合法编号）| 选中对应 feature，进入 Step 1.6 |
| 直接回车（仅情况 B 允许）| 等同回复 1 |
| `q` / `cancel` / `取消` | 终止 `/tc-test`，不跑测 |
| 其它 | 提示无效输入 + 重新列出选项 |

#### 1.5.4 选定后输出归属确认

```text
✓ 已选定 feature: 4.dashboard
   报告将写入:
     - docs/tc-spec/4.dashboard/test-report.md      (跑测结果)
     - docs/tc-spec/4.dashboard/coverage-report.md  (覆盖率，仅在跑覆盖率时生成)

进入跑测...
```

#### 1.5.5 兜底：项目根本没建 specs

如果 `SPECS_DIR` 不存在，或 `FEATURE_LIST` 为空 → **不要回退到把报告写到项目根**（这是用户明确反对的行为）。改为：

```text
⚠ 项目尚未通过 /tc-prd 建立 specs 目录，无法确定报告归属。

📌 选择（必须二选一，否则不跑测）:
   1) 跑 /tc-prd 立项后再回来跑 /tc-test（推荐）
   2) 显式临时落盘：/tc-test --report-dir <你想要的目录路径>

⛔ 默认不在项目根写 .tc-test-report.md（避免污染业务项目根）
```

`--report-dir` 是逃生舱，仅在用户明确传入时使用。否则一律拦截。

### Step 1.6: 「特殊情形」中断条件

遇到下列情况立即输出错误并停止（这些是 Step 0 没覆盖的边界情况）：

| 情形 | 输出 |
|---|---|
| 整个项目根目录连任何项目描述文件都没有 | "❌ 不是受支持的项目类型（未识别到 package.json / pyproject.toml / go.mod / Cargo.toml / foundry.toml 等任一）" + 列出根目录前 10 个文件 |
| 项目描述文件存在但 scripts/test 配置完全空 | "⚠ 项目存在但没有任何 test 命令配置（package.json scripts / pytest.ini / Makefile 都没 test 入口）" |
| 用户指定模式（如 `--watch` / `online` / `e2e`）但当前栈不支持该模式 | "❌ 当前项目栈（<实际栈名>）不支持 <模式名>" + 给出可用模式列表 |
| 用户指定的"组件名"不在探测清单里 | "❌ 探测中未识别到组件 <名>，可用组件：<列表>" |

### Step 2: 解析模式

按 `$ARGUMENTS` + Step 1 探测结果决定要跑哪几层。**模式名要映射到项目实际有的层；项目里没这层就直接报错，不要硬跑或猜测**。

| 用户输入 | 跑什么 |
|---|---|
| 空 / `default` / `fast` | 所有"快测层"（不依赖真服务的层：单元、组件、离线集成、snapshot 等）|
| `full` / `all` | 快测层 + 联调层（contract / e2e / 真服务集成）|
| `unit` / `integration` / `contract` / `e2e` | 仅该层（项目里没该层 → 报错）|
| `<组件名>` (如 `backend` / `frontend` / `agent` / `contracts`) | 仅该组件的所有测试（按 Step 1.1 探测出的组件名）|
| `--watch` | watch 模式（仅在支持的栈上）|
| `feature N` | 跑前过滤标签（仅跑包含 `[F-X]` / `[AC-X]` / 文件路径含 `N.{name}` 的测试）|

**输出执行计划**（强制，按实际项目组件描述）：

```text
🎯 执行计划（基于探测结果，按实际栈描述）：
   1️⃣  <组件 A> 单元测试 (vitest)        预计 ~3s
   2️⃣  <组件 B> pytest 测试               预计 ~50s
   3️⃣  <组件 A> 契约测试 (Playwright)    预计 ~10s
   4️⃣  <组件 A> e2e 测试 (Playwright)    预计 ~15s
   预计总耗时：~80s

执行顺序原则：快的先跑（fail-fast 让快测早失败）；联调层依赖快测层全绿。
```

### Step 3: 依次跑

按顺序执行，每完成一层立即输出结果。**任一层失败立即停止**（fail-fast），不继续后面的层 — 因为后面的层往往依赖前面的（如 e2e 依赖单元测试不回归）。

**每层标准输出**（组件名按实际项目替换，**不硬编码"前端/后端"**）：

```text
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🧪 <组件名> <测试层名>
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
命令: <实际跑的 shell 命令>
✅ N passed in Xs
（可选）📊 覆盖率: lines X% / branches X% / functions X%（门槛 X/X/X ✓）
```

**失败时**（同样按实际组件命名）：

```text
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🧪 <组件名> <测试层名>
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
命令: <实际命令>
❌ N failed / M passed

失败用例:
   - <文件名>:<行号> › [AC-XXX] 描述
     断言: ...
     实际: ...
   
🔧 复现命令:
   <最小复现 shell 命令>
   
📂 详细 trace:
   <报告 / trace 文件路径>
   
⛔ 停止后续层，请先修复此失败
```

### Step 4: 总结报告（双输出：终端 + 落到 feature 目录的 markdown 文件）

> ⚠ **前置条件**：必须真的跑过至少一个测试用例（情形 A 或 D）才能进入 Step 4。情形 B / C / 1.5.4 兜底里**不写任何报告文件**。

> ⚠ **落盘位置变更（强约束）**：报告**不再写入项目根**，而是写入 Step 1.5 选定的 `FEATURE_DIR`（即 `{SPECS_DIR}/N.{feature}/`）。原因：业务项目根上散落 `.tc-test-report.md` 难追溯"是哪次需求的产物"，且会被后续 feature 的跑测覆盖丢失。

全量跑完（或 fail-fast 停止）后**同时做两件事**：

1. **打到终端**（用户即时看）
2. **写两份 markdown 到 `FEATURE_DIR/`**：
   - `test-report.md` — 跑测结果（每层用例 / 失败 / 命令）
   - `coverage-report.md` — 覆盖率快照（数字 + HTML 链接），**仅在带 `--coverage` 或测试栈默认输出覆盖率时生成**

> 💡 **为什么默认写文件而不是开关**：参数多用户记不住，记不住就不会用，加了等于没加。报告默认产出，落到 specs 目录里就跟需求绑定，用户回看一目了然。

#### 4.1 终端输出

```text
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📊 /tc-test 总结  ·  feature 2.user-login
━━━━━━━━━━━━━━━━━━━━━━━━━━━━

| 组件 | 层 | 状态 | 数量 | 时间 |
|---|---|---|---|---|
| <组件 A> | 单元 | ✅ | 96 | 3s |
| <组件 A> | e2e | ✅ | 10 | 12s |
| <组件 B> | pytest | ✅ | 67 | 52s |
| 总计 | - | ✅ | 173 | 67s |

📊 覆盖率（如有，按实际组件列）：
   <组件 A>: lines X% / branches X% / functions X%
   关键路径覆盖情况...

📄 已写入 feature 目录:
   ✓ docs/tc-spec/2.user-login/test-report.md       (跑测结果)
   ✓ docs/tc-spec/2.user-login/coverage-report.md   (覆盖率快照)

📌 原始 HTML 报告（保留在测试框架默认位置，未挪动）：
   - 覆盖率 HTML: <实际路径>（如 frontend/coverage/index.html）
   - Playwright trace: <实际路径>（如 frontend/playwright-report/index.html）
   - forge coverage: <实际路径>（如 lcov.info）
```

#### 4.2 `FEATURE_DIR/test-report.md` 文件结构

每次跑完**覆盖写**到 `{FEATURE_DIR}/test-report.md`（固定文件名，不带时间戳，避免堆积）。文件内容比终端更全：

```markdown
# {feature 名} · 测试报告

> 由 `/tc-test` 自动生成 · 归属 feature: `2.user-login` · 下次跑测会覆盖此文件

**跑测时间**: 2026-04-28 14:30:12  
**跑测模式**: `/tc-test full --coverage`  
**git 分支**: feature-2-user-login @ 3aa505e  
**总耗时**: 67s  
**总状态**: ✅ 全绿（或 ❌ 失败 / ⚠️ 部分）

## 探测到的组件

- React 前端 (frontend/) — vitest + playwright
- Python 后端 (backend/) — pytest
- ...

## 各层结果

| 组件 | 层 | 命令 | 状态 | 数量 | 时间 |
|---|---|---|---|---|---|
| React 前端 | 单元 | `cd frontend && npm test` | ✅ | 96 passed | 3s |
| React 前端 | e2e | `cd frontend && npm run test:e2e` | ✅ | 10 passed | 12s |
| Python 后端 | pytest | `cd backend && pytest -q` | ✅ | 67 passed | 52s |

## 失败用例（如有，全绿则省略此节）

### React 前端 e2e
- **login.spec.ts:13** › [AC-002] 错误密码显示中文提示
  - 断言: `expect(error_text).toBe("用户名或密码错误")`
  - 实际: `"用户名错误"`
  - 复现: `cd frontend && npx playwright test e2e/tests/login.spec.ts:13`
  - Trace: `frontend/playwright-report/index.html`

## 后续建议

- 全绿可以提 PR / commit
- 失败用例需先修复（详见上面"失败用例"节）
- 覆盖率详见同目录 `coverage-report.md`
```

#### 4.3 `FEATURE_DIR/coverage-report.md` 文件结构

仅在跑测带覆盖率时写。如果本次跑测**没有产出覆盖率数据**（用户没加 `--coverage`、栈本身不出覆盖率）→ **不写此文件**，不写空壳。

```markdown
# {feature 名} · 覆盖率报告

> 由 `/tc-test --coverage` 自动生成 · 归属 feature: `2.user-login` · 下次跑测会覆盖

**跑测时间**: 2026-04-28 14:30:12  
**git 分支**: feature-2-user-login @ 3aa505e

## 各组件覆盖率

### React 前端
- lines: 93.37% / 门槛 70% ✓
- branches: 81.51% / 门槛 60% ✓
- functions: 74.56% / 门槛 70% ✓
- 关键路径: api.ts 100% / LoginPage 100%
- 原始 HTML: `frontend/coverage/index.html`（未挪动）

### Python 后端
- lines: 88.2% / 门槛 80% ✓
- 原始 HTML: `backend/htmlcov/index.html`（未挪动）

## 未达门槛的文件（如有）

- `frontend/src/utils/parser.ts` — lines 42% / 门槛 70%
- ...

## HTML 报告快捷链接

- 覆盖率: `frontend/coverage/index.html` / `backend/htmlcov/index.html`
- E2E trace: `frontend/playwright-report/index.html`
```

#### 4.4 关于报告文件与 git

**强约束**：

- ✅ 默认写到 `{FEATURE_DIR}/test-report.md` 与 `{FEATURE_DIR}/coverage-report.md`
- ✅ 这些文件**应该跟 specs 一起被 git 追踪** —— 它们是需求验证的证据，归档价值高
- ✅ 报告每次跑都覆盖（不存历史，避免堆积）；要存档由用户自己 mv 或靠 git history
- ❌ **不再写到项目根**（兼容性：如果项目里已有遗留的 `/.tc-test-report.md`，本命令**不删它**，但也不再更新它，用户自行清理）
- ❌ **不**自动 commit / push 报告文件
- ❌ **不**把覆盖率 HTML 整个 copy 进 specs 目录（HTML 体积大且依赖相对路径，挪动会损坏）—— 只在 markdown 里写路径链接

**已有 `.tc-test-report.md` 在项目根的项目**（旧版本残留）：

终端额外提示一次：

```text
ℹ 检测到项目根存在旧版 .tc-test-report.md（来自 v0.x 版本 /tc-test）。
   新版报告已写入 docs/tc-spec/2.user-login/test-report.md。
   旧文件请自行删除（rm .tc-test-report.md），并确认 .gitignore 不再需要它。
```

## Feature 过滤模式

`/tc-test feature 2` — 只跑与 feature 2 相关的测试。

实现方式（按优先级）：

1. **测试名 grep**：跑测试时加 `--grep "F-X" / "AC-X"`（feature 2 的 F-XXX/AC-XXX 编号范围由 specs 决定）
2. **路径 grep**：测试文件路径含 feature 涉及的模块名
3. **如果都不可行**：跑全量并 warning

**输出**（强制，组件名按实际项目）：

```text
🎯 Feature 过滤：feature 2
   匹配的测试（基于 [F-XXX] / [AC-XXX] grep）:
   - <组件 A>: N 个用例（<匹配模式>）
   - <组件 B>: N 个用例（<匹配模式>）
```

## 失败处理（fail-fast 默认）

任一层失败立即停止，不跑后续层。**例外**：用户加 `--no-fail-fast` 强制全量跑。

**失败时必须输出**：

1. **失败用例的最小复现命令**（让用户能立即点进去看）
2. **trace / report 文件路径**（HTML 报告）
3. **常见原因 hint**（如 "e2e 失败 → 检查真服务是否启动"）

## 全局规则（与其它 tc-* 命令一致）

**强制拦截（绝不放行）：**
- Step 0 任一项失败 → 立即停止 + 修复命令
- 探测到测试基础设施缺失（情形 B / C）→ 立即停止 + 引导 + **不写任何报告文件**
- 用户指定的模式 / 组件不存在 → 立即停止 + 列出可用项
- 项目无 specs 目录且未传 `--report-dir` → 立即停止 + 引导（**绝不**回退写到项目根）

**报告但不阻止：**
- 单个测试用例失败 → fail-fast 停在该层，但不"否认"已通过的层
- 覆盖率门槛失败 → 当作信息输出，不当作命令失败
- Deprecation warnings → 静默忽略

## 输出口径

每层用 `━━━━━` 分隔，状态用图标：

- ✅ 全绿
- ❌ 失败
- ⚠️ 警告（覆盖率达标但有边界用例 / 测试有 skip / xfail）
- ⏭ 跳过（项目里该层不存在或被显式排除）
- 🔄 重试（自动 retry，如 Playwright flaky）
- ⛔ 拦截（环境问题 / 探测失败，根本没开跑）

## 反模式（强化）

环境与硬编码相关（**新增重点**）：

- ❌ **绝不允许在 Step 0 失败时继续跑测试** —— 跑出来的结果不可信，会污染 `tc-qa-engineer` 后续设计；宁可什么都不跑，也不要跑出"假信号"
- ❌ **绝不允许尝试自动激活 venv / 自动 npm install / 自动改 PATH** —— 这是项目状态，由用户决定。我只能告知"这里坏了"和给修复命令
- ❌ **绝不允许"试一下不同的命令"碰运气** —— 比如 `python` 不行就试 `python3`、`pytest` 不行就试 `python -m pytest` —— 必须一次性确认正确入口，确认不了就拦截
- ❌ **绝不允许在输出里硬编码"前端 / 后端"** —— 项目可能就只有一个组件、可能是合约项目、可能是 AI agent 项目；按 Step 1 探测出的实际组件名描述
- ❌ **绝不允许在没探测前就预设执行顺序** —— 如"先后端再前端"、"先 pytest 再 vitest" —— 顺序由探测结果 + Step 2 计划决定
- ❌ **绝不允许"跳过环境警告硬跑"** —— 比如 venv 没激活但 pytest 全局可用 —— 看似能跑，实际可能 import 错版本

报告落盘相关（**新增重点，与本次重构对应**）：

- ❌ **绝不允许把报告写到项目根** —— 不写 `.tc-test-report.md` / `test-report.md` / `coverage-report.md` 到 `PROJECT_DIR/`。报告必须落到 `{SPECS_DIR}/N.{feature}/`，业务项目根上禁止散落
- ❌ **绝不允许在情形 B / C / 1.5.4 兜底场景下生成报告** —— 没有真实跑过用例的"报告"是噪音 + 误导，会让用户/CI 误判"测试通过过"
- ❌ **绝不允许"自动挑了某个 feature 直接开跑"** —— 即使只有 1 个 feature 也要列出来确认，多个 feature 必须让用户**从列表按编号选**（参考 1.5.2 情况 B / C）。归属错了写到的报告会污染其它需求文件夹
- ❌ **绝不允许让用户"手动打 feature 名"** —— 不要让用户记编号、记需求名。命令必须列出 `FEATURE_LIST`，用户回复 1/2/3 按编号选；命令行参数 `feature N` 是给已知编号的快捷方式，不是默认交互
- ❌ **绝不允许"一次性跑所有 feature 的报告"** —— 多 feature 时按编号顺序逐个跑（用户跑完一个再回来选下一个），而不是合并到一份报告里
- ❌ **绝不允许把覆盖率 HTML 整目录 copy 进 specs** —— HTML 体积大且依赖相对路径，挪动会损坏。只在 markdown 里写源路径链接
- ❌ **绝不允许覆盖率为空时仍写 coverage-report.md** —— 用户没加 `--coverage` 或栈本身不出覆盖率时，不要写空壳文件凑数

通用反模式：

- ❌ **不应修改任何业务代码** — 只是跑测试
- ❌ **不应自动修复失败的测试** — 失败就报告
- ❌ **不应静默跳过失败用例** — fail-fast 默认行为，明确告知停在哪一层
- ❌ **不应替代 `/tc-ai`** — 不开发新功能、不写新测试
- ❌ **不应吞 `tc-qa-engineer` 的职责** — 不设计用例

## 与已有命令的关系

```
混沌阶段                半结构化               结构化               代码                   验证

[聊需求]   →   docs/feature-X.md   →   N.feature/三件套   →   实施   →   /tc-test ✅

/tc-discuss     /tc-prd <path>           /tc-ai <path>            /tc-test <args>
                                            ↓
                                    内部 N6 自动调
                                    tc-qa-engineer
                                    （设计新用例）
```

| 命令 | 输入 | 输出 | 核心职责 |
|---|---|---|---|
| `/tc-init` | （项目根目录）| `.claude/CLAUDE.md` + `rules/` | 项目初始化 |
| `/tc-test-setup` | （项目根目录）| 测试基建（依赖+配置+hello world）| **零测试设施时**搭基建 |
| `/tc-discuss` | 一句话需求 | `docs/feature-N.md` | 聊清需求 |
| `/tc-prd` | docs/*.md | `N.{feature}/` 三件套 | 立项 |
| `/tc-ai` | specs + 代码项目 | 实施代码 + 测试 | 自动开发 |
| **`/tc-test`** | **测试范围参数** | **结构化测试报告** | **跑现有测试** |

## 示例用法（按不同项目形态）

> 下例展示**不同项目类型下** `/tc-test` 的输出形态，证明命令不硬编码"前端/后端"。

### 示例 1：全栈项目（Python 后端 + React 前端）

```bash
/tc-test
   ├─ Step 0: 检查 venv + node_modules
   ├─ Python 后端 pytest（~50s）
   └─ React 前端 vitest + 覆盖率（~3s）
```

### 示例 2：纯前端组件库（无后端）

```bash
/tc-test
   ├─ Step 0: 检查 node_modules
   └─ vitest 单元 + storybook 组件测试（~5s）
   # 不会出现"后端"二字
```

### 示例 3：Solidity 合约项目

```bash
/tc-test
   ├─ Step 0: 检查 forge + lib/
   └─ forge test（~10s）
```

### 示例 4：纯 Python AI Agent 项目

```bash
/tc-test
   ├─ Step 0: 检查 venv + ANTHROPIC_API_KEY 是否需要
   ├─ pytest 单元（mock LLM）
   └─ pytest 集成（如需真 API，**先确认用户授权**）
```

### 示例 5：watch 模式

```bash
/tc-test --watch
   └─ 仅在支持 watch 的栈上启动，否则报错
```

### 示例 6：调试某个 feature

```bash
/tc-test feature 2
   └─ 只跑与 feature 2 相关的测试（按 Step 1 探测出的组件分别 grep）
```

## 失败模式

### 环境问题（Step 0 拦截）

| 场景 | 处理 |
|---|---|
| Python venv 未激活 | ⛔ 拦截 + `source .venv/bin/activate` 指引 |
| node_modules 不存在 | ⛔ 拦截 + `npm install` 指引 |
| 关键测试包 import 失败 | ⛔ 拦截 + 重装命令 |
| 测试端口被占用 | ⛔ 拦截 + `lsof -i :PORT` + 杀进程指引 |
| AI 项目 API key 缺失 | ⛔ 拦截 + 提示设置环境变量（不打印 key 值）|
| Playwright 浏览器未安装 | ⛔ 拦截 + `npx playwright install` 指引 |

### 项目结构问题（Step 1 探测）

| 场景 | 处理 |
|---|---|
| 项目根目录无任何项目描述文件 | "❌ 不是受支持的项目类型" + 列出根目录前 10 个文件 |
| **L1：完全没有测试框架** | 输出情形 C 引导（推荐 `/tc-test-setup` 一键搭建）+ **不写报告文件** |
| **L2：测试脚本未配** | 输出情形 C，给"在 package.json / pytest.ini 加 test 入口"的最小修复方案 + **不写报告文件** |
| **L3：测试代码不存在** | 输出情形 B 引导（让 `/tc-ai` 的 N6 调 `tc-qa-engineer` 设计补充用例）+ **不写报告文件** |
| L1+L2+L3 部分有部分无（混合）| 跳过缺失层，只跑能跑的，明确输出"跳过 X 因为 Y"；只要至少跑过一个用例就写报告到 FEATURE_DIR |
| 项目无 `docs/tc-spec/N.*/` specs 目录 | 拦截 + 引导跑 `/tc-prd` 立项 或 `/tc-test --report-dir <path>` 显式指定（**绝不**默认写到项目根）|

### 运行期问题

| 场景 | 处理 |
|---|---|
| 测试套件超时（>5min）| 报错 + 建议加 `feature N` 缩范围或拆 batch 跑 |
| 覆盖率门槛失败 | 不当作命令失败，正常报告 + 提示哪些文件未达标 + 给覆盖率 HTML 报告路径 |
| 单测试用例失败 | fail-fast 默认停在该层 + 给最小复现命令 + trace 路径 |

### 跨层依赖问题

| 场景 | 处理 |
|---|---|
| 快测层失败但用户只跑联调 | 提示"快测层没全绿，联调大概率挂；建议先 `/tc-test fast` 修好" |
| 联调模式但缺独立测试服务入口 | 报错"联调需要独立测试服务入口" + 引导参考其它项目 |
| `feature N` 模式但找不到匹配测试 | warning"feature N 未识别到测试，跑全量" + 列出实际跑的范围 |
