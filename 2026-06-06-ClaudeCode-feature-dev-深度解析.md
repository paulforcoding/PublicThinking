---
title: "Claude Code /feature-dev 深度解析：七阶段流水线式功能开发"
description: "逐层拆解 /feature-dev 的七阶段流水线——从代码库勘探到架构设计到质量审查，三个专用 subagent 的完整 Prompt 源码分析和设计哲学。"
---

# Claude Code `/feature-dev` 深度解析：七阶段流水线式功能开发

> 不是"帮我写个功能"然后闭眼接受。`/feature-dev` 把功能开发拆成七个阶段，每阶段有专门的 Agent 把关——从代码库勘探到架构设计到质量审查，每个环节都有用户决策点。本文逐层拆解源码和设计哲学。

---

AI 写代码最大的坑不是写得烂，而是**写得快**。你一句话丢过去，它三秒钟吐出一堆文件——然后你要花三十分钟理解它做了什么、为什么要这样做、有没有更好的做法。

Claude Code 的 `/feature-dev` 解决的就是这个问题。它把一个功能开发任务拆成七个阶段，每阶段有专门的 subagent 执行、有人工决策卡点，确保代码**在写之前就已经想清楚了**。

## 不是什么

先说 `/feature-dev` 不是什么：

- **不是 plan mode** — plan mode 只读不写，feature-dev 从探索到实现到审查走完全程
- **不是自动编程** — 每阶段有明确的用户决策点，用户掌握方向和节奏
- **不是 `/simplify`** — simplify 是写完后清理代码，feature-dev 是管理从零到一的完整开发流程。两者解决完全不同的问题

## 七阶段全景

```
Phase 1: Discovery         → "你要做什么？"（需求澄清）
Phase 2: Codebase Explore  → 2-3 个 Agent 并行勘探代码库
Phase 3: Clarify Questions → 列出所有模糊点，等你回答
Phase 4: Architecture      → 2-3 个架构师给出不同方案
Phase 5: Implementation    → 主 Agent 直接写代码（等你拍板后）
Phase 6: Quality Review    → 3 个 Reviewer 并行审查
Phase 7: Summary           → 交付清单
```

三个关键设计决策贯穿全程：

1. **不猜需求，反复确认** — Phase 1 和 Phase 3 两轮澄清
2. **不跳步勘探** — Phase 2 必须跑完才能设计架构
3. **不直接改代码** — Phase 5 必须用户明确批准才开始

---

## Phase 1: Discovery — 需求澄清

入口。命令定义在 `commands/feature-dev.md`：

```yaml
---
description: Guided feature development with codebase understanding and architecture focus
argument-hint: Optional feature description
---
```

启动时可以带参数也可以不带：

```bash
/feature-dev Add rate limiting to API endpoints
/feature-dev    # 不带参数会先问你要做什么
```

做的事很直接：问清楚问题、目标、约束。然后**整理成一段总结让你确认**。看起来简单，但这是防止后面返工的第一道防线。

---

## Phase 2: Codebase Exploration — 并行勘探

这是 feature-dev 最独特的设计。它不是让你自己去翻代码，也不是让主 Agent 一个一个文件看。

**它同时启动 2-3 个 `code-explorer` Agent，每个从一个维度深入追踪代码库。**

源码中给出的典型 prompt 模板：

```
"Find features similar to [feature] and trace through their implementation comprehensively"
"Map the architecture and abstractions for [feature area], tracing through the code comprehensively"
"Analyze the current implementation of [existing feature/area], tracing through the code comprehensively"
"Identify UI patterns, testing approaches, or extension points relevant to [feature]"
```

每个 Agent 返回的不是"这里有个 auth 模块"，而是带有 `file:line` 引用的深度分析：入口点、调用链、数据流、架构层、依赖关系。

### code-explorer Agent 的内部规则

从源码 `agents/code-explorer.md` 可以看到它的配置：

```yaml
name: code-explorer
tools: Glob, Grep, LS, Read, NotebookRead, WebFetch, TodoWrite, WebSearch, KillShell, BashOutput
model: sonnet
```

**工具集里没有写操作。** 这是有意为之——勘探阶段不应该有任何副作用。用 Sonnet 而非 Opus，因为勘探任务更偏向广度搜索，不需要深度推理。

系统提示定义了四个分析层次：

1. **Feature Discovery** — 找入口点（API、UI 组件、CLI 命令）
2. **Code Flow Tracing** — 从入口追踪调用链、数据转换
3. **Architecture Analysis** — 映射抽象层、识别设计模式
4. **Implementation Details** — 关键算法、错误处理、性能考量

最关键的是输出要求的最后一条：

> "List of files that you think are absolutely essential to get an understanding of the topic in question"

