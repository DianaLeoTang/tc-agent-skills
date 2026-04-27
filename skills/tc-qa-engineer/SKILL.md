---
name: tc-qa-engineer
description: QA 工程师 Skill，对当前 task 或 feature 做功能级测试 — 自动识别测试栈（Jest/Vitest/Playwright/Pytest/Go test/Foundry 等）、设计补充用例、跑测、形成 bug 闭环
---

# tc-qa-engineer — QA 工程师

对当前 task 或 feature 做**行为级**验证。区别于 N4 Review（代码 diff 审查），QA 关注"功能是否真的按需求工作"。

## 触发条件

由 `/tc-ai` 的 N6 节点根据评分或 must-trigger 条件调用。该 skill 不自行决定要不要跑 QA — 入口已经做过决策。

## 输入

由 N6 节点传入：

- 当前 task 编号、描述、变更文件列表（`git diff` 范围）
- 累积变更：上次 QA 后涉及的全部 task 与文件
- 触发原因：`feature 完成` / `API 变更` / `数据库 migration` / `auth/支付` / `累积 5 task` / `评分 ≥ 8`
- specs 路径（requirements.md + design.md + tasks.md）

不同触发原因决定后面的"必查清单"。

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

### 3. 测试设计

**先列出本次必须验证的"行为点"**，再决定每个点用什么层级的测试覆盖：

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

### 4. 测试执行

**先跑现有测试套件**，确认基线绿：

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
