# Plan: 实现 Review Agent 场景扩展文件 + SKILL.md 审查标准解耦

## 概述

基于 `.claude/spec/review.md` 规格文档，创建 Review Agent 的**三个场景扩展文件**，并将 `skills/spec-create-harness/SKILL.md` 中的内联审查标准替换为对这些扩展文件的引用。

### 背景分析

| 项目 | 当前状态 | 目标 |
|------|---------|------|
| `.claude/spec/review.md` | ✅ 框架文件已完成 | 不变 |
| `skills/spec-create-harness/references/review-spec.md` | ❌ 不存在（spec 中标记为已完成但文件缺失） | 创建（内容以 common_review.md 为基础，做场景适配） |
| `skills/spec-create-harness/references/review-plan.md` | ❌ 不存在 | 创建 |
| `skills/spec-create-harness/references/review-exec.md` | ❌ 不存在 | 创建 |
| `skills/spec-create-harness/SKILL.md` 3.2 节 | 内联审查标准（硬编码在 SKILL.md 中） | 替换为引用 `references/review-spec.md` |
| `skills/spec-create-harness/SKILL.md` 4.2 节 | 内联审查标准（硬编码在 SKILL.md 中） | 替换为引用 `references/review-plan.md` |

### 核心设计决策

三个扩展文件采用**统一结构**（spec 0.5 节约定）：

```
# {场景名} 审查 — P0/P1/P2 分类表

## 1. 审查场景说明
场景描述、产物文件位置、建议文件位置。

## 2. P0/P1/P2 分类表
等级 | 类别 | 描述 的表格。

## 3. 检查重点（Reviewer Agent 注入用）
Reviewer Agent 启动时需要关注的审查要点列表。

## 4. 判断标准
父 Agent 退出循环的具体条件。
```

## 涉及文件总览

| 文件 | 操作 | 说明 |
|------|------|------|
| `skills/spec-create-harness/references/review-spec.md` | **新建** | Spec 审查专用 — 文档导向 P0/P1/P2 分类 |
| `skills/spec-create-harness/references/review-plan.md` | **新建** | Plan 审查专用 — 流程导向 P0/P1/P2 分类 |
| `skills/spec-create-harness/references/review-exec.md` | **新建** | 执行/代码审查专用 — 代码导向 P0/P1/P2 分类 |
| `skills/spec-create-harness/SKILL.md` | **修改** | 3.2 和 4.2 节审查标准替换为引用 |

## 执行步骤

---

### Step 1：创建 `references/review-spec.md`

**目标**：创建 Spec 规格文档审查专用扩展文件。

**设计依据**（spec 0.2 节 + common_review.md）：

| 等级 | 类别 | 设计理由 |
|------|------|---------|
| 🔴 P0 — 内容遗漏 | 关键功能规格、边界条件、异常流程缺失 | 直接沿用 common_review.md 的 "内容遗漏"，聚焦文档场景 |
| 🔴 P0 — 逻辑矛盾 | 同一 spec 中存在冲突的功能描述或数据定义 | 直接沿用 common_review.md 的 "逻辑矛盾" |
| 🔴 P0 — 方案缺陷 | 技术方案明显不可行或选型错误 | 保留（Spec 审查核心——方案要在 spec 层确定） |
| 🔴 P0 — 不可验证 | 验收标准缺失或模糊，无法判断是否达标 | 沿用 common_review.md，强调 spec 的验收标准必须可衡量 |
| 🟡 P1 — 描述不清 | 表述含糊、有歧义 | 沿用 common_review.md |
| 🟡 P1 — 边界缺失 | 输入校验、空状态、并发、超时未覆盖 | 沿用 common_review.md "边界缺失" |
| 🟡 P1 — 上下文断裂 | 依赖外部系统或前置假设但不做说明 | 沿用 common_review.md |
| 🟡 P1 — 不完整 | 某部分有改进空间，但当前版本尚可工作 | 沿用 common_review.md |
| 🟢 P2 — 措辞润色 / 格式风格 / 过度设计 | 沿用 common_review.md | 默认不输出 |

**检查重点清单**（spec 0.2 ②）：

