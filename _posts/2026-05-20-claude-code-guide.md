---
layout: post
title: "Claude Code 快速学习指南"
date: 2026-05-20
categories: [tools]
mermaid: false
math: false
toc: true
---
作为程序员快速上手 Claude Code 的完整学习路线与实战指南，涵盖从基础到高级的所有核心功能。

---

## 一、什么是 Claude Code？

## 🔥 近期新增能力速览（2026-03，建议持续对照官方 Changelog 校验）

> 下面是近期在社区与第三方解读中被频繁提及的更新点，我先按“你关心的产品能力”做了归类汇总。权威口径建议以官方 Changelog 为准：

- 官方 Changelog：[https://code.claude.com/docs/en/changelog](https://code.claude.com/docs/en/changelog)

### 1）自动化与长任务：/loop（定时循环任务）

- 让 Claude Code 以“循环执行”的方式做周期性工作。
- 典型用途：PR 巡检、依赖更新、部署监控、告警摘要等。

### 2）更强的“可安装生态”：Plugins（插件打包分发）

- 将 **Skills + Subagents + Hooks + MCP** 打包成可安装、可版本管理、可团队共享的插件。
- 配合 Marketplace 工作流，提升团队可复制性。

### 3）更强的“电脑级行动能力”：Computer Use / Remote Control（预览类能力）

- 讨论热度较高的方向是把 Claude Code 的行动能力从“终端”扩展到“桌面/远程控制”。
- 注意：通常属于研究预览或灰度能力，落地依赖环境与权限。

### 4）后台并行与任务隔离：Background Agent（后台执行）

- 将部分任务放到后台并行跑，避免阻塞主会话。
- 常见搭配：Git worktree 隔离、并行研究、长时间运行的测试与扫描。

### 5）安全与质量：Security 扫描（自动化安全检查）

- 目标是把“漏洞发现→建议修复”融入日常开发循环，减少人工审查成本。

### 6）上下文与长对话：更强的压缩与续航（Context Compaction）

- 对长会话做自动压缩，保留关键决策与约束，降低上下文膨胀带来的“遗忘”。

> 参考解读（第三方）：[https://help.apiyi.com/claude-code-2026-new-features-loop-computer-use-remote-control-guide.html](https://help.apiyi.com/claude-code-2026-new-features-loop-computer-use-remote-control-guide.html)

---

Claude Code 是 Anthropic 推出的一款**命令行 AI 编程助手**。它直接在终端中运行，能够**读取文件、执行命令、修改代码**，以高度自主的方式完成开发任务。

<div class="callout callout-tip" markdown="1">
**核心架构：**
- 基于 **Agentic Loop**（代理循环）自主决策和执行
- 内置五大类 **Tools**（工具）：文件读写、命令执行、任务委托等
- 通过 **Skills** 扩展能力，创建可复用的工作流
- 支持 **Subagents**（子代理）进行任务委托和并行研究
- 支持 **Agent Teams**（多代理团队）实现跨会话协作
- 通过 **Hooks**（钩子）在生命周期各阶段注入自定义行为
- 通过 **MCP** 和 **Plugins** 无限扩展外部能力
</div>

---

## 二、安装与配置

### 前置条件

- [ ] 一个终端或命令行工具
- [ ] 一个代码项目
- [ ] Claude 订阅（Pro、Max、Teams 或 Enterprise）或 Claude Console 账号

### 安装方法

```bash
# macOS / Linux / WSL（推荐）
curl -fsSL https://claude.ai/install.sh | bash

# Homebrew (macOS)
brew install claude-code

# Windows PowerShell（原生安装）
irm https://claude.ai/install.ps1 | iex

# WinGet (Windows)
winget install claude-code

# npm 安装（兼容方式，需 Node.js 18+）
npm install -g @anthropic-ai/claude-code
```

### 安装后验证

```bash
claude --version
claude login
```

---

### Windows 安装详解与注意事项

<div class="callout callout-note" markdown="1">
**Windows 用户必读：** Claude Code 在 Windows 上有多种安装途径，每种都有不同的前置条件和适用场景。以下按**推荐优先级**排列。
</div>

自 2025 年 8 月起，Claude Code **不再强制依赖 WSL**，可以直接在 Windows 原生环境下运行。

**前置条件：**

- Windows 10 Build 1809+ 或 Windows 11
- [Git for Windows](https://git-scm.com/downloads/win)（**必需**，Claude Code 依赖 Git 命令）
- 4 GB+ RAM
- 管理员权限

**安装步骤：**

```powershell
# 1. 打开 PowerShell（以管理员身份运行）
# 2. 执行官方安装脚本
irm https://claude.ai/install.ps1 | iex

# 3. 验证安装
claude --version

# 4. 登录认证
claude login
```

```powershell
winget install claude-code
```

如果原生脚本出现问题，可以使用 npm 方式安装：

**前置条件：** [Node.js 18+](https://nodejs.org/)（推荐 LTS 版本）

```powershell
# 1. 确认 Node.js 版本
node -v
# 应该看到 v18.x.x 或更高

# 2. 全局安装 Claude Code
npm install -g @anthropic-ai/claude-code

# 3. 启动
claude
```

<div class="callout callout-warning" markdown="1">
**注意：** 不要使用 `sudo npm install -g`，这会导致权限问题和安全风险。如遇权限错误，参考官方的[权限故障排除文档](https://code.claude.com/docs/en/setup)。
</div>

如果你更习惯 Linux 环境，或需要与 Linux 工具链配合：

```powershell
# 1. 启用 WSL 2
wsl --install
# 重启后选择 Ubuntu 22.04 LTS

# 2. 在 WSL 中更新系统
sudo apt update && sudo apt upgrade -y

# 3. 安装 Git
sudo apt install git -y

# 4. 安装 nvm（Node 版本管理器）
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
source ~/.bashrc
nvm install --lts
nvm use --lts

# 5. 安装 Claude Code
npm install -g @anthropic-ai/claude-code

# 6. 验证
claude --version
```

社区提供的图形化安装工具，适合不熟悉命令行的用户：

1. 从 [GitHub Releases](https://github.com/InterWorks/CCInstaller/releases) 下载 `ClaudeCodeInstaller.exe`
2. **以管理员身份运行**
3. 选择安装目标（终端、VSCode 或两者）
4. 等待自动完成

<div class="callout callout-note" markdown="1">
**排障清单：**
- **环境变量冲突**：如果之前安装过 Cursor、NVM 等工具，检查 `Path` 环境变量中是否有冲突路径，清理后重试
- **Node.js 版本过低**：确保 `node -v` 显示 18+ 版本，建议使用 LTS 版本
- **网络问题**：Claude Code 需要联网运行，确保可以访问 Anthropic API（中国大陆用户可能需要配置代理）
- **Shell 选择**：支持 PowerShell 和 CMD，推荐使用 **PowerShell 7+** 或 **Windows Terminal**
- **首次启动白屏/报错**：尝试清理 `~/.claude` 目录后重新运行 `claude login`
- **Git 未安装**：原生模式下 Git for Windows 是硬性依赖，安装后需重启终端
- **磁盘空间**：Claude Code 及其依赖约占 150 MB，确保目标盘有足够空间
- **订阅要求**：需要 Claude Pro（$20/月）、Max、Teams 或 Enterprise 订阅，**不支持免费版**
</div>

---

## 三、核心 CLI 命令

### 基础命令速查

| 命令 | 用途 |
| --- | --- |
| `claude` | 启动交互式会话 |
| `claude -p "问题"` | 单次提问模式（pipe 模式） |
| `claude -c` | 继续上一次对话 |
| `claude -r prompt.md` | 从文件读取 prompt |
| `claude --agent name` | 使用指定 agent 启动 |

### 交互模式斜杠命令

| 命令 | 用途 |
| --- | --- |
| `/init` | 初始化项目（生成 [CLAUDE.md](http://claude.md/)） |
| `/compact` | 压缩上下文，节省 token |
| `/clear` | 清除上下文 |
| `/config` | 打开设置界面 |
| `/agents` | 管理子代理 |
| `/hooks` | 管理生命周期钩子 |
| `/permissions` | 管理工具权限 |
| `/context` | 查看当前上下文使用情况 |
| `/resume` | 恢复之前的会话 |

### 高级 CLI 标志

```bash
# 追加系统提示词（保留默认能力，最安全的方式）
claude --append-system-prompt "所有代码使用 TypeScript"

# 从文件加载追加提示词（适合团队共享）
claude --append-system-prompt-file ./team-rules.md

# 完全替换系统提示词（白板模式）
claude --system-prompt "你是一个 Python 专家"

# 禁用特定工具
claude --disallowedTools "Task(Explore)"

# 使用 CLI 定义的临时 agents（仅当前会话有效）
claude --agents '{"reviewer": {"description": "Code reviewer", "prompt": "Review code quality", "tools": ["Read", "Grep", "Glob"]}}'
```

---

## 四、权限系统与工作模式

<div class="callout callout-note" markdown="1">
**关键认知：** 权限模式不仅仅是安全功能，更是**工作流工具**——让你控制 Claude 拥有多大的自主权。
</div>

### 权限模式

| 模式 | 行为 | 适用场景 |
| --- | --- | --- |
| **default** | 标准权限检查，每次操作前提示确认 | 日常开发，推荐初学者 |
| **acceptEdits** | 自动接受文件编辑，其余需确认 | 信任度较高的代码修改 |
| **dontAsk** | 自动拒绝权限请求（已允许的除外） | 受限环境中运行 |
| **bypassPermissions** | 跳过所有权限检查 | ⚠️ 仅限沙箱/容器环境 |
| **plan** | 只读探索模式，不做任何修改 | 复杂任务前的规划阶段 |
| **delegate** | 仅限团队协调工具 | Agent Teams 的 Team Lead 专用 |

### 三种交互模式详解

Claude Code 有三种核心交互模式，对应不同的工作场景：

**1. 默认模式（Default Mode）**

- 启动方式：直接输入 `claude` 进入交互式会话
- Claude 接收你的指令后会分析、选择工具、执行操作
- 每次执行写入操作前会**请求确认**，你可以选择允许或拒绝
- 适合日常开发，安全可控

**2. 自动接受模式（Auto-accept Mode）**

- 启动方式：`claude --permission-mode acceptEdits` 或在会话中按 `Shift+Tab` 切换
- Claude 自动接受文件编辑操作，无需逐一确认
- 仍会对危险操作（如 Bash 命令）请求确认
- 适合你信任 Claude 的代码修改能力时使用，大幅提升效率

**3. 规划模式（Plan Mode）**

- 启动方式：`claude --permission-mode plan` 或在会话中按 `Shift+Tab` 切换
- Claude **只读**探索代码库，不做任何修改
- 先用 Read、Grep、Glob 等工具全面分析，制定详细方案
- 方案确认后再切回默认或自动模式执行
- 适合复杂任务前的规划阶段，避免盲目修改

<div class="callout callout-tip" markdown="1">
**模式切换技巧：** 在交互式会话中，随时按 `Shift+Tab` 可以在三种模式之间循环切换，无需退出重启。
</div>

### 跳过所有权限检测（dangerously-skip-permissions）

```bash
# ⚠️ 危险操作！跳过所有权限检查
claude --dangerously-skip-permissions
```

这个标志会让 Claude **完全跳过所有权限确认**，包括文件写入、Bash 命令执行等。相当于上面权限表中的 `bypassPermissions` 模式。

<div class="callout callout-warning" markdown="1">
**安全警告：**
- **仅在沙箱或容器环境中使用**，绝不要在本地开发机上使用
- 适用场景：Docker 容器内、CI/CD 流水线中、一次性虚拟机中
- Claude 将无限制地执行任何命令，包括 `rm -rf`、修改系统文件等
- 在生产环境或包含重要数据的机器上使用此标志**极其危险**
</div>

### Plan Mode 与 Think Mode

- **Plan Mode**：Claude 先用只读工具探索代码库，制定方案后再执行，适合复杂任务
- **Think Mode**：Claude 进行更深层次的推理和分析。可在 Skill 中通过加入 `ultrathink` 关键词启用扩展思考

---

## 五、Tools 工具系统（Agentic Loop 核心）

<div class="callout callout-note" markdown="1">
**核心概念：** Tools 是让 Claude Code 从"纯文本回复"变成"能行动的代理"的关键。没有 Tools，Claude 只能说话；有了 Tools，Claude 可以**行动**。
</div>

### Agentic Loop 工作原理

Claude Code 的核心是一个**代理循环（Agentic Loop）**：

5. Claude 接收你的输入
6. Claude 分析后决定使用哪个 Tool
7. Tool 执行并返回结果
8. 结果反馈给 Claude，Claude 决定下一步
9. 循环继续，直到任务完成

### 五大类内置 Tools

| 类别 | 工具名 | 功能 |
| --- | --- | --- |
| **读取类** | Read, Grep, Glob | 读取文件内容、搜索代码、匹配文件路径 |
| **写入类** | Write, Edit | 创建新文件、编辑已有文件 |
| **执行类** | Bash | 执行终端命令（npm test、git 操作等） |
| **委托类** | Task | 将任务委托给 Subagent（子代理） |
| **交互类** | AskUserQuestion | 向用户提问以获取澄清信息 |

### Tools 权限控制

在 `/permissions` 中可以精细控制每个 Tool 的允许/拒绝规则：

```json
{
  "permissions": {
    "allow": [
      "Read", "Grep", "Glob",
      "Bash(npm test *)",
      "Skill(review-pr *)"
    ],
    "deny": [
      "Write", "Edit",
      "Task(Explore)"
    ]
  }
}
```

- `Bash(npm test *)` — 只允许匹配模式的 Bash 命令
- `Task(worker)` — 只允许启动特定的 Subagent
- `Skill(deploy *)` — 只允许特定 Skill

---

## 六、上下文管理

这是高效使用 Claude Code 的**关键技能**。

### [CLAUDE.md](http://claude.md/) 项目记忆文件

`CLAUDE.md` 是 Claude Code 的**项目记忆文件**，放在项目根目录，每次启动时自动读取。

```markdown
# 项目上下文

## 技术栈
- 语言：Java 17
- 框架：Spring Boot 3.x
- 构建工具：Maven
- 数据库：MySQL 8.0

## 代码规范
- 使用 Google Java Style
- 所有 API 需要写单元测试
- 提交信息使用 Conventional Commits

## 项目结构
- src/main/java/com/example - 主代码
- src/test - 测试代码
- docs/ - 文档
```

### 上下文压缩与清除（/compact 与 /clear）

随着对话进行，上下文窗口会逐渐被填满。Claude Code 提供两种方式管理上下文：

`**/compact**` — 压缩上下文

- 将当前对话历史**智能压缩**，保留关键信息，丢弃冗余内容
- 压缩后 Claude 仍然记得重要的决策和上下文
- 当你看到上下文使用率较高时（用 `/context` 查看），主动执行 `/compact`
- 也可以带参数：`/compact 重点保留关于数据库设计的讨论`

`**/clear**` — 清除上下文

- 完全清空对话历史，从零开始
- 适合切换到完全不同的任务时使用
- 清除后 Claude 不会记得之前的任何对话内容（但 `CLAUDE.md` 仍会被加载）

<div class="callout callout-note" markdown="1">
**何时使用？**
- 上下文使用 > 60%：考虑 `/compact`
- 完全换一个任务：使用 `/clear`
- 长对话中 Claude 开始"忘记"之前的内容：使用 `/compact` 而不是重复信息
</div>

### @ 引用

在对话中使用 `@` 来引用文件和文件夹，给 Claude 提供精确上下文：

- `@src/main.py` — 引用特定文件
- `@src/` — 引用整个目录
- `@package.json` — 引用配置文件

### /init 命令

在项目中首次使用时运行 `/init`，Claude 会自动分析项目结构并生成 `CLAUDE.md`。

### 版本回滚（Rewind）

当 Claude 做出不满意的修改时，可以使用 **Rewind** 功能回退到之前的状态：

- 在交互模式中，Claude 完成一步操作后，你可以输入 `/undo` 撤销上一步工具调用的所有文件更改
- 也可以使用 **Checkpoint Diff** 查看每一步修改的具体内容，决定是否保留
- Claude Code 会在每次工具调用前自动创建 git stash 或 checkpoint，确保你可以安全回退
- 如果 Claude 走错了方向，直接告诉它"撤销刚才的修改"或"回到上一步"即可

<div class="callout callout-tip" markdown="1">
**最佳实践：** 在让 Claude 执行大批量修改前，先用 `git commit` 保存当前状态。这样即使 Rewind 不够用，你也可以通过 `git reset` 完全恢复。
</div>

### 图片处理（多模态输入）

Claude Code 支持处理图片，可以理解截图、设计稿、错误日志截图等视觉内容：

- **拖拽图片**：直接将图片文件拖入终端窗口
- **粘贴图片**：复制图片后在终端中粘贴（macOS 支持）
- **文件路径引用**：在对话中提及图片文件路径，Claude 会自动读取

常见用法：

```javascript
// 拖入一张报错截图
帮我分析这个错误截图，找出问题原因

// 拖入 UI 设计稿
根据这张设计稿实现页面布局

// 引用图片文件
分析 @screenshots/bug.png 中的错误信息
```

### 恢复历史会话（Resume）

使用 `/resume` 或 `claude -c` 可以恢复之前的对话：

- `**/resume**`：在交互模式中，列出最近的会话历史，选择要恢复的会话
- `**claude -c**`：启动时直接继续上一次的对话
- `**claude --resume <session-id>**`：恢复指定的历史会话

<div class="callout callout-note" markdown="1">
**提示：** 每次退出 Claude Code 时，会话会自动保存。下次可以随时恢复，上下文和对话历史都会保留。但请注意，恢复的会话不会自动感知文件在会话之间的变更，建议恢复后先让 Claude 检查一下当前代码状态。
</div>

---

## 七、Skills 技能系统

<div class="callout callout-note" markdown="1">
**Skills（技能）** 是 Claude Code 的能力扩展机制。创建一个 `SKILL.md` 文件，Claude 就能将其加入工具箱，在需要时自动使用，或者你通过 `/skill-name` 直接调用。
```markdown
---
name: fix-issue
description: 修复 GitHub Issue
disable-model-invocation: true
---
按照我们的编码规范修复 GitHub Issue $ARGUMENTS。
1. 读取 issue 描述
2. 理解需求
3. 实现修复
4. 编写测试
5. 创建 commit
```
</div>

### Skills 与 Slash Commands 的关系

**自定义 Slash Commands 已合并到 Skills 中。** `.claude/commands/review.md` 和 `.claude/skills/review/SKILL.md` 都会创建 `/review` 命令，效果完全相同。旧的 `.claude/commands/` 文件依然有效。Skills 额外支持目录文件、Frontmatter 配置和自动发现。

### 创建你的第一个 Skill

**目录结构：**

```javascript
my-skill/
├── SKILL.md           # 主指令文件（必需）
├── template.md        # Claude 可填充的模板
├── examples/
│   └── sample.md      # 示例输出
└── scripts/
    └── validate.sh    # Claude 可执行的脚本
```

**示例 **`**SKILL.md**`**：**

```markdown
---
name: explain-code
description: 用图表和类比解释代码。当用户问"这怎么工作的？"时使用。
---

解释代码时，始终包含：
1. **先打一个比方**：将代码比作日常事物
2. **画一个图**：用 ASCII 展示流程或结构
3. **走读代码**：逐步解释发生了什么
4. **指出陷阱**：常见的错误或误解是什么？
```

### Skill 存放位置

| 位置 | 路径 | 适用范围 |
| --- | --- | --- |
| 个人级别 | `~/.claude/skills/<<name>>/SKILL.md` | 你的所有项目 |
| 项目级别 | `.claude/skills/<name>/SKILL.md` | 仅当前项目 |
| 插件级别 | `<plugin>/skills/<name>/SKILL.md` | 插件启用的地方 |
| 企业级别 | 通过 managed settings 配置 | 组织内所有用户 |

### Frontmatter 配置详解

```yaml
---
name: deploy                       # Skill 名称，也是 /deploy 命令名
description: 部署应用到生产环境      # Claude 根据描述决定何时使用
disable-model-invocation: true     # true = 只能手动调用，Claude 不会自动触发
user-invocable: true               # false = 从菜单隐藏，只让 Claude 自动使用
allowed-tools: Read, Grep, Bash    # 限制 Skill 可用的工具
model: sonnet                      # 指定运行模型
context: fork                      # 在子代理中隔离运行
agent: Explore                     # context: fork 时使用的 agent 类型
---
```

### 参数传递

Skill 支持通过 `$ARGUMENTS` 接收参数：

调用：`/fix-issue 123`，Claude 会收到"修复 GitHub Issue 123"。

还可以按位置访问参数：`$ARGUMENTS[0]`、`$ARGUMENTS[1]` 或缩写 `$0`、`$1`。

### 动态上下文注入

使用  !`command`  语法在 Skill 内容发送给 Claude 之前运行 shell 命令，输出会替换占位符：

```markdown
---
name: pr-summary
description: 总结 PR 变更
context: fork
agent: Explore
---
## PR 上下文
- PR diff: !`gh pr diff`
- PR 评论: !`gh pr view --comments`
- 修改的文件: !`gh pr diff --name-only`

## 你的任务
总结这个 Pull Request...
```

### 在子代理中运行 Skill

添加 `context: fork` 使 Skill 在隔离的子代理中运行：

```markdown
---
name: deep-research
description: 深度研究某个主题
context: fork
agent: Explore
---
深入研究 $ARGUMENTS：
1. 用 Glob 和 Grep 查找相关文件
2. 阅读并分析代码
3. 总结发现，附带具体文件引用
```

---

## 八、Subagents 子代理系统

<div class="callout callout-note" markdown="1">
**Subagents（子代理）** 是具有独立上下文窗口、自定义系统提示词、特定工具权限的专门化 AI 助手。Claude 遇到匹配任务时会自动委托给合适的子代理。
</div>

### 内置子代理

| 名称 | 模型 | 能力 | 用途 |
| --- | --- | --- | --- |
| **Explore** | Haiku（快速） | 只读工具 | 文件搜索、代码分析、代码库探索 |
| **Plan** | 继承主会话 | 只读工具 | Plan Mode 下的代码库研究 |
| **general-purpose** | 继承主会话 | 所有工具 | 复杂的多步骤任务 |
| **Bash** | 继承主会话 | Bash | 在独立上下文中执行终端命令 |

### 创建自定义子代理

**方式一：交互式创建**

```javascript
/agents → Create new agent → 选择 User-level 或 Project-level → 按提示配置
```

**方式二：手动创建 Markdown 文件**

在 `.claude/agents/` 或 `~/.claude/agents/` 目录创建文件：

```markdown
---
name: code-reviewer
description: 代码审查专家。在代码修改后主动审查质量、安全性和可维护性。
tools: Read, Grep, Glob, Bash
model: sonnet
---

你是一位高级代码审查员，确保代码质量和安全性。

被调用时：
1. 运行 git diff 查看最近更改
2. 聚焦于修改过的文件
3. 立即开始审查

审查清单：
- 代码是否清晰可读
- 函数和变量命名是否合理
- 是否有重复代码
- 错误处理是否正确
- 是否暴露了密钥或 API Key
- 是否有输入验证
- 测试覆盖是否足够
- 性能问题是否考虑到

按优先级提供反馈：
- 🔴 严重问题（必须修复）
- 🟡 警告（应该修复）
- 🟢 建议（可以改进）
```

### Subagent Frontmatter 完整字段

| 字段 | 必需 | 说明 |
| --- | --- | --- |
| `name` | ✅ | 唯一标识，小写字母和连字符 |
| `description` | ✅ | Claude 根据此决定何时委托 |
| `tools` | ❌ | 可用工具列表。省略则继承所有 |
| `disallowedTools` | ❌ | 禁用的工具列表 |
| `model` | ❌ | `sonnet`、`opus`、`haiku` 或 `inherit` |
| `permissionMode` | ❌ | 权限模式覆盖 |
| `maxTurns` | ❌ | 最大执行轮次 |
| `skills` | ❌ | 启动时预加载的 Skill 列表 |
| `mcpServers` | ❌ | 可用的 MCP 服务器 |
| `hooks` | ❌ | 生命周期钩子 |
| `memory` | ❌ | 持久化记忆：`user`、`project` 或 `local` |

### 前台与后台运行

- **前台子代理**：阻塞主会话直到完成，权限提示会传递给你
- **后台子代理**：并发运行，你可以继续工作。按 **Ctrl+B** 可将正在运行的任务切到后台

### 并行研究（Parallel Research）

对于独立的调查任务，可以同时生成多个子代理并行工作：

```javascript
同时使用三个子代理并行研究 authentication、database 和 API 三个模块
```

每个子代理独立探索各自领域，然后 Claude 汇总发现。适合**任务互不依赖**的场景。

### 链式调用（Chaining）

对于多步工作流，让 Claude 按顺序使用子代理：

```javascript
先用 code-reviewer 子代理找出性能问题，再用 optimizer 子代理修复它们
```

### 持久化记忆（Persistent Memory）

`memory` 字段赋予子代理跨会话的持久目录，可积累知识：

```markdown
---
name: code-reviewer
description: 审查代码质量
memory: user
---
你是代码审查员。在审查过程中，将发现的模式、约定和反复出现的问题更新到你的 agent memory 中。
```

记忆范围：

- `user`：`~/.claude/agent-memory/<name>/`，跨所有项目
- `project`：`.claude/agent-memory/<name>/`，项目特定，可提交版本控制
- `local`：`.claude/agent-memory-local/<name>/`，项目特定，不提交

### 🔍 Skill vs Subagent 深度对比与选型指南

很多人觉得 Skill 和 Subagent "感觉都差不多"，这很正常——因为它们都能扩展 Claude Code 的能力。但它们的**设计哲学**完全不同：

<div class="callout callout-note" markdown="1">
**一句话区分：**
- **Skill** = 给 Claude **一份操作手册**，Claude 按手册自己干活
- **Subagent** = 给 Claude **一个专职助手**，Claude 把活派给助手干
</div>

**1. 上下文隔离 — 最关键的区别**

- **Skill**：默认在**主会话上下文**中运行。Claude 读取 [SKILL.md](http://skill.md/) 后，就像你给它发了一段指令，它在当前对话中执行。所有中间过程、工具调用结果都会**占用主会话的上下文窗口**。
- **Subagent**：始终在**独立上下文窗口**中运行。主会话只收到最终结果摘要，中间过程不会污染主上下文。

> 💡 这意味着：如果一个任务会产生大量中间输出（比如读取 50 个文件的审查结果），用 Subagent 可以保护主上下文不被撑爆。

**2. 运行身份**

- **Skill**：不是一个新的 AI 实例，而是一段**注入到当前 Claude 的指令**。Claude 还是那个 Claude，只是多了一份"参考手册"。
- **Subagent**：是一个**全新的 Claude 实例**，有自己的系统提示词、工具权限、甚至可以用不同的模型（如 Haiku 更快更省）。

**3. 调用方式**

- **Skill**：用户通过 `/skill-name` 手动调用，或 Claude 根据 description 自动匹配触发
- **Subagent**：Claude 通过内置的 `Task` 工具自动委托，用户通常无需手动干预

| 场景 | 为什么适合 Skill |
| --- | --- |
| 定义**代码风格规范** | 只是告诉 Claude "怎么写"，不需要隔离上下文 |
| 封装**提交流程**（lint → test → commit） | 是一个固定流程，Claude 按步骤执行即可 |
| 生成**模板代码** | 需要在当前上下文中直接创建文件 |
| 定义**审查清单** | 是一份检查标准，不需要独立的 AI 实例 |
| **快速小任务** | 启动成本低，不需要新建上下文窗口 |

| 场景 | 为什么适合 Subagent |
| --- | --- |
| **大规模代码审查**（读取大量文件） | 中间输出太多，需要隔离上下文，只返回摘要 |
| **并行研究**多个模块 | 多个独立实例同时工作，互不干扰 |
| 需要**不同权限**的任务 | Subagent 可以限制只用只读工具，更安全 |
| 需要**不同模型**的任务 | 简单探索用 Haiku（快且省），复杂推理用 Opus |
| **长时间运行**的后台任务 | 可以切到后台，不阻塞主会话 |

最强大的模式是**结合两者**——在 Skill 中设置 `context: fork`，让 Skill 的指令在子代理中执行：

```markdown
---
name: deep-review
description: 深度代码审查
context: fork          # 在子代理中运行 → 隔离上下文
agent: Explore         # 使用只读的 Explore 子代理
---
按以下清单审查代码：      # Skill 的指令 → 定义"怎么做"
1. 安全性检查
2. 性能分析
3. 可维护性评估
```

这样你同时获得了：

- Skill 的**流程定义能力**（清晰的审查清单）
- Subagent 的**上下文隔离**（不污染主会话）

<div class="callout callout-note" markdown="1">
**快速决策流程：**
10. 任务会产生大量中间输出吗？→ **是** → Subagent
11. 需要并行执行多个独立任务吗？→ **是** → Subagent
12. 需要限制工具权限或使用不同模型吗？→ **是** → Subagent
13. 只是定义一个工作流程或规范？→ **是** → Skill
14. 以上都不确定？→ 先用 Skill，不够再升级为 Subagent
</div>

---

## 九、Agent Teams 多代理团队

<div class="callout callout-note" markdown="1">
**Agent Teams（多代理团队）** 让多个 Claude Code 实例协同工作。一个会话作为 Team Lead 协调工作，Teammate 各自独立工作，并可以**互相通信**。这是 Subagents 的升级版——Subagents 只能汇报回主代理，而 Agent Teams 的成员之间可以直接交流。
</div>

### 启用 Agent Teams

Agent Teams 是实验性功能，默认关闭。在 `settings.json` 中启用：

```json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

### Agent Teams vs Subagents

| 特性 | Subagents | Agent Teams |
| --- | --- | --- |
| **上下文** | 独立窗口，结果返回给调用者 | 独立窗口，完全独立运行 |
| **通信** | 只能向主代理汇报 | 成员之间可直接通信 |
| **协调** | 主代理管理所有工作 | 共享任务列表，自我协调 |
| **适合** | 聚焦任务，只需结果 | 需要讨论和协作的复杂工作 |
| **Token 消耗** | 较低（结果摘要返回） | 较高（每个成员是独立实例） |

### 启动团队

直接用自然语言描述任务和团队结构：

```javascript
我正在设计一个追踪 TODO 注释的 CLI 工具。
创建一个 Agent Team 从不同角度探索：
一个 teammate 负责 UX，一个负责技术架构，一个扮演"魔鬼代言人"。
```

### 显示模式

- **In-process 模式**：所有 teammate 在主终端内运行，用 `Shift+Up/Down` 切换查看
- **Split pane 模式**：每个 teammate 独占一个窗格，需要 tmux 或 iTerm2

### 关键操作

- **Delegate Mode**：按 `Shift+Tab` 切换，让 Lead 只做协调不写代码
- **直接与 Teammate 对话**：`Shift+Up/Down` 选择后输入消息
- **Ctrl+T**：查看任务列表
- **要求计划审批**：`生成一个架构师 teammate，要求在修改代码前先审批计划`

### 最佳使用场景

15. **并行 Code Review**：安全、性能、测试覆盖各一个审查员
16. **竞争假设调试**：多个 teammate 各自验证不同假说并互相质疑
17. **跨层协调**：前端、后端、测试各一个 teammate
18. **新模块开发**：每个 teammate 负责独立的模块

---

## 十、Hooks 生命周期钩子

<div class="callout callout-note" markdown="1">
**Hooks（钩子）** 是在 Claude Code 生命周期特定时间点自动执行的 shell 命令或 LLM 提示。可以用来自动格式化代码、验证命令、发送通知等。
</div>

### Hook 事件一览

| 事件 | 触发时机 | 可阻断？ |
| --- | --- | --- |
| `SessionStart` | 会话开始或恢复 | ❌ |
| `UserPromptSubmit` | 用户提交提示词时（Claude 处理前） | ✅ |
| `PreToolUse` | 工具调用执行前 | ✅ |
| `PermissionRequest` | 权限对话框出现时 | ✅ |
| `PostToolUse` | 工具调用成功后 | ❌ |
| `PostToolUseFailure` | 工具调用失败后 | ❌ |
| `Notification` | Claude 发送通知时 | ❌ |
| `SubagentStart` | 子代理启动时 | ❌ |
| `SubagentStop` | 子代理完成时 | ✅ |
| `Stop` | Claude 完成响应时 | ✅ |
| `TeammateIdle` | Agent Team 成员即将空闲 | ✅ |
| `TaskCompleted` | 任务被标记完成时 | ✅ |
| `PreCompact` | 上下文压缩前 | ❌ |
| `SessionEnd` | 会话终止时 | ❌ |

### 三种 Hook 类型

19. **Command Hook**（`type: "command"`）：运行 shell 命令
20. **Prompt Hook**（`type: "prompt"`）：发送给 Claude 模型进行单轮评估
21. **Agent Hook**（`type: "agent"`）：生成子代理使用工具验证条件

### 配置示例

在 `settings.json` 中配置 Hook：

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "./scripts/run-linter.sh"
          }
        ]
      }
    ],
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": ".claude/hooks/block-rm.sh"
          }
        ]
      }
    ]
  }
}
```

### Exit Code 机制

- **Exit 0**：成功，允许操作继续。stdout 会被解析为 JSON 输出
- **Exit 2**：阻断错误。stderr 内容反馈给 Claude 作为错误信息
- **其他 Exit Code**：非阻断错误，stderr 显示在 verbose 模式中

### 实战示例：阻止危险的 Bash 命令

```bash
#!/bin/bash
# .claude/hooks/block-rm.sh
COMMAND=$(jq -r '.tool_input.command' < /dev/stdin)

if echo "$COMMAND" | grep -q 'rm -rf'; then
  jq -n '{
    hookSpecificOutput: {
      hookEventName: "PreToolUse",
      permissionDecision: "deny",
      permissionDecisionReason: "危险命令已被 Hook 阻止"
    }
  }'
else
  exit 0
fi
```

### 异步 Hook

添加 `"async": true`，Hook 在后台运行而不阻塞 Claude：

```json
{
  "type": "command",
  "command": "./scripts/run-tests-async.sh",
  "async": true
}
```

### 在 Subagent 和 Skill 中定义 Hook

Hook 也可以直接写在 Subagent 或 Skill 的 frontmatter 中，仅在该组件活跃期间生效：

```markdown
---
name: code-reviewer
description: 代码审查并自动 lint
hooks:
  PostToolUse:
    - matcher: "Edit|Write"
      hooks:
        - type: command
          command: "./scripts/run-linter.sh"
---
```

---

## 十一、MCP 集成

**MCP（Model Context Protocol）** 允许 Claude Code 连接外部工具和数据源。

### 安装 MCP Server（以 Figma 为例）

安装 MCP Server 非常简单，以 Figma MCP 为例：

```bash
# 在 Claude Code 交互模式中直接安装
/mcp add figma -- npx -y figma-developer-mcp --figma-api-key figd_YOUR_API_KEY

# 或者在 settings.json 中手动配置
```

手动配置方式，编辑 `~/.claude/settings.json`：

```json
{
  "mcpServers": {
    "figma": {
      "command": "npx",
      "args": ["-y", "figma-developer-mcp", "--figma-api-key", "YOUR_API_KEY"]
    }
  }
}
```

安装完成后，Claude Code 会自动发现 MCP 提供的新工具，你可以直接在对话中使用：

```javascript
// 使用 Figma MCP 还原设计稿
从 Figma 链接 https://figma.com/design/xxx 中获取设计稿，
用 React + Tailwind CSS 还原这个页面

// Claude 会自动调用 MCP 工具获取设计数据，然后生成代码
```

### 常见 MCP Server

- **Playwright MCP**：浏览器自动化，用于 E2E 测试和页面交互
- **Figma MCP**：从 Figma 设计稿直接生成代码
- **数据库 MCP**：查询和操作 PostgreSQL、MySQL 等数据库
- **GitHub MCP**：直接操作 GitHub API
- **Memory MCP**：为 Claude 提供持久化知识图谱

### 使用 MCP 工具还原设计稿

以 Figma 为例，完整的设计稿还原流程：

22. 安装 Figma MCP Server（如上）
23. 在对话中粘贴 Figma 设计链接
24. Claude 调用 `mcp__figma__get_file` 获取设计数据（布局、颜色、字体等）
25. Claude 根据设计数据自动生成前端代码
26. 你可以进一步要求 Claude 调整细节

### MCP 工具在 Hook 中的匹配

MCP 工具名遵循 `mcp__<server>__<tool>` 格式，例如：

- `mcp__memory__create_entities`
- `mcp__github__search_repositories`

可以用正则匹配：`mcp__memory__.*` 匹配所有 Memory 操作。

### Skill 与 SubAgent 的区别

很多人容易混淆 Skill 和 SubAgent，它们虽然都扩展了 Claude Code 的能力，但设计目的完全不同：

| 维度 | Skill（技能） | SubAgent（子代理） |
| --- | --- | --- |
| **本质** | 一份**指令文档**（[SKILL.md](http://skill.md/)），告诉 Claude 怎么做某件事 | 一个**独立的 AI 实例**，有自己的上下文窗口和系统提示词 |
| **上下文** | 在**主会话的上下文**中运行（除非设置 `context: fork`） | 始终拥有**独立的上下文窗口**，不污染主会话 |
| **调用方式** | 用户通过 `/skill-name` 手动调用，或 Claude 根据 description 自动触发 | Claude 通过 `Task` 工具自动委托，用户无需手动干预 |
| **配置复杂度** | 简单：一个 [SKILL.md](http://skill.md/) 文件即可 | 较复杂：需要定义 name、description、tools、model 等 |
| **适合场景** | 定义**工作流程和规范**：代码风格、提交规范、审查清单等 | **隔离执行**大任务：代码审查、并行研究、独立探索 |
| **类比** | 像一份**操作手册**，Claude 照着做 | 像一个**专职助手**，Claude 把任务交给它 |

<div class="callout callout-note" markdown="1">
**简单判断标准：**
- 如果你想让 Claude **按特定方式做事** → 用 Skill
- 如果你想让 Claude **把任务交给别人做** → 用 SubAgent
- 如果两者结合：创建一个带 `context: fork` 的 Skill，它会在子代理中运行
</div>

---

## 十二、Plugins 插件系统

**Plugins（插件）** 将 Skills、Subagents 和 Hooks 打包在一起，形成一个**可复用、可共享的功能包**。如果说 Skill 是单个工具，SubAgent 是单个助手，那 Plugin 就是一整个**工具箱**。

### 为什么需要 Plugin？

- **团队共享**：把团队的最佳实践封装成插件，新成员安装即用
- **社区分发**：发布到 npm 或 GitHub，供其他开发者使用
- **组合能力**：一个插件可以同时包含多个 Skills、SubAgents 和 Hooks，形成完整的工作流

### 插件结构

```javascript
my-plugin/
├── skills/                # 插件提供的 Skills
│   ├── review/
│   │   └── SKILL.md
│   └── deploy/
│       └── SKILL.md
├── agents/                # 插件提供的 SubAgents
│   └── reviewer.md
├── hooks/                 # 插件提供的 Hooks
│   └── hooks.json
├── scripts/               # 辅助脚本
│   └── setup.sh
├── PLUGIN.md              # 插件元信息（名称、描述、版本等）
└── README.md              # 使用文档
```

### 安装和使用

```bash
# 从 npm 安装插件
claude plugin add @team/code-quality

# 从 GitHub 安装
claude plugin add github:user/my-plugin

# 从本地路径安装
claude plugin add ./path/to/my-plugin

# 查看已安装插件
claude plugin list

# 移除插件
claude plugin remove @team/code-quality
```

### Plugin vs 单独的 Skill / SubAgent

| 维度 | 单独 Skill / SubAgent | Plugin |
| --- | --- | --- |
| **范围** | 单一功能 | 多个功能打包 |
| **分发** | 手动复制文件 | 通过 npm / GitHub 一键安装 |
| **版本管理** | 手动 | 支持语义化版本 |
| **适合** | 个人使用或简单需求 | 团队共享或完整工作流 |

---

## 十三、Git 与 CI/CD 集成

### Git 工作流

Claude Code 深度集成 Git，常见操作：

```javascript
# 查看改动并生成规范的 commit message
查看最近的改动，帮我写一个 Conventional Commits 格式的 commit message

# 创建 PR
创建一个 PR，描述这次改动的目的和影响

# 分析 PR
用 code-reviewer 子代理审查 PR #142
```

### GitHub Actions 集成

Claude Code 可以在 CI/CD 中运行，自动化：

- PR Review 和代码质量检查
- Issue 自动修复
- 部署验证
- 代码生成任务

### Git Worktrees 并行开发

多个 Claude Code 实例在同一个仓库上并行工作的推荐方式：

```bash
# 创建 worktree
git worktree add ../feature-auth -b feature/auth
git worktree add ../feature-api -b feature/api

# 在不同 worktree 中启动 Claude
cd ../feature-auth && claude
cd ../feature-api && claude
```

---

## 十四、常见工作流实战

### 场景 1：探索和理解新代码库

```javascript
帮我分析这个项目的整体架构和核心模块
@src/services/ 这些服务之间的调用关系是什么？
画一个模块依赖图
```

### 场景 2：Bug 修复（使用 Debugger 子代理）

```javascript
用户登录时偶尔出现 500 错误，帮我排查 @src/auth/login.ts
找到问题后直接修复，并添加对应的单元测试
```

### 场景 3：并行 Code Review（使用 Agent Teams）

```javascript
创建一个 Agent Team 来审查 PR #142，生成三个 reviewer：
- 一个聚焦安全影响
- 一个检查性能影响
- 一个验证测试覆盖
让他们各自审查后汇总发现。
```

### 场景 4：链式 Skill 调用

```javascript
/fix-issue 456
# → 自动读取 Issue → 分析需求 → 实现修复 → 编写测试 → 创建 commit
```

### 场景 5：后台并行研究

```javascript
同时在后台研究三个竞品的 API 设计：Stripe、Square、Adyen
# Claude 生成三个 Subagent 并行工作，结果汇总返回
```

---

## 十五、最佳实践与技巧

### ✅ 高效使用技巧

27. **先 **`**/init**`** 再开始工作**：让 Claude 了解你的项目
28. **善用 **`**@**`** 引用**：给 Claude 精确上下文，避免无关信息干扰
29. **复杂任务用 Plan Mode**：先规划再执行，减少返工
30. **维护好 **`**CLAUDE.md**`：跨会话保持上下文的关键
31. **封装重复工作流为 Skills**：一次编写，到处使用
32. **用 Subagents 隔离大输出**：测试结果、日志等放在子代理中，避免污染主上下文
33. **独立任务用并行 Subagents**：多个研究任务同时进行
34. **需要协作时用 Agent Teams**：成员之间需要交流讨论的复杂任务
35. **用 Hooks 自动化质量检查**：每次文件修改后自动 lint/format
36. **启用 Persistent Memory**：让子代理跨会话积累知识

### ⚠️ 注意事项

37. **初期使用 default 权限模式**：先熟悉 Claude 的行为再放开
38. **Agent Teams 消耗大量 Token**：只在确实需要并行协作时使用
39. **定期 **`**git diff**`** 审查修改**：保持对代码变更的掌控
40. **Subagents 不能嵌套**：子代理无法再生成子代理
41. **Hooks 在会话启动时快照**：修改后需要重启或通过 `/hooks` 审查

---

## 十六、学习资源

### 官方文档

- [Claude Code 快速入门](https://code.claude.com/docs/en/quickstart)
- [Agentic Loop 工作原理](https://code.claude.com/docs/en/how-claude-code-works)
- [Skills 技能系统](https://code.claude.com/docs/en/skills)
- [Subagents 子代理](https://code.claude.com/docs/en/sub-agents)
- [Agent Teams 多代理团队](https://code.claude.com/docs/en/agent-teams)
- [Hooks 钩子指南](https://code.claude.com/docs/en/hooks-guide)
- [Hooks 完整参考](https://code.claude.com/docs/en/hooks)
- [CLI 完整参考](https://code.claude.com/docs/en/cli-reference)
- [常见工作流](https://code.claude.com/docs/en/common-workflows)

### 推荐课程

- [Anthropic 官方课程：Claude Code in Action](https://anthropic.skilljar.com/claude-code-in-action)（1 小时视频 + 15 节课 + 证书）
- [DeepLearning.AI](http://deeplearning.ai/)[：Claude Code - A Highly Agentic Coding Assistant](https://www.deeplearning.ai/short-courses/claude-code-a-highly-agentic-coding-assistant/)（免费短课）

### 社区资源

- [Claude Code 学习路径实践指南（Medium）](https://medium.com/@dan.avila7/claude-code-learning-path-a-practical-guide-to-getting-started-fcc601550476)

### 建议学习计划

**第 1 周：基础**

42. 安装 Claude Code，完成官方 Quickstart
43. 在自己的项目中练习基础命令和 `@` 引用
44. 配置 `CLAUDE.md`，学习上下文管理
45. 理解权限系统，尝试 Plan Mode

**第 2 周：进阶**

46. 创建第一个 Skill（自定义 slash command）
47. 学习 Anthropic 官方课程
48. 尝试使用内置 Subagents（Explore、Plan）

**第 3 周：高级**

49. 创建自定义 Subagent
50. 配置第一个 Hook（如自动 lint）
51. 尝试 MCP 集成（如 Playwright）

**第 4 周：精通**

52. 尝试 Agent Teams 并行协作
53. 配置 Persistent Memory
54. 构建完整的自动化工作流（Skills + Hooks + Subagents）

---

> *"权限模式不仅仅是安全功能，更是工作流工具——让你控制 Claude 拥有多大的自主权。"*

---

## 十七、接入其他模型：CC-Switch 与多模型管理

<div class="callout callout-note" markdown="1">
**Claude Code 默认只连接 Anthropic API**，但通过 **CC-Switch** 等工具，你可以让 Claude Code 接入几乎任何兼容 OpenAI 接口的模型服务（如 DeepSeek、GPT、Gemini 等），实现多模型自由切换。
</div>

### CC-Switch 简介

[CC-Switch](https://github.com/farion1231/cc-switch) 是一个跨平台桌面工具（GitHub ⭐ 19K+），专门用于管理 Claude Code、Codex、OpenCode 和 Gemini CLI 等多种 AI 编程 CLI 工具的配置。

**核心功能：**

- **多供应商切换**：一键切换不同 AI 模型供应商（Anthropic、OpenAI 兼容接口、AiHubMix、CrazyRouter 等）
- **MCP 服务器管理**：统一管理 MCP Server 配置
- **Skills 安装/卸载**：可视化管理 Claude Code Skills
- **系统提示词管理**：管理多套系统提示词配置
- **API 速度测试**：可视化延迟，优化性能
- **配置备份与恢复**：自由实验不怕搞坏配置
- **故障自动切换**：网络波动时自动切换到备用供应商

### CC-Switch 安装

**macOS（推荐 Homebrew）：**

```bash
brew tap farion1231/ccswitch
brew install --cask cc-switch

# 更新
brew upgrade --cask cc-switch
```

**Windows：**

- 从 [GitHub Releases](https://github.com/farion1231/cc-switch/releases) 下载 `.msi` 安装包，双击安装
- 或下载 `.zip` 便携版，解压即用

**Linux：**

- 从 GitHub Releases 下载对应平台版本

### CC-Switch 使用流程

55. 启动 CC-Switch 桌面应用
56. 点击 **"Add Vendor"**（添加供应商）
57. 从预设列表选择供应商（如 AiHubMix、CrazyRouter 等），或自定义 OpenAI 兼容接口
58. 填入 **API Key** 和 **Base URL**
59. 返回主页，从供应商列表选择目标，点击 **"Start"** 即可使用

<div class="callout callout-tip" markdown="1">
**提示：** CC-Switch 还支持 `ccswitch://` 深度链接协议，可以通过 URL 快速切换配置。
</div>

### 整体架构：Claude Code + CC-Switch + CCR

如果你需要更复杂的模型路由能力，典型的三层架构是：

| 层级 | 组件 | 职责 |
| --- | --- | --- |
| **执行层** | Claude Code | AI 编程助手，执行代码任务 |
| **配置层** | CC-Switch | 管理多个供应商配置，一键切换 |
| **路由层** | Claude Code Router（CCR） | 协议转换 + 智能路由，将 Anthropic API 格式转为 OpenAI 兼容格式 |

这样就可以让 Claude Code 连接到几乎任何模型服务，包括：

- **DeepSeek**（国产大模型，性价比高）
- **GPT-4o / GPT-4.1**（OpenAI）
- **Gemini**（Google）
- **Qwen**（通义千问）
- 任何兼容 OpenAI Chat Completions API 的服务

### claude-provider：轻量级供应商切换

[claude-provider](https://github.com/nicekid1/claude-provider) 是另一个轻量级工具，支持以 **Claude Code 插件** 或 **CLI** 两种方式使用：

```bash
# 作为 Claude Code 插件安装
claude plugin add claude-provider

# 或作为 CLI 工具使用
npx claude-provider
```

**特点：**

- 将不同供应商配置保存为 **profiles**（配置文件）
- 一条命令即可切换 API Key、Base URL、模型
- 内置常见供应商预设，也支持自定义
- API Key 自动脱敏显示，更安全

---

## 十八、Claude Code 插件市场与推荐

<div class="callout callout-note" markdown="1">
Claude Code 拥有活跃的插件生态，社区已有 **481+** 个扩展插件，涵盖开发工程、工作流管理、AI 集成等多个方向。
</div>

### 插件市场与资源

| 平台 | 说明 | 链接 |
| --- | --- | --- |
| **Build with Claude** | 最大的插件市场，481+ 扩展 | [buildwithclaude.com](http://buildwithclaude.com/) |
| **Claude Marketplaces** | 插件目录聚合站，自动更新 | [claudemarketplaces.com](http://claudemarketplaces.com/) |
| **Awesome Claude Plugins** | GitHub 社区精选列表 | [GitHub](https://github.com/Chat2AnyLLM/awesome-claude-plugins) |
| **官方插件文档** | 创建和使用插件的完整指南 | [code.claude.com/docs/en/plugins](http://code.claude.com/docs/en/plugins) |

### 插件安装完整指南

插件通过**市场（Marketplace）**分发。安装插件前，你需要先添加市场，再从市场中安装插件——类似"先添加应用商店，再下载 App"。

<div class="callout callout-note" markdown="1">
**前置条件：**
- Claude Code **v1.0.33+**（运行 `claude --version` 检查）
- 如果 `/plugin` 命令无效，先升级：`brew upgrade claude-code` 或重新运行安装脚本
</div>

**官方 Anthropic 市场**（`claude-plugins-official`）在启动时**自动可用**，无需手动添加。

如需添加**第三方或社区市场**，使用 `/plugin marketplace add` 命令：

```bash
# 1. 从 GitHub 添加市场（owner/repo 格式）
/plugin marketplace add anthropics/claude-code

# 2. 从其他 Git 托管平台添加（GitLab、Bitbucket 等）
/plugin marketplace add https://gitlab.com/company/plugins.git

# 3. 添加指定分支或标签的市场
/plugin marketplace add https://gitlab.com/company/plugins.git#v1.0.0

# 4. 从本地路径添加（开发调试时常用）
/plugin marketplace add ./my-marketplace

# 5. 从远程 URL 添加
/plugin marketplace add https://example.com/marketplace.json
```

**方式一：交互式界面（推荐）**

```bash
# 打开插件管理器
/plugin
```

插件管理器有四个 Tab 页，按 `Tab` 键切换：

| Tab | 功能 |
| --- | --- |
| **Discover** | 浏览所有已添加市场中的可用插件 |
| **Installed** | 查看和管理已安装的插件 |
| **Marketplaces** | 添加、更新、移除市场 |
| **Errors** | 查看插件加载错误 |

在 **Discover** 页选择插件后按 Enter，选择安装范围即可。

**方式二：命令行直接安装**

```bash
# 从指定市场安装插件
/plugin install plugin-name@marketplace-name

# 示例：从官方市场安装 commit-commands
/plugin install commit-commands@claude-plugins-official

# 示例：从 Anthropic 演示市场安装
/plugin install commit-commands@anthropics-claude-code
```

**方式三：本地开发插件加载（**`**--plugin-dir**`**）**

```bash
# 加载本地插件目录（开发测试用，无需安装步骤）
claude --plugin-dir ./my-plugin

# 同时加载多个本地插件
claude --plugin-dir ./plugin-one --plugin-dir ./plugin-two
```

安装插件时可以选择**三种范围**：

| 范围 | 说明 | 适用场景 |
| --- | --- | --- |
| **User**（默认） | 安装到当前用户，所有项目可用 | 个人常用插件 |
| **Project** | 安装到项目级别（写入 `.claude/settings.json`），协作者共享 | 团队统一工具链 |
| **Local** | 安装到当前仓库的个人配置，不共享 | 个人实验、不想影响团队 |

```bash
# 通过 --scope 指定安装范围
claude plugin install formatter@your-org --scope project
```

安装完成后，插件的命令**立即可用**，通过插件名前缀调用：

```bash
# 插件命令格式：/插件名:命令名
/commit-commands:commit
/my-first-plugin:hello
/pr-review-toolkit:review
```

```bash
# 查看已安装插件
/plugin list

# 禁用插件（不卸载）
/plugin disable plugin-name@marketplace-name

# 重新启用插件
/plugin enable plugin-name@marketplace-name

# 更新指定插件
/plugin update plugin-name@marketplace-name

# 卸载插件
/plugin uninstall plugin-name@marketplace-name

# 列出所有已添加的市场
/plugin marketplace list

# 刷新市场的插件列表
/plugin marketplace update marketplace-name

# 移除市场（会同时卸载该市场的所有插件）
/plugin marketplace remove marketplace-name
```

<div class="callout callout-tip" markdown="1">
**小技巧：**
- `/plugin market` 可以简写代替 `/plugin marketplace`
- `/plugin marketplace rm` 可以简写代替 `/plugin marketplace remove`
- 官方市场默认开启**自动更新**，第三方市场需手动开启
- 如果插件 Skill 不显示，尝试清理缓存：`rm -rf ~/.claude/plugins/cache`，然后重启 Claude Code 并重新安装
</div>

```bash
# 代码智能（LSP）— 需要本地已安装对应语言服务器
/plugin install typescript-lsp@claude-plugins-official   # TypeScript
/plugin install pyright-lsp@claude-plugins-official      # Python
/plugin install gopls-lsp@claude-plugins-official        # Go
/plugin install jdtls-lsp@claude-plugins-official        # Java
/plugin install rust-analyzer-lsp@claude-plugins-official # Rust

# 外部集成
/plugin install github@claude-plugins-official           # GitHub
/plugin install slack@claude-plugins-official             # Slack
/plugin install notion@claude-plugins-official            # Notion
/plugin install figma@claude-plugins-official             # Figma
/plugin install linear@claude-plugins-official            # Linear
/plugin install sentry@claude-plugins-official            # Sentry

# 开发工作流
/plugin install commit-commands@claude-plugins-official   # Git 提交工作流
/plugin install pr-review-toolkit@claude-plugins-official # PR 审查
/plugin install plugin-dev@claude-plugins-official        # 插件开发工具
```

<div class="callout callout-warning" markdown="1">
**安全提醒：** Anthropic 不控制第三方插件中包含的 MCP Server、文件或其他软件，无法验证其安全性。安装前请先在插件主页确认其来源和可信度。
</div>

### 推荐插件分类

| 插件名 | 功能 |
| --- | --- |
| **ai-engineer** | AI/ML 功能集成、语言模型接入、推荐系统、智能自动化 |
| **vision-specialist** | 视觉模型、OCR、条码检测（支持 GPT-4V、Claude Vision、Mistral-OCR） |
| **claude-provider** | 一键切换 AI 供应商配置，支持 Profiles 管理 |

| 插件名 | 功能 |
| --- | --- |
| **code-quality** | 代码质量检查、Lint 集成、最佳实践检测 |
| **pr-reviewer** | 自动化 PR 审查，安全/性能/测试覆盖多维度 |
| **doc-generator** | 自动生成 API 文档、README、Changelog |

| 插件/工具 | 功能 |
| --- | --- |
| **CC-Switch** | 桌面端多供应商管理（支持 Claude Code / Codex / OpenCode / Gemini CLI） |
| **Claude Code Router（CCR）** | 协议转换 + 智能路由，接入任何 OpenAI 兼容模型 |
| **claude-provider** | CLI / 插件双模式，轻量级供应商切换 |

### 插件 vs Skills vs MCP

<div class="callout callout-note" markdown="1">
**三者关系：**
- **Skill**：单个可复用的指令/工作流（一个 `SKILL.md` 文件）
- **MCP Server**：连接外部工具和数据源的标准协议接口
- **Plugin**：将 Skills + Agents + Hooks + MCP 打包成一个**可安装、可分发**的功能包

简单说：Plugin 是 Skills、Agents 和 Hooks 的**打包分发格式**，MCP 是**连接外部能力的协议**，三者互补。
</div>

[Claude Code 跨项目开发：一个项目依赖另一个项目的实操方案](/2026/05/20/claude-code-cross-project/)

**gen-dev-doc：自动生成开发过程文档（Claude Code Custom Command）**

**Claude Code 教学视频 · 即梦AI 分镜脚本**

[Claude Code 分屏功能实战指南（macOS + Windows）](/2026/05/20/claude-code-split-screen/)

**接口复核插件方案（api-review-plugin）**

**接口复核插件 — 落地执行计划**

[Claude Code 前后端同步开发：标准执行流程与最佳实践](/2026/05/20/claude-code-fullstack-workflow/)
