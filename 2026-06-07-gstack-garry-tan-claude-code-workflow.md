---
title: "gstack：Y Combinator CEO 的 Claude Code 工作流框架"
description: "Garry Tan 开源的 46 个 skill 框架，把 Claude Code 从通用助手变成虚拟工程团队——CEO 做战略、Staff Engineer 做审查、QA Lead 开浏览器测试"
---

# gstack：Y Combinator CEO 的 Claude Code 工作流框架

> 一个人 + AI agent，能不能干过一个 20 人团队？gstack 的答案是：能，前提是你给它一套结构化的工作流。

## 项目概况

| 指标 | 数据 |
|------|------|
| ⭐ Stars | 107,677 |
| 🔀 Forks | 16,014 |
| 📅 创建时间 | 2026-03-11 |
| 📅 最近更新 | 2026-06-07 |
| 📝 Open Issues | 610 |
| 📄 License | MIT |
| 🏷️ 维护状态 | 个人开源项目（非 YC/Anthropic 官方） |
| 当前版本 | 1.56.0.0 |
| 语言 | TypeScript |
| 代码规模 | 645 个 .ts 文件（364 非测试 + 281 测试），52,032 行 SKILL.md |

**作者**：Garry Tan，Y Combinator 现任 CEO。Palantir 早期工程/产品/设计负责人，Posterous 联合创始人（后被 Twitter 收购）。

**GitHub**：https://github.com/garrytan/gstack

---

## gstack 是什么

一个早期创业团队（1-2 人）需要同时覆盖产品思考、架构设计、写代码、代码审查、测试、安全审计、发布部署——但人手不够。

gstack 把 Claude Code 从一个通用 AI 助手变成一支虚拟工程团队。它有 46 个 slash command，每个 command 对应一个专家角色：CEO 做战略审视，工程经理做架构评审，设计师做 UX 评审，Staff Engineer 做代码审查，QA Lead 做浏览器测试，安全官做 OWASP 审计，发布工程师做 PR 管理。你在 Claude Code 里输入对应的 slash command，Claude 就切换到那个角色的视角和行为模式。

这 46 个 command 不是孤立的工具集合。它们按软件开发的完整流程排列——**想清楚 → 规划 → 写代码 → 审查 → 测试 → 发布 → 复盘**——前一个 command 的输出自动成为后一个的输入。gstack 把这个完整流程叫一次 **sprint**（冲刺，借用敏捷开发的术语，指从想法到发布的一次完整迭代）。

gstack 不是一个独立 IDE 或浏览器插件。它的核心是一组 SKILL.md 文件——每个文件用 YAML frontmatter 声明可用工具，用 Markdown 正文定义该角色的完整行为规范。安装后，你在 Claude Code 里直接输入 slash command 即可触发。

Garry Tan 自己报告的数据：2026 年前 108 天，41 个仓库中 351 次 commit，1,233,062 行逻辑代码（非空非注释），日均 11,417 行。对比 2013 年全年数据（5,143 行，日均 14 行），倍数是 810x。按 AI 代码冗余度 2x 折算仍有 408x。revert 率 2.0%，与成熟开源项目 1-3% 基准持平。

---

## 一次 sprint 用了哪些命令

在展开 46 个 skills 之前，先看一次完整 sprint 的命令序列。下面这个表格是全文的导航图——后面所有详解都围绕它展开：

