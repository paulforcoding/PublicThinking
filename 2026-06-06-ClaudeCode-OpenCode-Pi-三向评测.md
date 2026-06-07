---
title: "Claude Code vs OpenCode vs Pi Agent 三向深度评测"
description: "三个终端编程 Agent 的哲学、架构、基准、成本和选型框架——不是比谁更好，是比它们对 AI 编程的理解有什么不同。"
---

# Claude Code vs OpenCode vs Pi Agent 三向深度评测

> 数据来源：Jock.pl 六工具对比（2026.05）、Artificial Analysis Coding Agent Index、Terminal-Bench 2.0 排行榜、SWE-bench Verified、MorphLLM（2026.05）、AlterSquare 生产环境实测、Grigio.org 实测、Pragmatic Engineer 采访、Armin Ronacher 博客、Mario Zechner 博客及 AI Engineer 演讲、GitHub 仓库（2026-06-07 抓取）

---

## 核心论点：三个工具在回答三个不同的问题

这不是一个"谁更好"的评测。这三个工具的分歧不在功能表上——它们在回答完全不同的根本问题：

| 工具 | 回答的问题 |
|------|-----------|
| **Claude Code** | "如何让一个模型跑出最好的效果？" |
| **OpenCode** | "如何让任何模型都能工作？" |
| **Pi** | "如何让你造出你需要的那个 agent？" |

所有功能差异、速度差异、成本差异、哲学差异——都从这三个问题出发。

---

## 一、基础数据

| 维度 | Claude Code | OpenCode | Pi |
|------|-------------|----------|-----|
| **作者** | Anthropic | SST → AnomalyCo | Mario Zechner + Armin Ronacher（Earendil Inc.，2026.05 起） |
| **许可证** | 闭源 | MIT | MIT（核心），未来商业层 Fair Source |
| **GitHub Stars** | ~124K（闭源，npm 分发） | **~169K**（已超越 Claude Code） | **60.2K** |
| **语言** | TypeScript 单进程 | Go TUI + Bun/JS 服务端 | TypeScript monorepo（4 包） |
| **模型绑定** | 仅 Claude 系列 | 75+ 提供商 | ~20 提供商 |
| **本地模型** | ❌ | ✅ Ollama | ✅ 尤其 Mac MLX/GGUF |
| **安装方式** | `npm install -g @anthropic-ai/claude-code` | `curl \| bash` | `npm install -g @earendil-works/pi-coding-agent` |
| **最新版本** | v2.1.161（2026.06） | v1.15.13（2026.06） | v0.78.1（2026.06） |
| **定价** | Pro $20 / Max $100-200/月 | 工具免费 + 自带 Key | 工具免费 + 自带 Key |
| **月活开发者** | 未公开 | 2.5M+（Grokipedia） | pi-ai npm 周下载 1.2M+ |

**关键变化（2026.05）：** Pi 从 Mario Zechner 个人项目搬迁至 **Earendil Inc.**（Armin Ronacher/mitsuhiko 联合创办），仓库从 `badlogic/pi-mono` 移至 `earendil-works/pi`。Mario 在博客"I've sold out"中宣布这一转变，Pi 核心保持 MIT 开源，未来计划推出 Fair Source + 企业版商业层。OpenCode 的仓库也从 `sst/opencode` 移至 `anomalyco/opencode`。

---

## 二、架构：三种对"agent 应该长什么样"的回答

### Claude Code：一体化精密仪器

```
单进程 TypeScript
├── 核心 System Prompt（~14,300 token）
├── 子代理模块（Plan Mode / Explore / Task）
├── Agent View 舰队管理（多会话仪表板）
├── /goal 自主完成（validator 模型每步检查）
├── /workflows 动态多 Agent 编排（2026.06 新增）
├── 20+ 内置工具
├── MCP 惰性加载（134K→5K token）
├── Worktree 隔离沙箱
├── 29-hook 治理系统
└── Auto Mode（2026.06 扩展至 Bedrock/Vertex/Foundry）
```

