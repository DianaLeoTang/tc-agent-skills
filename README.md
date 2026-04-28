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
| `/tc-init` | 项目 `.claude` 初始化 — 生成 `CLAUDE.md` 与 `rules/`（增量友好，不覆盖现有内容） |
| `/tc-discuss` | 对话式需求澄清 — 一句话 + 多轮对话 → 结构化 `docs/feature-{name}.md` |
| `/tc-prd` | 需求文档 → specs 三件套生成（支持新建与变更） |
| `/tc-ai` | 自动开发 — 按节点流程执行 specs 任务 |
| `/tc-test` | 跑测试 — 自动识别项目测试栈，跑现有测试套件，输出结构化报告 |
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
/tc-test             # （可选，随时调）跑现有测试，结构化报告
  ↓
tc-doc-syncer        # 同步文档（由 tc-ai 自动触发或手动调用）
```

**`/tc-test` 与 `tc-qa-engineer` 的区别**：
- `tc-qa-engineer`（skill）— **设计** 新测试用例 + 跑测；由 `/tc-ai` 的 N6 节点自动触发
- `/tc-test`（命令）— **只跑** 现有测试 + 结构化报告；由你随时手动调用

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
│   ├── tc-test.md
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

## 本地开发（贡献者 / 二次开发）

如果你要本地修改命令并立刻在自己 Claude Code 里生效（不通过 marketplace），推荐用 **symlink 方案**：让 `~/.claude/commands/tc-*` 与 `~/.claude/skills/tc-*` 直接指向本仓库，改一处两边都更新，永远不漂移。

### 一键设置

```bash
PLUGIN="$(pwd)"   # 在本仓库根目录执行
LOCAL="$HOME/.claude"

# 替换 commands 文件
for f in tc-init.md tc-prd.md tc-ai.md tc-discuss.md; do
  rm -f "$LOCAL/commands/$f"
  ln -s "$PLUGIN/commands/$f" "$LOCAL/commands/$f"
done

# 替换 commands/tc-ai-nodes 整个目录
rm -rf "$LOCAL/commands/tc-ai-nodes"
ln -s "$PLUGIN/commands/tc-ai-nodes" "$LOCAL/commands/tc-ai-nodes"

# 替换 skills 目录
for s in tc-frontend-engineer tc-qa-engineer tc-doc-syncer; do
  rm -rf "$LOCAL/skills/$s"
  ln -s "$PLUGIN/skills/$s" "$LOCAL/skills/$s"
done

ls -la "$LOCAL/commands/"tc-* "$LOCAL/skills/"tc-*  # 验证 symlink
```

### 工作流

```
本仓库 commands/tc-init.md   ← 唯一来源
       ↑↓ symlink
~/.claude/commands/tc-init.md ← 本地立刻生效
```

- 在任一处编辑文件 = 改的是同一份内容
- 提交 git = 直接在本仓库 `git commit`，无需「同步」步骤
- 如果未来本仓库移动到别的路径 → symlink 会断；重跑上面脚本即可

### 解除 symlink（恢复独立副本）

如果想撤销 symlink 改回独立副本：

```bash
LOCAL="$HOME/.claude"
PLUGIN="/path/to/tc-agent-skills"  # 改成本仓库实际路径

for f in tc-init.md tc-prd.md tc-ai.md tc-discuss.md; do
  rm "$LOCAL/commands/$f"
  cp "$PLUGIN/commands/$f" "$LOCAL/commands/$f"
done
# tc-ai-nodes 与 skills 同理用 cp -r
```

## License

MIT
