---
title: "Superpowers-Zh：让你的 AI 编程工具真正会干活"
description: "面向没用过这个项目的程序员，5 分钟搞懂它是什么、怎么装、怎么用。"
---

# Superpowers-Zh：让你的 AI 编程工具真正会干活

> 面向没用过这个项目的程序员，5 分钟搞懂它是什么、怎么装、怎么用。

---

## 你在用 AI 编程工具时遇到过这些问题吗？

让 Claude Code / Cursor / Copilot 帮你加一个"用户批量导出"功能：

```
你：给用户模块加个批量导出功能
AI：好的，我来实现...（直接开始写代码）
    export async function exportUsers() { ... }
你：等等，格式不对，没分页，大数据量会 OOM...
```

AI 没有错——它只是缺少**工作方法论**。它能写代码，但不知道"动手之前该先想清楚需求"、"写完代码要先跑测试"、"提交前要自查一遍"。

**Superpowers-zh** 就是来解决这个问题的。

## 一句话解释 Superpowers-Zh

它是一套 **AI 编程 skills 框架**——不是代码库、不是 SDK、不是 IDE 插件，而是一组 **Markdown 格式的工作方法论文件**，放到你的项目里之后，AI 编程工具会在合适时机自动加载并遵循这些方法论。

类比：给 AI 装了一个"老师傅的脑子"。

