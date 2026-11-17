---
layout: post
title: "Claude Code 分屏功能实战指南（macOS + Windows）"
date: 2026-05-20
categories: [tools]
mermaid: false
math: false
toc: true
---
高效利用 Claude Code 的分屏（Split Pane）能力，同时运行多个 AI 编程会话，实现真正的**并行开发**。

---

## 一、什么是 Claude Code 分屏？

Claude Code 的「分屏」是指在终端中**同时打开多个 Claude Code 会话窗格**，让多个 AI 代理各自独立工作、互不干扰，你可以实时观察每个会话的输出并随时介入。

分屏能力主要通过两种方式实现：

1. **Agent Teams 的 Split Pane 模式**（官方内置）—— 多个 Claude Code 实例组成团队，自动协作
2. **终端复用器手动分屏**（tmux / iTerm2）—— 手动开多个独立会话，各做各的

<div class="callout callout-tip" markdown="1">
**核心价值：** 从「一次只做一件事」变成「同时推进多件事」，大幅提升开发效率。比如一个窗格写功能代码，一个窗格写测试，一个窗格做代码审查——全部并行。
</div>

---

## 二、分屏的核心优势

### ⚡ 效率倍增

| 场景 | 单会话 | 分屏多会话 |
| --- | --- | --- |
| 功能开发 + 测试编写 | 串行，等一个做完再做下一个 | **并行**，两个窗格同时进行 |
| 多模块重构 | 逐个模块处理，耗时长 | 每个窗格负责一个模块，**同步推进** |
| 代码审查 | 一个视角、一次审查 | 安全 / 性能 / 测试覆盖**多维度同步审查** |
| Bug 排查 | 逐一验证假设 | 多个假设**同时验证**，快速收敛 |

### 🎯 其他关键优势

- **上下文隔离**：每个窗格是独立的 Claude Code 实例，互不污染上下文窗口
- **实时可视**：一眼看到所有会话的进度，不用来回切换
- **灵活介入**：点击任意窗格即可直接与该会话对话，给出新指令
- **团队协作能力**：Agent Teams 模式下，多个代理可以互相通信、共享任务列表
- **降低返工风险**：Plan Mode + 分屏可以先规划再执行，多角度验证方案

---

## 三、方式一：Agent Teams Split Pane 模式（推荐）

这是 Claude Code **官方内置**的分屏协作方案，让多个 Claude Code 实例组成团队，自动分工、互相沟通。

### 3.1 前置条件

- Claude Code 已安装并可正常使用
- Claude Max 订阅（多实例 Token 消耗较大，Pro 计划容易触发限流）
- **macOS**：需要安装 `tmux` 或使用 iTerm2
- **Windows**：需要使用 WSL 2 + tmux（Windows 原生终端暂不支持 Split Pane 模式）

### 3.2 安装 tmux

<div class="callout callout-note" markdown="1">
**macOS 安装 tmux：**
```javascript
brew install tmux
```

验证安装：

```javascript
tmux -V
```

**Windows（WSL 2）安装 tmux：**
```javascript
# 在 WSL 2 的 Ubuntu 中执行
sudo apt update && sudo apt install tmux -y
tmux -V
```

**macOS 替代方案 — iTerm2：**
如果你使用 iTerm2，安装 `it2` CLI 并启用 Python API：

3. 安装 it2 CLI：`brew install mkusaka/tap/it2`
4. 打开 iTerm2 → Settings → General → Magic → **勾选 Enable Python API**
5. 使用 `tmux -CC` 启动（iTerm2 会自动将 tmux 窗格转为原生标签页/分割窗口）
</div>

### 3.3 启用 Agent Teams

Agent Teams 是实验性功能，需要手动启用。编辑 `settings.json`：

```json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  },
  "teammateMode": "tmux"
}
```

配置文件位置：

- **macOS / Linux / WSL**：`~/.claude/settings.json`
- 也可以在项目级别配置：`.claude/settings.json`

