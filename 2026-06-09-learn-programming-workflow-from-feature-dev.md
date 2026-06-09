---
title: "跟着 feature-dev 学写自己的编程工作流"
description: "深入 Claude Code /feature-dev 的源码，逐行拆解它的 prompt 写作手法、Agent 权限设计、阶段编排逻辑，学习如何为自己的场景设计 AI 编程工作流。"
---

# 跟着 feature-dev 学写自己的编程工作流

Claude Code 的 `/feature-dev` 插件只有 7 个文件、不到 600 行源码，但它展示了一套成熟的 AI 编程工作流应该怎么设计：怎么编排阶段、怎么分配 Agent 权限、怎么写 prompt 让 Agent 各司其职。

这篇文章不教你怎么用 `/feature-dev`（那是使用指南的事），而是拆开源码，逐行讲清楚**它为什么这么写**，以及**你从中学到什么来设计自己的工作流**。

## 文件结构：少即是多

feature-dev 的文件结构极简：

```
plugins/feature-dev/
├── .claude-plugin/
│   └── plugin.json        # 插件元数据
├── commands/
│   └── feature-dev.md     # 主命令：七阶段编排逻辑
├── agents/
│   ├── code-explorer.md   # 勘探 Agent
│   ├── code-architect.md  # 架构 Agent
│   └── code-reviewer.md   # 审查 Agent
├── README.md              # 文档
└── LICENSE                # Apache-2.0
```

四个 Markdown 文件定义了全部行为。没有 Python、没有 TypeScript、没有复杂的配置——全是 prompt。

**这是第一个值得学习的设计决策：用 prompt 而非代码来编排 AI 工作流。** Markdown prompt 可读性好、可 fork 性好、修改门槛低。你不需要写一个框架，只需要写好 prompt 然后让 Claude Code 的插件系统去执行。

## plugin.json：最小化的元数据

```json
{
  "name": "feature-dev",
  "description": "Comprehensive feature development workflow with specialized agents
                  for codebase exploration, architecture design, and quality review",
  "author": {
    "name": "Anthropic",
    "email": "support@anthropic.com"
  }
}
```

三行。没有版本依赖、没有运行时配置、没有 hook 注册。Claude Code 的插件系统只需要知道"这个插件叫什么、做什么"，剩下的全靠 prompt 驱动。

---

## 主命令文件：七阶段的编排艺术

`commands/feature-dev.md` 是整个工作流的大脑。125 行，定义了七个阶段和贯穿全程的核心原则。我们逐段分析。

### Frontmatter：声明式配置

```yaml
---
description: Guided feature development with codebase understanding and architecture focus
argument-hint: Optional feature description
---
```

`argument-hint` 是个巧妙的设计。它告诉用户"你可以传参数也可以不传"，降低了使用门槛。Claude Code 的命令系统根据这个 hint 在 UI 上给出提示。

### Core Principles：五个原则定全局

主命令的开头不是直接跳到 Phase 1，而是先列出五条核心原则：

```markdown
## Core Principles

- **Ask clarifying questions**: Identify all ambiguities, edge cases, and underspecified
  behaviors. Ask specific, concrete questions rather than making assumptions. Wait for
  user answers before proceeding with implementation.
- **Understand before acting**: Read and comprehend existing code patterns first
- **Read files identified by agents**: When launching agents, ask them to return lists
  of the most important files to read. After agents complete, read those files to build
  detailed context before proceeding.
- **Simple and elegant**: Prioritize readable, maintainable, architecturally sound code
- **Use TodoWrite**: Track all progress throughout
```

**写作手法解析：** 这五条原则不是装饰，它们在每个 Phase 中被反复引用和强化。比如"Read files identified by agents"这条原则，在 Phase 2 中被具体化为：

```markdown
2. Once the agents return, please read all files identified by agents to build
   deep understanding
```

**这是一个关键的工作流设计模式：顶层原则 + 阶段细化。** 原则告诉 Agent "大的方向是什么"，阶段指令告诉它"在这个具体步骤里怎么做"。如果你只写阶段指令不写原则，Agent 在遇到歧义时会自行决策；有了原则，它的决策空间被收窄到你期望的范围内。

### `$ARGUMENTS` 变量：灵活的输入设计

