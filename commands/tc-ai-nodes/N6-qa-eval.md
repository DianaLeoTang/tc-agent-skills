# N6: QA — 写测试 + 跑测试

N3 写代码，N6 写测试。改了代码就必须调 `tc-qa-engineer` 写测试，**不打分、不做"小改动跳过"判断**。

## 进入

```text
🧪 N6 — Task {T-编号}
```

## 跳过判定（极小白名单）

跑下面这段四路合并扫描。**不准只跑 `git diff --name-only`**——commit 后会输出空，把刚做完的 task 误判成"无改动"。

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
| 空 | ⏭ 跳过（task 是讨论 / NO-OP）→ N7 |
| 全部是文档（`.md` / `.txt` / `.rst`）| ⏭ 跳过 → N7 |
| 有非文档代码改动 | **必触发**，调 tc-qa-engineer |

## 调用 tc-qa-engineer

**Skill 工具调用与说明文本必须在同一回合发出**，不准只 narrate "我将调用..." 然后下一回合再发。

```
Skill(skill="tc-qa-engineer", args="
  触发原因: 普通 task 默认必触发 [+ feature 收尾 / API 变更 / auth/支付 / 数据库 migration / 公共组件改动]（命中即加）
  当前 task: T-{编号} + 描述
  变更文件: <上面四路扫描的完整列表>
  累积变更: <自上次 QA 后的所有 task 与文件>
  specs 路径: requirements.md / design.md / tasks.md 绝对路径
  feature 收尾: <true / false>  # N5 标记后所有 task 都 [x] 则 true
")
```

判定 `feature 收尾`：N5 标记完成后读 `tasks.md`，所有 task 都 `[x]` → `true`，按 feature 收尾口径全量验收 AC。

## 决策输出

```text
🧪 触发 QA — 原因: {理由}
   变更文件: {N} 个
   feature 收尾: {true/false}
   → Skill(tc-qa-engineer)

   {返回后:}
   ✓ 通过 — 新增/扩展测试 {N} 个
   或
   ⚠ 发现 {N} 个问题 → 修复后复测 1 轮通过
   或
   ❌ 1 轮复测仍失败 → 暂停交用户裁决
```

## 退出

```text
→ N7
```

## 复测策略

QA 通过 → 继续。失败 → 修复后**复测 1 轮**。1 轮仍失败 → **暂停**，不再自动循环（更复杂的失败让人介入比 AI 反复试便宜）。
