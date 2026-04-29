# N3: 执行 Task — 写代码

N3 只负责写代码。测试由 N6 统一调 `tc-qa-engineer` 来写。

## 进入

```text
🔨 Task {T-编号}: {描述}  Feature {F}/{总F} | 任务 {N}/{总数}
```

## Skill 匹配

按工种调对应 `tc-*` skill：前端 → `tc-frontend-engineer`；数据库 → `tc-database-engineer`；合约 → `tc-contract-engineer`。没匹配 → AI 直接执行。

`tc-qa-engineer` 不在此列 —— 测试由 N6 统一处理。

```text
🎯 Skill: {skill名}
   或
🎯 Skill: 无 → AI 直接执行（理由: ...）
```

## 开发

- 参考 `design.md` 与 N1 加载的项目 rules
- 编码决策遵循项目自身 rules（落位 / 复用 / 样式 / API / 状态 / 修改边界）
- 技术选型自行选最优解，不暂停
- 业务逻辑/产品方向歧义 → 暂停问用户

## 退出

```text
✓ Task {T-编号} 代码完成
   📝 变更: {N} 个文件
      - {file1}
      - {file2}
   ⏱  ~{时间}（vs 预估 {预估}）
   {如有暂停: ⚠ 暂停 {N} 次: {简述}}

→ N4 Review
```
