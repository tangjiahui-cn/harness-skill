# Plan: Agent 职责重构 — 父 Agent 生成 spec/plan，子 Agent 仅负责 Review

## 概述

基于 `.claude/spec/20260805_agent_refactor.md` 规格文档，重构 `skills/spec-create-harness/SKILL.md` 的 Agent 架构：**spec/plan 的生成与修补职责从子 Agent（`spec-generator` / `plan-generator`）回归父 Agent**，子 Agent 仅保留 `spec-reviewer` / `plan-reviewer` 的独立审查职责，父 Agent 读取建议文件后**亲自修复产物并删除建议文件**。

### 核心变更

```
修改前： 父 Agent ──启动──▶ generator(生成/修补) ⇄ reviewer(审查)，父 Agent 仅编排
修改后：  父 Agent 亲自生成/修补 spec、plan，仅在校验时启动 reviewer 子 Agent 审查
```

### 角色变化

| 角色 | 变更 |
|------|------|
| `spec-generator` / `plan-generator` 子 Agent | **取消**（生成与修补职责并入父 Agent） |
| `spec-reviewer` / `plan-reviewer` 子 Agent | **保留**，职责不变（读产物 → 写 `*-suggest.md` → 报告） |
| `execution` 子 Agent | **保留**，职责不变 |
| 父 Agent | **扩权**：新增"生成 spec / 生成 plan / 亲自修补产物 / 删除建议文件" |

## 涉及文件总览

| 文件 | 操作 | 说明 |
|------|------|------|
| `skills/spec-create-harness/SKILL.md` | **修改** | 重写 §3.1/§3.3 与 §4.1/§4.3；更新交互要点、各选项行为、§4.5 措辞、中断恢复逻辑 |
| `.claude/spec/20260805_agent_refactor.md` | **已完成** | 重构规格文档（已由用户创建） |
| `.claude/plan/20260805_agent_refactor.md` | **本文件** | 执行计划 |

> **不修改**：`references/review-spec.md`、`references/review-plan.md`、`references/review-exec.md`、`.claude/spec/review.md`（框架文档同步另行安排）、状态机取值、Step 5/6 执行与推送逻辑。

## 执行步骤

---

### Step 1：更新交互要点（Agent 命名 + 子 Agent 协作）

**目标**：同步 §交互要点 中关于 Agent 角色与协作方式的描述，反映"父 Agent 生成、子 Agent 仅审查"。

**涉及文件**：`skills/spec-create-harness/SKILL.md`（§交互要点，约 L153-154）

**具体修改**：

1. **Agent 命名**（L153）：
   - 修改前：`使用 \`spec-generator\` / \`spec-reviewer\` / \`plan-generator\` / \`plan-reviewer\` / \`execution\` 区分，每次循环启动新实例`
   - 修改后：`子 Agent 使用 \`spec-reviewer\` / \`plan-reviewer\` / \`execution\` 区分，每次循环启动新实例；spec/plan 的生成与修补由父 Agent 直接完成`

2. **子 Agent 协作**（L154）：
   - 修改前：`spec 和 plan 的生成均交给独立子 Agent 处理，父 Agent 只负责启动子 Agent、接收报告、与用户交互确认。……然后驱动下一轮生成或退出。子 Agent 之间不直接交互……`
   - 修改后：`spec 和 plan 的生成与修补均由父 Agent 完成，子 Agent 仅承担独立审查（spec-reviewer / plan-reviewer）与执行（execution）。父 Agent 负责生成产物、启动子 Agent、接收报告、与用户交互确认。子 Agent 与父 Agent 通过校验建议文件（spec-suggest.md / plan-suggest.md）协作——reviewer 输出结构化建议，父 Agent 读取并判断是否有实质性问题，亲自将建议修复到 spec/plan 后删除建议文件。子 Agent 之间不直接交互，所有通信通过父 Agent 协调`

**验收标准**：
- 命名列表不再包含 `spec-generator` / `plan-generator`
- 协作描述明确"父 Agent 生成与修补 + 子 Agent 仅审查"

---

### Step 2：重写 Step 3 概述段与 §3.1（父 Agent 生成 spec）

**目标**：将 spec 生成阶段从"两个独立子 Agent 协作"改为"父 Agent 亲自生成 + 可选 spec-reviewer 审查"。

