---
name: pipeline-lite
description: 轻量流程控制 skill。用户输入文本需求后，先自检 .gitignore 忽略临时目录并删除它，由主会话生成规格文档(spec)、生成执行计划(plan)，每个关键阶段由用户把关，确认后才按 plan 执行任务（不自行提交），执行完成后询问是否提交（自动生成的提交信息先经用户确认）、是否推送。无子 Agent、无状态文件、无 AI 校验，运行时只依赖 .pipeline-lite/tmp/。触发词：轻量流程、pipeline-lite、简化流程、快速开发流程、lite 流程、精简流程、简单开发流程
compatibility:
  type: claude
  version: ">=4.0"
---

# Pipeline-Lite — 轻量流程控制 Skill

将一次用户需求，沿着一条线性、带门禁的交互流程推进：**删除临时目录 → 生成 spec → 用户把关 → 生成 plan → 用户把关 → 执行（不自行提交）→ 询问提交 → 询问推送**。所有动作由主会话（父 Agent）顺序完成，每个关键阶段都以用户显式选择作为推进条件。

## 核心定位

`pipeline-lite` 是一套**自包含的轻量方案**，与项目内现存的 `pipeline`（workflow）这类重 harness 不同，去掉子 Agent、状态机文件与 AI 校验循环，只保留「生成产物 → 用户把关 → 执行 → 提交 → 推送」的最小交互骨架。

| 维度 | 重 harness（pipeline 类） | **pipeline-lite（本 skill）** |
|------|--------------------------|------------------------------|
| 子 Agent | spec/plan reviewer、execution 等 | **无**，全部由主会话（父 Agent）完成 |
| 状态文件 | state.json + status 状态机 | **无**，不持久化状态 |
| AI 校验 | spec/plan 多轮 reviewer 校验 | **无**，以用户确认代替 AI 校验 |
| 文件范围 | allowedPaths 白名单校验 | **无** |
| 产物目录 | .harness/temp/{vId}/ | **`.pipeline-lite/tmp/`** |
| 流程 | 生成 → 校验循环 → 确认 → 执行 → 验收 → 推送 | 生成 → 用户把关 → 执行 → 提交 → 推送 |

> **自包含约定**：本 skill 实现时不得复用/拷贝项目内现有 harness 的代码或目录结构，按本文件规格独立实现。

## 设计原则

1. **轻量**：不引入子 Agent、状态文件、校验循环，所有动作由主会话顺序完成。
2. **门禁式**：每个关键阶段（spec / plan / 提交 / 推送）都由用户显式选择后推进。
3. **用户可控**：任何一步用户都可以选择「重新生成」或「补充信息修改」，而非只能一路到底。
4. **不越权**：执行阶段不自行 `git commit`；提交与推送必须等用户明确指示。
5. **自包含**：运行时只依赖 `.pipeline-lite/tmp/` 一个目录，不触碰 `.harness/` 或其它 skill 的产物。

## 工作流总览

```
调用 skill
  │
  ▼
Step 0  自检 .gitignore 忽略 .pipeline-lite/ + 删除 .pipeline-lite/   （每次开始必做）
  │
  ▼
Step 1  读取用户需求 + 上下文
  │
  ▼
Step 2  生成 spec.md → .pipeline-lite/tmp/
  │
  ▼
Step 3  用户选择：1. 下一步（生成 plan） / 2. 重新生成 / 3. 补充信息修改
  │         （选 2/3 回到 Step 2，直到选 1）
  ▼
Step 4  生成 plan.md → .pipeline-lite/tmp/
  │
  ▼
Step 5  用户选择：1. 下一步（开始执行） / 2. 重新生成 / 3. 补充信息修改
  │         （选 2/3 回到 Step 4，直到选 1）
  ▼
Step 6  按 plan 执行任务（不自行 git commit）
  │
  ▼
Step 7  完成 → 用户选择：1. 提交·自动生成提交信息 / 2. 提交·手动输入 / 3. 不提交
  │         （选 3 直接结束）
  ▼
Step 8  已提交 → 用户选择：是否推送远程服务（1. 是 / 2. 否）
```

## 目录规则

### 1. 运行时目录（.pipeline-lite/tmp/）

所有运行时产物只放这一个目录：

| 文件 | 说明 |
|------|------|
| `.pipeline-lite/tmp/spec.md` | 规格文档（Step 2 生成，Step 3 重生成时覆盖） |
| `.pipeline-lite/tmp/plan.md` | 执行计划（Step 4 生成，Step 5 重生成时覆盖） |

