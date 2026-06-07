# 为什么 AI 编码不能让 subagent 写代码：从两个开源项目的源码设计说起

上周六，我跑回公司加了一天班。

起因是周五同事用了一个"全自动 AI 开发流程"——让 Claude Code 的 subagent（一个叫"高级后端工程师"的角色）去完成一个功能开发。subagent 确实写了代码，也确实跑通了。但周一早上我发现，它把几个模块的依赖关系搞乱了，一些原本清晰的接口被重构成了只有 subagent 自己看得懂的形状。修这些"AI 写的代码"花了我一整天。

这件事让我开始认真查一个问题：**成熟的 AI 编码工作流，到底让谁来写代码？**

答案出乎意料地一致。我查了两个完全独立的项目——Anthropic 官方的 feature-dev 插件和 YC CEO Garry Tan 的 gstack——它们在设计上得出了同一个结论：**subagent 只能做研究和审查，写代码必须是主 agent。**

这不是巧合。这是上下文工程的物理定律。

---

## 两个项目，同一个设计

### feature-dev：Anthropic 官方的七阶段开发流程

feature-dev 是 Claude Code 的官方内置插件，源码在 `anthropics/claude-code` 仓库（⭐130k）的 `plugins/feature-dev/` 目录下。它把一个功能开发任务拆成七个阶段：

```
Phase 1: Discovery         → 主 agent（需求澄清）
Phase 2: Codebase Explore  → 2-3 个 code-explorer subagent（并行勘探）
Phase 3: Clarify Questions → 主 agent（澄清模糊点）
Phase 4: Architecture      → 2-3 个 code-architect subagent（并行设计）
Phase 5: Implementation    → 主 agent（写代码）  ← 注意这里
Phase 6: Quality Review    → 3 个 code-reviewer subagent（并行审查）
Phase 7: Summary           → 主 agent（交付清单）
```

Phase 2、4、6 都启动了 subagent，唯独 Phase 5——写代码的那个阶段——没有。

源码中 Phase 5 的指令是这样的：

```
## Phase 5: Implementation

Goal: Build the feature.

DO NOT START WITHOUT USER APPROVAL

Actions:
1. Wait for explicit approval
2. Read all relevant files from previous phases
3. Implement following chosen architecture
4. Follow codebase conventions strictly
5. Write clean, well-documented code
6. Update todos as you progress
```

没有提到任何 agent。指令直接说"Read all relevant files"和"Implement"——两个都是主 agent 的直接动作。

三个 subagent（code-explorer、code-architect、code-reviewer）的工具集完全相同，全部是只读工具：

```
tools: Glob, Grep, LS, Read, NotebookRead, WebFetch, TodoWrite, WebSearch, KillShell, BashOutput
```

没有一个有 `Write` 或 `Edit` 权限。写代码的权力只在主 agent 手中。

GitHub Issue #24932（讨论 feature-dev 迁移到 agent teams 的提案）明确提到了这个设计的意图：

> "Implementation (Phase 5) and review fixes (Phase 6) run in the lead's context, bloating it with implementation details."

翻译过来就是：Phase 5 和 Phase 6 故意在主 agent 的上下文中运行，即使这会让主 agent 的上下文膨胀。提案提出的改进是给 Phase 5 加一个 `code-implementer` agent，但这被描述为"proposed change"——意味着当前设计是刻意不这样做的。

### gstack：YC CEO 的 Claude Code 工作流

gstack 是 Garry Tan（Y Combinator CEO）开源的 Claude Code 工作流，46 个 skills，GitHub 上 107k stars。它的 sprint 流程是七步：

```
Think → Plan → Build → Review → Test → Ship → Reflect
```

和 feature-dev 的结构高度相似：

