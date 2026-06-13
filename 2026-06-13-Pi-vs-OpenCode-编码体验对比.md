---
title: "Pi vs OpenCode：作为 Claude Code 备用，编码体验是变好还是变坏？"
description: "从基础框架到增强生态，完整对比 Pi 和 OpenCode 两条技术路线的编码质量与开发者体验"
---


# Pi vs OpenCode：作为 Claude Code 备用，编码体验是变好还是变坏？

## 从基础框架到增强生态，完整对比两条技术路线

---

想象一个场景：你主力用 Claude Code，日常写代码的体验已经相当顺畅——模型聪明，工具顺手，skill 生态也攒了不少。但万一哪天 Claude Code 不能用了呢？也许是 ToS 变更（2026 年 1 月 Anthropic 确实封锁过 OpenCode 使用 Claude 订阅 [Reddit r/opencodeCLI]），也许是网络封锁让你连不上 API，也许是价格突然翻倍让个人开发者扛不住。不管原因是什么，你需要一个备用方案。

放眼 2026 年 6 月的开源编码 Agent 市场，两个最常被提起的名字是 **Pi** 和 **OpenCode**。它们都是开源项目，都支持接入多种大模型（Claude、GPT、Gemini、本地模型），都在快速增长——Pi 有 60K+ GitHub stars，OpenCode 更是达到了 140K+ [GitHub]。很多对比文章写到这里就开始下结论了："Pi 更好"或者"OpenCode 更强"。

但这类对比忽略了一个关键事实：**"Pi"和"OpenCode"不是单一产品。**

打个比方，这就像比较"VS Code"和"JetBrains"——光说编辑器本体意义不大，真正决定体验的是你装了什么插件。Pi 和 OpenCode 都是**基础框架（harness）**，类似 VS Code 本体。但它们各自有丰富的增强生态，装上不同的增强之后，体验完全不同。一个裸装的 Pi 只有 2K tokens 的系统提示，极简到几乎"什么都不管"；而装了 Oh-My-Pi 增强包之后，它变成一个拥有 32 个工具、完整 LSP 和调试器集成的重型编码环境 [plugin-ecosystem-research]。OpenCode 也一样——裸版自带 10K tokens 的系统提示和基础 LSP，装上 OMO 插件后则获得 11 个专业 Agent 的并行编排能力 [plugin-ecosystem-research]。

所以真正要回答的问题其实是三个：

- **Pi 生态**（基础框架 + 增强选项）的整体编码体验如何？
- **OpenCode 生态**（基础框架 + 增强选项）的整体编码体验如何？
- 两个生态之间，谁更适合做 Claude Code 的备用？

本文的对比路径是这样走的：首先看基础框架——Pi 本体和 OpenCode 本体各自的"裸"状态是什么样的；然后展开增强生态的全景图，看看各自有哪些增强选项、最流行的组合是什么；最后在"增强模式"下做多个维度的深度对比。

需要提前说明的是，软件安装、环境配置这些不构成真正障碍的事情不在讨论范围内，API Key 获取也一样——国内主流大模型都支持。另外，Aider、Cline、Cursor 等其他工具也不在本文覆盖范围内，这里只聚焦 Pi 和 OpenCode 这两条技术路线。

---

## 第一部分：基础框架对比

### 什么是"基础框架"？

在聊增强之前，得先搞清楚 Pi 和 OpenCode 各自"裸"着的时候长什么样。所谓基础框架，就是不装任何增强包或插件，只用项目本身提供的原生能力。这个状态决定了后续增强的起点在哪里、空间有多大。

### Pi：极简主义的极致

Pi 由 Mario Zechner 和 Nutrient 公司（原 PSPDFKit，如果你用过 libGDX 就知道这位作者的份量）维护 [plugin-ecosystem-research]。它的系统提示只有大约 **2K tokens**——这个数字意味着模型几乎拿到了全部可用的上下文窗口。原生能力包括 Bash 执行、文件读写、搜索和基础编辑，够用来写代码，但仅此而已。没有内置 LSP，没有子 Agent，没有 MCP 支持。

这种设计是有意为之。Pi 的哲学是"最小核心"——把所有增强空间留给用户自己选择 [plugin-ecosystem-research]。好处是启动极快、token 消耗极低、本地模型跑起来毫无压力；代价是你需要自己决定要补什么能力。

### OpenCode：开箱即用的"瑞士军刀"

OpenCode 由 AnomalyCo 维护，社区规模是 Pi 的两倍多（140K+ stars vs 60K+）[GitHub]。它的系统提示约 **10K tokens**，是 Pi 的五倍，但这些 token 换来了不少原生能力：基础 LSP 集成、子 Agent（通过 @mention 调用）、MCP 服务器支持、自定义工具、REST API、以及一套 Hooks 系统让插件可以在启动时注入逻辑 [plugin-ecosystem-research]。

界面方面，OpenCode 同时提供 CLI TUI 和桌面 GUI 两种模式，Pi 只有 CLI TUI。对于习惯图形界面的开发者来说，OpenCode 的上手门槛更低一些。

### 放在一起看

| 维度 | Pi（基础） | OpenCode（基础） |
|------|-----------|------------------|
| 系统提示大小 | ~2K tokens | ~10K tokens |
| 内置 LSP | 无 | 基础 |
| 子 Agent | 无 | @mention 调用 |
| MCP 支持 | 无（需 pi-mcp-adapter） | 原生支持 |
| Hooks 系统 | 无 | 有（插件可在启动时注入） |
| 界面 | CLI TUI | CLI TUI + 桌面 GUI |
| 本地模型优势 | MLX/GGUF 原生优化 | Ollama/LM Studio 支持 |
| 社区规模 | 60K+ stars | 140K+ stars |

简单说，基础框架层面 OpenCode 内置能力更多，像一个装好了常用插件的 VS Code；Pi 更极简，像一个纯净安装的编辑器——但也正因如此，它为增强留出了更大的空间。这个差异会直接影响后面增强生态的设计思路：Pi 的增强倾向于"从零搭建"，OpenCode 的增强倾向于"在已有能力上叠加"。

---

## 第二部分：增强生态全景

理解了基础框架的差异之后，接下来看看两个生态的增强选项。就像 VS Code 有插件市场一样，Pi 和 OpenCode 都有自己的增强生态。选择不同的增强，得到的是截然不同的编码体验。

### Pi 增强生态：3,811 个包的"长尾市场"

Pi 的包目录托管在 pi.dev/packages，截至 2026 年 6 月共有 **3,811 个包** [pi.dev/packages]。类型涵盖 package、extension、skill、prompt、theme，安装方式是 `pi install npm:<package>`。这个规模在编码 Agent 领域算是相当丰富了。

