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

**目的**：把分支上已落地代码的测试欠债补回来。**与步骤 4 的执行计划独立**，欠债该补就补，新 task 该开发还是要进 N3。

> ⚠️ 重要原则：本步骤**全部走工具调用**（Bash + Agent），**禁止**只在文本里 narrate "我将运行 / 我将派发"。每个子步骤都要有真实的工具输出做证据，否则用户根本不知道 AI 跑到哪一步、卡在哪里。

#### 3.1 收集分支上的全部代码改动（用 Bash 跑，给具体命令）

**主体原则**：以 git 为唯一权威，不依赖 `design.md` / `requirements.md` 来圈定范围。

用 Bash 工具一次性跑下面这条命令（**直接执行，不要 narrate**）：

```bash
# 自动发现 base 分支（main / master / develop 之一），找不到时显式报错让用户给
BASE=$(git rev-parse --verify --quiet main || git rev-parse --verify --quiet master || git rev-parse --verify --quiet develop)
if [ -z "$BASE" ]; then
  echo "ERROR: 找不到 main/master/develop 任一基线分支，请告知 base 分支名"
  exit 1
fi
MERGE_BASE=$(git merge-base HEAD $BASE 2>/dev/null || echo "")

# 四路合并
{
  [ -n "$MERGE_BASE" ] && git diff --name-only ${MERGE_BASE}...HEAD
  git diff --name-only HEAD
  git diff --name-only --cached
  git ls-files --others --exclude-standard
} | sort -u | grep -E '\.(ts|tsx|js|jsx|vue|svelte|py|go|rs|sol|java|kt|swift|css|scss|html)$' \
  | grep -vE '\.(test|spec)\.|__tests__/|^tests?/|/dist/|/build/|/\.next/|/coverage/|/node_modules/'
```

**期望输出**：每行一个代码文件路径。若一行没有 → 没改动 → 直接进 3.6（无欠债）。若 Bash 报错 → 立即停止并告知用户问题，**不要静默跳过**。

#### 3.2 扫描已有测试覆盖（用 Bash 跑）

对 3.1 输出的每个文件 `f`，按命名规则查对应测试文件。给一段可执行 Bash：

```bash
# 输入：$FILES（3.1 的输出，逐行）
# 输出：欠债清单（没找到对应测试的文件）
while IFS= read -r f; do
  base=$(basename "$f")
  stem="${base%.*}"
  ext="${base##*.}"
  dir=$(dirname "$f")
  
  # 按语言列举可能的测试文件位置
  candidates=()
  case "$ext" in
    ts|tsx|js|jsx|vue|svelte)
      candidates+=("$dir/$stem.test.$ext" "$dir/$stem.spec.$ext")
      candidates+=("$dir/__tests__/$stem.test.$ext" "$dir/__tests__/$stem.spec.$ext")
      candidates+=("tests/${f%.*}.test.$ext")
      ;;
    py)
      candidates+=("$dir/test_$stem.py" "tests/test_$stem.py")
      ;;
    go) candidates+=("$dir/${stem}_test.go") ;;
    rs) candidates+=("tests/$stem.rs") ;;
    sol) candidates+=("test/$stem.t.sol" "test/${stem^}.t.sol") ;;
  esac
  
  found=false
  for c in "${candidates[@]}"; do
    [ -f "$c" ] && { found=true; break; }
  done
  $found || echo "$f"
done <<< "$FILES"
```

**期望输出**：每行一个**欠债文件**路径。若空行 → 进 3.6 无欠债。

#### 3.3 输出审计报告（强制 console 输出）

```text
🔍 测试欠债审计 — Feature {名}
   git 收集: {N1} 个代码文件（去重去测试去构建产物）
   ✓ 已有测试: {M}
   ⚠ 欠债: {K}
     1) src/utils/format.ts
     2) src/components/Button.tsx
     3) src/api/user.ts
     ...
   
   分类（仅供参考，不影响派发）:
   - design.md / requirements.md 范围内: {n_in} 个
   - 分支临时改动（task 描述外）: {n_out} 个
```

#### 3.4 派发计划（先输出再执行）

**先**用一段文本明确"我将派发哪些子 agent、并行还是串行"，让用户在派发前看到决策：

```text
🚚 派发计划:
   并行批 1（互不依赖的 utility/独立模块）:
     · src/utils/format.ts → Agent
     · src/utils/parse.ts → Agent
   串行批 2（公共组件 → 调用方）:
     · src/components/Button.tsx → Agent (先)
     · src/pages/LoginPage.tsx → Agent (后，依赖 Button)
   合计: {K} 个子 agent
```

#### 3.5 派发子 agent（用 Agent 工具，每文件一个）

按 3.4 的计划，**实际**用 Agent 工具发起调用。每个文件一个独立子 agent，prompt 用统一短模板（见 [skills/tc-qa-engineer/SKILL.md](../../skills/tc-qa-engineer/SKILL.md) 的「单文件子 agent 调用契约」段，N2 不再展开）：

```
Agent(
  description="为 <basename> 写测试",
  subagent_type="general-purpose",
  prompt="参考 skills/tc-qa-engineer/SKILL.md「单文件子 agent 调用契约」执行。
          source_file: <绝对路径>
          specs: <requirements.md / design.md / tasks.md 绝对路径>
          accumulated_changes: <本分支累积变更文件列表>
          feature_finalizing: <true / false>
          返回严格 JSON 契约，test_files_written 不能为空，ran/passed 必须 true。"
)
```

并行批 → 一条消息里同时发 N 个 Agent 调用。串行批 → 等前批返回再发后批。

#### 3.6 退出校验（强制，不靠子 agent 自我汇报）

子 agent 全部返回后，**重新跑一次 3.2 的 Bash**对清单查证：

- 每个原欠债文件都找到对应测试文件 → ✓ 通过，进步骤 4
- 仍缺测试的文件 → **重派最多 1 轮**
- 1 轮重试仍缺 → **暂停并报告**，列出所有未补的文件，交用户裁决

```text
🔍 退出校验:
   ✓ 已补: {M}/{K}
   ⚠ 仍缺: {K - M}（重派中...）
   {1 轮重试结果}
   或
   ❌ 重试后仍缺 {K1} 个 → 暂停
```

#### 3.7 无欠债（短路出口）

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
- ❌ **严禁单次 Skill 调用承担多文件** — N 个欠债文件就要派发 N 个独立子 agent（Agent 工具），每个子 agent 只负责一个文件。一次大调用 = AI 必偷懒
- ❌ **不准跳过退出校验（步骤 3.5）** — 派发后必须重新扫一遍清单。子 agent 的自我汇报不可信，要靠 N2 机械核对落盘文件
