---
title: "AgentAra Engineering Skills 使用指南：把 Google 工程实践装进 AI Agent"
description: "六个 skill 文件、六套工程工作流，从 code review 到紧急变更，让 AI Agent 像 Google 工程师一样审查代码、拆分 PR、处理反馈。附每个 skill 的实战用法。"
---

# AgentAra Engineering Skills 使用指南：把 Google 工程实践装进 AI Agent

AI Agent 写代码已经不是新闻了。但 AI Agent 做 code review、拆 PR、写变更描述、处理 reviewer 反馈呢？这些"写代码之外"的工程实践，恰恰是决定代码质量的关键环节。

AgentAra 的 `skills/engineering` 目录包含六个 skill 文件，它们把 Google Engineering Practices 这个被全球顶级工程团队验证过的方法论，转化成了 AI Agent 可以直接执行的工作流。不是让 Agent 学会"写代码"，而是让 Agent 学会"像工程师一样工作"。

本文是**使用指南**——讲清楚这六个 skill 分别做什么、怎么用、什么时候用。想深入理解这些 skill 背后的 prompt 设计哲学和工作流编排逻辑，请看姊妹篇《从 Google 工程实践学 Agent Skill 设计：六个 Prompt 工程模式的深度拆解》。

## 源码在哪

`engineering` 是 agentara/skills 仓库中的一个 skill 套件，源码在仓库的 `skills/engineering/` 目录下。

| 指标 | 数据 |
|------|------|
| ⭐ Stars | 369 |
| 🔀 Forks | 16 |
| 📄 License | MIT |
| 📅 最近更新 | 2026-06 |
| 来源 | Google Engineering Practices (CC-BY 3.0) |

GitHub: https://github.com/agentara/skills/tree/main/skills/engineering

## 六个 Skill 全景

```
skills/engineering/
├── README.md
├── engineering-code-review/        → 代码审查
├── engineering-review-comments/    → 审查评论撰写
├── engineering-small-prs/          → PR 拆分策略
├── engineering-change-descriptions/ → 变更描述撰写
├── engineering-review-feedback/    → 处理审查反馈
└── engineering-emergency-changes/  → 紧急变更决策
```

六个 skill 覆盖了软件工程中"写完代码之后"的几乎所有环节。它们不是独立的 prompt 片段，而是一个有完整逻辑链的工作流系统：

```
写代码 → [small-prs] 拆分成小 PR
       → [change-descriptions] 写好描述
       → [code-review] Agent 做代码审查
       → [review-comments] Agent 写审查评论
       → [review-feedback] 作者处理反馈
       → [emergency-changes] 紧急情况走快速通道
```

每个 skill 有明确的触发条件（TRIGGER）和排除条件（DO NOT TRIGGER），让 Agent 在正确的时机自动加载对应的 skill。

---

## Skill 1: engineering-code-review — 代码审查

**一句话：** Agent 化身 Google 标准的 code reviewer，按 8 步工作流 + 16 项检查清单审查代码。

### 什么时候用

- 有人让你 review 一个 PR
- 你想让 Agent 帮你审查自己的代码
- 提交前想做一次自查
- 评估一段代码是否可以安全合并/部署

### 审查标准：不是完美主义

这是整个 skill 最值得注意的设计理念：

> Favor approval when the change definitely improves the code health of the touched system, even if it is not perfect.

标准不是"代码完美了才能通过"，而是"代码变好了就可以通过"。这避免了 AI 审查中最常见的问题——无休止地挑刺，连空格不对都要 BLOCKER。

同时有一条硬底线：

> Do not approve changes that make the system worse unless this is a true emergency and the follow-up cleanup is explicit.

让系统变差的代码，除非是真正的紧急情况且有明确的后续清理计划，否则不通过。

### 8 步审查工作流

1. **Establish scope** — 确认审查范围（哪些文件、哪些行为）
2. **Gather review material** — 收集 PR 描述、关联 issue、CI 状态
3. **Take the broad view first** — 先判断"这个改动应不应该存在"
4. **Review the main design before details** — 先审架构再审细节
5. **Review every human-written line** — 审查范围内每一行人写的代码
6. **Check against the review checklist** — 16 项检查清单
7. **Check verification evidence** — 测试和 CI 结果
8. **Make a verdict** — 四种裁定：APPROVE / APPROVE_WITH_COMMENTS / REQUEST_CHANGES / NEEDS_SPECIALIST

### 16 项检查清单

这不是一个"看看就好"的清单，而是 Agent 必须逐项检查的强制列表：

- Design / Functionality / Edge cases / Complexity
- Tests / Test quality / Naming / Comments
- Documentation / Style and consistency / Context
- Operations / Security / Privacy
- Accessibility / Internationalization