最让人意外的是，**Oh-My-Pi 不是 Pi 生态的代名词**——它只是 3,811 个包中最知名的一个。很多文章把"Pi vs OpenCode"写成"Oh-My-Pi vs OMO"，这就好比把"VS Code 生态"等同于"装了某个特定插件包的 VS Code"，显然不全面。来看看 Pi 生态里实际有哪些值得关注的增强：

**context-mode**（月下载 131.5K）是下载量最高的包 [pi.dev/packages]。它是一个 MCP 插件，核心能力是节省 98% 的上下文窗口——通过沙盒执行和 FTS5 知识库实现意图驱动的搜索，而不是把所有文件内容塞进上下文。对于那些代码库庞大、经常遇到"上下文不够用"问题的开发者来说，这个包几乎是必装的。

**pi-subagents**（月下载 103.2K）让 Pi 获得了子 Agent 委派能力 [pi.dev/packages]。它支持链式执行和并行模式，弥补了 Pi 基础框架没有子 Agent 的短板。有意思的是，这个包的下载量说明大量用户选择"不装 Oh-My-Pi，但需要子 Agent"的路径。

**pi-mcp-adapter**（月下载 99.2K）为 Pi 补上了 MCP 协议支持 [pi.dev/packages]。Pi 基础框架不原生支持 MCP，这个适配器让 Pi 能接入整个 MCP 生态的服务器——对于依赖 MCP 工具的开发者来说是刚需。

**pi-lens**（月下载 26K）走的是另一条路：实时代码反馈 [pi.dev/packages]。它把 LSP、linter、formatter、type-checking 的结果实时喂给模型，让 Agent 在写代码的同时就能看到编译错误和类型问题，而不是写完才发现一堆红线。

**gentle-pi**（月下载 14.7K）是一个比较特别的包——它不是给 Pi 加功能，而是把 Pi 变成一个"高级架构师"模式 [plugin-ecosystem-research]。内置 SDD/OpenSpec 流程、严格 TDD、审查守卫，适合需要规范化开发流程的团队。它和 Oh-My-Pi 是互斥的——两者代表了不同的增强方向。

**piolium**（月下载 29.1K）专注安全方向，提供多阶段安全审计和专家子 Agent [plugin-ecosystem-research]。在安全敏感的项目中，这个包让 Pi 能系统性地检查代码漏洞，而不只是依赖模型的"直觉"。

此外还有 **pi-studio**（双面板浏览器工作区，带注释、批评、测验功能）、**pi-spark**（小型精选扩展集合，适合不想自己挑选的用户）、**pi-twins**（同一 prompt 发给两个模型，合成一个答案）等方向各异的包 [plugin-ecosystem-research]。Pi 生态给人的感觉是：不管你有什么需求，大概率已经有人做了对应的包。

至于 **Oh-My-Pi**，它的定位是"最完整的一站式增强"——一次性加入 32 个工具、Hashline 编辑、13 个 LSP 操作、27 个 DAP 操作 [plugin-ecosystem-research]。如果 Pi 是 VS Code，Oh-My-Pi 就是"一键装好所有常用插件"的套装。代价是系统提示从 2K 暴涨到 22K tokens，但换来的是一个功能齐全的编码环境。

Pi 增强的真正魅力在于灵活性。用户完全可以自由组合：只装 context-mode + pi-subagents 而跳过 Oh-My-Pi；用 gentle-pi 的架构师模式替代 Oh-My-Pi 的全功能路线；甚至让 Pi 帮忙写自定义扩展——这在社区里是常见做法，Reddit 上经常有人分享"让 Pi 帮我写了一个专门处理我们团队发布流程的扩展" [Reddit r/PiCodingAgent]。

### OpenCode 增强生态：重型插件 + 外部编排

OpenCode 的增强来源更多元：opencode.im 插件中心、npm 包、GitHub 仓库，以及 Composio 等第三方推荐 [opencode.im]。虽然没有 Pi 那样统一的包目录和精确的下载量统计，但头部插件的影响力不容小觑。

**Oh-My-OpenCode（OMO）** 是 OpenCode 生态最重量级的插件，50K stars，由 Yeongyu Kim 和 Sisyphus Labs 维护 [GitHub]。它加入了 11 个专业 Agent（Sisyphus 负责编排、Prometheus 负责规划、Atlas 负责执行、Oracle 负责架构咨询等），以及 ultrawork 多 Agent 并行编排模式和 v4.0 的 Team Mode。装上 OMO 之后，OpenCode 从"一个聪明的 Agent"变成"一支 Agent 团队"。代价是系统提示增加 15-25K tokens，以及一个需要认真对待的成本风险——2026 年 3 月的 Bug #2571 中，Gemini 模型在 OMO 编排下陷入无限循环，3.5 小时内发出 809 次请求，账单高达 $438 [GitHub Issues]。

**OMO-Slim** 是 OMO 的轻量替代方案，由 alvinunreal 维护 [GitHub]。它保留了 6 个核心 Agent（Orchestrator、Explorer、Oracle、Librarian、Designer、Fixer），提供上下文隔离和并行执行，但去掉了完整 OMO 的臃肿部分。对于那些"想要多 Agent 但不想承担全部开销"的用户，这是一个务实的中间选择 [dataleadsfuture.com]。

**Superpowers** 走了一条完全不同的增强路线——它不加 Agent，而是加 **Skills 系统** [blog.fsck.com]。作者 Jesse Vincent 利用了 OpenCode 的 Hooks 机制，在会话启动时注入 find_skills 和 use_skill 两个工具，让 Agent 能主动发现和激活各种技能。核心亮点是头脑风暴技能：强制要求模型在写代码之前先做规划和澄清，而不是直接动手。这个插件同时支持 Claude Code、OpenCode、Codex CLI，跨平台设计降低了锁定风险。不过社区报告显示它的 token 消耗可能是裸版的 20 倍——Reddit 用户报告同一项目用了 68.5M tokens，而裸 OpenCode 只用了 3.3M [Reddit r/opencodeCLI]。

**Composio** 把 OpenCode 连接到了 1,000+ 外部应用 [composio.dev]。GitHub、Linear、Jira、Slack、Figma、Stripe——只要你能想到的开发工具链，几乎都能通过 Composio 的远程 MCP 服务器接入。这让 OpenCode 不只是写代码的 Agent，而是变成了整个开发工作流的执行者。

其他值得关注的还有：**Context7**（获取最新库文档，防止模型幻觉出过时的 API）、**Opencode Mem**（跨会话持久记忆，记住项目的包管理器、架构模式等上下文）、**Envsitter Guard**（保护 .env 文件和密钥不被 Agent 意外读取或修改）、**Dynamic Context Pruning**（修剪过期上下文，降低 token 消耗）、**Daytona Sandbox**（在隔离沙盒中运行 Agent 会话）[composio.dev]。

### 生态层级图

把两个生态的增强层级画出来，能更直观地看到它们的结构：

