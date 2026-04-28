---
description: 对话式需求澄清 — 一句话 + 多轮对话 → 结构化 docs/feature-{name}.md（作为 /tc-prd 的输入）
---

# /tc-discuss — 需求澄清

把"我要做个 XX"这种粗糙想法，通过多轮对话整理成结构化需求文档（docs/feature-{name}.md），用作 `/tc-prd` 的输入。

## 何时用

- 需求是聊出来的，不是直接的 PRD
- 一个想法可能要拆成多个子需求（一开始你也不确定）
- 不知道 docs/ 文件该写成什么样子
- 需要 AI 帮你识别「这个是不是已有 feature 的变更」

## 何时不用

- 已有现成 PRD 文档 → 直接放进 docs/，跑 `/tc-prd`
- 已存在 specs 要变更 → 直接跑 `/tc-prd --change`
- 不需要拆分、需求很简单一句话能说清 → 也可手写 docs，跑 `/tc-prd`

## 输入参数

`$ARGUMENTS` 是用户的初始需求描述（可空）：

```bash
/tc-discuss 我想加个用户能改密码的功能
/tc-discuss 我们要做用户续费，但还没决定支付方式
/tc-discuss 想让医生头像可以上传，最好支持裁剪
/tc-discuss   # 空也行，命令会反问"告诉我你想做什么"
```

## 流程

### Step 0: 接收初始描述

- 如果 `$ARGUMENTS` 为空：输出「告诉我你想做什么？一句话就行」并暂停等输入
- 否则：把 `$ARGUMENTS` 作为起点，进入 Step 1

### Step 1: 探测项目上下文

读取并解析：

- `.claude/CLAUDE.md`（项目技术栈与约定）
- `.claude/rules/*.md`（如有）
- 项目根的 `ROADMAP.md`（如有，识别已规划但未做的 feature）
- 已有 specs 目录：`{SPECS_DIR}/[0-9]*.{feature-name}/`
- `docs/` 下已有的需求文档列表

记录到工作记忆：
- `EXISTING_FEATURES`: 已规划/进行中/已完成的 feature 列表
- `EXISTING_DOCS`: docs/ 下文件列表
- `NEXT_FEATURE_NUM`: 已有最大编号 + 1（用于命名新 docs）

### Step 2: 重叠检测（关键防御）

把初始描述与 `EXISTING_FEATURES` 做语义匹配：

**如果命中已存在 feature：**

```
🔍 检测到你的描述可能与已有功能重叠：

匹配 #1：[N.feature-name] 描述 (status: specs-ready / done / pending)
  - 匹配点：{解释为什么觉得相关}

请选择：
  A) 实施已存在的 feature → 跳出 tc-discuss，运行 /tc-ai 即可
  B) 我描述的是新需求，与上述无关 → 继续 Step 3
  C) 改/扩展现有 feature → 跳出 tc-discuss，运行 /tc-prd --change {N}.{feature-name}
```

⏸ 等用户回复后再继续。

**如果检测到 ROADMAP 已规划：**

```
📋 这个想法似乎对应 ROADMAP 里的：
  - {ID}：{条目描述}（归属 feature {N}，状态：pending）

要把它从 ROADMAP "升级"成正式 feature 吗？
  A) 是 → 继续 Step 3，本次澄清的 docs 会引用 ROADMAP 编号
  B) 否，这是 ROADMAP 之外的新东西 → 继续 Step 3
```

### Step 3: 多轮对话澄清（核心）

按主题分轮，**每轮只问 2-3 个问题**，避免一次抛 10 个把用户压垮。

#### 轮 1：业务 / 场景

```
为了把需求写清楚，我先问几个业务问题：

1. **谁用这个功能？**（用户角色：付费用户 / 管理员 / 外部访客 / ...）
2. **什么场景下用到？**（在哪个页面、什么操作链路上）
3. **多频繁？**（高频日常 / 低频应急 / 一次性）
```

⏸ 等用户回答。如果用户某条说「不知道」「先这样」，记录为 `[未明确]` 不强求。

#### 轮 2：范围 / 边界

基于轮 1 答案展开：

```
继续问 3 个范围问题：

1. **必须有哪些子功能？**（核心 must-have，列 2-5 条）
2. **明确不做哪些？**（避免范围蔓延，列出 nice-to-have 但本期不做）
3. **与项目现有哪些功能交互？**（如登录、用户表、医生表、评价表等）
```

⏸ 等回答。

#### 轮 3：决策（已知 vs 待定）

```
最后两类问题：

A. **你已经决定的事**（例如"用阿里云短信服务"、"密码 6 位以上"）—— 直接列出来
B. **还没决定、需要后续确认的事**（例如"卡密兑换还是支付？"）—— 列出来留给 /tc-prd Step 5.5

要回答多少给多少，不知道就跳过。
```

⏸ 等回答。

#### 轮 4（可选）：拆分识别 — 关键判断