| 阶段 | 命令 | 角色 | 做什么 | 产出物（喂给下游） |
|------|------|------|--------|-------------------|
| 想清楚 | `/office-hours` | YC Office Hours 导师 | 6 个 forcing question 逼你暴露需求真相 | design doc |
| 规划 | `/plan-ceo-review` | CEO / 创始人 | 审视 design doc，挑战范围，找到 10 星级产品 | CEO review report |
| 规划 | `/plan-eng-review` | 工程经理 | 锁定架构、数据流、边界情况、测试策略 | 架构文档 + test plan |
| 规划 | `/plan-design-review` | 高级设计师 | 每个设计维度打分，解释 10 分长什么样 | 设计规格 |
| 写代码 | （退出 plan mode，Claude 直接写） | 开发者 | 按规划实现功能 | 代码变更（git diff） |
| 审查 | `/review` | Staff Engineer | 分析 diff，自动修复明显问题，标记风险 | review 意见 + auto-fix |
| 测试 | `/qa` | QA Lead | 打开真实浏览器测试，找 bug 并修复 | bug fix + 回归测试 |
| 安全 | `/cso` | Chief Security Officer | OWASP Top 10 + STRIDE 威胁模型 | 安全审计报告 |
| 发布 | `/ship` | Release Engineer | 跑测试、审计覆盖率、push、开 PR | PR + 测试报告 |
| 部署 | `/land-and-deploy` | Release Engineer | merge PR、等 CI、验证生产环境 | 生产验证 |
| 复盘 | `/retro` | 工程经理 | 周回顾：per-person 拆解、shipping streaks | 改进建议 |

不是每次 sprint 都要跑完所有步骤。小改动可能只需要 `/review` → `/ship`。新功能从 `/office-hours` 开始。安全审计可能单独跑 `/cso`。但当你需要完整流程时，这个序列保证了没有遗漏。

### `/autoplan`：一键跑完所有规划 review

如果你不想手动跑 `/plan-ceo-review` → `/plan-design-review` → `/plan-eng-review`，`/autoplan` 一条命令自动跑完全部。它编码了 6 条决策原则，自动做大部分决定，只把品味判断（"方案 A 还是方案 B"这种没有客观正确答案的选择）留给用户审批。

---

## skills 之间怎么传递信息

这是 gstack 和普通 prompt 模板的根本区别。46 个 skills 不是各说各话——它们通过文件系统中的产物（artifact）衔接。

### 产物驱动的交接链

```
/office-hours  →  写 design doc 到 ~/.gstack/projects/{repo}/
                      ↓
/plan-ceo-review  →  读取 design doc，输出 CEO review report
                      ↓
/plan-eng-review  →  读取 design doc + CEO review，输出 test plan + 架构文档
                      ↓
（Claude 写代码，产出 git diff）
                      ↓
/review  →  分析 git diff（自动检测 base branch），产出 review 意见
                      ↓
/qa  →  读取 test plan，打开浏览器测试，每个 bug 用原子 commit 修复
                      ↓
/ship  →  跑测试（包含 /qa 生成的回归测试），审计覆盖率，push + 开 PR
```

具体机制：

1. **设计文档链**：`/office-hours` 把 design doc 写到 `~/.gstack/projects/{repo_slug}/` 目录下。`/plan-ceo-review` 的 YAML frontmatter 中有 `gbrain.context_queries`，用 glob 模式找到这些文件并注入上下文。`/plan-eng-review` 同理。三个 skill 读同一份 design doc，各自从自己的角色视角审视。

2. **git diff 链**：`/review` 不依赖前序 skill 的任何输出——它直接分析当前分支相对 base branch 的 diff。这意味着你可以跳过所有规划步骤，直接在已有代码上跑 `/review`。

3. **测试链**：`/qa` 的每个 bug 修复是一个原子 commit，同时生成一个回归测试文件。`/ship` 跑全部测试时自然包含这些回归测试。bug 修了没有？测试说了算。

4. **文档链**：`/ship` 自动调用 `/document-release`，后者扫描项目中所有文档文件，与 diff 交叉引用，更新漂移内容。代码改了，文档跟着改。

### 为什么这样设计：避免 agent 交接中的信息丢失

多次迭代的 agent 工作流中，最危险的环节是交接。开发 agent 写了 2000 行代码，code review agent 拿到一堆 diff，不知道哪些是核心逻辑、哪些是 boilerplate。review 提了 5 个意见，开发 agent 改了 3 个，忘了 2 个，又引入了 1 个新 bug。

