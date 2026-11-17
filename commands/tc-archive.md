---
description: 归档已完成 feature — 把 specs 目录从主区移到 _archive/，保留 git 历史，主区只留进行中需求
---

# /tc-archive — 归档已完成 feature

把**已完成**的 feature specs 目录从 `{SPECS_DIR}/` 主区移到 `{SPECS_DIR}/_archive/`，让主区只剩**进行中 / 待开工**的需求。**绝不删除任何文件**,**绝不绕过校验**,**绝不自动 commit**。

## 核心原则（铁律）

1. **不删除,只搬迁**：永远用 `git mv`（或 `mv` 后让用户自己 commit），保留 git 历史。**禁止**任何 `rm` / `rm -rf`。
2. **强校验未完成不让归档**：`tasks.md` 必须**全部任务**都是 `[x]` / `[DROPPED]` / `[SKIP]` / `[CANCELLED]` 之一，**只要有一个 `[ ]`** 就拒绝归档,并指出哪些任务还没收尾。**不提供 `--force` 跳过校验**——归档错了恢复成本远高于"再跑一次 tc-ai 收尾"。
3. **让用户从列表选编号**：与 `/tc-test` 一致——不让用户手打 feature 名字。命令会列出"可归档清单"，用户回复 1/2/3。
4. **不自动 commit**：搬完 + INDEX 更新后输出"待 commit 的文件清单"，由用户自己 `git add && git commit`。
5. **不动 specs 内容**：归档过程**不修改** `requirements.md` / `design.md` / `tasks.md` / `LESSONS.md` 内任何字符——只是改路径。
6. **保留原编号**：归档后路径是 `_archive/N.{feature}/`（不重新编号、不加日期后缀）。后续新 feature 编号继续从主区最大编号 +1，不复用归档区编号。

## 何时用

- 一个 feature 全部 task 已完成，想从主区"清场"，让主区只剩进行中的需求
- 长期项目里历史 feature 堆了很多，想缩小 `tc-ai`/`tc-prd` 的扫描范围
- 准备开新一轮 sprint，归档上一轮成果
- 项目交付前清理目录结构

## 何时不用

- feature 还在做（哪怕只剩 1 个 task）→ 继续 `/tc-ai` 跑完
- 想"暂停"一个 feature → **不要归档**。归档语义是"已完结",不是"暂停"。暂停请在 `tasks.md` 顶部加注释（如 `> ⏸ paused 2026-04-30 because ...`），保留在主区。
- 想删除一个废弃 feature → 也**不归档**。直接 `git rm -r N.{feature}/`（破坏性操作由用户自己执行；本命令不提供）。

## 输入参数

```bash
/tc-archive                    # 列出"可归档清单"让用户从编号选（默认）
/tc-archive 4                  # 直接归档编号 4（仍校验完成度）
/tc-archive 1 4 5              # 批量归档（按编号列表，逐个独立校验，单个失败不影响其它）
/tc-archive --list             # 仅打印可归档清单 + 主区清单 + 已归档清单，不执行任何动作
/tc-archive --dry-run 4        # 模拟归档：完整校验 + 打印将要执行的 mv 命令，但不实际移动
```

**不支持的参数**（明确拒绝）：

```bash
/tc-archive --force            # ❌ 不存在。归档不允许跳过完成度校验
/tc-archive --unarchive 4      # ❌ 不存在。反归档请直接 git 操作（git mv _archive/4.foo 4.foo + git commit）
/tc-archive --delete 4         # ❌ 不存在。归档不删；删除请用户自己 git rm
/tc-archive all                # ❌ 不存在。必须按编号选，避免一键打包错归档
```

## 流程

### Step 0: 解析 `SPECS_DIR`

按 `/tc-prd` / `/tc-test` 的统一约定查找：

1. `{PROJECT_DIR}/docs/tc-spec/` — 优先（标准约定）
2. `{PROJECT_DIR}/doc/tc-spec/` — 兼容
3. `{PROJECT_DIR}/specs/` — 兼容（旧版本约定）
4. **平铺布局兜底**：项目根 / 一级子目录里直接散落形如 `N.{name}/` 且含 `tasks.md` 的目录（如 `A_frontend_backend/1.medscout-platform/`）。这种情况记 `SPECS_DIR = 包含 N.{name}/ 的那个父目录`，并提示用户："检测到平铺布局,归档目录将创建在 `{SPECS_DIR}/_archive/`。"

记为 `SPECS_DIR`。**全部找不到** → 报错 + 引导跑 `/tc-prd` 立项后再来：

