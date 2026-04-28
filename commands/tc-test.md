---
description: 跑测试 — 自动识别项目测试栈，跑现有测试套件，输出结构化报告（不设计新用例）
---

# /tc-test — 跑测试

随时跑项目的现有测试套件。**不设计新用例、不修改业务代码**——只是「按下按钮、跑、报告结果」。

> 💡 与 [`tc-qa-engineer`](../skills/tc-qa-engineer/SKILL.md) skill 的区别：
> - `tc-qa-engineer`：**设计** 新的测试用例（基于 requirements.md AC-XXX）+ 跑测 + 形成 bug 闭环 — 由 `/tc-ai` 的 N6 节点自动触发
> - `/tc-test`（本命令）：**只跑** 现有测试、结构化报告 — 由你随时手动调用

## 何时用

- 改完代码想验证「没破现有测试」
- PR 提交前最后一道闸
- 上线前跑完整 Part 2 联调
- 开发中想 watch 模式实时反馈

## 何时不用

- 想**新增**测试用例 → 用 `tc-qa-engineer` skill（在 `/tc-ai` 流程里自动调）
- 测试基础设施还没搭 → 先跑 `/tc-prd` 立项一个测试基础设施 feature，再用 `/tc-ai` 实施

## 「零测试设施」项目的正确路径

如果你的项目**完全没有测试框架 / 没有测试代码**，本命令**不会自动安装依赖、不会写示例测试**——这是有意的设计（避免污染你的项目结构）。

正确的引导路径分三档：

### 档位 1：项目刚立完 specs，还没实施代码

```bash
# 你已经有 N.feature/{requirements,design,tasks}.md
/tc-ai <项目路径>
   ├─ N3 写代码
   └─ N6 自动调 tc-qa-engineer 写测试 + 跑 → 测试代码与基础设施一起诞生
   
# 然后才跑：
/tc-test
```

### 档位 2：项目已有代码，但完全无测试基础设施

```bash
# 立项一个"测试基础设施"feature：
/tc-discuss "我们项目还没测试体系，要建立完整的（单元/组件/契约/E2E/CI）"
   └─ 输出 docs/feature-N-test-infra.md
   
/tc-prd <项目路径>
   └─ 生成 N.test-infra/ 三件套（含技术方案 + 任务列表）
   
/tc-ai <项目路径>
   └─ 实施测试基础设施（装依赖 + 写 setup + 写第一批测试 + CI）
   
/tc-test                    # 现在能跑了
```

