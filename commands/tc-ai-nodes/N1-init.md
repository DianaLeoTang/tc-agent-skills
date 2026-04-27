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
4. **强制：逐个代码项目执行以下操作**
   - 读取 `.claude/CLAUDE.md`，提取技术栈、目录结构、命令
   - **扫描 `.claude/rules/` 目录下所有 `.md` 文件，逐一读取并记录**
   - 加载 `{SPECS_DIR}/LESSONS.md`（架构决策和踩坑记录，开发时必须参考）
5. 验证各代码项目路径存在

**⚠️ 关键约束**：步骤 4 中读取的所有 rules 文件，**必须在后续 N3（开发）和 N4（Review）中严格遵守**，不得违反。任何编码决策（组件放置、样式写法、API 调用方式、状态管理方案等）都需对照 rules 验证。

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
   规则清单:
   - {rule1.md} — {description}
   - {rule2.md} — {description}
   ...
🔢 总进度: {done}/{total} task | 待开发 Feature: {N} 个

✅ N1 完成 → 进入 N2
```

任何缺失（specs 不存在、feature 缺三件套、代码路径不存在）→ **暂停并报错**，不要静默跳过。
