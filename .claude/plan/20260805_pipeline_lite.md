# Plan: Pipeline-Lite — 轻量流程控制 Skill 实现

## 概述

基于 `.claude/spec/20260805_pipeline_lite.md` 规格文档，新建一套**轻量级、自包含**的流程控制 skill：`skills/pipeline-lite/`。核心是一条线性、带门禁的交互流程：**清空临时目录 → 生成 spec → 用户把关 → 生成 plan → 用户把关 → 执行（不自行提交）→ 询问提交 → 询问推送**。

与现有重 harness（`skills/pipeline/`）的本质区别：

```
pipeline（重）：  子 Agent 生成/审查 + state.json 状态机 + allowedPaths 白名单 + AI 校验循环 + .harness/
pipeline-lite：  全部由父 Agent（主会话）完成 + 无状态文件 + 无白名单 + 用户确认代替 AI 校验 + .harness-lite/
```

**设计约定（规格 §1/§8）**：实现时**不得复用/拷贝**项目内现有 harness 的代码或目录结构，`pipeline-lite` 按规格自包含实现。运行时只依赖仓库根目录下的 `.harness-lite/tmp/` 一个目录。

### 核心变更

```
新增：  skills/pipeline-lite/SKILL.md      （规格 Step 0–8 的操作化落地 + 触发词）
新增：  skills/pipeline-lite/{assets,references,scripts}/  （占位目录）
修改：  .gitignore                         （追加 .harness-lite/ 忽略）
```

## 涉及文件总览

| 文件 | 操作 | 说明 |
|------|------|------|
| `skills/pipeline-lite/SKILL.md` | **新建** | 完整工作流 Step 0–8：触发词、目录规则、分步行为、用户选择界面文案 |
| `skills/pipeline-lite/assets/.gitkeep` | **新建** | 占位目录 |
| `skills/pipeline-lite/references/.gitkeep` | **新建** | 占位目录 |
| `skills/pipeline-lite/scripts/.gitkeep` | **新建** | 占位目录 |
| `.gitignore` | **修改** | 追加 `.harness-lite/`（规格 §4.1 约束 8） |
| `.claude/spec/20260805_pipeline_lite.md` | **已完成** | 实现规格（已由用户创建） |
| `.claude/plan/20260805_pipeline_lite.md` | **本文件** | 执行计划 |

> **不修改**：`skills/pipeline/` 下任何文件（现有重 harness 保持原样）、`README.md`（已含 pipeline-lite 安装说明）、`.claude/settings*`。

## 执行步骤

---

### Step 1：创建 skill 目录骨架

**目标**：建立 `skills/pipeline-lite/` 标准 skill 目录结构。

**涉及文件**：`skills/pipeline-lite/SKILL.md`、`skills/pipeline-lite/assets/.gitkeep`、`skills/pipeline-lite/references/.gitkeep`、`skills/pipeline-lite/scripts/.gitkeep`

**具体操作**：

1. 创建目录 `skills/pipeline-lite/` 及 `assets/`、`references/`、`scripts/` 三个子目录。
2. 每个占位子目录内创建空 `.gitkeep`（沿用 `skills/pipeline/` 的占位惯例）。
3. 创建空的 `SKILL.md`，后续步骤填充。

**验收标准**：`skills/pipeline-lite/` 存在，含 3 个 `.gitkeep` 占位文件与空的 `SKILL.md`。

---

### Step 2：编写 SKILL.md frontmatter 与定位说明

**目标**：定义 skill 的名称、触发词与核心定位，让用户与 Claude Code 能正确识别与触发。

**涉及文件**：`skills/pipeline-lite/SKILL.md`（frontmatter + 核心定位/工作流总览）

**具体修改**：

1. **frontmatter**：
   - `name: pipeline-lite`
   - `description` 需包含触发词与一句话定位，建议：`轻量流程控制 skill。用户输入文本需求后，生成规格文档(spec)、生成执行计划(plan)、按 plan 执行任务，每一步由用户把关，完成后询问是否提交与推送。不做 AI 校验、无子 Agent、无状态文件。触发词：轻量流程、pipeline-lite、简化流程、快速开发流程、lite 流程`
   - `compatibility: type: claude / version: ">=4.0"`（与 `skills/pipeline/SKILL.md` 一致）

2. **核心定位段落**：明确「与 `pipeline`（重 harness）的区别」，写明本 skill 为最小交互骨架：无子 Agent、无状态文件、无 AI 校验、无 allowedPaths，全部动作由主会话完成。