参考案例：本仓库的 [0.7.frontend-test-infra](https://github.com/callustang/wangzhaoqing/tree/main/A_frontend_backend/0.7.frontend-test-infra) 就是这种 feature 的范例。

### 档位 3：测试框架装了，只是没写测试代码

```bash
# 直接用 /tc-ai，N6 节点会调 tc-qa-engineer 基于已有 specs 的 AC-XXX 设计测试：
/tc-ai <项目路径>
   ├─ N3 走任务（如果没新代码要写，可空 task）
   └─ N6 必触发 tc-qa-engineer（feature 完成 = must-trigger 条件）
        └─ 自动设计单元 + 集成 + 端到端用例 → 落到 *.test.* 文件
   
/tc-test                    # 然后跑
```

### 反模式（**禁止**自动做的事）

- ❌ 不要自己 `npm install vitest`、`pip install pytest` —— 会改变项目依赖图，违反"只跑不修"原则
- ❌ 不要自己创建 `*.test.tsx` 示例文件 —— 测试用例必须基于 specs 设计
- ❌ 不要自己创建 `vitest.config.ts` —— 配置选项是技术决策，应在 `/tc-prd` Step 5.5 让用户确认
- ❌ 不要"跳过"缺失层后假装成功 —— 必须明确输出"跳过 X 因为 Y"

## 输入参数

`$ARGUMENTS` 灵活，下面任一格式都接受：

```bash
# 模式
/tc-test                    # 默认 = offline（前端 Part 1 + 后端 pytest），不跑 online
/tc-test offline            # 仅前端 Part 1（单元 + 组件 + 覆盖率）
/tc-test backend            # 仅后端 pytest
/tc-test online             # 前端 Part 2（contract + e2e，自动启后端）
/tc-test all                # 全量 = backend + offline + online

# 修饰符
/tc-test --watch            # 前端 vitest watch 模式（仅 offline，开发期实时反馈）
/tc-test --coverage         # 强制带覆盖率报告（默认 offline 已开）
/tc-test feature 2          # 只跑与 feature 2 相关的测试（grep [F-XXX] 或 [AC-XXX] 标签）

# 组合
/tc-test all --coverage     # 全量 + 覆盖率
/tc-test backend feature 2  # 仅后端，且只跑 feature 2 关联的测试
```

## 流程

### Step 1: 探测项目结构（含「零测试设施」判定）

扫描项目根目录：

```bash
{PROJECT_ROOT}/
├── package.json           # 前端 npm 项目？看 scripts 里有没有 test*
├── frontend/              # 或前端在子目录
├── backend/               # 后端目录？
├── pyproject.toml / requirements.txt   # Python 项目？
└── go.mod / Cargo.toml / foundry.toml  # 其它语言？
```

**对每个识别出的"项目部分"做三层判断**：

| 层级 | 判断方法 | 状态 |
|---|---|---|
| L1 测试**框架**装了吗？ | 看依赖：`vitest`/`jest`/`pytest`/`go test` 等是否在 `package.json` / `requirements.txt` / `go.mod` | 装了 ✓ / 未装 ✗ |
| L2 测试**脚本**配了吗？ | 看 `package.json` scripts 有没有 `test*` 命令；Python 看 `pytest.ini` / `pyproject.toml` 有没有 `[tool.pytest]` | 配了 ✓ / 未配 ✗ |
| L3 测试**代码**有吗？ | grep 测试文件：`**/*.test.{ts,tsx,js}` / `**/test_*.py` / `**/*_test.go` | 有 ✓ / 无 ✗ |

**输出探测结果**（强制，按状态四选一）：

#### 情形 A：完整有 — 所有层都 ✓

```text
🔍 项目探测：
   - 前端：✓ frontend/ (vitest + playwright + 96 个测试文件)
   - 后端：✓ backend/ (pytest + 67 个测试用例)
   测试入口齐全，进入 Step 2 →
```

#### 情形 B：测试框架装了但没测试代码（L1 ✓ / L2 ✓ / L3 ✗）

```text
🔍 项目探测：
   - 前端：⚠ frontend/ (vitest 已装，但找不到 *.test.* 文件)
   - 后端：⚠ backend/ (pytest 已装，但找不到 test_*.py)

⚠ 测试框架已就绪，但**没有任何测试代码**。

🤔 推荐路径（你选）：
   1) 跑 /tc-ai 让 N6 节点自动调 tc-qa-engineer 基于现有 specs 的 AC-XXX 设计测试
   2) 用 /tc-discuss 讨论"测试基础设施"feature → /tc-prd 立项 → /tc-ai 实施
   3) 如果你已知道要测什么，手动写 *.test.tsx / test_*.py 后再跑 /tc-test

⛔ 本命令不写新测试，停止于此。
```

#### 情形 C：完全没有测试框架（L1 ✗）

```text
🔍 项目探测：
   - 前端：❌ frontend/ (package.json 未发现 vitest / jest / cypress / playwright 等)
   - 后端：❌ backend/ (requirements.txt 未发现 pytest / unittest / nose 等)

❌ 项目尚未建立测试基础设施。

🤔 推荐路径（按项目语言）：
   - 前端 (React/Vue/Svelte) → 推荐 Vitest + @testing-library + MSW
     参考: ~/.claude/projects/.../{你项目}/0.7.frontend-test-infra/ （如有）
   - 后端 (Python) → 推荐 pytest + httpx + pytest-asyncio
   - Go → go test + testify
   - 合约 → Foundry / Hardhat

✋ 建议工作流：
   1) /tc-discuss "我们项目还没测试基础设施，要做完整测试体系（单元/组件/契约/E2E/CI）"
   2) /tc-prd <项目路径>     ← 立项 N.test-infra feature
   3) /tc-ai                  ← 实施测试基础设施
   4) /tc-test                ← 设施搭好后再回来跑

⛔ 本命令不引入新依赖，停止于此。
```

#### 情形 D：部分有部分没（混合）

```text
🔍 项目探测：
   - 前端：✓ frontend/ (vitest + 96 测试)
   - 后端：❌ backend/ (无 pytest)

⚠ 部分测试设施缺失。本命令只跑能跑的层，跳过缺失部分。

🎯 实际可跑：
   - 前端 offline ✓
   - 前端 online ✓ (vitest webServer 自动启 backend，但 backend 自身无 pytest 不影响)
   
🚫 跳过：后端 pytest（请按情形 C 路径补全）
```

### Step 1.5: 「特殊情形」中断条件

遇到下列情况立即输出错误并停止：

| 情形 | 输出 |
|---|---|
| 整个项目根目录连 package.json / pyproject.toml 都没有 | "❌ 不是受支持的项目类型（前端 npm / Python / Go / Rust / 合约都未识别）" |
| `package.json` 存在但 `scripts` 完全空 | "⚠ package.json 没有任何 npm scripts，无法跑测试" |
| 用户 `--watch` 但项目没装 vitest / jest 等 watch 工具 | "❌ watch 模式需要 vitest 或 jest，当前未装" |
| 用户 `online` 但项目没 Playwright | "❌ online 模式需要 @playwright/test，当前未装" |

### Step 2: 解析模式

按 `$ARGUMENTS` 决定要跑哪几层：

| 用户输入 | 跑什么 |
|---|---|
| 空 / `default` | offline + backend |
| `offline` | 前端 Part 1 only |
| `backend` | 后端 pytest only |
| `online` | 前端 Part 2（contract + e2e）|
| `all` | backend + offline + online |
| `--watch` | offline watch 模式 |
| `feature N` | 跑前过滤标签（仅跑包含 `[F-X]` / `[AC-X]` / 文件路径含 `N.{name}` 的测试）|

**输出执行计划**（强制）：

```text
🎯 执行计划：
   1️⃣  后端 pytest (~50s)
   2️⃣  前端 type-check + offline (含覆盖率) (~5s)
   3️⃣  前端 contract (Playwright + 真后端) (~10s)
   4️⃣  前端 e2e (Playwright + 浏览器) (~15s)
   预计总耗时：~80s
```

### Step 3: 依次跑

按顺序执行，每完成一层立即输出结果。**任一层失败立即停止**（fail-fast），不继续后面的层 — 因为后面的层往往依赖前面的（如 e2e 依赖 backend pytest 不回归）。

**每层标准输出**：

```text
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🧪 后端 pytest
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
命令: cd backend && pytest test_api.py -v
✅ 67 passed in 52s

━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🧪 前端 offline (vitest + 覆盖率)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
命令: cd frontend && npm run test:offline
✅ 96 passed in 3s
📊 覆盖率: lines 93.37% / branches 81.51% / functions 74.56% （门槛 70/60/70 ✓）
```

**失败时**：

```text
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🧪 前端 e2e
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
命令: cd frontend && npm run test:e2e
❌ 1 failed / 9 passed

失败用例:
   - login.spec.ts › [AC-002] 错误密码显示中文提示
     文件: e2e/tests/login.spec.ts:13
     断言: expect(error_text).toBe("用户名或密码错误")
     实际: "用户名错误"
   
🔧 复现命令:
   cd frontend && ./node_modules/.bin/playwright test e2e/tests/login.spec.ts:13
   
📂 详细 trace:
   open frontend/playwright-report/index.html
   
⛔ 停止后续层，请先修复此失败
```

### Step 4: 总结报告

全量跑完（或 fail-fast 停止）后输出：

```text
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📊 /tc-test 总结
━━━━━━━━━━━━━━━━━━━━━━━━━━━━

| 层 | 状态 | 数量 | 时间 |
|---|---|---|---|
| 后端 pytest | ✅ | 67 passed | 52s |
| 前端 offline | ✅ | 96 passed | 3s |
| 前端 contract | ✅ | 32 passed | 7s |
| 前端 e2e | ✅ | 10 passed | 12s |
| 总计 | ✅ | 205 测试 | 74s |

📊 前端覆盖率：
   lines 93.37% / branches 81.51% / functions 74.56%
   关键路径: api.ts 100% / LoginPage 100%

📌 后续建议：
   - 全绿可以提 PR / commit
   - 覆盖率 HTML: open frontend/coverage/index.html
   - Playwright trace: open frontend/playwright-report/index.html
```

## Feature 过滤模式

`/tc-test feature 2` — 只跑与 feature 2 相关的测试。

实现方式（按优先级）：

1. **测试名 grep**：跑测试时加 `--grep "F-0\|F-1\|AC-0\|AC-1"`（feature 2 的 F-XXX/AC-XXX 编号范围由 specs 决定）
2. **路径 grep**：测试文件路径含 `users` / `auth` / `query-logs`（feature 2 改造的模块）
3. **如果都不可行**：跑全量并 warning

**输出**（强制）：

```text
🎯 Feature 过滤：feature 2
   匹配的测试（基于 [F-XXX] / [AC-XXX] grep）:
   - 后端: 11 个用例（TestFeature2*）
   - 前端 offline: 16 个用例（AdminUsers / AdminQueryLogs）
   - 前端 contract: 6 个（auth + users）
   - 前端 e2e: 2 个（login + admin-create-user）
```

## 失败处理（fail-fast 默认）

任一层失败立即停止，不跑后续层。**例外**：用户加 `--no-fail-fast` 强制全量跑。

**失败时必须输出**：

1. **失败用例的最小复现命令**（让用户能立即点进去看）
2. **trace / report 文件路径**（HTML 报告）
3. **常见原因 hint**（如 "Playwright 失败 → 检查后端是否启动"）

## 全局规则（与其它 tc-* 命令一致）

**暂停条件：**
- 测试基础设施缺失（如没装 vitest）→ 报错，提示先装
- 后端启不起来（python 路径错、端口占用）→ 报错并给具体修复方案
- 覆盖率门槛失败 → **不暂停**，按失败报告

**不暂停：**
- 单个测试用例失败 → 直接报告
- 警告（如 deprecation warnings）→ 静默忽略

## 输出口径

每层用 `━━━━━` 分隔，状态用图标：

- ✅ 全绿
- ❌ 失败
- ⚠️ 警告（覆盖率达标但有边界用例）
- ⏭ 跳过（如 `online` 模式但 `--watch`）
- 🔄 重试（Playwright 自动 retry）

## 反模式

- ❌ **本命令不应修改任何业务代码** — 只是跑测试，不修代码
- ❌ **不应自动修复失败的测试** — 失败就报告，让用户决定
- ❌ **不应静默跳过失败用例** — fail-fast 默认行为，明确告知用户停在哪一层
- ❌ **不应替代 `/tc-ai`** — 本命令不开发新功能、不写新测试，只跑现有的
- ❌ **不应吞 `tc-qa-engineer` 的职责** — QA skill 的核心是设计用例，本命令只跑现有的

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
| `/tc-discuss` | 一句话需求 | `docs/feature-N.md` | 聊清需求 |
| `/tc-prd` | docs/*.md | `N.{feature}/` 三件套 | 立项 |
| `/tc-ai` | specs + 代码项目 | 实施代码 + 测试 | 自动开发 |
| **`/tc-test`** | **测试范围参数** | **结构化测试报告** | **跑现有测试** |

## 示例用法

### 开发中实时反馈

```bash
/tc-test --watch
   └─ vitest watch 模式，文件改动自动重跑前端单元/组件测试
```

### PR 提交前

```bash
/tc-test
   ├─ 后端 pytest（~50s）
   └─ 前端 offline（~5s）
   合计 ~1min，最快验证基本盘
```

### 上线前完整验证

```bash
/tc-test all
   ├─ 后端 pytest
   ├─ 前端 offline
   ├─ 前端 contract（真后端）
   └─ 前端 e2e（真后端 + 浏览器）
   合计 ~2min
```

### 调试某个 feature

```bash
/tc-test feature 2
   └─ 只跑与 2.account-management 相关的测试
```

### 联调阶段

```bash
/tc-test online
   └─ 跑 contract + e2e（默认 fail-fast，contract 先跑）
   └─ contract 失败 → 反向推动后端改 API（不修测试）
```

## 失败模式

### 项目结构问题（Step 1 探测期）

| 场景 | 处理 |
|---|---|
| 项目根目录没有 package.json / pyproject.toml / go.mod 等任何项目描述文件 | "❌ 不是受支持的项目类型" + 列出根目录前 10 个文件供用户排查 |
| **L1：完全没有测试框架** | 输出情形 C 引导（`/tc-discuss → /tc-prd → /tc-ai` 立项测试基础设施 feature）|
| **L2：测试脚本未配** | 输出情形 C，给"在 package.json 加 `test` script"的最小修复方案 |
| **L3：测试代码不存在** | 输出情形 B 引导（让 `/tc-ai` 的 N6 调 `tc-qa-engineer` 设计补充用例）|
| L1+L2+L3 部分有部分无（混合）| 跳过缺失层，只跑能跑的，明确输出"跳过 X 因为 Y" |

### 运行期问题

| 场景 | 处理 |
|---|---|
| 测试栈未安装（如 `command not found: pytest`，但依赖声明里有）| "依赖装漏了" + 给具体安装命令（`pip install -r requirements.txt` / `npm install`）|
| 后端启动失败（端口占用）| 报错 + `lsof -i :8000` 指引 + 建议先杀进程 |
| 后端启动失败（python 路径错）| 报错 + `which python` / `which python3` 指引 + 建议激活 venv |
| Playwright 浏览器未安装 | 报错 + 给 `npm run test:e2e:install` 指引 |
| 测试套件超时（>5min）| 报错 + 建议加 `feature N` 缩范围或拆 batch 跑 |
| 覆盖率门槛失败 | 不当作命令失败，正常报告 + 提示哪些文件未达标 + 给覆盖率 HTML 报告路径 |
| 单测试用例失败 | fail-fast 默认停在该层 + 给最小复现命令 + trace 路径 |

### 跨层依赖问题

| 场景 | 处理 |
|---|---|
| 后端 pytest 失败，但用户只跑 `online` | 提示"后端 pytest 没全绿，e2e 大概率挂；建议先 `/tc-test backend` 修好" |
| `online` 模式但后端 e2e_server.py 不存在 | 报错"online 需要独立 e2e backend 入口" + 引导参考其它项目的 backend/test_e2e_server.py |
| `feature N` 模式但找不到匹配测试 | warning"feature N 未识别到测试，跑全量" + 列出实际跑的范围 |