```text
❌ 找不到 specs 目录

已检查路径（按优先级）：
  - docs/tc-spec/         不存在
  - doc/tc-spec/          不存在
  - specs/                不存在
  - 一级子目录平铺扫描     未找到任何 N.{name}/tasks.md

📌 处理：
  1) 项目还没立项 → 先跑 /tc-prd 创建第一个 feature
  2) specs 在非常规位置 → 把目录路径告诉我（暂不支持自定义 SPECS_DIR 参数）

⛔ /tc-archive 在此结束
```

### Step 1: 扫描三类目录

在 `SPECS_DIR/` 下分别扫描：

| 类别 | 路径模式 | 含义 |
|---|---|---|
| 主区 feature | `SPECS_DIR/N.{name}/` 且含 `tasks.md` | 进行中 / 待归档 |
| 已归档 feature | `SPECS_DIR/_archive/N.{name}/` | 历史归档 |
| 其它 | 不匹配上面两类的内容 | 忽略（不动它） |

对每个**主区 feature**做完成度判断：

```
读 tasks.md → grep checkbox 行
  - 计数: done = `[x]` 数量
  - 计数: dropped = `[DROPPED]` / `[SKIP]` / `[CANCELLED]` 数量（含在描述里的）
  - 计数: pending = `[ ]` 数量
  
判定:
  - pending == 0 且 (done + dropped) > 0  → 完成 ✓ 可归档
  - pending > 0                           → 未完成 ✗ 不可归档（列出未完成任务编号）
  - done + dropped == 0 且 pending == 0   → 空 tasks.md，警告但允许归档（边界）
```

**严格匹配规则**（避免误判）：

- checkbox 行必须以 `^\s*-\s*\[(x| |DROPPED|SKIP|CANCELLED)\]` 匹配
- `[DROPPED v2]` / `[CHANGED v2]` 这类**版本注解**出现在任务**描述里**（不是 checkbox 内）→ **不算 dropped**，按 checkbox 实际状态判定
- 形如 `- [ ] ~~T-005: ...~~ \`[DROPPED v2]\`` 的"删除线 + 注解"组合 → checkbox 是 `[ ]` 但描述含 `[DROPPED`、`~~~~` 删除线 → **算 dropped**（这是项目里的 v2 变更惯例，详见 `/tc-prd --change`）
- 行尾的 markdown 注释（`<!-- ... -->`）忽略

### Step 2: 输出"归档候选清单"

无论用户传不传编号，**都要先打印一次完整清单**，让用户在动作前对全局可见：

```text
🗂  /tc-archive — feature 状态总览
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

📂 主区（{SPECS_DIR}/）

  ✅ 可归档（已完成）:
     1) 0.5.frontend-ts-migration       12/12 done
     2) 0.6.backend-response-models     9/9 done
     3) 0.7.frontend-test-infra         26/26 done
     4) 1.medscout-platform             16/16 done + 2 dropped
  
  🚧 未完成（不可归档）:
     5) 2.account-management            12 done / 1 dropped / 4 pending
        ↳ 待办: T-001-NEW, T-002-NEW, T-013-NEW（去 tasks.md 看完整列表）
  
📦 已归档（{SPECS_DIR}/_archive/）

     （空 — 还没归档过任何 feature）

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

**未完成 feature 必须列出 pending 任务编号**（取前 3 个，超出标 `…N more`），帮用户判断"还差啥"。

### Step 3: 选定要归档的 feature

按以下决策树（**绝不直接默认开跑**）：

##### 情况 A：用户传了编号（`/tc-archive 4` 或 `/tc-archive 1 4 5`）

逐个校验编号：

- 编号不存在于 `FEATURE_LIST` → 报错 + 列出可用编号
- 编号在主区但**未完成** → 报错 + 列出该 feature 的 pending 任务
- 编号在 `_archive/` 已归档 → 报错"已归档"
- 多个编号有部分合法、部分不合法 → **整体取消**，输出哪些不合法（不允许"先归档合法的，剩下报错"）

全部合法 → 进入情况 D 二次确认。

##### 情况 B：用户没传编号（`/tc-archive`）

接 Step 2 的清单后追加：

```text
🤔 选择要归档的 feature（多个用空格分隔，例如 "1 2 4"）:
   回复编号，或回复 q 取消。
   
   提示: 一次归档多个 feature 也会逐个独立校验,
        但建议每次最多归档 3 个，方便 review。