规则：

1. **每次调用 skill 时先删除**整个 `.pipeline-lite/` 目录（用 `remove-dir.js`，详见 Step 0），保证上次运行残留不影响本次；目录不预先创建，`spec.md` / `plan.md` 写入时自动生成。
2. 该目录为纯运行时状态，**不得进入 git 提交**。开始任务前须先自检项目根目录 `.gitignore`：若其中未忽略 `.pipeline-lite/`（该条目覆盖 `.pipeline-lite/tmp/`），用本 skill `scripts/` 下的 `check-exist.js` / `write-file.js` 两个 CLI 工具检测并追加写入（详见 Step 0）。
3. 主会话执行期间实时读取/覆盖这两个文件，不使用子 Agent 报告往返。

### 2. Skill 源文件目录

```
skills/
└── pipeline-lite/            # 本 skill 独立文件夹
    ├── SKILL.md               # 入口：触发词 + 完整工作流（本文件）
    ├── assets/                # （可选）占位
    ├── references/            # （可选）占位
    └── scripts/               # 三个 CLI 工具（Node.js，跨平台）
        ├── check-exist.js     # 检测文件中是否存在指定字符串
        ├── write-file.js      # 向文件追加内容（不存在则连同父目录创建）
        └── remove-dir.js      # 递归删除目录（不存在则静默跳过）
```

- `assets/`、`references/` 保持占位，不强制承载内容。
- `scripts/` 承载 Step 0 所需的三个 CLI 工具，均为 **Node.js 脚本**（要求 Node.js v16 及以上，脚本头部注释已注明；mac/linux/windows 均可运行）：
  - `check-exist.js <文件路径> <查找字符串>`：检测文件（如 `.gitignore`）中是否存在指定字符串。退出码 `0`=存在、`1`=不存在（含文件不存在）、`2`=用法错误。
  - `write-file.js <文件路径> <要追加的内容>`：向文件末尾追加一行，文件不存在时连同父目录一起创建；文件末尾已有内容且未以换行结尾时先补换行。
  - `remove-dir.js <目录路径>`：递归删除目录（含其下全部内容）；目录不存在则静默成功。退出码 `0`=成功、`2`=用法错误。
  - 调用时相对本 skill 目录定位脚本，用 `node` 执行，如 `node scripts/remove-dir.js .pipeline-lite`。

## 分步流程

### Step 0：初始化（gitignore 自检 + 删除运行时目录）

**第一步：gitignore 自检（确保临时目录不进版本库）**

1. 用 `check-exist.js` 检测项目根目录 `.gitignore` 中是否已包含 `.pipeline-lite/`（该条目覆盖 `.pipeline-lite/tmp/`）：
   ```
   node scripts/check-exist.js .gitignore '.pipeline-lite/'
   ```
   （`scripts/` 相对本 skill 目录，即 SKILL.md 所在目录的 `scripts/` 子目录；两个工具均为 Node.js 脚本，要求 Node.js v16 及以上，mac/linux/windows 均可运行。）
2. 依据退出码处理：
   - `0`（已忽略）→ 跳过写入，直接进入下一步删除。
   - `1`（未忽略，或 `.gitignore` 文件本身不存在）→ 用 `write-file.js` 追加写入：
     ```
     node scripts/write-file.js .gitignore '.pipeline-lite/'
     ```
   - `2`（用法错误）→ 检查参数后重新执行。

**第二步：删除运行时目录（清空临时文件）**

1. 用 `remove-dir.js` 删除项目根目录的 `.pipeline-lite` 目录（含 `spec.md`、`plan.md` 及任何残留）：
   ```
   node scripts/remove-dir.js .pipeline-lite
   ```
   （`scripts/` 相对本 skill 目录，即 SKILL.md 所在目录的 `scripts/` 子目录；目录不存在时脚本静默成功，无需特殊处理。）
2. 目录**不预先创建**；后续 Step 2 / Step 4 写入 `spec.md` / `plan.md` 时自动生成 `.pipeline-lite/tmp/`。
3. 无需向用户展示状态，静默完成即可。

### Step 1：需求与上下文

读取用户本次输入的需求，并结合当前会话/项目上下文理解意图。保持轻量：**不做额外澄清提问**，直接进入 spec 生成；信息不足的部分在 Step 3 由用户通过「补充信息修改」弥补。

### Step 2：生成 spec.md

