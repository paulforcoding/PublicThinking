---
title: "Claude Code /feature-dev 使用指南：七阶段流水线式功能开发"
description: "从安装到实战，手把手教你用 /feature-dev 管理从零到一的完整开发流程。三个专用 subagent、五个人工决策卡点，让 AI 在写代码之前先想清楚。"
---

# Claude Code `/feature-dev` 使用指南：七阶段流水线式功能开发

AI 写代码最大的坑不是写得烂，而是写得快。你一句话丢过去，它三秒钟吐出一堆文件——然后你要花三十分钟理解它做了什么、为什么要这样做、有没有更好的做法。

Claude Code 的 `/feature-dev` 是一个功能开发工作流插件，它把"加个功能"这件事拆成七个阶段：需求澄清 → 代码库勘探 → 模糊点澄清 → 架构设计 → 实现 → 质量审查 → 交付总结。每个阶段有专门的 subagent 执行，有明确的人工决策卡点，确保代码在写之前就已经想清楚了。

本文是**使用指南**——讲清楚怎么用、什么时候用、每个阶段你该做什么。想深入理解 feature-dev 背后的设计哲学和源码实现，请看姊妹篇《跟着 feature-dev 学写自己的编程工作流》。

## 源码在哪

`/feature-dev` 是 Claude Code 官方内置插件，源码在 `anthropics/claude-code` 仓库的 `plugins/feature-dev/` 目录下。

| 指标 | 数据 |
|------|------|
| ⭐ Stars | 130,000+ |
| 🔀 Forks | 21,100+ |
| 📄 License | Apache-2.0 |
| 📅 最近更新 | 2026-06 |
| 插件作者 | Sid Bidasaria (Anthropic) |
| 插件版本 | 1.0.0 |

GitHub: https://github.com/anthropics/claude-code/tree/main/plugins/feature-dev

**注意**：`anthropics/claude-plugins-official` 是 marketplace 索引仓库（约 29k stars），只提供安装入口，不包含源码。

## `/feature-dev` 是什么

一句话：**它是一个带人工卡点的结构化功能开发流程。**

你给 Claude Code 一句需求描述，它不会立刻动手写代码，而是按七个阶段一步步走。每个阶段之间都有"停下来等你拍板"的卡点，你掌握方向和节奏。三个专用的 subagent（代码勘探员、架构师、代码审查员）在后台做分析和检查，但真正写代码的只有主 Agent，而且必须你批准后才开始。

它的核心价值不是让 AI 写更多代码，而是让 AI **在想清楚之后再写代码**。

### 七个阶段全景

```
Phase 1: Discovery         → "你要做什么？"（需求澄清）
Phase 2: Codebase Explore  → 2-3 个 Agent 并行勘探代码库
Phase 3: Clarify Questions → 列出所有模糊点，等你回答
Phase 4: Architecture      → 2-3 个架构师给出不同方案
Phase 5: Implementation    → 主 Agent 直接写代码（等你拍板后）
Phase 6: Quality Review    → 3 个 Reviewer 并行审查
Phase 7: Summary           → 交付清单
```

三个设计决策贯穿全程：不猜需求（Phase 1 和 Phase 3 两轮澄清）、不跳步勘探（Phase 2 必须跑完才能设计架构）、不直接改代码（Phase 5 必须用户批准才开始）。

---

## 什么时候该用 `/feature-dev`

**适合用的场景：**
- 新功能，涉及多个文件的改动
- 需要做架构决策的功能（比如选哪种存储方案、用什么设计模式）
- 和现有代码有复杂集成的改动
- 需求不完全清楚，需要边讨论边明确的场景

**不适合用的场景：**
- 一行代码的 bug 修复
- 需求明确、范围很小的改动
- 紧急 hotfix
- 重构已有代码（用其他工具更合适）

判断标准很简单：**如果你自己手动做这件事，会不会先想想再做？** 如果会，那 `/feature-dev` 就值得用。

---

## 安装和启动

### 安装

`/feature-dev` 是 Claude Code 官方插件，通过 Claude Code 的插件市场安装：

```bash
claude plugins install feature-dev
```

安装后，在 Claude Code 会话中就可以直接使用 `/feature-dev` 命令了。

### 两种启动方式

**带参数启动**——需求比较明确时：