Phase 1 的第一行：

```markdown
Initial request: $ARGUMENTS
```

`$ARGUMENTS` 是 Claude Code 命令系统的内置变量，用户输入 `/feature-dev Add caching` 时，`$ARGUMENTS` 就是 `Add caching`。用户只输入 `/feature-dev`，它就是空的。

这个设计的巧妙之处在于：**同一个命令文件同时处理有参数和无参数两种情况。** Phase 1 的逻辑是：

```markdown
2. If feature unclear, ask user for:
   - What problem are they solving?
   - What should the feature do?
   - Any constraints or requirements?
```

有参数就直接用，没参数就问。不需要写两个命令。

### Phase 2 的 Agent 启动模板

Phase 2 的源码给出了 2-3 个 `code-explorer` agent 的 prompt 模板：

```markdown
**Example agent prompts**:
- "Find features similar to [feature] and trace through their implementation comprehensively"
- "Map the architecture and abstractions for [feature area], tracing through the code comprehensively"
- "Analyze the current implementation of [existing feature/area], tracing through the code comprehensively"
- "Identify UI patterns, testing approaches, or extension points relevant to [feature]"
```

注意这里写的是 "Example agent prompts" 而不是固定模板。这意味着主 Agent 会根据具体任务灵活调整每个 explorer 的探索方向。**这是 prompt 编排和硬编码编排的核心区别——你定义的是意图，不是路径。**

每个 prompt 都以 "comprehensively" 结尾。这不是冗余，而是刻意设计。在 AI prompt 中，重复强调"要全面"确实能提高输出的完整度。

### Phase 3 的 CRITICAL 标记

```markdown
**CRITICAL**: This is one of the most important phases. DO NOT SKIP.
```

为什么单独给 Phase 3 加 CRITICAL 标记？因为 AI 模型有一种倾向——当上下文已经看起来"足够清楚"时，它会倾向于跳过澄清步骤直接往下走。这个标记是在对抗这种倾向。

后面还有一条防御性指令：

```markdown
If the user says "whatever you think is best", provide your recommendation and get
explicit confirmation.
```

**写作手法解析：** 这是"最坏情况防御"的 prompt 写法。它预判了用户可能的敷衍回答（"你看着办"），然后规定即使面对敷衍，也必须给出建议并获得确认。这确保了流程的安全性——不会因为没有用户输入就默默做决定。

### Phase 4 的推荐机制

```markdown
3. Present to user: brief summary of each approach, trade-offs comparison,
   **your recommendation with reasoning**, concrete implementation differences
4. **Ask user which approach they prefer**
```

注意第 3 步要求"your recommendation with reasoning"——不是只列方案让用户选，而是先给出自己的判断和理由。

**这是一个微妙但重要的工作流设计：AI 不是信息搬运工，它应该有观点。** 用户面对三个方案时，如果没有推荐，决策压力全在用户身上。有了推荐 + 理由，用户只需要判断"AI 的理由是否适用于我的场景"，这比从零判断容易得多。

### Phase 5 的权限隔离

```markdown
**DO NOT START WITHOUT USER APPROVAL**
```

Phase 5 是七个阶段中唯一没有启动 subagent 的阶段。为什么？

因为前四个阶段已经完成了所有准备工作——代码库勘探完了、需求澄清了、架构方案选好了。实现阶段就是一个线性的执行过程，不需要并行。更重要的是，**subagent 的上下文是隔离的**——它看不到 Phase 2 的勘探结果和 Phase 4 的架构决策。写代码需要完整的项目上下文，只有主 Agent 具备这个条件。

**写作手法解析：** 这里有一个隐含的工作流设计原则——**信息流决定执行权。** 谁拥有最完整的上下文，谁就负责最关键的执行。subagent 适合做专项分析（窄而深），主 Agent 适合做综合实现（宽而全）。

---

## 三个 Agent 的 Prompt 工程

三个 agent 文件是 prompt 工程的精品案例。每个文件不到 50 行，但结构严谨、角色明确。

### Agent Frontmatter：统一格式

三个 agent 共享同一个 frontmatter 格式：

```yaml
---
name: code-explorer / code-architect / code-reviewer
description: [一句话描述职责]
tools: Glob, Grep, LS, Read, NotebookRead, WebFetch, TodoWrite, WebSearch, KillShell, BashOutput
model: sonnet
color: yellow / green / red
---
```