`teammateMode` 的可选值：

| 值 | 行为 | 要求 |
| --- | --- | --- |
| `"auto"`（默认） | 如果已在 tmux 中则用分屏，否则用 in-process | - |
| `"tmux"` | **强制使用 Split Pane 分屏模式** | 需安装 tmux 或 iTerm2 |
| `"in-process"` | 所有 teammate 在主终端内运行，用快捷键切换 | 无额外依赖 |

### 3.4 启动分屏团队

在终端中启动 Claude Code 后，用自然语言描述你的团队需求：

```plain text
创建一个 Agent Team，3 个 teammate：
- 一个负责实现用户认证模块
- 一个负责编写对应的单元测试
- 一个负责安全审查
```

Claude 会自动：

6. 创建团队并生成共享任务列表
7. 为每个 teammate 启动独立的 tmux 窗格
8. 每个窗格运行一个完整的 Claude Code 实例
9. 你可以看到所有窗格的实时输出

### 3.5 分屏操作速查

| 操作 | tmux 快捷键 | 说明 |
| --- | --- | --- |
| 切换窗格 | `Ctrl+B` 然后 `方向键` | 在不同 teammate 窗格间移动 |
| 与 teammate 对话 | 点击 / 切换到目标窗格后直接输入 | 每个窗格是完整的 Claude Code 会话 |
| 调整窗格大小 | `Ctrl+B` 然后 `Ctrl+方向键` | 拖拽窗格边界 |
| 最大化单个窗格 | `Ctrl+B` 然后 `z` | 再按一次恢复 |
| 新建窗格（水平分割） | `Ctrl+B` 然后 `"` | 上下分屏 |
| 新建窗格（垂直分割） | `Ctrl+B` 然后 `%` | 左右分屏 |
| 关闭当前窗格 | `Ctrl+B` 然后 `x` | 确认后关闭 |
| 查看任务列表 | `Ctrl+T`（Claude Code 内） | 查看团队共享任务进度 |
| 滚动窗格内容 | `Ctrl+B` 然后 `[`，用方向键滚动 | 按 `q` 退出滚动模式 |

<div class="callout callout-warning" markdown="1">
**注意：** iTerm2 用户使用 `tmux -CC` 时，tmux 窗格会自动转换为 iTerm2 的原生分割窗口，可以直接用鼠标点击切换，无需记忆 tmux 快捷键。
</div>

---

## 四、方式二：手动 tmux 分屏（轻量方案）

如果你不需要 Agent Teams 的自动协作能力，只想**手动开多个独立的 Claude Code 会话并排工作**，直接用 tmux 手动分屏即可。

### 4.1 快速上手

```bash
# 1. 启动 tmux 会话
tmux new -s coding

# 2. 垂直分屏（左右两栏）
# 按 Ctrl+B 然后按 %

# 3. 在左侧窗格启动 Claude Code
claude

# 4. 切换到右侧窗格
# 按 Ctrl+B 然后按 →

# 5. 在右侧窗格也启动 Claude Code
claude

# 现在你有两个并排的 Claude Code 会话了！
```

### 4.2 推荐布局：三窗格开发模式

```bash
# 启动 tmux
tmux new -s dev

# 左侧大窗格：主开发会话
claude

# 垂直分割（Ctrl+B %）→ 右上：测试/审查会话
claude

# 水平分割右侧（Ctrl+B "）→ 右下：普通终端（运行项目、查看日志）
# 不启动 claude，留作普通 shell 用
```

最终布局：

```javascript
┌──────────────────┬──────────────────┐
│                  │  Claude Code     │
│  Claude Code     │  （测试/审查）     │
│  （主开发）       ├──────────────────┤
│                  │  Terminal        │
│                  │  （运行/日志）     │
└──────────────────┴──────────────────┘
```

### 4.3 配合 Git Worktrees 实现多分支并行

