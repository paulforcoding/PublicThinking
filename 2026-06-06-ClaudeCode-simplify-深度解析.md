---
title: "Claude Code /simplify 深度解析：三个 Agent 并行审查的代码净化术"
description: "拆解 /simplify 的并行 Agent 架构、code-simplifier subagent 的完整 Prompt、以及这两个组件如何配合工作。"
---

# Claude Code `/simplify` 深度解析：三个 Agent 并行审查的代码净化术

AI 写的代码有个通病：第一次跑通时，代码能工作，但读起来像翻译腔。多余的嵌套，重复的逻辑，明明项目里已经有工具函数却自己又写了一遍。

Claude Code 的 `/simplify` 命令解决的就是这件事。但它不是一个简单的 lint 工具——它是三个专门做代码审查的 AI agent 并行运行，各自从不同角度审视你刚改的代码，最后汇总修复。

这个命令背后涉及两个组件，原文社区讨论中经常混淆，这里先理清：

1. **`/simplify` built-in skill**：编译进 Claude Code 二进制的内置技能（v2.1.63 起），负责编排三路并行审查和自动修复流程。源码不可公开查看，但 prompt 已被社区逆向工程记录
2. **`code-simplifier` plugin**：Anthropic 官方开源的 subagent 定义，位于 `anthropics/claude-code` 仓库的 `plugins/` 目录下。它是一个单独的 Opus 模型 agent，定义了代码简化的行为规范

两者配合工作：`/simplify` skill 是调度器，`code-simplifier` 是执行者之一。

**重要更新**：`/simplify` 在 Claude Code v2.1.146 中被移除，替换为 `/code-review`（行为不同——不再自动修复）。社区已有 feature request（issue #61581）要求恢复。

## 一个真实的例子

Anthropic 官方文档里给的一个例子：

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

这段代码能跑，因为 JavaScript 闭包捕获了 `agentSessionManager`。但它需要一个 8 行的注释来解释为什么这样写。

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

---

## `/simplify` 的三阶段流程

### Phase 1: 捕获变更

通过 `git diff`（或 `git diff HEAD` 获取暂存变更）确定审查范围。如果没有 git 变更，回退到当前会话中最近修改的文件。

审查边界要精确——不审整个代码库，只审最近改的。

### Phase 2: 三个并行审查 Agent

每个 agent 获得完整的 diff 内容，然后各自深入代码库搜索特定类别的问题：

**Agent 1 — 代码复用审查**

只问一个问题：这段代码是不是在重复发明轮子？

- 项目中是否已有工具函数能做同样的事？（字符串处理、手动路径拼接、自定义环境变量检查、临时类型守卫）
- diff 中的逻辑是否应该用已有工具函数替代？

**Agent 2 — 代码质量审查**

关注结构和可维护性：

- 冗余的状态变量（重复存储已有状态、派生缓存值）
- 参数列表是否过长
- 近重复的代码块（copy-paste 变体）
- 泄露的抽象（把内部实现暴露出去）
- 裸字符串代替常量 / 枚举（"stringly-typed"）

**Agent 3 — 效率审查**

关注性能和资源使用：

- 重复计算、N+1 查询模式
- 独立操作是否可以并行化
- 启动路径上是否有不必要的阻塞
- 无界数据结构（内存泄漏风险）
- 事件监听器是否正确清理
- 读取整个文件但只需要部分内容

### Phase 3: 汇总与修复

三个 agent 完成后，主会话汇总结果并过滤误报。然后直接应用修复——不是只给一个报告让你自己改。

### 为什么是三个而不是一个？

用一个 agent 做全面审查的问题是上下文稀释。代码复用、代码质量、效率问题各需要深入搜索代码库的不同部分——一个 agent 的注意力有限，会顾此失彼。三个 agent 各自获得完整的 diff，各自对准一个维度深入搜索。这在 Claude Code 的 subagent 架构中天然支持：subagent 有自己独立的上下文窗口。

多个 GitHub issue 确认了三路并行架构（#47191、#39830），其中 #39830 直接描述了"一次 /simplify 运行启动了 3 个并行审查 agent"。

---