Agent 勘探完后，主流程**不直接跳到设计**——它会把 Agent 返回的关键文件列表全部读一遍，建立深度上下文。这是 `feature-dev` 的核心哲学：**AI Agent 可以帮你找文件，但真正理解代码库的是读代码的过程。**

---

## Phase 3: Clarifying Questions — 不跳过这一步

源码中用 `CRITICAL` 和 `DO NOT SKIP` 标注的环节。

这个阶段的价值不在提问本身，而在于**强迫你在写代码之前想清楚边界**。Phase 2 的勘探结果 + 原始需求放在一起，自然会暴露出模糊地带：

- 边界情况（Edge cases）
- 错误处理策略
- 集成点
- 范围边界
- 向后兼容性
- 性能需求

源码中有一条关键规则：

> "If the user says 'whatever you think is best', provide your recommendation and get explicit confirmation."

即使你说"你定吧"，它也会给出建议然后**等你确认**。这是刻意设计的安全阀——不让 Agent 替你做决定。

---

## Phase 4: Architecture Design — 三套方案

同时启动 2-3 个 `code-architect` Agent，每个给一个不同的设计方向：

| 方向 | 目标 | 适用场景 |
|------|------|----------|
| **Minimal changes** | 最小改动，最大复用 | 小功能、热修复 |
| **Clean architecture** | 可维护性、优雅抽象 | 核心功能、长期维护 |
| **Pragmatic balance** | 速度 + 质量平衡 | 大多数情况 |

### code-architect Agent 的内部规则

```yaml
name: code-architect
tools: Glob, Grep, LS, Read, NotebookRead, WebFetch, TodoWrite, WebSearch, KillShell, BashOutput
model: sonnet
```

它的系统提示有一条反直觉的指令：

> "Make decisive choices - pick one approach and commit."

这是架构 Agent 和主流程的分工：**架构 Agent 给出一个确定的方案**（带着理由和 trade-off），**主流程把 2-3 个 Agent 的不同方案汇总对比**，然后让用户选。

Agent 的输出非常具体：

- 从代码库提取的模式和惯例（带 file:line 引用）
- 一个确定的架构决策（附理由和 trade-off）
- 每个组件的文件路径、职责、依赖、接口
- 分阶段的构建序列

到 Phase 4 为止，**一行代码都还没写。** 架构设计全部在只读 Agent 中完成。主流程拿到 2-3 套方案后，做对比表给用户选择。这一步的设计哲学是：**AI 负责生成选项，人负责做选择。**

---

## Phase 5: Implementation — 主 Agent 动手写

这是整个流程中**唯一不启动 subagent 的阶段**。

为什么？因为前四个阶段已经把该准备的都准备好了——代码库已经勘探透了、需求已经澄清了、架构方案已经选定了。实现阶段不需要再并行搜索或分析，它就是一个线性的执行过程。

源码中的指令：

> "DO NOT START WITHOUT USER APPROVAL"
>
> "Read all relevant files identified in previous phases"
>
> "Implement following chosen architecture"
>
> "Follow codebase conventions strictly"

实现由主 Agent 直接执行。主 Agent 拥有完整的上下文（Phase 2 勘探结果 + Phase 4 选定的架构蓝图），以及完整工具权限。它读文件、写代码、更新 TodoWrite 追踪进度。

这里有一个微妙的设计选择：**subagent 负责"理解和规划"，主 Agent 负责"执行"**。subagent 的上下文是隔离的、聚焦的，适合做专项分析。但写代码需要完整的项目上下文——subagent 看不到 Phase 2 的勘探结果和 Phase 4 的架构决策，所以写代码这件事必须回归到主 Agent。

---

## Phase 6: Quality Review — 三维并行审查

写完代码不是终点。Phase 6 启动 3 个 `code-reviewer` Agent 并行审查：

| Agent | 维度 | 关注点 |
|-------|------|--------|
| 1 | Simplicity / DRY / Elegance | 代码质量、可维护性 |
| 2 | Bugs / Functional Correctness | 逻辑错误、边界处理 |
| 3 | Conventions / Abstractions | 项目规范符合度 |

### code-reviewer Agent 的内部规则

```yaml
name: code-reviewer
tools: Glob, Grep, LS, Read, NotebookRead, WebFetch, TodoWrite, WebSearch, KillShell, BashOutput
model: sonnet
color: red
```

这个 Agent 最精妙的设计是**置信度过滤**：

```
0:   不自信，误报或已有问题
25:  不太确定，可能是误报
50:  中等，可能有但不太重要
75:  高度确信，验证过，会实际命中
100: 绝对确定，证据直接支持

只报告置信度 ≥ 80 的问题。
```

这意味着 code-reviewer 不会像传统 linter 那样报一大堆 noise。它只报真正值得关注的问题，每个带 `file:line` 引用和**具体修复建议**。

### 和 `/simplify` 的审查模式对比