当多个 Claude Code 会话需要修改**同一个仓库**时，推荐使用 Git Worktrees 避免冲突：

```bash
# 为不同功能创建 worktree
git worktree add ../feature-auth -b feature/auth
git worktree add ../feature-api -b feature/api

# tmux 分屏后，每个窗格进入不同的 worktree
# 窗格 1：
cd ../feature-auth && claude

# 窗格 2：
cd ../feature-api && claude
```

<div class="callout callout-success" markdown="1">
**为什么用 Worktrees？**
- 每个窗格在不同的目录工作，**零文件冲突**
- 各自独立的 Git 分支，互不影响
- 完成后合并分支即可整合所有工作
</div>

---

## 五、平台详细配置

### 🍎 macOS 完整配置

macOS 是 Claude Code 分屏的**最佳平台**，原生支持 tmux 和 iTerm2。

**方案 A：Homebrew + tmux（推荐）**

```bash
# 安装 tmux
brew install tmux

# 安装 Claude Code
brew install claude-code
# 或
curl -fsSL https://claude.ai/install.sh | bash

# 启用 Agent Teams
cat > ~/.claude/settings.json << 'EOF'
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  },
  "teammateMode": "tmux"
}
EOF

# 启动 tmux 并运行 Claude Code
tmux new -s dev
claude
```

**方案 B：iTerm2（更友好的 UI 体验）**

```bash
# 安装 it2 CLI
brew install mkusaka/tap/it2

# iTerm2 中启动 tmux -CC（iTerm2 原生集成模式）
tmux -CC new -s dev

# 之后 tmux 的窗格会变成 iTerm2 的原生分割窗口
# 可以用鼠标拖拽调整大小，点击切换
claude
```

### 🪟 Windows 完整配置

<div class="callout callout-warning" markdown="1">
**重要提示：** Agent Teams 的 Split Pane 模式目前**不支持** Windows Terminal 和 PowerShell 原生终端。Windows 用户需要通过 **WSL 2** 使用 tmux 来实现分屏。
</div>

**步骤 1：确保 WSL 2 已安装**

```powershell
# 在 PowerShell（管理员）中执行
wsl --install
# 重启后选择 Ubuntu 22.04 LTS
```

**步骤 2：在 WSL 2 中安装依赖**

```bash
# 更新系统
sudo apt update && sudo apt upgrade -y

# 安装 tmux
sudo apt install tmux -y

# 安装 Git
sudo apt install git -y

# 安装 Claude Code
curl -fsSL https://claude.ai/install.sh | bash

# 验证
claude --version
tmux -V
```

**步骤 3：配置 Agent Teams**

```bash
# 创建配置文件
mkdir -p ~/.claude
cat > ~/.claude/settings.json << 'EOF'
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  },
  "teammateMode": "tmux"
}
EOF
```

**步骤 4：启动分屏**

```bash
# 启动 tmux 会话
tmux new -s dev

# 启动 Claude Code
claude

# 然后让 Claude 创建 Agent Team，自动分屏
```

**Windows 替代方案：Windows Terminal 手动多标签**

如果你不想使用 WSL，可以用 Windows Terminal 的多标签功能模拟分屏：

10. 打开 Windows Terminal
11. `Alt+Shift+D`：在当前标签页旁边打开新窗格
12. 在每个窗格中分别启动 `claude`
13. `Alt+方向键`：在窗格间切换

<div class="callout callout-note" markdown="1">
**注意：** Windows Terminal 的窗格分割只是视觉上的分屏，各个 Claude Code 会话之间**没有** Agent Teams 的自动协作能力（无共享任务列表、无互相通信）。如果需要完整的 Agent Teams 分屏体验，请使用 WSL 2 + tmux。
</div>

---

## 六、实战场景演示

### 场景 1：并行 Code Review（3 个窗格）

