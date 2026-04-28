# N2: 进入 Feature

## 进入消息

```text
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📂 Feature {N}/{总数} — {feature名}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

## 总纲：N2 要做两件**独立**的事

| 支线 | 处理对象 | 出口 |
| ---- | -------- | ---- |
| **A. 测试欠债审计**（步骤 3） | 已完成 `[x]` 但没写测试的代码 | 调 `tc-qa-engineer` 补测试 → 跑通 |
| **B. 执行计划**（步骤 4，默认主线） | 未完成 `[ ]` / `[CHANGED]` 的 task | 进入 N3 开发新代码 → N4 → N5 → N6 写新测试 |

> 🔑 **A 与 B 不互相替代**：
> - 有 `[ ]` 待执行 task → **必须**走 B 出执行计划进 N3，不能用"已经在补欠债"为由跳过新开发
> - 有 `[x]` 缺测试 → **必须**走 A 补齐，不能用"还有新 task 要做"为由把欠债拖到 feature 收尾
> - 两条支线在本节点都跑：先 A 后 B（A 先跑是因为补欠债不依赖新 task；B 紧随其后正常进 N3）
> - 仅当 A 通过且 **B 没有任何 `[ ]` task** 时，才能整个 feature 跳过

## 步骤

### 1. 读取 specs

读取该 feature 的 `requirements.md`、`design.md`、`tasks.md`。

### 2. 断点恢复

逐项识别 task 状态：

- `[x]` 已完成 → 跳过 N3 开发，但代码可能没测试（进入步骤 3 审计）
- `[DROPPED]` → 跳过
- `[CHANGED]` → 按更新后描述执行（计入步骤 4 执行计划）
- `[ ]` → 未执行（计入步骤 4 执行计划）

### 3. 测试欠债审计（支线 A —— 已完成代码缺测试 → 立即补齐）

**目的**：把已经标 `[x]` 但没写过测试的 task，对应代码的测试补回来。**与步骤 4 的执行计划是独立的两件事**，无论本 feature 还有多少 `[ ]` 待开发，欠债该补就补；无论欠债多严重，待办的新 task 也照常进 N3 开发。

#### 3.1 列出本分支上的全部代码改动文件（以 git 为权威）

**核心原则**：以 **git** 为唯一权威基准，**不**依赖 `design.md` / `requirements.md` 来圈定范围。分支上**任何代码改动**都要审计，不分"是否在某个 task 描述里"。

> ⚠️ 真实场景里，开发分支上常有大量改动**不在任何 task 范围内**：临时 bug 修复、refactor、改样式、调样板等。如果用 `design.md` 当主源去过滤，这些改动会被全部漏掉，结果就是"只给一个测试" —— 这是必须避免的反模式。

**取四个 git 来源，全部叠加去重**（不是优先级）：

| 来源 | 命令 | 捕获什么 |
| ---- | ---- | -------- |
| A. 分支 vs main 已 commit 的累积改动 | `git diff --name-only $(git merge-base HEAD main)...HEAD` | 本分支所有 commit 涉及的文件（多 commit 累积） |
| B. working tree 未 commit 改动 | `git diff --name-only HEAD` | 本地手改还没 commit 的文件 |
| C. 已 staged 但未 commit 改动 | `git diff --name-only --cached` | 已 `git add` 但未提交的文件 |
| D. 未跟踪（untracked）新文件 | `git ls-files --others --exclude-standard` | 新建但还没 `git add` 的代码文件 |

**最终清单 = A ∪ B ∪ C ∪ D 去重**。

**过滤规则**（仅做语言类型过滤，不做范围过滤）：
- ✅ 保留：`.ts/.tsx/.js/.jsx/.vue/.svelte/.py/.go/.rs/.sol/.java/.kt/.swift/.css/.scss/.html` 等代码后缀
- ❌ 剔除：`.md/.txt/.lock/LICENSE` 纯文档；测试文件本身（避免自指）；构建产物（`dist/` / `build/` / `.next/` / `coverage/`）

`design.md` / `requirements.md` 里列的"实现文件清单"**仅作分类参考**（用于在审计报告里把文件按"task 涉及 / task 外"分组展示，方便用户判断），**不允许用它把 git 里实际存在的文件过滤掉**。

#### 3.1.x 反模式（与本步骤强相关）

- ❌ **不准用 design.md / requirements.md 当主源** — task 描述追不上实际改动
- ❌ **不准只看 `git log` 的某些 commit** —— working tree / staged / untracked 都要查，否则漏掉本地改动
- ❌ **不准以"这不在当前 task 范围"为由跳过某些文件** —— 既然在分支上动了，就要为它写测试
- ❌ **不准把列表给 skill 时砍短** —— A/B/C/D 去重后多少就是多少，整份传给 `tc-qa-engineer`

#### 3.2 比对测试覆盖

对每个代码文件 `path/to/foo.ext`，检查是否存在对应测试文件（满足任一即视为有覆盖）：

| 语言 | 测试文件位置 |
| ---- | ------------ |
| TS/JS/Vue/Svelte | 同目录 `foo.test.{ts,tsx,js,jsx}` / `foo.spec.*` / 上层 `__tests__/foo.test.*` / 镜像 `tests/<相对路径>.test.*` |
| Python | 同目录或 `tests/` 下 `test_foo.py` / `foo_test.py` |
| Go | 同目录 `foo_test.go` |
| Rust | 同文件 `#[cfg(test)] mod tests` 或 `tests/foo.rs` |
| Solidity | `test/foo.t.sol` / `test/Foo.t.sol` |
| CSS/HTML | 至少有一个 e2e / 视觉快照 / 组件测试断言到该文件的 className 或路径 |