最后两项——无障碍访问和国际化——是很多团队会忽略的。但 Google 的标准里明确包含了它们，Agent 审查时也会检查。

### 输出格式

```markdown
## Findings
- [BLOCKER] path/file.ext:123 - 问题。为什么重要。要求的修改。
- [IMPORTANT] path/file.ext:45 - 问题。为什么重要。建议的修复。

## Open Questions
- ...

## Verdict
REQUEST_CHANGES | APPROVE_WITH_COMMENTS | APPROVE | NEEDS_SPECIALIST

## Notes
- Scope reviewed:
- Tests/commands checked:
- Follow-ups:
```

最高严重级别的问题排在最前面，每个问题都带文件路径和行号。干净利落。

---

## Skill 2: engineering-review-comments — 审查评论撰写

**一句话：** 教 Agent 写出清晰的、有严重级别的、对事不对人的审查评论。

### 什么时候用

- 你把审查发现转化正式的 PR 评论
- 你想让评论更清晰、更友善
- 你在处理 reviewer 的 pushback
- 你给审查发现加严重级别标签

### 评论结构

非琐碎的评论必须包含三个部分：

```text
SEVERITY: 观察到的问题。

Why this matters: 工程原因、用户影响、可维护性影响、或违反的项目规则。

Requested action: 具体的修复方案、问题、或需要作者做的决策。
```

六个严重级别：

| 级别 | 含义 | 是否阻塞合并 |
|------|------|-------------|
| BLOCKER | 必须修复 | 是 |
| IMPORTANT | 应该修复，除非有书面理由 | 通常是 |
| QUESTION | 需要信息才能判断 | 是 |
| NIT | 小修饰 | 否 |
| OPTIONAL | 建议，作者可拒绝 | 否 |
| FYI | 学习性内容 | 否 |

### 写作规则

几条核心规则：

- **对事不对人**：评论代码、设计、测试、行为，不评价作者
- **说 why 不说 like**："this makes X harder because..." 而不是 "I don't like X"
- **代码能表达的，不要在评论里解释**：如果代码本身不够清晰，应该改代码而不是加注释
- **指出好的选择**：看到好的工程决策时，说清楚为什么好

### 处理 Pushback

当作者不同意你的评论时：

1. 先重新评估他们的论点——他们可能更了解代码
2. 如果他们是对的，直说并 resolve 评论
3. 如果你仍然不同意，先复述他们的观点，再解释你的工程理由
4. 区分"必须改"和"我建议改"
5. 如果讨论卡住了，转到高带宽沟通（面对面/视频），然后在 review 里总结决策

> Do not keep arguing because you wrote the first comment. The goal is the best code, not defending the initial wording.

---

## Skill 3: engineering-small-prs — PR 拆分策略

**一句话：** 把大功能、大重构、大迁移拆成小的、可审查的、可安全合并的 PR 序列。

### 什么时候用

- 一个 PR 太大，reviewer 说"this diff is too large"
- 你在规划一个大的功能或重构
- 你需要做数据库/API 迁移
- 你想用 stacked PR 的方式推进工作

### 什么是"好的小 PR"

> "Small" means conceptually focused and self-contained, not merely a low line count.

小不是行数少，而是概念上聚焦、自包含。一个好的小 PR：

- 做一个自包含的改动
- 包含验证该改动所需的测试
- 合并后系统仍然正常工作
- 不会小到无法理解行为——一个新 API 通常至少需要一个使用示例

### 7 步拆分工作流

1. **Name the final outcome** — 用一句话说清楚最终目标
2. **List independent changes** — 列出所有独立的行为变更、重构、schema 变更、测试、文档
3. **Identify dependencies** — 哪些必须先合并
4. **Choose split strategies** — 从拆分策略库中选择
5. **Build merge sequence** — 每个 PR 保持 build 绿色
6. **Add context notes** — 让 reviewer 理解每个 PR 在整体中的位置
7. **Get reviewer consent** — 对于无法拆分的大变更，提前获取 reviewer 同意

### 8 种拆分策略

| 策略 | 适用场景 |
|------|----------|
| 先补测试 | 重构前先给现有行为加测试保护 |
| 先重构 | 移动/重命名/提取代码，不改行为 |
| 垂直切片 | 交付一个小的用户可见行为，端到端 |
| 水平切片 | 加共享接口/适配器，但必须配一个真实使用 |
| 按文件/owner 分组 | 按 reviewer 专业领域拆分 |
| 配置分离 | rollout 配置和代码分开 |
| 机械变更分离 | 自动生成的改动和手写的逻辑分开 |
| 删除分离 | 删死代码可以单独做一个 PR |

