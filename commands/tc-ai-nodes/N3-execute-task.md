# N3: 执行 Task

## 进入消息（强制输出）

```text
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🔨 Task {T-编号}: {任务描述} ~{预估时间}
   Feature {F}/{总F} | 任务 {N}/{总数}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

## Skill 匹配

根据任务涉及的工种，查看可用的 `tc-*` skills：

- 前端 → `tc-frontend-engineer`
- 数据库 → `tc-database-engineer`
- 合约 → `tc-contract-engineer`
- QA/测试 → `tc-qa-engineer`
- 没有匹配 → AI 直接执行

**输出 skill 决策**（强制）：

```text
🎯 Skill 匹配: {skill名}
   或
🎯 Skill 匹配: 无 → AI 直接执行（理由: {为什么没匹配}）
```

有匹配的 skill → 调用该 skill 执行。

## 开发

- 参考 design.md 技术设计和 `.claude/rules/` 规范
- 技术选型自行选最优解，不暂停
- 业务逻辑/产品方向问题 → 暂停与用户沟通

## 退出消息（强制输出）

```text
✓ Task {T-编号} 开发完成
   📝 变更文件: {N} 个
      - {file1}
      - {file2}
      ...
   ⏱  实际耗时: ~{时间}（vs 预估 {预估}）
   {如有暂停: ⚠ 暂停 {N} 次: {简述}}

→ 进入 N4 Review
```