- 功能规格是否完整覆盖用户需求
- 技术方案是否可行且合理（选型理由、替代方案对比）
- 边界条件和异常场景是否覆盖（输入校验、空值、网络超时、并发冲突）
- 非功能需求是否明确（性能指标、安全要求、兼容性）
- 验收标准是否可验证（不应出现"系统应流畅运行"这类模糊表述）
- 涉及的文件结构是否完整列出

**"无问题"判断标准**（spec 0.2 ③）：

- P0 列表为空（无遗漏、无矛盾、方案可行、验收标准可衡量）
- P1 列表中无"内容缺失"类的实质性问题，仅含措辞或格式建议

**涉及文件**：
- `skills/spec-create-harness/references/review-spec.md`

**验收标准**：
- 文件结构符合 spec 0.5 节约定的四段式结构
- 分类表与 common_review.md 保持一致性（spec 0.2 ① 要求）
- 检查重点覆盖 spec 0.2 ② 的全部项目
- "无问题"判断标准与 spec 0.2 ③ 一致

---

### Step 2：创建 `references/review-plan.md`

**目标**：创建 Plan 执行计划审查专用扩展文件。

**设计依据**（spec 0.3 节）：

| 等级 | 类别 | 与 review-spec.md 的差异 |
|------|------|-------------------------|
| 🔴 P0 — 步骤遗漏 | 必要开发步骤缺失 | 新增（Plan 特有——确保执行完整性） |
| 🔴 P0 — 依赖错误 | 步骤间依赖关系不合理 | 新增（Plan 特有——替换"上下文断裂"） |
| 🔴 P0 — 逻辑矛盾 | 同一 plan 中出现冲突的描述 | 沿用 |
| 🔴 P0 — 不可验证 | 某步骤的验收标准缺失或模糊 | 沿用，聚焦到"每步"粒度的可验证性 |
| 🟡 P1 — 描述不清 | 步骤目标或操作描述含糊 | 沿用 |
| 🟡 P1 — 粒度不当 | 步骤划分过大或过小 | **新增**（Plan 特有——替换 review-spec 的"边界缺失"） |
| 🟡 P1 — 文件遗漏 | 涉及文件未在步骤中列出 | **新增**（Plan 特有——影响 allowedPaths 准确性） |
| 🟡 P1 — 不完整 | 步骤覆盖面合理但可更细致 | 沿用 |
| 🟢 P2 — 措辞润色 / 格式风格 / 过度拆分 | 沿用 | "过度设计" 调整为 "过度拆分" |

**与 spec 审查的差异**（spec 0.3 节底部说明）：
- 去除了"方案缺陷"（方案由 spec 决定，plan 不应对此发表意见）
- 将"上下文断裂"替换为"依赖错误"
- 新增"粒度不当"

**检查重点清单**（spec 0.3 ②）：

- 步骤划分粒度是否合理（每个步骤应是独立的可提交单位）
- 步骤依赖顺序是否正确（前置条件是否满足）
- 涉及文件是否完整列出，allowedPaths 是否覆盖所有需要变更的文件
- 是否有步骤同时修改同一文件的不同部分，是否需要合并或拆分
- 每步的验收标准是否可自动/手动验证
- 总的步骤数是否合理（过少意味着粒度太粗，过多意味着过于琐碎）

**"无问题"判断标准**（spec 0.3 ③）：

- P0 列表为空（无步骤遗漏、依赖正确、顺序合理、验收标准可衡量）
- P1 列表中无"步骤遗漏"或"依赖错误"类的实质性问题

**涉及文件**：
- `skills/spec-create-harness/references/review-plan.md`

**验收标准**：
- 分类表体现了与 review-spec.md 的差异化（去掉"方案缺陷"，新增"依赖错误"和"粒度不当"）
- 检查重点覆盖 spec 0.3 ② 的全部项目
- 判断标准与 spec 0.3 ③ 一致

---

### Step 3：创建 `references/review-exec.md`

**目标**：创建执行/代码产物审查专用扩展文件。

**设计依据**（spec 0.4 节）：

