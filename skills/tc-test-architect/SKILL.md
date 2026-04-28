---
name: tc-test-architect
description: 测试基础设施架构师 Skill — 按项目实际栈推荐并搭建完整测试方案（依赖+目录+配置+hello world），强制用户确认后再装；不写业务测试用例
---

# tc-test-architect — 测试基础设施架构师

为零测试设施的项目设计一套**符合项目实际栈**的测试方案，给用户审阅，确认后搭建完成并跑通 hello world。

## 触发条件

由 [`/tc-test-setup`](../../commands/tc-test-setup.md) 命令调用。该 skill 不应被其它流程自动调用——基建搭建是一次性、不可逆的操作，必须由用户显式触发。

## 输入

由 `/tc-test-setup` 传入：

- 项目根目录路径
- 探测到的栈摘要（前端/后端/合约 子目录的语言+框架版本）
- 用户偏好（如 `--no-e2e` / `--dry-run` / 只搭某子目录）

## 核心原则

1. **以项目实际栈为准**——禁止套用通用模板或硬塞项目未使用的测试库
2. **强制用户确认**——装依赖前**必须**输出方案让用户审阅；用户可改方案；未确认前不动任何文件
3. **最小可用**——只装必要依赖，不为了"完整性"塞 Storybook / Chromatic / Stryker 等高级工具
4. **跑通 hello world**——hello world 没跑过等于没搭

## 工作流程

### 1. 识别项目栈

读取项目配置自动判断，**不做硬编码假设**：

#### JS/TS 前端

读 `package.json`：

- **框架** — `react` / `vue` / `svelte` / `solid` / `next` / `nuxt` / `astro` / `remix`
- **构建工具** — `vite` / `webpack` / `next` / `turbopack` / `parcel`
- **TS** — 看 `tsconfig.json` 是否存在
- **样式方案** — `tailwindcss` / `styled-components` / `@emotion/*` / CSS Modules / UnoCSS
- **包管理器** — 看 lock 文件：`pnpm-lock.yaml` → pnpm，`yarn.lock` → yarn，`package-lock.json` → npm，`bun.lockb` → bun

#### Python 后端

读 `pyproject.toml` / `requirements*.txt`：

- **Web 框架** — `fastapi` / `flask` / `django` / `aiohttp` / `starlette`
- **异步** — 是否有 `anyio` / `asyncio` 依赖
- **HTTP 客户端** — `httpx` / `requests`（影响测试用什么 mock 工具）
- **包管理** — `poetry` / `pip` / `uv` / `pdm`

#### Go

读 `go.mod`：

- **Web 框架** — `gin` / `echo` / `fiber` / 标准 `net/http`
- **DB 库** — `gorm` / `sqlx` / 标准 `database/sql`
- 是否已引入 `testify` / `gomock`

#### Rust / 合约 / 其它

- **Rust** — `Cargo.toml` 看是否引入 `mockall` / `proptest` / `tokio-test`
- **合约** — `foundry.toml` → Foundry，`hardhat.config.*` → Hardhat，`anchor.toml` → Anchor，`truffle-config.*` → Truffle
- **Flutter** — `pubspec.yaml` 看 `flutter_test` 是否在 `dev_dependencies`

#### 多模块

monorepo 项目（前端 + 后端 + 合约 同仓库）→ 分别识别每个子目录的栈，**分别推荐**，不强行统一。

### 2. 推荐方案（按栈分支）

每个识别到的子项目都生成一份独立方案。下面是**参考方案库**，最终输出必须根据实际探测结果调整：

#### React + TypeScript + Vite

```yaml
工具:
  - 单元/组件: vitest + @testing-library/react + jsdom
  - mock: msw
  - E2E: @playwright/test

devDependencies:
  - vitest
  - @testing-library/react
  - @testing-library/jest-dom
  - @testing-library/user-event
  - @vitest/coverage-v8
  - jsdom
  - msw
  - "@playwright/test"  # 仅 --no-e2e=false 时

scripts:
  test: "vitest run"
  test:watch: "vitest"
  test:coverage: "vitest run --coverage"
  test:e2e: "playwright test"

config_files:
  - vitest.config.ts          # globals/jsdom/setupFiles/coverage
  - playwright.config.ts      # baseURL/projects (chromium/firefox/webkit)
  - src/test/setup.ts         # 引入 @testing-library/jest-dom + MSW server
  - src/test/msw-server.ts    # MSW handlers 入口

dirs:
  - src/__tests__/   # 单元/组件测试（或就近 *.test.ts*）
  - e2e/             # E2E 用例
  - src/test/        # fixtures/setup/mocks

hello_world:
  - src/__tests__/hello.test.ts:  it('1+1=2', () => expect(1+1).toBe(2))
  - e2e/smoke.spec.ts:            test('homepage loads', ...)

coverage_thresholds:
  lines: 70
  branches: 60
  functions: 70
  statements: 70
```