```
Pi 生态                          OpenCode 生态
━━━━━━━━━━━                      ━━━━━━━━━━━━━
Pi（基础 ~2K tokens）            OpenCode（基础 ~10K tokens）
  │                                │
  ├── Oh-My-Pi（重型，22K）       ├── OMO（重型，+15-25K）
  ├── gentle-pi（架构师模式）     ├── OMO-Slim（中型，6 Agent）
  ├── context-mode（上下文优化）  ├── Superpowers（Skills 系统）
  ├── pi-subagents（子 Agent）    ├── Composio（外部集成）
  ├── pi-lens（LSP 反馈）        ├── Context7（文档查询）
  ├── pi-spark（精选集合）        ├── Opencode Mem（记忆）
  └── 3,800+ 其他包               └── 更多插件...
```

两个生态都遵循同一模式：极简或适中的基础框架 + 可插拔的增强层。Oh-My-Pi 和 OMO 分别是各自生态最知名的增强——但远不是唯一选择。用户可以从"裸"到"全装"之间自由调节增强级别，这才是两个生态真正的竞争力所在。

---

## 总览表：增强模式下的核心差异

当讨论从"基础框架"升级到"增强模式"，最流行的组合分别是 Pi + Oh-My-Pi 和 OpenCode + OMO。把这两个"满配"版本放在一起，差异就清晰了：

| 维度 | Pi + Oh-My-Pi | OpenCode + OMO |
|------|---------------|----------------|
| **设计哲学** | 极简核心 + 重型增强层 | 开箱即用 + 编排叠加 |
| **系统提示** | Pi 2K → Oh-My-Pi ~22K | OpenCode ~10K → OMO ~15-25K |
| **代码编辑** | Hashline（内容哈希锚点） | 传统 diff（字符串匹配） |
| **内置工具数** | 32（Oh-My-Pi 提供） | ~15（OpenCode）+ OMO Agent 体系 |
| **LSP 集成** | 13 个 LSP 操作（Oh-My-Pi 内置） | OpenCode 内置基础 LSP |
| **调试器** | 27 个 DAP 操作（Oh-My-Pi 内置） | 无内置调试器 |
| **多 Agent 编排** | subagent（顺序执行） | Sisyphus/Prometheus 等 11 Agent 并行 |
| **GitHub Stars** | Pi 60K+ / Oh-My-Pi ~3K | OpenCode 140K+ / OMO ~50K |
| **增强包总量** | 3,811（pi.dev/packages） | 数十个插件 + MCP 服务器 |
| **SKILL.md 兼容** | 兼容（同格式） | 兼容（直接扫描 .claude/skills/） |

这张表里有几个值得注意的地方。系统提示方面，Pi 的"2K → 22K"和 OpenCode 的"10K → 25K"看似终点差不多，但起点不同意味着"不装增强"时的体验差距很大——Pi 裸版几乎什么都不做，OpenCode 裸版已经能应对大部分日常编码。代码编辑机制的差异（Hashline vs 传统 diff）会在后面的"代码生成质量"维度中展开讨论，这里只需要知道：Hashline 通过内容哈希锚定代码行，解决了"编辑时文件已变"的经典问题 [plugin-ecosystem-research]。

多 Agent 编排是 OMO 的核心卖点，但"并行"不一定是好事——后面会看到基准测试数据揭示的反直觉结论。而调试器这一栏，Pi + Oh-My-Pi 的 27 个 DAP 操作是目前所有编码 Agent 中最强的调试集成，这个优势在 debug 密集型场景中几乎是决定性的。

---

## 能力边界表：什么场景该用谁

不同的编码场景对 Agent 的要求不一样。下面这张表不是"谁更好"的简单判断，而是"在什么条件下谁更合适"的场景匹配：

| 场景 | Pi + Oh-My-Pi | OpenCode + OMO |
|------|---------------|----------------|
| **单文件修复** | ⭐⭐⭐⭐⭐（快，token 少） | ⭐⭐⭐⭐ |
| **多文件架构重构** | ⭐⭐⭐ | ⭐⭐⭐⭐⭐（ultrawork 并行） |
| **调试密集型 Bug** | ⭐⭐⭐⭐⭐（内置 DAP） | ⭐⭐（只能 print 猜测） |
| **研究 + 实现混合** | ⭐⭐⭐ | ⭐⭐⭐⭐⭐（Librarian + 编排） |
| **本地模型开发** | ⭐⭐⭐⭐⭐（MLX/GGUF 优势） | ⭐⭐⭐ |
| **隐私敏感/离线** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |
| **Skill 生态复用** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |

这张表的读法是：如果你的工作以单文件修复和调试为主（比如维护一个成熟项目，日常修 Bug），Pi + Oh-My-Pi 在这两个场景的优势是实质性的——2K 的基础系统提示意味着更快的响应和更低的 token 消耗，27 个 DAP 操作让调试不再是"猜 → 改 → 试"的循环。但如果你经常做跨多个文件的架构重构，或者需要一边研究文档一边实现功能，OpenCode + OMO 的多 Agent 并行编排就有用武之地了——Sisyphus 编排器能同时派出多个 Agent 处理不同文件，Librarian Agent 专门负责查找文档和代码库信息 [plugin-ecosystem-research]。

本地模型和离线场景是 Pi 的独有优势。Pi 原生支持 MLX 和 GGUF 格式，Mac 上本地模型性能有 2-3 倍的提升 [plugin-ecosystem-research]。而 OMO 的多 Agent 编排需要同时运行多个模型实例，对本地资源的消耗是普通开发者难以承受的。如果你的"备用方案"场景包含完全离线的需求，Pi 几乎是唯一的选择。

---

## 维度1：代码生成质量

### 同模型不同 Agent，输出质量一样吗？

一个常见的误解是：既然 Pi + Oh-My-Pi 和 OpenCode + OMO 背后跑的都是同一个大模型（Opus 4.7、GPT-5.5 或 Gemini 2.5 Pro），那代码生成质量应该差不多。实际上，系统提示的结构和大小会显著影响模型的"思考空间"——就像同样一个厨师，给他一个拥挤杂乱的厨房和一间井然有序的工作室，出品质量不会一样。

grigio.org 的实测提供了一个直观证据：同一道物理题（计算抛体运动轨迹），用同一个模型，Pi 一次通过给出了正确答案，OpenCode 却在数值计算上出了错 [grigio.org]。这不是模型能力的差异，而是系统提示"挤占"了模型推理空间的差异。Pi 原生只有 2K tokens 的系统提示，留给模型思考和输出的空间远大于 OpenCode 的 10K [plugin-ecosystem-research]。

但这个结论不能简单化。系统提示的"税"要看它花在什么地方。

Pi 原生 2K tokens 几乎不含任何工具描述——极简到了"什么都不告诉你"的程度。Oh-My-Pi 把它拉到 22K tokens，但这 22K 里包含 32 个工具的精确描述、13 个 LSP 操作和 27 个 DAP 操作的详细说明 [plugin-ecosystem-research]。每一行都有功能对应，不是废话。OpenCode 原生 10K tokens 里包含基础工具、LSP 和子 Agent 的描述，OMO 叠加 15-25K tokens 后加入 11 个 Agent 的配置和编排逻辑 [plugin-ecosystem-research]。

