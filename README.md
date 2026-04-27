# tc-agent-skills

一个服务于小程序和客户端前端工程的 Claude Code 工作流插件。

涵盖从需求澄清 → PRD 三件套 → 自动开发 → Code Review → QA → 文档同步的全流程命令与 skills。

## 安装

确保你已经装好 [Claude Code](https://docs.claude.com/en/docs/claude-code/overview)，然后在 Claude Code 里执行：

```
/plugin marketplace add DianaLeoTang/tc-agent-skills
/plugin install tc-agent-skills@DianaLeoTang
```

装好后命令和 skills 会以 `tc-agent-skills:` 命名空间出现，例如 `/tc-agent-skills:tc-init`。

## 命令一览

| 命令 | 用途 |
| --- | --- |
| `/tc-init` | 项目 `.claude` 初始化 — 生成 `CLAUDE.md` 与 `rules/` |
| `/tc-discuss` | 对话式需求澄清 — 一句话 + 多轮对话 → 结构化 `docs/feature-{name}.md` |
| `/tc-prd` | 需求文档 → specs 三件套生成（支持新建与变更） |
| `/tc-ai` | 自动开发 — 按节点流程执行 specs 任务 |
| `/tc-ai-nodes:N1-init` … `N8-finish` | `tc-ai` 内部使用的 8 个节点子命令 |

## Skills 一览

| Skill | 用途 |
| --- | --- |
| `tc-frontend-engineer` | 前端工程师 — 自动适配项目技术栈（React/Vue/Svelte/Next.js 等），支持 Figma/Stitch 设计稿还原 |
| `tc-qa-engineer` | QA 工程师 — 自动识别测试栈（Jest/Vitest/Playwright/Pytest/Go test/Foundry 等）、设计补充用例、跑测、形成 bug 闭环 |
| `tc-doc-syncer` | 文档同步 — 开发完成后自动更新 README、`.claude/` 配置、specs CHANGELOG，保持文档与代码一致 |

## 推荐工作流

```
/tc-init             # 一次性：初始化项目 .claude
  ↓
/tc-discuss          # 对话式澄清需求 → docs/feature-{name}.md
  ↓
/tc-prd              # 生成 specs 三件套
  ↓
/tc-ai               # 自动按节点完成开发 (内部依次调用 N1~N8)
  ↓
tc-doc-syncer        # 同步文档（由 tc-ai 自动触发或手动调用）
```

## 目录结构

```
tc-agent-skills/
├── .claude-plugin/
│   └── plugin.json
├── commands/
│   ├── tc-init.md
│   ├── tc-discuss.md
│   ├── tc-prd.md
│   ├── tc-ai.md
│   └── tc-ai-nodes/
│       ├── N1-init.md
│       ├── N2-enter-feature.md
│       ├── N3-execute-task.md
│       ├── N4-review.md
│       ├── N5-mark-done.md
│       ├── N6-qa-eval.md
│       ├── N7-context.md
│       └── N8-finish.md
└── skills/
    ├── tc-doc-syncer/SKILL.md
    ├── tc-frontend-engineer/SKILL.md
    └── tc-qa-engineer/SKILL.md
```

## 升级

```
/plugin marketplace update tc-agent-skills
```

## License

MIT