### 迁移安全：Expand/Contract 模式

对于 schema、数据、API 或基础设施迁移，skill 要求使用五阶段的 expand/contract 序列：

```
1. Expand    → 加向后兼容的新 schema/API，不删旧的
2. Dual path → 新旧路径同时读写，加观测
3. Backfill  → 迁移存量数据，带进度追踪和验证
4. Cut over  → 验证安全后切换流量
5. Contract  → 删除旧 schema、旧代码路径、兼容层
```

每个迁移 PR 必须说明：部署顺序、回滚行为、验证信号、什么必须等下一个 PR 合并后才能做。

---

## Skill 4: engineering-change-descriptions — 变更描述撰写

**一句话：** 写出好的 PR 描述和 commit message，让 reviewer 现在能审查、让未来的维护者能找到。

### 什么时候用

- 写 PR/CL 描述
- 改进 commit message
- 总结一个 diff 的内容
- 让 PR 描述更容易审查

### 好的描述回答五个问题

1. **What changed?** — 改了什么
2. **Why did it change?** — 为什么改
3. **What context is not obvious from the code?** — 代码里看不到的上下文
4. **How was it tested?** — 怎么测试的
5. **What risks, migrations, or follow-ups?** — 风险、迁移、后续工作

### 第一行的规则

- 在版本控制历史中独立可读
- 短、具体、聚焦于改动做了什么
- 用祈使句（"Add rate limiting" 而不是 "Added rate limiting"）
- 禁止模糊标签："fix bug"、"cleanup"、"phase 1"、"misc"

### 四种模板

**标准 PR：**

```markdown
[祈使句一行总结]

## Why
[问题、用户需求、生产事故、可维护性原因]

## What Changed
- ...

## Testing
- ...

## Risk / Rollback
- Risk:
- Rollback:
```

**小改动：**

```markdown
[祈使句一行总结]

[一段话解释为什么这个改动存在，以及不明显的上下文]

Testing: [命令或手动检查]
```

**重构：**

```markdown
[描述结构性变更和它替代了什么]

This prepares for [未来改动] by [具体原因], without intentionally changing behavior.

Testing: [保护行为的测试]
```

**机械变更：**

```markdown
[描述机械性重写]

Generated by [工具/版本/命令] to [原因]. Manual edits: [none/list].

Verification: [格式/测试/构建命令]
```

---

## Skill 5: engineering-review-feedback — 处理审查反馈

**一句话：** 作为代码作者，专业地处理 reviewer 的评论——什么时候改代码、什么时候回复、什么时候升级到高带宽沟通。

### 什么时候用

- 你收到了 review 评论需要处理
- 你不知道怎么回复 reviewer
- 你和 reviewer 有分歧
- 你想高效地处理一轮 review 的所有评论

### 响应工作流

1. **Pause** — 如果评论让你不爽，先暂停
2. **Identify** — 弄清楚 reviewer 在要求什么
3. **Classify** — 分类每条评论：
   - 需要改代码
   - 需要改测试
   - 需要改文档或注释
   - 需要澄清
   - 有分歧需要讨论
   - 非阻塞建议
4. **Prefer improving the artifact** — 优先改代码/测试/文档，而不是在 review thread 里解释
5. **Run verification** — 跑相关测试
6. **Reply** — 回复改了什么、为什么、怎么验证的

### 四种场景的回复模板

**同意时：**
```text
Done. I renamed the helper to describe normalization rather than validation
and updated the two call sites.

Tested with: npm test -- user-profile.test.ts
```

**需要澄清时：**
```text
I want to make sure I understand the concern. Are you asking for this to
reject duplicate requests at the API boundary, or for the worker to make
duplicate processing idempotent?
```

**不同意时：**
```text
I chose X because it keeps the retry policy in one place and avoids a second
queue state. My concern with Y is that it makes cancellation observable in
two different layers.

If you think Y better serves the rollback requirement, I can switch to it.
Is that the tradeoff you want prioritized?
```

**面对非建设性反馈时：**
> Do not reply in kind. Extract any technical concern that can be addressed in code, tests, or docs.

### "在正确的地方修复"原则

| 问题类型 | 修复位置 |
|---------|---------|
| 代码不清晰 | 简化或重命名 |
| 隐藏约束 | 编码到类型、验证或注释中 |
| 缺少行为保障 | 加测试 |
| 缺少上下文 | 更新 PR 描述、代码注释、README |
| Reviewer 因为旧 diff 误解 | rebase 或更新描述 |
| 纯偏好 | 问是否是必须的，否则当 optional 处理 |

---

## Skill 6: engineering-emergency-changes — 紧急变更决策

**一句话：** 判断一个变更是不是真正的紧急情况，如果是，走快速审查通道而不丢失工程判断。