```bash
/feature-dev Add rate limiting to API endpoints
```

**不带参数启动**——想边讨论边明确：

```bash
/feature-dev
```

不带参数时，Phase 1 会先问你要做什么。两种方式没有优劣，取决于你此刻对需求的清晰度。

### 前置要求

- Claude Code 已安装
- 项目在 Git 仓库中（代码审查阶段依赖 git diff）
- 项目有已有代码（流程假设存在可以学习的代码库）

---

## 七阶段实战详解

下面以一个真实场景走完全流程：假设你在一个 Express + TypeScript 项目中，想给 API 加上 OAuth 认证。

### Phase 1: Discovery — 需求澄清

```
You: /feature-dev Add OAuth authentication for API endpoints
```

Claude 会先整理你的需求。如果你的描述不够清晰，它会问你：

- 你在解决什么问题？
- 这个功能应该做什么？
- 有什么约束或特殊要求？

然后给你一段总结让你确认。这是防止后面返工的第一道防线。

**你该做什么：** 仔细看 Claude 的需求总结，确认它理解对了。如果有偏差，在这里纠正——后面所有阶段都建立在这个基础上。

**实际对话示例：**

```
You: /feature-dev Add OAuth authentication for API endpoints

Claude: Let me understand what you need:
        - What OAuth providers? (Google, GitHub, custom?)
        - What should be protected? All endpoints or specific ones?
        - Any constraints? (existing auth system, performance requirements?)

You: Google and GitHub. Protect all /api/v1/ endpoints. We have existing
     JWT auth for internal users, OAuth should work alongside it.
```

### Phase 2: Codebase Exploration — 并行勘探

这是 feature-dev 最独特的设计。它同时启动 2-3 个 `code-explorer` agent，每个从一个维度深入追踪代码库。

每个 agent 返回的不是"这里有个 auth 模块"这种笼统的描述，而是带有 `file:line` 引用的深度分析：入口点、调用链、数据流、架构层、依赖关系。

**agent 勘探完后，主流程不直接跳到设计——它会把 agent 返回的关键文件列表全部读一遍，建立深度上下文。** 这是 feature-dev 的核心哲学：agent 帮你找文件，但真正理解代码库的是读代码的过程。

**你该做什么：** 看 Claude 给出的勘探总结。重点关注：
- 它发现的代码模式是不是你项目里真实的模式
- 有没有遗漏重要的模块或依赖关系
- 关键文件列表是否合理

**实际输出示例：**

```
Found similar features:
- User authentication (src/auth/): Uses JWT tokens, middleware pattern
- Session management (src/session/): Redis-backed, 24hr expiry
- API security (src/api/middleware/): Rate limiting, CORS

Key files to understand:
- src/auth/AuthService.ts:45 - Core authentication logic
- src/middleware/authMiddleware.ts:12 - Request authentication
- src/config/security.ts:8 - Security configuration
```

### Phase 3: Clarifying Questions — 不跳过这一步

源码中用 `CRITICAL` 和 `DO NOT SKIP` 标注。

这个阶段的价值不在提问本身，而在于强迫你在写代码之前想清楚边界。Phase 2 的勘探结果 + 原始需求放在一起，自然暴露出模糊地带：边界情况、错误处理策略、集成点、范围边界、向后兼容性、性能需求。

**你该做什么：** 认真回答每个问题。即使 Claude 说"whatever you think is best"也行——但它会给出建议然后等你确认，不会替你做决定。

**实际对话示例：**

```
Claude: Before designing the architecture, I need to clarify:

1. OAuth provider: Google and GitHub confirmed. What about others later?
2. User data: Store OAuth tokens in DB or just use user profile?
3. Existing auth: Keep JWT auth alongside OAuth, or migrate everything?
4. Sessions: Integrate with existing Redis session management?
5. Error handling: How to handle OAuth provider failures?
6. Token refresh: Automatic or manual?

You: 1. Just Google/GitHub for now, design for extensibility
     2. Store tokens in DB, encrypted
     3. Keep JWT alongside, OAuth is additional
     4. Yes, use existing Redis sessions
     5. Graceful degradation - fall back to JWT auth
     6. Automatic refresh
```

**这一步花 5 分钟，能省下后面 50 分钟的返工。**

### Phase 4: Architecture Design — 三套方案