3. **工作流总览**（用 ASCII 图表达 Step 0–8 线性流程，对齐规格 §3）。

**验收标准**：
- frontmatter 含 `pipeline-lite` 名称与触发词
- 定位段落明确「自包含、不依赖重 harness」

---

### Step 3：编写目录规则（运行时目录 `.harness-lite/tmp/`）

**目标**：将规格 §4 落为 SKILL.md 的目录规则章节。

**涉及文件**：`skills/pipeline-lite/SKILL.md`（目录规则章节）

**具体修改**：

1. 说明运行时目录为仓库根目录 `.harness-lite/tmp/`，仅存放两个文件：
   - `.harness-lite/tmp/spec.md`（Step 2 生成，重生成时覆盖）
   - `.harness-lite/tmp/plan.md`（Step 4 生成，重生成时覆盖）

2. 写入三条规则（对齐规格 §4.1）：
   - **每次调用先清空**该目录全部内容（目录不存在则先创建），保证无残留。
   - 该目录为纯运行时状态，**不得进入 git 提交**（配合 `.gitignore` 的 `.harness-lite/`）。
   - 主会话实时读写/覆盖这两个文件，**不启动子 Agent 报告往返**。

3. 说明 skill 源文件目录结构 `skills/pipeline-lite/{SKILL.md, assets/, references/, scripts/}`。

**验收标准**：目录规则章节与规格 §4 逐条对应，运行时目录仅为 `.harness-lite/tmp/`。

---

### Step 4：编写 Step 0 初始化与 Step 1 需求上下文

**目标**：落地面向用户的「开始前清场」与「需求理解」行为。

**涉及文件**：`skills/pipeline-lite/SKILL.md`（Step 0 / Step 1 章节）

**具体修改**：

1. **Step 0 初始化**（对齐规格 §5.1）：
   - 若 `.harness-lite/tmp/` 不存在则创建。
   - 删除该目录下全部内容（`spec.md`、`plan.md` 及任何残留）。
   - 静默完成，无需向用户展示状态。

2. **Step 1 需求与上下文**（对齐规格 §5.2）：
   - 读取用户本次输入的需求，结合当前会话/项目上下文理解意图。
   - **不做额外澄清提问**，直接进入 spec 生成；信息不足由用户在 spec 门禁处通过「补充信息修改」弥补。

**验收标准**：Step 0/1 描述与规格 §5.1/§5.2 一致，体现「静默清空 + 不追问」的轻量姿态。

---

### Step 5：编写 Step 2 生成 spec.md 与 spec 模板

**目标**：落地「父 Agent 亲自撰写 spec」的行为与精简结构模板。

**涉及文件**：`skills/pipeline-lite/SKILL.md`（Step 2 章节）

**具体修改**：

1. 说明由**主会话亲自撰写** `spec.md` 并写入 `.harness-lite/tmp/spec.md`，不启动 AI 校验、不生成建议文件、不维护状态。

2. 内嵌 spec 精简结构模板（对齐规格 §5.3）：
   ```
   # {需求标题}

   ## 背景与目标
   ## 功能需求
   ## 非功能需求        （无则注明"无"）
   ## 技术方案（简要）   （一段话）
   ## 文件结构（简要）   （无则注明"待 plan 细化"）
   ```

**验收标准**：Step 2 明确「主会话亲自撰写 + 写入 tmp/spec.md」，模板五节与规格 §5.3 一致。

---

### Step 6：编写 Step 3 spec 门禁（用户选择界面）

**目标**：落地 spec 生成后的用户把关界面与三选项行为。

**涉及文件**：`skills/pipeline-lite/SKILL.md`（Step 3 章节）

**具体修改**：

1. 展示 spec 概要（主会话直接总结）并给出三选项，文案严格对齐规格 §5.4 / §6：
   ```
   spec 已生成：.harness-lite/tmp/spec.md

   ① 下一步        → 进入 Step 4 生成 plan
   ② 重新生成      → 回到 Step 2，重新撰写 spec.md（覆盖旧文件）
   ③ 补充信息修改  → 用户补充/修正需求 → 回到 Step 2 带补充内容重新撰写
   ```

2. 写明各选项行为（对齐规格 §5.4）：
   - 选 ②：删除或直接覆盖当前 `spec.md`，重新执行 Step 2，回到本步。
   - 选 ③：等待用户输入补充信息，清除后带补充内容重新执行 Step 2，回到本步。
   - 选 ①：进入 Step 4。

