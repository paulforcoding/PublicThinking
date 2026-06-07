# Claude Code 插件全览：36 个官方插件 + 15 个合作伙伴插件的架构拆解

Claude Code 最被低估的能力不是它写代码有多好——而是它的插件系统。

2026 年 1 月，Anthropic 团队开源了他们内部使用的官方插件集合。36 个 Anthropic 自建插件，加上 15 个合作伙伴插件（GitHub、Microsoft、Google、HashiCorp 等），构成了一个覆盖从需求澄清到代码部署的完整开发生命周期的扩展体系。

仓库：[anthropics/claude-plugins-official](https://github.com/anthropics/claude-plugins-official) ⭐29k 🔀3.2k 📅2026-06-07 📄Apache-2.0

但这不是一份"插件列表"。读完 36 个插件的源码，你会看到 Anthropic 团队**如何设计 agent 工作流**——什么该交给 subagent、什么该留在主 agent、什么该用 hook 即时拦截、什么该用 skill 自动加载。这些模式可以被任何支持 subagent 的 AI 编码工具复现。

---

## 对开发者有什么用

安装这些插件后，Claude Code 从一个"对话式编码助手"变成了一个"可组合的开发工作流引擎"。具体来说：

- **写功能**：输入 `/feature-dev`，Claude 按七阶段流水线推进——勘探代码库、澄清需求、设计架构、写代码、审查质量。全程有 5 个决策卡点等你确认。
- **审查 PR**：输入 `/code-review`，5 个并行 agent 分别从 bug、规范、历史上下文等维度审查 diff，只报告置信度 ≥80 的问题。
- **安全防护**：装一个 security-guidance，Claude 每次编辑代码时自动匹配 25 种危险模式（`yaml.load`、裸 `innerHTML`、硬编码密钥），编辑完用 LLM 审查 diff，提交前再过一道闸门。
- **遗留系统改造**：输入 `/code-modernization`，七阶段强制流程——评估、映射、提取业务规则、设计方案、实施、加固。防止"在没理解代码前就开始改"。

Reddit 上有个高赞帖标题是"大多数人不知道的 28 个 Claude Code 官方插件"，原话是：

> "My rough estimate is that Claude Code at default settings is running at maybe 60% of what it can actually do."

---

## 插件系统的四种组件

Claude Code 插件的核心是一个目录 + 一个 `plugin.json` 清单文件。每个插件可以使用以下四种扩展机制的任意组合：

| 组件 | 位置 | 谁触发 | 干什么 |
|------|------|--------|--------|
| **Skill** | `skills/<name>/SKILL.md` | Claude 自动判断 | 可复用的知识/流程包，支持 auto-invocation |
| **Command** | `commands/<name>.md` | 用户输入 `/xxx` | 斜杠命令，手动触发 |
| **Agent** | `agents/<name>.md` | 主 agent 委派 | 独立上下文窗口的 subagent |
| **Hook** | `hooks/hooks.json` | 系统生命周期事件 | 编辑前拦截、编辑后审查、会话结束时触发 |

Skill 和 Command 的区别值得说清楚。Command 是早期实现——一个扁平 Markdown 文件，用户手动 `/xxx` 触发。Skill 是进化版——一个目录结构，可以包含 `reference.md`（参考文档）和 `scripts/`（脚本），并且支持 auto-invocation（Claude 根据 `description` 字段自动判断何时加载）。官方文档明确建议新插件使用 Skill 而非 Command。

Agent 是最特殊的组件。每个 agent 有自己的上下文窗口、工具权限集、甚至可以选择不同的模型（Sonnet/Opus/Haiku）。它运行在隔离环境中，和主 agent 的唯一通信方式是返回结果摘要。这个设计的后果，后面会反复出现。

---

## 第一类：LSP 集成（12 个）

```
clangd-lsp       C/C++       基于 clangd
csharp-lsp       C#          基于 OmniSharp
gopls-lsp        Go          基于 gopls
jdtls-lsp        Java        基于 Eclipse JDT.LS
kotlin-lsp       Kotlin      基于 Kotlin Language Server
lua-lsp          Lua         基于 lua-language-server
php-lsp          PHP         基于 Intelephense
pyright-lsp      Python      基于 Pyright
ruby-lsp         Ruby        基于 ruby-lsp (Shopify)
rust-analyzer-lsp Rust       基于 rust-analyzer
swift-lsp        Swift       基于 SourceKit-LSP
typescript-lsp   TypeScript  基于 typescript-language-server
```

12 个插件的结构完全相同：每个目录只有 `LICENSE` 和 `README.md`，没有 `.claude-plugin/plugin.json`、没有 Skills、没有 Commands、没有 Hooks。它们是纯配置型插件——`README.md` 中声明了对特定 Language Server 的支持。

安装后，Claude Code 在编辑对应语言的文件时自动启动 LSP，获得实时类型检查和跳转定义能力。零配置，零逻辑，声明即生效。

其中 **typescript-lsp** 是社区安装量最高的插件之一（Composio 统计 177,136 次安装）。Reddit 用户 igbins09 的评价：

> "The difference in code quality is noticeable. Claude stops guessing at types and actually checks them."

---

## 第二类：开发工作流（5 个）

这是整个插件生态的核心层。

### feature-dev：七阶段功能开发流水线

- **作者**：Anthropic
- **组件**：3 个 subagent + 1 个 command + 多阶段流程
- **描述**（plugin.json 原文）：*"Comprehensive feature development workflow with specialized agents for codebase exploration, architecture design, and quality review"*

七个阶段，三种 subagent 分工：

```
Phase 1: Discovery         → 主 agent
Phase 2: Codebase Explore  → 2-3 个 code-explorer subagent（并行，只读）
Phase 3: Clarify Questions → 主 agent
Phase 4: Architecture      → 2-3 个 code-architect subagent（并行，只读）
Phase 5: Implementation    → 主 agent（写代码）
Phase 6: Quality Review    → 3 个 code-reviewer subagent（并行，只读）
Phase 7: Summary           → 主 agent
```

5 个用户决策卡点：需求确认、澄清问题确认、方案选择、批准开工、审查修复确认。

社区评价——Reddit r/ClaudeCode 专帖"The feature-dev plugin leveled up my code"：

> "I don't recommend it for every change, just medium complexity, not more or less. If it's too low, it's not worth the time to go through the phases."

> "I also tested the feature dev plugin on another full-stack project, and it completely eliminated the chaos of my usual workflow. It asked very good clarifying questions."

关键限制：社区一致认为不适合小改动，只适合中等复杂度以上的功能开发。

### code-simplifier：事后代码净化

- **作者**：Anthropic
- **组件**：1 个 agent（code-simplifier，Opus）
- **描述**：*"Agent that simplifies and refines code for clarity, consistency, and maintainability while preserving functionality"*

审查后直接修复——不是只给报告，而是直接改代码。运行在 Opus 上，因为精简代码需要深度理解能力。

和 feature-dev Phase 6 的 code-reviewer 的区别：code-reviewer 只审查不修改（给人类看报告），code-simplifier 审查后直接改。

### code-review：PR 审查

- **作者**：Anthropic
- **组件**：5 个并行 Sonnet agent + 置信度过滤
- **描述**：*"Automated code review for pull requests using multiple specialized agents with confidence-based scoring"*

五个 agent 各查一个维度：CLAUDE.md 规范符合度、bug 检测、历史上下文（相关 commit/issue）、PR 历史（之前的评论和修改）、代码注释准确性。置信度评分只报 ≥80 的问题。

Reddit 评价：

> "Five parallel Sonnet agents check for bugs, CLAUDE.md compliance, historical context, pull request history, and stale comments."

> "Runs every session, costs nothing, catches things review misses."

被称为"the plugin most teams need but never install"。

### pr-review-toolkit：六个专项审查 Agent

- **作者**：Anthropic
- **组件**：6 个专项 agent
- **描述**：*"Comprehensive PR review agents specializing in comments, tests, error handling, type design, code quality, and code simplification"*

比 code-review 更细粒度。六个 agent 各自专攻：

| Agent | 专攻 | 典型发现 |
|-------|------|----------|
| `comment-analyzer` | 注释准确性 | 注释说"返回 User"但实际返回 `User \| null` |
| `pr-test-analyzer` | 测试覆盖 | 新代码路径没有被已有测试覆盖 |
| `silent-failure-hunter` | 静默失败 | `try { ... } catch {}` 空 catch 块 |
| `type-design-analyzer` | 类型设计 | `string` 用于有限状态值，应该用 union type |
| `code-simplifier` | 简洁性 | 不必要的抽象层 |
| `code-reviewer` | 综合质量 | 违反项目规范 |

和 code-review 的关系：pr-review-toolkit 提供 agent 集合，code-review 提供编排流程。

### code-modernization：遗留系统现代化

- **作者**：Anthropic
- **组件**：多阶段流程 + 多个 specialist agent
- **描述**：*"Modernize legacy codebases (COBOL, legacy Java/C++, monolith web apps) with a structured assess → map → extract-rules → brief → reimagine/transform → harden workflow and specialist review agents"*

36 个插件中最重的一个。七阶段强制流程：

```
assess → map → extract-rules → brief → reimagine | transform → harden
```

每个阶段有专门的 agent：`legacy-analyst`（映射模块边界）、`business-rules-extractor`（提取业务规则）、`test-engineer`/`security-auditor`/`architecture-critic`（加固阶段三管齐下）。

设计哲学：现代化失败的原因通常不是目标技术栈选错了，而是跳过了步骤——在没理解代码前就开始改，在没提取业务规则前就重新设计架构。

社区评价：理念好但太重，实际使用者极少。被称为"最重但最少人用的插件"。

---

## 第三类：插件与技能开发（3 个）

这三个是"元插件"——用来开发 Claude Code 插件本身。

### plugin-dev：七个专家 Skill

- **作者**：Anthropic
- **描述**：*"Plugin development toolkit with skills for creating agents, commands, hooks, MCP integrations, and comprehensive plugin structure guidance"*

创建插件时会遇到的七个话题，每个有一个专门的 skill：

| Skill | 覆盖内容 |
|-------|----------|
| `plugin-structure` | 目录组织、plugin.json 配置 |
| `plugin-settings` | 用户可配置项模式 |
| `command-development` | 斜杠命令设计 |
| `agent-development` | Subagent 定义 |
| `skill-development` | Skill 创建 |
| `hook-development` | Hook API、事件驱动 |
| `mcp-integration` | MCP 服务器集成 |

还有 `/create-plugin` 命令，走八阶段引导式创建流程。三个 agent（`agent-creator`、`skill-reviewer`、`plugin-validator`）分别在创建、审查、验证阶段介入。

已知问题：GitHub Issue #1758 指出 14 个审计的插件中有 8 个显示"Version: unknown"，包括 plugin-dev 自身，导致缓存目录和更新检测失效。

### skill-creator：技能工厂

- **作者**：Anthropic
- **描述**：*"Create new skills, improve existing skills, and measure skill performance. Use when users want to create a skill from scratch, update or optimize an existing skill, run evals to test a skill, or benchmark skill performance with variance analysis."*

不是写代码的 skill——是**写 skill 的 skill**。提供了：从零创建 skill 的流程、评估 skill 效果的 benchmark 系统（带方差分析）、三个评估 agent（`analyzer`/`comparator`/`grader`）、打包和发布脚本。

一个"用 agent 改进 agent 的指令"的闭环。skill 写得好不好，跑一遍 eval 就知道。

已知问题：`run_eval.py` 在 skill 已安装时所有查询打 0 分，且 Windows 管道不兼容。

### mcp-server-dev：MCP 服务器构建

- **作者**：Anthropic
- **描述**：*"Skills for designing and building MCP servers that work seamlessly with Claude — guides you through deployment models (remote HTTP, MCPB, local), tool design patterns, auth, and interactive MCP apps."*

三个 skill 覆盖 MCP 服务器的完整开发路径：`build-mcp-server`（入口，选部署模型和工具设计模式）、`build-mcp-app`（添加交互式 UI 组件）、`build-mcpb`（将本地 stdio 服务器打包为 MCP 包）。

已知问题：`mcp-integration` skill 文档已过时（远程 MCP transport / OAuth 配置形状）。

---

## 第四类：项目管理（2 个）

### claude-code-setup：智能配置推荐

- **作者**：Anthropic
- **描述**：*"Analyze codebases and recommend tailored Claude Code automations such as hooks, skills, MCP servers, and subagents."*

安装后自动分析代码库，推荐适合的自动化配置：MCP 服务器（如 context7 查文档、Playwright 做前端测试）、Skills（如 plan agent、frontend-design）、Hooks（自动格式化、自动 lint、阻止敏感文件提交）、Subagents（安全审查、性能审查、无障碍审查）。

一个"帮你的 agent 配置 agent"的插件。

### claude-md-management：CLAUDE.md 维护

- **作者**：Anthropic
- **描述**：*"Tools to maintain and improve CLAUDE.md files - audit quality, capture session learnings, and keep project memory current."*

两个工具：`claude-md-improver`（skill）定期审计 CLAUDE.md 是否和当前代码库一致，自动更新过时的约定；`/revise-claude-md`（command）在会话结束时捕捉"这次会话暴露了哪些 CLAUDE.md 没写到的知识"，追加进去。

设计理念：CLAUDE.md 不是写完就一劳永逸的文档。代码库在变，团队的约定在变，agent 应该主动帮你维护它。

---

## 第五类：安全与防护（2 个）

### security-guidance：三层安全网

- **作者**：David Dworken（Anthropic 安全工程师）
- **版本**：2.0.3
- **描述**：*"Security review for Claude-generated code. Pattern-based warnings on edits, LLM-powered diff review on Stop, and an agentic commit reviewer that catches injection, XSS, SSRF, hardcoded secrets, and 25+ other vulnerability classes."*

三层防护，每层用了不同的扩展机制：

1. **Pattern warnings（hook: PreToolUse）** — agent 调用 `Edit`/`Write` 时，正则匹配约 25 种已知危险模式：`yaml.load`（应为 `yaml.safe_load`）、`torch.load(weights_only=False)`、`pickle.load` 处理不可信数据、裸 `innerHTML` 赋值、硬编码密钥。命中则注入警告。

2. **LLM diff review（hook: Stop）** — agent 完成一轮修改后，将 diff 发给 Opus 做安全审查，高危发现反馈给 agent，让它在返回用户前修复。

3. **Agentic commit review（hook: PostToolUse on git commit）** — 通过 SDK 启动的审查器，检查注入、XSS、SSRF、硬编码密钥等 25+ 类安全问题。

三层逐级加码：第一层即时正则（几乎无延迟），第二层 LLM 审查（有延迟但更准确），第三层提交前最后一道闸门。

社区评价极好。Reddit r/ClaudeAI：

> "Caught a real auth bypass in my codebase that Claude had originally written and never flagged. Worth it for that alone."

> "I've been using it and it works pretty well. It already found a few things that I did not notice."

Firecrawl 评价："Blocks risky code edits automatically, non-annoying (shows once per session)"。

### hookify：用自然语言创建 Hook

- **作者**：Anthropic
- **描述**：*"Easily create hooks to prevent unwanted behaviors by analyzing conversation patterns"*

传统方式创建 hook 要手写 `hooks.json` 和 Python 脚本。hookify 把这个过程变成对话：描述一个不想要的行为（如"不要用 `console.log` 做调试"），一个 `conversation-analyzer` agent 分析对话、生成匹配规则、自动创建 hook 配置文件。

还支持从已有对话中**反向分析**：扫描历史对话，找出反复出现的问题模式，自动生成拦截规则。

---

## 第六类：Git 与提交（1 个）

### commit-commands：三个 Git 命令

- **作者**：Anthropic
- **描述**：*"Streamline your git workflow with simple commands for committing, pushing, and creating pull requests"*

- `/commit` — 分析暂存和未暂存的变更，自动生成 commit message，提交
- `/commit-push-pr` — 一步完成 commit → push → create PR
- `/clean_gone` — 清理本地已经不存在于远程的分支

减少了上下文切换。不需要切出终端、手写 commit message、手动 push。

Reddit 评价：

> "Handles commit message formatting, branch hygiene, and the small repetitive stuff that breaks flow when you have to type it out each time."

---

## 第七类：输出与学习风格（2 个）

### explanatory-output-style

- **作者**：Anthropic
- **描述**：*"Adds educational insights about implementation choices and codebase patterns (mimics the deprecated Explanatory output style)"*

用 SessionStart hook 注入指令，让 agent 在回复中加入教学性解释——为什么这样实现、代码库中相关的模式、设计决策的背景。注意描述中写了"mimics the deprecated Explanatory output style"——这说明原来有一个内置的输出风格选项被废弃了，这个插件是它的替代品。

### learning-output-style

- **作者**：Anthropic
- **描述**：*"Interactive learning mode that requests meaningful code contributions at decision points (mimics the unshipped Learning output style)"*

explanatory 的增强版。除了教育性解释外，还会在关键决策点**停下来让你做选择**，实现"边做边学"的交互模式。描述中写了"mimics the unshipped Learning output style"——这个功能从未上线过，这个插件是它的实现。

这两个插件的存在说明：**agent 的输出风格是可编程的**。不是模型能力强弱的区别，而是系统提示注入带来的体验差异。

---

## 第八类：前端与设计（2 个）

### frontend-design

- **作者**：Anthropic
- **描述**：*"Frontend design skill for UI/UX implementation"*

让 agent 在生成前端界面时避免"AI 美学"——那种一眼就能看出是 AI 生成的、千篇一律的 UI。自动选择大胆的配色方案、独特的排版、高冲击力的动画和视觉细节。

Composio 安装量统计显示 frontend-design 是最受欢迎的官方插件之一。Redwerk 博客列入"7 Best Claude Code Plugins"，评价："Results feel designed, not AI-slop"。

### playground

- **作者**：Anthropic
- **描述**：*"Creates interactive HTML playgrounds — self-contained single-file explorers with visual controls, live preview, and prompt output with copy button"*

创建自包含的单文件 HTML 交互式演示。带可视化控件、实时预览、即时反馈。

---

## 第九类：Agent SDK（1 个）

### agent-sdk-dev

- **作者**：Anthropic
- **描述**：*"Claude Agent SDK Development Plugin"*

为 Claude Agent SDK（Python 和 TypeScript）应用提供脚手架和验证。`/new-sdk-app` 交互式创建新 SDK 应用，`agent-sdk-verifier` agent 检查代码是否符合官方 SDK 文档的最佳实践。

---

## 第十类：专项工具（5 个）

### math-olympiad：对抗性数学验证

- **作者**：Anthropic
- **描述**：*"Solve competition math (IMO, Putnam, USAMO) with adversarial verification that catches what self-verification misses. Fresh-context verifiers attack proofs with specific failure patterns. Calibrated abstention over bluffing."*

解决学术论文暴露的问题——arXiv:2503.21934 显示 85.7% 自验证的 IMO 成功率在人工评分后下降到 <5%。方案：上下文隔离验证（验证者只看最终证明，不看推理过程）、对抗性检查模式（不问"对吗？"，而问"是不是不小心证明了黎曼猜想？"）、校准性放弃（说不确定比猜错强）。

### ralph-loop：持续自循环

- **作者**：Anthropic
- **描述**：*"Continuous self-referential AI loops for interactive iterative development, implementing the Ralph Wiggum technique. Run Claude in a while-true loop with the same prompt until task completion."*

Geoffrey Huntley 的创意——一个 `while true` 循环，反复把同一个 prompt 喂给 agent，让它在迭代中持续改进。灵感来自 Leslie Lamport（图灵奖得主）在 DE Shaw 期间的 bug 修复方法论：执行→发现错误→修改→重新执行。

Firecrawl 评价："AFK coding, no circular reasoning, guardrails"，但"Requires PRD files, many small commits, token burn"。

### mcp-tunnels：私有 MCP 隧道

- **作者**：Anthropic
- **描述**：*"Connect Claude to a private MCP server through an Anthropic MCP tunnel. Drives the Docker Compose quickstart end to end: certificates, proxy config, cloudflared, and a verifiable sample server."*

通过 Anthropic MCP 隧道将 Claude Code 连接到私有网络中的 MCP 服务器——不需要开放入站端口、不需要公开暴露、不需要在服务器上加 IP 白名单。流量走纯出站连接。

### session-report：会话分析

- **作者**：Anthropic
- **结构**：`skills/` 目录（无 plugin.json）

分析 Claude Code 会话数据，生成统计报告。已知问题：GitHub Issue #1758 指出该插件缺少 version 字段。

### example-plugin：示例插件

- **作者**：Anthropic
- **描述**：*"A comprehensive example plugin demonstrating all Claude Code extension options including commands, agents, skills, hooks, and MCP servers"*

展示所有扩展机制的参考实现。开发自己的插件时，从这个模板开始是最快的路径。

### cwc-makers：Makers Cardputer 引导

- **作者**：Anthropic
- **描述**：*"Seamless onboarding for the Code-with-Claude Makers Cardputer: one /maker-setup command clones the build-with-claude repo, flashes UIFlow firmware, and installs the Claude Buddy app bundle onto a freshly-plugged-in M5Stack Cardputer-Adv."*

面向 Code-with-Claude Makers Cardputer 硬件的一键引导。一个 `/maker-setup` 命令完成仓库克隆、固件烧录和应用安装。

---

## 合作伙伴插件（15 个）

`external_plugins/` 目录下有 15 个由第三方公司维护的插件：

| 插件 | 维护方 | 做什么 |
|------|--------|--------|
| **github** | GitHub | 仓库管理、issue 创建、PR 管理 |
| **playwright** | Microsoft | 浏览器自动化和端到端测试 |
| **context7** | Upstash | 实时文档查询，减少 API 幻觉（57k+ 安装） |
| **firebase** | Google | Firestore 数据库、认证、云函数管理 |
| **terraform** | HashiCorp | Terraform 生态集成 |
| **linear** | Linear | issue 追踪、项目管理、状态更新 |
| **asana** | Asana | 任务创建和管理、项目搜索 |
| **gitlab** | GitLab | 仓库、MR、CI/CD 管道管理 |
| **greptile** | Greptile | AI 代码审查 agent |
| **serena** | Oraios | 语义代码分析、智能重构 |
| **laravel-boost** | Laravel | Laravel 开发工具包 |
| **discord** | 社区 | Discord 消息桥接 |
| **telegram** | 社区 | Telegram 消息桥接 |
| **imessage** | 社区 | iMessage 消息桥接 |
| **fakechat** | 社区 | 本地 iMessage 风格测试界面 |

其中 **context7** 是最受欢迎的第三方插件（57k+ 安装）。Redwerk 评价："No more outdated API suggestions. It actually looks up current docs." **playwright**（Microsoft 维护）是前端测试的标准选择，但 Firecrawl 警告："Heavy token usage, browser sessions can be unpredictable。"

---

## 贯穿始终的 11 个设计模式

读完 36 个官方插件，以下模式反复出现：

### 1. 多 Agent 并行 > 单 Agent 全栈

几乎每个涉及分析的插件都用了 2-6 个并行 agent。code-simplifier 三个 reviewer、feature-dev 三个 explorer、pr-review-toolkit 六个专项 agent、code-review 五个并行 Sonnet。上下文隔离 + 维度专注，这是 agent 架构的第一原则。

### 2. Subagent 只读，主 Agent 写代码

code-explorer、code-architect、code-reviewer——所有 subagent 的工具集都是 `Glob, Grep, LS, Read, NotebookRead, WebFetch, WebSearch`。没有一个有 `Write` 或 `Edit` 权限。写代码的权力保留在主 agent 中，通常需要用户显式批准。这不是技术限制，是刻意设计——subagent 的上下文是隔离的，它不知道前序讨论的决策和权衡。

### 3. 置信度过滤

code-review、pr-review-toolkit 都用了置信度评分（0-100），只报告 ≥80 的问题。传统 lint 工具最大的问题是 noise——报 100 个 warning，99 个不重要。置信度过滤把"发现"和"噪音"分开。

### 4. Hook 做守护，Agent 做分析

security-guidance 的三层设计是范式：Hook 做即时拦截（正则在 `Edit` 时匹配危险模式），Agent 做深度分析（LLM 审查 diff），Agent 做最后闸门（commit 前审查）。Hook 快但浅，Agent 慢但深，两者互补。

### 5. 流程强制 > 自由发挥

code-modernization 的 `assess → map → extract-rules → brief → reimagine/transform → harden`，feature-dev 的七个阶段——这些插件都在强制一个顺序。不是 agent 觉得该做什么就做什么，而是流程模板锁定了步骤。

### 6. Skill 自动加载，Command 手动触发

Skill 有 `description` 字段，agent 根据描述自动判断何时加载。Command 是用户手动 `/xxx` 触发。一个隐式、一个显式，覆盖不同场景。新插件应该优先用 Skill。

### 7. 元插件闭环

plugin-dev 开发插件、skill-creator 创建 skill、mcp-server-dev 构建 MCP 服务器——三个插件形成了一个自己扩展自己的闭环。用 agent 来改进 agent 的配置，用 agent 来创建 agent 的工具。

### 8. 用户决策点精确卡位

feature-dev 的五个决策点不是随意放的。每个决策点都在"信息刚好足够做出判断"的时刻。Phase 2 勘探完才能问澄清问题，Phase 3 澄清完才能设计架构。信息不够时不给选择，信息够了才问。

### 9. 事后自动修复 vs 事前审查给人类看

code-simplifier 是事后自动修复（写完了→直接改），code-review 是事前审查（PR 阶段→给人类看报告）。同一个"审查代码"的动作，根据在开发流程中的位置不同，走完全不同的处理路径。

### 10. 模型选择按任务分级

code-simplifier 用 Opus（深度理解代码），feature-dev 的 subagent 全用 Sonnet（广度搜索 + 常规推理），security-guidance 的 LLM diff review 用 Opus。不是越强的模型越好——任务的认知需求决定模型选择。

### 11. 开源即教学

每个插件的源码就是一份"Anthropic 教你如何设计 agent 工作流"的教学材料。Prompt 怎么写、边界怎么设、权限怎么分、流程怎么卡——全在代码里。这是比任何文档都更有价值的 agent 工程参考资料。

---

## 与 Cursor Rules 和 Copilot Extensions 的对比

Claude Code 插件系统不是唯一的选择。三个主流 AI 编码工具的扩展机制正在走不同的路：

| 维度 | Claude Code Plugins | Cursor Rules | Copilot Extensions |
|------|--------------------|--------------|-------------------|
| **本质** | 文件夹 + manifest 的包分发 | `.cursor/rules/*.mdc` 规则文件 | IDE 扩展 + MCP |
| **运行环境** | 终端/CLI，IDE 无关 | AI 原生 IDE（VS Code fork） | 多 IDE（VS Code、JetBrains） |
| **扩展方式** | Skills + Agents + Hooks + MCP + LSP + Monitors | Rules（MDC 格式 Markdown）+ MCP | Copilot Extensions（GitHub App）+ MCP |
| **Agent 隔离** | Subagent 可有独立 worktree、工具权限、模型选择 | 无隔离，规则注入当前会话 | 有限隔离 |
| **Hook 系统** | 20+ 事件类型，支持 command/http/mcp_tool/prompt/agent | 无 | 无（依赖 GitHub Actions） |
| **分发** | Marketplace（官方+社区+第三方） | 手动复制 `.cursor/rules/` | GitHub Marketplace 安装 App |

Claude Code 的插件系统是三者中最复杂、最强大的。它的核心设计哲学是**"文件夹即包"**——通过约定优于配置的目录结构，将 Skills、Agents、Hooks、MCP、LSP、Monitors 打包成一个可分发的单元。

Cursor 的优势在开发者体验——一个 MDC 文件就能配置规则，IDE 一体化。Copilot 的优势在价格和生态——$10/月，GitHub 深度集成。

社区常见的组合策略是："Cursor for daily editing + Claude Code for complex tasks"。

---

## 已知问题

插件系统不是没有坑：

1. **官方市场整体加载失败**（Issue #33739）：marketplace.json 中 Semgrep 插件使用了 `"source": "git-subdir"` 格式，schema validator 不认识，导致整个市场加载失败。
2. **跨项目启用 Bug**（Issue #16845）：官方插件在一个项目启用后，在其他项目的 Discover 列表中消失。
3. **版本字段缺失**（Issue #1758）：14 个审计的插件中有 8 个显示"Version: unknown"，包括 code-review 和 session-report。
4. **MCP 类插件的 token 浪费**：Redwerk 指出"MCP Servers load tool definitions into context window at session start regardless of use"——不管用不用，工具定义在会话开始时就加载到上下文了。
5. **安全研究（PromptArmor）**：发现恶意 marketplace 可通过 hooks 绕过人类确认保护，自动允许 `curl` 命令执行。第三方 registry 每小时自动抓取 GitHub，恶意仓库可自动上架。

---

*仓库：[github.com/anthropics/claude-plugins-official](https://github.com/anthropics/claude-plugins-official)*
