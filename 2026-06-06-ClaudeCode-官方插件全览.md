---
title: "Claude Code 官方插件全览：36 个插件的设计哲学"
description: "Anthropic 团队开源的 36 个 Claude Code 插件全览——按十大功能域逐类拆解，提取 11 个贯穿始终的 Agent 设计模式。"
---

# anthropics/claude-plugins-official 全览：36 个官方插件的设计哲学

> 这不是一份插件列表。这是 Anthropic 团队如何通过插件系统把 Claude Code 从一个"对话式编码助手"改造成"可组合的开发工作流引擎"的完整展示。36 个插件，12 个 LSP 集成，24 个功能性插件，本文按功能域逐个解读，并提取贯穿其中的设计模式。

---

Claude Code 最被低估的能力不是它写代码有多好，而是它的插件系统。2026 年 1 月 Boris Cherny 开源了 Anthropic 团队内部使用的官方插件集合——36 个插件，覆盖从需求澄清到代码审查到部署安全的完整开发生命周期。

仓库：`anthropics/claude-plugins-official` ⭐29,479 🔀3,162 📅2026-06-06 📄Apache-2.0

读完这 36 个插件，你会看到的不只是"这些插件能做什么"，而是 Anthropic 团队**如何设计 Agent 工作流**——什么该交给 subagent、什么该留在主 Agent、什么该用 hook、什么该用 skill。这些模式可以被任何支持 subagent 的 AI 编码工具复现。

---

## 插件全景：四大组件类型

Claude Code 插件系统提供四种扩展机制，每个插件可以从零到全部使用：

| 组件 | 位置 | 作用 | 谁触发 |
|------|------|------|--------|
| **Commands** | `commands/*.md` | 斜杠命令，用户手动调用 | 用户 |
| **Agents** | `agents/*.md` | 专用 subagent，独立上下文窗口 | 主 Agent 或用户 |
| **Skills** | `skills/*/SKILL.md` | 可复用的知识/流程包，自动按需加载 | 主 Agent 自动判断 |
| **Hooks** | `hooks/` | 生命周期事件触发（会话开始、编辑文件、停止等） | 系统自动 |

36 个插件中，大部分使用了其中 2-3 种。只有 LSP 集成类插件是纯配置，不含任何 Commands/Agents/Skills/Hooks。

---

## 第一部分：LSP 集成（12 个）

```
clangd-lsp  csharp-lsp  gopls-lsp    jdtls-lsp
kotlin-lsp  lua-lsp     php-lsp      pyright-lsp
ruby-lsp    rust-analyzer-lsp  swift-lsp  typescript-lsp
```

这 12 个插件都是"配置型"插件——没有 Commands、没有 Agents、没有 Hooks。它们只是在 `.claude-plugin/plugin.json` 中声明了对特定 Language Server 的支持。

安装后，Claude Code 在编辑对应语言的文件时会自动启动对应的 LSP，获得实时的类型检查、自动补全、跳转定义等能力。这是插件系统中最简单的一类——不需要任何逻辑，只需要声明能力。

---

## 第二部分：开发工作流（5 个）

这是整个插件生态的核心层。五个插件覆盖了从"写新功能"到"审查已有代码"到"清理遗留系统"的完整流程。

### feature-dev：七阶段流水线

已在前一篇深度文章中详细拆解。核心设计：

- **3 种 subagent 分工**：code-explorer（勘探）→ code-architect（设计）→ code-reviewer（审查）
- **5 个用户决策卡点**：需求确认 → 澄清问题 → 选方案 → 批准开工 → 修不修
- **Subagent 只有读权限**，写代码仅在 Phase 5 由主 Agent 执行

### code-simplifier：事后代码净化

同样已在前一篇深度文章中拆解。核心设计：

- **3 个并行 reviewer**：代码复用、代码质量、效率
- **审查后直接修复**，不是只给报告
- **运行在 Opus 上**，因为精简代码需要深度理解

### code-review：PR 审查

```
/code-review
```

启动多个专用 Agent 并行审查一个 PR，用置信度评分过滤误报。和 feature-dev Phase 6 的 code-reviewer 不同，这是**独立的 PR 审查工作流**：

1. 检查是否需要审查（已审查过的 PR 跳过）
2. 并行启动多个 reviewer Agent，每个专注一个维度：
   - CLAUDE.md 规范符合度
   - Bug 检测
   - 历史上下文（相关 commit/issue）
   - PR 历史（之前的评论和修改）
   - 代码注释准确性