**设计逻辑**：模型是引擎，harness 是为这个特定引擎精调的赛车底盘。换引擎就废。

### OpenCode：可替换组件的平台

```
客户端-服务端分离
├── Go TUI（前端） ←→ Bun/JS HTTP Server（后端）
├── 75+ 提供商统一接口
├── 文件式 Agent 配置（.opencode/agents/*.md）
├── LSP 诊断反馈循环（独有）
├── Scout 子代理（外部文档研究）
├── 后台子代理
├── 自动上下文压缩
├── Git 快照 /undo /redo
└── Desktop App v2（2026.06，macOS/Windows/Linux）
```

**设计逻辑**：模型是燃料，harness 是对任何燃料都能工作的通用发动机。优化目标不是极速，是兼容性。

### Pi：Agent 乐高积木

```
四层 monorepo（earendil-works/pi）
├── pi-ai             ← 统一多提供商 API（周下载 1.2M+）
├── pi-agent-core     ← Agent 运行时（核心循环）
├── pi-tui            ← 终端 UI 库
└── pi-coding-agent   ← 编程 CLI 皮肤

出厂只配 4 个工具：read / write / edit / bash
System Prompt：~200 token
会话模型：树形结构（可分支、可回溯）
运行模式：interactive / print / JSON-RPC / SDK
```

**设计逻辑**：模型是执行者，harness 是最小骨架。一切功能都由 agent 自己写 TypeScript 扩展来增加。不是下载插件——是让 agent 修改自己。

Armin Ronacher（Flask 作者，Earendil 联合创始人）在博客中这样解释：

> *"You don't go and download an extension or a skill … You ask the agent to extend itself. It celebrates the idea of code writing and running code."*

---

## 三、基准测试：实测数据对比

### Terminal-Bench 2.0（2026）

| 排名 | Harness | 模型 | 得分 |
|:--:|---------|------|------|
| #1 | vix | Claude Opus 4.7 | **90.2%** ± 2.1 |
| #6 | Codex CLI | GPT-5.5 | 82.2% ± 2.2 |
| #52 | Claude Code | Claude Opus 4.6 | **58.0%** ± 2.9 |
| — | OpenCode | 未发布 | — |
| — | Pi | 未发布 | — |

**⚠️ 重要说明：** Terminal-Bench 2.0 测试的是 harness + 模型组合，但排名高度依赖 harness 对该基准的适配程度。Claude Code 在 Terminal-Bench 上只排 #52（58%），但在 SWE-bench 上（使用 mini-SWE-agent harness）Claude Opus 4.8 以 88.6% 位居第一。Reddit 社区直接评价：**"Terminal-Bench is a useless, misleading benchmark that doesn't reflect Claude Code's actual capabilities."**

这意味着 Terminal-Bench 2.0 测试的是"agent 在 terminal 里的自主执行能力"，而不是"agent 在实际编码中的综合能力"。OpenCode 和 Pi 都没有发布该基准的成绩——不是做不到，是这些工具的开发者没有选择针对这个基准优化。

### SWE-bench Verified（同一 harness，不同模型）

| 模型 | 得分 |
|------|------|
| Claude Opus 4.8 | **88.6%** |
| GPT-5.5 | 82.6% |
| Claude Opus 4.7 | 82.0% |
| Gemini 3.5 Flash | 78.8% |
| GPT-5.3 Codex | 78.0% |

**SWE-bench 测的是模型能力，不是 harness 能力。** 所有模型都通过 mini-SWE-agent 这个统一 harness 运行。

### Harness 效应（同一个模型，不同 harness 的差距）

这是最被低估的维度：

| 模型 | Harness A | Harness B | 差距 |
|------|-----------|-----------|------|
| Claude Opus | Claude Code: 77% | Cursor: **93%** | 16pp 来自 harness |
| Claude Opus | 最小 scaffold: 42% | Claude Code: **78%** | 36pp 来自 harness |
| 综合范围 | — | — | **5–40pp** 取决于 harness 质量 |

