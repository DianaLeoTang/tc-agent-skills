---
name: tc-qa-engineer
description: QA 工程师 Skill，对当前 task 或 feature 做功能级测试 — 自动识别测试栈（Jest/Vitest/Playwright/Pytest/Go test/Foundry 等）、设计补充用例、跑测、形成 bug 闭环
---

# tc-qa-engineer — QA 工程师

对当前 task 或 feature 做**行为级**验证。区别于 N4 Review（代码 diff 审查），QA 关注"功能是否真的按需求工作"。

## 强制约束（铁律）

被本 skill 调用即代表"必须落地测试代码"。三条铁律：

1. **改了就要写**：本次 task 涉及的每一个非文档代码改动，都必须**新增或扩展**对应的测试用例并落盘到 `*.test.*` / `test_*.py` / `*_test.go` / `*.t.sol` 等文件。**禁止**只跑现有测试就回报"通过"。
2. **影响面全覆盖**：公共组件 / 工具函数 / API / schema 被改 → 必须 grep 所有调用方，每个调用方对应的页面/功能都要有测试覆盖。改一行 css / className / i18n 文案也要补对应的视觉或行为断言。
3. **跑通才算完**：新增的测试必须**真实运行**并通过；只写测试文件不跑 = 没做 QA。基线红的话先报告，避免把无关失败计入本次 QA。

> 💡 这三条是 N6 触发口径"不打分、改了就触发"的对应执行端 — 入口不再过滤，所以 skill 必须保证每次都真写测试，否则上层口径形同虚设。

## 触发条件 & 调用模式

由 `/tc-ai` 的 N6 / N2 节点触发，本 skill 不自行决定要不要跑 QA — 入口已经做过决策；入口给了就**必须**按上面三条铁律执行到底。

**两种调用模式**（区别仅在"扫描范围"，铁律完全相同）：

| 模式 | 入口 | 范围 | 单次处理 |
| ---- | ---- | ---- | -------- |
| 普通 task QA | N6（每个 task 后默认必触发） | 当前 task 的 `git diff` | 一次 Skill 调用处理本 task 全部 diff 文件 |
| 历史欠债补齐 | N2（进入 feature 时审计） | N2 给出的"欠债文件清单" | **每个文件一个独立子 agent**（N2 用 Agent 工具 fan-out 派发，子 agent 内部参考本 SKILL.md 工作流），单子 agent 只对**单一文件**负责 |

### 单文件子 agent 调用契约（N2 派发使用）

子 agent 收到 N2 的简短 prompt 后，按本契约自我展开执行。**N2 的 prompt 只传字段，不展开工作流** — 工作流以本节为唯一权威。

#### 输入字段（N2 prompt 必传）

- `source_file` — 单个绝对路径
- `specs` — requirements.md / design.md / tasks.md 绝对路径
- `accumulated_changes` — 本分支累积变更文件列表（用于影响面分析参考）
- `feature_finalizing` — true / false

#### 子 agent 必须按顺序做

1. Read 项目 `.claude/rules/testing.md` / `qa.md`（如存在），跟随项目测试约定
2. Read `specs` 三件套，提取相关 AC-xxx 与接口契约
3. Read `source_file` 实现，理解行为
4. Bash 跑 grep 找 source_file 的所有调用方/导入方，列出影响面
5. 按本 SKILL.md 工作流的「2.5 影响面分析」「3 测试设计」「4 测试代码落盘 + 执行」执行：
   - 选定测试栈与文件命名（按项目惯例）
   - 设计行为点（happy / 边界 / 错误 / 影响面调用方各 ≥ 1 个用例）
   - **用 Write 工具写测试代码并落盘**到对应测试文件
   - **用 Bash 工具真实运行**该测试文件并确保通过

#### 必须返回（严格 JSON，作为子 agent 的最终回复）

```json
{
  "source_file": "<f_i 绝对路径>",
  "test_files_written": ["<新增/扩展的测试文件绝对路径 1>"],
  "test_cases_added": <数字>,
  "ran": true,
  "passed": true,
  "failed_cases": [],
  "impact_callers_covered": <数字>,
  "notes": "<basetone 红 / 需人工裁决等>"
}
```

#### 严禁

