---
title: "代码质量：大模型重要还是 Harness 重要？"
description: "用数据回答一个看似简单的问题——编辑工具格式改动→10 倍提升、同模型跨 harness 差 40pp、成本差 32 倍。结论不是比重，是数量级。"
---

# 代码质量：大模型重要还是 Harness 重要？

> 数据来源：*Agent Harness for LLM Agents: A Survey*（学术综述，2026.04）、Artificial Analysis Coding Agent Index（2026.05）、Vals AI SWE-bench 排行榜、Jock.pl 六工具对比（2026.05）、METR RCT 实验（2025）及其 2026 更新、Terminal-Bench 2.0 排行榜、SWE-Bench Pro（Scale AI）

---

## 这个问题本身需要先被拆解

"模型和 harness 各占多大比重"——这个问题假设答案是固定的百分比。证据不支持这个假设。

一个具体的例子说明为什么：

- 同一模型，只改编辑工具的格式（怎么把 diff 喂给模型）→ 得分从 **6.7% 跳到 68.3%**
- 同一个 harness，把 Claude Opus 4.1 升级到 4.8 → 得分从 **70.1% 升到 88.6%**

第一个变化是 **10 倍**（harness 改动）。第二个变化是 **+18.5 个百分点**（模型升级）。它们是不同数量级的事。

**所以这个问题没有固定答案。答案取决于你当前处于什么阶段。**

---

## 一、Harness 能单独产生多大差距

### 证据 1：编辑工具格式改动 → 10 倍提升，零模型变化