| 等级 | 类别 | 说明 |
|------|------|------|
| 🔴 P0 — 功能不正确 | 实现与 spec 不符，或核心逻辑存在明显错误 | 代码审查的第一优先级 |
| 🔴 P0 — 安全漏洞 | SQL 注入、XSS、敏感信息泄露、认证绕过 | 从 spec 审查的"方案缺陷"分化而来 |
| 🔴 P0 — 破坏已有功能 | 修改导致未预期的模块或已有功能受损 | 需要验证测试覆盖 |
| 🔴 P0 — 测试缺失 | 核心逻辑无对应测试覆盖 | 确保可回归 |
| 🟡 P1 — 错误处理不完善 | 网络超时、空值、非法输入等边界未处理 | 代码层面的错误处理 |
| 🟡 P1 — 性能隐患 | N+1 查询、重复计算、未使用缓存 | 性能角度 |
| 🟡 P1 — 可维护性 | 逻辑过于复杂、缺少必要注释 | 人因工程 |
| 🟡 P1 — 测试不充分 | 仅覆盖 happy path，未覆盖异常路径 | 测试质量 |
| 🟢 P2 — 代码风格 / 重构机会 / 过度设计 | 默认不输出 | 与 spec/plan 审查的 P2 不同 |

**与文档审查的差异**（spec 0.4 节底部说明）：
- 这是一套专门面向代码的完整分类，与 spec/plan 完全不同
- "方案缺陷"已不适用（方案由 spec 确定）
- 核心关注点是正确性和安全性

**检查重点清单**（spec 0.4 ②）：

- 实现是否与 spec 规格一致（功能正确性）
- 是否存在安全漏洞（输入校验、认证、数据保护）
- 是否有未处理的错误或异常路径
- 是否引入不必要的依赖或副作用
- 测试是否覆盖核心逻辑和边界条件
- 代码是否可读、可维护
- 是否存在明显的性能问题

**"无问题"判断标准**（spec 0.4 ③）：

- P0 列表为空（功能正确、无安全漏洞、不破坏已有功能、核心逻辑有测试覆盖）
- P1 列表中无"功能错误"或"安全"类的实质性问题

**涉及文件**：
- `skills/spec-create-harness/references/review-exec.md`

**验收标准**：
- 分类表与 spec/plan 审查完全区分（证明是"全新代码导向分类"）
- 检查重点覆盖 spec 0.4 ② 的全部项目
- 判断标准与 spec 0.4 ③ 一致

---

### Step 4：修改 SKILL.md —— 审查标准解耦（3.2 节）

**目标**：将 SKILL.md 3.2 节中 spec-reviewer 的内联审查标准替换为对 `references/review-spec.md` 的引用。

**当前内容**（位于 3.2 节末尾）：
```markdown
> - 父 Agent 判断"无问题"的标准：spec-suggest.md 中没有指出逻辑遗漏、功能缺失、矛盾之处，仅可能有少量措辞建议
```

**修改为**（注入方式参考 spec 6.3）：
```markdown
> **审查标准**：spec-reviewer Agent 使用 `references/review-spec.md` 中定义的 P0/P1/P2 分类表和检查重点作为审查标准（详见 [review-spec.md](../references/review-spec.md)）。
>
> **启动 prompt 改造**：启动 spec-reviewer Agent 时，将 `references/review-spec.md` 的以下内容注入 prompt：
> - P0/P1/P2 分类表（用于分类建议）
> - 检查重点清单（用于指导审查方向）
> - 判断标准（用于指导建议的"度"）
>
> **父 Agent 判断"无问题"的标准**：
> - spec-reviewer 输出的建议文件中：
>   - P0 列表为空（无遗漏、无矛盾、方案可行、验收标准可衡量，见 `references/review-spec.md §4`）
>   - P1 列表中无"内容缺失"类的实质性问题，仅含措辞或格式建议
```

**涉及文件**：`skills/spec-create-harness/SKILL.md`

**验收标准**：
- 3.2 节的审查标准从"内联硬编码"变为"引用 review-spec.md"
- spec-reviewer 的启动方式明确了扩展文件内容的注入位置

---

### Step 5：修改 SKILL.md —— 审查标准解耦（4.2 节）