换句话说，Oh-My-Pi 的 22K 是"工具说明书"，OMO 的 25K 是"团队组织图"——两种不同的开销方向，对模型行为的影响也不同。工具说明书让模型更精确地使用每个工具，团队组织图让模型学会何时调用哪个 Agent。

### Hashline 编辑：一个真正改变游戏规则的创新

在代码编辑这个环节，Oh-My-Pi 引入了一个名为 **Hashline** 的机制，它解决了一个所有编码 Agent 都头疼的问题：编辑的时候文件已经变了。

传统 diff 编辑基于行号和字符串匹配——"把第 50 行的 `x = 10` 改成 `x = 20`"。但如果在这个过程中，另一个操作（比如格式化、自动导入、或者另一个 Agent 的并行编辑）在文件前面插入了几行，第 50 行就不再是原来的内容了。编辑失败，Agent 需要重新读取文件、重新定位、重新尝试。

Hashline 的做法是给每一行代码附加一个内容哈希——编辑时引用的是哈希值而非行号。即使文件发生了变化，只要目标行的内容没变，哈希就匹配，编辑照样成功。如果目标行内容也变了（哈希不匹配），系统会自动重新读取文件而不是盲目执行。

这个机制的效果有数据支撑。在 Grok Code Fast 1 基准测试中，引入 Hashline 后编辑成功率从 6.7% 飙升到 68.3%——十倍级的提升 [plugin-ecosystem-research]。OMO 后来也借鉴了 Hashline 的实现，同样获得了从 6.7% 到 68.3% 的提升 [plugin-ecosystem-research]。但要注意：OMO 只在部分 Agent 中实现了 Hashline，其他 Agent 仍然使用传统 diff，编辑成功率约 30% [plugin-ecosystem-research]。

来看一个具体的对比：

```
场景：文件在编辑过程中被格式化工具自动修改

[Oh-My-Pi / OMO Hashline Agent]
> Edit LINE#abc123: change `x = 10` to `x = 20`
✅ Hash matched → edit applied (即使行号已从 50 变成 53)

[传统 diff Agent]
> Edit line 50: change `x = 10` to `x = 20`
❌ Error: context expired, file has changed
→ Agent 重新读取文件 → 重新定位到第 53 行 → 重新尝试编辑
→ 浪费了一轮对话和额外的 token
```

在多 Agent 并行场景下（OMO 的 ultrawork 模式），Hashline 的优势更加明显。多个 Agent 同时修改同一个文件时，传统 diff 几乎必然遇到冲突，而 Hashline 只要目标内容没被其他 Agent 改过，就能稳定命中。

### OMO 编排：什么时候值得启动？

OMO 的多 Agent 编排是它最大的卖点，但这个能力有一个容易被忽略的触发条件：必须在 prompt 中包含 `ultrawork` 关键字。没有这个关键字，OMO 的行为和普通 OpenCode 几乎没有区别 [plugin-ecosystem-research]。

更关键的是，即使触发了 ultrawork，也不是所有任务都能受益。对于单上下文的推理任务（比如"给这个函数写单元测试"），编排模式会消耗 3 倍的 token，但产出和单 Agent 基本相同 [plugin-ecosystem-research]。编排真正有效的场景是：跨多个文件的架构变更、需要查文档的研究密集型任务、以及大型 feature 的分步实现。这些场景下，Sisyphus 编排器能把任务拆解给不同专长的 Agent 并行处理，效率提升是实实在在的。

### 质量总结

| 指标 | Pi + Oh-My-Pi | OpenCode + OMO |
|------|---------------|----------------|
| 单文件生成质量 | ⭐⭐⭐⭐⭐（系统提示精简，模型思考空间大） | ⭐⭐⭐⭐ |
| 多文件协调质量 | ⭐⭐⭐（单 Agent 顺序执行） | ⭐⭐⭐⭐⭐（多 Agent 并行编排） |
| 编辑成功率 | 68.3%（Hashline 全覆盖） | 68.3%（Hashline Agent）/ ~30%（传统 diff Agent） |
| Token 效率 | 高（Pi 原生 2K 基础） | 中低（OMO 启动即 15-25K） |

简单说：单兵作战 Pi + Oh-My-Pi 更精准，团队作战 OpenCode + OMO 更高效。选哪个，取决于你的日常任务是"一个人修一栋楼"还是"指挥一个施工队"。

---

## 维度2：调试与错误修复效率

这是两个生态差距最大的维度之一。差距不是"好一点"的程度，而是"有"和"没有"的区别。

### Pi + Oh-My-Pi：调试器是 first-class tool

Oh-My-Pi 内置了完整的 DAP（Debug Adapter Protocol）集成，提供 27 个调试操作 [plugin-ecosystem-research]。这意味着 Agent 可以像一个人类开发者一样使用调试器：设置断点、暂停执行、检查调用栈帧、读取局部变量的值、单步执行。支持的后端包括 lldb（C/C++/Swift）、dlv（Go）、debugpy（Python）等主流调试器 [plugin-ecosystem-research]。

这个能力的意义在于：当测试失败或运行时行为不符合预期时，Agent 不需要"猜"。它能直接启动调试器，在关键位置设断点，观察程序运行时的实际状态——就像人类开发者 F5 启动调试一样自然。调试不是事后补救，而是编码流程的一部分。

### OpenCode + OMO：只能靠"看 → 猜 → 改 → 试"

OpenCode 没有内置调试器，OMO 也没有加入调试能力 [plugin-ecosystem-research]。OMO 的 Oracle Agent 可以做架构层面的咨询和分析，但它不能直接启动调试器、设断点或读取运行时变量 [plugin-ecosystem-research]。

这意味着当遇到复杂 Bug 时，OpenCode + OMO 的 Agent 只能走一条老路：读错误日志 → 猜测原因 → 修改代码 → 重新运行 → 看结果 → 如果还不对就重复。这个循环在简单 Bug 上效率还行，但碰到需要观察运行时状态才能定位的问题（并发条件竞争、闭包变量捕获、递归边界条件），效率会急剧下降。

### 同一个 Bug 的修复流程对比

来看一个具体场景：Python 项目中 `calculate_total()` 函数在某些条件下返回 `None`，导致测试失败。