三个并行 Reviewer + 各自一个维度，这个模式 `/simplify` 也在用。但两个命令的审查维度不同：

| | `/simplify` 的审查 | `/feature-dev` Phase 6 的审查 |
|---|---|---|
| Agent 1 | 代码复用（重复造轮子） | 简洁性 / DRY / 可读性 |
| Agent 2 | 代码质量（结构问题） | Bug / 功能正确性 |
| Agent 3 | 效率（性能、资源） | 项目规范 / 抽象合理性 |
| 时机 | 写完后的清理阶段 | 功能开发流程的最后一步 |
| 处理方式 | 自动修复 | 列出问题，用户决定修不修 |

核心区别：`/simplify` 关注的是"这段代码能不能更干净"，它直接上手改；feature-dev 的审查关注的是"这段代码有没有 bug、是否符合项目规范、结构是否合理"，它只报告，改不改由用户决定。

审查结果出来后，主流程呈现发现，让你三选一：现在修 / 以后修 / 不改。

---

## Phase 7: Summary — 交付清单

没有长篇总结。就是四样东西：做了什么、关键决策、修改了哪些文件、建议的下一步。干净利落。

---

## 三个 Agent 的权限模型

回看源码，三个 subagent 的工具集完全一样：

```
tools: Glob, Grep, LS, Read, NotebookRead, WebFetch, TodoWrite, WebSearch, KillShell, BashOutput
```

没有一个有 `Write` 或 `Edit` 权限。

feature-dev 的设计原则很清晰：**subagent 只做分析和建议，不做修改。** 写代码的权力只在 Phase 5 的主 Agent 手中，而且必须用户批准。

---

## 为什么这套流程值得认真对待

### 1. Subagent 和主 Agent 的分工明确

| 角色 | 做什么 | 权限 |
|------|--------|------|
| code-explorer | 勘探代码库 | 只读 |
| code-architect | 设计架构 | 只读 |
| code-reviewer | 审查代码 | 只读 |
| 主 Agent | 写代码（Phase 5） | 读写 |

写代码这件事被严格隔离在 Phase 5，并且前面四个阶段的准备工作已经到位。

### 2. 用户决策点精确卡位

Phase 1 确认需求 → Phase 3 澄清模糊点 → Phase 4 选择方案 → Phase 5 批准开工 → Phase 6 决定修不修。五个决策点，确保人始终在 loop 里，但不是在 micro-manage。

### 3. 架构是选择题，不是填空题

Phase 4 不给"一个设计方案"——它给 2-3 个，每个有不同的 trade-off。这不是因为 AI 不自信，而是因为**架构决策的权重不在技术上，在上下文里**——你比 AI 更了解团队偏好、时间压力、长期规划。

### 4. 显式的"理解代码库"阶段

大多数 AI 编码工具把"理解代码库"当作一个隐式的、自动发生的背景任务。feature-dev 把它变成一个显式的、2-3 个 Agent 并行的、必须交付关键文件清单的阶段。勘探不完成，设计不开始。

---

## 如何在其他工具中复现

`/feature-dev` 的核心不是一个专有算法，它是一个**流程模板 + 三个专用 subagent prompt**。

### Phase 2 勘探 Agent 提示词

```
你是代码分析专家。深入追踪 [功能区域] 的实现。

分析层次：
1. 找到所有入口点（API、UI、CLI），标注 file:line
2. 追踪完整调用链，记录每一步的数据转换
3. 映射抽象层（表现层→业务逻辑→数据层）
4. 识别设计模式和架构决策
5. 记录依赖关系

输出必须包含：必须读懂的文件列表（5-10 个，带 file:line）
```

### Phase 4 架构 Agent 提示词

```
你是架构师。基于代码库分析，设计 [功能] 的完整实现方案。

要求：
1. 分析现有代码模式（带 file:line 引用）
2. 做出确定的架构选择，不要给多个选项
3. 给出每个组件的文件路径、职责、依赖、接口
4. 实现地图：哪些文件要新建/修改
5. 数据流：入口→转换→输出
6. 分阶段构建序列
```

### Phase 6 审查 Agent 提示词

```
你是代码审查专家。审查以下 diff。

只报告置信度 ≥ 80 的问题。评分标准：
- 0-25: 忽略（误报/不确定）
- 50: 可能有但不重要
- 75+: 高度确信，会实际命中

审查维度：[简洁性/DRY] 或 [Bug/正确性] 或 [项目规范符合度]

每个问题给：描述 + 置信度 + file:line + 具体修复建议。
```

---

`/feature-dev` 源码：[github.com/anthropics/claude-plugins-official/tree/main/plugins/feature-dev](https://github.com/anthropics/claude-plugins-official/tree/main/plugins/feature-dev)

（仓库：`anthropics/claude-plugins-official` ⭐29,479 🔀3,162 📅2026-06-06 📄Apache-2.0）
