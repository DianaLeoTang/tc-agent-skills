---
description: 自动开发 — 按节点流程执行 specs 任务
---

# /tc-ai — 自动开发

> 💡 **本命令的输入是已生成的 specs 三件套**（由 [`/tc-prd`](tc-prd.md) 产出）。如果还没 specs，请先按完整工作流走：[`/tc-init`](tc-init.md)（一次性）→ [`/tc-discuss`](tc-discuss.md)（聊需求）→ [`/tc-prd`](tc-prd.md) → 本命令。

`$ARGUMENTS` — specs 文件夹路径 + 代码项目路径。

```bash
/tc-ai specs在~/projects/my-app-specs，代码在~/code/my-app
/tc-ai ~/projects/specs 前端~/code/fe 后端~/code/api
```

## 流程图

按此流程执行，到达每个节点时按优先级读取节点文件获取详细规则：
1. 代码项目 `.claude/commands/tc-ai-nodes/`（项目级覆盖，可选）
2. 全局 `.claude/commands/tc-ai-nodes/`（默认）

```text
START
  │
  ▼
[N1: 初始化] ── 解析输入、扫描 features、加载项目约束规则
  │
  ▼
┌─► [N2: 进入 Feature] ── 读取 specs、分析依赖、输出执行计划
│     │
│     ▼
│   ┌─► [N3: 执行 Task] ── 检查 skill → 开发
│   │     │
│   │     ▼
│   │   [N4: Review] ── AI 自审 → Codex Review
│   │     │
│   │     ▼
│   │   [N5: 标记完成] ── tasks.md 标 [x]、写 LESSONS.md
│   │     │
│   │     ▼
│   │   [N6: QA 评估] ── 评分决定是否触发 tc-qa-engineer
│   │     │
│   │     ▼
│   │   [N7: 上下文管理] ── /clear → 重新加载 specs
│   │     │
│   │     ▼
│   │   还有未完成 task? ──YES──┘
│   │     │
│   │    NO
│   │     │
│   │     ▼
│   └── Feature 完成 → /clear
│         │
│         ▼
│       还有下一个 Feature? ──YES──┘
│         │
│        NO
│         │
│         ▼
      [N8: 完成] ── 调用 tc-doc-syncer → 输出总结
        │
        ▼
       END
```

## 全局规则

**暂停：** 业务逻辑歧义、不确定的安全问题、破坏性变更、环境阻塞。
**不暂停：** 纯技术选型 — 选最优解直接执行。

**执行策略：** AI 自主决策串行或并行（无依赖 + 不同项目 → 并行，否则串行）。

## 全局可观测性（强制）

**每个节点必须回吐消息**，让用户随时知道流程在哪个阶段、做了什么决策、发生了什么。

- **进入节点** → 输出节点标识（如 `🔨 Task T-xxx`、`🔍 N4 Review`）
- **关键决策点** → 输出决策与理由（skill 匹配/未匹配、QA 触发/跳过、Codex 通过/跳过）
- **退出节点** → 输出结果与下一步（`✅ Nx 完成 → 进入 Nx+1`）
- **执行 `/clear` / `/compact` 之前** → 必须先输出说明，避免上下文消失得"莫名其妙"
- **依赖的 skill 不存在** → 明确输出"⏭ 跳过（{skill名} 未注册）"，**不要静默跳过**
- **暂停** → 明确输出暂停原因和需要用户做什么

输出风格统一：
- 节点之间用 `━━━━━` 分隔
- 状态用图标：✓/✅ 成功，⚠ 警告，❌ 失败，⏭ 跳过，🔄 重试，🔁 循环
- 进度信息形如 `Feature {F}/{总F} | 任务 {N}/{总数}`