```

接受用户回复：

| 回复 | 行为 |
|---|---|
| `1` / `1 2 4`（合法编号集合） | 进入情况 D |
| `q` / `cancel` / `取消` | 终止，不做任何动作 |
| 包含未完成 feature 编号（如 `5`） | 报错指出该项未完成 + 列出 pending，重新等待输入 |
| 包含不存在编号 | 报错 + 重新列出可用编号 |
| 空回复 / 直接回车 | 提示"请回复编号或 q 取消"，重新等待 |

##### 情况 C：`--list` 模式

打印完 Step 2 清单后**直接结束**，不进入选择。

##### 情况 D：选定后二次确认（**强制**，不可跳过）

```text
即将归档以下 feature（共 N 个）:

  1) 0.5.frontend-ts-migration
     {SPECS_DIR}/0.5.frontend-ts-migration/  →  {SPECS_DIR}/_archive/0.5.frontend-ts-migration/
  2) 0.6.backend-response-models
     {SPECS_DIR}/0.6.backend-response-models/  →  {SPECS_DIR}/_archive/0.6.backend-response-models/

执行后:
  ✓ 用 git mv 移动（如不在 git 仓库则用普通 mv）
  ✓ 更新 {SPECS_DIR}/_archive/INDEX.md
  ✓ 不会自动 commit

回复 y 确认执行 / n 取消 / d 改用 dry-run 看具体命令但不执行
```

`y` → 进入 Step 4
`n` 或 `cancel` → 终止
`d` → 切到 dry-run 模式（不执行 mv,只打印命令）

`--dry-run` 参数模式直接走 `d` 路径，不再问。

### Step 4: 执行归档（按 feature 逐个）

对每个选中的 feature：

#### 4.1 创建归档目录（如尚不存在）

```bash
mkdir -p {SPECS_DIR}/_archive/
```

#### 4.2 移动 feature 目录

**优先 `git mv`**（保留追踪历史，diff 显示为 rename 而非 delete + add）：

```bash
# 在 git 仓库内
git mv {SPECS_DIR}/N.{feature}/ {SPECS_DIR}/_archive/N.{feature}/
```

**不在 git 仓库 / 该路径未被 git 追踪** → 退化用普通 `mv`：

```bash
mv {SPECS_DIR}/N.{feature}/ {SPECS_DIR}/_archive/N.{feature}/
```

任一命令失败（目标已存在 / 权限错 / 跨设备等）→ **立即停止**，输出：

```text
❌ 归档失败: 0.5.frontend-ts-migration
   原因: {错误信息}
   原始命令: {命令}
   
✓ 已成功归档（在本次批量中先完成的）:
   - 1.medscout-platform
   
🔧 处理建议: 检查 {错误原因相关} 后重跑 /tc-archive
⛔ 停止后续归档操作
```

**已成功的不回滚**（避免引入二次破坏性操作）。后续编号失败需要用户自己解决环境问题再重跑。

#### 4.3 更新 `_archive/INDEX.md`

`INDEX.md` 是归档区的**自动维护**目录索引,格式：

```markdown
# 已归档 Features

> 自动维护 by `/tc-archive` · 不要手改

| 编号 | feature 名 | 归档时间 | 任务统计 | 路径 |
|---|---|---|---|---|
| 0.5 | frontend-ts-migration | 2026-04-28 | 12 done | [./0.5.frontend-ts-migration/](./0.5.frontend-ts-migration/) |
| 0.6 | backend-response-models | 2026-04-28 | 9 done | [./0.6.backend-response-models/](./0.6.backend-response-models/) |
| 0.7 | frontend-test-infra | 2026-04-28 | 26 done | [./0.7.frontend-test-infra/](./0.7.frontend-test-infra/) |
| 1   | medscout-platform | 2026-04-28 | 16 done + 2 dropped | [./1.medscout-platform/](./1.medscout-platform/) |

## 反归档（如需恢复到主区）

```bash
git mv {SPECS_DIR}/_archive/N.{feature}/ {SPECS_DIR}/N.{feature}/
# 然后手动从此表移除对应行
```
```

**写入规则**：

- INDEX 不存在 → 用上面骨架创建,然后追加本批次行
- INDEX 已存在 → 在表中按**编号升序**插入新行（不要无脑 append 到末尾）
- 不要重排既有行的格式（保持用户改过的微调）
- 归档时间用本机日期 `YYYY-MM-DD`
- "任务统计"按 `done + dropped` 简述，与 Step 1 判定的字段保持一致

#### 4.4 输出本次归档总结

```text
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📦 /tc-archive 完成
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