gstack 的解法：**不让 agent 之间直接对话，而是通过产物交接。** 每个 skill 读的是文件（design doc、git diff、test plan），不是上一个 skill 的聊天记录。这有几个好处：

- 产物是持久化的，不会因为 session 中断丢失
- 任何 skill 可以单独跑，不依赖前序 skill 的上下文
- 审查者（`/review`）看的是 git diff 这个客观事实，不是开发者 agent 的自我报告
- `/investigate` 调试时自动 `/freeze` 到被调查模块，防止修复过程中意外改动范围外代码

还有一个跨 session 记忆机制：`/learn` 管理 gstack 学到的项目特有 pattern、pitfall、preference，写入 `~/.gstack/projects/{slug}/learnings.jsonl`。下次任何 skill 启动时，这些 learnings 会被注入上下文。gstack 在你的代码库上越用越聪明。

---

## 浏览器测试的原理

### 为什么 gstack 需要浏览器

`/qa`、`/design-review`、`/canary`、`/benchmark` 这些 skill 有一个共同需求：看到你的 web 应用实际渲染出来的样子。

单元测试和 CI 能验证逻辑正确性，但抓不到这些问题：
- CSS 在不同视口下错位
- 按钮点了没反应（事件绑定问题）
- 加载时序导致的内容闪烁
- 表单交互流程断裂
- 移动端适配崩溃

只有"看到"页面才能发现。

### 技术栈：Playwright + CDP + 常驻 daemon

gstack 的浏览器测试基于 Playwright——微软开源的浏览器自动化框架，支持 Chromium/Firefox/WebKit。Playwright 通过 Chrome DevTools Protocol (CDP) 与 Chromium 通信，可以执行点击、输入、截图、获取页面快照等所有浏览器操作。

但 Playwright 的标准用法是每次操作启动一个新的浏览器实例，冷启动 2-3 秒，且每次启动 cookies、localStorage、登录状态全部清零。对 QA session（通常 20+ 命令）来说，这意味着 40+ 秒纯等待，加上没法测试需要登录的页面。

gstack 的方案是**常驻 daemon 模型**：

```
Claude Code                     gstack
─────────                      ──────
                               ┌──────────────────────┐
  Tool call: $B snapshot -i    │  CLI (compiled binary)│
  ─────────────────────────→   │  • reads state file   │
                               │  • POST /command      │
                               │    to localhost:PORT   │
                               └──────────┬───────────┘
                                          │ HTTP
                               ┌──────────▼───────────┐
                               │  Server (Bun.serve)   │
                               │  • dispatches command  │
                               │  • talks to Chromium   │
                               │  • returns plain text  │
                               └──────────┬───────────┘
                                          │ CDP
                               ┌──────────▼───────────┐
                               │  Chromium (headless)   │
                               │  • persistent tabs     │
                               │  • cookies carry over  │
                               │  • 30min idle timeout  │
                               └───────────────────────┘
```

一个长驻 Chromium 进程在后台运行。CLI 是 Bun 编译的单二进制（~58MB），通过 localhost HTTP 与 daemon 通信。首次调用启动所有组件（~3 秒），之后每次调用 ~100-200ms。

**持久状态的实际意义**：
- 登录一次，整个 QA session 保持登录
- `/setup-browser-cookies` 可以从你的真实 Chrome/Arc/Brave/Edge 导入 cookies，直接测试已认证页面
- 打开的 tab 在命令间保留
- 30 分钟无操作自动关闭，下次使用自动重启

**Ref 系统**：agent 与页面元素交互时不写 CSS 选择器。`$B snapshot -i` 获取页面 ARIA accessibility 树，为每个元素分配 ref（@e1、@e2...）。`$B click @e3` 在 server 端解析为 Playwright 的 `getByRole(role, { name }).nth(index)` 然后执行。这比 CSS 选择器更稳定——页面重构不影响 ref，只要 ARIA 语义没变。