由主会话**亲自撰写** `spec.md` 并写入 `.pipeline-lite/tmp/spec.md`，内容采用精简结构。生成时识别需求中的**边界问题**（如：范围是否包含某项、特殊/异常输入的预期行为、未明确的规则、技术选型取舍、兼容性/性能边界等），若存在则在 spec 末尾以「需要确认点」逐条列举并附**推荐做法**；无边界问题则注明"无"：

```
# {需求标题}

## 背景与目标
（为什么做、要达成什么）

## 功能需求
（本次要实现的功能点清单）

## 非功能需求
（性能、兼容性、约束等，无则注明"无"）

## 技术方案（简要）
（选用的技术/做法，一段话即可）

## 文件结构（简要）
（预计涉及/新增的文件，无则注明"待 plan 细化"）

## 需要确认点
（边界问题/歧义/不确定决策，存在则逐条列举并附推荐做法；无则注明"无"）
- 确认点 1：{边界问题描述}
  - 推荐做法：{建议采用的做法}
- 确认点 2：{边界问题描述}
  - 推荐做法：{建议采用的做法}
```

> 轻量约定：不启动 AI 校验、不生成建议文件、不维护状态。

### Step 3：spec 门禁（用户选择）

向用户展示 spec 概要（由主会话直接总结；若末尾含「需要确认点」一并突出展示供用户确认），并给出三个选项：

```
spec 已生成：.pipeline-lite/tmp/spec.md

1. 下一步     → 进入 Step 4 生成 plan
2. 重新生成   → 回到 Step 2，重新撰写 spec.md（覆盖旧文件）
3. 补充信息修改 → 用户补充/修正需求 → 回到 Step 2 带补充内容重新撰写
```

- 选 2：删除或直接覆盖当前 `spec.md`，重新执行 Step 2，再次回到本步。
- 选 3：等待用户输入补充信息，清除后带补充内容重新执行 Step 2，回到本步。
- 选 1：进入 Step 4。

### Step 4：生成 plan.md

主会话读取已确认的 `spec.md`，亲自拆解为可独立完成/可独立提交的步骤，写入 `.pipeline-lite/tmp/plan.md`：

```
# {需求标题} — 执行计划

## 步骤 1：{步骤名称}
- 目标：（本步达成什么）
- 涉及文件：（预计改动/新增的文件）
- 验收要点：（如何判断本步完成）

## 步骤 2：{步骤名称}
...
```

> 轻量约定：每步可独立完成即可，不汇总 allowedPaths，不启动 plan-reviewer。

### Step 5：plan 门禁（用户选择）

向用户展示 plan 概要，并给出三个选项：

```
plan 已生成：.pipeline-lite/tmp/plan.md

1. 下一步开始执行 → 进入 Step 6
2. 重新生成       → 回到 Step 4，重新撰写 plan.md
3. 补充信息修改   → 用户补充信息 → 回到 Step 4 带补充内容重新撰写
```

- 选 2：删除或直接覆盖当前 `plan.md`，重新执行 Step 4，回到本步。
- 选 3：等待用户补充信息，带补充内容重新执行 Step 4，回到本步。
- 选 1：进入 Step 6。

### Step 6：执行

按 `plan.md` 从步骤 1 依次执行：

1. 每步完成后可运行必要的验证/测试（轻量原则：不强制分层验证）。
2. 若某步执行失败，停下来向用户说明失败原因与已做改动，由用户决定继续/修正。
3. **关键约束：执行全程不自行 `git commit`**，所有改动保留在工作区，交由 Step 7 由用户决定提交方式。
4. 全部步骤完成后，向用户简要汇报执行结果（完成哪些、改动哪些文件、测试结果）。

### Step 7：提交询问

执行完成后，向用户展示三个选项：

```
执行完成。请选择提交方式：

1. 提交 - 自动生成提交信息  → 主会话根据本次改动自动撰写 commit message，提交前先给用户确认
2. 提交 - 手动输入提交信息  → 用户提供 commit message，主会话执行 git add . + commit
3. 不提交                  → 保留工作区改动，流程结束
```