**涉及文件**：`skills/spec-create-harness/SKILL.md`（Step 3 概述段 L183-188、§3.1 L190-213）

**具体修改**：

1. **Step 3 概述段**（L183-188）：
   - 修改前：`spec 生成阶段由**两个独立子 Agent** 协作完成（注意：每次循环都启动新的 Agent 实例，角色名仅描述职责）：` + 两条 generator/reviewer 职责 + `spec-generator 完成初始生成后……`
   - 修改后：
     ```
     spec 生成由**父 Agent 直接完成**，可选配合一个独立子 Agent 进行 AI 校验：

     - **父 Agent**：负责生成 spec.md，以及根据校验建议修补 spec.md
     - **spec-reviewer Agent**：负责校验 spec.md，生成 `spec-suggest.md`（校验建议文件）

     父 Agent 完成初始生成后，向用户展示生成结果，由用户决定是否进入 AI 校验循环。如果选择 AI 校验，则驱动"生成 → 校验 → 修补 → 再校验"的循环（修补由父 Agent 完成）；如果直接确认，则跳过校验进入下一阶段。
     ```

2. **§3.1 标题**：`#### 3.1 初始生成（spec-generator Agent）` → `#### 3.1 初始生成（父 Agent）`

3. **§3.1 正文**（L192-213）：
   - 修改前：`父 Agent 启动 **spec-generator Agent**，分配以下任务：` + `**spec-generator Agent 的职责：**` + 4 条职责（含"向父 Agent 报告完成"）+ 报告格式块
   - 修改后：`父 Agent 亲自完成 spec 的初始生成，不再启动 spec-generator 子 Agent。` + `**父 Agent 的生成职责：**` + 4 条职责（最后一条改为"向用户展示生成结果"）+ **删除"报告格式"块**（与 §3.2 展示界面重复，且父 Agent 不再需要向谁报告）

**验收标准**：
- §3.1 无 `spec-generator` 字样
- 职责清单完整保留（背景目标/功能规格/非功能规格/技术方案/文件结构）
- 报告格式块已移除

---

### Step 3：更新 §3.2 生成后的用户选择（spec）

**目标**：将 §3.2 开头与"重新生成/补充信息"选项中的 generator 措辞改为父 Agent。

**涉及文件**：`skills/spec-create-harness/SKILL.md`（§3.2，L215-242）

**具体修改**：

1. **§3.2 开头**（L217）：
   - 修改前：`spec-generator Agent 报告完成后，更新 \`state.json\`（\`status: "spec_pending"\`），父 Agent 向用户展示 spec 关键内容，并提供选择：`
   - 修改后：`父 Agent 完成 spec.md 生成后，更新 \`state.json\`（\`status: "spec_pending"\`），向用户展示 spec 关键内容，并提供选择：`

2. **各选项行为表**（L240-241）：
   - **选项 3 重新生成**：`删除当前 spec.md，重新进入 §3.1（启动 spec-generator Agent 重新生成）` → `删除当前 spec.md，重新进入 §3.1（父 Agent 重新生成）`
   - **选项 4 补充信息**：`……用户提供后清除 \`waitingFor\`，重新启动 spec-generator Agent（带上补充信息作为额外输入）` → `……用户提供后清除 \`waitingFor\`，父 Agent 重新生成 spec.md（带上补充信息作为额外输入）`

**验收标准**：
- §3.2 无 `spec-generator` 字样
- 展示界面、5 个选项编号与文案不变

---

### Step 4：重写 §3.3 AI 校验循环（spec）

**目标**：校验循环中"修补"环节从"启动 spec-generator 修补模式"改为"父 Agent 亲自修补 + 删除建议文件"。

**涉及文件**：`skills/spec-create-harness/SKILL.md`（§3.3，L244-308）

**具体修改**：

1. **§3.3 开头**（L246）：
   - 修改前：`当用户在 §3.2 选择"AI 校验"后，父 Agent 启动校验循环，驱动 **spec-reviewer Agent** 和 **spec-generator Agent** 交替工作：`
   - 修改后：`当用户在 §3.2 选择"AI 校验"后，父 Agent 启动校验循环，驱动 **spec-reviewer Agent** 审查、由 **父 Agent 亲自修补**：`

