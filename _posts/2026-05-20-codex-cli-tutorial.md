---
layout: post
title: "Codex CLI 完整教程：安装 · 使用 · 更新 · 工具集成"
date: 2026-05-20
categories: [tools]
mermaid: false
math: false
toc: true
---
> Codex CLI 是 OpenAI 开源的本地编码 Agent，基于 Rust 构建，可在终端中读取、修改和运行代码。包含在 ChatGPT Plus / Pro / Business / Edu / Enterprise 计划中。

> 

> 🔗 官方文档：[https://developers.openai.com/codex/cli/](https://developers.openai.com/codex/cli/)

> 🔗 GitHub 仓库：[https://github.com/openai/codex](https://github.com/openai/codex)

---

## 一、安装教程

### 前置要求

- **操作系统**：macOS / Linux 原生支持；Windows 支持为实验性（建议使用 WSL）
- **Node.js**：需要 npm（推荐 Node 18+）或 Homebrew
- **账号**：ChatGPT Plus / Pro / Business / Edu / Enterprise 计划，或 OpenAI API Key

### 方式一：npm 安装（推荐）

```bash
npm i -g @openai/codex
```

### 方式二：Homebrew 安装（macOS）

```bash
brew install --cask codex
```

### 首次登录

安装后运行 `codex`，首次会提示登录：

- **ChatGPT 账号登录**：浏览器自动跳转认证
- **API Key 登录**：

```bash
codex login --api-key "your-openai-api-key"
```

### Windows 用户

建议在 **WSL (Windows Subsystem for Linux)** 中使用，参考官方 [Windows 安装指南](https://developers.openai.com/codex/cli/windows/)。

---

## 二、更新教程

Codex CLI 更新频率较高，建议定期升级以获取最新模型和功能。

### npm 更新

```bash
npm i -g @openai/codex@latest
```

### Homebrew 更新

```bash
brew upgrade --cask codex
```

### 查看更新日志

官方 Changelog：[https://developers.openai.com/codex/changelog/](https://developers.openai.com/codex/changelog/)

---

## 三、使用教程

### 3.1 交互模式（最常用）

启动全屏终端 UI，可实时对话、审查代码改动：

```bash
codex
```

也可以带上初始 prompt 直接启动：

```bash
codex "Explain this codebase to me"
```

### 3.2 常用操作

| **操作** | **说明** |
| --- | --- |
| `codex` | 启动交互式会话 |
| `codex "prompt"` | 单次提问模式，快速获取答案后退出 |
| `codex exec "prompt"` | 非交互式执行，适合脚本自动化 |
| `codex --model gpt-5.3-codex` | 指定模型启动 |
| `codex -i screenshot.png "Explain"` | 附带图片输入（支持截图/设计稿） |
| `codex --cd /path/to/project` | 指定工作目录 |
| `codex resume` | 恢复之前的会话 |
| `codex resume --last` | 直接恢复最近一次会话 |

### 3.3 会话内斜杠命令

| **命令** | **功能** |
| --- | --- |
| `/model` | 切换模型 / 调整推理级别 |
| `/review` | 本地代码审查（对比分支、未提交改动、指定 commit） |
| `/permissions` | 切换审批模式（Auto / Read-only / Full Access） |
| `/clear` | 清屏并开始新对话 |
| `/copy` | 复制最近一次 Codex 输出 |
| `/exit` 或 `Ctrl+C` | 退出会话 |
| `/theme` | 预览和设置颜色主题 |
| `/fork` | 从某个历史消息处分叉会话 |

### 3.4 审批模式

| **模式** | **权限** | **适用场景** |
| --- | --- | --- |
| **Auto**（默认） | 可读写工作目录内文件、运行命令；目录外操作需确认 | 日常开发 |
| **Read-only** | 只读，不修改文件、不运行命令 | 代码理解/咨询 |
| **Full Access** | 完全权限，包括网络访问，不需确认 | 受信任环境下的自动化 |
| **YOLO 模式** | `--yolo` 跳过所有审批和沙箱 | ⚠️ 仅在安全隔离环境中使用 |

### 3.5 快捷键

- `@` — 在输入框中模糊搜索文件路径
- `Enter`（运行中按）— 向当前轮次注入新指令
- `Tab`（运行中按）— 排队一个后续 prompt
- `!ls` — 前缀 `!` 执行本地 shell 命令
- `Esc × 2` — 编辑上一条用户消息
- `Ctrl+G` — 打开外部编辑器写长 prompt

### 3.6 脚本化 / 自动化

```bash
# 非交互式执行
codex exec "fix the CI failure"

# JSON 输出（便于管道处理）
codex exec --json "list all TODO comments"

# 恢复上次会话并追加指令
codex exec resume --last "Implement the plan"
```

### 3.7 配置文件

配置文件位于 `~/.codex/config.toml`，可持久化设置：

```toml
# 默认模型
model = "gpt-5.3-codex"

# Web 搜索设置（live / cached / disabled）
web_search = "cached"

# 代码审查专用模型
review_model = "gpt-5.3-codex"
```

### 3.8 [AGENTS.md](http://agents.md/) 与规则文件

在项目根目录创建 `AGENTS.md` 或 `.codex/` 目录下放置规则文件，可自定义 Codex 在该项目中的行为（类似 Claude Code 的 `CLAUDE.md`）。

---

## 四、可集成的好用工具

### 4.1 MCP Server（Model Context Protocol）

Codex 原生支持 MCP，可以在 `~/.codex/config.toml` 中配置或通过 `codex mcp` 命令管理。

<div class="callout callout-note" markdown="1">
**📝 🔌**

**MCP 是 Codex 最强大的扩展机制** — 通过它你可以把任何第三方工具接入 Codex，让它像内置工具一样调用。
</div>

**热门 MCP Server 推荐：**

| **工具** | **用途** | **链接 / 安装** |
| --- | --- | --- |
| **飞书 Lark MCP** | 读写飞书多维表格（Bitable），适合你的接口复核插件场景 | `@larksuiteoapi/lark-mcp` |
| **JetBrains MCP** | 连接 IntelliJ IDEA，让 Codex 操作 IDE 功能 | JetBrains 官方插件 |
| **Figma MCP** | 读取 Figma 设计稿，直接生成对应代码 | 社区维护 |
| **Browser MCP** | 控制浏览器进行自动化测试或数据抓取 | Playwright MCP 等 |
| **Context7** | 自动查阅框架/库文档，避免 AI 瞎猜 API | [https://context7.com](https://context7.com/) |
| **Supabase MCP** | 直接操作 Supabase 数据库/Auth/Storage | Supabase 官方 |

**配置示例（config.toml）：**

```toml
[mcp_servers.lark]
command = "npx"
args = ["-y", "@larksuiteoapi/lark-mcp"]

[mcp_servers.context7]
command = "npx"
args = ["-y", "@context7/mcp-server"]
```

**MCP 管理命令：**

```bash
codex mcp list          # 查看已配置的 MCP Server
codex mcp add <name>    # 添加 MCP Server
codex mcp login <name>  # OAuth 登录（支持 OAuth 的 Server）
```

### 4.2 Skills（技能包）

<div class="callout callout-note" markdown="1">
**📝 🧩**

Skills 是 Codex 的**可复用任务配方**，用 `SKILL.md` + 脚本封装特定工作流。支持 CLI、IDE、App 三端。
</div>

- 基于开放标准 [https://agentskills.io](https://agentskills.io/)
- 一个 Skill = 一个目录 + `SKILL.md`（包含 name、description、指令）+ 可选脚本
- Codex 按需加载：先读元数据，决定使用时才加载完整指令
- 你的 **api-review-check / api-review-test** 就是 Skill 的典型应用

### 4.3 Multi-Agent（多 Agent 协作）

```bash
codex --enable multi_agent
```

- 实验性功能，可并行多个 Agent 分工协作
- 适合大型任务拆分（如前后端同时改动）

### 4.4 Codex 作为 MCP Server

Codex 自身也可以**被其他 Agent 调用**：

```bash
# 在 Claude Code 中使用 Codex
claude mcp add codex-cli -- npx -y codex-mcp-server
```

这样 Claude Code 就能调用 Codex 的能力，实现**多引擎协作**。

### 4.5 其他好用工具 & 集成

| **工具** | **类型** | **说明** |
| --- | --- | --- |
| **GitHub Action** | CI/CD 集成 | 在 GitHub Actions 中运行 Codex 自动修复 / 审查 |
| **Codex SDK** | 编程接口 | 将 Codex 能力嵌入自己的应用或脚本 |
| **Codex App Server** | 服务端 | 托管 Codex 实例，供团队共享使用 |
| **SuperAgent MCP** | 多 Agent 编排 | 协调 Codex + Gemini CLI + Claude 并行执行，支持 16 种专业角色 |
| **Slack 集成** | 团队协作 | 通过 Slack 触发 Codex 任务 |
| **Linear 集成** | 项目管理 | 自动关联 Linear issue 与代码改动 |

---

## 五、与 Claude Code 的对比（快速参考）

| **维度** | **Codex CLI** | **Claude Code** |
| --- | --- | --- |
| 厂商 | OpenAI | Anthropic |
| 语言 | Rust（更快） | TypeScript |
| 默认模型 | GPT-5.3-Codex | Claude Sonnet / Opus |
| MCP 支持 | ✅ 原生支持 | ✅ 原生支持 |
| Skills / 技能包 | ✅ [SKILL.md](http://skill.md/)  • [agentskills.io](http://agentskills.io/) | ✅ Custom Commands / Subagent |
| 代码审查 | ✅ `/review` 内置 | 需手动 prompt |
| Web 搜索 | ✅ 内置（cached/live） | ❌ 需 MCP |
| 云端任务 | ✅ Codex Cloud | ❌ |
| 多 Agent | ✅ 实验性 | ✅ Subagent + Task |
| 会话恢复 | ✅ `codex resume` | ✅ `claude --continue` |

<div class="callout callout-tip" markdown="1">
**💡 💡**

**实战建议**：Codex CLI 和 Claude Code 并不冲突，可以通过 MCP 互相调用。根据任务选择最擅长的引擎，或者用 SuperAgent 等工具做多 Agent 编排。
</div>

---

## 六、快速上手 Cheat Sheet

```bash
# 安装
npm i -g @openai/codex

# 启动交互
codex

# 带 prompt 启动
codex "帮我重构这个 Service 类"

# 指定目录
codex --cd ~/projects/my-app

# 更新到最新版
npm i -g @openai/codex@latest

# 脚本化执行
codex exec "fix all lint errors"

# 代码审查
# 交互模式中输入 /review

# 添加 MCP Server
codex mcp add context7 -- npx -y @context7/mcp-server

# 查看功能开关
codex features list

# Shell 补全
eval "$(codex completion zsh)"
```