学术综述 [*Agent Harness for LLM Agents: A Survey*](https://www.preprints.org/manuscript/202604.0428/v1)（2026.04）记录了一个定量实验结果：

> *"Changing only the edit-tool format — with no model modification — yielded an order-of-magnitude improvement on coding benchmarks for certain models (e.g., from 6.7% to 68.3% for one model)."*

**同一个模型，什么都没改，只换了编辑工具的输入格式。61.6 个百分点的差距。零模型改动。**

这不是孤例。同一篇综述还记录了：

> *"Meta-Harness achieved 76.4% on TerminalBench-2, surpassing hand-engineered approaches at 74.7% — with zero model changes."*

> *"LangChain's DeepAgents improved from 52.8% to 66.5% on TerminalBench 2.0 (+26%) through harness-only changes."*

三条独立数据，三个不同实验，结论一致：**只改 harness，不改模型，可以产生从 +13% 到 +1000% 的提升。**

oh-my-pi（Pi Agent 的强化 fork）的实测数据更加直接：

| 模型 | 改动 | 效果 |
|------|------|------|
| Grok Code Fast 1 | 只改编辑格式 | **6.7% → 68.3%**（10×） |
| Gemini 3 Flash | hashline 替代 str_replace | **+5pp**（超过 Google 官方优化） |
| Grok 4 Fast | 工具调用优化 | **输出 token 减少 61%** |
| MiniMax | 整体 harness 优化 | **通过率 2.1×** |

### 证据 2：同一模型，不同 harness → 5-40pp 差距

来自 Jock.pl 和 Artificial Analysis 的实测数据：

| 同模型 | Harness A | Harness B | 纯 harness 差距 |
|--------|-----------|-----------|:--:|
| Claude Opus | Claude Code: 77% | Cursor: 93% | **16pp** |
| Claude Opus | 最小 scaffold: 42% | Claude Code: 78% | **36pp** |
| 综合范围 | — | — | **5–40pp** |

**同一个模型，只是换了 harness，可以差出 40 个百分点。这不是模型问题——这是 harness 问题。**

Reddit 社区讨论直接给出了一个标题：**"Same model, different harness: 30-50 point performance swing"**。Hacker News 的帖子标题更直白：**"Improving 15 LLMs at Coding in One Afternoon. Only the Harness Changed"**。

### 证据 3：成本——Harness 的决定性远超模型

Artificial Analysis Coding Agent Index（2026.05）的核心发现：

> *"The same model, wrapped in two different harnesses, can generate a bill that differs by 32× — the cost of a single task ranges from $0.07 to $2.26 with nearly identical code quality."*

**同一个模型，harness 不同，成本差 32 倍，代码质量几乎一样。** 你的月度账单不是模型决定的——是 harness 的 token 效率决定的。

Pi Agent 和 Claude Code 的实测对比说明了为什么：Pi 的 System Prompt 不到 1,000 token，Claude Code 约 14,300 token。同样是 10 步编码循环（150K tokens），Claude Code 花费约 $1.53，Pi 不到 $1.00。差距不在于模型贵不贵，在于 harness 往 prompt 里塞了多少你看不到的东西。

---

## 二、模型能单独产生多大差距

### 证据 4：同一 harness，不同模型 → 10-20pp 差距

SWE-bench Verified 排行榜（Vals AI，2026.05），所有模型使用 mini-SWE-agent 同一 harness：

| 排名 | 模型 | 得分 |
|:--:|------|------|
| 1 | Claude Opus 4.8 | **88.6%** |
| 2 | GPT-5.5 | 82.6% |
| 3 | Claude Opus 4.7 | 82.0% |
| 4 | Gemini 3.5 Flash | 78.8% |
| 5 | Gemini 3.1 Pro Preview | 78.8% |
| 6 | GPT-5.4 (xhigh) | 78.2% |
| 7 | Claude Opus 4.6 (Thinking) | 78.2% |
| 8 | GPT-5.3 Codex | 78.0% |
| 9 | Claude Sonnet 4.6 | 77.4% |
| 10 | DeepSeek V4 | 77.4% |

**同一 harness 下，第 1 名与第 10 名差 11.2pp。这个差距是纯模型质量产生的。**

这是一个真实但有限的效应。11 个百分点不是小数字，但和 harness 产生的 61.6 个百分点（证据 1）相比，不在一个数量级。

### 证据 5：在更难的基准上，模型差距更小

Scale AI 推出的 SWE-Bench Pro（比 Verified 更难）给出了一个反直觉的数据：

| 模型 | SWE-bench Verified | SWE-bench Pro |
|------|:--:|:--:|
| GPT-5 | ~78% | **23.3%** |
| Claude Opus 4.1 | ~82% | **23.1%** |

在 Verified 上差 4 个百分点的两个模型，在 Pro 上只差 0.2 个百分点。**当任务足够难时，模型之间的差距被压缩到接近于零——瓶颈不在模型，在 harness 提供的上下文和工具。**

### 证据 6：SWE-bench Mobile 中的直接对比

| 同一维度 | 对照组 | 差距 |
|----------|--------|:--:|
| 同 harness（OpenCode）| Sonnet 4.5: 4.0% vs Opus 4.5: 2.0% | 模型贡献 **2pp** |
| 同模型（Sonnet 4.5） | OpenCode: 4.0% vs Cursor: 12.0% | Harness 贡献 **8pp** |

**在这个 case 里，harness 效应是模型效应的 4 倍。**

---

## 三、为什么不能给出一个固定的百分比

### 问题 1：模型和 harness 不是加法关系——是乘法

学术综述中 AgencyBench 的发现：

> *"Proprietary models demonstrate superior performance within their native execution ecosystems — a result interpreted as evidence for co-optimizing model architecture with agentic frameworks."*

**模型和 harness 是一对一绑定的。** Claude 在 Claude Code 里表现最佳，不是因为模型本身无敌，是因为 Anthropic 把 Claude 和 Claude Code 一起优化。把 Claude 放到 OpenCode 里，或者把 GPT-5 放到 Claude Code 里——两边都废。

Terminal-Bench 2.0 的数据给出了一个刺眼的例证：

| Harness | 模型 | 得分 | 排名 |
|---------|------|------|:--:|
| vix | Claude Opus 4.7 | **90.2%** | #1 |
| Claude Code | Claude Opus 4.6 | **58.0%** | #52 |
| Codex CLI | GPT-5.5 | 82.2% | #6 |

Claude Opus 4.7 在 vix harness 里拿到第一，Claude Opus 4.6 在 Claude Code 自己的 harness 里只排第 52。Reddit 社区对此的评价：**"Terminal-Bench is a useless, misleading benchmark that doesn't reflect Claude Code's actual capabilities."** ——这个争议本身就说明了问题：不同的 benchmark 测试不同的 harness 能力，没有一个 benchmark 能公平地比较所有 harness。

### 问题 2：存在天花板效应

| 阶段 | 提升杠杆 | 典型幅度 |
|------|----------|:--:|
| 无 harness（裸调 API） | **构建 harness** | 10×（6.7%→68.3%） |
| 有基础 harness | **优化 harness** | +13–26% |
| 有成熟 harness | **升级模型** | +10–20pp |
| 接近天花板 | **两者都改不动了** | 瓶颈在问题本身 |

最大的回报在第一步：从无到有建 harness。越往后，harness 优化的边际回报递减，模型升级的相对重要性上升。

### 问题 3：METR 实验的 2026 更新揭示了一个更深的真相

METR 在 2025 年的 RCT 实验得出了一个反直觉的结论：

> *"Experienced developers using AI were 19% slower on average, while believing they were faster."*

**用了 AI 反而慢了 19%，但自己觉得更快。**

2026 年 2 月，METR 发布了更新（*"We are Changing our Developer Productivity Experiment Design"*），承认情况更复杂：

- 原始实验的开发者（n=10）：**有 AI 快 18%**，但置信区间横跨 -38% 到 +9%
- 新招募的开发者（n=47）：**有 AI 快 4%**，但置信区间横跨 -15% 到 +9%

METR 的关键发现：**开发者拒绝在没有 AI 的情况下完成实验。** 他们在有 AI 的条件下挑简单任务，在无 AI 的条件下挑难任务。METR 的结论是：AI 大概率确实加速了开发者，但**当前的实验设计已经无法可靠地测量这个效果**。

HandsOnArchitects 的结论：

> *"The teams winning with AI are not the ones with the best models. They are the ones who have stopped treating AI output as a deliverable and started treating it as an unverified input to a system that catches errors."*

**赢家用 AI 的方式不是换更好的模型，是构建验证系统。**

---

## 四、一个阶段性框架

不能用 50/50 或 70/30 这种数字，但可以给出一个基于证据的判断框架：

| 阶段 | 模型重要性 | Harness 重要性 | 核心证据 |
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
- Harness 工程：10×

学术综述用一句话总结了整个领域在这个问题上的共识：

> *"Investment in harness-level engineering can yield performance improvements that model changes alone do not replicate."*

> *"The resulting infrastructure — not any model advance — identified as the primary bottleneck."*

Medium 上一篇 2026 年 4 月的分析给了一个更直白的标题：**"The Agent Harness: Why 70% of Your AI Agent's Performance Lives Outside the Model"**。

**不是模型不重要。是模型能给你的上限已经写死了，而你的 harness 还在拖后腿。先修 harness，再追模型——这是数据说的，不是我说的。**