3. 用**置信度过滤**（只报 ≥80 分的问题）
4. 汇总并提交 review comment

和 `/simplify` 的区别：code-review 是**审查后给人类看**，simplify 是**审查后直接改**。

### pr-review-toolkit：六个专项审查 Agent

这是一个 Agent 集合，比 code-review 更细粒度。六个 Agent 各自专攻一个维度：

| Agent | 专攻 | 典型发现 |
|-------|------|----------|
| `comment-analyzer` | 注释准确性 | 注释说"返回 User"但实际返回 `User \| null` |
| `pr-test-analyzer` | 测试覆盖 | 新代码路径没有被已有测试覆盖 |
| `silent-failure-hunter` | 静默失败 | `try { ... } catch {}` 空 catch 块 |
| `type-design-analyzer` | 类型设计 | `string` 用于有限状态值，应该用 union type |
| `code-simplifier` | 简洁性 | 不必要的抽象层 |
| `code-reviewer` | 综合质量 | 违反项目规范 |

六个 Agent 可以单独调用，也可以通过 `/review-pr` 命令一键启动。和 code-review 插件的关系是：**pr-review-toolkit 提供 Agent 集合，code-review 提供编排流程**。

### code-modernization：遗留系统现代化

这是 36 个插件中最重的一个。它为遗留代码库（COBOL、老旧 Java/C++、单体 Web 应用）设计了一个七阶段工作流：

```
assess → map → extract-rules → brief → reimagine | transform → harden
```

每个阶段有专门的命令和 Agent：

- **assess**：评估代码库规模、复杂度、风险
- **map**：映射模块边界、数据流、依赖关系（Agent: `legacy-analyst`）
- **extract-rules**：提取业务规则，在现代化前后保持行为一致（Agent: `business-rules-extractor`）
- **brief**：生成现代化需求文档
- **reimagine** | **transform**：两条路径——架构重设计或直接转换
- **harden**：加固（测试、安全检查、性能验证）（Agent: `test-engineer`, `security-auditor`, `architecture-critic`）

设计哲学很明确：**现代化失败的原因通常不是目标技术栈选错了，而是跳过了步骤**——在没理解代码前就开始改，在没提取业务规则前就重新设计架构。这个插件用 Agent 强制执行了完整流程。

---

## 第三部分：插件与技能开发（3 个）

这三个插件是"元插件"（meta-plugins）：用来开发 Claude Code 插件本身的工具。

### plugin-dev：七个专家技能

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

还有一个 `/create-plugin` 命令，走八阶段引导式创建流程。三个 Agent（`agent-creator`、`skill-reviewer`、`plugin-validator`）分别在创建、审查、验证阶段介入。

### skill-creator：技能工厂

专门用来创建和优化 skill 的工具。它不是写代码的 skill——它是**写 skill 的 skill**。

提供了：
- 从零创建 skill 的流程
- 评估 skill 效果的 benchmark 系统（带方差分析）
- 三个评估 Agent：`analyzer`（分析 skill 表现）、`comparator`（对比 skill 变体）、`grader`（按标准打分）
- 打包和发布脚本

这是一个"用 Agent 改进 Agent 的指令"的闭环。skill 写得好不好，跑一遍 eval 就知道。

### mcp-server-dev：MCP 服务器构建

三个 skill 覆盖 MCP 服务器的完整开发路径：

- `build-mcp-server`：入口 skill，根据用例选择部署模型（远程 HTTP / MCPB / 本地 stdio）、选择工具设计模式
- `build-mcp-app`：添加交互式 UI 组件（表单、选择器、确认对话框）
- `build-mcpb`：将本地 stdio 服务器打包为 MCP 包（含运行时依赖）

---

## 第四部分：项目管理（2 个）

### claude-code-setup：智能配置推荐

安装后，Agent 会自动分析你的代码库，推荐适合的自动化配置：

- MCP 服务器（如 context7 查文档、Playwright 做前端测试）
- Skills（如 plan agent、frontend-design）
- Hooks（自动格式化、自动 lint、阻止敏感文件提交）
- Subagents（安全审查、性能审查、无障碍审查）

这是一个"帮你的 Agent 配置 Agent"的插件。

### claude-md-management：CLAUDE.md 维护