未找到任一对应测试文件 → 标为"欠债文件"。

#### 3.3 输出审计报告（强制）

报告必须**按"task 内 / task 外"分组**显示，让用户一眼看到分支上是否有 tasks.md 没覆盖的改动：

```text
🔍 测试欠债审计 — Feature {名}
   git 来源汇总: A {N1} + B {N2} + C {N3} + D {N4} 去重 = {总数} 个代码文件
   
   ✓ 已有测试: {M} 个
   ⚠ 缺测试 (欠债): {K} 个
   
   ┌─ 分类（仅供查看，不作过滤）
   │
   ├ 在 design.md / requirements.md 描述范围内（task 内）:
   │   - components/Button.tsx
   │   - api/user.ts
   │
   └ 不在 task 描述里（task 外，分支上的临时改动）:
       - utils/format.ts        ← 注意：仍然要补测试
       - styles/theme.scss      ← 注意：仍然要补测试
       - components/Modal.tsx   ← 注意：仍然要补测试
```

> ⚠️ "task 外"那一栏出现文件**不是异常**，分支上常有合理的临时改动；但它们**必须和 task 内的文件一视同仁**，全部进入 3.4 调 skill 补测试。如果发现"task 外"文件特别多，说明开发流程把太多事情塞进了同一分支，可向用户提示但**不能**作为跳过测试的理由。

#### 3.4 强制补齐（有欠债时）

**有欠债 → 必须立即调 tc-qa-engineer 补齐**。用 Skill 工具发起真实调用，**不允许**只复述：

```
Skill(skill="tc-qa-engineer", args=<把下列字段拼成一段说明传给 skill>)
```

`args` 必须传齐：

- **触发原因**：`历史欠债补齐`
- **欠债文件列表**：3.3 输出的"⚠ 缺测试"完整清单（绝对路径）
- **累积变更**：本 feature 所有 [x] task 的代码改动范围
- **specs 路径**：`requirements.md` / `design.md` / `tasks.md` 的绝对路径
- **是否 feature 收尾**：若所有 task 都 `[x]` 则 `true`，否则 `false`

skill 返回后输出：

