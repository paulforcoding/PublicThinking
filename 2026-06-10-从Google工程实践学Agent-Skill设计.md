---
title: "从 Google 工程实践学 Agent Skill 设计：六个 Prompt 工程模式的深度拆解"
description: "拆开 AgentAra engineering skill 套件的源码，逐行分析它的 prompt 写作手法、触发条件设计、工作流编排逻辑，提炼出六个可复用的 Agent Skill 设计模式。"
---

# 从 Google 工程实践学 Agent Skill 设计：六个 Prompt 工程模式的深度拆解

AgentAra 的 `skills/engineering` 套件有六个 skill 文件，总计约 600 行 prompt，却覆盖了一个工程团队日常最核心的协作流程：代码审查、PR 拆分、变更描述、评论撰写、反馈处理、紧急变更。

这篇文章不教你怎么用这些 skill（那是使用指南的事），而是拆开它们的源码，逐行讲清楚**它们为什么这么写**，以及**你从中学到什么来设计自己的 Agent skill**。

## 文件结构：六个文件，一个闭环

```
skills/engineering/
├── README.md                          # 索引：六个 skill 的关系总览
├── engineering-code-review/SKILL.md   # 代码审查（核心 skill）
├── engineering-review-comments/SKILL.md  # 审查评论撰写
├── engineering-small-prs/SKILL.md     # PR 拆分策略
├── engineering-change-descriptions/SKILL.md  # 变更描述撰写
├── engineering-review-feedback/SKILL.md  # 处理审查反馈
└── engineering-emergency-changes/SKILL.md  # 紧急变更决策
```

六个 Markdown 文件，每个文件一个独立能力。没有 Python、没有 TypeScript、没有复杂的配置——全是 prompt。

**这是第一个值得学习的设计决策：用 Markdown prompt 而非代码来定义 Agent 行为。** 可读性好、可 fork 性好、修改门槛低。你不需要写一个框架，只需要写好 prompt 然后让 Agent 去执行。

## README：关系图谱而非目录

README 不是一个简单的文件列表。它先用一句话定义整个套件的价值：

> Practical software engineering workflows for code review, small change planning, PR communication, reviewer feedback, and emergency changes.

然后用列表展示每个 skill，每个条目包含：skill 名称（带链接）+ 一句话描述职责。

```markdown
- **[engineering-code-review](...)** — Review code changes for code health,
  design, functionality, complexity, tests, maintainability, specialist risk,
  and approval risk.
```

最后标注来源：