#### Vue 3 + Vite

同 React，但：
- `@testing-library/react` → `@testing-library/vue`
- 增加 `@vue/test-utils`
- jsdom 不变，setup 文件相同

#### Next.js

- 用 vitest 而非 Jest（除非项目已用 Jest）
- vitest config 需 `defineConfig({ plugins: [react(), tsconfigPaths()] })`
- E2E 默认 Playwright

#### Svelte / SvelteKit

- 单元/组件：vitest + @testing-library/svelte
- E2E：Playwright（SvelteKit 默认配置已支持）

#### Python + FastAPI

```yaml
工具:
  - 测试框架: pytest + pytest-asyncio
  - HTTP 测试客户端: httpx + ASGITransport (FastAPI 推荐)
  - 覆盖率: pytest-cov

依赖（dev）:
  - pytest
  - pytest-asyncio
  - pytest-cov
  - httpx

config_files:
  - pyproject.toml [tool.pytest.ini_options]:
      asyncio_mode = "auto"
      testpaths = ["tests"]
  - 或单独 pytest.ini

dirs:
  - tests/
  - tests/conftest.py    # fixture 入口

hello_world:
  - tests/test_hello.py:   def test_hello(): assert 1 + 1 == 2

scripts (Makefile or pyproject scripts):
  test: "pytest -q"
  test:cov: "pytest --cov=src --cov-report=term-missing"
```

#### Python + Django

- pytest-django 替换原生 unittest
- 增加 `DJANGO_SETTINGS_MODULE` 配置

#### Go

```yaml
工具:
  - 测试: go test (标准库)
  - 断言: testify (推荐) - 仅当项目尚未引入
  - mock: gomock 或 testify/mock

config:
  - go.mod 加入 require github.com/stretchr/testify

dirs:
  - 跟随 Go 惯例，*_test.go 与源码同目录

hello_world:
  - internal/hello/hello_test.go:
      func TestHello(t *testing.T) { assert.Equal(t, 2, 1+1) }

scripts (Makefile):
  test: "go test ./..."
  test:cov: "go test -cover ./..."
```

#### Foundry (Solidity)

```yaml
工具:
  - forge test (内置)
  - forge coverage

config:
  - foundry.toml [profile.default]:
      src = "src"
      test = "test"

dirs:
  - test/

hello_world:
  - test/Hello.t.sol:
      contract HelloTest is Test {
          function test_hello() public { assertEq(1+1, 2); }
      }
```

#### Hardhat

```yaml
工具:
  - mocha + chai (Hardhat 默认)
  - ethers / viem (按项目)
  - hardhat-coverage

依赖（dev）:
  - chai
  - mocha
  - "@nomicfoundation/hardhat-toolbox"
  - solidity-coverage

scripts:
  test: "hardhat test"
  test:cov: "hardhat coverage"
```

### 3. 输出方案给用户审阅

把推荐方案以**可读的结构化文本**输出（见 [`/tc-test-setup`](../../commands/tc-test-setup.md) Step 2 的示例格式）。

**关键要素**（缺一不可）：

- 检测到的栈摘要（让用户验证你识别正确）
- 要装的依赖清单（精确版本范围，让用户能预估 lock 影响）
- 要新增的脚本（让用户知道之后怎么跑）
- 要新增的配置文件路径（让用户知道哪里会被动）
- 要新增的目录
- hello world 测试预览（让用户知道这只是无业务意义的占位）
- 推荐覆盖率门槛（明确告知是建议，可改）
- **明确的"请确认"卡点**

### 4. 强制用户确认

**未收到用户明确确认前，不动任何文件**。

用户可能的回复：

