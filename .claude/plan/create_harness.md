# Plan: 创建 Spec Create Harness Skill

## 概述

基于 `.claude/spec/create_harness.md` 规格文档，在 `skills/spec-create-harness/` 目录下创建完整的软件开发工作流 skill。

## 涉及文件总览

| 文件 | 操作 | 说明 |
|------|------|------|
| `skills/spec-create-harness/SKILL.md` | 新建 | Skill 核心入口，完整工作流定义 |
| `skills/spec-create-harness/references/.gitkeep` | 新建 | 参考文档目录占位 |
| `skills/spec-create-harness/scripts/.gitkeep` | 新建 | 辅助脚本目录占位 |
| `skills/spec-create-harness/assets/.gitkeep` | 新建 | 静态资源目录占位 |
| `.gitignore` | 修改 | 添加 `.harness/temp/` 忽略规则 |
| `package.json` | 修改 | 如有必要，添加测试脚本 |

## 执行步骤

---

### Step 1：创建 skill 目录结构

**目标**：创建 `skills/spec-create-harness/` 及其子目录。

**涉及文件**：
- `skills/spec-create-harness/references/.gitkeep`
- `skills/spec-create-harness/scripts/.gitkeep`
- `skills/spec-create-harness/assets/.gitkeep`

**验收标准**：目录结构完整，空目录通过 `.gitkeep` 占位。

**命令**：
```bash
mkdir -p skills/spec-create-harness/{references,scripts,assets}
touch skills/spec-create-harness/references/.gitkeep
touch skills/spec-create-harness/scripts/.gitkeep
touch skills/spec-create-harness/assets/.gitkeep
```

---

### Step 2：编写 SKILL.md —— 元信息与核心框架

**目标**：创建 SKILL.md 文件头、核心理念、目录结构说明、初始化信息生成、命名规则、状态流转。

**涉及文件**：`skills/spec-create-harness/SKILL.md`

**内容要点**：
1. **Frontmatter**：`name: spec-create-harness`，`description`（含触发词），`compatibility`
2. **核心理念**：先理解清楚需求再写代码，不要跳到实现
3. **Skill 目录结构**：技能目录树 + 各文件/目录职责表
4. **初始化信息生成**：`name` / `shortId` / `vId` 的生成规则
5. **state.json 结构**：完整 JSON 字段定义
6. **状态流转**：`initialized → spec_reviewing → spec_confirming → plan_reviewing → plan_confirming → executing → acceptance → accepted → completed`
7. **命名规则**：运行时文件名固定，不携带 name/shortId 前缀

**验收标准**：SKILL.md 包含完整的元信息和框架定义，Frontmatter 格式正确。

**提交信息**：`feat: 添加 spec-create-harness skill 核心框架`

---

### Step 3：编写 SKILL.md —— Step 1~2（需求输入与初始化）

**目标**：在工作流阶段中补充 需求输入(Step 1) 和 初始化(Step 2) 的完整描述。

**涉及文件**：`skills/spec-create-harness/SKILL.md`

**内容要点**：
1. **Step 1 需求输入**：
   - 复述需求 → 澄清模糊点 → 确认范围
   - 确认完成后进入 Step 2
2. **Step 2 初始化**：
   - 创建 `.harness/temp/` 和 `.harness/history/`
   - 生成 vId → 创建目录 → 写入 state.json
   - 告知用户初始化完成，显示 vId

**验收标准**：SKILL.md 文档结构完整，Step 1~2 可读、可执行。

**提交信息**：`feat: 添加需求输入和初始化阶段描述`

---

### Step 4：编写 SKILL.md —— Step 3（Spec 生成 + 双 Agent 校验循环 + 用户确认）

**目标**：实现 spec 生成阶段的完整流程描述，包括双 Agent 协作模式和用户交互选项。

**涉及文件**：`skills/spec-create-harness/SKILL.md`

**内容要点**：
1. **3.1 初始生成**：spec-generator Agent 职责、生成内容要求、报告格式
2. **3.2 AI 校验循环**：父 Agent 编排的循环流程（最多 5 轮）、各角色职责表
3. **3.3 用户确认**：5 个选项（继续/重新生成/补充信息/再次 AI 校验/手动修改）、每种选项的行为、state.json 更新

**验收标准**：双 Agent 协作模式的流程清晰，校验循环有明确的退出条件，用户交互选项完整。

**提交信息**：`feat: 添加 spec 生成和双 Agent 校验循环`

---

### Step 5：编写 SKILL.md —— Step 4（Plan 生成 + 双 Agent 校验循环 + 用户确认 + 文件范围锁定）

**目标**：实现 plan 生成阶段的完整流程描述，结构与 Step 3 对应。

**涉及文件**：`skills/spec-create-harness/SKILL.md`

