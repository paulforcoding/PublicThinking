---
title: "代码质量：大模型重要还是 harness 重要？"
description: "用六个独立数据源回答一个看似简单的问题。编辑工具格式改动→10 倍提升、同模型跨 harness 差 26pp、成本差 32 倍——但答案不是'比重'，是'你当前在哪个阶段'。"
---

# 代码质量：大模型重要还是 harness 重要？

> **数据来源：** *Agent Harness Engineering: A Survey*（OpenReview, 2026）、Artificial Analysis Coding Agent Index（2026.05）、Endor Labs Agent Security League（2026.05）、Vals AI SWE-bench Verified 排行榜（2026.06）、Terminal-Bench 2.0 排行榜（2026.05）、SWE-Bench Pro（Scale AI SEAL）、SWE-Bench Mobile（arXiv 2602.09540）、Can Bölük oh-my-pi harness 实验（2026.02）、METR 开发者生产力 RCT（2025 + 2026.02 更新）

---

## 这个问题本身需要先被拆解

"模型和 harness 各占多大比重"——这个问题假设答案是固定的百分比。证据不支持这个假设。

两个具体数字说明为什么：

- 同一个模型，只改编辑工具的格式（怎么把 diff 喂给模型）→ 得分从 **6.7% 跳到 68.3%**，提升 10 倍
- 同一个模型（GPT-5.5），从 Codex 换到 Cursor → 功能正确率从 **61.5% 升到 87.2%**，+25.7 个百分点

第一个变化来自 **harness 的编辑工具设计**，零模型改动。第二个变化来自 **harness 的系统提示词和工具链调优**，同样零模型改动。

但与此同时，在同一个 harness（mini-swe-agent）下，Claude Opus 4.8 以 88.6% 领跑 SWE-bench Verified，而第 10 名 DeepSeek V4 为 77.4%——这 **11.2 个百分点是纯模型质量产生的**。

**所以这个问题没有固定答案。答案取决于你当前处于什么阶段。**

---

## 一、harness 能单独产生多大差距

### 证据 1：编辑工具格式改动 → 10 倍提升，零模型变化