- 选 1：主会话根据本次改动自动撰写 commit message（格式见下方「提交信息规范」），**先展示给用户确认**：
  ```
  拟提交信息：feat: 添加用户登录接口

  请选择：
  1. 确定提交   → 使用该信息执行 git add . + commit，随后进入 Step 8
  2. 修改内容   → 根据用户意见修订信息，重新展示确认
  3. 其他描述   → 用户直接提供 commit message，使用之执行 git add . + commit
  ```
  - 选 1：执行 `git add .` + `git commit`（使用已确认的信息），随后进入 Step 8。
  - 选 2：等待用户给出修改意见，主会话修订后重新展示上述确认界面（可反复，直到用户选 1 或 3）。
  - 选 3：等待用户直接提供描述，主会话使用该描述执行 `git add .` + `git commit`，随后进入 Step 8。
- 选 2：执行 `git add .` + `git commit`（使用用户提供的 message），随后进入 Step 8。
- 选 3：不执行任何 git 操作，流程结束。

> **提交前统一执行 `git add .`**（在项目根目录）：一次性暂存全部改动（新增 / 修改 / 删除），避免遗漏文件。`git commit` 仅出现在提交分支，Step 6 执行阶段仍禁止提交。

#### 提交信息规范

自动生成的提交信息遵循 **Conventional Commits** 规范（[conventionalcommits.org](https://www.conventionalcommits.org/zh-hans/v1.0.0/)）：

```
<type>[optional scope]: <description>

[optional body]

[optional footer(s)]
```

规则：

- **type**（小写）：本次改动的类型，取值 `feat`（新功能）/ `fix`（修复）/ `docs`（文档）/ `style`（格式，不影响逻辑）/ `refactor`（重构，非新增功能或修复）/ `perf`（性能）/ `test`（测试）/ `build`（构建）/ `ci`（CI）/ `chore`（杂务）/ `revert`（回滚）。
- **scope**（可选）：影响范围，写在 type 后的括号内，如 `feat(auth): ...`。
- **description**：中文，一句话说清改动，**不超过 50 个汉字**。
- **breaking change**（可选）：破坏性变更在冒号前加 `!`（如 `feat!: ...`），或正文中写 `BREAKING CHANGE: <说明>`。

示例：

```
feat: 添加用户登录接口
fix: 修复登录超时未提示的问题
refactor: 提取通用认证中间件
feat(auth): 支持 JWT 登录
feat!: 升级 API 响应格式，不兼容旧客户端
```

### Step 8：推送询问

仅在 Step 7 选择了「提交」后触发：

```
是否推送远程服务？

1. 是   → 执行 git push
2. 否   → 结束流程
```

- 选 1：执行 `git push`，结束后汇报推送结果。
- 选 2：流程结束。

## 用户选择界面汇总

| 阶段 | 选项 |
|------|------|
| spec 生成后（Step 3） | 1. 下一步（生成 plan） / 2. 重新生成 / 3. 补充信息修改 |
| plan 生成后（Step 5） | 1. 下一步（开始执行） / 2. 重新生成 / 3. 补充信息修改 |
| 执行完成后（Step 7） | 1. 提交 - 自动生成提交信息 / 2. 提交 - 手动输入提交信息 / 3. 不提交 |
| 自动生成提交信息后（Step 7 确认） | 1. 确定提交 / 2. 修改内容 / 3. 其他描述 |
| 提交完成后（Step 8） | 1. 推送远程 / 2. 不推送 |

选项编号与文案以本表为准，实现时保持一致。

## 关键约束

| # | 约束 |
|---|------|
| 1 | 每次调用先自检 `.gitignore` 忽略 `.pipeline-lite/`（用 `check-exist.js`/`write-file.js`），再用 `remove-dir.js` 删除 `.pipeline-lite/`，不留上次残留 |
| 2 | `spec.md` / `plan.md` 一律写入 `.pipeline-lite/tmp/` |
| 3 | 不启动任何子 Agent、不创建状态文件、不做 AI 校验 |
| 4 | 执行阶段**不自行 `git commit`** |
| 5 | 提交信息可自动生成或用户手动输入，且必须由用户先选择提交方式；自动生成的提交信息须先经用户确认（确定提交/修改内容/其他描述）后才执行 `git commit`；提交前统一 `git add .` 暂存全部改动，避免遗漏 |
| 6 | 自动生成的提交信息遵循 Conventional Commits 规范：`<type>[scope]: <描述>`，type 11 类、可带 scope 与 breaking change |
| 7 | 推送必须在提交完成后、且经用户同意才执行 |
| 8 | 实现自包含，不引用/复用项目内其它 harness 的代码或目录 |
| 9 | 开始任务前确保项目根目录 `.gitignore` 忽略 `.pipeline-lite/`（`check-exist.js` 检测，缺失时 `write-file.js` 写入），不进入版本库 |