**为什么选 Bun 而非 Node.js**：`bun build --compile` 产生单一可执行文件，运行时不需要 node_modules（gstack 安装在 `~/.claude/skills/` 里，用户不该在那里管理 Node.js 项目）；内置 SQLite 直接读 Chromium cookie 数据库，不需要 better-sqlite3 的 native addon 编译；原生 TypeScript 支持，开发时直接 `bun run server.ts`。

### `/qa` 的实际工作流程

`/qa` 的三个测试层级：
- **Quick**：只测 critical/high 严重度问题
- **Standard**：+ medium
- **Exhaustive**：+ cosmetic

执行过程：打开浏览器 → 访问目标 URL → 按 accessibility 树遍历页面交互 → 发现问题 → 在源码中原子 commit 修复 → 重新浏览器验证修复有效 → 生成回归测试。

Garry Tan 说 `/qa` 是他的"massive unlock"——让他从 6 个并行 worker 扩展到 12 个。之前他只能让 agent 写代码和做 review，有了浏览器测试后，agent 也能独立做 QA。

---

## 46 个 skills 详解

以下按 sprint 阶段分组。每个 skill 标注对应角色。

### 一、想清楚（Think）

| 命令 | 角色 | 2-3 句描述 |
|------|------|-----------|
| `/office-hours` | YC Office Hours 导师 | 两种模式：Startup（6 个 forcing question 验证创业想法的需求真实性、现状替代方案、最窄可行楔子）和 Builder（设计思维脑暴，适合 side project 和学习）。输出 design doc 写入 `~/.gstack/projects/{repo}/`，自动喂给下游 skills。当你对 Claude 说"I have an idea"或"is this worth building"时主动触发 |

### 二、规划（Plan）

| 命令 | 角色 | 描述 |
|------|------|------|
| `/plan-ceo-review` | CEO / 创始人 | 读取 `/office-hours` 的 design doc，从战略视角挑战范围。4 种模式：Expansion（dream big）、Selective Expansion（保持范围 + 挑重点扩展）、Hold Scope（严格执行）、Reduction（精简到核心）。目标：找到需求中隐藏的 10 星级产品 |
| `/plan-eng-review` | 工程经理 | 读取 design doc + CEO review，锁定架构决策。用 ASCII 图画数据流、状态机、错误路径。输出测试矩阵、失败模式分析、安全关注点。强制把隐藏假设暴露到明面上 |
| `/plan-design-review` | 高级设计师 | 每个设计维度打 0-10 分，解释 10 分长什么样，然后编辑计划达到 10 分。内置 AI Slop 检测（识别 AI 生成的平庸设计）。交互式：每个设计决策单独问你 |
| `/plan-devex-review` | DX Lead | 专门面向开发者体验。探索开发者画像、对标竞品 TTHW、设计 magical moment、逐步追踪摩擦点。三种模式：DX EXPANSION / DX POLISH / DX TRIAGE。产生 20-45 个 forcing questions |
| `/autoplan` | 自动化流水线 | 一条命令跑完 CEO → design → eng → DX review。编码 6 条决策原则，只把品味判断留给用户。输出完整 reviewed plan |
| `/spec` | Spec Author | 五阶段将模糊意图变成可执行 spec：why → scope → technical（强制读源码）→ draft → file。Codex 质量门禁（低于 7/10 阻止写入）。`--execute` 在 fresh worktree 中 spawn `claude -p` 直接实现。`/ship` 在 merge 时自动关闭源 issue |
| `/design-consultation` | Design Partner | 从零构建完整设计系统。研究市场现有设计、提出创意风险、生成产品 mockup。输出 DESIGN.md |

### 三、写代码 + 设计实现（Build）

