---
layout: post
title: "Hermes Agent 深度解读与高阶使用指南"
date: 2026-05-20
categories: [tools]
mermaid: false
math: false
toc: true
---
<div class="callout callout-note" markdown="1">
**Hermes Agent** —— Nous Research 开源的自进化 AI Agent 框架。MIT 协议,GitHub 5 万+ Star。一句话:*它不是工具,是会和你一起成长的搭档。*
</div>

## 快速上手：常用命令 & Cheat Sheet（日常高频）

### 1) 安装 & 环境自检

```bash
hermes setup        # 初始化：选模型 Provider、填 Key、建 profile
hermes doctor       # 环境体检（依赖、网络、配置）
hermes --version    # 版本确认
```

### 2) 模型 / Provider 切换

```bash
hermes model        # 交互式切换 provider / 模型
```

### 3) 对话入口

```bash
hermes chat
```

### 4) Skills（沉淀可复用能力）

```bash
hermes skills list
hermes skills show <skill-name>
hermes skills edit <skill-name>
```

### 5) Gateway（把 Agent 接到消息入口）

```bash
hermes gateway --platform telegram --verbose
hermes gateway --platform feishu
hermes gateway --platform slack
```

### 6) Cron（定时任务 / 数字员工）

```bash
hermes cron add "0 9 * * 1" \
  --prompt "生成上周周报..." \
  --gateway feishu --channel ops-weekly

hermes cron list
hermes cron logs <task-id>
```

### 7) MCP（打通工具链）

```bash
# 作为 MCP Client：Hermes 调外部工具
hermes mcp add github --url https://api.githubcopilot.com/mcp/
hermes mcp add linear --url https://mcp.linear.app/mcp
hermes mcp list

# 作为 MCP Server：Claude Code / OpenClaw / Cursor 调 Hermes
hermes mcp serve --port 7654
```

### 8) Daemon（常驻运行 / 可观测）

```bash
hermes daemon start
hermes daemon status
hermes daemon logs -f
```

### 9) 记忆与成长报告（Honcho / Insights）

```bash
hermes memory status
hermes memory dump --user zhangchao > ~/hermes-profile.json
hermes insights --since 30d
```

## 一、Hermes Agent 是什么

Hermes Agent 是 Nous Research 于 2026 年 2 月开源的**自主型 AI Agent 框架**,定位 *"The agent that grows with you"*。