**三个设计决策值得关注：**

**1. 工具集完全相同。** 三个 agent 都是只读权限——没有 `Write` 和 `Edit`。这不是疏忽，而是刻意的权限隔离。subagent 只做分析和建议，写代码的权力只在 Phase 5 的主 Agent 手中。

**2. 模型都是 sonnet。** 三个 agent 都用 Sonnet 而不是 Opus。原因很务实：这些任务是偏广度的分析和审查，不需要深度推理。Sonnet 更快、更便宜，对这类任务绰绰有余。**这是一个重要的工程判断：不是每个环节都需要最强的模型。**

**3. 颜色区分。** yellow、green、red——不是装饰，是在 Claude Code 的 UI 中帮助用户一眼区分不同 agent 的输出。当三个 agent 并行返回结果时，颜色让信息层次一目了然。

### code-explorer.md：四层分析框架

```markdown
**1. Feature Discovery**
- Find entry points (APIs, UI components, CLI commands)
- Locate core implementation files
- Map feature boundaries and configuration

**2. Code Flow Tracing**
- Follow call chains from entry to output
- Trace data transformations at each step
- Identify all dependencies and integrations
- Document state changes and side effects

**3. Architecture Analysis**
- Map abstraction layers (presentation → business logic → data)
- Identify design patterns and architectural decisions
- Document interfaces between components
- Note cross-cutting concerns (auth, logging, caching)

**4. Implementation Details**
- Key algorithms and data structures
- Error handling and edge cases
- Performance considerations
- Technical debt or improvement areas
```

**写作手法解析：** 四层分析从宏观到微观，形成了一个固定的分析框架。这不是随意列的四个方向，而是一个有逻辑递进关系的序列：先找到入口 → 追踪调用链 → 理解架构 → 深入细节。

这种"分析框架"式的 prompt 写法，比"请全面分析代码"要有效得多。它给 Agent 一个明确的思维路径，减少遗漏。

输出要求的最后一条也值得注意：

```markdown
- List of files that you think are absolutely essential to get an understanding
  of the topic in question
```

这一条是整个 Phase 2 的关键输出。主流程拿到这个列表后，会逐一读取这些文件来建立深度上下文。**explorer agent 的价值不在于它写了多少分析文字，而在于它找到了哪些文件。**

### code-architect.md：果断决策 vs 多方案对比

这里有一个有趣的张力。主命令（`feature-dev.md`）要求启动 2-3 个 architect agent 各出一个方案，而每个 architect agent 自身被要求：

```markdown
Make decisive choices - pick one approach and commit.
```

以及输出指导中的：

```markdown
Make confident architectural choices rather than presenting multiple options.
```

**写作手法解析：** 这是"分层决策"的设计。每个 architect agent 给出一个确定的方案（带理由和 trade-off），主流程把 2-3 个 agent 的不同方案汇总对比，然后让用户选。

如果每个 architect 都列一堆选项，主流程汇总时就是 6-9 个方案的排列组合，用户根本选不过来。让每个 agent "果断出牌"，主流程汇总 2-3 个清晰方案，这才是可操作的。

输出指导也很具体：

```markdown
- **Patterns & Conventions Found**: Existing patterns with file:line references
- **Architecture Decision**: Your chosen approach with rationale and trade-offs
- **Component Design**: Each component with file path, responsibilities, dependencies,
  and interfaces
- **Implementation Map**: Specific files to create/modify with detailed change descriptions
- **Data Flow**: Complete flow from entry points through transformations to outputs
- **Build Sequence**: Phased implementation steps as a checklist
- **Critical Details**: Error handling, state management, testing, performance,
  and security considerations
```

七个输出维度，每个都是具体的、可操作的。注意它要求 `file:line` 引用和具体的文件路径——不是空泛的"在 service 层加一个方法"，而是"在 `src/auth/AuthService.ts` 里加一个 `refreshToken()` 方法"。

### code-reviewer.md：置信度过滤

这是三个 agent 中设计最精巧的一个。核心机制是置信度评分：