```
[Pi + Oh-My-Pi 的修复流程]

1. 运行测试 → AssertionError: expected 150.0, got None
2. 启动 debugpy，断点设在 return 语句
3. 检查 locals: items = [], discount = 0.1
4. 发现 items 为空列表 → 追溯到调用方
5. 在调用方设断点 → 发现 data_loader 返回了空数据
6. 定位到 data_loader 的过滤条件有 Bug → 修复
7. 重跑测试 → 全部通过
耗时：约 3 分钟 | 一次修复 | 确定性定位

[OpenCode + OMO 的修复流程]

1. 运行测试 → AssertionError: expected 150.0, got None
2. 读取 calculate_total() 源码 → 猜测"可能是 items 为空"
3. 加了一个 if not items: return 0 的保护 → 重跑测试
4. 测试仍然失败（另一个测试期望 None 而非 0）
5. 重新读取相关代码 → 猜测"可能是 data_loader 的问题"
6. 在 data_loader 里加了 print 语句 → 重跑 → 看到空输出
7. 修复 data_loader 的过滤条件 → 重跑测试
8. 全部通过
耗时：约 8 分钟 | 2-3 次迭代 | 猜测式定位
```

这个对比不是要说明 OpenCode + OMO"不行"——它最终也能修复 Bug。但效率差距是显而易见的：3 分钟 vs 8 分钟，确定性定位 vs 猜测式迭代。当你的项目里有几十个这样的 Bug 要修，差距会被放大到显著影响工作节奏的程度。

如果你的日常工作中 debug 占比较大——复杂业务逻辑、并发问题、性能调优——Pi + Oh-My-Pi 的调试器集成是一个实质性优势，不是锦上添花而是核心能力。OpenCode + OMO 在这个维度明显落后，它更适合那些"代码写对了就能跑"的场景，而不是"需要钻进运行时才能搞清楚"的场景。

---

## 维度3：多文件重构与架构变更

如果说调试是 Pi + Oh-My-Pi 的主场，那多文件重构就是 OpenCode + OMO 的天下。

### OpenCode + OMO 的编排利器

OMO 的 ultrawork 模式在大规模重构时真正展示了它的价值。Sisyphus 编排器负责把一个大任务拆解成多个子任务，Prometheus 负责制定整体规划，然后 Atlas、Fixer 等执行 Agent 并行处理不同文件的修改 [plugin-ecosystem-research]。v4.0 引入的 Team Mode 更进一步：一个 lead agent 编排最多 8 个按类别专业化的成员 Agent，每个人管自己擅长的那块 [plugin-ecosystem-research]。

适合的场景包括：跨 10+ 文件的 API 迁移、整体架构模式变更（比如从 MVC 迁移到 Clean Architecture）、大型 feature 的多模块同步实现。在这些场景下，单 Agent 顺序处理不仅慢，还容易"改了前面忘了后面"。多 Agent 并行处理则能同时覆盖多个文件，由编排器确保一致性。

### Pi + Oh-My-Pi 的"精确手术刀"

Pi + Oh-My-Pi 没有多 Agent 并行编排能力，但它在另一个维度上提供了重构支持：LSP 集成。Oh-My-Pi 内置了 13 个 LSP 操作，其中包括 `workspace/willRenameFiles`——当一个文件被重命名时，LSP 自动找到所有引用该文件的位置并更新 [plugin-ecosystem-research]。

此外，pi-subagents 包（月下载 103.2K）提供了子 Agent 委派能力，但它是顺序执行而非并行 [pi.dev/packages]。这意味着 Pi 也能做分工协作，只是不能同时处理多个文件。

### 一个反直觉的基准数据

这里有一个值得深思的数据：OMO 的 Ultrawork 模式在标准编码基准测试中，通过率为 69.2%，反而低于 vanilla OpenCode（不装 OMO）的 73.1%——而它用了 3 倍的请求量和多出 10 分钟的执行时间 [plugin-ecosystem-research]。

这个数字的解读是：编排开销在标准编码任务上抵消了并行优势。启动 Sisyphus 编排器、分配任务、协调 Agent 之间的依赖——这些"管理工作"本身就需要 token 和时间。当任务不够复杂（比如单文件的函数修改），这些管理开销就是纯浪费。只有在研究密集型任务（需要查文档、理解复杂 API、分析多个模块的交互）时，编排才真正带来净收益 [plugin-ecosystem-research]。

### 符号重命名的对比

```
场景：将 utils.formatBytes() 重命名为 utils.formatSize()，
      该函数在 3 个文件中有 5 处引用。

[Pi + Oh-My-Pi（LSP 集成）]
> Rename symbol: formatBytes → formatSize
✅ LSP: Found 5 references across 3 files
   - src/utils.ts:15 (definition)
   - src/components/FileList.tsx:42 (call)
   - src/components/FileList.tsx:87 (call)
   - src/pages/Dashboard.tsx:23 (call)
   - src/api/response.ts:67 (call)
✅ Updated all references automatically
✅ Verified: formatBytes 0 matches remaining
耗时：~5 秒 | 确定性完成

[OpenCode + OMO（ultrawork）]
> ultrawork: rename formatBytes to formatSize across the codebase
[Prometheus] Analyzing scope... found 5 references in 3 files
[Sisyphus] Delegating to 3 agents (one per file)...
[Atlas-1] Updated src/utils.ts ✅
[Atlas-2] Updated src/components/FileList.tsx ✅
[Atlas-3] Updated src/pages/Dashboard.tsx + src/api/response.ts ✅
耗时：~47 秒 | 12K tokens | 确定性完成
```

在这个具体场景中，Pi + Oh-My-Pi 的 LSP 集成更快更省——因为符号重命名恰好是 LSP 最擅长的事。但如果场景变成"把所有用了 deprecated API 的地方迁移到新 API，同时更新对应的类型定义和测试"，OMO 的多 Agent 并行就有优势了——因为这种任务需要理解上下文、做判断、写新代码，不是简单的文本替换。

---

## 维度4：Skill 生态与 Claude Code 兼容性

对于 Claude Code 用户来说，选择备用方案时一个很实际的考虑是：已经积累的 Skill 能不能直接复用？

### SKILL.md 已经成为事实标准

2026 年的 Agent Skill 生态已经收敛到 SKILL.md 格式。这个格式最初由 Claude Code 推广开来，现在已经被主流编码 Agent 普遍采纳 [plugin-ecosystem-research]。各平台的 skill 存放路径分别是：

- Claude Code：`~/.claude/skills/*/SKILL.md`
- OpenCode：`.opencode/skills/` + `.claude/skills/` + `.agents/skills/`（全部扫描）
- Pi：`~/.pi/agent/skills/**/SKILL.md`
- Codex CLI：`~/.codex/skills/`

这意味着一个 SKILL.md 文件写好之后，理论上在所有平台上都能用——格式是统一的，区别只在于存放路径。

### OpenCode 的 Skill 兼容性：最"懒"的方案

OpenCode 做了一个特别省事的设计：它直接扫描 `.claude/skills/` 目录 [plugin-ecosystem-research]。也就是说，如果你电脑上已经有 Claude Code 的 skill，装好 OpenCode 之后什么都不用做——OpenCode 会自动发现并加载它们。零修改，零迁移，直接可用。