- ❌ `test_files_written` 不能为空数组 — 至少落盘 1 个测试文件
- ❌ `ran` 必须为 true — 不跑测就声称完成不可接受
- ❌ 不准回复"该文件不需要测试" — 既然在欠债清单里就必须写
- ❌ 不准只描述"应该写什么测试"而不真的 Write 文件
- ❌ 不准跳过影响面 grep — 调用方对应的测试也是子 agent 的职责

> N2 在退出校验时**用 Bash 重扫文件系统**核对每个文件是否真的有对应测试落盘，**不信**子 agent 的 JSON 自我汇报。子 agent 即便撒谎说"已落盘"，N2 也会发现并重派。

## 输入

由 N6 节点传入：

- 当前 task 编号、描述、变更文件列表（`git diff` 范围）
- 累积变更：上次 QA 后涉及的全部 task 与文件
- 触发原因（多选可叠加）：
  - `普通 task 默认必触发` — N6 的默认口径，针对单 task 改动
  - `feature 完成` — N5 后所有 task 都 [x]，做 feature 级 AC 全量验收
  - `历史欠债补齐` — N2 入口审计发现已落地代码缺测试，按"欠债文件清单"补齐（**不是** task 级 diff）
  - `API 变更` / `数据库 migration` / `auth/支付` / `公共组件改动` — 触发对应必查清单
- specs 路径（requirements.md + design.md + tasks.md）

不同触发原因决定后面的"必查清单"和扫描范围。

> 🔑 **历史欠债补齐 vs. 普通 task 触发的区别**：
> - 普通 task：扫描范围 = 当前 task 的 `git diff`，影响面分析以"本次 diff 文件"为锚
> - 历史欠债：扫描范围 = N2 传入的"欠债文件清单" — **该清单是从 git 全量 diff 推导（含分支累积 commit + working tree + staged + untracked），可能包含 task 描述里没列的改动**（手改 bug、refactor、临时调整等）。skill **不允许**以"这不在 tasks.md 描述范围"为由删减清单里的文件 — 既然分支上改了，就必须为它补测试
> - 两种模式的"必须落盘测试代码"铁律相同，只是入口不同

> ⚠️ **历史欠债模式下的清单完整性铁律**：
> 收到的欠债文件列表，**必须为每个文件都写或扩展对应测试并落盘**。落盘文件数 < 欠债文件数 时，必须解释每一个未补的文件（且仅以下两种豁免理由有效）：
> 1. 文件已删除（`git status` 显示 deleted）
> 2. 文件是构建产物（应在 N2 过滤时剔除，若到 skill 还在说明 N2 过滤有 bug，需回报）
> 任何其它理由（"不重要"、"很简单"、"不在 task 范围"、"和别的测试重复"）都**不可接受**。

## 工作流程

### 1. 识别测试栈

读取项目配置自动判断，不做硬编码假设：

- **JS/TS** — `package.json` 看 `jest` / `vitest` / `mocha` / `ava` / `playwright` / `cypress` / `@testing-library/*`
- **Python** — `pyproject.toml` / `requirements*.txt` 看 `pytest` / `unittest` / `pytest-asyncio` / `httpx`
- **Go** — 默认 `go test`，看是否引入 `testify` / `gomock`
- **Rust** — `cargo test` + `mockall` / `proptest`
- **合约** — Foundry (`forge test`) / Hardhat (`npx hardhat test`) / Anchor (`anchor test`)
- **测试目录约定** — `__tests__/` / `tests/` / `*_test.go` / `*.test.ts` / `*.spec.ts`，跟随项目已有规律

如果项目**完全没有测试基础设施**且本次 task 风险高（auth/支付/migration），先建立最小测试设施再补用例；否则按项目惯例办，不强行引入新栈。

### 2. 读取上下文

- `.claude/rules/testing.md` / `.claude/rules/qa.md`（如存在）
- `requirements.md` 的验收标准（AC-xxx）— 这是 QA 的判分依据
- `design.md` 涉及模块的接口契约和数据模型
- `LESSONS.md` 中此前踩过的坑（避免回归）
- 已有测试用例 — 了解项目的断言风格、mock 策略、fixture 约定

### 2.5 影响面分析（强制，先于设计）