**验收标准**：界面编号与文案与规格 §6 表格一致；三选项行为闭环（②/③ 均回到 Step 2）。

---

### Step 7：编写 Step 4 生成 plan.md 与 plan 模板

**目标**：落地「父 Agent 读取已确认 spec、亲自拆解 plan」的行为与模板。

**涉及文件**：`skills/pipeline-lite/SKILL.md`（Step 4 章节）

**具体修改**：

1. 说明主会话读取已确认的 `spec.md`，亲自拆解为可独立完成/可独立提交的步骤，写入 `.harness-lite/tmp/plan.md`。

2. 内嵌 plan 模板（对齐规格 §5.5）：
   ```
   # {需求标题} — 执行计划

   ## 步骤 1：{步骤名称}
   - 目标：（本步达成什么）
   - 涉及文件：（预计改动/新增的文件）
   - 验收要点：（如何判断本步完成）

   ## 步骤 2：{步骤名称}
   ...
   ```

3. 轻量约定：每步可独立完成即可，不汇总 `allowedPaths`，不启动 plan-reviewer。

**验收标准**：Step 4 明确「父 Agent 亲自拆解 + 写入 tmp/plan.md」，模板含目标/涉及文件/验收要点三要素。

---

### Step 8：编写 Step 5 plan 门禁（用户选择界面）

**目标**：落地 plan 生成后的用户把关界面与三选项行为。

**涉及文件**：`skills/pipeline-lite/SKILL.md`（Step 5 章节）

**具体修改**：

1. 展示 plan 概要并给出三选项，文案严格对齐规格 §5.6 / §6：
   ```
   plan 已生成：.harness-lite/tmp/plan.md

   ① 下一步开始执行 → 进入 Step 6
   ② 重新生成       → 回到 Step 4，重新撰写 plan.md
   ③ 补充信息修改   → 用户补充信息 → 回到 Step 4 带补充内容重新撰写
   ```

2. 写明各选项行为（对齐规格 §5.6）：
   - 选 ②：删除或直接覆盖当前 `plan.md`，重新执行 Step 4，回到本步。
   - 选 ③：等待用户补充信息，带补充内容重新执行 Step 4，回到本步。
   - 选 ①：进入 Step 6。

**验收标准**：界面编号与文案与规格 §6 表格一致；三选项行为闭环（②/③ 均回到 Step 4）。

---

### Step 9：编写 Step 6 执行

**目标**：落地「按 plan 逐步执行、失败停下、不自行 git commit」的执行行为。

**涉及文件**：`skills/pipeline-lite/SKILL.md`（Step 6 章节）

**具体修改**（对齐规格 §5.7）：

1. 按 `plan.md` 从步骤 1 依次执行。
2. 每步完成后可运行必要验证/测试（轻量原则：不强制分层验证）。
3. 某步执行失败：停下来向用户说明失败原因与已做改动，由用户决定继续/修正。
4. **关键约束：执行全程不自行 `git commit`**，所有改动保留在工作区，交由 Step 7 由用户决定提交方式。
5. 全部步骤完成后，向用户简要汇报执行结果（完成哪些、改动哪些文件、测试结果）。

**验收标准**：Step 6 明确「不自行 commit」与「失败停下询问」两条关键行为。

---

### Step 10：编写 Step 7 提交询问 与 Step 8 推送询问

**目标**：落地执行完成后的提交/推送双门禁。

**涉及文件**：`skills/pipeline-lite/SKILL.md`（Step 7 / Step 8 章节）

**具体修改**：

1. **Step 7 提交询问**（对齐规格 §5.8 / §6）：
   ```
   执行完成。请选择提交方式：

   ① 提交 - 自动生成提交信息 → 主会话根据本次改动自动撰写 commit message，执行 git add + commit
   ② 提交 - 手动输入提交信息 → 用户提供 commit message，主会话执行 git add + commit
   ③ 不提交                 → 保留工作区改动，流程结束
   ```
   - 选 ①/②：执行 `git add` + `git commit`（手动输入时使用用户提供的 message），随后进入 Step 8。
   - 选 ③：不执行任何 git 操作，流程结束。

2. **Step 8 推送询问**（对齐规格 §5.9 / §6）：
   ```
   是否推送远程服务？

   ① 是 → 执行 git push
   ② 否 → 结束流程
   ```
   - 仅在 Step 7 选择「提交」后触发。
   - 选 ①：执行 `git push`，结束后汇报推送结果；选 ②：流程结束。