| 步骤 | 谁在干活 | 产出 |
|------|----------|------|
| Think（`/office-hours`） | 主 agent 扮演 YC Partner | 设计文档 |
| Plan（`/plan-ceo-review` 等） | 主 agent 扮演多角色 | 锁定架构 |
| **Build** | **主 agent（标准 Claude Code 编码）** | **代码** |
| Review（`/review`） | 7 个并行 subagent | 审查报告 |
| Test（`/qa`） | 主 agent | 测试通过 |
| Ship（`/ship`） | 主 agent | 发布 |

Build 步骤的描述是"Standard Claude Code coding, using the reviewed plan"。Plan 阶段产出的设计文档留在主 agent 的上下文中，Build 阶段直接读取这些文档来实现。

gstack 的 office-hours skill 中有一条硬性门控：

> "This skill produces design docs, not code. **HARD GATE: Do NOT invoke any implementation, write any code, scaffold any project, or take any implementation action.**"

规划阶段严禁写代码。写代码是 Build 阶段的事，由主 agent 执行。

---

## 为什么必须这样设计

两个完全独立的项目得出同一个结论，说明这不是偏好问题，而是技术约束。

### 约束一：subagent 的上下文是隔离的

Claude Code 的 subagent（通过 Task tool 启动）有自己独立的上下文窗口。它不继承主 agent 的对话历史、工具调用结果或 system prompt。它只接收一段精炼的 task prompt，完成后只返回最终摘要。所有中间过程——文件读取、搜索、推理——全部丢弃。

这意味着什么？

假设主 agent 在 Phase 2 勘探了代码库，Phase 3 和你澄清了需求，Phase 4 设计了架构方案。这些决策、权衡、上下文——全在主 agent 的脑子里。

如果你让一个 subagent 去写代码，它看到的是 task prompt 里的一段描述。它不知道 Phase 3 你回答了什么边界情况的问题，不知道 Phase 4 你为什么选了"最小改动"而不是"干净架构"，不知道代码库中有哪些隐含的约定。

一个实际案例（来自 wmedia.es 的技术分析）：

```
主 agent 的上下文：
- 正在重构 auth 模块，因为 token 过期 bug
- refreshToken() 有一个边界情况
- 需要知道当前中间件是否处理了这个情况

主 agent 委派给 subagent：
> "调查当前 auth 中间件的工作方式"

subagent 返回：
> "中间件使用 express-jwt，每个请求验证 token，有 3 个受保护路由。"

丢失的信息：refreshToken() 的边界情况——这才是任务的真正原因。
```

subagent 回答了字面上的问题，但丢失了语义上的意图。

### 约束二：摘要压缩必然丢失细节

subagent 的设计目的就是压缩信息——它读了几千行代码，返回几百字的摘要，保护主 agent 的上下文不被淹没。

但压缩是有损的。wmedia.es 的分析给出了一个具体数字：

> "A sub-agent can read 6,100 tokens of files and return a 420-token summary. That compression is the whole point — it protects your main context. But it's also where the details that matter get lost."

6100 token 压成 420 token，压缩比 14:1。被丢掉的是什么？边界情况、隐含依赖、微妙的调用顺序——恰恰是写代码时最不能丢的东西。

研究任务可以容忍有损压缩（你需要的是方向性的结论）。但写代码不行——少一个 null check，少一个 import，少一个 await，代码就坏了。

### 约束三：跨 subagent 通信不存在

Claude Code 的 subagent 之间无法互相通信。每个 subagent 是独立的上下文孤岛。

当两个 subagent 分别写前端和后端代码时，前端 agent 不知道后端 agent 返回了什么数据结构，后端 agent 不知道前端 agent 期望什么接口。集成必然出问题。

GitHub Issue #24932 也指出了这个问题：

> "Subagents report back to the caller only — they can't communicate with each other."

feature-dev 和 gstack 的解决方案是相同的：让所有 subagent 只做只读研究，把结果返回给主 agent。主 agent 汇总所有信息后，在自己的上下文中统一实现。主 agent 的上下文就是共享内存。

### 约束四：上下文溢出会导致会话永久崩溃