基于前三轮答案，分析：

**判断标准：**
- 涉及 ≥ 3 个不同页面 / router → 可能要拆
- 范围涵盖前端 + 后端 + 第三方集成 → 可能要拆
- 「必做范围」列出了 ≥ 5 条本质不同的能力 → 可能要拆
- 用户提到「先做 X，再做 Y」「Y 暂缓」→ 暗示天然拆分

**如果判断需要拆：**

```
我观察你描述的需求其实可能拆成多个独立 feature：

建议拆分：
  - feature-A：{名称}（包含子功能 1, 2）
  - feature-B：{名称}（包含子功能 3, 4）
  - feature-C：{名称}（依赖 A 完成后才能做）

理由：{为什么拆 — 例如"前端表单 + 后端服务集成属于不同模块"}

你的看法：
  1) 同意按这样拆 → 生成多个 docs
  2) 全合并成一个 feature → 生成一个 docs
  3) 我重新拆，告诉我你的方案
```

⏸ 等回答。如果用户给新的拆分方案，按用户的来。

**如果是单 feature（无需拆）：**

直接进入 Step 4。

### Step 4: 生成 docs/

按拆分结果生成 1 个或 N 个文件，命名规则：

```
docs/feature-{N}-{kebab-case-name}.md
```

`N` 取 `NEXT_FEATURE_NUM`（多个 feature 顺延 N、N+1、N+2）。

**单文件模板**（每个 docs 都用这个）：

```markdown
# Feature {N}: {名称}（聊出来的需求草稿）

> ⚠️ 由 `/tc-discuss` 生成的需求草稿，作为 `/tc-prd` 的输入。
> 生成日期：{YYYY-MM-DD}
> 拆分情况：{单 feature / N 个之 1}

## 起源（一句话）

{用户最初的描述，原话引用}

> 用户原话："{$ARGUMENTS 或对话开头那句}"

## 业务背景

- **谁用**：{角色，如"付费用户" / "管理员" / "未明确"}
- **场景**：{在哪个页面/操作链路触发}
- **频率**：{高频 / 低频 / 一次性 / 未明确}
- **要解决的问题**：{基于轮 1 答案归纳}

## 必做范围

1. {基于轮 2 必做项，每条一行}
2. ...

## 不做范围（避免蔓延）

- ❌ {基于轮 2 明确不做的}
- ❌ ...

## 已知决策

- {基于轮 3 A 部分，每条一行；如果用户没说，写"无"}

## 待定决策（留给 /tc-prd Step 5.5 确认）

- [Q-001] {基于轮 3 B 部分的开放问题}
- [Q-002] ...

## 与现有功能交互

- {基于轮 2 第 3 题答案 + 项目上下文，列具体 router / 模块 / 表名}

## 拆分说明

{如果是 N 拆 1 之一}：
- 原始想法："{用户初始描述}"
- 拆成 {N} 个 feature：
  - feature-{N1}: {名称} ← **当前文档**
  - feature-{N2}: {名称}
  - feature-{N3}: {名称}
- 当前文档对应：{N1} 部分

{如果是单 feature}：
- 单 feature，无需拆分

## 与 ROADMAP 关系

{如果在 Step 2 关联到 ROADMAP 项}：
- 对应 ROADMAP 条目：{ID} — {描述}
- 升级原因：{用户为什么决定从 ROADMAP 升到正式 feature}

{否则}：
- 与 ROADMAP 无关联（新发现的需求）
```

### Step 5: 输出总结

```
✅ 需求澄清完成

生成的 docs/ 文件：
- docs/feature-{N}-{name}.md
{如果拆分了多个，全部列出}

下一步：
1. （可选）打开上面文件人工微调，把"未明确"补全或保留
2. 跑 `/tc-prd <项目路径>` 生成 specs 三件套
3. 如果某个 feature 实际是改已有 feature → 改用 `/tc-prd --change`

注：本次对话产生的 Q-XXX 待定问题会由 /tc-prd Step 5.5 再次确认。
```

### Step 6: 自检（命令完成前必查）

- [ ] 至少经过 1 轮对话（不允许从 $ARGUMENTS 直接出 docs，必须有澄清）
- [ ] 生成的每个 docs 都包含全部 9 个章节（起源 / 业务背景 / 必做 / 不做 / 已知决策 / 待定决策 / 现有交互 / 拆分说明 / ROADMAP 关系）
- [ ] 没有写技术方案 / API 接口 / 数据库 schema（那是 /tc-prd 的事）
- [ ] 没有自行假设过拆分 — 必须用户确认
- [ ] 多文件之间没有内容重复（拆分后每个文件职责清晰）

## 输出口径（强制）

每轮对话遵循 `/tc-ai` 的可观测性约定：

