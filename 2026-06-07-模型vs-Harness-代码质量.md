---
title: "代码质量：大模型重要还是 Harness 重要？"
description: "用数据回答一个看似简单的问题——编辑工具格式改动→10倍提升、同模型跨harness差40pp、成本差32倍。结论不是比重，是数量级。"
---

# 代码质量：大模型重要还是 Harness 重要？

> 数据来源：*Agent Harness for LLM Agents: A Survey*（学术综述，2026.04）、Artificial Analysis Coding Agent Index（2026.05）、Vals AI SWE-bench 排行榜、Jock.pl 六工具对比、METR RCT 实验、Digital Applied Benchmark Methodology

---

## 这个问题本身需要先被拆解

"模型和 harness 各占多大比重"——这个问题假设答案是固定的百分比。证据不支持这个假设。

一个具体的例子说明为什么：

- 同一模型，只改编辑工具的格式（怎么把 diff 喂给模型）→ 得分从 **6.7% 跳到 68.3%**。
- 同一个 harness，把 Claude Opus 4.1 升级到 4.8 → 得分从 **70.1% 升到 88.6%**。

第一个变化是 **10 倍**（harness 改动）。第二个变化是 **+18.5 个百分点**（模型升级）。它们是不同数量级的事。

**所以这个问题没有固定答案。答案取决于你当前处于什么阶段。**

---

## 一、Harness 能单独产生多大差距

### 证据 1：编辑工具格式改动 → 10 倍提升，零模型变化

学术综述 [*Agent Harness for LLM Agents: A Survey*](https://www.preprints.org/manuscript/202604.0428/v1)（2026.04）记录了一个定量实验结果：

> *"Changing only the edit-tool format — with no model modification — yielded an order-of-magnitude improvement on coding benchmarks for certain models (*e.g., from 6.7% to 68.3%* for one model)."*

**同一个模型，什么都没改，只换了编辑工具的输入格式。61.6 个百分点的差距。零模型改动。**

这不是孤例。同一篇综述还记录了：

> *"Meta-Harness achieved 76.4% on TerminalBench-2, surpassing hand-engineered approaches at 74.7% — with zero model changes."*

> *"LangChain's DeepAgents improved from 52.8% to 66.5% on TerminalBench 2.0 (+26%) through harness-only changes."*

三条独立数据，三个不同实验，结论一致：**只改 harness，不改模型，可以产生从 +13% 到 +1000% 的提升。**

### 证据 2：同一模型，不同 harness → 5-40pp 差距

来自 Jock.pl 和 Artificial Analysis 的实测数据：

| 同模型 | Harness A | Harness B | 纯 harness 差距 |
|--------|-----------|-----------|:--:|
| Claude Opus | Claude Code: 77% | Cursor: 93% | **16pp** |
| Claude Opus | 最小 scaffold: 42% | Claude Code: 78% | **36pp** |
| 综合范围 | — | — | **5–40pp** |

**同一个模型，只是换了 harness，可以差出 40 个百分点。这不是模型问题——这是 harness 问题。**

### 证据 3：成本——Harness 的决定性远超模型

Artificial Analysis Coding Agent Index（2026.05）的核心发现：

> *"The same model, wrapped in two different harnesses, can generate a bill that differs by 32× — the cost of a single task ranges from $0.07 to $2.26 with nearly identical code quality."*

**同一个模型，harness 不同，成本差 32 倍，代码质量几乎一样。** 你的月度账单不是模型决定的——是 harness 的 token 效率决定的。

---

## 二、模型能单独产生多大差距

### 证据 4：同一 harness，不同模型 → 10-20pp 差距

SWE-bench Verified 排行榜（Vals AI，2026.05），所有模型使用 mini-SWE-agent 同一 harness：

| 模型 | 得分 |
|------|------|
| Claude Opus 4.8 | 88.6% |
| GPT-5.5 | 82.6% |
| Claude Opus 4.7 | 82.0% |
| GPT-5.3 Codex | 78.0% |
| Claude Sonnet 4.6 | 77.4% |
| DeepSeek V4 | 77.4% |

**同一 harness 下，第 1 名与第 6 名差 11.2pp。这个差距是纯模型质量产生的。**

这是一个真实但有限的效应。11 个百分点不是小数字，但和 harness 产生的 61.6 个百分点（证据 1）相比，不在一个数量级。

### 证据 5：SWE-bench Mobile 中的直接对比

SWE-bench Mobile 是少数同时比较工具+模型组合的榜单：

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

**模型和 harness 是一对一绑定的。** Claude 在 Claude Code 里表现最好，不是因为模型本身无敌，是因为 Anthropic 把 Claude 和 Claude Code 一起优化。把 Claude 放到 OpenCode 里，或者把 GPT-5 放到 Claude Code 里——两边都废。

有 CTO 实测：把 Claude Code 的 `ANTHROPIC_BASE_URL` 指向 OpenRouter，底层换成 GLM-5。输出质量断崖式下降。不是 GLM-5 差——是 Claude Code 的 System Prompt 和工具描述没有被 GLM-5 的训练数据覆盖过，GLM-5 也没有针对这个 harness 优化。

**所以模型和 harness 不能独立打分然后相加。它们的关系是锁和钥匙。**

### 问题 2：存在天花板效应

| 阶段 | 提升杠杆 | 典型幅度 |
|------|----------|:--:|
| 无 harness（裸调 API） | **构建 harness** | 10×（6.7%→68.3%） |
| 有基础 harness | **优化 harness** | +13–26%（证据 1 的 Meta-Harness 和 DeepAgents） |
| 有成熟 harness | **升级模型** | +10–20pp（证据 4） |
| 接近天花板 | **两者都改不动了** | 瓶颈在问题本身——SWE-EVO 长周期任务中 GPT-5.2 从 72.8% 跌到 22.9% |

最大的回报在第一步：从无到有建 harness。越往后，harness 优化的边际回报递减，模型升级的相对重要性上升。

### 问题 3：生产环境中大多数团队的 harness 远未成熟

METR 在 2025 年做了一个 RCT 实验：

> *"Experienced developers using AI were 19% slower on average, while believing they were faster."*

**用了 AI 反而慢了 19%，但自己觉得更快。** 这个反直觉结果的原因是：没有 harness discipline 时，AI 生成代码看起来对，review 起来耗时间，修 bug 更耗时间。速度是幻觉。

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

**不是模型不重要。是模型能给你的上限已经写死了，而你的 harness 还在拖后腿。先修 harness，再追模型——这是数据说的，不是我说的。**