### 什么时候用

- 有人说"这是紧急的，跳过 review"
- 生产事故需要 hotfix
- 安全漏洞需要紧急修复
- 有人想用"紧急"来绕过正常流程

### Emergency Gate：什么算真正的紧急

**算紧急的：**

- 阻止一个重大发布失败，且回滚不是正确的选择
- 修复显著影响生产用户的 bug
- 解决紧迫的法律或合规问题
- 关闭重大安全漏洞
- 满足硬性外部截止日期（合同、法律、安全）

**不算紧急的：**

- 想早点发布
- 功能做了很久，作者想赶紧合并
- Reviewer 在睡觉或在另一个时区
- 周五下午了
- 经理说今天要完成（软性截止日期）
- 团队想避免正常 review 的不适

> Most deadlines are soft. Do not sacrifice code health for a soft deadline.

### 紧急审查工作流

1. **State the emergency claim** — 一句话说清楚为什么紧急
2. **Decide** — 是否通过 emergency gate
3. **Shrink** — 缩小到最小修复
4. **Confirm** — 确认修复直接解决紧急问题
5. **Review** — 审查正确性、影响范围、回滚方案、安全问题
6. **Verify** — 跑最快有意义的验证
7. **Record** — 记录跳过了哪些正常检查以及为什么
8. **Merge** — 仅在紧急风险大于代码风险时合并
9. **Follow up** — 紧急解决后，做一次正常深度的 review
10. **File cleanup** — 立即创建后续清理任务

### 输出模板

```markdown
## Emergency Decision
QUALIFIES | DOES_NOT_QUALIFY | NEEDS_MORE_INFO

## Reason
[与生产用户影响、安全、法律/合规、重大发布或硬性截止日期相关的具体原因]

## Minimum Safe Change
- ...

## Immediate Verification
- ...

## Release / Rollback Notes
- ...

## Follow-up Required
- Normal-depth review:
- Tests:
- Cleanup:
- Documentation:
- Owner:
```

---

## 六个 Skill 怎么配合使用

六个 skill 不是孤立的，它们形成一个完整的工程实践闭环。一个典型的 PR 生命周期：

```
开发者写完代码
    ↓
[small-prs] 拆分大 PR 为小 PR 序列
    ↓
[change-descriptions] 写好 PR 描述
    ↓
提交 PR
    ↓
[code-review] Agent（或人类）审查代码
    ↓
[review-comments] 审查者写出带严重级别的评论
    ↓
[review-feedback] 作者处理反馈、修改代码
    ↓
合并
```

如果中间出现紧急情况：

```
生产事故
    ↓
[emergency-changes] 判定是否真正紧急
    ↓
是 → 走快速通道，缩小范围，事后补 review
否 → 走正常流程
```

## 使用技巧

### 让 Agent 审查自己的代码

写完代码后，让 Agent 用 engineering-code-review skill 做一轮自查。这不是替代人类 review，而是在提交前先把明显问题修掉。

### 用 small-prs 规划大功能

开始一个大功能前，先让 Agent 用 engineering-small-prs 拆分 PR 序列。这比写完了再拆要好得多——你在规划阶段就能发现依赖关系和风险点。

### 用 change-descriptions 的模板

不要从零开始写 PR 描述。四种模板覆盖了绝大多数场景。选一个，填空就行。

### 遇到 pushback 用 review-feedback 的框架

当 reviewer 不同意你的做法时，不要急着反驳。按 review-feedback 的工作流：先重新评估、再分类、然后在正确的地方修复。

### 对"紧急"保持警惕

emergency-changes 的反向清单是这个 skill 最有价值的部分。下次有人说"这个很急"时，对照清单看看——大多数"紧急"其实不紧急。

---

*源码：[github.com/agentara/skills/tree/main/skills/engineering](https://github.com/agentara/skills/tree/main/skills/engineering)*

*姊妹篇：《从 Google 工程实践学 Agent Skill 设计：六个 Prompt 工程模式的深度拆解》——拆开源码，看这些 skill 的 prompt 是怎么写的、为什么这么写。*

---

## 参考链接

- [Google Engineering Practices Documentation](https://google.github.io/eng-practices/)
- [Google Code Review Overview](https://google.github.io/eng-practices/review/reviewer/)
- [Google Small CLs Guide](https://google.github.io/eng-practices/review/developer/small-cls.html)
- [Google CL Descriptions](https://google.github.io/eng-practices/review/developer/cl-descriptions.html)
- [Google Handling Review Comments](https://google.github.io/eng-practices/review/developer/handling-comments.html)
- [Google Emergency Changes](https://google.github.io/eng-practices/review/emergencies.html)
