# Agent 职责重构 — 父 Agent 生成 spec/plan，子 Agent 仅负责 Review

将 `skills/spec-create-harness/SKILL.md` 的 Agent 架构从「父 Agent 编排 + generator/reviewer 双子 Agent」改为「父 Agent 亲自生成 spec/plan + 子 Agent 仅审查 + 父 Agent 亲自修补」。

**核心变更一句话：** 生成与修补职责从子 Agent（`spec-generator` / `plan-generator`）**回归父 Agent**；子 Agent 仅保留 `spec-reviewer` / `plan-reviewer` 的**独立审查**职责；父 Agent 读取建议文件后**亲自将建议修复到 spec/plan**，随后**删除建议文件**。

---

## 1. 需求背景

原架构（`分析.md` 所描述）中，spec/plan 的生成、修补全部由子 Agent 完成：

```
父 Agent ──启动──▶ spec-generator ──写──▶ spec.md ──报告──▶ 父 Agent
父 Agent ──启动──▶ spec-reviewer  ──写──▶ spec-suggest.md ──报告──▶ 父 Agent
父 Agent 判断有实质问题 → ──启动──▶ spec-generator(修补) ──改──▶ spec.md
```

这条链路存在以下成本与矛盾：

| 问题 | 说明 |
|------|------|
| **上下文重复搬运** | 父 Agent 在 Step 1 已完成需求澄清、掌握完整用户上下文，却要在生成时把"需求 + 项目上下文"重新打包传给无状态的 generator 子 Agent；修补时还要把 suggest 内容**二次打包**传给新的 generator 实例 |
| **角色冗余** | generator 与父 Agent 的实际产出高度重合（生成 spec/plan 本就可以由拥有全量上下文的父 Agent 完成），却以两个角色分居两端 |
| **链路过长** | 每轮校验 = 启动 reviewer（报告）→ 父判断 → 启动 generator（报告）→ 父删文件，消息往返多 |
| **状态割裂** | 产物质量与父 Agent 的对话上下文割裂：父 Agent 判断完还得靠"打包给新实例"传递修补指令，中断恢复时依赖 suggest 文件自包含 |

**因此重构**：让拥有最完整上下文、最有判断权的父 Agent **直接成为 spec/plan 的作者与修补者**；子 Agent 退化为**纯审查方**，保留其独立、挑剔、不修改产物的价值。子 Agent 越少，上下文搬运越少，父 Agent 对产物所有权越强。

---

## 2. 修改目标

1. **父 Agent 亲自生成 spec/plan**：不再启动 `spec-generator` / `plan-generator` 子 Agent，生成职责并入父 Agent（读取需求/项目上下文 → 写产物文件）
2. **子 Agent 仅保留 Review 职责**：只保留 `spec-reviewer` / `plan-reviewer`，职责不变（读产物 → 写 `*-suggest.md` → 报告）
3. **父 Agent 亲自修补**：读取建议文件后，**父 Agent 自己**将建议落实到 spec/plan，不再启动 generator 修补模式
4. **删除建议文件**：父 Agent 每次处理完建议后**立即删除**建议文件（维持"每轮视角刷新"）
5. **Review 独立性不丢失**：reviewer 仍是独立子 Agent、"不直接修改产物"，"提建议者不改文件"原则完整保留
6. **执行阶段不变**：`execution` 子 Agent 及其 allowedPaths 校验、分层验证、提交逻辑全部保留

---

## 3. 角色边界变更前后对比

### 3.1 变更前（现 SKILL.md）

| 角色 | 生成 spec | 审查 spec | 修补 spec | 生成 plan | 审查 plan | 修补 plan | 执行 |
|------|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| 父 Agent | — | — | 决策/编排 | — | — | 决策/编排 | 编排/纠错 |
| spec-generator 子 Agent | ✅ | — | ✅ | — | — | — | — |
| spec-reviewer 子 Agent | — | ✅ | — | — | — | — | — |
| plan-generator 子 Agent | — | — | — | ✅ | — | ✅ | — |
| plan-reviewer 子 Agent | — | — | — | — | ✅ | — | — |
| execution 子 Agent | — | — | — | — | — | — | ✅ |

### 3.2 变更后（本规格）