2. **循环伪代码**（L249-262）：
   - 修改前第 6-9 步：
     ```
     6. 父 Agent 启动 spec-generator Agent（修补模式），传递 spec-suggest.md 内容
     7. spec-generator Agent 读取 spec-suggest.md，逐条采纳建议并修改 spec.md
     8. spec-generator Agent 向父 Agent 报告修补完成
     9. 父 Agent 删除 spec-suggest.md
     10. 回到第 1 步进行下一轮校验
     ```
   - 修改后：
     ```
     6. 父 Agent 亲自修补：将 spec-suggest.md 中的建议逐条落实到 spec.md
     7. 父 Agent 删除 spec-suggest.md
     8. 回到第 1 步进行下一轮校验
     ```

3. **各角色职责表**（L264-270）：
   - 删除行：`| **spec-generator Agent（修补模式）** | 读取 \`spec-suggest.md\`，逐条评估并采纳合理建议，修改 \`spec.md\`。不质疑 reviewer 的发现，专注修补 |`
   - 更新父 Agent 行：`| **父 Agent** | 读取 \`spec-suggest.md\` 判断是否有实质性问题；驱动循环流程；管理临时文件 |` → `| **父 Agent** | 生成 spec.md；读取 \`spec-suggest.md\` 判断是否有实质性问题；亲自将建议修复到 spec.md；删除建议文件；驱动循环流程 |`

4. **校验完成后部分**（L287-308）**保持不变**（选项界面、P0/P1/P2 统计等不动）。

**验收标准**：
- §3.3 无 `spec-generator` 字样
- 5 轮上限、审查标准引用（review-spec.md）、"第一次审查"心态等说明保留
- 父 Agent 行职责含"亲自修补 + 删除建议文件"

---

### Step 5：重写 Step 4 概述段与 §4.1（父 Agent 生成 plan）

**目标**：将 plan 生成阶段改为"父 Agent 亲自生成 + 可选 plan-reviewer 审查"，与 Step 3 对称。

**涉及文件**：`skills/spec-create-harness/SKILL.md`（Step 4 概述段 L326-331、§4.1 L333-357）

**具体修改**：

1. **Step 4 概述段**（L326-331）：
   - 修改前：`plan 生成阶段与 spec 生成阶段采用相同的**双 Agent 协作模式**（同样每次启动新的 Agent 实例）：` + generator/reviewer 两条职责 + `plan-generator 完成初始生成后……`
   - 修改后：
     ```
     plan 生成阶段与 spec 生成阶段采用相同的**父 Agent 生成 + 子 Agent 审查**模式：

     - **父 Agent**：负责生成 plan.md，以及根据校验建议修补 plan.md
     - **plan-reviewer Agent**：负责校验 plan.md，生成 `plan-suggest.md`（校验建议文件）

     父 Agent 完成初始生成后，向用户展示生成结果，由用户决定是否进入 AI 校验循环。如果选择 AI 校验，则驱动"生成 → 校验 → 修补 → 再校验"的循环（修补由父 Agent 完成）；如果直接确认，则跳过校验进入执行阶段。
     ```

2. **§4.1 标题**：`#### 4.1 初始生成（plan-generator Agent）` → `#### 4.1 初始生成（父 Agent）`

3. **§4.1 正文**（L335-357）：
   - 修改前：`父 Agent 启动 **plan-generator Agent**，分配以下任务：` + `**plan-generator Agent 的职责：**` + 5 条职责（含"向父 Agent 报告完成"）+ 报告格式块
   - 修改后：`父 Agent 亲自完成 plan 的初始生成，不再启动 plan-generator 子 Agent。` + `**父 Agent 的生成职责：**` + 5 条职责（最后一条改为"向用户展示生成结果"）+ **删除"报告格式"块**

**验收标准**：
- §4.1 无 `plan-generator` 字样
- 职责清单完整保留（步骤拆解/目标/涉及文件/验收标准/allowedPaths）
- 报告格式块已移除

---

### Step 6：更新 §4.2 生成后的用户选择（plan）

**目标**：将 §4.2 开头与"重新生成/补充信息"选项中的 generator 措辞改为父 Agent。

**涉及文件**：`skills/spec-create-harness/SKILL.md`（§4.2，L359-387）

**具体修改**：