同时启动 2-3 个 `code-architect` agent，每个给一个不同的设计方向：

| 方向 | 目标 | 适用场景 |
|------|------|----------|
| **Minimal changes** | 最小改动，最大复用 | 小功能、热修复 |
| **Clean architecture** | 可维护性、优雅抽象 | 核心功能、长期维护 |
| **Pragmatic balance** | 速度 + 质量平衡 | 大多数情况 |

Claude 会把 2-3 个方案汇总对比，然后给出它的推荐（附理由），最后让你选。

**你该做什么：** 不要无脑选推荐。考虑你的上下文——团队偏好、时间压力、长期规划。架构是选择题，不是填空题，答案不在技术上，在你的判断里。

**实际输出示例：**

```
I've designed 3 approaches:

Approach 1: Minimal Changes
- Extend existing AuthService with OAuth methods
- Add new OAuth routes to existing auth router
- Minimal refactoring required
Pros: Fast, low risk
Cons: Couples OAuth to existing auth, harder to test

Approach 2: Clean Architecture
- New OAuthService with dedicated interface
- Separate OAuth router and middleware
- Refactor AuthService to use common interface
Pros: Clean separation, testable, maintainable
Cons: More files, more refactoring

Approach 3: Pragmatic Balance
- New OAuthProvider abstraction
- Integrate into existing AuthService
- Minimal refactoring, good boundaries
Pros: Balanced complexity and cleanliness
Cons: Some coupling remains

Recommendation: Approach 3 - gives you clean boundaries without
excessive refactoring, and fits your existing architecture well.

Which approach would you like to use?
```

**到 Phase 4 为止，一行代码都还没写。** 这是刻意的——架构设计全部在只读 agent 中完成。

### Phase 5: Implementation — 等你拍板后主 Agent 动手写

这是整个流程中**唯一不启动 subagent 的阶段**，也是唯一真正写代码的阶段。

为什么？因为前四个阶段已经把该准备的都准备好了——代码库已经勘探透了、需求已经澄清了、架构方案已经选定了。实现阶段不需要再并行搜索或分析，它就是一个线性的执行过程。

源码中的指令很明确：

> "DO NOT START WITHOUT USER APPROVAL"

Claude 会等你明确说"开始"或"go"之后才动手。

**你该做什么：** 确认 Claude 理解了选定的架构方案，然后让它开工。实现过程中可以随时问问题，Claude 会边写边更新进度。

### Phase 6: Quality Review — 三维并行审查

写完代码不是终点。Phase 6 启动 3 个 `code-reviewer` agent 并行审查：

| Agent | 维度 | 关注点 |
|-------|------|--------|
| 1 | Simplicity / DRY / Elegance | 代码质量、可维护性 |
| 2 | Bugs / Functional Correctness | 逻辑错误、边界处理 |
| 3 | Conventions / Abstractions | 项目规范符合度 |

审查结果出来后，Claude 让你三选一：现在修 / 以后修 / 不改。

**你该做什么：** 看审查结果，特别关注 Critical 级别的问题。对于每个问题，判断是现在修还是以后修。不是所有问题都需要立刻修——有些可以记到技术债里。

**实际输出示例：**

```
Code Review Results:

High Priority Issues:
1. Missing error handling in OAuth callback (src/auth/oauth.ts:67)
   → If OAuth provider returns an error, the callback crashes
   → Fix: Add try-catch with fallback to JWT auth
2. Memory leak: OAuth state not cleaned up (src/auth/oauth.ts:89)
   → CSRF state tokens accumulate in Redis
   → Fix: Add TTL to state entries

Medium Priority:
1. Could simplify token refresh logic (src/auth/oauth.ts:120)
2. Consider extracting OAuth config validation

What would you like to do? Fix now, fix later, or proceed as-is?
```

### Phase 7: Summary — 交付清单

没有长篇总结。四样东西：做了什么、关键决策、修改了哪些文件、建议的下一步。干净利落。

```
Feature Complete: OAuth Authentication

What was built:
- OAuth provider abstraction supporting Google and GitHub
- OAuth routes and middleware integrated with existing auth
- Token refresh and session integration
- Error handling for all OAuth flows

Key decisions:
- Used pragmatic approach with OAuthProvider abstraction
- Integrated with existing session management
- Added OAuth state to prevent CSRF

Files modified:
- src/auth/OAuthProvider.ts (new)
- src/auth/AuthService.ts
- src/routes/auth.ts
- src/middleware/authMiddleware.ts

Suggested next steps:
- Add tests for OAuth flows
- Add more OAuth providers (Microsoft, Apple)
- Update documentation
```