| 命令 | 角色 | 描述 |
|------|------|------|
| `/design-shotgun` | Design Explorer | 生成 4-6 个 AI mockup 变体（GPT Image），在浏览器中打开对比板让你并排看。你选喜欢的，留反馈，它迭代。内置 taste memory：学习你实际选择的偏好。重复直到满意，然后交给 `/design-html` |
| `/design-html` | Design Engineer | 把 mockup 变成生产级 HTML/CSS。使用 Pretext 做计算文本布局——文本真正地在 resize 时 reflow，不是那种"一个视口好看、其他全崩"的 AI HTML。30KB 开销，零依赖。检测 React/Svelte/Vue 输出对应格式 |

### 四、审查（Review）

| 命令 | 角色 | 描述 |
|------|------|------|
| `/review` | Staff Engineer | 分析当前分支相对 base branch 的 diff。检查 SQL 安全、LLM 信任边界违反、条件副作用、结构问题。自动修复明显的，标记完整性缺口。当你要 merge 时主动建议 |
| `/codex` | 跨模型审查员 | 调用 OpenAI Codex CLI 做独立审查。三种模式：review（pass/fail 门禁）、adversarial challenge（主动尝试破坏代码）、open consultation。当 `/review` 和 `/codex` 都审查了同一分支，输出跨模型分析：哪些发现重叠，哪些各自独有 |
| `/investigate` | Debugger | 四阶段：investigate → analyze → hypothesize → implement。铁律：没有调查不能修复。追踪数据流、测试假设、3 次失败后停止。自动 `/freeze` 到被调查模块 |
| `/design-review` | 会代码的设计师 | 与 `/plan-design-review` 相同方法论，但这次打开真实浏览器看线上页面，找到问题后直接原子 commit 修复，附 before/after 截图 |
| `/devex-review` | DX Tester | 实际测试你的 onboarding：导航文档、走 getting started 流程、计时 TTHW、截图错误。与 `/plan-devex-review` 的分数对比——"回力标"，显示计划是否匹配现实 |

### 五、测试（Test）

| 命令 | 角色 | 描述 |
|------|------|------|
| `/qa` | QA Lead | 打开真实 Chromium 浏览器（Playwright），访问 staging URL，像用户一样交互。发现 bug → 原子 commit 修复源码 → 重新浏览器验证 → 生成回归测试。三个层级：Quick / Standard / Exhaustive。输出 before/after 健康分数 |
| `/qa-only` | QA Reporter | 与 `/qa` 相同方法论，只报告不改代码。纯 bug report |
| `/benchmark` | Performance Engineer | 基线页面加载时间、Core Web Vitals、资源大小。每个 PR 对比 before/after |
| `/ios-qa` | iOS QA Engineer | 通过 USB CoreDevice 驱动真实 iPhone。读 Swift 源码、codegen accessor、运行 agent 循环。可选 `--tailnet` 暴露设备给远程 agent。能力分层 allowlist |
| `/ios-fix` | iOS Bug Fixer | 自主 iOS bug 修复循环，带回归快照捕获 |
| `/ios-design-review` | iOS 设计审查 | 10 维度 Apple HIG 评审 |

### 六、安全（Secure）

| 命令 | 角色 | 描述 |
|------|------|------|
| `/cso` | Chief Security Officer | OWASP Top 10 + STRIDE 威胁模型。基础设施优先：secrets 考古、依赖供应链、CI/CD 管道安全、LLM/AI 安全。零噪音：17 种误报排除，8/10+ 置信度门禁。每个发现附具体利用场景。daily 和 comprehensive 两种模式 |
| `/careful` | Safety Guardrails | rm -rf、DROP TABLE、force-push 前弹出警告。说"be careful"激活。可 override |
| `/freeze` | Edit Lock | 锁定文件编辑到一个目录。硬阻止，不是警告。`/investigate` 自动激活 |
| `/guard` | Full Safety | `/careful` + `/freeze` 一条命令 |

### 七、发布（Ship）