```text
✓ 欠债清理完成 — 新增/扩展测试 {N} 个
   落盘文件:
     - {test 文件 1}
     - {test 文件 2}
   → 进入步骤 4 执行计划
```

skill 失败 / 复测仍红 → **暂停并报告**，不继续进步骤 4。

#### 3.5 无欠债

```text
✓ 测试欠债审计通过 — 所有已落地代码均有测试覆盖
   → 进入步骤 4 执行计划
```

### 4. 输出执行计划（支线 B —— 默认主线，不可跳过）

**只要还存在 `[ ]` / `[CHANGED]` 待执行 task，就必须输出执行计划并进 N3 开发**。这是 N2 的默认出口，**不能用"步骤 3 已经在补欠债"为由跳过**。

#### 4.1 分析依赖

分析 `tasks.md` 的依赖关系，自行决定串行或并行：

| 串行 | 并行 |
| ---- | ---- |
| 有显式依赖 | 无依赖 |
| 会修改同一文件/模块 | 分属不同代码项目 |
| 涉及共享状态定义（schema、API） | 天然隔离 |

并行时用 Agent 工具派发子 agent。所有任务都有依赖时退化为全串行。

#### 4.2 输出计划

```text
📋 执行计划:
   串行 1: T-001 → T-002
   并行 2: T-003 + T-004
   串行 3: T-005 ← 依赖 T-003, T-004
```

#### 4.3 边界：feature 已全部完成（罕见，仅在重跑 /tc-ai 时出现）

仅当**所有 task 都已 `[x]` / `[DROPPED]`、没有任何 `[ ]` / `[CHANGED]`**，且**步骤 3 测试欠债审计通过/已补齐**时，才输出跳过消息进入下一个 feature：

```text
⏭  Feature {N} 全部完成（{total}/{total}），测试覆盖已审计 → 跳过
```

否则一律出执行计划进 N3。

## 退出消息（强制输出）

```text
📊 断点恢复: 已完成 {done}/{total} | 本轮待执行 {pending} 个
   ⏭  跳过 [x]: {N} 个
   ⏭  跳过 [DROPPED]: {N} 个
   🔄 [CHANGED]: {N} 个（按 v{N} 描述执行）

🔍 测试欠债审计:
   ✓ 已覆盖: {M} 个文件
   {如有欠债已补:}
   🛠  补齐欠债: {K} 个文件 → 新增测试 {N} 个

📋 执行计划:
   串行 1: T-001 → T-002
   并行 2: T-003 + T-004
   串行 3: T-005 ← 依赖 T-003, T-004

✅ N2 完成 → 进入 N3 执行首个 task
```

## 反模式（强禁止）

- ❌ **不准把欠债审计与执行计划做成二选一** — 它们是两条独立支线，有 `[ ]` task 就必须出执行计划进 N3，欠债再多也不能"用补欠债代替开发新代码"
- ❌ **不准把执行计划删掉** — 即使整个 feature 全是 `[x]` 也只是边界情况，正常 feature 永远要走步骤 4
- ❌ **不准在审计前就跳过 feature** — 即使所有 task 都 [x]，也必须先过测试欠债审计
- ❌ **不准把欠债拖到 feature 收尾** — 在本节点立刻补，避免边写新 task 边累积分散的欠债
- ❌ **不准只复述"调用 tc-qa-engineer..."不发起 Skill 工具调用** — 与 N6 的口径一致
- ❌ **不准把"找不到测试文件"当作"不需要测试"** — 找不到就是欠债
- ❌ **不准让用户手动判断哪些文件需要补测试** — 由 skill 的影响面分析裁定，N2 只负责把欠债清单交出去
- ❌ **不准用 `design.md` / `requirements.md` / `tasks.md` 当过滤器** — 它们仅作分类参考。分支上 git 看到的所有代码改动都要审计，"不在 task 描述里" ≠ "不需要测试"
- ❌ **不准只取 `git diff main...HEAD` 一路** — 必须叠加 working tree、staged、untracked 四路，不然会漏掉本地未 commit 的手改