---

## 单独使用 Agent

除了完整的七阶段流程，三个 subagent 也可以单独调用。这在某些场景下更实用。

### 单独勘探代码库

当你需要理解一个已有功能怎么实现的，但不想改它：

```
Launch code-explorer to trace how authentication works
```

适合场景：接手别人的代码、排查某个功能为什么这样表现、准备重构前先理解现状。

### 单独做架构设计

当你有明确的设计需求但不想走完整流程：

```
Launch code-architect to design the caching layer
```

适合场景：技术方案评审前的预研、评估某种架构在当前项目中的可行性。

### 单独做代码审查

当你写完代码想快速过一遍审查：

```
Launch code-reviewer to check my recent changes
```

适合场景：提交 PR 前的自查、想从另一个角度审视自己的代码。

---

## 使用技巧

### 需求描述越具体，Phase 3 的问题越少

如果你在 Phase 1 就把约束条件说清楚（OAuth providers、存储策略、集成方式），Phase 3 的澄清问题会大幅减少。但这不是说你要写一份 spec——留一些空间让 Claude 主动提问，往往能发现你没想到的盲点。

### 认真对待 Phase 3

很多人在这一步敷衍回答"你看着办吧"。Claude 会给出建议让你确认，但这意味着你放弃了审查权。至少对核心问题（集成方式、错误处理策略、性能要求）给出明确回答。

### Phase 4 选方案时考虑上下文

三个方案不是随便出的。Minimal changes 适合你赶时间；Clean architecture 适合这个功能长期会频繁修改；Pragmatic balance 适合大多数日常场景。结合你的项目节奏来选。

### 不要跳过 Phase 6

代码审查不是走形式。三个 reviewer 从不同角度看，经常能发现你自己没注意到的问题。即使你选择"以后修"，至少知道有哪些隐患。

### 大项目里勘探阶段会比较慢

这是正常的。2-3 个 agent 并行跑，要读很多文件、追踪很多调用链。慢说明它在认真勘探，不是在敷衍你。

---

## 为什么这套流程值得认真对待

**Subagent 和主 Agent 的分工明确。** 三个 subagent 只做只读分析，写代码的权力严格隔离在 Phase 5。这不是因为 AI 不可信，而是因为分析和执行需要不同的上下文粒度——分析需要聚焦，执行需要全局。

**用户决策点精确卡位。** Phase 1 确认需求 → Phase 3 澄清模糊点 → Phase 4 选择方案 → Phase 5 批准开工 → Phase 6 决定修不修。五个决策点，确保人始终在 loop 里，但不是 micro-manage。

**显式的"理解代码库"阶段。** 大多数 AI 编码工具把"理解代码库"当作一个隐式的、自动发生的背景任务。feature-dev 把它变成一个显式的、2-3 个 Agent 并行的、必须交付关键文件清单的阶段。勘探不完成，设计不开始。

---

## 常见问题

**Q: agent 勘探太慢了怎么办？**
大型代码库是正常的。agent 并行跑，勘探越彻底后面返工越少。如果实在太慢，可以在 Phase 1 把需求描述得更具体，减少勘探范围。

**Q: Phase 3 问题太多了？**
说明 Phase 1 的需求描述不够清晰。下次在启动时就带上更多约束条件。或者说"whatever you think is best"让它给建议——但核心问题最好自己回答。

**Q: Phase 4 的方案选择困难？**
信它的推荐——推荐是基于代码库分析得出的。如果还不确定，选 Pragmatic balance 通常不会错。

**Q: 可以不走完全部七个阶段吗？**
可以。如果需求很小很明确，Phase 1 和 Phase 3 会很快过。你也可以单独调用 agent（见上文）。但核心流程不建议跳步。

---

*源码：[github.com/anthropics/claude-code/tree/main/plugins/feature-dev](https://github.com/anthropics/claude-code/tree/main/plugins/feature-dev)*

*姊妹篇：《跟着 feature-dev 学写自己的编程工作流》——深入源码分析，学习如何设计自己的 AI 编程工作流。*