| 角色 | 生成 spec | 审查 spec | 修补 spec | 生成 plan | 审查 plan | 修补 plan | 执行 |
|------|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| 父 Agent | ✅ | — | ✅ | ✅ | — | ✅ | 编排/纠错 |
| spec-reviewer 子 Agent | — | ✅ | — | — | — | — | — |
| plan-reviewer 子 Agent | — | — | — | — | ✅ | — | — |
| execution 子 Agent | — | — | — | — | — | — | ✅ |

> **取消**：`spec-generator`、`plan-generator` 两个子 Agent 角色被移除（职责并入父 Agent）。
> **保留**：`spec-reviewer`、`plan-reviewer`、`execution` 三个子 Agent，职责不变。

---

## 4. 新调用流程

### 4.1 Step 3：Spec 生成（父 Agent 主导）

```
Step 1 需求澄清（不变）
Step 2 初始化（不变）
Step 3 父 Agent 亲自生成 spec：
  1. 父 Agent 读取项目上下文 + 用户需求（Step 1 澄清结果已在上下文中）
  2. 父 Agent 亲自撰写 spec.md 并写入 .harness/temp/{vId}/spec.md
     （内容仍含：背景与目标 / 功能规格 / 非功能规格 / 技术方案 / 文件结构）
  3. 更新 state.json：status = spec_pending
  4. 父 Agent 向用户展示 spec 概要 + 选项界面（概要由父 Agent 直接总结，无需等待报告往返）
      1. AI 校验（推荐） → 进入 §4.3 校验循环
      2. 直接确认         → status = spec_confirming → Step 4
      3. 重新生成         → 父 Agent 删除当前 spec.md，回到本段第 1 步重新撰写
      4. 补充信息         → waitingFor = user_supplement → 用户补充后清除 → 父 Agent 重新撰写（带补充）
      5. 手动修改         → 用户编辑 spec.md → 输入"继续"回到本界面
```

> **与旧流程的差异**：原 §3.1 是"启动 spec-generator 子 Agent 生成 + 等报告"；现改为父 Agent 直接写文件、直接展示。用户看到的界面与选项完全不变。

### 4.2 Step 4：Plan 生成（父 Agent 主导）

```
Step 4 父 Agent 亲自生成 plan：
  1. 父 Agent 读取已确认的 .harness/temp/{vId}/spec.md
  2. 父 Agent 亲自拆解开发过程为可独立提交的步骤并写入 plan.md
     （每步仍含：步骤序号与名称 / 目标 / 涉及文件 / 验收标准，每步必须可独立通过测试）
  3. 父 Agent 汇总所有步骤"涉及文件" → 形成 allowedPaths（确认时写入 state.json，见 §4.4）
  4. 更新 state.json：status = plan_pending
  5. 父 Agent 向用户展示 plan 概要 + 选项界面（同上，选项行为同 §4.1）
```

### 4.3 AI 校验循环（通用，spec/plan 共用）

```
循环（最多 5 轮）：
  1. 父 Agent 启动 reviewer 子 Agent（新实例）
     - spec 阶段：spec-reviewer，prompt 注入 references/review-spec.md 的
       P0/P1/P2 分类表 + 检查重点 + 判断标准
     - plan 阶段：plan-reviewer，prompt 注入 references/review-plan.md 的对应内容
  2. reviewer 读取产物文件（spec.md / plan.md）
  3. reviewer 生成校验建议 → 写入建议文件（spec-suggest.md / plan-suggest.md）
  4. reviewer 向父 Agent 报告完成
  5. 父 Agent 读取建议文件，判断是否有实质性问题：
     a. 无问题（或仅轻微措辞建议） → 父 Agent 删除建议文件 → 退出循环 ✅
     b. 有实质问题 → 进入第 6 步
  6. 父 Agent 亲自修补：将建议逐条落实到产物文件
     （修改 spec.md / plan.md，规则见 §5）
  7. 父 Agent 删除建议文件
  8. 回到第 1 步进行下一轮校验
```

**与旧循环的差异（仅第 6-7 步）**：