| 项目 | 数据 |
|------|------|
| 上游 | [obra/superpowers](https://github.com/obra/superpowers)（159k+ ⭐） |
| 本项目 | [jnMetaCode/superpowers-zh](https://github.com/jnMetaCode/superpowers-zh)（中文增强版） |
| Skills 数量 | **20 个**（14 翻译 + 6 中国原创） |
| 支持工具 | **18 款**（Claude Code / Cursor / Copilot / Gemini CLI / Hermes Agent / Codex / Trae / Kiro / Windsurf / Qoder / 通义灵码 等） |
| 安装方式 | 一条命令 |
| 协议 | MIT |

## 这 20 个 Skills 能干什么？

### 核心工作流 Skills（自动触发）

| Skill | 干什么 | 装之前 AI 的表现 | 装之后 AI 的表现 |
|-------|--------|-----------------|-----------------|
| **头脑风暴** (brainstorming) | 需求分析 → 设计规格 | 拿到需求直接写代码 | 先问清格式、数据量、权限，给 2-3 个方案再动手 |
| **编写计划** (writing-plans) | 把规格拆成可执行步骤 | 一口气写 500 行然后到处是 bug | 拆成 10 个独立可验证的小任务 |
| **执行计划** (executing-plans) | 按计划逐步实施 | 跳步、漏步、改一半忘了另一半 | 严格逐步执行，每步验证 |
| **测试驱动开发** (test-driven-development) | 严格 TDD | 写完代码再补测试（或不写） | 先写失败测试 → 写最小代码通过 → 重构 |
| **系统化调试** (systematic-debugging) | 四阶段调试法 | 改一行试试 → 不对再改 → 随机碰运气 | 定位 → 分析 → 假设 → 修复，每步有证据 |
| **请求代码审查** (requesting-code-review) | 派遣审查 agent | 写完直接提交 | 完成后自动派 agent 检查质量、安全、规范 |
| **完成前验证** (verification-before-completion) | 证据先行 | 写完说"完成了" | 声称完成前先跑验证、贴输出证据 |
| **子 Agent 驱动开发** (subagent-driven-development) | 每个任务一个 agent | 串行做 10 个任务 | 并行派发，每个 agent 独立完成后汇总 |
| **Git Worktree 使用** | 隔离式特性开发 | 在 main 分支直接改 | 每个功能独立 worktree，互不干扰 |
| **MCP 服务器构建** | 构建生产级 MCP 工具 | — | 按生产标准构建，扩展 AI 能力边界 |
| **工作流执行器** | 多角色 YAML 编排 | — | 在 AI 工具内运行多角色协作工作流 |

### 🇨🇳 中国特色 Skills（手动调用）

以下 4 个 skill 设计为**手动触发**（对话中输入 `/chinese-xxx`），避免干扰上游 skill 的自动调度：

| Skill | 用途 |
|-------|------|
| `/chinese-code-review` | 适配国内团队文化的代码审查规范 |
| `/chinese-git-workflow` | Gitee / Coding / 极狐 GitLab / CNB 工作流 |
| `/chinese-documentation` | 中文排版规范、中英混排规则、告别机翻味 |
| `/chinese-commit-conventions` | Conventional Commits 中文适配 |

## 安装：一条命令搞定

### 前置条件

- Node.js 18+（用于 `npx`）
- 你正在使用的 AI 编程工具（任选一个）

### 安装步骤

```bash
# 1. 进入你的项目目录
cd /your/project

# 2. 一条命令安装
npx superpowers-zh
```

就这么简单。安装器会自动：

1. **检测**你项目中使用了哪个 AI 编程工具（看有没有 `.claude/`、`.cursor/`、`.hermes/` 等目录）
2. **复制** 20 个 skills 到对应位置
3. **生成** bootstrap 引导文件，让 skills 能自动触发
4. **配置** hooks，在合适时机加载合适的 skill

### 检测不到你的工具？显式指定

```bash
npx superpowers-zh --tool hermes    # Hermes Agent
npx superpowers-zh --tool cursor    # Cursor
npx superpowers-zh --tool qoder     # Qoder（阿里 AI IDE）
npx superpowers-zh --tool copilot   # VS Code Copilot
```

完整 `--tool` 可选值：`claude` / `copilot` / `hermes` / `cursor` / `windsurf` / `kiro` / `gemini` / `codex` / `aider` / `trae` / `vscode` / `deerflow` / `opencode` / `openclaw` / `qwen` / `antigravity` / `claw` / `qoder`

### ⚠️ 不要在家目录运行

```bash
# ❌ 错误
cd ~
npx superpowers-zh

# ✅ 正确
cd ~/projects/my-app
npx superpowers-zh
```

v1.2.1 起会拒绝并提示。老版本会把 skills 写到 home 目录污染所有项目。如果已经误装过：

```bash
cd ~
npx superpowers-zh@latest --uninstall
```

### 卸载

```bash
cd /your/project
npx superpowers-zh@latest --uninstall
```

卸载器会精确删除 superpowers-zh 安装的内容（基于哨兵注释标记），不会动你自己写的配置。

## 安装之后：怎么验证它生效了？

装完之后不需要做任何额外配置。下次开 AI 编程工具的新会话时，skills 已经就位。

**验证方法**——给 AI 一个会触发 brainstorming skill 的请求：

```
你：帮我做一个 React Todo List
```

如果安装成功，AI 不会直接写代码，而是会先问你需求细节（UI 风格、数据存储、是否需要后端等）。这就是 `brainstorming` skill 被自动触发了。

如果 AI 直接开始写代码，说明安装可能没成功。检查：

```bash
# 看你项目的 skills 目录有没有文件
ls .claude/skills/     # Claude Code
ls .cursor/skills/     # Cursor
ls .hermes/skills/     # Hermes Agent
```

## 日常使用：自动触发，但程度因工具而异

装好之后，skills 的介入程度取决于你用的 AI 工具支持哪种加载机制：

### 第一梯队：完全自动触发（你什么都不用做）

| 工具 | 机制 |
|------|------|
| **Claude Code** | `SessionStart` hook，会话开始时自动注入 `using-superpowers` 引导内容 |
| **Copilot CLI** | 同上，走 SDK 标准 hook 格式 |
| **Cursor** | 同上，Cursor plugin hooks |
| **Trae** | 安装时生成 `.trae/rules/superpowers-zh.md`（`alwaysApply: true`），每次对话自动加载 |
| **Qoder** | 安装时生成 `.qoder/rules/superpowers-zh.md`（`trigger: always_on`），始终生效 |
| **Aider** | 安装时写入/追加 `CONVENTIONS.md`，Aider 启动时自动读取 |
| **Gemini CLI** | 安装时写入/追加 `GEMINI.md`，会话开始时自动加载 |
| **Hermes Agent** | 安装时写入/追加 `HERMES.md`，会话开始时自动加载 |

这些工具装完 superpowers-zh 之后，你正常对话就行。AI 会在合适时机自动调用对应 skill：

| 你说的话 | 自动触发的 Skill |
|---------|-----------------|
| "帮我加个 XX 功能" | 头脑风暴 → 编写计划 → 执行计划 |
| "跑一下测试" / 写完代码 | 测试驱动开发 |
| "这个 bug 怎么修" | 系统化调试 |
| "帮我 review 一下" | 请求代码审查 |
| "开一个新分支做 XX" | Git Worktree 使用 |

### 第二梯队：skills 已就位，但引导较弱

| 工具 | 情况 |
|------|------|
| **Windsurf** | skills 复制到了 `.windsurf/skills/`，但没有生成额外引导文件 |
| **OpenCode** | 同上，`.opencode/skills/` |
| **Kiro** | skills 复制到了 `.kiro/steering/` |
| **DeerFlow** | skills 复制到了 `skills/custom/` |
| **OpenClaw** | skills 复制到了 `skills/` |
| **Qwen Code** | skills 复制到了 `.qwen/skills/` |
| **Claw Code** | skills 复制到了 `.claw/skills/` |

这些工具能不能自动发现并加载 skills，取决于工具本身的能力。如果 AI 没有主动调用 skill，你可能需要在对话开头说一句"请查看 `.xxx/skills/` 目录下的技能并使用"来引导它。

### 中国特色 Skills 需要手动调用

以下 4 个 skill 设计为手动触发（避免干扰上游 skill 的自动调度）：

```
你：/chinese-code-review
你：/chinese-commit-conventions
你：/chinese-documentation
你：/chinese-git-workflow
```

## 手动安装（当 npx 不可用时）

极端无网络环境下可以手动复制：

```bash
# 克隆仓库
git clone https://github.com/jnMetaCode/superpowers-zh.git

# 复制到你的项目（选择对应工具的目录）
cp -r superpowers-zh/skills /your/project/.claude/skills      # Claude Code
cp -r superpowers-zh/skills /your/project/.cursor/skills      # Cursor
cp -r superpowers-zh/skills /your/project/.hermes/skills      # Hermes Agent
# ... 其他工具见 README
```

> **注意**：手动复制只是"低保版"——skills 文件在了，但 hooks 和 bootstrap 不会自动配置，需要每次手动让 AI 加载对应 skill。强烈建议优先用 `npx`。

## 和其他类似项目有什么区别？

| 维度 | 裸用 AI 工具 | Superpowers-Zh | 自定义 CLAUDE.md |
|------|-------------|----------------|-----------------|
| 安装成本 | 无 | 一条命令 | 手写几十行 |
| 方法论覆盖 | 无 | 20 个经过实战验证的 skill | 取决于你写了多少 |
| 自动触发 | — | ✅ hooks 自动调度 | ❌ 需要手动引用 |
| 工具兼容 | 单一工具 | 18 款工具一条命令切换 | 工具专属 |
| 中文化 | 英文 | 中文（技术术语保留英文） | 自己翻译 |
| 社区维护 | — | 活跃（同步上游 + 国产增量） | 自己维护 |

## 总结

**Superpowers-zh 解决的问题**：AI 编程工具能写代码，但缺乏系统化的工作方法论（先想清楚再动手、TDD、调试要有章法、提交前要自查）。

**它的解法**：20 个 Markdown 格式的 skill 文件，放到项目里后自动被 AI 加载和遵循。

**你需要做的**：`cd /your/project && npx superpowers-zh`，一条命令，30 秒搞定。

---

🔗 项目地址：[github.com/jnMetaCode/superpowers-zh](https://github.com/jnMetaCode/superpowers-zh)
📖 配套教程：[《AI 编程实战 · 方法论三卷书》](https://book.aibuzhiyu.com/)