```plain text
创建一个 Agent Team 来审查 PR #88，生成三个 reviewer：
- 一个聚焦安全影响（SQL 注入、XSS、敏感信息泄露）
- 一个检查性能影响（N+1 查询、内存泄漏、慢查询）
- 一个验证测试覆盖（是否有遗漏的边界条件测试）
让他们各自审查后汇总发现。
```

### 场景 2：前后端并行开发（2 个窗格）

```plain text
创建一个 Agent Team：
- teammate-1：在 src/backend/api/ 目录实现用户注册 API（Spring Boot）
- teammate-2：在 src/frontend/pages/ 目录实现注册页面（React）
两者完成后互相校验接口契约是否一致。
```

### 场景 3：竞争假设调试（多窗格）

```plain text
用户反馈登录后偶尔闪退。创建 Agent Team，生成 3 个调查员：
- 一个检查 Session/Token 管理逻辑
- 一个检查前端状态管理和路由守卫
- 一个检查网络请求超时和错误处理
让他们互相质疑对方的假设，最终得出最可能的根因。
```

---

## 七、最佳实践

### ✅ 推荐做法

14. **从 3 个 teammate 开始**：平衡并行效率与协调开销，后续按需增加
15. **任务粒度适中**：每个 teammate 5-6 个小任务最佳，既保持忙碌又不过于分散
16. **用 Git Worktrees 避免冲突**：多个会话修改同一仓库时必备
17. **给 teammate 足够上下文**：在启动指令中包含具体路径、技术栈、关注点
18. **定期检查进度**：不要放任团队无人监管太久
19. **先用 Plan Mode 规划**：复杂任务先让 teammate 在只读模式下规划，审批后再执行

### ⚠️ 注意事项

20. **Token 消耗会成倍增加**：每个 teammate 是独立的 Claude 实例，建议使用 Max 订阅
21. **避免多个 teammate 编辑同一文件**：这会导致覆盖冲突
22. **Windows 用户需要 WSL 2**：原生 Windows 终端不支持 tmux Split Pane
23. **Agent Teams 仍是实验性功能**：会话恢复、任务状态同步等可能有 Bug
24. **团队用完后记得清理**：在 Lead 会话中说 `Clean up the team` 释放资源

---

## 八、快速对比：两种分屏方式

| 维度 | Agent Teams Split Pane | 手动 tmux 分屏 |
| --- | --- | --- |
| **自动协作** | ✅ 共享任务列表、互相通信 | ❌ 各自独立，无协作 |
| **配置复杂度** | 中等（需启用实验性功能） | 低（安装 tmux 即可） |
| **Token 消耗** | 较高（协调通信额外消耗） | 较低（无协调开销） |
| **macOS 支持** | ✅ 完美支持 | ✅ 完美支持 |
| **Windows 支持** | ⚠️ 需要 WSL 2 | ⚠️ 需要 WSL 2 或 Windows Terminal 手动分屏 |
| **适用场景** | 需要多代理协作的复杂任务 | 简单的并行独立任务 |
| **上手难度** | ⭐⭐⭐ | ⭐⭐ |

---

## 九、常见问题排查

### Q：分屏后看不到 teammate 窗格？

- 确认 tmux 已安装：`which tmux`
- 确认 `teammateMode` 设置为 `"tmux"`
- 如果使用 in-process 模式，按 `Shift+Down` 循环切换 teammate

### Q：Windows 上 Split Pane 不生效？

- Split Pane 模式不支持 Windows Terminal 和 PowerShell 原生终端
- 请使用 **WSL 2 + tmux** 方案

### Q：tmux 会话残留怎么清理？

```bash
# 查看所有 tmux 会话
tmux ls

# 关闭指定会话
tmux kill-session -t <session-name>

# 关闭所有会话
tmux kill-server
```

### Q：权限提示太多怎么办？

在启动 teammate 之前，先在 `/permissions` 中预批准常用操作（如 Read、Grep、Glob），减少中断。

---

> *"分屏不只是视觉上的多窗口，而是让 AI 真正并行工作的能力——从串行等待变成并行推进。"*