1. **§4.2 开头**（L361）：
   - 修改前：`plan-generator Agent 报告完成后，更新 \`state.json\`（\`status: "plan_pending"\`），父 Agent 向用户展示 plan 关键内容，并提供选择：`
   - 修改后：`父 Agent 完成 plan.md 生成后，更新 \`state.json\`（\`status: "plan_pending"\`），向用户展示 plan 关键内容，并提供选择：`

2. **各选项行为表**（L385-386）：
   - **选项 3 重新生成**：`删除当前 plan.md，重新进入 §4.1（启动 plan-generator Agent 重新生成）` → `删除当前 plan.md，重新进入 §4.1（父 Agent 重新生成）`
   - **选项 4 补充信息**：`……否则直接重新启动 plan-generator Agent（带上补充信息），回到本界面` → `……否则父 Agent 直接重新生成 plan.md（带上补充信息），回到本界面`

**验收标准**：
- §4.2 无 `plan-generator` 字样
- 展示界面、5 个选项编号与文案不变

---

### Step 7：重写 §4.3 AI 校验循环（plan）

**目标**：与 Step 4 对称，将 plan 校验循环的"修补"环节改为父 Agent 亲自修补 + 删除建议文件。

**涉及文件**：`skills/spec-create-harness/SKILL.md`（§4.3，L389-453）

**具体修改**：

1. **§4.3 开头**（L391）：
   - 修改前：`当用户在 §4.2 选择"AI 校验"后，父 Agent 启动校验循环，驱动 **plan-reviewer Agent** 和 **plan-generator Agent** 交替工作：`
   - 修改后：`当用户在 §4.2 选择"AI 校验"后，父 Agent 启动校验循环，驱动 **plan-reviewer Agent** 审查、由 **父 Agent 亲自修补**：`

2. **循环伪代码**（L394-407）：
   - 修改前第 6-9 步：
     ```
     6. 父 Agent 启动 plan-generator Agent（修补模式），传递 plan-suggest.md 内容
     7. plan-generator Agent 读取 plan-suggest.md，逐条采纳建议并修改 plan.md
     8. plan-generator Agent 向父 Agent 报告修补完成
     9. 父 Agent 删除 plan-suggest.md
     10. 回到第 1 步进行下一轮校验
     ```
   - 修改后：
     ```
     6. 父 Agent 亲自修补：将 plan-suggest.md 中的建议逐条落实到 plan.md
     7. 父 Agent 删除 plan-suggest.md
     8. 回到第 1 步进行下一轮校验
     ```

3. **各角色职责表**（L409-415）：
   - 删除行：`| **plan-generator Agent（修补模式）** | 读取 \`plan-suggest.md\`，逐条评估并采纳合理建议，修改 \`plan.md\` |`
   - 更新父 Agent 行：`| **父 Agent** | 读取 \`plan-suggest.md\` 判断是否有实质性问题；驱动循环流程；管理临时文件 |` → `| **父 Agent** | 生成 plan.md；读取 \`plan-suggest.md\` 判断是否有实质性问题；亲自将建议修复到 plan.md；删除建议文件；驱动循环流程 |`

4. **校验完成后部分**（L432-453）**保持不变**。

**验收标准**：
- §4.3 无 `plan-generator` 字样
- 5 轮上限、审查标准引用（review-plan.md）、"第一次审查"心态等说明保留

---

### Step 8：更新 §4.5 allowedPaths 措辞

**目标**：allowedPaths 数据来源从"plan-generator 报告"改为"父 Agent 生成 plan 时汇总"。

**涉及文件**：`skills/spec-create-harness/SKILL.md`（§4.5，L463）

**具体修改**：

- 修改前：`1. **写入 allowedPaths**：将 plan-generator Agent（最新修补后）报告的 \`allowedPaths\` 写入 \`state.json\`：`
- 修改后：`1. **写入 allowedPaths**：将父 Agent 生成 plan 时汇总的 \`allowedPaths\` 写入 \`state.json\`：`

**验收标准**：§4.5 无 `plan-generator` 字样。

---

### Step 9：更新中断恢复机制

**目标**：`spec_reviewing` / `plan_reviewing` 状态恢复时的动作从"启动 generator 修补模式"改为"父 Agent 读取建议文件亲自修补"。

**涉及文件**：`skills/spec-create-harness/SKILL.md`（§中断恢复机制，约 L726-733）

**具体修改**：