| 命令 | 角色 | 描述 |
|------|------|------|
| `/ship` | Release Engineer | 同步 main、跑全部测试（含 `/qa` 生成的回归测试）、审计覆盖率、push、开 PR。没有测试框架就 bootstrap。自动调用 `/document-release` 更新文档。Workspace-aware 版本队列 |
| `/land-and-deploy` | Release Engineer | merge PR → 等 CI → 等部署 → 验证生产环境健康。一条命令从 approved 到 verified in production |
| `/canary` | SRE | 部署后监控循环：console 错误、性能回归、页面失败 |
| `/document-release` | Technical Writer | 扫描所有文档文件，与 diff 交叉引用，更新漂移内容。构建 Diataxis 覆盖地图 |
| `/document-generate` | Doc Author | Diataxis 框架从零生成缺失文档。先研究代码库再写 |
| `/setup-deploy` | Deploy Configurator | `/land-and-deploy` 的一次性设置。检测平台、URL、部署命令 |

### 八、复盘（Reflect）

| 命令 | 角色 | 描述 |
|------|------|------|
| `/retro` | 工程经理 | 团队感知周回顾。按人拆解、shipping streaks、测试健康趋势。`/retro global` 跨所有项目和 AI 工具运行 |
| `/health` | Code Health | 代码质量仪表盘：type checker、linter、tests、dead code |
| `/learn` | Memory | 管理跨 session learnings。审查、搜索、修剪、导出。写入 JSONL，下次任何 skill 启动时注入 |

### 九、浏览器 + 跨 agent 协作

| 命令 | 角色 | 描述 |
|------|------|------|
| `/browse` | QA Engineer | 所有浏览器 skill 的底层。真实 Chromium，~100ms 每命令。Ref 系统交互 |
| `/open-gstack-browser` | Browser Operator | 启动可见 AI 控制浏览器。Sidebar agent：在 Chrome 侧边栏输入自然语言执行任务。Auto model routing（Sonnet 做动作，Opus 做分析） |
| `/setup-browser-cookies` | Session Manager | 从真实浏览器导入 cookies，测试已认证页面 |
| `/pair-agent` | Multi-Agent Coordinator | 让不同 AI agent（OpenClaw、Hermes、Codex）共享同一浏览器，各自独立 tab。Scoped tokens、tab isolation、rate limiting。支持 ngrok tunnel 给远程 agent |

---

## 安全模型

### 浏览器 daemon

- HTTP server 绑定 `127.0.0.1`，网络不可达
- 每个 session 生成随机 UUID token，state file mode 0o600
- Cookie 解密在进程内进行（PBKDF2 + AES-128-CBC），不以明文写磁盘
- Cookie DB 只读（复制到临时文件后打开）
- Dual-listener tunnel 架构：本地 listener 暴露完整命令面，tunnel listener（ngrok 转发）只暴露 `/connect` + 受限 `/command`

### Prompt injection 防御

Sidebar agent 读取恶意网页，是最暴露的部分。5 层防御：
1. L1-L3 内容安全：datamarking、隐藏元素剥离、ARIA 正则、URL 黑名单
2. L4 ML 分类器：22MB BERT-small ONNX 模型，本地运行。可选 721MB DeBERTa-v3
3. L4b transcript 分类器：Claude Haiku 看完整对话形状
4. L5 canary token：随机 token 注入系统提示，出现在输出中则确定性 BLOCK
5. L6 ensemble 组合器：BLOCK 需两个分类器达成一致，防止 Stack Overflow 式误报

---

## 设计哲学：Builder Ethos

gstack 的 ETHOS.md 定义三条核心原则，自动注入每个 skill。

### Boil the Lake

AI 辅助让"完整"的边际成本趋近于零。完整实现比走捷径只多花几分钟——做完整的事。Lake 是可以煮沸的（100% 测试覆盖），Ocean 不可以（从头重写系统）。煮沸 lakes，标记 oceans 为 out of scope。

gstack 给出的压缩比：