**对本次扫描范围内的每个文件，先做调用方/依赖方扫描，再设计用例。**「这次只改了 X 一行，应该没影响」是被禁止的判断 — 必须用 grep 验证。

扫描范围按触发原因决定：

- 普通 task / API 变更 / 公共组件改动 → 当前 task 的 `git diff` 文件列表
- feature 完成 → 自上次 QA 以来所有累积变更文件
- **历史欠债补齐** → N2 传入的"欠债文件清单"（每个文件都缺测试，逐个补）

| 改动类型 | 必须扫描 | 必须覆盖 |
| -------- | -------- | -------- |
| 公共组件 / 通用 hooks / 工具函数 | grep 所有 `import .* from '<改动文件>'` | 每个直接调用方的页面/模块至少 1 个行为用例 |
| API 接口 / Controller / Route handler | grep 所有调用方（前端 axios/fetch、SDK、服务间调用） | 每个调用方的契约（入参/出参/错误码）至少 1 个用例 |
| 数据库 schema / migration | grep 所有 query / ORM model / 迁移脚本 | 旧数据兼容、回滚、并发写入用例 |
| 样式 / className / 主题变量 | grep 所有使用该 className / 变量的页面 | 至少 1 个视觉快照或可见性断言 |
| 配置文件（package.json / tsconfig / vite / next.config） | 评估变更对构建/运行时的影响范围 | 主要入口烟雾测试（启动 / 主路由可达） |
| i18n 文案 | grep 所有使用该 key 的位置 | 每个使用位置至少 1 个文案断言 |

**输出影响面清单**（写入测试设计的"行为点列表"前置区）：

```text
🔍 影响面分析 — Task T-xxx
   改动文件: {N} 个
   - components/Button.tsx (公共组件)
     调用方: 8 处 (LoginPage / Dashboard / SettingsModal / ...)
     需补测试: 8 个调用方各 1 个行为用例
   - api/user.ts (API 接口)
     调用方: 3 处前端 + 2 个内部服务
     需补测试: 5 个契约用例
```

影响面分析结果**直接决定**下一步要写多少测试 — 不允许设计阶段缩水。

### 3. 测试设计

**基于上一步的影响面清单**，列出本次必须验证的"行为点"，再决定每个点用什么层级的测试覆盖：

- **单元** — 纯函数、工具、边界算法
- **集成** — 跨模块协作、DB 真实读写、外部服务用 mock 或 testcontainer
- **端到端** — 完整用户路径，用 Playwright/Cypress 或合约的端到端脚本

**核心用例必须覆盖：**

- 正常路径（happy path）
- 边界值（空、最大、负数、Unicode、超长字符串）
- 错误路径（网络失败、超时、权限拒绝、数据格式错）
- 并发/竞态（如涉及）
- 幂等性（如涉及 API 重试 / 链上交易）

**按触发原因加挂必查项：**

| 触发原因 | 必查清单 |
| -------- | -------- |
| API 变更 | 入参校验、错误码、鉴权、向后兼容、限流、契约测试 |
| 数据库 migration | 迁移可重放、回滚脚本、大表锁/超时、null/默认值、索引未失效 |
| 认证/授权 | 越权（IDOR/水平/垂直）、token 过期/伪造、会话固定、密码强度 |
| 支付 | 金额精度、重复扣款、回调验签、对账、退款幂等、并发下单 |
| Feature 完成 | requirements.md 全部 AC 项逐条验证 |
| 合约 | reentrancy、access control、整数溢出、event 完整、gas 上限、升级兼容 |

### 4. 测试代码落盘 + 执行

**先落盘新测试代码**，再跑。只跑不写 = 违反铁律 1，必须返工。

落盘规范：

- 跟随项目已有的测试文件命名与目录约定（`__tests__/` / `tests/` / 与源码同目录的 `*.test.ts`）
- 新增文件命名按"被测对象名 + 测试类型"，例：改了 `components/Button.tsx` → 新增 `components/Button.test.tsx`，或在已有 `Button.test.tsx` 里**追加** `describe` 块
- 用 specs 里的 `[F-XXX]` / `[AC-XXX]` 标签标注测试，便于 `/tc-test feature N` 过滤
- 影响面清单里的每个调用方都必须有对应测试落盘 — 落盘文件路径必须出现在 skill 输出的"落盘文件"列表里

