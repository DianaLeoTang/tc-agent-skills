# N2: 进入 Feature

```text
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📂 Feature {N}/{总数} — {名}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

## 1. 读 specs

读该 feature 的 `requirements.md` / `design.md` / `tasks.md`。

## 2. 断点恢复

逐项识别 task 状态：`[x]` 已完成（跳）/ `[DROPPED]` 跳 / `[CHANGED]` 按新描述执行 / `[ ]` 未执行。

## 3. 历史欠债审计

N6 是常规写测试节点，但分支上可能存在**未经 N6 流程的代码**——比如旧版 task 完成时还没建立 N6、用户手改了 bug/refactor 没走 N3→N6。本步用一段四路合并扫描把这类欠债找出来：

```bash
BASE=$(git rev-parse --verify --quiet main \
    || git rev-parse --verify --quiet master \
    || git rev-parse --verify --quiet develop \
    || git rev-parse --verify --quiet origin/main \
    || git rev-parse --verify --quiet origin/master)
[ -z "${BASE:-}" ] && { echo "ERROR: 找不到 base 分支"; exit 1; }
MERGE_BASE=$(git merge-base HEAD "$BASE" 2>/dev/null || echo "")

{
  [ -n "$MERGE_BASE" ] && git diff --name-only "${MERGE_BASE}...HEAD"
  git diff --name-only HEAD
  git diff --name-only --cached
  git ls-files --others --exclude-standard
} | sort -u \
  | grep -E '\.(ts|tsx|js|jsx|vue|svelte|py|go|rs|sol|java|kt|swift|css|scss|html)$' \
  | grep -vE '\.(test|spec)\.|__tests__/|^tests?/|/dist/|/build/|/\.next/|/coverage/|/node_modules/' \
  | while IFS= read -r f; do
      base=$(basename "$f"); stem="${base%.*}"; ext="${base##*.}"; dir=$(dirname "$f")
      found=0
      case "$ext" in
        ts|tsx|js|jsx|vue|svelte)
          for c in "$dir/$stem.test.$ext" "$dir/$stem.spec.$ext" \
                   "$dir/__tests__/$stem.test.$ext" "$dir/__tests__/$stem.spec.$ext" \
                   "tests/${f%.*}.test.$ext"; do
            [ -f "$c" ] && { found=1; break; }
          done ;;
        py)  for c in "$dir/test_$stem.py" "tests/test_$stem.py"; do
               [ -f "$c" ] && { found=1; break; }; done ;;
        go)  [ -f "$dir/${stem}_test.go" ] && found=1 ;;
        rs)  [ -f "tests/$stem.rs" ] && found=1 ;;
        sol) for c in "test/$stem.t.sol" "test/${stem^}.t.sol"; do
               [ -f "$c" ] && { found=1; break; }; done ;;
      esac
      [ "$found" = 0 ] && echo "$f"
    done
```

| 输出 | 处理 |
|---|---|
| 空 | ✓ 进步骤 4 |
| 非空 | 列给用户 → 用户选：A) 现在补（每个文件派一个独立 Agent 并行写测试） B) 跳过先开发新 task C) 取消 |

补齐选 A 时，**派发计划与 Agent 调用同回合发出**（不准只 narrate）。每个欠债文件派一个独立子 agent，prompt 模板见 [skills/tc-qa-engineer/SKILL.md](../../skills/tc-qa-engineer/SKILL.md) 的「单文件子 agent 调用契约」段。Agent 返回后**重跑上面整段扫描** 核对，仍缺则重派 1 轮，再缺暂停。

## 4. 执行计划

按 `tasks.md` 的依赖关系决定串行 / 并行（无依赖 + 不同项目 → 并行用 Agent 派发；有依赖或共享文件 → 串行）。

```text
📋 执行计划:
   串行 1: T-001 → T-002
   并行 2: T-003 + T-004
```

## 退出

```text
📊 已完成 {done}/{total} | 待执行 {pending}
🔍 欠债: {0 / 已补 K / 用户跳过}
📋 见上方执行计划

→ N3
```

如果 feature 整个全 `[x]` 且欠债为 0 → 直接跳过进下一个 feature。
