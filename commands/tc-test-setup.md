---
description: 测试基础设施搭建 — 项目零测试设施时，按实际栈推荐并搭建一套完整测试方案
---

# /tc-test-setup — 测试基础设施搭建

为**完全没有测试设施**的项目搭建一套测试基建：装依赖、写配置、建目录、写一个能跑通的 hello world 测试。**不写业务测试用例**。

> 💡 完整工作流：[`/tc-init`](tc-init.md) → [`/tc-discuss`](tc-discuss.md) → [`/tc-prd`](tc-prd.md) → [`/tc-ai`](tc-ai.md) → [`/tc-test`](tc-test.md)
> 当 `/tc-test` 探测到项目零测试设施时，会引导你跑 **`/tc-test-setup`**，搭好基建后再回到主流程。

## 何时用

- 项目**没有任何测试框架**（`package.json` 里没 vitest/jest/playwright，`requirements.txt` 里没 pytest…）
- 项目装了测试框架但**没配脚本/目录约定**，跑不起来
- `/tc-test` 探测后建议你跑这个命令

## 何时不用

| 场景 | 用什么 |
|---|---|
| 测试基建齐全，只是没写业务测试 | `/tc-ai` → N6 节点自动调 `tc-qa-engineer` 设计用例 |
| 想跑现有测试 | `/tc-test` |
| 历史项目要补历史代码的回归测试 | `/tc-test-legacy`（独立命令，**与本命令无关**） |
| 想对当前需求/feature 设计测试 | `tc-qa-engineer` skill（由 `/tc-ai` N6 触发） |

## 核心边界（强约束）

- ✅ **只搭基建**：依赖 + 配置 + 目录 + 1 个 hello world 测试
- ❌ **不写业务测试用例** — 那是 `tc-qa-engineer` 的活
- ❌ **不针对 git diff 生成测试** — 那是 `/tc-test-legacy` 的活
- ❌ **不强行套用其它项目的测试架构** — 必须按检测到的栈推荐
- ⚠️ **强制要用户确认方案** — 装依赖会改 lock file，不可逆操作

## 工作流程

### Step 1: 探测项目栈

读取项目配置自动判断：

| 文件 | 判定 |
|---|---|
| `package.json` | 前端/Node.js — 读 `dependencies` / `devDependencies` 推断框架（React/Vue/Svelte/Next/Nuxt…） |
| `pyproject.toml` / `requirements*.txt` / `setup.py` | Python |
| `go.mod` | Go |
| `Cargo.toml` | Rust |
| `foundry.toml` / `hardhat.config.*` / `truffle-config.*` / `anchor.toml` | 智能合约 |
| `pubspec.yaml` | Flutter/Dart |
| `*.xcodeproj` / `Package.swift` | iOS/Swift |

**多模块项目**（前后端 monorepo）→ 分别识别每个子目录的栈，分别推荐。

### Step 2: 调用 `tc-test-architect` skill 生成方案

把探测到的栈交给 [`tc-test-architect`](../skills/tc-test-architect/SKILL.md) skill，它会输出**结构化方案**：

```text
🏗 测试方案推荐（基于检测到的项目栈）

📦 检测到的栈:
  - 前端 frontend/ → React 18 + TypeScript + Vite + Tailwind
  - 后端 backend/  → Python 3.11 + FastAPI

═══════════════════════════════════════════════
🎯 推荐方案（前端 frontend/）
═══════════════════════════════════════════════

测试金字塔:
  - 单元/组件层 → Vitest + @testing-library/react
  - 网络 mock   → MSW
  - E2E 层      → Playwright
  - 覆盖率工具  → @vitest/coverage-v8

要装的依赖（devDependencies）:
  - vitest@^2
  - @testing-library/react@^16
  - @testing-library/jest-dom@^6
  - @testing-library/user-event@^14
  - @vitest/coverage-v8@^2
  - jsdom@^25
  - msw@^2
  - @playwright/test@^1

要新增的脚本（package.json scripts）:
  - "test": "vitest run"
  - "test:watch": "vitest"
  - "test:coverage": "vitest run --coverage"
  - "test:e2e": "playwright test"

要新增的配置文件:
  - vitest.config.ts          (test 配置 + coverage 阈值)
  - playwright.config.ts      (E2E 配置)
  - src/test/setup.ts         (全局 setup)
  - src/test/msw-server.ts    (MSW handlers 入口)

要新增的目录:
  - src/__tests__/            (单元/组件测试，跟随源码就近放也可)
  - e2e/                      (E2E 用例)
  - src/test/                 (test fixtures + setup)

要新增的 hello world:
  - src/__tests__/hello.test.ts        (vitest 跑通验证)
  - e2e/smoke.spec.ts                  (playwright 跑通验证)

推荐覆盖率门槛:
  - lines 70% / branches 60% / functions 70%

═══════════════════════════════════════════════
🎯 推荐方案（后端 backend/）
═══════════════════════════════════════════════

... (同上结构)

═══════════════════════════════════════════════
⚠️  请确认方案（必须）
═══════════════════════════════════════════════

回复 "确认" 开始搭建，或告诉我要改什么（换 jest？不要 e2e？覆盖率门槛改 80%？）。
```

