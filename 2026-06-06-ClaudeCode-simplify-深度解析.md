---
title: "Claude Code /simplify 深度解析：三个 Agent 并行审查的代码净化术"
description: "逐层拆解 /simplify 的并行 Agent 架构、开源 subagent 的完整 Prompt、实际效果数据，以及如何在其他工具中复现这套工作流。"
---

# Claude Code `/simplify` 深度解析：三个 Agent 并行审查的代码净化术

> 不是又一个 "用 AI 清理代码" 的泛泛之谈。本文逐层拆解 `/simplify` 的并行 Agent 架构、开源 subagent 的完整 Prompt、实际效果数据，以及如何在其他工具中复现这套工作流。

---

AI 写的代码有个通病：第一次跑通时，代码能工作，但读起来像翻译腔。多余的嵌套，重复的逻辑，明明项目里已经有工具函数却自己又写了一遍。

Claude Code 的 `/simplify` 命令解决的就是这件事。它不是一个简单的 lint 工具或格式化器——它是三个专门做代码审查的 AI Agent 并行运行，各自从不同角度审视你刚改的代码，最后汇总修复。

这个命令的背后，是 Anthropic 团队自己内部的 code-simplifier agent——2026 年 2 月 Boris Cherny 宣布开源后，才被外界看到全貌。**它也是 Anthropic 团队日常用来清理自己代码库的工具。**

## 一个真实的例子

先看 Anthropic 官方文档里给的一个例子，简洁地说明问题：

```javascript
// BEFORE: AgentSessionManager 构造函数里把自己作为参数传给自己
const agentSessionManager = new AgentSessionManager(
  issueTracker,
  (childSessionId) => { /* ... */ },
  async (...) => { await this.handleResumeParentSession(..., agentSessionManager); },
  this.procedureAnalyzer,
  this.sharedApplicationServer,
);
```

这段代码能跑，因为 JavaScript 闭包捕获了 `agentSessionManager`。但它需要一个 8 行的注释来解释为什么这样写。读了还是会困惑。

```javascript
// AFTER: 构造和回调配置分离
const agentSessionManager = new AgentSessionManager(
  issueTracker,
  this.procedureAnalyzer,
  this.sharedApplicationServer,
);
agentSessionManager.setParentSessionCallbacks(
  (childSessionId) => { /* ... */ },
  async (...) => { await this.handleResumeParentSession(..., agentSessionManager); },
);
```

257 个测试全部通过。行为没变，但读起来不需要注释了。

## 不是"一个 Agent 审查代码"

这是 `/simplify` 最关键的架构设计。它不是丢给一个 AI 说 "帮我把代码整理干净"。

**它是三个 Agent 同时审查同一份 diff，每个 Agent 只聚焦一个维度。**

### Phase 1: 捕获变更

首先，`/simplify` 通过 `git diff`（或 `git diff HEAD` 获取暂存变更）确定审查范围。如果没有 git 变更，回退到当前会话中最近修改的文件。

### Phase 2: 三个并行审查 Agent

每个 Agent 获得完整的 diff 内容，然后各自深入代码库搜索特定类别的问题：

#### Agent 1 — 代码复用审查

这个 Agent 不关心代码写得漂不漂亮，它只问一个问题：**这段代码是不是在重复发明轮子？**

- 项目中是否已有工具函数能做同样的事？（字符串处理、手动路径拼接、自定义环境变量检查、临时类型守卫）
- Diff 中的逻辑是否应该用已有工具函数替代？

#### Agent 2 — 代码质量审查

关注结构和可维护性：

- 是否有冗余的状态变量（重复存储已有状态、派生缓存值）
- 参数列表是否过长
- 是否存在近重复的代码块（copy-paste 变体）
- 是否存在泄露的抽象（把内部实现暴露出去）
- 是否用裸字符串代替了常量 / 枚举（"stringly-typed"）

#### Agent 3 — 效率审查

关注性能和资源使用：

- 是否有重复计算、N+1 查询模式
- 独立的操作是否可以并行化
- 启动路径上是否有不必要的阻塞
- 是否存在无界数据结构（内存泄漏风险）
- 事件监听器是否正确清理
- 是否读取了整个文件但只需要部分内容

### Phase 3: 汇总与修复

三个 Agent 完成后，主会话汇总结果并过滤掉误报（false positive）。然后**直接应用修复**，不是只给一个报告让你自己改。

### 为什么是三个而不是一个？

用一个 Agent 做全面审查的问题是**上下文稀释**。代码复用、代码质量、效率问题各需要深入搜索代码库的不同部分——一个 Agent 的注意力有限，会顾此失彼。