**同一个模型，在不同 harness 里可以差 40 个百分点。Harness 不是"壳"，它是模型能力的放大器或衰减器。**

---

## 四、速度与质量：同一任务的三种表现

### Builder.io 四项任务基准（Claude Sonnet 4.5）

| 任务 | Claude Code | OpenCode | 差距 |
|------|-------------|----------|------|
| Bug 修复 | ~40s | ~40s | 持平 |
| 跨文件重命名 | 3m 06s | 3m 13s | 接近 |
| 测试编写 | 3m 12s（73 测试） | 9m 11s（94 测试） | OC 多 29% 但慢 2.9× |
| **总用时** | **9m 09s** | **16m 20s** | CC 快 78% |

Pi 在这个基准上没有直接数据，但从 Grigio.org 的实测来看：同模型下 Pi 比 OpenCode 快 2–3×（本地模型场景），因为上下文开销极小——Pi 的 System Prompt 不到 200 token，OpenCode 可能膨胀到 10K+。

### 生产环境实测（AlterSquare）

| 维度 | Claude Code | OpenCode |
|------|-------------|----------|
| 技术债务 | 架构不一致、冗余依赖、过度抽象 | 未授权重格式化代码 |
| 测试覆盖 | 只跑子集 | ✅ 跑全量，覆盖率 +29% |
| Bug 发现 | ❌ 漏掉并发/竞态 | ✅ 抓到无关类型错误 |
| 长期代价 | 4 个月后重复代码率 3.1%→14.2% | — |

**Pi 的技术债特征**（来自 Mario 自己的分析）：不设权限提示，全 YOLO 模式，出错了靠 agent 自身的验证循环和 Git 回退兜底。由于 4 工具极简，agent 不会过度工程化。

---

## 五、扩展性：三种对"如何增加能力"的回答

| | Claude Code | OpenCode | Pi |
|---|---|---|---|
| **扩展方式** | Skills 市场 / MCP / Plugin | Markdown 配置 / MCP | **让 agent 写 TypeScript 扩展** |
| **Marketplace** | ✅ Skills 市场 + Plugin 生态 | ❌（靠社区 MCP） | ✅ pi.dev/packages |
| **MCP** | ✅ 惰性加载（134K→5K） | ✅ 声明式 | ❌ 故意的——让 agent 自己写 |
| **自修改** | ❌ | ❌ | ✅ 核心卖点 |
| **子代理** | Plan/Explore/Task + Agent Teams + /workflows | Scout + 后台子代理 | ❌ 靠 tmux 代替 |
| **权限模型** | 默认只读，每次写弹确认 | 可配置，Git 快照 /undo | 全 YOLO |

Pi 的扩展哲学意味着：Pi 的能力上限是你（和你的 agent）写 TypeScript 扩展的能力。Claude Code 的能力上限是 Anthropic 工程师和社区 plugin 作者的能力。OpenCode 介于两者之间——配置文件驱动 + MCP 生态。

---

## 六、成本：同模型在不同 harness 里的真实花费

### 典型 10 步编码循环（Claude Sonnet 4.6，150K tokens）

| Harness | 每循环成本 | 原因 |
|---------|-----------|------|
| Claude Code | **$1.53** | 高缓存命中率（90%+） |
| OpenCode | ~$1.80–2.00 | 客户端-服务端多一跳 + LSP 循环 |
| Pi | **<$1.00** | 极简上下文 = 最少 token |

### 团队月度成本

| | Claude Code | OpenCode | Pi |
|---|---|---|---|
| **单人重度** | $130–260/月 | API 按量（同模型略贵） | API 按量（同模型最省） |
| **5 人团队** | $500–1,500/月 | ~$300–800/月 | ~$200–600/月 |
| **最低成本路径** | Pro $20/月（限速） | Ollama 免费本地模型 | Ollama 免费本地模型（最快） |