- 每轮提问前先输出「📍 第 X 轮：{主题}」
- 每轮回答后输出确认「✓ 已记录：{摘要}」
- 拆分判断输出「🔀 拆分检查：建议 / 不需要」
- 生成文件后输出「📝 已生成：docs/...」

## 反模式

- ❌ **越俎代庖写 design**：不要写技术方案、API 设计、数据模型 — 那是 /tc-prd Step 9 的事
- ❌ **写完整 F-XXX / AC-XXX**：那是 /tc-prd Step 8 的事；本命令只列「必做范围」自然语言条目
- ❌ **自行决定拆分不问用户**：哪怕觉得明显该拆，也要 Step 4 给出方案让用户确认
- ❌ **一次问 10 个问题**：每轮 2-3 个，分主题分批
- ❌ **追问到用户烦**：3 轮没说清就停下来，把"未明确"标进 docs，让 /tc-prd Step 5.5 再问
- ❌ **跳过 Step 2 重叠检测**：直接生成新 docs 可能与已存在 feature 重复
- ❌ **生成 1 个 docs 但内容是 N 个 feature 大杂烩**：违反 tc-prd 的「单 feature 任务 ≤ 15」限制

## 与已有命令的关系

```
混沌阶段                半结构化               结构化               代码                   验证

[聊需求]   →   docs/feature-X.md   →   N.feature/三件套   →   实施   →   跑测试

/tc-discuss     /tc-prd <path>           /tc-ai <path>            /tc-test
   ↑                ↑                          ↑                       ↑
   多轮问答         读 docs                   按 specs 实施            跑现有测试
   输出 docs        生成 specs                 调 Skills
```

| 命令 | 输入 | 输出 | 何时用 |
|---|---|---|---|
| `/tc-init` | （项目根目录）| `.claude/CLAUDE.md` + `rules/` | 项目首次创建 |
| **`/tc-discuss`** | **一句话 + 多轮对话** | **`docs/feature-N-{name}.md`（1 或 N 个）** | **每次有新需求想头脑风暴** |
| `/tc-prd <path>` | `docs/*.md` | `N.{feature}/` 三件套 | docs 已就绪，要正式立项 |
| `/tc-prd --change` | 变更描述 | 已有 specs 加 v2 标注 | 已立项的 feature 改需求 |
| `/tc-ai <specs> <code>` | specs 三件套 + 代码项目 | 实施代码 | specs 审过，开始干活 |

## 示例用法

### 单 feature 走完全流程

```bash
# 1. 聊需求
/tc-discuss 想给医生加头像上传，最好支持裁剪
   ↓ 多轮对话澄清
   ↓ 用户确认不需要拆分
   ↓ 生成 docs/feature-11-doctor-avatar-upload.md

# 2. 立项
/tc-prd /path/to/project
   ↓ 读取 docs/feature-11-*.md
   ↓ Step 5.5 再次确认 Q-XXX
   ↓ 生成 11.doctor-avatar-upload/{requirements,design,tasks}.md

# 3. 开发
/tc-ai specs在/path 代码也在/path
   ↓ N1 → N2 → Task 循环 → N8
   ↓ 完成
```

### 多 feature 拆分

```bash
/tc-discuss 我们要做用户续费，要支持卡密兑换 + 微信支付 + 发票管理 + 订单查询
   ↓ 命令在 Step 4 识别需要拆
   ↓ 建议拆 4 个 feature 让用户确认
   ↓ 用户改成拆 3 个（卡密+订单合并）
   ↓ 生成 docs/feature-12-card-redemption-and-order.md
   ↓ 生成 docs/feature-13-wechat-payment.md
   ↓ 生成 docs/feature-14-invoice-management.md

# 后续每个 feature 单独立项 + 单独开发
/tc-prd /path  # 会读 3 个 docs，可能要选「为哪个 doc 立项」（暂时是命令限制：一次跑一个）
```

### 直接命中已有 feature（节省时间）

```bash
/tc-discuss 我想加用户改密码

🔍 检测到匹配：2.account-management 已规划 [F-001] 用户自助改密
   状态：specs-ready

请选择：
  A) 实施 → 跳出，运行 /tc-ai
  B) 描述的是别的 → 继续

→ 用户选 A → 命令退出，提示运行 /tc-ai
```

## 失败模式

| 场景 | 处理 |
|---|---|
| 用户初始描述太抽象（如「优化前端」）| 第一轮就让用户具体化：「具体哪个页面 / 哪种优化？性能 / 设计 / 体验？」|
| 用户拒绝任何澄清问题（直接说「随便你」）| 不强求，把所有未答标 `[未明确]` 写进 docs；Step 5.5 时让 /tc-prd 再问 |
| 多轮后用户改主意「先不做这个了」| 不生成 docs，输出「✋ 取消，未生成任何文件」就退出 |
| 检测到 docs/ 已有同名文件（feature-N-{name}.md 冲突）| 提示「已存在 docs/feature-X，是覆盖 / 改名 / 跳过？」|