三个 Agent 各自获得完整的 diff，各自对准一个维度深入搜索。这在 Claude Code 的 subagent 架构中天然支持：subagent 有自己独立的上下文窗口。

## 开源实现：code-simplifier agent 的完整 Prompt

`/simplify` 命令加载的是 Anthropic 官方开源的 code-simplifier subagent。它的完整 Prompt 在这里：

> 仓库：`anthropics/claude-plugins-official` ⭐29,479 🔀3,162 📅2026-06-06 📄Apache-2.0
>
> 源码：[plugins/code-simplifier/agents/code-simplifier.md](https://github.com/anthropics/claude-plugins-official/blob/main/plugins/code-simplifier/agents/code-simplifier.md)

这个 agent 被配置为始终使用 Claude Opus 模型（在 YAML frontmatter 中指定 `model: opus`），因为代码简化是一个需要深度理解的任务。

我们逐层拆解这个 Prompt 的设计：

### 第一层：角色定义

```
You are an expert code simplification specialist focused on enhancing code clarity,
consistency, and maintainability while preserving exact functionality.
```

关键句是 **"preserving exact functionality"**。这个 agent 不允许改变代码的行为，只能改变表达方式。这是 Prompt 工程中最重要的一条边界约束。

### 第二层：五条核心规则

1. **保护功能** — 永远不改代码做什么，只改怎么做
2. **应用项目标准** — 从 `CLAUDE.md` 读取项目约定（ES modules、函数声明风格、返回类型注解、命名规范）
3. **增强清晰度** — 减嵌套、去冗余、改善命名、合并相关逻辑、删除不必要的注释。**明确禁止嵌套三元运算符**，优先用 switch 或 if/else。**清晰优于简短**
4. **保持平衡** — 避免过度简化。不创造难以理解的巧妙写法。不因为追求"更少行数"而牺牲可读性
5. **聚焦范围** — 只看最近修改的代码，不看整个代码库

### 第三层：执行流程

```
1. 识别最近修改的代码段
2. 分析改进机会
3. 应用项目最佳实践
4. 确保功能不变
5. 验证简化后的代码更清晰、更可维护
6. 只记录对理解有重要影响的变更
```

最后一点值得注意：**不记录琐碎的改动**。如果改动本身是显而易见的，就不需要加注释解释。

### 关键设计决策

回看 Prompt 中的细节，有几处很精妙的设计：

**"避免 try/catch 如果可以"** — 项目标准中明确写了这条。过度使用 try/catch 会掩盖错误、让控制流变复杂。code-simplifier 会评估是否真的需要异常处理，还是可以用更简单的方式。

**"删除描述显而易见代码的注释"** — 这是 AI 生成代码的常见问题：代码旁边附着一句注释，用英文把代码翻译了一遍。code-simplifier 会删掉这类注释。

**"选择清晰而非简短"** — 这不是一个 minifier。如果展开一段逻辑能让代码更容易理解，它会展开。

## 效果数据

社区反馈了几个可量化的数据点：

- **Token 消耗降低 20-30%** — 日本开发者テツメモ在使用 `/simplify` 后的实际测量。简化后的代码更短、冗余更少，在后续的 AI 编码会话中消耗更少的上下文 token。
- **257 个测试零失败** — 官方 AgentSessionManager 重构案例中，所有已有测试全部通过。功能零回归。
- **分析数千行代码仅需数秒** — 三个 Agent 并行运行，整体完成时间接近单个 Agent 处理一个维度的时间。

Anthropic 团队自己把这个 agent 用在 Claude Code 自身的代码库上——写代码的团队信任用它来清理自己的代码，这是最有说服力的验证。

## 怎么用

### 安装

```bash
# 终端直接安装
claude plugin install code-simplifier

# 或在 Claude Code 会话内
/plugin marketplace update claude-plugins-official
/plugin install code-simplifier
```

安装后重启会话。

### 触发方式

在 Claude Code 会话中，输入：

```
/simplify
```

它会自动抓取当前会话中修改的文件，启动三路并行审查。

### 最佳使用时机

| 时机 | 为什么 |
|------|--------|
| **每次主要编码会话结束** | AI 写完代码后立即清理是最有效的 |
| **创建 PR 之前** | 在人类审查之前先跑一遍机器审查 |
| **复杂重构之后** | 重构时容易留下不一致的代码模式 |
| **清理 AI 生成的代码** | AI 倾向生成工作但不优雅的代码 |

### 配合 CLAUDE.md 效果翻倍

code-simplifier 会读取项目的 `CLAUDE.md` 来理解编码规范。文件中写得越具体，简化结果越好：

```markdown
# CLAUDE.md

## 代码风格
- 用 ES modules，import 要排序
- 命名函数用 `function` 关键词，不要用箭头函数
- 顶层函数必须标注返回类型
- React 组件必须有显式 Props 类型
- 优先 switch/if-else，禁止嵌套三元运算符
- 避免无意义的 try/catch
```

### 配合 hooks 实现全自动

可以在 Claude Code 的 hooks 配置中，让写完文件后自动触发简化：

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "echo '考虑运行 /simplify 清理刚才的修改'"
          }
        ]
      }
    ]
  }
}
```

更高阶的用法是配合 Stop hook，在每次 Agent 完成后自动跑一遍 `/simplify`。

## 如何在自己的工具中复现

`/simplify` 的架构并不依赖 Claude Code 的专有功能。核心要素有三个：

### 1. 多 Agent 并行（而非一个全能 Agent）

用三个专门的审查 Agent 而不是一个。每个 Agent 的系统提示描述一个审查维度。关键提示词模板：

**Agent 1 — 复用审查：**
```
你是代码复用审查专家。分析以下 diff，在代码库中搜索是否已有功能相似的
工具函数、helper、组件。你的唯一任务是找出重复造轮子的地方。不要提风格
或性能建议。
```

**Agent 2 — 质量审查：**
```
你是代码质量审查专家。分析以下 diff，找出：冗余状态、参数过多、近重复
代码块、泄露的抽象、裸字符串应替换为常量/枚举。只关注结构和可维护性。
```

**Agent 3 — 效率审查：**
```
你是效率审查专家。分析以下 diff，找出：重复计算、N+1 查询、可并行化的
串行操作、阻塞启动路径、未清理的监听器/定时器、过度读取。只关注性能。
```

### 2. Git diff 作为审查范围

审查边界要精确——不审整个代码库，只审最近改的：

```bash
git diff HEAD -- . ':!node_modules' ':!dist'
```

把 diff 内容传给每个 Agent，让它们在代码库中搜索相关上下文，但只对 diff 中的代码提出修改建议。

### 3. 汇总过滤 + 直接修复

不是生成报告让开发者自己改，而是：
- 收集三个 Agent 的所有建议
- 去重和过滤明显不合理的建议
- 直接应用修改

## 框架特定的变体

除了通用的 code-simplifier，社区已经出现了框架特定的版本：

- **Laravel/PHP** — Taylor Otwell（Laravel 作者）维护 `laravel-simplifier@laravel`，理解 Laravel 的 Facade、Service Container、Eloquent 模式
- **Rust** — MCP marketplace 上的 Rust 简化器，处理所有权和生命周期模式

这些变体证明了 `/simplify` 不是一个通用 LLM prompt 的简单包装——它是一套审查维度的框架，可以针对特定技术栈定制。

## 局限和注意事项

- **误报** — Agent 可能提出看起来合理但不适合特定上下文的建议。始终用 `git diff` 审查修改后再提交
- **成本** — 三个 Agent 并行运行 = 3 倍的 API 调用。对大 diff 来说成本显著
- **不适合初次实现** — code-simplifier 改的是表达方式，不是设计。如果一开始的设计就有问题，应该先重构，再简化
- **不替代人类审查** — 机器能抓住模式问题，但理解不了业务意图。流程应该是：AI 简化 → 人类审查 → 合并
- **受限于 CLAUDE.md 质量** — 如果项目没有写编码规范（或写得很空泛），code-simplifier 只能靠自己判断。结果会不稳定

## 为什么这件事值得关注

`/simplify` 最值得关注的不是它做了什么，而是**它怎么做的方式**：

1. **专门化 Agent 优于全能 Agent** — 三个窄领域专家协同，比一个通才好。这是 Agent 架构设计的一条普适原则
2. **修改代码而不只是建议** — 从"AI 帮你发现问题"到"AI 帮你解决问题"，是一个关键的体验跃迁
3. **Agent 的边界约束决定输出质量** — code-simplifier 的 Prompt 中"preserving exact functionality" 和 "choose clarity over brevity" 这些明确的约束，比任何优化技巧都重要
4. **开源即信任** — Anthropic 把自己团队日常用的工具开源出来，让所有人看到 Prompt 怎么写、规则怎么设、怎么验证。这是一种新的技术传播方式——不是发论文，而是开源工作流

如果你在用 Claude Code 但还没用 `/simplify`，现在就可以安装。如果你在用其他 AI 编码工具，这套并行审查架构完全可以照着实现——三个专门的 subagent 审查一份 diff，汇总修复。不需要什么专有技术。

---

*code-simplifier 源码：[github.com/anthropics/claude-plugins-official](https://github.com/anthropics/claude-plugins-official/tree/main/plugins/code-simplifier)*