| 步骤 | 旧流程 | 新流程 |
|------|--------|--------|
| 6 | 父 Agent 启动 spec-generator/plan-generator（修补模式），传递 suggest 内容 | **父 Agent 亲自修补产物文件** |
| 7 | generator 逐条采纳并修改产物，报告完成 | **父 Agent 直接删除建议文件** |

> 判断标准、5 轮上限、reviewer"第一次审查"心态、建议文件生命周期规则（review.md §5.3）均**不变**。

### 4.4 状态写入与文件范围（§4.5，仅属主变化）

父 Agent 生成 plan 时即汇总 `allowedPaths`；用户选择"直接确认"后，父 Agent 将 `allowedPaths` 写入 `state.json.plan.allowedPaths`（时机与格式不变）。手动修改选项（§4.2 选项 5）要求同步更新 allowedPaths 的规则不变。

### 4.5 Step 5/6（执行与推送，不变）

`execution` 子 Agent 的启动、allowedPaths 校验、分层验证、测试、commit、出错处理，以及 Step 6 验收归档、推送，**全部保持不变**。

---

## 5. 父 Agent 修补规则（替代原"Generator 修补模式"）

原 review.md §7.1 定义的"Generator 修补模式行为"迁移为**父 Agent 修补规则**：

| 规则 | 说明 |
|------|------|
| 逐条处理 | 对建议文件中的每条建议逐一评估并落实 |
| 默认采纳 | 除非有充分理由（如建议本身存在事实错误），否则应当采纳并落实到产物 |
| 不质疑 | 不质疑 reviewer 的发现（有异议时在下一轮由 reviewer 独立复核，而非当场争论） |
| 不透支 | 只处理建议文件中明确指出的问题，不擅自做额外优化 |
| 记录透明 | 修补完成后向用户简要说明"修复了 N 条建议"，保持过程可见 |

> 唯一新增约束：父 Agent 修补时须保持"作者视角"与"审查视角"分离——按 reviewer 建议改，而不是按自己喜好重写；若发现建议与自己的判断冲突，可借助"再次 AI 校验"让 reviewer 复核。

---

## 6. 对状态机 / state.json 的影响

**状态流转不变**（`spec_pending → [spec_reviewing] → spec_confirming → plan_pending → [plan_reviewing] → plan_confirming → executing → acceptance → accepted → completed`）。

| 项目 | 是否变更 | 说明 |
|------|:---:|------|
| `status` 取值 | 不变 | 5 个大状态 + 2 个可选 reviewing 状态原样保留 |
| `step` / `currentStep` | 不变 | 语义不变 |
| `waitingFor` | 不变 | 补充信息场景用法不变 |
| `plan.allowedPaths` | 不变 | 仍由父 Agent 在确认时写入（数据来源改为父 Agent 生成 plan 时汇总） |
| 建议文件 `spec-suggest.md` / `plan-suggest.md` | 不变 | 仍为 reviewer 输出、父 Agent 读删；生命周期不变 |

> 中断恢复逻辑中仅一处措辞变化：`spec_reviewing` / `plan_reviewing` 状态下检测到 suggest 文件存在时，由"启动 generator 修补模式继续"改为"**父 Agent 读取 suggest 文件继续修补**"。恢复路径本身（`status` + suggest 文件是否存在）判断逻辑不变。

---

## 7. 对参考文件的影响

| 文件 | 是否修改 | 说明 |
|------|:---:|------|
| `references/review-spec.md` | 否 | 仍为 spec-reviewer 的审查标准，注入方式不变 |
| `references/review-plan.md` | 否 | 仍为 plan-reviewer 的审查标准，注入方式不变 |
| `references/review-exec.md` | 否 | 与本次重构无关 |
| `.claude/spec/review.md`（框架文档） | **需同步** | §3.2 角色表、§7.1"Generator 修补模式行为"、§7.2"子 Agent 隔离原则"中关于 generator 子 Agent 的描述，需改为"父 Agent 承担生成与修补" |
| `.claude/spec/create_harness.md`（旧规格） | 可后续同步 | 本身已落后于 SKILL.md（见分析.md §7 观察①），本次一并由 SKILL.md 重构结果覆盖 |

---

## 8. SKILL.md 具体修改清单