Superpowers 插件进一步强化了 Skill 生态：它在 OpenCode 的 Hooks 系统中注入了 `find_skills` 和 `use_skill` 两个工具，让 Agent 能主动发现和激活各种技能 [blog.fsck.com]。而且 Superpowers 内置了 Claude Code skill 格式翻译，能自动把 Claude Code 的 skill 引用转换为 OpenCode 的等效路径 [blog.fsck.com]。

在社区层面，OMO 自身就作为 skill 发布在 agensi.io 和 claudemarketplaces.com 上 [plugin-ecosystem-research]。Composio 等第三方也有专门的"10 Best OpenCode Skills"推荐列表 [composio.dev]。140K+ stars 的用户基数带来了大量社区贡献的 skill。

### Pi 的 Skill 兼容性：同样完善

Pi 这边的 Skill 兼容性同样令人满意。官方的 pi-skills 仓库（badlogic/pi-skills）明确标注兼容 Claude Code、Codex CLI、Amp 和 Droid [GitHub]。通用的 skill.sh 安装器同时支持 Pi、Claude Code、Goose、Windsurf 等多个平台 [plugin-ecosystem-research]。

pi.dev/packages 的 3,811 个包中有一部分是 skill 类型，pi-skill-deck 提供了一个两栏式 skill 浏览器方便查找 [plugin-ecosystem-research]。LobeHub marketplace 也有 Pi skill 专区。不过客观说，由于 Pi 的社区规模（60K+ stars）比 OpenCode（140K+ stars）小，社区贡献的 skill 总量也相应少一些。

### 从 Claude Code 迁移的兼容性

| 迁移路径 | 兼容性 | 说明 |
|----------|--------|------|
| Claude Code → OpenCode | 100% | OpenCode 直接扫描 .claude/skills/，零修改 |
| Claude Code → Pi | 100% | 同 SKILL.md 格式，复制到 ~/.pi/agent/skills/ 即可 |
| Claude Code → OMO | 100% | OMO 运行在 OpenCode 上，继承其兼容性 |
| Claude Code → Oh-My-Pi | 100% | Oh-My-Pi 运行在 Pi 上，继承其兼容性 |
| Claude Code → Superpowers | 100% | Superpowers 内置 Claude Code skill 格式翻译 |

这是一个好消息：不管你选择哪条备用路线，已有的 Claude Code skill 投资都不会浪费。OpenCode 在这一点上更"懒"——连目录都不用换；Pi 生态的 skill 总量目前少于 OpenCode（社区规模差距），但官方 skill 质量高，Mario Zechner 亲自维护 [GitHub]。

---

## 维度5：Token 效率与成本风险

用编码 Agent 写代码是有真金白银成本的——API 调用按 token 计费。增强级别越高，token 开销越大，这在两个生态中都是如此。但真正的差异不在于"贵不贵"，而在于"会不会突然失控"。

### 启动开销对比

| 配置 | 启动 Token 消耗 | 说明 |
|------|-----------------|------|
| Pi（裸） | ~2,000 | 极简核心，几乎不占上下文 |
| Pi + Oh-My-Pi | ~22,000 | 32 个工具描述，每轮都占上下文 |
| Pi + context-mode | ~5,000 | 上下文优化，运行中节省 98% |
| OpenCode（裸） | ~10,000 | 内置较多能力 |
| OpenCode + OMO | ~15,000-25,000 | 11 个 Agent 配置和编排逻辑 |
| OpenCode + Superpowers | ~12,000-15,000 | Skills 系统 + 规划模式 |

启动开销只是"入场费"。运行中的 token 效率同样重要：Pi + Oh-My-Pi 虽然启动需要 22K tokens，但它是单 Agent 顺序执行，每轮对话只消耗一份上下文。OpenCode + OMO 的 ultrawork 模式下多个 Agent 各自维护独立上下文，总 token 消耗高但分摊到多个并行任务 [plugin-ecosystem-research]。

Superpowers 的 token 消耗需要特别关注。Reddit 用户报告了一个案例：同一个项目，装了 Superpowers 的 OpenCode 消耗了 68.5M tokens，而裸版 OpenCode 只用了 3.3M——差距是 20 倍 [Reddit r/opencodeCLI]。根因在于 Superpowers 的规划模式会在写代码之前做大量头脑风暴和澄清，这些"思考过程"全部计入 token 消耗。

### OMO 的计费风险：Bug #2571 的警示

2026 年 3 月，OMO 社区曝出一个严重的计费风险事件：Bug #2571 记录了 Gemini 模型在 OMO 编排下陷入无限循环，3.5 小时内发出 809 次请求，产生 $438 的账单 [GitHub Issues]。根因分析指出了三个问题：无断路器（循环不会被自动终止）、静默模型路由（用户不知道实际调用了什么模型）、成本显示错误（界面没有实时反映真实花费）[glukhov.org]。

社区反馈也很直接："Gemini constantly spins into loops for me" [Reddit r/opencodeCLI]。这不是个例——多 Agent 编排天然存在循环风险，当编排器的任务分解逻辑和模型的实际行为产生不一致时，就可能出现"Agent A 等 Agent B 的输出，Agent B 又在等 Agent A"的死循环。

Pi + Oh-My-Pi 不存在这个问题。它是单 Agent 顺序执行，没有编排层，没有多 Agent 之间的依赖关系，也就不存在编排导致的循环风险 [plugin-ecosystem-research]。最坏的情况是模型本身陷入了推理循环，但这个行为在所有 Agent 中都一样，不是增强引入的额外风险。

### 成本控制策略

不管用哪个生态，有几个策略能降低 token 失控的风险：

- **设置 provider 并发硬限制**：在 API 层面限制同时发出的请求数量，防止并行编排失控
- **选择轻量增强**：OMO-Slim（6 个 Agent）比完整 OMO（11 个 Agent）开销小得多 [GitHub]
- **按需加载**：Pi 用户可以只装 context-mode 而不装 Oh-My-Pi，启动开销从 22K 降到 5K
- **监控用量**：定期检查 API 账单，设置消费上限告警
- **避免在简单任务上用重型增强**：修一个 typo 不需要启动 ultrawork

---

## 维度6：本地模型与离线能力

这个维度的结论很干脆：如果你的备用方案场景包含"完全离线"或"本地模型为主"，Pi + Oh-My-Pi 有明显优势。

### Pi + Oh-My-Pi 的本地模型优势

Pi 原生支持 MLX 和 GGUF 格式，在 Mac 上本地模型性能有 2-3 倍的提升 [plugin-ecosystem-research]。更关键的是，Pi 原生系统提示只有 2K tokens——这意味着在本地模型有限的上下文窗口（通常 8K-32K）中，留给实际代码和对话的空间远大于 OpenCode 的 10K 基础。