| 任务类型 | 人类团队 | AI 辅助 | 压缩比 |
|---------|---------|--------|--------|
| Boilerplate | 2 天 | 15 分钟 | ~100x |
| 测试编写 | 1 天 | 15 分钟 | ~50x |
| 功能实现 | 1 周 | 30 分钟 | ~30x |
| Bug 修复 + 回归测试 | 4 小时 | 15 分钟 | ~20x |
| 架构 / 设计 | 2 天 | 4 小时 | ~5x |
| 研究 / 探索 | 1 天 | 3 小时 | ~3x |

### Search Before Building

第一反应是"有人解决过这个了吗"而非"从零设计"。三层知识：Layer 1（tried and true）、Layer 2（new and popular，需审查）、Layer 3（first principles，最有价值）。

### 并行 sprint

gstack 对一个 sprint 强大，对 10-15 个并行 sprint 变革性。Conductor 并行运行多个 Claude Code session，每个在隔离 workspace 中。sprint 结构是并行化能工作的前提——没有流程，十个 agent 是十个混乱源。

---

## 多 host 支持

| Agent | Flag | Skills 安装位置 |
|-------|------|----------------|
| Claude Code | 默认 | `~/.claude/skills/gstack/` |
| OpenAI Codex CLI | `--host codex` | `~/.codex/skills/gstack-*/` |
| OpenCode | `--host opencode` | `~/.config/opencode/skills/gstack-*/` |
| Cursor | `--host cursor` | `~/.cursor/skills/gstack-*/` |
| Kiro | `--host kiro` | `~/.kiro/skills/gstack-*/` |
| Hermes | `--host hermes` | `~/.hermes/skills/gstack-*/` |

添加新 agent 支持只需一个 TypeScript 配置文件。

---

## LOC 争议

Garry Tan 报告 810x 生产力提升。批评者说 LOC 无意义，AI 只是冗余代码。

Garry 在 `docs/ON_THE_LOC_CONTROVERSY.md` 中回应：批评的三个分支中，前两个对（LOC 不等于质量，AI 膨胀 LOC），第三个（因此吹嘘是尴尬的）跳轨了。

| | 2013（全年） | 2026（108 天） | 倍数 |
|--|--:|--:|--:|
| Logical SLOC | 5,143 | 1,233,062 | 240x |
| Logical SLOC/day | 14 | 11,417 | 810x |

应用 2x AI 冗长 deflation：日均 5,708 行，408x。10x 病理性 deflation 仍然 81x。

周分布不是单次爆发：Jan ~8,800/day → Feb ~12,100 → Mar ~10,900 → Apr ~13,200。

Revert 率 2.0%（351 commits 中 7 次 revert）。

---

## 安装

```bash
git clone --single-branch --depth 1 https://github.com/garrytan/gstack.git ~/.claude/skills/gstack && cd ~/.claude/skills/gstack && ./setup
```

团队模式：`./setup --team`，每个 Claude Code session 自动更新检查（节流到每小时一次，网络失败安全）。

遥测默认关闭。如果 opt in，只发送 skill 名称、持续时间、成功/失败、版本、OS。从不发送代码、文件路径、仓库名、提示词。

---

## 适合谁

- 技术创始人，1-2 人团队，需要覆盖产品、工程、设计、QA、安全多个职能
- Tech lead / staff engineer，需要每个 PR 都有严格的 review + QA + release 流程
- 已在用 Claude Code 但觉得 ad hoc prompting 效率不够的开发者

不适合：大团队有专职角色的、不想定制 CLAUDE.md 的、追求即插即用的。

107,677 stars 3 个月达成。核心价值不是 46 个 prompt 模板，而是 skills 之间的产物驱动衔接 + 浏览器 daemon 让 agent 有了眼睛。AI 编码工具很强，但没有结构化工作流，你就只是在 ad hoc prompting。gstack 把 ad hoc 变成了 sprint。

---

*GitHub: https://github.com/garrytan/gstack*