- **官网**:[hermes-agent.nousresearch.com](http://hermes-agent.nousresearch.com/)
- **GitHub**:[NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent)
- **协议**:MIT
- **语言**:Python 92% + 少量 TeX/Shell/JS
- **环境**:Python ≥ 3.11,Node ≥ 18
- **迭代**:2026 年 3 月起一个月连发 4 个大版本(v0.2 → v0.8)

它的架构可分三层:

1. **Agent 执行层** — 主对话循环 + 工具调用 + 上下文压缩(`run_agent.py`)
2. **平台能力层** — CLI、消息网关、SQLite 状态库(WAL + FTS5)、Cron 调度
3. **扩展生态层** — Skills、MCP、自定义工具、ACP 编辑器适配、RL 训练框架

本质是一个「**可运行的 Agent Operating Layer**」,而不是简单的 LLM 应用。

## 二、解决了什么问题

| 传统 Agent 痛点 | Hermes 的解法 |
| --- | --- |
| 用完即弃,不积累经验 | **闭环学习系统**:执行 ≥5 次工具调用的复杂任务后自动沉淀为 `SKILL.md`,并持续自我迭代 |
| 跨平台上下文丢失 | **SQLite + WAL + FTS5** 持久化 + Honcho 记忆,微信/Telegram/Slack 共享一份记忆 |
| 绑定单一模型厂商 | **模型无关**:18+ Provider(Nous Portal、OpenAI、Claude、DeepSeek、GLM、Kimi、Qwen、MiniMax、Ollama/vLLM),`hermes model` 一键切换 |
| AI 助手无法考核 | `/insights` 输出周报式成长报告,AI 员工首次拥有可观测的绩效数据 |

## 三、核心优势(四张王牌)

### 🧠 1. 自进化学习循环(最大亮点)

- 任务完成后 Agent 自问「这个模式值得记住吗?」,是则自动生成结构化 Skill 文档
- Skill 在后续使用中自我迭代,越用越准
- 用户实测:2 小时内 Agent 自动产出 3 份 Skill,重复性任务提速 **40%**
- **内置 RL 训练框架 + 轨迹导出**,模型本身也能自进化

### 🔌 2. 模型无关 + 多入口统一

- 一套代理,可跑在 **14+ 平台**:CLI / Telegram / Discord / Slack / WhatsApp / Signal / Email / Webhook / 飞书 …
- 支持 Claude Pro 订阅认证(OpenClaw 已被封)、OpenAI Codex OAuth、OpenRouter 200+ 模型、HuggingFace、本地 vLLM/Ollama

### 💾 3. 持久记忆 + 用户画像

- SQLite + FTS5 全文检索,跨会话永不失忆
- Honcho AI 原生记忆,自动构建对「你」的认知模型
- 上下文压缩,长对话不爆炸

### 🧩 4. 生态友好

- `hermes claw migrate` **一行命令从 OpenClaw 无缝迁移**
- MCP 原生支持,可让 Hermes 挂成 MCP Server 供 OpenClaw / Claude Code 调用
- ACP Adapter 可接入 VSCode / Cursor

## 四、与 OpenClaw 的本质差异

两者外表相似(都开源、多平台、多模型),但**设计哲学完全不同**。

| 维度 | OpenClaw 🦞 | Hermes Agent 👜 |
| --- | --- | --- |
| 架构理念 | Hub-and-Spoke 中心辐射:Gateway 统一调度 | Concentric Growth 同心成长:Agent 自身是内核 |
| 核心定位 | 多渠道个人 AI 工作台 / 消息中枢 | 有生命力的自进化数字分身 |
| Skill 来源 | ClawHub 市场下载(13,700+,但 41% 有安全漏洞) | Agent 自动生成 + 自迭代 `SKILL.md` |
| 记忆 | 会话级为主 | 跨会话持久记忆 + 用户画像 + Honcho |
| 自主度 | 人定规则、Agent 执行([SOUL.md](http://soul.md/) / [AGENTS.md](http://agents.md/)) | Agent 自主迭代闭环 |
| Claude Pro 订阅 | ❌ 已被 Anthropic 封杀 | ✅ 仍可用 Claude Code OAuth |
| 无服务器 / 闲置零成本 | ❌ | ✅ Daytona / Modal |
| RL 训练框架 | ❌ | ✅ 轨迹导出 + 强化学习 |
| 多 Agent 团队协作 | ✅ 成熟(配置驱动) | ⚠️ 单 Agent 为主,DM pairing 做协作 |
| 安全管控 | ✅ 显式授权、命令批准 | ⚠️ 开箱即用,透明度取决于用户理解 |
| 适合人群 | IDE 重度用户 / 团队协作 / 已有成熟 Skill 体系 | 全栈 / 运维 / 长期 AI 伙伴 / RL 数据生成 |

<div class="callout callout-tip" markdown="1">
**一句话**:OpenClaw 是**瑞士军刀**(开箱即用、功能齐全),Hermes 是**会长大的搭档**(今天比昨天更懂你)。两者**不是替代而是互补**——可让 Hermes 做 MCP Server,OpenClaw 做前端编排。
</div>

## 五、安装与配置(macOS 实战)

### 前置条件

- Python ≥ 3.11(建议 `uv` 管理)
- Node.js ≥ 18
- Homebrew、git
- (可选)Claude Code 已登录 Pro 订阅

### 方式 A:一键脚本(推荐新手)

```bash
curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash
hermes setup      # 向导式:选 Provider + 输 API Key + 建 profile
hermes doctor     # 环境体检
```

### 方式 B:源码手动安装(高阶 / 贡献者)

```bash
git clone https://github.com/NousResearch/hermes-agent.git
cd hermes-agent
uv venv && source .venv/bin/activate
uv pip install -e ".[all]"     # 含 messaging/cron/voice/mcp/honcho
hermes --version
```

可选 extras:`[messaging]` `[cron]` `[voice]` `[mcp]` `[honcho]` —— 按需裁剪。

### 模型配置(挑一种即可)

```bash
hermes model          # 交互式切换
# 1) Nous Portal      订阅制零配置,OAuth 登录
# 2) OpenAI Codex     ChatGPT 账号 OAuth 设备码
# 3) Anthropic        Claude Code 认证(复用 Pro 订阅!)
# 4) OpenRouter       填 key 即可用 200+ 模型,含免费额度
# 5) DeepSeek         export DEEPSEEK_API_KEY=sk-xxx
# 6) 本地/自定义      OPENAI_BASE_URL=http://localhost:8000/v1
```

### 从 OpenClaw 迁移

```bash
hermes claw migrate    # 一行命令,数据 / Skill / 配置无缝迁移
```

## 五-B、安装与配置（Windows 实战）

### 前置条件

- Windows 10 / 11（推荐 64-bit）
- Python ≥ 3.11（建议用 `winget` 或官方安装包，勾选 **Add to PATH**）
- Node.js ≥ 18（[nodejs.org](http://nodejs.org/) LTS 安装包）
- Git for Windows（[git-scm.com](http://git-scm.com/)）
- （推荐）Windows Terminal + PowerShell 7
- （可选）WSL2 —— 如果你想要与 macOS/Linux 完全一致的体验

### 方式 A：PowerShell 一键安装（推荐新手）

```powershell
# 以管理员身份打开 PowerShell
irm https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.ps1 | iex
hermes setup      # 向导式：选 Provider + 输 API Key + 建 profile
hermes doctor     # 环境体检
```

> 如果没有官方 PowerShell 脚本，可改用 pip 安装（见方式 B）。

### 方式 B：pip / uv 手动安装

```powershell
# 1. 安装 uv（Python 包管理加速器）
pip install uv

# 2. 克隆仓库
git clone https://github.com/NousResearch/hermes-agent.git
cd hermes-agent

# 3. 创建虚拟环境并激活
uv venv
.\.venv\Scripts\Activate.ps1

# 4. 安装（含全部可选依赖）
uv pip install -e ".[all]"
hermes --version
```

### 方式 C：WSL2（Linux 子系统，与 macOS 步骤一致）

```powershell
# 1. 启用 WSL2（首次需要）
wsl --install
# 重启后默认安装 Ubuntu

# 2. 在 WSL2 Ubuntu 中
sudo apt update && sudo apt install python3.11 python3.11-venv nodejs npm git -y
curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash
hermes setup
```

<div class="callout callout-tip" markdown="1">
**WSL2 vs 原生 Windows**：如果你需要 Cron 定时任务、Daemon 常驻、消息网关等 Linux 特性，WSL2 体验最佳。纯 CLI 对话 + MCP 集成用原生 PowerShell 即可。
</div>

### 模型配置（与 macOS 一致）

```powershell
hermes model          # 交互式切换，支持所有 18+ Provider
# 环境变量示例（PowerShell）：
$env:DEEPSEEK_API_KEY = "sk-xxx"
$env:OPENAI_BASE_URL = "http://localhost:8000/v1"   # 本地模型
```

### Windows 特有注意事项

| 问题 | 解决方案 |
| --- | --- |
| PowerShell 执行策略阻止脚本 | `Set-ExecutionPolicy RemoteSigned -Scope CurrentUser` |
| 路径含中文/空格导致报错 | 项目目录放在纯英文路径下，如 `C:\Dev\hermes-agent` |
| `hermes` 命令找不到 | 确认 `.venv\Scripts` 在 PATH 中，或用 `python -m hermes` |
| SQLite WAL 文件锁冲突 | 避免多进程同时访问同一 DB 文件；WSL2 不受影响 |
| MCP Server 端口被防火墙拦截 | `netsh advfirewall firewall add rule name="Hermes MCP" dir=in action=allow protocol=TCP localport=7654` |
| Gateway（Telegram/飞书等）常驻 | 用 `nssm` 注册为 Windows 服务，或在 WSL2 中用 `systemd` |

### 从 OpenClaw 迁移（Windows）

```powershell
hermes claw migrate    # 与 macOS 完全一致
```

## 六、高阶使用指南

### 1. 消息网关 —— 手机随时指挥

5 刀 VPS 挂一台,24h 在线:

```bash
hermes gateway --platform telegram --verbose
# .env:
# TELEGRAM_BOT_TOKEN=xxx
# TELEGRAM_ALLOWED_USER_IDS=123456789
hermes gateway --platform feishu
hermes gateway --platform slack
```

典型玩法:用微信 / 飞书让 Agent 监控 Kafka 告警、整理 Doris 慢查询日志、自动写值班周报。

### 2. Cron 定时任务 —— 变身「数字员工」

```bash
hermes cron add "0 9 * * 1" \
  --prompt "拉取上周异常锁车监控 Doris 数据,输出 Markdown 周报并推送到飞书" \
  --gateway feishu --channel ops-weekly
hermes cron list
hermes cron logs <task-id>
```

### 3. Skills 自动沉淀

```bash
hermes chat
> 帮我按照 Doris Stream Load 异步写入方案的 Starter 结构,生成一个 Kafka→Flink→Doris 的 Sink 模板

# 完成后 Hermes 会自动生成:
# ~/.hermes/skills/doris-stream-load-sink/SKILL.md
hermes skills list
hermes skills show doris-stream-load-sink
hermes skills edit doris-stream-load-sink
```

Skill 是**可版本化**的 Markdown,适合放进团队知识仓库沉淀。

### 4. MCP 集成 —— 与现有工具链打通

```bash
# 作为 MCP 客户端,接入 Figma / Linear / GitHub 等
hermes mcp add github --url https://api.githubcopilot.com/mcp/
hermes mcp add linear --url https://mcp.linear.app/mcp
hermes mcp list

# 作为 MCP Server,让 Claude Code / OpenClaw / Cursor 调用 Hermes
hermes mcp serve --port 7654
```

**双向打通**:你可以在 OpenClaw 里通过 ACP/MCP 调用 Hermes 的自学习能力,反之亦然。

### 5. Honcho 长期记忆 —— 用户画像

```bash
uv pip install -e ".[honcho]"
export HONCHO_API_KEY=xxx
hermes memory status
hermes memory dump --user zhangchao > ~/hermes-profile.json
hermes insights --since 30d     # 看 Agent 这一月学了什么
```

### 6. 批处理与轨迹导出 —— RL 数据生成

```bash
hermes batch run --config batch.yaml \
  --env environments/iot-ops \
  --export trajectories/
# 导出 JSONL 轨迹,可直接喂 LoRA / GRPO 训练
```

### 7. ACP 编辑器适配 —— Cursor / VSCode

```bash
hermes acp serve
# VSCode 安装 ACP 扩展,指向 localhost:7777
```

等价于在 IDE 里获得一个带记忆和自学习能力的 Claude Code。

### 8. Daemon 与生产部署

```bash
hermes daemon start
hermes daemon status
hermes daemon logs -f

# 关键环境变量
export HERMES_MAX_CONCURRENT_TASKS=20
export HERMES_AGENT_TIMEOUT=2h
export HERMES_WORKSPACES_ROOT=~/hermes_workspaces
```

### 9. Slash Commands 速查

| 命令 | 作用 |
| --- | --- |
| `/skills` | 查看/应用已沉淀技能 |
| `/insights` | Agent 成长周报 |
| `/tools` | 查看本会话工具集 |
| `/export` | 导出当前会话轨迹 |

## 七、落地建议(结合车联网 + AI 转型路线)

<div class="callout callout-note" markdown="1">
**短期(本周)**:用 `hermes claw migrate` 从 OpenClaw 迁过来,保留 OpenClaw 做 IDE 侧重度编码,Hermes 做长期跑腿 + 飞书触达。

**中期(1 个月)**:把「异常锁车监控」值班 SOP、「Doris Stream Load 写入」排障手册交给 Hermes,让它自动生成 `SKILL.md`,沉淀到 `dst-ai-coding` 仓库。

**高阶(2–3 个月)**:配合「智能股票助手」项目,用 `hermes batch run` 导出多 Agent 辩论轨迹,喂给 DeepSeek / Qwen 做 LoRA 微调——对应你 AI 应用开发工程师路线图里的「微调与工程化」里程碑,面试时能讲出「自建 Agent 框架 + RL 轨迹 + 自进化 Skill」完整故事。
</div>

## 参考链接

- GitHub: [https://github.com/NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent)
- 官网: [https://hermes-agent.nousresearch.com/](https://hermes-agent.nousresearch.com/)
- 菜鸟教程: [https://www.runoob.com/ai-agent/hermes-agent.html](https://www.runoob.com/ai-agent/hermes-agent.html)
- vs OpenClaw 深度对比: [https://www.cnblogs.com/jucunqi/p/19847010](https://www.cnblogs.com/jucunqi/p/19847010)
- AWS 投研助手实战: [https://aws.amazon.com/cn/blogs/china/building-investment-research-assistant-with-hermes-agent-on-aws/](https://aws.amazon.com/cn/blogs/china/building-investment-research-assistant-with-hermes-agent-on-aws/)