当 subagent 的结果返回主 agent 时，如果结果太大，会直接撑爆主 agent 的上下文窗口。

GitHub Issue #23463 记录了一个灾难性案例：

- 7 个并行 subagent 审计一个大型代码库
- 每个产出 15K-37K 字符的结果
- 合计 150K+ 字符涌入主 agent 的上下文
- 会话永久崩溃，反复报 "Prompt is too long"
- 无法恢复，无法 /compact，只能放弃整个会话

如果 subagent 不只是返回分析结果，还返回了它写的代码 diff——上下文消耗只会更大。

### 约束五：上下文耗尽的恶性循环

GitHub Issue #21776（讨论 feature-dev 的 self-checkpoint 功能）揭示了一个更深层的问题：

> "By Phase 5: Claude starts losing track of plan structure, skips tasks, or produces lower-quality code. Context window 75% consumed by irrelevant history."

> "Phases 5-7 involve the most complex work — integration across the full stack, wiring frontend to backend, and testing edge cases. These are exactly the phases that suffer most from context exhaustion, and exactly the phases where full context matters most. Today, they get the least."

这段话精确描述了困境：Phase 5（写代码）是最需要完整上下文的阶段，但也是上下文最少的阶段——因为前四个阶段的讨论已经消耗了 75% 的窗口。

如果把 Phase 5 交给 subagent，情况更糟：subagent 不仅没有前四个阶段的上下文，它自己的上下文窗口还要从零开始装载代码库信息。它写出来的代码，怎么可能比拥有完整上下文的主 agent 写得好？

---

## 社区的惨痛教训

不是我一个人踩了这个坑。社区的集体经验已经形成了明确的共识。

### "Subagent 是研究员，不是开发者"

AI Agents Hub 博客基于 8 亿 token 的使用经验，给出了这个判断：

> "Stop treating sub-agents like developers. Start treating them like specialized consultants who plan but don't build."

他们的实测数据：

> "Teams using sub-agents correctly see 2-3x productivity gains. Teams using them wrong burn 3-4x more tokens for worse results."

正确使用（subagent 做研究）：2-3 倍生产力提升。错误使用（subagent 写代码）：3-4 倍 token 消耗，更差的结果。

他们给出的 subagent 配置模板：

```yaml
name: codebase-analyzer
description: Scans entire codebase for implementation planning
tools: Read, Grep, Glob   # 注意：没有 Write 或 Edit
---
You analyze codebases and return implementation plans.
Never modify files. Only research and report.
```

### 上下文泄漏导致的崩溃

GitHub Issue #15191：

> "My chat with many background subagents (8), crashed with no context, and unable to /compact."

Reddit r/ClaudeAI 上的讨论：

> "When subagent is used in background, the main agent seems to always trying to poke and check the output of it, and this make the main agent context window used up quickly for nothing."

主 agent 会反复检查 subagent 的输出，每次检查都在消耗自己的上下文窗口。subagent 越多，上下文消耗越快，代码质量越差。

### Victor Dibia 的基准测试

微软研究院的 Victor Dibia 在五种 agent 配置上做了基准测试，结论是：

> "Context in agents is cumulative — every message, tool result, and model response from previous steps gets carried forward into each new LLM call. But longer tasks mean more accumulated context, and that creates three problems: the context window fills up and requests get rejected, token costs scale with context size, and model performance degrades as context grows."

他的推荐架构是 Isolation 模式——subagent 作为只读工具，结果压缩后返回主 agent，subagent 的上下文立即丢弃：

```python
sub_agent = Agent(
    name="code_reviewer",
    instructions="Read all .py files and document classes and functions.",
    tools=[read_file, list_directory],  # 只读工具
)
coordinator = Agent(
    instructions="Delegate each directory to the code_reviewer tool.",
    tools=[sub_agent.as_tool()],
)
# coordinator 的上下文保持小巧；subagent 的上下文被丢弃
```

---

## 行业正在收敛到同一个架构