Oh-My-Pi 的 22K 系统提示对本地模型来说确实是个负担——很多本地模型的上下文窗口放不下 22K 的工具描述。但 Pi 的灵活性在这里体现出来了：你可以不装 Oh-My-Pi，只选择性加载需要的工具。社区里有人用小模型跑 Pi 原生 + 几个精选工具，体验相当不错。Reddit 上有用户报告 Qwen 3.6 27B 在 5090 显卡上通过 Pi 运行良好 [Reddit r/PiCodingAgent]。另有 little-coder 扩展包专门为小模型优化，精简了工具描述以降低 token 消耗 [plugin-ecosystem-research]。

### OpenCode + OMO 的本地模型局限

OpenCode 支持 Ollama、LM Studio 和 llama.cpp 等本地模型运行方式 [plugin-ecosystem-research]。基础功能在本地模型上可用，但 OMO 的多 Agent 编排就力不从心了——它需要同时运行多个模型实例（每个 Agent 一个），对本地 GPU 和内存的消耗是普通开发者的硬件难以承受的。

OMO 的 category routing 系统（visual-engineering、deep、quick、ultrabrain 四个类别）默认映射到不同的云端模型 [plugin-ecosystem-research]。比如 ultrabrain 类任务会路由到最强的云端模型，quick 类任务用更快更便宜的模型。这个设计在云端 API 场景下很优雅，但在纯离线场景下就失效了——你不可能在本地同时跑四种不同规格的模型。

OMO-Slim 相对更轻量，但仍是多 Agent 架构，本地资源需求依然高于单 Agent 的 Pi [plugin-ecosystem-research]。

如果你的"备用方案"不只是"另一个云端 Agent"，而是包含了"万一断网了也能继续写代码"的需求，Pi 几乎是唯一的选择。

---

## 维度7：社区规模与长期维护

选一个编码工具不只是选当下的体验，还要赌它两年后还在不在。

### 数字对比

| 指标 | Pi | OpenCode | Oh-My-Pi | OMO |
|------|-----|----------|-----------|-----|
| GitHub Stars | 60K+ | 140K+ | ~3K | ~50K |
| 维护者 | Mario Zechner + Nutrient | AnomalyCo | Can Bölük | Yeongyu Kim + Sisyphus Labs |
| 更新频率 | 高（226 releases） | 高 | 中 | 高（8,541 commits） |
| 社区渠道 | Discord、Reddit | Discord、Reddit | GitHub Issues | Discord（公开开发） |
| 包/插件总量 | 3,811 | 数十个 + 丰富 MCP | 1 | 2（OMO + OMO-Slim） |

### 长期风险评估

**Pi 的风险最低。** 背后有 Nutrient（原 PSPDFKit）公司支持，Mario Zechner 是 libGDX 作者——这位开发者在游戏引擎圈的名望意味着他不会轻易放弃项目 [plugin-ecosystem-research]。226 个 release 说明项目一直在积极迭代。3,811 个包的生态也不容易一夜消失。

**OpenCode 的社区规模最大。** 140K+ stars 带来的网络效应很强——用户多 → 贡献者多 → 插件多 → 更多用户。AnomalyCo 作为公司运营也比个人项目更稳定 [plugin-ecosystem-research]。

**Oh-My-Pi 有单人维护风险。** Can Bölüt 是安全研究员，在逆向工程圈很知名，但 Oh-My-Pi 本质上是个人项目 [plugin-ecosystem-research]。如果作者精力转移，这个 3K stars 的项目可能面临维护放缓。不过考虑到 Oh-My-Pi 是 Pi 的一个增强包而非独立产品，即使它停止更新，Pi 生态中的其他增强选项仍然可用。

**OMO 的情况比较特殊。** Yeongyu Kim 和 AI 助手 Jobdori 协同开发，采用 SUL-1.0 许可证（非标准开源），项目正在重构以支持多 harness [GitHub]。50K stars 和 8,541 commits 说明项目很活跃，但非标准许可证可能给企业用户带来法务顾虑 [GitHub]。

---

## 维度8：增强的选择自由度

这是一个经常被忽略的维度。很多人把选择简化成"Pi vs OpenCode"或"Oh-My-Pi vs OMO"，但实际上，两个生态都提供了从"裸"到"全装"的连续光谱。你完全可以精细调节增强级别。

### Pi 的增强自由度

Pi 的增强组合几乎是无限的：
- 只装 context-mode（上下文优化），启动开销仅 5K tokens
- 用 gentle-pi 替代 Oh-My-Pi，获得架构师模式而非全功能套装
- 装 pi-subagents + pi-lens，获得子 Agent 和实时 LSP 反馈，但跳过 Oh-My-Pi 的 32 个工具
- 完全不装增强，享受 2K tokens 的极简体验
- 让 Pi 帮忙写自定义扩展——这在社区里是标准做法 [Reddit r/PiCodingAgent]

3,811 个包的长尾市场意味着几乎任何需求都有对应的包，你可以像自助餐一样按需取用 [pi.dev/packages]。

### OpenCode 的增强自由度

OpenCode 同样提供了灵活的选择空间：
- 只装 Superpowers，获得 Skills 系统和规划模式，不加多 Agent 编排
- 装 OMO-Slim 代替完整 OMO，6 个 Agent 而非 11 个 [GitHub]
- 只装 Context7 + Composio，走实用工具集成方向
- 完全不装插件，10K tokens 的开箱体验已经足够日常使用

### 社区的智慧

Reddit 上关于"裸版 vs 增强版"的讨论中，几条建议特别有参考价值：

> "Try nothing → try everything → reset → cherry-pick what works." [Reddit r/opencodeCLI]

> "Only get extensions when you have a problem to solve." [Reddit r/PiCodingAgent]

> "Commands are where the bulk of value is — specific to your stack, business, release process." [Reddit r/opencodeCLI]

翻译过来就是：先裸着用，遇到问题再找对应的增强，最终留下真正有用的那几个。不要为了"万一需要"而全装——那只会让 token 账单膨胀，同时增加维护成本。

---

## 生态演进：OMO 正在变成什么

### Oh-My-OpenCode → Oh-My-OpenAgent

2026 年 1 月，Anthropic 封锁了 OpenCode 使用 Claude 订阅的行为（判定为 ToS 违规），这件事直接推动了 OMO 的定位转型 [Reddit r/opencodeCLI]。项目从 Oh-My-OpenCode 更名为 **Oh-My-OpenAgent**，从"OpenCode 的增强插件"变为"通用编排层" [GitHub]。

目前有两个版本：**Ultimate**（运行在 OpenCode 上）提供完整的多 Agent 编排能力；**Light**（运行在 lazycodex/Codex CLI 上）是轻量替代 [GitHub]。ROADMAP 中明确提到将支持 Pi harness——如果实现，这将是一个重要的融合点 [GitHub]。

### Conductor：第三条路