### 为什么 Pi 最省 token

Grigio.org 实测对比：同一个物理问题，Pi 一次解决，OpenCode 同模型解决不了。原因：OpenCode 的 System Prompt 膨胀到 10K+ token，Pi 的不到 200。**上下文污染直接影响模型推理质量——这是 Pi "minimal beats maximal" 论点的直接证据。**

---

## 七、哲学光谱：三者在"信任 vs 验证"轴上的位置

```
信任模型（少验证）                          验证模型（多检查）
    │                                              │
Claude Code ──────── OpenCode ────────────────── Pi
    │                    │                         │
 "模型输出值得信任"   "模型输出需要验证"    "模型自己验证自己"
 速度优先              正确性优先             自修改优先
 人做最终审查          LSP + 全量测试          agent 自扩展 + Git 回退
 只读默认              可配置权限              YOLO 全开
```

三种哲学没有对错。但它们的代价不同：

- Claude Code：人类 review 时间 +91%，速度优势被抵消
- OpenCode：慢 78%，但更少生产事故
- Pi：需要你会写 TypeScript 扩展，否则能力上限就是 4 工具

---

## 八、三者共同的盲点

Mario 在 Pragmatic Engineer 采访中指出：

> *"Your biggest enemy is still complexity. It's also your agent's biggest enemy. But it has no holistic view of your code base, so it keeps adding complexity."*

三个工具都在解决"如何让 agent 更好地执行任务"，但没有任何一个解决了"agent 不理解自己在整个系统中的位置"的问题。这是当前所有编程 agent 的天花板——不是模型质量不够，是上下文窗口和架构理解的物理限制。

Pi 的树形会话结构（可以分支出子任务再 merge 回来）是这个方向上最有意思的尝试，但它仍然依赖于人在树的高处做全局判断。

---

## 九、选型决策框架

### 不是"选最好的"，是"选匹配你工作方式的"

| 你的情况 | 推荐 |
|----------|------|
| 快速出活，复杂推理，人在回路的交互式编程 | **Claude Code** |
| 多模型混用，省钱，类型语言开发（LSP 反馈） | **OpenCode** |
| 你会写 TypeScript，想完全掌控，跑本地模型，YOLO 风格 | **Pi** |
| 入门 AI 编程，不想折腾 | Claude Code 或 Cursor |
| 开源信仰，绝不被供应商绑定 | OpenCode（最成熟）或 Pi（最自由） |
| 团队需要统一治理和审批流程 | Claude Code（29-hook 治理系统） |

### 实战组合（多数资深工程师的做法）

| 场景 | 工具 | 理由 |
|------|------|------|
| 复杂重构/设计 | Claude Code | 质量第一 |
| 日常编码 | Pi | 快、省、可控 |
| 模型实验/隐私 | OpenCode | 75+ 提供商 |
| 省钱跑 Opus | Pi + BYOK | 同样模型质量，$30–80/月 vs $100–200/月 |

---

## 十、最重要的一条结论

Jock.pl 的评测说出了最根本的事实：

> *"Two harnesses running the same model on the same task can produce dramatically different results. Harness tuning isn't window dressing — it can swing outcomes by 5–40 percentage points."*

**模型不是产品。Harness 才是。**

Claude Code、OpenCode、Pi —— 它们不是在卖同一个东西的不同包装。它们在卖三种对 AI 编程完全不同的理解：

- **Claude Code** 相信：把最好的模型嵌进最精密的 harness，就能得到最好的结果。
- **OpenCode** 相信：让任何模型都能接入同一个 harness，选择权比精调更重要。
- **Pi** 相信：harness 不应该替你决定任何事——它应该是最小的骨架，让 agent 自己长出肌肉。

你选的不只是一个工具。你选的是你认同的对 AI 编程的理解。
