# N6: QA — 写测试 + 跑测试 + 覆盖率

**两种调用模式**：

- **内部模式**：被 `/tc-ai` 流程调用（前面经过 N3/N4/N5），有 task 上下文（T-编号、specs 路径、feature 收尾标志）
- **独立模式**：用户在任意项目里直接 `/tc-ai-nodes:N6-qa-eval`，无 task 上下文。本节点能识别"无上下文"状态自动降级，跳过 specs 相关逻辑，只做"扫描 → 写测试 → 跑测试 → 出覆盖率"

行为对两种模式相同：四路扫描分支改动 → 缺测试就调 `tc-qa-engineer` 写 → 跑测 + 覆盖率。

## 进入

```text
🧪 N6 — {内部: Task T-编号 / 独立: 当前分支}
```

## 跳过判定（极小白名单）

跑下面这段四路合并扫描。**不准只跑 `git diff --name-only`** —— commit 后会输出空被误判。

```bash
BASE=$(git rev-parse --verify --quiet main \
    || git rev-parse --verify --quiet master \
    || git rev-parse --verify --quiet develop \
    || git rev-parse --verify --quiet origin/main \
    || git rev-parse --verify --quiet origin/master)
MERGE_BASE=$(git merge-base HEAD "${BASE:-HEAD}" 2>/dev/null || echo "")

{
  [ -n "$MERGE_BASE" ] && git diff --name-only "${MERGE_BASE}...HEAD"
  git diff --name-only HEAD
  git diff --name-only --cached
  git ls-files --others --exclude-standard
} | sort -u \
  | grep -E '\.(ts|tsx|js|jsx|vue|svelte|py|go|rs|sol|java|kt|swift|css|scss|html|json|yml|toml)$' \
  | grep -vE '\.(test|spec)\.|__tests__/|^tests?/|/dist/|/build/|/\.next/|/coverage/|/node_modules/'
```

| 输出 | 处理 |
|---|---|
| 空 | ⏭ 跳过 → 内部模式进 N7；独立模式直接结束 |
| 全部是文档（`.md` / `.txt` / `.rst`）| ⏭ 跳过 |
| 有非文档代码改动 | **必触发**，调 tc-qa-engineer |

## 调用 tc-qa-engineer

**Skill 调用与说明文本同一回合发出**，不准只 narrate "我将调用..." 然后下一回合再发。

`args` 字段（可选项缺省就传"无"）：

- **触发原因**（必填）：`普通 task 默认必触发` / `feature 收尾` / `API 变更` / `auth/支付` / `数据库 migration` / `公共组件改动` / `独立模式补测试`（命中即加）
- **当前 task**（可选，独立模式留空）：`T-{编号}` + 描述
- **变更文件**（必填）：上面四路扫描的完整列表
- **累积变更**（可选，独立模式留空）：自上次 QA 后涉及的所有 task 与文件
- **specs 路径**（可选，独立模式留空）：`requirements.md` / `design.md` / `tasks.md` 绝对路径
- **feature 收尾**（可选，独立模式留空）：N5 后所有 task 都 `[x]` 则 `true`

调用：

```
Skill(skill="tc-qa-engineer", args="<上面字段拼成一段说明>")
```

## 跑覆盖率（写完测试、跑通后）

skill 通过后，按项目实际栈跑覆盖率（自动探测，**不写新配置**）：

| 栈 | 命令 |
|---|---|
| Vitest | `npx vitest run --coverage` |
| Jest | `npx jest --coverage` |
| Pytest | `pytest --cov --cov-report=term-missing` |
| Go | `go test -cover ./...` |
| Foundry | `forge coverage` |

输出汇总到终端：lines / branches / functions 百分比 + 未达门槛文件（如 specs 里有定义门槛）。

## 决策输出

```text
🧪 触发 QA — 原因: {理由}
   变更文件: {N} 个
   {内部模式额外: feature 收尾 true/false}
   → Skill(tc-qa-engineer)

   {返回后:}
   ✓ 通过 — 新增/扩展测试 {N} 个
   或 ⚠ 修复后复测 1 轮通过
   或 ❌ 1 轮复测仍失败 → 暂停

📊 覆盖率: lines X% / branches X% / functions X%
   未达门槛: {file 列表，如有}
```

## 退出

- **内部模式** → N7
- **独立模式** → 结束（用户直接看终端报告）

## 复测策略

QA 通过 → 继续。失败 → 修复后**复测 1 轮**。1 轮仍失败 → **暂停**。