Conductor 代表了另一种思路：不在 harness 内部增强，而在 harness 外部编排 [dev.to]。它是一个独立应用（目前 macOS only，Windows/Linux coming），用 Claude Code 或 Codex 作为执行引擎，自己做多 Git worktree 的编排 [dev.to]。

Conductor 的关键特性包括 Linear/GitHub 深度集成和一键修复 CI。有开发者分享了从 OpenCode+OMO 迁移到 Conductor+Claude Code+Superpowers 的经历，认为"外挂式编排"比"内嵌式增强"更灵活 [dev.to/chand1012]。

### 这意味着什么

未来可能出现的组合是：Pi 基础 + OMO 编排层——把 Pi 的高效执行和 OMO 的多 Agent 编排结合起来 [GitHub ROADMAP]。目前这还只是路线图上的计划，但方向值得关注。如果实现，将同时获得 Pi 在代码质量和调试方面的优势，以及 OMO 在大规模重构中的并行能力。

---

## 结论：选择建议

### 分层判断

选择备用方案不是一个单一维度的决定。需要在三个层面上分别考虑：

**基础框架层面：** Pi 基础更极简（2K tokens），给增强留出了更大空间，本地模型优化更好；OpenCode 基础更完备（10K tokens，内置 LSP/子 Agent/MCP），开箱即用能力更强。如果你倾向于"从零搭建自己想要的工具链"，Pi 的起点更干净；如果你希望"装好就能用"，OpenCode 的起点更舒适。

**增强生态层面：** Pi 生态包更多（3,811 个），长尾效应明显，几乎任何需求都有对应方案；OpenCode 生态插件方向更多元，OMO 做多 Agent 编排、Superpowers 做 Skills 系统、Composio 做外部集成，各有专精。

**最终增强体验层面：** Pi + Oh-My-Pi 在代码质量、调试能力、本地模型支持上更强，但多文件编排是短板；OpenCode + OMO 在多文件编排、社区规模、Skill 生态丰富度上更强，但调试能力弱，且有 token 失控的风险。

### 按场景选择

| 你的情况 | 推荐 |
|----------|------|
| 工作中调试占比大 | Pi + Oh-My-Pi |
| 经常做大规模重构 | OpenCode + OMO |
| 主要用本地模型 | Pi + Oh-My-Pi |
| 已有大量 Claude Code skill | 两个都行，OpenCode 更省心（直接扫描 .claude/skills/） |
| 想要最低的切换摩擦 | OpenCode + OMO |
| 需要完全离线能力 | Pi + Oh-My-Pi |
| 想精细调节增强级别 | 两个都行，Pi 的选择更多（3,811 个包） |
| 预算敏感、怕 token 失控 | Pi + Oh-My-Pi |

### 如果只能选一个做备用？

实际的建议是：**两个都装，按任务切换。**

- 单文件修复、调试、本地模型任务 → Pi + Oh-My-Pi
- 多文件重构、研究密集型任务 → OpenCode + OMO

这比"选一个"更实际，因为两者的优势分布在不同维度上。就像你不会只用锤子或只用螺丝刀——工具箱里两样都备着，遇到什么问题拿什么工具。两个框架的安装不冲突，可以同时存在于同一台机器上，共享同一套 Claude Code skill。

### 长期关注

有几个方向值得持续关注：

- **OMO 的 Pi harness 支持进展**——如果实现，可能催生"Pi + OMO 编排"的最强组合
- **Oh-My-Pi 的维护持续性**——单人维护的项目存在风险，社区是否需要培养更多贡献者
- **SKILL.md 标准的进一步收敛**——跨平台 skill 的兼容性会越来越好还是会再次分裂
- **Conductor 类工具是否会改变格局**——"外挂式编排"是否比"内嵌式增强"更有前途

---

## 附录

### 信息来源

1. grigio.org — "OpenCode vs Pi: Which AI Coding Agent Should You Use?" (2026)
2. glukhov.org — "Oh My Opencode Review: Honest Results, Billing Risks" (2026)
3. knightli.com — "What is oh-my-pi?" (2026-05-23)
4. birkey.co — "Why I Love Agent Pi" (2026-04-19)
5. GitHub: code-yeongyu/oh-my-openagent (8,541 commits, dev branch)
6. GitHub: badlogic/pi-skills (official, cross-compatible)
7. GitHub: earendil-works/pi (60K+ stars, 226 releases)
8. agensi.io — "OpenCode Skills Guide" (2026)
9. composio.dev — "10 Best OpenCode Skills" (2026)
10. Reddit: r/OnlyAICoding — Oh-My-Pi 讨论
11. Reddit: r/opencodeCLI — OMO vs bare OpenCode 讨论、ToS 封锁事件
12. dev.to/chand1012 — 迁移路径：Claude Code → OpenCode+OMO → Conductor
13. pi.dev/packages — Pi 包目录（3,811 包，截至 2026 年 6 月）
14. blog.fsck.com — "Superpowers (and Skills) for OpenCode" (2025-11-24)
15. opencode.im — OpenCode 插件中心
16. dataleadsfuture.com — "How I Use OpenCode, OMO-Slim, and OpenSpec" (2026-04-15)
17. GitHub: obra/superpowers — Superpowers plugin
18. GitHub: alvinunreal/oh-my-opencode-slim — OMO-Slim
19. Reddit: r/PiCodingAgent — Pi 扩展生态讨论、本地模型使用经验

### 术语表

| 术语 | 定义 |
|------|------|
| **基础框架（Harness）** | Pi 或 OpenCode 本体，不含任何增强包或插件 |
| **增强（Enhancement）** | 安装在基础框架上的包/插件/扩展，如 Oh-My-Pi、OMO、Superpowers |
| **Hashline** | 内容哈希锚点编辑方式，每行代码附带一个内容哈希值，编辑时引用哈希而非行号，解决"编辑时文件已变"的问题 |
| **ultrawork** | OMO 的关键字，在 prompt 中包含此关键字可触发多 Agent 编排模式 |
| **DAP** | Debug Adapter Protocol，调试器标准协议，允许 Agent 以编程方式使用调试器 |
| **LSP** | Language Server Protocol，语言服务器协议，提供代码补全、跳转定义、引用查找等 IDE 级能力 |
| **SKILL.md** | Agent Skill 的事实标准格式文件，跨 Claude Code、OpenCode、Pi、Codex CLI 等平台通用 |
| **MCP** | Model Context Protocol，模型上下文协议，让 Agent 能连接外部工具和数据源 |
| **Sisyphus** | OMO 的编排 Agent，负责将大任务拆解分配给多个执行 Agent |
| **Prometheus** | OMO 的规划 Agent，负责制定任务执行计划 |
| **category routing** | OMO 的任务路由机制，根据任务类型（visual-engineering/deep/quick/ultrabrain）分配不同模型 |
| **Team Mode** | OMO v4.0 引入的多 Agent 协作模式，一个 lead agent 编排最多 8 个专业化成员 Agent |