> These skills are adapted from [Google Engineering Practices](https://google.github.io/eng-practices/) under [CC-BY 3.0](https://creativecommons.org/licenses/by/3.0/).

**写作手法解析：** 来源标注不是客气话。它做了三件事：① 建立可信度（"这不是我瞎编的，是 Google 的工程实践"）；② 明确边界（"我改编自 CC-BY 3.0 的内容"）；③ 给读者一个深入学习的入口。

---

## Frontmatter 的触发条件设计

每个 skill 的 frontmatter 是整个 skill 设计中最重要的部分。它决定了 Agent 在什么时机自动加载这个 skill。我们逐个分析。

### 三段式描述模式

每个 skill 的 `description` 都遵循同一个三段式结构：

```yaml
description: >
  Use when [场景描述].
  TRIGGER on [触发关键词列表].
  DO NOT TRIGGER for [排除条件列表].
```

以 engineering-code-review 为例：

```yaml
description: >
  Use when reviewing code, pull requests, patches, CLs, diffs, or proposed
  implementations for engineering quality...
  TRIGGER on "review this PR", "code review", "LGTM?", "approve?",
  "is this code/diff/change safe to merge/deploy?", and local diff reviews.
  DO NOT TRIGGER for PR descriptions, PR splitting, author review-feedback
  responses, or non-code safety questions unless code review is requested.
```

**写作手法解析：** 三段式描述本质上是一个路由规则：

1. **Use when** — 宽泛的场景描述，帮助 Agent 理解"这个 skill 大致是干什么的"
2. **TRIGGER on** — 精确的触发关键词，帮助 Agent 判断"用户的请求是否匹配"
3. **DO NOT TRIGGER for** — 明确的排除条件，防止误触发

为什么需要排除条件？因为六个 skill 之间有语义重叠。"review this PR" 可能触发 code-review，也可能触发 review-comments 或 review-feedback。排除条件让 Agent 知道"写 PR 描述不找 code-review，找 change-descriptions"。

**这是一个关键的设计模式：正向触发 + 负向排除 = 精确路由。** 只有正向触发，误触发率会很高；加上负向排除，路由精度大幅提升。

### 六个 skill 的路由关系

如果把六个 skill 的触发/排除条件画成网络图：

```
"review this PR"     → code-review（排除 PR descriptions, PR splitting）
"write review comment" → review-comments（排除 full code review）
"split this PR"      → small-prs（排除 full code review, PR descriptions）
"write PR description" → change-descriptions（排除 code review, PR splitting）
"respond to review"  → review-feedback（排除 reviewer comment writing）
"emergency change"   → emergency-changes（排除 ordinary urgency）
```

每个 skill 的排除条件都指向其他 skill 的触发条件。**这不是六个独立的 skill，而是一个互相引用的路由网络。** Agent 在收到用户请求后，通过触发/排除条件精确地加载对应的那一个。

---

## Prompt 结构分析：每个 skill 的四层架构

六个 skill 共享同一个内部结构，只是内容不同：

```
1. 标准/定义层 — "什么是好的 X"
2. 工作流层   — "做 X 的步骤是什么"
3. 检查清单层 — "做完 X 后检查什么"
4. 模板层     — "X 的输出格式是什么"
```

加上头尾两个固定组件：

```
0. 常见错误层 — "做 X 时容易犯什么错"（放在末尾）
```

这不是随意的排列，而是有逻辑递进关系的序列：先理解标准 → 按步骤执行 → 检查执行质量 → 用模板输出 → 最后检查有没有犯常见错误。

### 第一层：标准/定义 — 定基调

每个 skill 的第一段内容都是定义"什么是好的 X"。这个定义不是空泛的原则，而是具体的判断标准。

**engineering-code-review 的标准：**

```markdown
## Review Standard
- Favor approval when the change definitely improves the code health
  of the touched system, even if it is not perfect.
- Do not approve changes that make the system worse unless this is
  a true emergency and the follow-up cleanup is explicit.
- Prefer technical facts, data, existing project standards, and durable
  design principles over taste.
- Treat style guides and automated formatters as authorities.
  Personal style preferences are nits.
- If several designs are defensible, accept the author's preference
  unless it harms code health.
```

**写作手法解析：** 这五条标准的措辞精确到了可执行的程度。不是"审查要全面"（模糊），而是"当改动确实改善了代码健康度时，倾向于批准"（具体）。

特别值得注意的是第四条："Treat style guides and automated formatters as authorities. Personal style preferences are nits." 这一条把"风格争议"这个人类 code review 中最浪费时间的话题，直接用一条规则消解了——格式化工具说了算，个人偏好是 NIT 级别。

**engineering-small-prs 的定义：**

```markdown
## Definition of a Good Small PR
A good PR:
- Makes one self-contained change.
- Includes the tests needed to validate that change.
- Leaves the system working after merge.
- Gives reviewers enough context in the PR, description, existing code,
  or earlier reviewed PRs.
- Is not so tiny that the behavior cannot be understood.
- Separates large refactors from behavior changes.
```

注意最后一条的反向定义："Is not so tiny that the behavior cannot be understood." 很多人以为"越小越好"，这个定义纠正了这个误解——小不是行数少，而是概念上聚焦。

**engineering-emergency-changes 的双向定义：**

```markdown
## Emergency Gate
Treat a change as an emergency only when at least one condition is true:
- It prevents a major launch from failing...
- It fixes a bug significantly affecting production users...
- It addresses a pressing legal or compliance issue...
- It closes a major security vulnerability...

These are usually not emergencies by themselves:
- Wanting to launch sooner.
- A feature has taken a long time...
- Reviewers are asleep, away, or in another time zone.
- It is late Friday.
- A manager wants it done today for a soft deadline.
```

**写作手法解析：** 这是六个 skill 中唯一一个同时使用正向定义和反向定义的。为什么？因为"紧急"这个概念的误判率极高。人类在压力下倾向于把所有事情都定义为"紧急"。只有正向定义不够——你需要反向定义来预判和拦截常见的误判。

> Most deadlines are soft. Do not sacrifice code health for a soft deadline.

这句话是整个 emergency-changes skill 的灵魂。它不是在描述事实，而是在**设定价值观**——代码健康比软性截止日期重要。

### 第二层：工作流 — 步骤化执行

每个 skill 都有一个编号的工作流步骤列表。步骤数量从 6 到 10 不等，但都有一个共同特征：**每一步都是可执行的动词开头的指令**。

**engineering-code-review 的 8 步工作流：**

```markdown
1. Establish scope.
2. Gather review material.
3. Take the broad view first.
4. Review the main design before details.
5. Review every human-written line in your assigned scope.
6. Check the change against the review checklist below.
7. Check verification evidence.
8. Make a verdict.
```

**写作手法解析：** 步骤的顺序是有意的。第 3 步"Take the broad view first"和第 4 步"Review the main design before details"都在对抗一种常见倾向——直接跳到细节。如果第 3 步就发现"这个改动不应该存在"，后面 5 步都不需要做了。

第 5 步有一个关键限定："Review every **human-written** line." 生成的文件、大型数据文件、机械输出可以抽样。这是务实的——Agent 不需要逐行审查自动生成的代码。

**engineering-review-feedback 的 6 步工作流：**

```markdown
1. Pause before responding if the comment feels frustrating.
2. Identify what the reviewer is asking for.
3. Classify each comment:
   - Code change required.
   - Test change required.
   - Documentation or comment change required.
   - Clarification needed.
   - Disagreement to discuss.
   - Non-blocking suggestion.
4. Prefer improving the artifact over explaining in the review tool.
5. Run the relevant verification.
6. Reply with what changed, why, and how it was tested.
```

第 1 步"Pause before responding if the comment feels frustrating"——这不是一个技术步骤，而是一个**情绪管理步骤**。它预判了人类在收到严厉评论时的本能反应（愤怒/防御），然后要求先暂停。这个步骤对 Agent 同样重要——Agent 在处理负面反馈时也可能"过度防御"（写一大段解释而不是改代码）。

第 4 步"Prefer improving the artifact over explaining in the review tool"是整个 review-feedback skill 的核心原则。它的含义是：**改代码比写解释好。** 如果 reviewer 看不懂你的代码，第一反应不应该是"让我在 review thread 里解释一下"，而是"让我把代码改得更清晰"。

### 第三层：检查清单 — 防遗漏

**engineering-code-review 的 16 项检查清单是整个套件中最长的：**

```markdown
## Review Checklist
- Design: Does the change belong here?
- Functionality: Does it do what the author intends?
- Edge cases: Are failures, empty states, retries, idempotency... considered?
- Complexity: Is it understandable quickly?
- Tests: Are unit, integration, or end-to-end tests appropriate?
- Test quality: Are assertions meaningful, focused, deterministic?
- Naming: Do names communicate purpose?
- Comments: Do comments explain why?
- Documentation: Are READMEs, API docs updated?
- Style and consistency: Does the change follow project conventions?
- Context: Does the change improve the system around it?
- Operations: Are deployment, migration, observability, rollback adequate?
- Security: Are authorization, authentication, secrets handled?
- Privacy: Does the change avoid unnecessary collection of sensitive data?
- Accessibility: For UIs, are keyboard access, semantics, focus preserved?
- Internationalization: Are locale, timezone, encoding handled?
```

**写作手法解析：** 每一项不是一个单词，而是一个**问句**。不是"Security"（模糊），而是"Are authorization, authentication, secrets, injection, dependency, deserialization, and privilege-boundary risks handled?"（具体到 Agent 可以逐项检查）。

清单的最后有一条隐含的兜底：

> Escalate to a specialist when you cannot confidently assess security, privacy, accessibility, internationalization, concurrency, infrastructure, data migration, or domain-specific correctness.

这是一个**自知之明条款**——告诉 Agent "如果你不确定，不要瞎说，去找专家"。

### 第四层：模板 — 标准化输出

每个 skill 都有一个或多个输出模板。模板的价值不在限制创造力，而在**降低启动成本和提高输出一致性**。

**engineering-review-comments 的评论模板：**

```text
SEVERITY: Observation about the code.

Why this matters: engineering reason, user impact, maintainability impact,
or violated project rule.

Requested action: concrete fix, question, or decision the author needs to make.
```

三段式：观察 → 原因 → 请求。这不是建议，而是强制格式。它对抗的是人类写评论时的常见模式——只说"这里有问题"不说为什么有问题、怎么改。

**engineering-change-descriptions 提供了四种模板：**

标准 PR、小改动、重构、机械变更——每种场景一个模板。这意味着 Agent 在写 PR 描述时，不需要从白纸开始，它先判断属于哪类场景，然后用对应模板填。

**engineering-small-prs 的拆分序列模板：**

```markdown
## Goal
[Final outcome]

## Proposed PR Sequence
| # | PR | Purpose | Depends On | Tests | Reviewer Notes |
|---|----|---------|------------|-------|----------------|
| 1 | ... | ... | None | ... | ... |
| 2 | ... | ... | 1 | ... | ... |
```

表格式的模板特别适合规划类任务——它强迫 Agent 思考每个 PR 的依赖关系和测试覆盖。

### 第五层：常见错误 — 防御性编程

每个 skill 的末尾都有一个 "Common Mistakes" 列表。这些不是泛泛的"注意事项"，而是**从 Google 工程实践中提炼的具体踩坑记录**。

**engineering-code-review 的常见错误：**

```markdown
- Blocking approval on preference rather than code health.
- Reviewing only the changed lines when the surrounding function
  or module makes the change unsafe.
- Accepting complex code because the author explained it only
  in the review thread.
- Letting "clean it up later" pass when the change itself
  introduces the complexity.
- Requesting large rewrites late instead of flagging design
  problems early.
- Delaying the author when a quick broad response would unblock
  useful work.
```

**写作手法解析：** 每一条都描述了一个具体的、可识别的行为模式。不是"审查不全面"（模糊），而是"只审查改动的行，不看周围函数是否让改动不安全"（具体到 Agent 可以对号入座）。

第三条特别精妙："Accepting complex code because the author explained it only in the review thread." 这抓住了一个非常常见的现象——作者在 review thread 里写了很长解释，reviewer 看懂了就通过了。但下一个读代码的人看不到 review thread，他面对的还是一段看不懂的代码。

**engineering-review-feedback 的常见错误：**

```markdown
- Replying in anger or with sarcasm.
- Explaining confusing code only in the review thread.
- Treating every reviewer suggestion as mandatory without clarifying severity.
- Saying "will clean up later" when the current change introduces the complexity.
- Making broad unrelated refactors while addressing a narrow review comment.
```

第一条"Replying in anger or with sarcasm"和工作流第 1 步"Pause before responding"呼应。常见错误层在工作流层之后，起到了强化和提醒的作用。

---

## 六个 Prompt 工程模式

从六个 skill 中提炼出的可复用设计模式：

### 模式一：三段式触发路由

```yaml
description: >
  Use when [宽泛场景].
  TRIGGER on [精确关键词].
  DO NOT TRIGGER for [排除条件].
```

**核心思想：** 用正向触发 + 负向排除来实现精确的 skill 路由。

**适用场景：** 任何有多个 skill 需要路由的套件。skill 之间的语义重叠越多，排除条件越重要。

**关键技巧：** 排除条件应该指向其他 skill 的触发条件。比如 code-review 的排除条件 "PR descriptions" 对应 change-descriptions 的触发条件 "write PR description"。

### 模式二：四层 Prompt 架构

```
标准/定义 → 工作流步骤 → 检查清单 → 输出模板
```

**核心思想：** 先告诉 Agent "什么是好的"，再告诉它"怎么做"，然后告诉它"检查什么"，最后告诉它"输出什么格式"。

**适用场景：** 任何需要 Agent 按标准执行复杂任务的 skill。

**关键技巧：** 标准层用问句而非名词（"Does the change belong here?" 而不是 "Design"）。问句让 Agent 知道要回答什么，名词只告诉它要看什么。

### 模式三：Severity 分级系统

```markdown
- BLOCKER: must be fixed before approval.
- IMPORTANT: should be fixed unless there is a written rationale.
- QUESTION: answer needed before final verdict.
- NIT: polish that must not block approval.
- OPTIONAL: suggestion the author may decline.
- FYI: learning or future consideration.
```

**核心思想：** 给 Agent 的输出加严重级别标签，让接收者一眼看出哪些问题必须处理、哪些可以忽略。

**适用场景：** 任何需要 Agent 报告问题或给出多条建议的场景。

**关键技巧：** 每个级别附带判断标准——不是"BLOCKER 是严重问题"（循环定义），而是"BLOCKER 是必须修复才能批准的问题，涉及正确性、可维护性、安全性、数据、测试或运营风险"（可操作的定义）。

### 模式四：双向定义（正向 + 反向）

```markdown
Treat a change as an emergency only when:
- [正向条件列表]

These are usually not emergencies:
- [反向条件列表]
```

**核心思想：** 对于容易误判的概念，同时给出正向和反向定义。

**适用场景：** 任何 Agent 可能过度泛化或误判的二元决策（紧急/非紧急、阻塞/非阻塞、安全/不安全）。

**关键技巧：** 反向条件应该预判人类找借口的常见模式。"Reviewer 在另一个时区"、"周五下午了"、"经理说今天要完成"——这些都是人类把非紧急说成紧急的常见理由。

### 模式五：场景化回复模板

```text
同意时：
Done. [改了什么]. Tested with: [验证命令]

需要澄清时：
I want to make sure I understand the concern. [具体问题]?

不同意时：
I chose X because [理由]. My concern with Y is [具体问题].
If you think Y better serves [目标], I can switch. Is that the tradeoff?
```

**核心思想：** 为高频沟通场景提供现成的回复模板，降低沟通成本、保持专业性。

**适用场景：** 任何需要 Agent 代表用户与他人沟通的场景（代码审查、客户回复、团队沟通）。

**关键技巧：** 不同意时的模板遵循"先复述对方观点 → 再给出自己的理由 → 最后把决策权交给对方"的结构。这比直接说"我不同意"有效得多。

### 模式六：Fix the Right Place 分类表

```markdown
- Confusing expression: simplify or rename it.
- Hidden invariant: encode it in types, validation, or a focused comment.
- Missing behavior confidence: add or improve tests.
- Missing context: update the PR description, code comment, or README.
```

**核心思想：** 问题在哪里产生，就在哪里修复。不要在 review thread 里解释代码应该怎么写——改代码。

**适用场景：** 任何需要 Agent 处理反馈并决定修复策略的场景。

**关键技巧：** 分类表中的每一对（问题类型 → 修复位置）都是具体的。不是"在合适的地方修复"（模糊），而是"隐藏约束 → 编码到类型、验证或注释中"（具体到 Agent 可以直接执行）。

---

## 和 Claude Code feature-dev 的对比

feature-dev 和 engineering skills 代表了两种不同的 Agent skill 设计哲学：

| 维度 | feature-dev | engineering skills |
|------|-------------|-------------------|
| 文件数量 | 4 个 Markdown | 6 个 Markdown |
| 核心机制 | subagent 并行 + 人工卡点 | 检查清单 + 模板 + severity |
| 编排方式 | 单入口七阶段流水线 | 六入口路由网络 |
| 权限设计 | subagent 只读，主 Agent 执行 | 所有 skill 都假设同一个 Agent |
| 来源 | Anthropic 的 Agent 设计理念 | Google Engineering Practices |
| 使用频率 | 偶尔（大功能） | 每天（每个 PR） |
| 哲学 | "想清楚再写代码" | "写完代码后的工程质量" |

feature-dev 解决的是"从零到一做一个功能"的问题，engineering skills 解决的是"持续维护和协作"的问题。两者互补，叠加使用效果最佳。

**最大的差异在编排方式：** feature-dev 是一个线性流水线（Phase 1 → Phase 7），engineering skills 是一个路由网络（用户请求 → 匹配 skill → 执行工作流）。前者适合单一入口的大任务，后者适合多入口的日常任务。

---

## 动手：设计你自己的 Skill 套件

如果你想为自己的场景设计类似的 skill 套件，这里有一个框架：

**第一步：画出工作流全景。** 不要只覆盖"写代码"，要覆盖从拆分到交付的全链路。用一张图展示各环节之间的关系。

**第二步：每个环节一个 skill 文件。** 命名统一：`{domain}-{capability}/SKILL.md`。

**第三步：每个文件包含四层。** 标准/定义 → 工作流步骤 → 检查清单 → 模板。末尾加常见错误。

**第四步：设计触发路由。** 每个 skill 写三段式 description：Use when + TRIGGER on + DO NOT TRIGGER for。排除条件指向其他 skill 的触发条件。

**第五步：写一个 README。** 不只是文件列表，而是关系图谱——让使用者看到完整的 skill 网络。

**第六步：标注来源。** 如果 skill 基于已有的最佳实践，标注来源增加可信度。

---

*源码：[github.com/agentara/skills/tree/main/skills/engineering](https://github.com/agentara/skills/tree/main/skills/engineering)*

*姊妹篇：《AgentAra Engineering Skills 使用指南：把 Google 工程实践装进 AI Agent》——从怎么用、什么时候用到每个 skill 的实战细节。*

---

## 参考链接

- [Google Engineering Practices Documentation](https://google.github.io/eng-practices/)