## code-simplifier agent 的完整 Prompt

`/simplify` 的三个并行 agent 是编译进 Claude Code 二进制的内置 skill，源码不可公开查看。但 Anthropic 同时开源了一个独立的 `code-simplifier` subagent，位于：

| 指标 | 数据 |
|------|------|
| 仓库 | anthropics/claude-code |
| ⭐ Stars | 130,000+ |
| 🔀 Forks | 21,100+ |
| 📄 License | Apache-2.0 |
| 路径 | plugins/code-simplifier/agents/code-simplifier.md |

这个 agent 的完整 Prompt：

```yaml
---
name: code-simplifier
description: Simplifies and refines code for clarity, consistency, and 
  maintainability while preserving all functionality. Focuses on recently 
  modified code unless instructed otherwise.
model: opus
---
```

注意 `model: opus`。code-simplifier 使用 Opus 而非 Sonnet，因为代码简化需要深度理解——你要在不改变行为的前提下改善表达方式，这比广度搜索（适合 Sonnet）更吃推理能力。

### 五条核心规则

1. **Preserve Functionality** — 永远不改代码做什么，只改怎么做
2. **Apply Project Standards** — 从 CLAUDE.md 读取项目约定（ES modules、函数声明风格、返回类型注解、命名规范）
3. **Enhance Clarity** — 减嵌套、去冗余、改善命名、合并相关逻辑、删除不必要的注释。明确禁止嵌套三元运算符，优先 switch 或 if/else。**清晰优于简短**
4. **Maintain Balance** — 避免过度简化。不创造难以理解的巧妙写法。不因为追求"更少行数"而牺牲可读性
5. **Focus Scope** — 只看最近修改的代码，不看整个代码库

### 执行流程

```
1. 识别最近修改的代码段
2. 分析改进机会
3. 应用项目最佳实践
4. 确保功能不变
5. 验证简化后的代码更清晰、更可维护
6. 只记录对理解有重要影响的变更
```

最后一条值得注意：不记录琐碎的改动。如果改动本身是显而易见的，就不需要加注释解释。

### 关键设计决策

Prompt 中几处精妙的设计：

**"避免 try/catch 如果可以"** — 过度使用 try/catch 会掩盖错误、让控制流变复杂。code-simplifier 会评估是否真的需要异常处理。

**"删除描述显而易见代码的注释"** — AI 生成代码的常见问题：代码旁边附一句注释用英文把代码翻译了一遍。code-simplifier 会删掉这类注释。

**"选择清晰而非简短"** — 这不是一个 minifier。如果展开一段逻辑能让代码更容易理解，它会展开。

**"You operate autonomously and proactively"** — agent 被设计为主动运行，不需要显式请求。配合 Claude Code 的 hooks 可以在写完文件后自动触发。

---

## `/simplify` skill 和 code-simplifier plugin 的关系

社区讨论中经常把这两个东西混为一谈。它们的区别：

| | `/simplify` built-in skill | `code-simplifier` plugin |
|---|---|---|
| **位置** | 编译进 Claude Code 二进制 | `anthropics/claude-code/plugins/` |
| **架构** | 三个并行 agent | 单个 agent |
| **模型** | 内置（不可配置） | Opus（YAML frontmatter 指定） |
| **修复方式** | 自动应用修复 | 需要手动调用 |
| **源码可见性** | 不可见（社区逆向记录了 prompt） | 完全开源 |
| **当前状态** | v2.1.146 移除，替换为 `/code-review` | 仍然存在 |

`/simplify` 被移除后，替代的 `/code-review` 行为不同——不再自动修复，审查 agent 数量和维度也有变化。社区 issue #61581 要求恢复 `/simplify`，截至本文撰写时仍处于 open 状态。

---

## 效果数据

关于 `/simplify` 效果的量化数据，需要诚实区分正式基准和个人观察：

**257 个测试零失败** — 官方 AgentSessionManager 重构案例。所有已有测试全部通过，功能零回归。这是 Anthropic 自己提供的数据点。