两个工具：

- **`claude-md-improver`（skill）**：定期审计 CLAUDE.md 是否和当前代码库一致，自动更新过时的约定
- **`/revise-claude-md`（command）**：在会话结束时捕捉"这次会话暴露了哪些 CLAUDE.md 没写到的知识"，追加进去

设计理念：CLAUDE.md 不是写完就一劳永逸的文档。代码库在变，团队的约定在变，Agent 应该主动帮你维护它。

---

## 第五部分：安全与防护（2 个）

### security-guidance：三层安全网

David Dworken（Anthropic 安全工程师）写的。这是插件系统中结构最复杂的插件之一，使用了所有四种扩展机制。

三层防护：

1. **Pattern warnings（hook: PreToolUse）** — Agent 在调用 `Edit`/`Write` 时，正则匹配约 25 种已知危险模式。命中则注入警告提示。模式包括：
   - `yaml.load`（应为 `yaml.safe_load`）
   - `torch.load(weights_only=False)`
   - `pickle.load` 处理不可信数据
   - 裸 `innerHTML` 赋值
   - 硬编码密钥

2. **LLM diff review（hook: Stop）** — Agent 完成一轮修改后，将 diff 发给 Opus 4.7 做安全审查，高危发现反馈给 Agent，让它在返回用户前修复

3. **Agentic commit review（hook: PostToolUse on git commit）** — 通过 SDK 启动的审查器，检查注入、XSS、SSRF、硬编码密钥等约 25 类安全问题

三层逐级加码：第一层是即时正则（几乎无延迟），第二层是 LLM 审查（有延迟但更准确），第三层是提交前最后一道闸门。

### hookify：用自然语言创建 Hook

传统方式创建 hook 要手写 `hooks.json` 和 Python 脚本。hookify 把这个过程变成对话：

1. 你描述一个不想要的行为（如"不要用 `console.log` 做调试"）
2. 一个 `conversation-analyzer` Agent 分析对话，生成匹配规则
3. 自动创建 hook 配置文件

它还支持从已有对话中**反向分析**：扫描历史对话，找出反复出现的问题模式，自动生成拦截规则。

---

## 第六部分：Git 与提交（1 个）

### commit-commands：三个 Git 命令

- **`/commit`** — 分析暂存和未暂存的变更，自动生成 commit message，提交
- **`/commit-push-pr`** — 一步完成 commit → push → create PR
- **`/clean_gone`** — 清理本地已经不存在于远程的分支

减少了上下文切换。不需要切出终端、手写 commit message、手动 push。

---

## 第七部分：输出与学习风格（2 个）

### explanatory-output-style

用 SessionStart hook 注入指令，让 Agent 在回复中加入教学性解释——为什么这样实现、代码库中相关的模式、设计决策的背景。

### learning-output-style

explanatory 的增强版。除了提供教育性解释外，还会在关键决策点**停下来让你做选择**，实现"边做边学"的交互模式。

这两个插件的存在说明了一件事：**Agent 的输出风格是可编程的**。不是模型能力强弱的区别，而是系统提示注入带来的体验差异。

---

## 第八部分：前端与设计（2 个）

### frontend-design

一个 skill，让 Agent 在生成前端界面时避免"AI 美学"——那种一眼就能看出是 AI 生成的、千篇一律的 UI。自动选择：

- 大胆的配色方案
- 独特的排版
- 高冲击力的动画和视觉细节
- 根据上下文适配的实现方式

### playground

创建自包含的单文件 HTML 交互式演示。带可视化控件、实时预览、即时反馈。

---

## 第九部分：Agent SDK（1 个）

### agent-sdk-dev

为 Claude Agent SDK（Python 和 TypeScript）应用提供脚手架和验证：

- **`/new-sdk-app`** — 交互式创建新 SDK 应用，自动拉取最新版本
- **`agent-sdk-verifier`**（Agent）— 检查代码是否符合官方 SDK 文档的最佳实践

---

## 第十部分：专项工具（4 个）

### math-olympiad：对抗性数学验证

解决学术论文中暴露的问题——自验证容易被骗（arXiv:2503.21934 显示 85.7% 自验证的 IMO 成功率在人工评分下降到 <5%）。

方案：
- **上下文隔离验证**：验证者只看最终证明，看不到推理过程
- **对抗性检查模式**：不问"这个对吗？"，而是"这个是不是不小心证明了黎曼猜想？""提取一般引理，找一个 2×2 反例"
- **校准性放弃**：说不确定比猜一个错的强