✓ 已归档 4 个 feature:
   - 0.5.frontend-ts-migration       (12 tasks done)
   - 0.6.backend-response-models     (9 tasks done)
   - 0.7.frontend-test-infra         (26 tasks done)
   - 1.medscout-platform             (16 done + 2 dropped)

📂 现在主区只剩:
   - 2.account-management            (进行中)

📄 已更新: {SPECS_DIR}/_archive/INDEX.md

🔻 下一步（手动执行,本命令不自动 commit）:

   git status                              # 查看变更
   git add {SPECS_DIR}/                    # 暂存（包含 _archive/ 与 INDEX.md）
   git commit -m "chore: 归档 4 个已完成 feature"

⚠ 提示:
   - 用 git diff 看不出内容（仅 rename）→ 用 git diff -M 或 git log --follow
   - 如需恢复: git mv {SPECS_DIR}/_archive/N.{feature}/ {SPECS_DIR}/N.{feature}/
```

### Step 5: 关联文档提示（**只提示，不自动改**）

归档完成后,扫描下列位置看是否引用了归档掉的 feature 路径，**给提示但不自动改**：

- `CLAUDE.md` / `.claude/CLAUDE.md` 里 grep `N.{feature}` 路径引用
- `LESSONS.md` 里 grep `N.{feature}` 引用
- `README.md` 里 grep `N.{feature}` 引用

**输出**（仅当扫到引用时打印）：

```text
ℹ️ 检测到以下文档可能仍引用了刚归档的 feature 路径:

   .claude/CLAUDE.md:42  → ../1.medscout-platform/requirements.md
   A_frontend_backend/LESSONS.md:13  → 1.medscout-platform/design.md

🤔 路径已变（移到 _archive/ 下了）。是否需要我帮你更新引用路径?
   回复 y 让我把引用改为指向 _archive/ 路径
   回复 n 保持原样（你自己决定是否改）

ℹ️ 注：N5/N8 节点写的 LESSONS 引用通常用相对路径,改了之后可能跨目录链接更长——
       如果你只是临时归档可能很快又改回，建议先 n 不动它。
```

不扫到引用 → 不输出此节。

## Dry-run 输出格式

`--dry-run 4` 或在 Step 3 用户回复 `d`：

```text
🔍 /tc-archive --dry-run

会执行以下命令（不实际执行）:

  mkdir -p docs/tc-spec/_archive/
  git mv docs/tc-spec/4.dashboard/ docs/tc-spec/_archive/4.dashboard/
  
会更新文件:
  docs/tc-spec/_archive/INDEX.md
  
  追加行:
  | 4 | dashboard | 2026-04-28 | 8 done + 1 dropped | [./4.dashboard/](./4.dashboard/) |

完成度校验结果:
  ✓ 4.dashboard — tasks.md: 8 done / 1 dropped / 0 pending

⛔ Dry-run 模式: 没有任何文件被改动。
```

## 完整示例

### 示例 1: 默认交互（无参数）

```bash
$ /tc-archive

🗂  /tc-archive — feature 状态总览
[完整清单输出, 见 Step 2]

🤔 选择要归档的 feature（多个用空格分隔,例如 "1 2 4"）:

> 1 2 3 4

即将归档以下 feature（共 4 个）:
[Step 3 情况 D 二次确认]

> y

[Step 4 执行]

📦 /tc-archive 完成
✓ 已归档 4 个 feature
[Step 4.4 总结输出]
```

### 示例 2: 直接传编号

```bash
$ /tc-archive 4
[跳过情况 B 列表，直接进入情况 D 二次确认]
> y
[执行]
```

### 示例 3: 试图归档未完成 feature

```bash
$ /tc-archive 5

❌ 不能归档 2.account-management — 还有 4 个待办任务

未完成的 task:
  - T-001-NEW: ...
  - T-002-NEW: ...
  - T-013-NEW: ...
  ...1 more

完整清单见: {SPECS_DIR}/2.account-management/tasks.md

📌 建议:
   1) 跑 /tc-ai 让节点流程把剩余 task 收尾
   2) 或确认这些 task 该 [DROPPED] → 编辑 tasks.md 把对应行的 [ ] 改成 [DROPPED]