```markdown
Rate each potential issue on a scale from 0-100:

- **0**: Not confident at all. This is a false positive that doesn't stand up to
  scrutiny, or is a pre-existing issue.
- **25**: Somewhat confident. This might be a real issue, but may also be a false
  positive. If stylistic, it wasn't explicitly called out in project guidelines.
- **50**: Moderately confident. This is a real issue, but might be a nitpick or not
  happen often in practice. Not very important relative to the rest of the changes.
- **75**: Highly confident. Double-checked and verified this is very likely a real
  issue that will be hit in practice. The existing approach is insufficient. Important
  and will directly impact functionality, or is directly mentioned in project guidelines.
- **100**: Absolutely certain. Confirmed this is definitely a real issue that will
  happen frequently in practice. The evidence directly confirms this.

**Only report issues with confidence ≥ 80.**
```

**写作手法解析：** 传统 linter 和静态分析工具的最大问题是 noise——报一大堆问题，真正重要的淹没在噪声里。置信度过滤是一个优雅的解法：让 AI 自己评估"我有多确定这是真问题"，然后只报高确定性的。

评分标准的措辞也值得学习。每个分数档不是简单的数字，而是附带了判断标准：

- 0 分："false positive" 或 "pre-existing issue"——明确排除了误报和已有问题
- 50 分："might be a nitpick"——承认有些问题是真的但不重要
- 75 分："double-checked and verified"——要求二次确认才给高分

这套标准实际上是在训练 reviewer agent 的判断力——教它区分"可能有问题"和"确定有问题"。

**另一个细节：** reviewer 的职责定义中特别提到了 `CLAUDE.md`：

```markdown
Your primary responsibility is to review code against project guidelines in CLAUDE.md
with high precision to minimize false positives.
```

这让 reviewer 不是泛泛地审查代码质量，而是对照项目自己的规范来审查。如果你的 `CLAUDE.md` 里写了"所有 API 必须有 rate limiting"，reviewer 就会检查这条。

---

## 从 feature-dev 学到的工作流设计模式

拆解完源码，我总结出几个通用的工作流设计模式，你可以用它们来设计自己的 AI 编程工作流。

### 模式一：阶段化 + 人工卡点

**核心思想：** 把大任务拆成小阶段，每个阶段之间有人工决策点。

feature-dev 的五个卡点：需求确认（Phase 1）→ 模糊点澄清（Phase 3）→ 方案选择（Phase 4）→ 开工批准（Phase 5）→ 审查处置（Phase 6）。

**怎么应用到你的工作流：** 任何超过 10 分钟的任务，都应该有至少一个人工卡点。卡点不是不信任 AI，而是确保方向正确——方向错了，做得越快越糟糕。

### 模式二：subagent 只读 + 主 Agent 执行

**核心思想：** 分析和执行分离。subagent 只做只读分析，主 Agent 负责所有修改。

feature-dev 三个 agent 的工具集都没有 `Write` 和 `Edit`。

**怎么应用到你的工作流：** 当你需要 AI 先调研再执行时，把调研和执行分成两个 agent。调研 agent 的只读权限天然防止了"还没想清楚就改了一堆文件"的问题。

### 模式三：并行专项 + 主流程汇总

**核心思想：** 多个 agent 并行从不同角度看同一个问题，主流程汇总结果。

feature-dev 的 Phase 2（2-3 个 explorer 并行勘探）、Phase 4（2-3 个 architect 并行设计）、Phase 6（3 个 reviewer 并行审查）都用了这个模式。

**怎么应用到你的工作流：** 当一个分析任务有多个维度（性能、安全、可用性），启动多个 agent 各看一个维度。主流程负责整合，避免遗漏。

### 模式四：置信度过滤

**核心思想：** 让 AI 自己评估输出的可靠度，只报高置信度的结果。

code-reviewer 的 80 分门槛是这个模式的典范。

**怎么应用到你的工作流：** 任何需要 AI 报告问题的工作流（代码审查、安全扫描、测试生成），都应该加上置信度过滤。它把信噪比从 1:10 提升到 1:1。

### 模式五：顶层原则 + 阶段细化

**核心思想：** 在文件开头声明全局原则，在每个阶段用具体指令细化。

feature-dev 的五条 Core Principles 贯穿全文，每个 Phase 都引用和强化这些原则。

