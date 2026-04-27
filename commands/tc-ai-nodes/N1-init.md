# N1: 初始化

## 进入消息

```text
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🚀 /tc-ai 启动 — N1 初始化
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

## 步骤

1. 从 `$ARGUMENTS` 提取 **specs 文件夹路径** 和 **代码项目路径**（可多个）
2. 扫描 specs 下所有编号目录（`1.xxx/`、`2.xxx/`），按编号排列
3. 每个 feature 目录须含 requirements.md、design.md、tasks.md
4. 加载：代码项目的 `.claude/CLAUDE.md` + `.claude/rules/`
5. 加载 `{SPECS_DIR}/LESSONS.md`（架构决策和踩坑记录，开发时必须参考）
6. 验证各代码项目路径存在

## 退出消息（强制输出）

```text
📂 Specs: {SPECS_DIR}
💻 代码项目:
   - {project_name}: {path}
   - ...
📋 Features: {N} 个
   - 1.{name} ({done}/{total} task)
   - 2.{name} ({done}/{total} task)
   ...
📚 上下文已加载: CLAUDE.md ✓ rules/ ({N}个) ✓ LESSONS.md {✓/—}
🔢 总进度: {done}/{total} task | 待开发 Feature: {N} 个

✅ N1 完成 → 进入 N2
```

任何缺失（specs 不存在、feature 缺三件套、代码路径不存在）→ **暂停并报错**，不要静默跳过。