### Step 3: 强制用户确认

**这一步不可跳过**。装依赖会改 `package.json` / `requirements.txt` / `go.sum` 的 lock file，是不可逆操作。

用户可以：

- ✅ **回复"确认"** → 进入 Step 4
- 🔧 **要求改方案**（如"换 jest"、"不要 e2e"、"覆盖率门槛太严"）→ skill 重新生成方案 → 再次确认
- ❌ **拒绝** → 终止，不做任何修改

### Step 4: 执行搭建

按确认后的方案分步执行，每一步执行后立即输出结果：

1. **装依赖**
   - 前端：`pnpm add -D ...` / `npm install --save-dev ...` / `yarn add -D ...`（按项目实际包管理器）
   - 后端：`pip install ...` + 写入 `requirements-dev.txt`，或 `pyproject.toml` 的 `[project.optional-dependencies]`
   - Go：`go get ...`
2. **写配置文件**（vitest.config.ts / playwright.config.ts / pytest.ini 等）
3. **建目录**（`mkdir -p src/__tests__ e2e src/test` 等）
4. **写 hello world 测试**（每层一个，验证基建能跑通）
5. **加 npm scripts / Makefile target**（让 `npm test` / `make test` 直接能跑）

### Step 5: 跑通 hello world 验证

搭完立刻跑一次，**确认基建真的能用**：

```bash
# 前端
npm test                    # 应输出 "1 passed"
npm run test:e2e            # 应输出 "1 passed"（如果有装 playwright）

# 后端
pytest -q                   # 应输出 "1 passed"
```

**任何一项跑不通都不算搭建成功**，需要修到能跑通才结束。

### Step 6: 总结 + 引导下一步

```text
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✅ /tc-test-setup 完成
━━━━━━━━━━━━━━━━━━━━━━━━━━━━

新增依赖（前端）: vitest, @testing-library/react, msw, @playwright/test ...
新增依赖（后端）: pytest, httpx, pytest-asyncio
新增配置: vitest.config.ts, playwright.config.ts, pytest.ini
新增目录: src/__tests__/, e2e/, backend/tests/
新增 hello world: 3 个测试，全部跑通 ✓

下一步:
  1. 用 /tc-discuss 聊清下一个 feature 的需求
  2. 用 /tc-prd 立项 → 生成 specs 三件套（含 AC-XXX 验收标准）
  3. 用 /tc-ai 实施 → N6 节点会自动调 tc-qa-engineer 基于 AC-XXX 设计真实业务测试
  4. 用 /tc-test 跑测试

⚠️ 本命令只搭基建，没写业务测试用例。如果你想给现有的"历史"代码补回归测试，
   请用 /tc-test-legacy（独立命令，不在本流程中）。
```

## 反模式（**禁止**）

- ❌ 不要跳过用户确认直接装依赖（lock file 不可逆）
- ❌ 不要硬塞项目未使用的栈（如 React 项目硬上 Cypress + Jest，明明该用 Playwright + Vitest）
- ❌ 不要为了"完整性"装一堆暂时用不到的工具（如默认装 Storybook、Chromatic、Stryker）
- ❌ 不要写业务相关的测试用例（hello world 必须是无业务意义的，如 `expect(1+1).toBe(2)`）
- ❌ 不要修改业务代码（除非业务代码阻塞了 hello world 跑通）
- ❌ 不要写 CI 配置（保持基建最小，CI 是后续独立决策）

## 与其它命令的关系

```
项目零测试设施
   ↓
/tc-test-setup        ← 你在这里（搭基建 + hello world）
   ↓
/tc-discuss           ← 聊需求
   ↓
/tc-prd               ← specs 三件套（含 AC-XXX）
   ↓
/tc-ai                ← 实施
   └─ N6 节点自动调 tc-qa-engineer
        └─ 基于 AC-XXX 设计**真正的业务测试用例**
   ↓
/tc-test              ← 跑测试
```

## 失败处理

| 场景 | 处理 |
|---|---|
| 项目根目录没有 `package.json` / `pyproject.toml` / `go.mod` 等任一项目描述 | "❌ 不是受支持的项目类型，请先初始化项目（`npm init` / `cargo init` / `go mod init` 等）" |
| 项目已有完整测试基建（vitest 装了 + hello 测试存在） | "✅ 项目已有测试基建，无需 setup。直接跑 `/tc-test`" |
| 项目部分有部分没（前端有 vitest，后端没 pytest） | 只对缺失的部分搭建，已有部分跳过并提示"已存在，跳过 frontend/" |
| 装依赖时网络失败 / 权限错 | 报错 + 给具体修复指引（换镜像源 / sudo 等），不静默忽略 |
| hello world 跑不通 | 暂停并报告失败原因，不假装成功；让用户决定继续修还是回滚 |

## 输入参数

```bash
/tc-test-setup                # 默认：探测全项目，分模块推荐
/tc-test-setup frontend       # 只搭前端
/tc-test-setup backend        # 只搭后端
/tc-test-setup --no-e2e       # 跳过 E2E 层（只搭单元/组件/集成层）
/tc-test-setup --dry-run      # 只输出方案，不执行搭建（用于看看推荐什么）
```