⛔ /tc-archive 在此结束（未做任何变更）
```

### 示例 4: 列表模式 (`--list`)

```bash
$ /tc-archive --list
[只打印 Step 2 清单后结束，不进入选择]
```

## 反模式

完成度校验相关：

- ❌ **绝不允许 `--force` 跳过完成度校验** —— 归档错的恢复成本远高于把 task 标完
- ❌ **绝不允许"自动把 [ ] 改成 [DROPPED] 然后归档"** —— 这是**修改 specs 内容**，违反核心原则 5；用户必须自己显式标记
- ❌ **绝不允许"任务版本注解里有 [DROPPED v2] 就当作 dropped"** —— 看 checkbox 实际状态(配合删除线惯例)，注解只是描述

搬迁动作相关：

- ❌ **绝不允许 `rm -rf` / 任何 delete 操作** —— 归档 = 搬迁，不是删除
- ❌ **绝不允许跳过 git 仓库判断直接 mv** —— 在 git 仓库里必须 `git mv`，否则丢追踪历史
- ❌ **绝不允许在归档时同时改 specs 内容**（如自动加"已归档"水印） —— 内容一字不动
- ❌ **绝不允许批量归档时部分失败后回滚已成功的** —— 已成功的保持归档,失败的跳过；用户自行 review

交互相关：

- ❌ **绝不允许"自动归档全部已完成 feature"** —— 必须列出 + 用户按编号选；即使是 `/tc-archive` 无参也要先列再选
- ❌ **绝不允许让用户手打 feature 名字** —— 一律按编号选
- ❌ **绝不允许跳过二次确认（情况 D）** —— 哪怕用户传了 `4` 也要列出"将要 mv 的命令"再问一次

文档同步相关：

- ❌ **绝不允许自动改 CLAUDE.md / README.md 里的引用路径** —— 只提示不动手；用户决定要不要改
- ❌ **绝不允许自动 git add / git commit** —— 归档是结构性变更，commit message 由用户决定

## 与已有命令的关系

```
/tc-prd       立项 → SPECS_DIR/N.{feature}/
   ↓
/tc-ai        实施 → tasks 标 [x]
   ↓
/tc-test      验证 → 报告写到 SPECS_DIR/N.{feature}/test-report.md
   ↓
/tc-archive   归档 → 移到 SPECS_DIR/_archive/N.{feature}/  ← 本命令
```

| 命令 | 触发时机 | 输出 |
|---|---|---|
| `/tc-prd` | 立项 | 主区新增 `N.{feature}/` 三件套 |
| `/tc-ai` | 实施 | tasks.md 勾 `[x]`，可能追加 LESSONS.md |
| `/tc-test` | 验证 | `N.{feature}/test-report.md` |
| **`/tc-archive`** | **完成后清场** | **`_archive/N.{feature}/` + INDEX.md** |

## 失败模式

| 场景 | 处理 |
|---|---|
| `SPECS_DIR` 不存在 | 拦截 + 引导跑 `/tc-prd` |
| 选中编号不存在 | 拦截 + 列出可用编号 |
| 选中 feature 未完成 | 拦截 + 列出 pending 任务 + 建议 `/tc-ai` 收尾 |
| 选中 feature 已在 `_archive/` | 拦截 + 提示"已归档" |
| `git mv` 失败（目标已存在） | 拦截 + 检查 `_archive/N.{feature}/` 是否已有同名（人工冲突）|
| 不在 git 仓库 | 提示"将用 mv 而非 git mv"，继续执行（但用户的版本控制责任自负） |
| 写 INDEX.md 失败 | 已 mv 的 feature 不回滚；提示用户手动追加 INDEX 行 |
| `tasks.md` 不存在 | 视为"非 feature 目录"，跳过（不当作可归档项） |
| 主区一个 feature 都没有 | 提示"主区为空，无可归档" 后退出 |

## 全局规则（与其它 tc-* 命令一致）

**强制拦截（绝不放行）：**

- `SPECS_DIR` 找不到 → 立即停止 + 引导 `/tc-prd`
- 任意选中编号未完成 → 立即停止 + 列出 pending（批量场景：整体停止，不"先归档已完成的"）
- 二次确认用户没明确 `y` → 视为取消

**报告但不阻止：**

- 关联文档（CLAUDE.md / README.md / LESSONS.md）有路径引用 → 仅提示，不自动改
- 不在 git 仓库 → 用普通 `mv`，但提示用户后果

## 输出口径

每步用 `━━━━━` 分隔，状态用图标：

- 🗂  全局清单
- ✅ 完成 / 可归档
- 🚧 未完成 / 不可归档
- 📦 已归档 / 归档完成
- 📂 主区
- 📄 文件操作
- ⚠️ 提示（不阻塞）
- ℹ️  信息
- ❌ 拦截
- ⛔ 停止