| 回复 | 处理 |
|---|---|
| "确认" / "OK" / "go" | 进入 Step 5 执行搭建 |
| "换 jest" / "不要 e2e" / "覆盖率改 80%" | 调整方案 → 重新输出 → 再次等待确认 |
| "拒绝" / "算了" | 终止，不动任何文件 |
| 模糊回复（如"嗯"） | **不要执行**，再次确认问"是确认开始搭建吗？" |

### 5. 执行搭建

按确认方案分步执行，每一步出错立即停止报告：

1. **装依赖** — 用项目实际包管理器（前端看 lock file，Python 看是否有 poetry/pyproject.toml）
2. **写配置文件** — 用 Write 工具写完整内容；如配置文件已存在但内容不同，**先输出 diff 给用户确认**再写（避免覆盖）
3. **建目录** — `mkdir -p`
4. **写 hello world 测试** — 内容必须是无业务意义的最小测试
5. **更新 scripts**（package.json / Makefile / pyproject.toml）— 仅追加，不覆盖已有 script

### 6. 跑通 hello world 验证

立刻跑一次：

- 前端 → `npm test`（按实际包管理器）
- 后端 → `pytest -q`
- Go → `go test ./...`
- Foundry → `forge test`

**输出必须是 "1 passed" 或同等成功标记**，否则视为搭建失败。

### 7. 输出最终报告

```text
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✅ tc-test-architect 搭建完成
━━━━━━━━━━━━━━━━━━━━━━━━━━━━

✅ 已装依赖:
  前端: vitest, @testing-library/react, msw, @playwright/test
  后端: pytest, pytest-asyncio, httpx

✅ 已写配置:
  - vitest.config.ts
  - playwright.config.ts
  - src/test/setup.ts
  - src/test/msw-server.ts
  - pytest.ini

✅ 已建目录:
  - src/__tests__/
  - e2e/
  - tests/

✅ Hello world 跑通:
  - vitest:    1 passed (40ms)
  - playwright: 1 passed (2.3s)
  - pytest:    1 passed (0.1s)

📌 推荐覆盖率门槛（已写入配置）:
  lines 70 / branches 60 / functions 70

⏭ 下一步: /tc-discuss → /tc-prd → /tc-ai （让 N6 调 tc-qa-engineer 设计真实业务测试）
```

## 反模式（**禁止**）

| 反模式 | 替代做法 |
|---|---|
| 不识别项目栈，套用通用 React 模板 | 严格按 `package.json` 推断 |
| 跳过用户确认直接装依赖 | **必须**等用户明确"确认"才动手 |
| 硬塞 Storybook、Chromatic、Stryker、Snapshot 测试等高级工具 | 只装跑测试必需的最小集 |
| 写带业务含义的 hello world（如"测试用户登录"） | hello world 必须无业务意义（`1+1=2` / `homepage loads`） |
| 修改业务代码（除非阻塞 hello world 跑通） | 不动业务代码 |
| 写 CI 配置 | 由用户后续单独决策（CI 平台/凭据/触发条件都需要更多信息） |
| hello world 没跑就报告完成 | 必须跑通才算成功 |
| 配置文件已存在直接覆盖 | 输出 diff 给用户确认后再写 |
| 不识别包管理器，统一用 npm | 看 lock 文件决定 pnpm/yarn/npm/bun |

## 常见坑

| 问题 | 处理 |
|---|---|
| 项目用 pnpm 但你跑了 npm install | 先看 `pnpm-lock.yaml`，用 `pnpm add -D` |
| Vite 项目想跑 vitest 但 import.meta 报错 | 在 `vitest.config.ts` 用 `defineConfig` 而非 `defineProject` |
| Next.js + vitest 路径别名（`@/`）找不到 | 装 `vite-tsconfig-paths` 并加进 vitest config |
| Playwright 安装后浏览器没装 | 提示用户跑 `npx playwright install` |
| Python 项目用 pytest-asyncio 没配 mode | 在 pyproject.toml 加 `asyncio_mode = "auto"` |
| Go 项目装了 testify 但 hello 测试用了原生 | 二选一，不要混用风格 |
| 用户已有 `vitest.config.ts` 但缺 coverage 配置 | 输出 diff，**不覆盖**，让用户合并 |
| MSW v2 API 与 v1 不同 | 装 v2 时 setup 用 `setupServer` from `msw/node`，handler 用 `http.get` 而非 `rest.get` |

## 输出

回到 `/tc-test-setup` 命令，按 Step 7 报告格式输出。**hello world 跑不通时**必须暴露失败原因，不假装成功。