| # | 位置（现 SKILL.md） | 修改内容 |
|---|--------------------|---------|
| 1 | §工作流总览 第 6 行"子 Agent 协作" | 更新为"生成由父 Agent 完成，子 Agent 仅承担 spec/plan 审查" |
| 2 | Step 3 概述段 | 去掉"两个独立子 Agent 协作完成"，改为"父 Agent 亲自生成 + 可选启动 spec-reviewer 审查" |
| 3 | §3.1（原 spec-generator） | **删除**子 Agent 启动描述，改写为"父 Agent 亲自生成 spec.md"的职责清单 |
| 4 | §3.3 AI 校验循环 | 第 6-7 步从"启动 spec-generator 修补模式"改为"父 Agent 亲自修补 + 删除建议文件" |
| 5 | §3.3 各角色职责表 | 移除 spec-generator 行，父 Agent 职责补充"亲自修补" |
| 6 | Step 4 概述段 | 同 #2 处理 plan 阶段 |
| 7 | §4.1（原 plan-generator） | 删除子 Agent 启动描述，改写为"父 Agent 亲自生成 plan.md + 汇总 allowedPaths" |
| 8 | §4.3 AI 校验循环 | 同 #4 处理 plan 阶段 |
| 9 | §4.3 各角色职责表 | 移除 plan-generator 行 |
| 10 | §交互要点·Agent 命名 | 更新命名列表：仅保留 `spec-reviewer` / `plan-reviewer` / `execution` 子 Agent |
| 11 | §交互要点·子 Agent 协作 | 更新为"父 Agent 生成与修补，子 Agent 仅审查"的描述 |
| 12 | §中断恢复机制 | `spec_reviewing` / `plan_reviewing` 恢复措辞更新（见 §6） |
| 13 | 各处"报告格式"（generator 报告） | 移除 generator 完成报告格式；概要改为父 Agent 直接总结展示 |

---

## 9. 兼容性

| 现有功能 | 是否受影响 | 说明 |
|---------|-----------|------|
| AI 校验循环（spec/plan） | 保留 | 结构不变，仅修补方从 generator 子 Agent 改为父 Agent |
| 最多 5 轮限制 | 保留 | 不变 |
| 用户选择界面（生成后/校验后） | 保留 | 选项与界面完全不变 |
| 补充信息 / 手动修改 / 重新生成 | 保留 | 逻辑不变，由父 Agent 直接重新生成 |
| allowedPaths 文件范围校验 | 保留 | 数据来源改为父 Agent 生成 plan 时汇总 |
| execution 执行流程 | 保留 | 完全不变 |
| 中断恢复 | 保留 | 判断逻辑不变，仅恢复动作措辞更新 |
| references 审查标准 | 保留 | 三个 review-*.md 均不改动 |

---

## 10. 不涉及的变更

- 不修改 `references/review-spec.md`、`review-plan.md`、`review-exec.md`
- 不修改 `execution` 子 Agent 的执行/校验/提交逻辑
- 不修改 Step 5（执行与测试）、Step 6（推送）流程
- 不修改状态机取值与 `state.json` 结构
- 不修改任何用户选择界面的文案与编号

---

## 11. 验收标准

1. **父 Agent 亲自生成**：spec/plan 均由父 Agent 直接写入 `.harness/temp/{vId}/`，不再出现 `spec-generator` / `plan-generator` 子 Agent 启动描述
2. **子 Agent 仅审查**：仅 `spec-reviewer` / `plan-reviewer` / `execution` 三类子 Agent 存在；reviewer 只写 `*-suggest.md`，不直接修改产物
3. **父 Agent 亲自修补**：AI 校验循环中发现实质问题后，由父 Agent 将建议落实到 spec/plan，不再启动任何"修补模式"子 Agent
4. **建议文件删除**：父 Agent 每处理完一轮建议即删除 `*-suggest.md`，退出循环时同样删除
5. **用户界面不变**：生成后/校验后选择界面、选项编号与文案与修改前一致
6. **状态机不变**：`status` 流转、`step` / `currentStep` / `waitingFor` / `allowedPaths` 语义与写入时机不变
7. **执行与推送不受影响**：Step 5 / Step 6 行为与修改前一致
8. **SKILL.md 无残留**：全文不再出现"启动 spec-generator/plan-generator（修补模式）"等描述