不只是 feature-dev 和 gstack。主流 AI 编码工具正在不约而同地收敛到一个三层架构：

### Claude Code

```
Explorer subagent（只读）→ 主 Agent（写代码）→ Reviewer subagent（只读）
```

Phase 2 勘探、Phase 4 设计、Phase 6 审查用 subagent。Phase 5 实现用主 agent。

### Cursor 2.4

```
Background agent（研究）→ 主 Agent（写代码）→ Background agent（审查）
```

Background agent 做代码搜索和分析，主 agent 执行修改。

### OpenAI Codex

```
整个 Codex 实例本质上就是一个巨大的 subagent → 结果返回给主 IDE agent
```

Codex 在隔离的沙箱中运行，完成后把 diff 返回给 IDE 中的主 agent 审查和应用。

### Aider（不同路线）

Aider 走了另一条路：不用 subagent，而是用两个模型在同一个上下文中分工：

```
Architect model（规划，用高端模型）→ Editor model（执行，用快速模型）
```

两个模型共享同一个上下文窗口，所以不存在上下文隔离问题。这从另一个角度验证了同一个论点：写代码的人必须知道前面讨论了什么。

三种不同的实现，同一个结论：**研究和实现必须分离，但实现必须拥有研究的完整上下文。**

---

## 那正确的做法是什么

从 feature-dev 和 gstack 的源码设计中，可以提取出一条清晰的架构原则：

```
Subagent 的职责边界：
├── ✅ 搜索代码库（只读）
├── ✅ 分析架构模式（只读）
├── ✅ 审查代码 diff（只读）
├── ✅ 生成设计文档（写入文件，不是直接写代码）
├── ❌ 修改源代码文件
├── ❌ 执行重构操作
└── ❌ 创建新项目脚手架

主 Agent 的职责边界：
├── ✅ 读取 subagent 返回的研究结果
├── ✅ 读取 subagent 写入的设计文档
├── ✅ 汇总所有上下文后写代码
├── ✅ 执行重构
└── ✅ 与用户交互确认方向
```

文件系统是唯一的共享内存。subagent 把研究结果写成 .md 文件，主 agent 读取这些文件后实现。这就是为什么 gstack 的 Plan 阶段产出的是设计文档而不是代码——文档是跨上下文的桥梁，代码不是。

### 如果你的工作流用了 subagent 写代码

检查一下你的 AI 编码工作流中是否有这样的配置：

1. **subagent 的工具集包含 Write/Edit** — 这是最明显的红线。subagent 的工具集应该和 feature-dev 的三个 agent 一样：`Glob, Grep, LS, Read, NotebookRead, WebFetch`。没有写操作。

2. **"全自动"流程跳过了人工确认点** — feature-dev 有五个用户决策点（Phase 1/3/4/5/6），gstack 的每一步都有人工检查。全自动意味着没有人在 loop 里验证 subagent 的产出质量。

3. **subagent 看不到前序讨论的上下文** — 如果 subagent 接收的只是一段任务描述，而不是完整的对话历史（它确实接收不到），那它写的代码不可能符合前面讨论的设计意图。

---

## 回到那个周六

我那个周六的加班，本质上是一个上下文工程问题的代价。

subagent 不知道我们之前讨论过的接口约定，不知道某些模块之间的依赖关系是有意的（不是设计缺陷），不知道有些"冗余"代码是为了向后兼容。它看到代码，按照自己的理解重构了一遍——技术上没问题，但在项目上下文里全是错的。

feature-dev 和 gstack 的设计者们显然也踩过同样的坑。他们的解决方案不是什么高深的算法，而是一条朴素的工程原则：

**让知道最多的人做最需要知道最多的事。**

在 AI 编码工作流中，知道最多的是主 agent——它经历了整个对话，知道所有决策的 why。最需要知道最多的是写代码——因为代码是决策的最终落地。把这两件事分开，就是上下文工程中最贵的错误。

subagent 是好工具。但好工具用错了地方，比没有工具更贵。