**怎么应用到你的工作流：** 不要只在每个阶段写指令，先写清楚"为什么要这样做"。当 Agent 遇到模糊地带时，原则比指令更可靠——指令覆盖不了所有情况，原则可以。

### 模式六：果断决策 vs 多方案汇总

**核心思想：** 每个 subagent 出一个确定方案，主流程汇总多个 agent 的方案让用户选。

code-architect 被要求 "pick one approach and commit"，但主流程启动 2-3 个 architect。

**怎么应用到你的工作流：** 不要让一个 agent 列出一堆方案——它会越列越犹豫。让每个 agent 果断出一个方案，然后你来汇总对比。果断出牌的质量通常高于犹豫列选项。

---

## 动手：设计你自己的编程工作流

基于以上模式，这里给出一个自定义工作流的框架，你可以根据自己的场景修改。

### 示例：`/api-migration` — API 迁移工作流

假设你经常做 API 迁移（旧接口 → 新接口），可以这样设计：

```markdown
---
description: Guided API migration with compatibility verification
argument-hint: API endpoint or module to migrate
---

# API Migration

You are helping migrate an API from an old implementation to a new one.

## Core Principles
- **Backward compatible**: Every change must preserve existing behavior
- **Test first**: Write tests for old behavior before changing anything
- **Incremental**: Migrate one endpoint at a time, verify after each

## Phase 1: Scope
Identify the exact endpoints/methods to migrate.
Ask user to confirm the scope.

## Phase 2: Compatibility Analysis
Launch 2 migration-analyzer agents:
- "Map all callers of [old API] across the codebase"
- "Document the exact input/output contract of [old API]"
Read all identified caller files.

## Phase 3: Test Coverage Check
Before touching any code:
1. Verify existing tests cover old behavior
2. If not, write characterization tests first
3. **DO NOT PROCEED without test coverage confirmation**

## Phase 4: Implementation
Migrate one endpoint at a time.
After each: run tests, show diff, wait for user approval.

## Phase 5: Regression Verification
Launch migration-verifier agent:
- "Compare old and new API behavior across all test cases"
- "Identify any behavioral differences"

## Phase 6: Summary
List migrated endpoints, test results, breaking changes (if any).
```

### 对应的 Agent 文件示例

```markdown
---
name: migration-analyzer
description: Maps API callers and documents behavior contracts
tools: Glob, Grep, LS, Read, NotebookRead, WebFetch, TodoWrite
model: sonnet
color: blue
---

You are an API migration specialist. Your job is to understand how an
existing API is used across the codebase.

## Analysis Approach

**1. Caller Discovery**
- Find every file that imports/calls the target API
- Include tests, documentation, and configuration files
- Note indirect callers (callers of callers)

**2. Contract Documentation**
- Input: parameter types, validation rules, defaults
- Output: response format, status codes, error shapes
- Side effects: database changes, event emissions, cache updates

**3. Dependency Mapping**
- What does this API depend on? (other services, DB tables, env vars)
- What depends on this API? (direct and indirect callers)

## Output
- Complete caller list with file:line references
- API contract specification
- Dependency graph (text-based)
- List of files that must be read to understand full usage
```

---

## 总结

feature-dev 的源码不到 600 行，但它展示了一套成熟的 AI 编程工作流应该具备的品质：

1. **阶段清晰**——每个阶段有明确的目标和输出
2. **权限隔离**——分析和执行分离，只读 agent 不写代码
3. **人工卡点**——五个决策点确保人始终在 loop 里
4. **并行专项**——多个 agent 从不同角度看问题，主流程汇总
5. **置信度过滤**——只报高确定性的问题，减少噪声
6. **果断决策**——每个 agent 出一个方案，主流程做选择题

这些不是 feature-dev 独有的——它们是 AI 编程工作流的通用设计模式。理解了这些模式，你就可以为自己的场景设计出同样精良的工作流。

不需要写代码，只需要写好 prompt。

---

*源码：[github.com/anthropics/claude-code/tree/main/plugins/feature-dev](https://github.com/anthropics/claude-code/tree/main/plugins/feature-dev)*

*姊妹篇：《Claude Code /feature-dev 使用指南：七阶段流水线式功能开发》——从安装到实战，手把手教你用 /feature-dev。*