### ralph-loop：持续自循环

"Ralph 是一个 Bash loop"——Geoffrey Huntley 的描述。

一个 `while true` 循环，反复把同一个 prompt 喂给 Agent，让它在迭代中持续改进，直到任务完成。灵感来自 Leslie Lamport（图灵奖得主）在 DE Shaw 期间的一个 bug 修复方法论：执行→发现错误→修改代码→重新执行，循环迭代。

### mcp-tunnels：私有 MCP 隧道

通过 Anthropic MCP 隧道将 Claude Code 连接到私有网络中运行的 MCP 服务器——**不需要开放入站端口、不需要公开暴露、不需要在服务器上加 IP 白名单**。流量走纯出站连接。

### session-report：会话分析

分析 Claude Code 会话数据，生成统计报告。包含一个 `analyze-sessions.mjs` 脚本和 HTML 模板。

---

## 11 个贯穿始终的设计模式

读完 36 个插件，以下模式反复出现：

### 1. 多 Agent 并行 > 单 Agent 全栈

几乎每个涉及分析的插件都用了 2-3 个并行 Agent，而不是一个全能 Agent。code-simplifier 三个 reviewer、feature-dev 三个 explorer、pr-review-toolkit 六个专项 Agent。这不是偶然——**上下文隔离+维度专注**是 Agent 架构的第一原则。

### 2. Subagent 不写代码

code-explorer、code-architect、code-reviewer——所有 subagent 都只有读权限。写代码的权力保留在主 Agent 中，而且通常需要用户显式批准。这不是技术限制，是**安全设计**。

### 3. 置信度过滤

code-review、pr-review-toolkit 都用了置信度评分（0-100），只报告 ≥80 的问题。传统的 lint 工具最大的问题是 noise——报 100 个 warning，99 个不重要。置信度过滤把"发现"和"噪音"分开。

### 4. Hook 做守护，Agent 做分析

security-guidance 的三层设计是一个范式：**Hook 做即时拦截**（正则在 `Edit` 时匹配危险模式），**Agent 做深度分析**（LLM 审查 diff），**Agent 做最后闸门**（commit 前审查）。Hook 快但浅，Agent 慢但深，两者互补。

### 5. 流程强制 > 自由发挥

code-modernization 的 `assess → map → extract-rules → brief → reimagine/transform → harden`，feature-dev 的七个阶段——这些插件都在**强制一个顺序**。不是 Agent 觉得该做什么就做什么，而是**流程模板锁定了步骤**。

### 6. Skill 自动加载，Command 手动触发

Skill 有 `description` 字段，Agent 会根据描述自动判断何时加载。Command 是用户手动输入 `/xxx` 触发。一个隐式、一个显式，覆盖了不同的使用场景。

### 7. 元插件闭环

plugin-dev 开发插件、skill-creator 创建 skill、mcp-server-dev 构建 MCP 服务器——这三个插件形成了一个**自己扩展自己的闭环**。用 Agent 来改进 Agent 的配置，用 Agent 来创建 Agent 的工具。

### 8. 用户决策点精确卡位

feature-dev 的五个决策点不是随意放的。每个决策点都在"信息刚好足够做出判断"的时刻。Phase 2 勘探完才能问澄清问题，Phase 3 澄清完才能设计架构。信息不够时不给选择，信息够了才问。

### 9. 事后自动 vs 事前审查

code-simplifier 是**事后自动修复**（写完了→直接改），code-review 是**事前审查**（PR 阶段→给人类看）。同一个"审查代码"的动作，根据在开发流程中的位置不同，走完全不同的处理路径。

### 10. 模型选择按任务分级

code-simplifier 用 Opus（深度理解代码），feature-dev 的 subagent 全用 Sonnet（广度搜索+常规推理）。不是越强的模型越好——任务的认知需求决定模型选择。

### 11. 开源即教学

每个插件的源码就是一份"Anthropic 教你如何设计 Agent 工作流"的教学材料。Prompt 怎么写、边界怎么设、权限怎么分、流程怎么卡——全在代码里。这是比任何文档都更有价值的 Agent 工程参考资料。

---

*仓库：[github.com/anthropics/claude-plugins-official](https://github.com/anthropics/claude-plugins-official)*