**目标**：将 SKILL.md 4.2 节中 plan-reviewer 的内联审查标准替换为对 `references/review-plan.md` 的引用。

**当前内容**（位于 4.2 节末尾）：
```markdown
> - 父 Agent 判断"无问题"的标准：plan-suggest.md 中没有指出步骤遗漏、文件缺失、依赖顺序错误等实质性问题
```

**修改为**：
```markdown
> **审查标准**：plan-reviewer Agent 使用 `references/review-plan.md` 中定义的 P0/P1/P2 分类表和检查重点作为审查标准（详见 [review-plan.md](../references/review-plan.md)）。
>
> **启动 prompt 改造**：启动 plan-reviewer Agent 时，将 `references/review-plan.md` 的以下内容注入 prompt：
> - P0/P1/P2 分类表（用于分类建议）
> - 检查重点清单（用于指导审查方向）
> - 判断标准（用于指导建议的"度"）
>
> **父 Agent 判断"无问题"的标准**：
> - plan-reviewer 输出的建议文件中：
>   - P0 列表为空（无步骤遗漏、依赖正确、顺序合理、验收标准可衡量，见 `references/review-plan.md §4`）
>   - P1 列表中无"步骤遗漏"或"依赖错误"类的实质性问题
```

**涉及文件**：`skills/spec-create-harness/SKILL.md`

**验收标准**：
- 4.2 节的审查标准从"内联硬编码"变为"引用 review-plan.md"
- plan-reviewer 的启动方式明确了扩展文件内容的注入位置
- 修改方式与 Step 4 对称、一致

---

### Step 6：全局验证

**目标**：确认三个扩展文件全部创建且完整，SKILL.md 审查标准解耦完成。

**涉及文件**：全部新建和修改的文件

**验证项**：
1. `references/review-spec.md` 存在，包含四段式结构（场景说明 → 分类表 → 检查重点 → 判断标准）
2. `references/review-plan.md` 存在，分类表与 review-spec.md 有明显的差异化设计
3. `references/review-exec.md` 存在，分类表与 spec/plan 审查完全不同（代码导向）
4. `SKILL.md` 3.2 节不再包含内联审查标准，改为引用 `references/review-spec.md`
5. `SKILL.md` 4.2 节不再包含内联审查标准，改为引用 `references/review-plan.md`
6. 三个扩展文件的分类表与 `.claude/spec/review.md` 6.2 节的参考方向一致
7. 三个扩展文件遵循 `.claude/spec/review.md` 0.5 节的统一结构约定

**提交信息**：不独立提交，验证通过即表明之前步骤全部合格

---

## 注意事项

### 1. review-spec.md 与 common_review.md 的关系

`review-spec.md` 是 `common_review.md` 在 Spec 审查场景的具体化。两者的关系是：

```
common_review.md（通用框架）
       │
       ▼
review-spec.md（Spec 场景适配）
  - 分类表沿用 common_review 的结构
  - 检查重点聚焦到 Spec 文档的特性（功能规格、技术方案、验收标准）
  - 判断标准具体化为 Spec 场景的"无问题"条件
```

review-spec.md 应**显式引用** common_review.md 作为基础框架，而不是独立设计一套新分类。

### 2. 三个扩展文件的区分

| 维度 | review-spec.md | review-plan.md | review-exec.md |
|------|---------------|---------------|---------------|
| 产物 | 定义性文档 | 执行流程 | 代码变更 |
| 审查导向 | 文档导向 | 流程导向 | 代码导向 |
| 核心关注 | 完整性、正确性 | 步骤划分、依赖关系 | 正确性、安全性 |
| 与 common_review 的关系 | 直接沿用 + 场景细化 | 在 common_review 基础上做场景调整 | 全新分类，与 common_review 无关 |

### 3. 文件范围（allowedPaths）

执行时由父 Agent 根据所有 plan 步骤涉及的文件的并集自动生成 `allowedPaths` 并写入 `state.json.plan.allowedPaths`（参见 spec Step 4.4），不在 plan 中预写。本计划的操作范围是 skill 源文件目录 `skills/spec-create-harness/` 下的引用文件和 SKILL.md，不涉及运行时文件。