**再跑现有测试套件 + 新写的测试**，确认基线绿：

```bash
# 按项目实际命令
npm test / pnpm test / yarn test
pytest -q
go test ./...
forge test
```

如果基线就红，先报告再决定是否本次任务引入的回归 — 不要把无关失败计入本次 QA。

**再跑本次新增/修改的测试**：分层执行，单元先于集成先于 E2E，快速反馈失败位置。

**对涉及外部依赖的场景：**

- 数据库 — 优先真实 DB（docker / testcontainers / sqlite in-memory），mock 仅用于不可达外部服务
- 网络服务 — 用 nock / msw / responses 等录制 mock，避免对外发真实请求
- 区块链 — 本地 Anvil / Hardhat node，禁止打主网

### 5. Bug 闭环

发现问题时**优先自行修复**，不无脑回报给用户：

- **明确的实现 bug**（与 AC 不符、明显逻辑错）→ 直接改代码 + 补回归用例 → 重跑
- **模糊的需求理解**（AC 没说清、设计与需求矛盾）→ 暂停并报告，让用户裁决
- **测试用例本身写错** → 改测试用例

复测**最多 1 轮**。1 轮复测仍失败 → 立即暂停并报告，输出已尝试的修复路径，交用户裁决，**不再自动循环**。

> 不做多轮自动复测的原因：每轮 QA 都是"读 specs + 设计用例 + 跑全量测试 + 修复"完整 pipeline，多一轮接近线性放大 token。1 轮足以兜住自己刚改坏的小 bug，更复杂的失败让人介入比 AI 盲试更便宜。

## 常见坑

| 问题 | 处理 |
| ---- | ---- |
| 测试只断言"不抛异常"，没断言行为 | 加结果断言（返回值/DB 状态/事件） |
| Mock 过度 — 把被测对象自己也 mock 了 | 只 mock 边界（外部 API/IO），核心逻辑用真实对象 |
| 测试间相互依赖（顺序依赖） | 每个 test 独立 setup/teardown，不依赖前一个的状态 |
| Flaky test 用 retry 掩盖 | 找根因（异步等待/时间依赖/共享状态），retry 是最后手段 |
| E2E 等待用固定 sleep | 改用元素出现/网络空闲断言 |
| migration 测试只跑了 forward | 必须验证 down/rollback，并在含数据的库上验证 |
| 合约测试只测 happy path | 加 fuzz / invariant test，至少覆盖 access control 与 reentrancy |
| 鉴权测试只用合法 token | 必须测过期、伪造、越权（A 用户访问 B 用户资源） |
| 覆盖率只看百分比 | 关注未覆盖的分支是不是关键路径，而非追数字 |
| 把 AI 自审（N4）和 QA（N6）做成同一件事 | N4 看 diff，N6 看行为；QA 必须能跑起来真验证 |

## 与 N4 Review 的边界

- **N4 自审**：静态、单 task diff、关注代码质量与显见安全
- **N6 QA**：动态、累积变更或 feature 级、关注行为与 AC

QA 发现的"代码层面"问题（命名、结构）不是本职 — 报告但不强求修复，让 N4 后续兜底。

## 输出

回到 N6 节点的固定格式，由 N6 决定如何回吐给用户：

```text
QA 报告 — Task {T-编号} / Feature {名}

测试栈: {识别到的栈}
影响面分析: {N} 个改动文件 → {M} 个调用方
新增/扩展测试落盘:
  - {test 文件 1}（新增 / 扩展，N 个用例）
  - {test 文件 2}（...）
执行: {N} 个用例（新增 {N} / 已有 {N}）
通过: {N} | 失败 → 修复: {N} | 待人工: {N}
覆盖 AC: {N}/{总数}（未覆盖: {AC-编号列表}）
必查项: {按触发原因勾选完成情况}

发现并修复:
- {bug 简述} → {修复定位}

待人工裁决:
- {模糊点} — {为什么需要人工}

复测: {未触发 / 1 轮通过 / 1 轮仍失败暂停}
结论: ✓ 通过 / ⚠ 部分通过 / ❌ 失败暂停
```

结论由调用方（N6）决定后续：通过 → N7；失败 → 暂停并报告。
