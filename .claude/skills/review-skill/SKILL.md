---
name: review-skill
description: Review code changes with structured P0/P1/P2 grading. Trigger when the user asks to review code, do a code review, audit changes, check a diff, or review a PR/MR. This is a general-purpose code review skill — use it for ANY code review request even if the user doesn't explicitly say "skill" or "review skill". It works on uncommitted changes, specific diffs, or PRs. Outputs a prioritized problem list with severity levels. Use this BEFORE approving any code change, writing merge comments, or giving code feedback.
compatibility:
  requires: [git, Skill tool]
---

# Review Skill — 代码审查

对代码变更进行结构化审查。**核心原则**：分级优先、范围锁定、务实导向。

## 审查哲学

1. **分级优先** — 按 P0 > P1 > P2 分级，默认只输出 P0 + P1
2. **范围锁定** — 只审查本次变更，不翻旧账（除非变更行暴露了相邻行的严重问题）
3. **务实导向** — 考虑修改成本和收益，不追求理论完美

## 输入方式（按优先级）

| 方式 | 说明 |
|------|------|
| 用户直接传入 diff 内容 | 直接用用户提供的内容审查 |
| `git diff` + `git diff --cached` | 自动获取未提交变更 |
| 参数指定范围 | 用户指定 commit 范围(`main..HEAD`) 或文件列表 |
| stdin 管道 | 通过管道传入 diff |

## 审查流程

### Step 1：获取变更内容

- 检查当前目录是否为 Git 仓库（非 Git 目录 -> 提示 "当前目录不是 Git 仓库"）
- 执行 `git diff` 和 `git diff --cached` 获取变更
- 如果 diff 为空 -> 提示 "当前没有检测到未提交的变更"，结束
- 如果 diff > 5000 行 -> 自动分块审查，提示 "变更较大，已分块审查"
- 跳过二进制文件、锁文件（package-lock.json、yarn.lock）、生成文件（dist/*、build/*）

### Step 2：逐文件逐块审查

对每个变更文件、每个 diff hunk，识别问题并按以下标准分级：

#### 🔴 P0 - 必须修改（阻塞合并）

| 类别 | 判定 | 示例 |
|------|------|------|
| 逻辑错误 | 条件/边界/算法明显错误 | 条件判断反了、循环越界 |
| 空指针/NPE | 未判空直接调用 | `obj.field` 但 obj 可能为 null |
| 安全漏洞 | 注入/硬编码/越权 | SQL 拼接、密码硬编码 |
| 数据丢失 | 事务/异常/并发处理缺失 | try 中操作 DB 但无事务 |
| 性能灾难 | 复杂度剧增/N+1 | 循环内查 DB |

#### 🟡 P1 - 建议修改（影响可维护性）

| 类别 | 判定 | 示例 |
|------|------|------|
| 代码异味 | 重复代码、过长函数(>50行) | 相同逻辑出现 ≥2 处 |
| 可读性 | 命名不清晰、魔法数字 | `let a = 86400` 无注释 |
| 健壮性 | 缺少日志、空 catch | `catch(e){}` 吞异常 |
| 测试覆盖 | 核心路径无单测 | 新增逻辑方法无对应测试 |

#### 🟢 P2 - 可选优化（默认不输出，除非用户要求）

| 类别 | 示例 |
|------|------|
| 设计模式 | 可重构但当前实现清晰 |
| 性能微调 | 收益 < 5% |
| 风格微差 | 与现有风格略有不一致 |
| 过度设计 | 为不确定未来做的提前扩展 |

### Step 3：过滤与排序

1. 按 `minSeverity` 配置过滤（默认只保留 P0 + P1）
2. 按 `ignorePatterns` 过滤文件
3. 按 P0 > P1 > P2 降序排列，同级别按文件路径排序
4. 裁到 `maxFindings` 条（默认 30）

### Step 4：生成报告

**每条问题包含**：
- `#[序号]` + `[P0/P1/P2]` 级别标签（终端加颜色：红/黄/绿）
- **文件**：`path:行号`
- **类别**：如 `逻辑错误`
- **问题**：简短标题
- **说明**：为什么这是个问题
- **建议**：具体修改方案（含代码示例）
- **成本**：修改成本（高/中/低）

**报告结构**：
```markdown
# 🔍 代码审查报告

## 概览
- 审查范围：N 个文件，±N 行
- 问题统计：P0: N | P1: N | P2: N
- 总体评价：✅ 通过 / ⚠️ 有条件通过 / ❌ 不通过

## 问题列表

### 🔴 P0 (N 个)

#1 [P0] [逻辑错误] 条件判断方向错误
- **文件**: src/auth.ts:42-48
- **问题**: `if (!isAdmin)` 应为 `if (isAdmin)`，导致非管理员才能通过
- **建议**: 
  ```diff
  - if (!isAdmin) { grantAccess() }
  + if (isAdmin) { grantAccess() }
  ```
- **成本**: 低

### 🟡 P1 (N 个)

#2 [P1] [代码异味] 重复的验证逻辑
...

## 总结
- **必须修复**：N 个 P0 问题阻塞合并
- **建议修复**：N 个 P1 问题影响可维护性
```

## 配置（通过 skill 参数传入）

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `minSeverity` | `P1` | 最低输出级别：`P0` 只输出阻塞项，`P1` 默认，`P2` 全输出 |
| `format` | `markdown` | 输出格式：`markdown` 或 `json` |
| `maxFindings` | `30` | 最多输出问题数 |
| `focus` | `""` | 仅审查指定文件，逗号分隔 |
| `ignore` | `""` | 忽略匹配文件，glob 逗号分隔 |

## 边界情况

| 情况 | 处理 |
|------|------|
| 无变更 | "当前没有检测到未提交的变更" |
| 超大 diff | 分块审查并提示 |
| 二进制文件 | "跳过二进制文件：<路径>" |
| 非 Git 目录 | "当前目录不是 Git 仓库" |
| 新文件 | 逐行审查，标注 "新增文件" |
| 删除文件 | 验证删除是否合理 |
| 重命名(无内容变更) | 审查文件名变更本身 |
| 锁文件/生成文件 | 自动跳过 |

## 注意事项

- **范围锁定**：不要审查变更范围外的代码，除非相邻行暴露了严重问题且修复成本极低
- **务实导向**：P0 必须提，P1 选择性提（重复3次以上才提代码异味），P2 除非用户要求否则不提
- **语言**：报告使用中文输出，代码示例使用实际编程语言
- **情绪**：语气专业平和，不要咄咄逼人
- **上下文**：如果用户提供了额外的上下文（项目背景、约束条件），在审查时考虑这些因素