**内容要点**：
1. **4.1 初始生成**：plan-generator Agent 职责、内容要求、报告格式
2. **4.2 AI 校验循环**：同 spec 校验结构，最多 5 轮
3. **4.3 用户确认**：5 个选项，补充信息中涉及 spec 变更的处理
4. **4.4 写入状态与文件范围**：allowedPaths 写入 state.json 的格式

**验收标准**：Plan 生成流程完整，allowedPaths 机制清晰，与 Step 3 结构对称。

**提交信息**：`feat: 添加 plan 生成和文件范围锁定机制`

---

### Step 6：编写 SKILL.md —— Step 5（执行与测试）

**目标**：实现执行阶段的完整流程描述，包括 Execution Agent 的详细执行循环。

**涉及文件**：`skills/spec-create-harness/SKILL.md`

**内容要点**：
1. **5.1 启动 Execution Agent**：分支检查、传递给 Agent 的信息
2. **Execution Agent 每步循环**：
   - 预先声明 → 预先校验(allowedPaths) → 实现 → 实现后校验 → 分层验证(编译/lint) → 运行测试 → git commit → 更新 state.json
   - 提交格式：Conventional Commits
   - 工作区干净检查 + 处理规则
   - 文件越界自动豁免规则（.gitignore、.harness/）
3. **5.2 父 Agent 处理结果**：
   - **情况 A**：全部通过 → 用户验收选项（通过/补充信息）
     - 验收通过 → 归档操作
     - 补充信息 → 回退逻辑（`git reset --soft`）
   - **情况 B**：出错暂停 → 3 个选项（重新执行该步骤/全部重来/补充信息）
     - 已提交 vs 未提交的不同处理
     - 3 次重试上限建议

**验收标准**：执行流程完整，每步校验点清晰，错误处理覆盖全面。

**提交信息**：`feat: 添加执行与测试阶段描述`

---

### Step 7：编写 SKILL.md —— Step 6 + 状态更新表 + 中断恢复

**目标**：完成验收推送阶段、状态文件更新时机表、中断恢复机制。

**涉及文件**：`skills/spec-create-harness/SKILL.md`

**内容要点**：
1. **Step 6 验收后询问推送**：
   - 推送/不处理 2 个选项
   - 推送失败的分支处理（无远程/权限/非快进冲突）
   - 功能分支上 force push 的安全建议
2. **状态文件更新时机表**：完整表格（状态/step/currentStep/位置）
3. **中断恢复机制**：
   - 检测 `.harness/temp/` 中未完成的需求
   - 恢复时根据 status 跳转对应环节
   - waitingFor 字段配合恢复
   - 多个未完成需求的列表选择
   - 丢弃旧需求

**验收标准**：推送机制完整，状态表覆盖所有状态变更点，中断恢复机制可执行。

**提交信息**：`feat: 添加推送、状态表和中断恢复机制`

---

### Step 8：编写 SKILL.md —— 检测规则 + 使用示例

**目标**：补充测试命令自动检测表和编译/Lint 自动检测表，以及尾部使用示例。

**涉及文件**：`skills/spec-create-harness/SKILL.md`

**内容要点**：
1. **测试命令自动检测**：Node.js/Python/Rust/Go/Java/Makefile 的检测规则表
2. **编译/Lint 自动检测**：各语言的编译检查、linter 匹配规则表
3. **使用示例**：一个完整的用户输入到 skill 响应的示例场景

**验收标准**：检测规则表完整，覆盖主流语言，使用示例与 spec 一致。

**提交信息**：`feat: 添加检测规则和使用示例`

---

### Step 9：更新 .gitignore

**目标**：将 `.harness/temp/` 添加到 `.gitignore`（运行时状态不应提交）。

**涉及文件**：`.gitignore`（如不存在则新建）

**内容**：
```
# Harness runtime state
.harness/temp/
```

**验收标准**：`.gitignore` 包含 `.harness/temp/` 规则。

**提交信息**：`chore: 添加 .harness/temp/ 到 gitignore`

---

### Step 10：全局验证

**目标**：确认所有部分组合为一个完整可用的 skill。

**涉及文件**：全部新建文件

**验证项**：
1. `skills/spec-create-harness/SKILL.md` 的 Frontmatter 格式正确（`name`/`description`/`compatibility` 字段完整）
2. SKILL.md 包含完整的工作流（Step 1~6）
3. SKILL.md 包含中断恢复机制
4. SKILL.md 包含测试命令/Lint 检测规则
5. `.gitignore` 包含 `.harness/temp/` 规则
6. 执行 `npm test`（或当前项目的测试命令）确保不破坏已有功能

**提交信息**：不独立提交，验证通过即表明之前步骤全部合格

---

## 注意：文件范围（allowedPaths）

执行时由父 Agent 根据所有 plan 步骤涉及的文件的并集自动生成 `allowedPaths` 并写入 `state.json.plan.allowedPaths`（参见 spec Step 4.4），不在 plan 中预写。