**验收标准**：提交三选项、推送两选项的编号与文案与规格 §6 一致；推送仅在提交后触发。

---

### Step 11：更新 `.gitignore` 忽略 `.harness-lite/`

**目标**：保证运行时目录不进入版本库（规格 §4.1 规则 2 / 约束 8）。

**涉及文件**：`.gitignore`

**具体修改**：

- 在现有 `.harness/temp/` 忽略项附近追加一行：`.harness-lite/`

**验收标准**：`git status` 中 `.harness-lite/` 目录不再被跟踪（若有残留则确认已忽略）。

---

### Step 12：全局验证

**目标**：确认新 skill 完整、自包含、与规格逐条对应，无对现有 harness 的引用残留。

**涉及文件**：`skills/pipeline-lite/SKILL.md`、`.gitignore`

**验证项**：

1. **自包含检查**：全文搜索 `spec-create-harness`、`pipeline/`、`.harness/`、`state.json`、`allowedPaths`、`子 Agent`、`reviewer` 等字样，确认 SKILL.md 中无对现有重 harness 的复用/拷贝引用（仅可用于对比说明，不得作为实现依据）。
2. **目录检查**：运行时目录唯一指向 `.harness-lite/tmp/`，且 `.gitignore` 已忽略 `.harness-lite/`。
3. **流程闭环**：Step 0 → Step 8 线性可执行；所有门禁（spec / plan / 提交 / 推送）均有用户显式选择。
4. **界面文案一致性**：Step 3、Step 5、Step 7、Step 8 的选项编号与文案与规格 §6 汇总表逐字一致。
5. **不越权检查**：执行阶段（Step 6）无 `git commit` 指令；`git add`/`git commit` 仅出现在 Step 7 提交分支；`git push` 仅出现在 Step 8 且依赖 Step 7 选择了提交。
6. **触发词可用**：frontmatter description 中的触发词覆盖「轻量流程」类意图。
7. **现有 harness 未受影响**：`skills/pipeline/` 目录内容无改动，重 harness 功能保持原样。

**提交信息**：`feat: 新增 pipeline-lite 轻量流程控制 skill`（或按项目规范），验证通过后提交。

---

## 注意事项

### 1. 自包含，不参考现有代码

本 skill 按规格文档独立实现，**不要打开/复制 `skills/pipeline/SKILL.md`、references、scripts 中的实现细节**。规格 §3 工作流总览、§4 目录规则、§5 分步流程、§6 界面汇总已是完整操作化描述，SKILL.md 直接据此落地。目录结构与占位惯例（`assets/references/scripts` + `.gitkeep`）可参考现有 skill 的形式，但内容一律按本规格写。

### 2. 无状态、无子 Agent、无 AI 校验

SKILL.md 中不得出现：`state.json`、`status` 状态机、`allowedPaths`、`spec-reviewer` / `plan-reviewer` / `execution` 子 Agent、`*-suggest.md` 建议文件、AI 校验循环。所有「校验」都替换为**用户确认**。

### 3. 界面文案是验收硬指标

Step 3 / Step 5 / Step 7 / Step 8 的四个选择界面，编号（①②③）与文案必须与规格 §6 汇总表**逐字一致**，这是规格 §9 验收标准 3/5/7/8 的直接依据。实现时以 §6 表格为准，避免在 SKILL.md 中自由发挥。

### 4. git 操作边界

- `git commit`：只允许出现在 Step 7「提交」分支（①自动生成 / ②手动输入），Step 6 执行阶段**禁止**。
- `git push`：只允许出现在 Step 8，且仅在 Step 7 选择提交后触发。
- 规格约束 5/6 是两条不可逾越的边界，全局验证时应逐段核对。

### 5. 运行时目录位置

`.harness-lite/tmp/` 位于**仓库根目录**（不是 skill 目录内），与现有 `.harness/temp/` 同级。所有对临时目录的引用都要用完整相对路径，避免歧义。

### 6. README 无需修改

`README.md` 安装说明已含 `--skill pipeline-lite` 一行，无需改动。若后续 README 增加了技能清单表格，再补充本 skill 条目（本次不做）。

### 7. 与重 harness 的关系

`skills/pipeline/` 现有 skill 保持不动，本次为纯新增。未来如规格要求对齐（如共享界面风格），另行计划，不在本次范围。
