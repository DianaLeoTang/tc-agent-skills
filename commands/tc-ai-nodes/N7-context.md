# N7: 上下文管理

## 进入消息

```text
🧹 N7 上下文管理 — Task {T-编号} 完成
```

## task 完成后

**执行前输出：**

```text
🧹 即将执行 /clear，下一轮将重新加载:
   - {SPECS_DIR}/{N}.{feature}/{requirements,design,tasks}.md
   - {SPECS_DIR}/LESSONS.md
   - {代码项目}/.claude/CLAUDE.md + rules/
```

执行 `/clear`，然后重新读取上述文件，继续下一个 task。

**重新加载完成后输出：**

```text
🔁 上下文已重置 → 继续 Feature {F} 下一个 task
```

## task 执行中

上下文达 80% → **先输出**：

```text
⚠ 上下文已达 80%，执行 /compact 压缩后继续当前 task
```

执行 `/compact` 后继续。

## feature 完成后

```text
🎯 Feature {F}/{总F} — {feature名} 全部 task 完成
🧹 执行 /clear → 进入下一个 Feature
```

执行 `/clear`，进入下一个 feature。

## 关键约束

- **每次 /clear / /compact 之前必须先输出消息**，让用户知道发生了什么
- `/clear` 之前必须确认 N5 标记已写入 tasks.md
- 全程自动继续，无需等待用户指令