2026 年 2 月，安全研究员 Can Bölük 发布了一篇博客 [*I Improved 15 LLMs at Coding in One Afternoon. Only the Harness Changed*](https://blog.can.ac/2026/02/12/the-harness-problem)，记录了一组对照实验：

> *"Grok Code Fast 1 went from 6.7% to 68.3%, a tenfold improvement, because patch was failing so catastrophically that the model's actual coding ability was almost completely hidden behind mechanical edit failures."*
> *"Grok Code Fast 1 从 6.7% 涨到 68.3%，提升了 10 倍。原因不是模型变了，而是 patch 格式的编辑失败率高到灾难性，模型真正的编码能力完全被机械性的编辑失败掩盖了。"*
> — Can Bölük, [blog.can.ac](https://blog.can.ac/2026/02/12/the-harness-problem)

实验设计：从 React 代码库随机抽取文件 → 注入机械性 bug（运算符替换、布尔值翻转、off-by-one 等）→ 用自然语言描述 bug → 让 agent 修复。16 个模型 × 3 种编辑工具（patch、str_replace、hashline），每组 180 个任务 × 3 次运行。

核心发现：

| 模型 | 改动 | 效果 |
|------|------|------|
| Grok Code Fast 1 | patch → hashline | **6.7% → 68.3%**（10×） |
| Gemini 3 Flash | str_replace → hashline | **+5.0pp**（超过 Google 官方最佳优化） |
| Grok 4 Fast | 切换到 hashline | 输出 token **减少 61%**（重试循环大幅减少） |
| MiniMax | 整体 harness 优化 | 通过率 **超过 2×** |

> *"patch is the worst format for nearly every model, hashline matches or beats replace for most, and the weakest models gain the most."*
> *"对几乎所有模型来说，patch 是最差的格式；hashline 对多数模型持平或优于 str_replace；最弱的模型获益最大。"*
> — Can Bölük

这篇博客被 Hacker News 收录，标题是 [*Improving 15 LLMs at Coding in One Afternoon. Only the Harness Changed*](https://news.ycombinator.com/item?id=46988596)。HN 评论区一条高赞总结：

> *"There's a ton of extremely interesting low hanging fruit that can vastly improve the effectiveness of even currently existing models hiding in how we design our agent harnesses."*
> *"在我们设计 agent harness 的方式中，隐藏着大量极其有趣的低垂果实，能大幅提升即便是现有模型的有效性。"*
> — Hacker News 评论

这不是孤例。学术综述 [*Agent Harness Engineering: A Survey*](https://openreview.net/forum?id=3hXEPbG0dh)（OpenReview, 2026）在其 110+ 篇文献综述中，将编辑工具格式问题归纳为 harness 工程的核心挑战之一，并引用了 JetBrains 的 Diff-XYZ 基准和 EDIT-Bench 的独立验证——结论一致：**没有一种编辑格式在所有模型上占优**。

### 证据 2：同一模型，不同 harness → 5-40pp 差距

2026 年 4 月，开发者 Pawel Jozefiak 在 [thoughts.jock.pl](https://thoughts.jock.pl/p/ai-coding-harness-agents-2026) 发表了一篇六大 AI 编码 harness 的深度对比（Claude Code vs Codex CLI vs Aider vs OpenCode vs Pi vs Cursor），整理了多个来源的 harness 效应数据：

| 同一模型 | Harness A | Harness B | 纯 harness 差距 | 来源 |
|----------|-----------|-----------|:--:|------|
| Claude Opus | Claude Code: 77% | Cursor: 93% | **16pp** | Matt Mayer 测试 |
| Claude Opus | 最小 scaffold: 42% | Claude Code: 78% | **36pp** | CORE-Bench |
| 综合范围 | — | — | **5–40pp** | 多项研究 |

> *"Two harnesses running the same model on the same task can produce dramatically different results because of tuning."*
> *"两个 harness 在同一个任务上运行同一个模型，可以因为调优不同而产生截然不同的结果。"*
> — Pawel Jozefiak, thoughts.jock.pl

2026 年 5 月，Endor Labs 的 Agent Security League 提供了一个更戏剧性的数据点。据 [MindStudio 的报道](https://www.mindstudio.ai/blog/agent-harnesses-beat-model-upgrades-5-benchmarks)：

| 模型 | Harness | 功能正确率 |
|------|---------|:--:|
| GPT-5.5 | Codex（OpenAI 原生） | 61.5% |
| GPT-5.5 | Cursor | **87.2%** |

**同一个 GPT-5.5，同一周，只换 harness，+25.7 个百分点。** Endor Labs 原文的结论：

> *"Codex + GPT-5.5 ties for the third-highest security score we've measured at 20.1%, but lands at 61.5% on functional correctness, a slight regression behind its predecessor Codex + GPT-5.4 (62.6%) and roughly 26 percentage points below the same model running through Cursor."*
> *"Codex + GPT-5.5 以 20.1% 并列我们测量过的安全评分第三，但功能正确率仅 61.5%——略低于前代 Codex + GPT-5.4（62.6%），且比同一模型在 Cursor 中运行时低约 26 个百分点。"*
> — [Endor Labs](https://www.endorlabs.com/learn/gpt-5-5-sets-a-new-code-security-record-with-cursor-not-codex-in-agent-security-league)

Reddit r/AI_Agents 上一个讨论帖的标题概括了社区共识：[*Same model, different harness: 30-50 point performance swing*](https://www.reddit.com/r/AI_Agents/comments/1t86dil/same_model_different_harness_3050_point)。

### 证据 3：成本——harness 的决定性远超模型

Artificial Analysis 在 2026 年 5 月发布了 [Coding Agent Index](https://artificialanalysis.ai/agents/coding-agents)，这是一个包含 SWE-Bench-Pro-Hard-AA、Terminal-Bench v2、SWE-Atlas-QnA 三大基准的综合评测。其中一个关键发现被 Medium 上的[分析文章](https://medium.com/@wasowski.jarek/coding-agent-index-2026-benchmarking-full-agent-stacks-model-harness-4183305e4b90)引用：

> *"The same model, wrapped in two different harnesses, can generate a bill that differs by 32× — the cost of a single task ranges from $0.07 to $2.26 with nearly identical code quality."*
> *"同一个模型，套上两个不同的 harness，账单可以差 32 倍——单个任务的成本从 $0.07 到 $2.26，但代码质量几乎一样。"*
> — Medium, Coding Agent Index 分析

Artificial Analysis 官方的 [X（Twitter）公告](https://x.com/ArtificialAnlys/status/2053865095076438427) 给出了更多细节：

> *"Token usage varies >3x: GLM-5.1 in Claude Code uses the most tokens at 4.8M/task, followed by Kimi K2.6 at 3.7M/task and DeepSeek V4 Pro at 3.5M/task."*
> *"Token 用量差异超过 3 倍：GLM-5.1 在 Claude Code 中用量最多，每任务 480 万 token，其次是 Kimi K2.6 的 370 万和 DeepSeek V4 Pro 的 350 万。"*
> — Artificial Analysis

你的月度账单不是模型决定的——是 harness 的 token 效率决定的。Pi Agent 和 Claude Code 的实测对比说明了为什么：Pi 的 System Prompt 不到 1,000 token，Claude Code 约 14,300 token。同样是编码循环，Claude Code 花费更多不是因为模型更贵，而是 harness 往 prompt 里塞了更多你看不到的东西。

---

## 二、模型能单独产生多大差距

### 证据 4：同一 harness，不同模型 → 11pp 差距

SWE-bench Verified 排行榜（[Vals AI](https://vals.ai/benchmarks/swebench)，更新至 2026.06），所有模型使用 mini-swe-agent 同一 harness（仅一个 bash 工具，无额外脚手架）：

| 排名 | 模型 | 得分 | 每次测试成本 | 延迟 |
|:--:|------|:--:|:--:|:--:|
| 1 | Claude Opus 4.8 | **88.6%** | $1.92 | 567s |
| 2 | GPT-5.5 | 82.6% | $1.36 | 426s |
| 3 | Claude Opus 4.7 | 82.0% | $2.42 | 442s |
| 4 | Gemini 3.5 Flash | 78.8% | $0.95 | 254s |
| 5 | Gemini 3.1 Pro Preview | 78.8% | $0.78 | 312s |
| 6 | GPT-5.4 (xhigh) | 78.2% | $0.80 | 307s |
| 7 | Claude Opus 4.6 (Thinking) | 78.2% | $1.22 | 351s |
| 8 | GPT-5.3 Codex | 78.0% | $0.46 | 247s |
| 9 | Claude Sonnet 4.6 | 77.4% | $1.30 | 512s |
| 10 | DeepSeek V4 | 77.4% | $0.44 | 635s |

> *"This puts the evaluation burden squarely on the model rather than the harness."*
> *"这把评测的重担完全放在了模型身上，而非 harness。"*
> — Vals AI 方法论说明

**同一 harness 下，第 1 名与第 10 名差 11.2pp。这个差距是纯模型质量产生的。** 这是一个真实但有限的效应——和 harness 产生的 61.6pp（证据 1）或 25.7pp（证据 2）相比，不在一个数量级。

Vals AI 还提供了一个有趣的维度——按任务难度分层：

| 模型 | <15 分钟任务 | 15 分钟-1 小时 | 1-4 小时 | >4 小时 |
|------|:--:|:--:|:--:|:--:|
| Claude Opus 4.8 | 93% | 88% | 74% | 67% |
| GPT-5.5 | 92% | 81% | 50% | 67% |
| Gemini 3.5 Flash | 87% | 78% | 52% | 33% |
| DeepSeek V4 | 88% | 76% | 45% | 0% |

**简单任务上模型差距极小（93% vs 87%），真正的分水岭在中等复杂度任务（15 分钟到 1 小时）**——这正是 harness 上下文管理能力发挥作用的地方。

### 证据 5：在更难的基准上，模型差距不是缩小了——而是重新洗牌了

Scale AI 推出的 [SWE-Bench Pro](https://scale.com/leaderboard/swe_bench_pro_public) 比 Verified 更难（1,865 个长周期任务，跨 41 个仓库，含 Python/Go/TS/JS）。旧版本文章引用了 GPT-5 和 Claude Opus 4.1 在此基准上都只得 ~23% 的数据——这个数据已过时。2026 年最新排行（使用 mini-swe-agent harness）：

| 排名 | 模型 | SWE-Bench Pro 得分 |
|:--:|------|:--:|
| 1 | GPT-5.4 (xHigh) | **59.10%** |
| 2 | Muse Spark | 55.00% |
| 3 | Claude Opus 4.6 (Thinking) | 51.90% |
| 4 | Gemini 3.1 Pro (Thinking) | 46.10% |
| 5 | Claude Opus 4.5 | 45.89% |
| 10 | GPT-5 (High) | 41.78% |
| 15 | Gemini 3 Flash | 34.63% |
| 24 | Llama 3.1 405B | 11.18% |

**第 1 名与第 5 名差 13pp，与第 15 名差 24pp。** 在更难的基准上，模型差距不仅没有压缩到零，反而拉开了——但这恰好是因为更强的模型能更好地利用 harness 提供的有限上下文。

一个值得注意的信号：OpenAI 在 2026 年 2 月[宣布弃用 SWE-bench Verified](https://openai.com/index/why-we-no-longer-evaluate-swe-bench-verified/)，原因是所有前沿模型（GPT-5.2、Claude Opus 4.5、Gemini 3 Flash）在测试中都能复现金标准补丁或问题描述中的细节——**训练数据污染**。这解释了为什么 Verified 上的分数越来越接近天花板：模型可能不是在解题，而是在回忆。

### 证据 6：SWE-Bench Mobile 中的直接对比

[SWE-Bench Mobile](https://arxiv.org/html/2602.09540v1)（arXiv 2602.09540）是一个基于生产级 iOS 代码库的基准，评测了 22 种 agent-model 组合。论文的核心发现：

> *"Agent design matters as much as model capability — the same model shows up to 6× performance gap across agents."*
> *"agent 设计与模型能力同等重要——同一模型在不同 agent 间的性能差距最高可达 6 倍。"*
> — SWE-Bench Mobile 论文摘要

具体数据（同一 harness 内跨模型 vs 同一模型跨 harness）：

| 维度 | 对照组 | 差距 |
|------|--------|:--:|
| 同模型（Opus 4.5），不同 harness | Cursor: 8.0% → Claude Code: 8.0% → OpenCode: 2.0% | harness 差 **4×** |
| 同模型（Sonnet 4.5），不同 harness | Cursor: 12.0% → OpenCode: 4.0% | harness 差 **3×** |
| 同 harness（Cursor），不同模型 | Opus 4.5: 12.0% vs Sonnet 4.5: 12.0% | 模型差 **0pp** |

在这个 case 里，Cursor harness 内的 Opus 和 Sonnet 得分完全一样（12.0%），而同一模型在 Cursor vs OpenCode 之间差 3-4 倍。**harness 效应碾压了模型效应。**

---

## 三、为什么不能给出一个固定的百分比

### 问题 1：模型和 harness 不是加法关系——是乘法

Terminal-Bench 2.0 排行榜（[tbench.ai](https://www.tbench.ai/leaderboard/terminal-bench/2.0)，截至 2026.05，144 个提交）给出了一个刺眼的例证：

| Harness | 模型 | 得分 | 排名 |
|---------|------|:--:|:--:|
| vix | Claude Opus 4.7 | **90.2%** | #1 |
| Codex CLI | GPT-5.5 | 82.2% | #6 |
| Claude Code | Claude Opus 4.6 | 约 58% | #52 |

Claude Opus 4.7 在 vix harness 里拿到第一，但 Claude Opus 4.6 在 Anthropic 自家的 Claude Code 里只排第 52。**不是模型不行，是 harness 把模型的上限锁死了。**

Reddit r/ClaudeAI 上的讨论帖 [*How come Claude Code is ranked 19th on the Terminal-Bench leaderboard?*](https://www.reddit.com/r/ClaudeAI/comments/1q4dnhr/how_come_claude_code_is_ranked_19th_on_the) 引发了关于基准测试公平性的激烈辩论——这本身就说明了问题：不同的 benchmark 测试不同的 harness 能力，没有一个 benchmark 能公平地比较所有 harness。

Artificial Analysis 的 Coding Agent Index 在固定模型（Claude Opus 4.7）的情况下对比了 Cursor、Claude Code、OpenCode 三个 harness，发现综合得分差异显著。这意味着：你在选模型的同时也在选 harness，两者不可分割。

### 问题 2：存在天花板效应

| 阶段 | 提升杠杆 | 典型幅度 |
|------|----------|:--:|
| 无 harness（裸调 API） | **构建 harness** | 10×（6.7%→68.3%） |
| 有基础 harness | **优化 harness** | +13–26pp |
| 有成熟 harness | **升级模型** | +10–20pp |
| 接近天花板 | **两者都改不动了** | 瓶颈在问题本身 |

最大的回报在第一步：从无到有建 harness。越往后，harness 优化的边际回报递减，模型升级的相对重要性上升。

### 问题 3：METR 实验揭示了一个更深的真相

METR 在 2025 年的 RCT 实验得出了一个反直觉的结论（[论文](https://arxiv.org/pdf/2507.09089)）：

> *"We conduct a randomized controlled trial to understand how early-2025 AI tools affect the productivity of experienced open-source developers working on their own repositories. Surprisingly, we find that when developers use AI tools, they take 19% longer than without — AI makes them slower."*
> *"我们进行了一项随机对照试验，以了解 2025 年初的 AI 工具如何影响经验丰富的开源开发者在自己仓库上的生产力。令人惊讶的是，我们发现开发者使用 AI 工具时，完成任务的时间增加了 19%——AI 让他们变慢了。"*
> — METR, 2025

> *"After completing the study, developers estimate that allowing AI reduced completion time by 20%. Surprisingly, we find that allowing AI actually increases completion time by 19%."*
> *"完成研究后，开发者估计使用 AI 减少了 20% 的完成时间。但实际上，使用 AI 增加了 19% 的完成时间。"*
> — METR, 2025

**用了 AI 反而慢了 19%，但自己觉得快了 20%。**

2026 年 2 月，METR 发布了[更新](https://metr.org/blog/2026-02-24-uplift-update)，承认情况更复杂：

| 实验组 | 估计加速 | 95% 置信区间 |
|--------|:--:|:--:|
| 2025 原始研究 | **-19%**（变慢） | -2% 到 -39% |
| 原始开发者（后续追踪） | **+18%**（变快） | -38% 到 +9% |
| 新招募开发者 | **+4%**（变快） | -15% 到 +9% |

METR 的关键发现：**开发者拒绝在没有 AI 的情况下完成实验。** 30-50% 的参与者刻意避免提交他们不想在无 AI 条件下完成的任务。METR 的结论是：AI 大概率确实加速了开发者，但**当前的实验设计已经无法可靠地测量这个效果**。

> *"I will feel so painful if the task is decided as AI-disallowed."*
> *"如果任务被判定为不允许使用 AI，我会非常痛苦。"*
> — METR 2026 更新中引用的一位开发者

这个实验的真正启示不是"AI 让人变快还是变慢"，而是：**赢家不是用更好的模型，而是构建验证系统。** 当开发者把 AI 输出当作"未经验证的输入"而非"成品"时，harness 中测试、lint、类型检查等验证环节的质量——而不是模型本身——决定了最终产出。

---

## 四、一个阶段性框架

不能用 50/50 或 70/30 这种数字，但可以给出一个基于证据的判断框架：

| 阶段 | 模型重要性 | harness 重要性 | 核心证据 |
|------|:--:|:--:|------|
| **无 harness**（裸调 API，信任输出） | 低 | **极高** | 编辑工具格式：6.7%→68.3%（10×） |
| **有基础 harness**（有测试、CI、lint） | 中 | **高** | 同模型跨 harness 5-40pp；成本差 32× |
| **有成熟 harness**（LSP、全量测试、类型检查、上下文管理） | **高** | 中 | 同 harness 模型差 11-18pp；harness 收益递减 |
| **接近天花板**（co-optimized 原生生态） | **很高** | 低 | 模型升级是最后杠杆；但换 harness 可能倒退 |

**你大概率在第一或第二阶段。**

---

## 五、最重要的结论

证据不支持"比重"这种说法。证据支持一个更清晰的结论：

**在绝大多数团队的实际使用场景中，harness 是当前的最短板，且 harness 改进的收益与模型升级不在同一个数量级。**

- 模型升级：+10–20pp
- harness 工程：10×

学术综述 [*Agent Harness Engineering: A Survey*](https://openreview.net/forum?id=3hXEPbG0dh)（OpenReview, 2026）用一句话总结了这个领域的共识：

> *"The rapid deployment of LLM agents in production has revealed a recurring pattern: task execution reliability depends less on the underlying model than on the infrastructure layer that wraps it, the agent execution harness."*
> *"LLM agent 在生产中的快速部署揭示了一个反复出现的模式：任务执行的可靠性，与其说取决于底层模型，不如说取决于包裹它的基础设施层——agent 执行 harness。"*
> — Agent Harness Engineering: A Survey, OpenReview 2026

**不是模型不重要。是模型能给你的上限已经写死了，而你的 harness 还在拖后腿。先修 harness，再追模型——这是数据说的，不是我说的。**