**Token 消耗降低 20-30%** — 来源是日本开发者 @tetumemo 在 X/Twitter 上的个人测量（"トークン消費が20-30パーセント減る"）。逻辑成立：简化后的代码更短、冗余更少，在后续 AI 编码会话中消耗更少的上下文 token。但这是非正式观察，没有发布方法论，不是正式基准。

**三个 agent 并行** — 整体完成时间接近单个 agent 处理一个维度的时间，而非三倍。这是并行架构的天然优势。

Anthropic 团队自己把这个 agent 用在 Claude Code 自身的代码库上——写代码的团队信任它来清理自己的代码，这是有说服力的验证。

---

## 怎么用

### code-simplifier plugin 安装

```bash
# 终端直接安装
claude plugin install code-simplifier

# 或在 Claude Code 会话内
/plugin marketplace update claude-plugins-official
/plugin install code-simplifier
```

### 触发方式

`/simplify` 已被移除。当前可用的替代方案：

- 直接调用 code-simplifier agent：在 Claude Code 会话中让 Claude 以 code-simplifier agent 的身份审查代码
- 使用 `/code-review`（行为不同，不自动修复）
- 手动 prompt："以 code-simplifier 的规则审查我最近的修改"

### 配合 CLAUDE.md

code-simplifier 会读取项目的 CLAUDE.md 来理解编码规范。文件中写得越具体，简化结果越好：

```markdown
# CLAUDE.md

## 代码风格
- 用 ES modules，import 要排序
- 命名函数用 function 关键词，不要用箭头函数
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
            "command": "echo '考虑运行 code-simplifier 清理刚才的修改'"
          }
        ]
      }
    ]
  }
}
```

---

## 如何在自己的工具中复现

`/simplify` 的架构并不依赖 Claude Code 的专有功能。核心要素有三个：

### 1. 多 Agent 并行（而非一个全能 Agent）

三个专门的审查 agent 而不是一个。每个 agent 的系统提示描述一个审查维度：

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

把 diff 内容传给每个 agent，让它们在代码库中搜索相关上下文，但只对 diff 中的代码提出修改建议。

### 3. 汇总过滤 + 直接修复

不是生成报告让开发者自己改，而是：
- 收集三个 agent 的所有建议
- 去重和过滤明显不合理的建议
- 直接应用修改

---

## 局限和注意事项

- **误报** — agent 可能提出看起来合理但不适合特定上下文的建议。始终用 `git diff` 审查修改后再提交
- **成本** — 三个 agent 并行运行 = 3 倍的 API 调用。对大 diff 来说成本显著
- **不适合初次实现** — code-simplifier 改的是表达方式，不是设计。如果一开始的设计就有问题，应该先重构，再简化
- **不替代人类审查** — 机器能抓住模式问题，但理解不了业务意图。流程应该是：AI 简化 → 人类审查 → 合并
- **受限于 CLAUDE.md 质量** — 如果项目没有写编码规范（或写得很空泛），code-simplifier 只能靠自己判断。结果会不稳定

---

## 这件事的意义

`/simplify` 最值得关注的不是它做了什么，而是它怎么做的方式：

**专门化 agent 优于全能 agent。** 三个窄领域专家协同，比一个通才好。这是 agent 架构设计的一条普适原则——上下文窗口有限，注意力是稀缺资源，分工让每个 agent 把有限的注意力集中在一个维度上。

**修改代码而不只是建议。** 从"AI 帮你发现问题"到"AI 帮你解决问题"，是一个关键的体验跃迁。当然前提是行为不变（257 个测试全部通过）且有人类审查兜底。

**agent 的边界约束决定输出质量。** code-simplifier 的 Prompt 中 "preserving exact functionality" 和 "choose clarity over brevity" 这些明确的约束，比任何优化技巧都重要。约束越精确，agent 的输出越可控。

**开源即信任。** Anthropic 把自己团队日常用的工具开源出来，让所有人看到 Prompt 怎么写、规则怎么设、怎么验证。不是发论文，而是开源工作流。

---

*code-simplifier 源码：[github.com/anthropics/claude-code/tree/main/plugins/code-simplifier](https://github.com/anthropics/claude-code/tree/main/plugins/code-simplifier)*