- `spec_reviewing` 分支（L726-728）：
  - 修改前：`是 → 从 **§3.3 AI 校验循环**继续（启动 spec-generator 修补模式）`
  - 修改后：`是 → 从 **§3.3 AI 校验循环**继续（父 Agent 读取 spec-suggest.md 亲自修补 spec.md）`
- `plan_reviewing` 分支（L731-733）：
  - 修改前：`是 → 从 **§4.3 AI 校验循环**继续（启动 plan-generator 修补模式）`
  - 修改后：`是 → 从 **§4.3 AI 校验循环**继续（父 Agent 读取 plan-suggest.md 亲自修补 plan.md）`

**验收标准**：
- 中断恢复逻辑中无 `spec-generator` / `plan-generator` 字样
- 判断逻辑（suggest 文件是否存在）与恢复路径不变

---

### Step 10：全局验证

**目标**：确认所有修改正确、完整、一致，无 generator 残留。

**涉及文件**：`skills/spec-create-harness/SKILL.md`

**验证项**：

1. **残留搜索**：全文搜索 `spec-generator`、`plan-generator`，确认除"不再启动 xxx-generator 子 Agent"这类否定描述外无残留
2. **流程验证**：`父 Agent 生成 spec → 展示 → [可选 AI 校验] → 确认 → 父 Agent 生成 plan → 展示 → [可选 AI 校验] → 确认 → execution 执行` 的流程可读、可执行
3. **对称性检查**：Spec 侧与 Plan 侧修改对称（概述段、§x.1、§x.2 选项、§x.3 循环、角色表、中断恢复）
4. **AI 校验循环完整性**：§3.3 与 §4.3 保留完整循环（reviewer 启动、suggest 写入、父判断、父修补、删文件、5 轮上限、审查标准引用）
5. **引用不变**：`references/review-spec.md` 与 `references/review-plan.md` 的引用方式不变
6. **状态机不变**：`status` 流转、`step` / `currentStep` / `waitingFor` / `allowedPaths` 语义不变
7. **Step 5/6 不受影响**：execution 启动、allowedPaths 校验、分层验证、提交、验收归档、推送逻辑未改动

**提交信息**：`refactor: 父 Agent 亲自生成并修补 spec/plan，子 Agent 仅负责审查`（或按项目规范），验证通过后提交。

---

## 注意事项

### 1. 不要修改 AI 校验循环的"审查"逻辑本身

本 plan 只改变**修补方**（从 generator 子 Agent 改为父 Agent）与**建议文件删除时机**（每轮修补后立即删除），不改变 reviewer 的审查标准、P0/P1/P2 分类、5 轮上限、"第一次审查"心态等。§3.3/§4.3 的循环伪代码只动第 6-9 步，其余保留。

### 2. 只删措辞、不删功能

删除 generator 角色时，其**职责清单**（spec 的背景目标/功能/非功能/技术方案/文件结构；plan 的步骤拆解/目标/涉及文件/验收标准/allowedPaths 汇总）必须原样迁移到父 Agent 职责中，不能连同删除。

### 3. 报告格式块的处理

原 §3.1/§4.1 的"报告格式"块与 §3.2/§4.2 的展示界面内容重复（父 Agent 生成后直接展示，不再有"子 Agent 报告"环节）。**删除 §x.1 的报告格式块**，保留 §x.2 的展示界面作为用户可见文本。

### 4. references 目录描述（可选一致性清理）

L45 / L84 的"参考文档，供子 Agent 在生成 spec/plan 时可按需加载"描述中，"子 Agent 生成"已不准确（生成者是父 Agent）。可在本 plan 执行时一并调整为"供父 Agent 生成与子 Agent 审查时按需加载"，属低风险措辞清理，不阻塞验收。

### 5. 框架文档同步（另行安排）

`.claude/spec/review.md` §3.2 角色表、§7.1"Generator 修补模式行为"、§7.2"子 Agent 隔离原则"中关于 generator 子 Agent 的描述，按规格文档 §7 要求需同步更新，但**不在本次 SKILL.md 修改范围内**，建议后续单独提交。

### 6. 用户界面与状态机不动

所有用户选择界面（生成后/校验后）、选项编号、`status` 流转取值均**保持不变**——本次重构是纯内部职责迁移，用户可感知的交互无变化。
