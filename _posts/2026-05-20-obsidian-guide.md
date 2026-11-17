---
layout: post
title: "Obsidian 使用指南 · 从入门到 AI 重度玩家"
date: 2026-05-20
categories: [tools]
mermaid: false
math: false
toc: true
---
<div class="callout callout-tip" markdown="1">
**一句话总结**：Obsidian = 纯本地 Markdown 文件夹 + 强大的双向链接 + 插件化生态。它不是一个笔记软件，而是一个「为你大脑而生的可编程知识操作系统」。
本指南面向**有工程背景 + 重度 AI 用户**，目标是让你在 1~2 小时内完成从安装到生产力工作流的搭建，并配置一套 AI 友好的插件栈。
</div>

## 目录

---

## 一、Obsidian 是什么，为什么值得用

### 1.1 核心理念

- **本地优先（Local-first）**：所有笔记都是 `.md` 文件，存放在你自己电脑上的一个文件夹（叫 **Vault / 仓库**），软件随时可换，数据永远是你的。
- **链接即思考**：用 `**双向链接**` 把零散笔记织成网络，强调「概念之间的关系」而不是「文件夹层级」。
- **插件化即一切**：核心很轻，几乎所有高级能力（日历、看板、数据库、AI、画图、同步…）都来自社区插件。

### 1.2 和其他工具的对比

| 维度 | Obsidian | Notion | Logseq | 语雀 / 飞书文档 |
| --- | --- | --- | --- | --- |
| 存储 | 本地 Markdown | 云端数据库 | 本地 Markdown / EDN | 云端 |
| 离线能力 | ✅ 完全离线 | ❌ 弱 | ✅ | ❌ |
| 双向链接 | ✅ 一等公民 | ⚠️ 支持但弱 | ✅ 一等公民 | ❌ |
| 数据库 / 视图 | ⚠️ 靠插件 (Dataview/Bases) | ✅ 原生强 | ⚠️ Query | ⚠️ |
| 插件生态 | ✅ 极强（2000+） | ⚠️ 封闭 | ⚠️ 中等 | ❌ |
| AI 集成 | ✅ 插件 + 本地模型友好 | ✅ 官方 | ⚠️ | ⚠️ |
| 协作 | ⚠️ 弱（靠 Git/Sync） | ✅ 强 | ⚠️ | ✅ 强 |
| 价格 | 个人免费 | 免费 + 订阅 | 免费 | 免费 + 订阅 |

<div class="callout callout-note" markdown="1">
**给你的建议（基于你的画像）**：Obsidian 用来做**个人知识库 + 工作日志 + AI 提示词与 Skill 沉淀**；Notion 继续做**结构化文档与团队协作**。两者互补，不必二选一。
</div>

---

## 二、安装与初始化（macOS Apple Silicon）

### 2.1 安装

```bash
# 方式一：Homebrew（推荐）
brew install --cask obsidian

# 方式二：官网下载
# https://obsidian.md/download
```

### 2.2 创建你的第一个 Vault

1. 打开 Obsidian → `Create new vault`。
2. 路径建议：`~/Documents/ObsidianVault`（或放进 iCloud / 坚果云 / OneDrive 文件夹用于同步，但**不要**放进 Dropbox 的 selective sync）。
3. 勾选 `Use existing folder` 也可以，把任何已有的 markdown 目录直接变成 Vault。

### 2.3 推荐的目录结构（PARA + Zettelkasten 混合）

```plain text
ObsidianVault/
├── 00-Inbox/              # 临时收集箱，每周清理
├── 10-Daily/              # 每日日记（Daily Notes 插件自动生成）
├── 20-Projects/           # 进行中的项目（车联网、TBOX、HTTP 转发观测台...）
├── 30-Areas/              # 长期负责领域（Java、SQL、AI Coding、健康...）
├── 40-Resources/          # 资料、剪藏、论文、教程
├── 50-Archive/            # 归档
├── 60-Zettel/             # 永久笔记（原子化卡片，用 [[双链]] 互连）
├── 70-Templates/          # 模板
├── 80-Attachments/        # 图片、PDF 附件
└── 99-Meta/               # 索引页 (MOC)、知识地图
```

<div class="callout callout-note" markdown="1">
**为什么用数字前缀？**
侧栏会按字母排序，数字前缀能保证你「最常用的目录」永远在最上面，迁移到其他工具时也清晰可读。
</div>

### 2.4 必做的初始设置

在 `Settings` ⚙️ 里依次调整：

- **Editor**
    - ✅ `Use tabs` → False（用空格更稳，4 空格）
    - ✅ `Show line number` → True
    - ✅ `Strict line breaks` → False（让单回车也换行）
    - ✅ `Readable line length` → 个人偏好，宽屏可关
- **Files & Links**
    - `Default location for new notes` → `00-Inbox`
    - `New link format` → **Relative path to file**（迁移友好）
    - `Use **Wikilinks**` → ✅ True
    - `Default location for new attachments` → `In the folder specified below` → `80-Attachments`
- **Appearance**
    - Base color scheme：跟随系统
    - 字体：等宽推荐 **JetBrains Mono / MapleMono**
- **Hotkeys**：先记住几个高频快捷键（见 §4）
- **Core plugins**（先把这些打开）：
    - Daily notes、Templates、Outline、Tag pane、Backlinks、Outgoing links、Graph view、Bookmarks、Page preview、Search、Command palette、Quick switcher、File recovery、Properties view

---

## 三、核心概念速通

### 3.1 Markdown 基础（Obsidian 方言）

```markdown
# H1 一级标题
## H2
### H3

**加粗**  *斜体*  ~~删除线~~  ==高亮==  `inline code`

- 无序列表
	- 嵌套（Tab 缩进）
1. 有序列表

- [ ] 待办
- [x] 完成

> 引用

---
分割线

[外链](https://obsidian.md)
![[图片.png]]              <!-- 嵌入附件 -->
![[某笔记]]                 <!-- 嵌入整篇笔记 -->
![[某笔记#某标题]]          <!-- 嵌入笔记的某节 -->
![[某笔记#^block-id]]       <!-- 嵌入块 -->

[[双向链接]]
[[双向链接|显示别名]]
#tag/sub-tag                <!-- 标签支持层级 -->
```

### 3.2 Properties / Frontmatter（**极其重要**）

Obsidian 1.4+ 把笔记顶部的 YAML 升级成了「Properties」，是后续 Dataview / Bases / AI 检索的基石：

```markdown
---
title: 2026 Q1 绩效自评
tags: [work, review, q1]
type: review
project: [[车联网新旧平台切换]]
date: 2026-04-15
status: done
rating: 9
---
```

字段类型支持：Text / List / Number / Checkbox / Date / Datetime / Tags / 链接。Cmd/Ctrl+; 可快速新增。

### 3.3 双向链接 & 反向链接

- 写 `**XXX**`，目标笔记的 **Backlinks** 面板会自动出现来源。
- 链接到不存在的笔记 → 蓝色虚线，点击直接创建（**这是最 Obsidian 的工作流**）。
- 别名：在目标笔记的 properties 加 `aliases: [TBOX, 车联网终端]`，写 `**TBOX**` 就能跳到「车联网终端」。

### 3.4 块引用（Block Reference）

在任意段落末尾加 `^block-id`，就可以用 `**笔记名#^block-id**` 在别处精确引用这一块。AI 总结里非常有用。

### 3.5 Graph View（关系图谱）

- 全局图谱：看整个 Vault 的链接网络。
- 局部图谱：只看当前笔记 N 步以内的邻居。
- 给不同 tag / folder 配色，知识结构一目了然。

### 3.6 Canvas（白板）

- 把笔记、网页、图片、文本卡片拖到无限画布上。
- 适合做架构图、复盘墙、AI Prompt 流程图。
- 文件本质是 `.canvas`（JSON），可被插件读取。

### 3.7 Bases（官方数据库视图，2025 起内置）

- 类似 Notion Database，把符合条件的 markdown 笔记聚合成表格 / 看板 / 列表。
- 用 Properties 做字段，类似 SQL 的 `WHERE` 过滤。
- **是 Dataview 的官方替代方向**，新项目优先考虑 Bases，复杂查询仍可用 Dataview。

---

## 四、必须背下来的快捷键（macOS）

| 动作 | 快捷键 | 说明 |
| --- | --- | --- |
| 命令面板 | ⌘ P | 所有功能入口，等价于 VSCode |
| 快速跳转 | ⌘ O | 按文件名跳转 |
| 全局搜索 | ⌘ ⇧ F | 全文搜索 |
| 切换源码 / 预览 | ⌘ E |   |
| 新建笔记 | ⌘ N |   |
| 插入链接 | [[ | 边打边搜 |
| 插入标签 | # |   |
| 插入属性 | ⌘ ; |   |
| 左右分屏 | ⌘ ⌥ →/← | 多 pane 编辑 |
| 关闭当前 pane | ⌘ W |   |
| 返回 / 前进 | ⌘ ⌥ ←/→ | 历史导航 |
| 折叠 / 展开标题 | ⌘ ⌥ ↑/↓ |   |
| 插入模板 | 自定义（推荐 ⌘ T） | 需在 Hotkeys 设 |

---

## 五、核心工作流（结合你的画像定制）

### 5.1 每日日志 (Daily Note)

配置 `Core plugins → Daily notes`：

- Date format：`YYYY-MM-DD`
- New file location：`10-Daily`
- Template：指向 `70-Templates/daily.md`

模板示例：

```markdown
---
date: date:YYYY-MM-DD
weekday: date:dddd
type: daily
tags: [daily]
---

# date:YYYY-MM-DD · date:dddd

## 🎯 今日三件事
- [ ] 
- [ ] 
- [ ] 

## 🧱 工作日志（车联网 / DMP）
- 

## 🧠 学习 / 想法（Zettel 入口）
- [[]]

## 🐞 排障 / SQL 片段
```

```javascript

## 📝 复盘
- 做得好：
- 不足：
- 明天：
```

### 5.2 Zettelkasten（原子化卡片）

- 每条只讲一件事，标题就是「概念名」，例如：`TBOX 上线率口径`、`Doris vs SelectDB 物化视图差异`。
- 顶部加 properties：`type: zettel`、`tags: [java, jvm]`、`source: **某论文**`。
- 鼓励**多写小卡片 + 多用 [[]] 串联**，而不是写一篇万字长文。

### 5.3 项目页（Project Page）

每个项目一个目录 + 一张「主页」：

```markdown
---
project: 车联网新旧平台切换
status: in_progress
owner: 张朝
start: 2026-02-01
tags: [project, iov]
---

# 车联网新旧平台切换

## 背景与目标
## 关键里程碑
```

TASK FROM "20-Projects/车联网新旧平台切换" WHERE !completed

```javascript

## 决策记录 (ADR)
- [[ADR-001 转发链路选型]]
- [[ADR-002 SelectDB 表设计]]

## 排障日志
- [[2026-03-12 TBOX 心跳丢失排查]]
```

### 5.4 MOC（Map of Content，知识地图）

在 `99-Meta/` 下建索引页，例如 `MOC-Java.md`：

```markdown
# Java MOC

## JVM
- [[JVM 内存模型]]
- [[GC 调优手册]]

## Spring
- [[Spring Bean 生命周期]]
- [[Spring AOP 实战]]
```

MOC 是「目录页」+「兴趣地图」，避免知识库变成黑盒。

### 5.5 AI Prompt / Skill 沉淀

在 `30-Areas/AI/` 下专门放：

- `prompts/`：可复用提示词，每个一文件，properties 标 `tags: [prompt, role-coder]`。
- `skills/`：把你的工作流（如「冒烟测试 Skill」「Ruflo Agent 编排」）变成可执行 markdown + 代码骨架。
- `agents/`：Agent 设定、记忆、Few-shot 示例。

---

## 六、热门插件清单（按使用频率分级）

安装方式：`Settings → Community plugins → Browse`（首次需关闭 Safe Mode）。

### 6.1 ⭐⭐⭐ 必装（生产力底座）

| 插件 | 作用 | 推荐配置 / 用法 |
| --- | --- | --- |
| **Dataview** | 用类 SQL 把 markdown 当数据库查询 | 见下文 6.6 |
| **Templater** | 增强模板：JS 脚本、动态变量、命令 | 替代官方 Templates，几乎所有自动化的起点 |
| **QuickAdd** | 一键创建笔记 / 插入模板 / 跑脚本 | 把 Daily / Project / Zettel 模板挂到命令面板 |
| **Advanced Tables** | Markdown 表格自动对齐、Tab 切换 | 开箱即用 |
| **Outliner** | 列表项 Tab/Shift+Tab 智能缩进 + 折叠 | 类 Workflowy |
| **Calendar** | 侧边栏日历，点击跳转 Daily Note | 搭配 Daily notes |
| **Periodic Notes** | 支持周/月/季/年笔记 | 做季度复盘必备 |
| **Omnisearch** | 更强的全文检索（模糊 + 排序） | 替代原生搜索 |
| **Style Settings** | 主题/插件可视化微调 | 大量主题依赖它 |

### 6.2 ⭐⭐⭐ AI 类（你的重点）

| 插件 | 定位 | 说明 |
| --- | --- | --- |
| **Copilot for Obsidian** (logancyang) | 聊天 + RAG，支持 OpenAI / Anthropic / Gemini / 本地 Ollama | **首选**。可针对当前笔记或整个 Vault 提问，支持向量索引 |
| **Smart Connections** | 本地向量检索，自动给当前笔记找语义近邻 | 侧栏永远显示「相关笔记」，弥补双链遗漏 |
| **Text Generator** | Prompt 模板化、链式生成 | 适合做内容生产流水线 |
| **Obsidian Web Clipper**（官方） | 浏览器扩展，一键剪藏网页为 markdown | 配合自定义模板自动打标签、抽要点 |
| **Khoj** | 本地 RAG + 桌面/移动端 Agent | 注重隐私时可选 |
| **Ollama 集成** | 本地大模型（Llama 3 / Qwen / DeepSeek） | 通过 Copilot 接入，离线可用 |
| **BMO Chatbot / GPT-Assistant** | 轻量聊天面板 | 替代品，按口味挑一个 |

<div class="callout callout-note" markdown="1">
**AI 推荐组合（结合你 Java + 车联网 + AI Coding 的工作）**
- **Copilot for Obsidian**（主对话 + RAG）
- **Smart Connections**（语义近邻）
- **Templater + QuickAdd**（把 Prompt 变成一键命令）
- **本地 Ollama 跑 Qwen2.5-Coder / DeepSeek-Coder** 用于代码片段补全与解释，敏感工作内容不出本机。
</div>

### 6.3 ⭐⭐ 写作 / 阅读体验

- **Editing Toolbar**：浮动格式工具栏，类似 Word。
- **Iconize / Iconic**：给文件夹和文件加图标。
- **Minimal Theme + Minimal Theme Settings**：最受欢迎的主题之一。
- **Typewriter Scroll** / **Focus Mode**：专注写作。
- **Pandoc Plugin**：一键导出 PDF / DOCX / EPUB。
- **Better Word Count**：中英文分词字数统计。
- **Linter**：保存时自动格式化 markdown / properties。

### 6.4 ⭐⭐ 任务与时间管理

- **Tasks**：跨整个 Vault 的待办引擎，支持 `due:`、`every week`、过滤视图。**强烈推荐**。
- **Day Planner**：把 Daily Note 里的时间块渲染成时间轴。
- **Kanban**：看板视图，做 Sprint / OKR 跟踪。
- **Habit Tracker** / **Tracker**：习惯打卡，从 Daily Note 抽数据画图。

### 6.5 ⭐⭐ 可视化 / 画图

- **Excalidraw**：手绘风格白板，支持把笔记嵌入画布，AI 友好。
- **Mermaid**（内置）：流程图、时序图。
- **Charts / Obsidian Charts**：从 Dataview / properties 出图。
- **Advanced Canvas**：增强官方 Canvas，支持便签连线、组、AI 节点。

### 6.6 ⭐⭐ Dataview 速成（必学）

```javascript

```

TABLE status, file.mtime as "最近编辑"

FROM "20-Projects"

WHERE status != "archived"

SORT file.mtime DESC

```javascript

```

常用 4 种查询：

- `LIST` — 列表
- `TABLE col1, col2 as 别名` — 表格
- `TASK` — 跨笔记聚合所有 `- [ ]`
- `CALENDAR file.cday` — 日历视图

搭配 **DataviewJS**（JS 引擎）可以做任何复杂逻辑，例如**自动生成「本周车联网相关排障笔记」面板**。

### 6.7 ⭐ 同步与备份

| 方案 | 优点 | 缺点 |
| --- | --- | --- |
| **Obsidian Sync**（官方付费） | 端到端加密、版本历史、跨平台稳定 | $4/月起 |
| **iCloud Drive** | 免费、Apple 生态无缝 | 偶发冲突，移动端慢 |
| **Git + Obsidian Git 插件** | 免费、可控、有历史 | 需要会 Git，移动端略折腾 |
| **坚果云 / OneDrive** | 国内速度好 | 大量小文件偶尔丢失，注意冲突 |
| **Self-hosted LiveSync** | 实时同步、可自部署 CouchDB | 运维成本 |

**给你的建议**：开发机用 **Git + GitHub 私有仓库**（你已经熟 Git），手机用 **Working Copy + Obsidian Mobile** 拉取。重要内容多一份 Time Machine。

### 6.8 ⭐ 开发者 / 进阶

- **CustomJS**：在 Templater / Dataview 里复用自己的 JS 工具函数。
- **MetaEdit / Metadata Menu**：给 properties 做表单化编辑。
- **Citations / Zotero Integration**：论文引用管理。
- **Hover Editor**：悬浮预览也能编辑。
- **Workspaces Plus**：保存 / 切换工作区布局。

---

## 七、推荐主题与外观

- **Minimal**（极简，可配色丰富）— 大多数人首选
- **Things**（仿 macOS Things）
- **Border**（更现代的卡片风）
- **AnuPpuccin**（Catppuccin 配色，开发者爱用）
- **Blue Topaz**（中文社区流行，功能极多）

字体推荐（中英混排）：

- 中文：**LXGW WenKai / 霞鹜文楷**、**Sarasa Mono SC**
- 等宽：**JetBrains Mono**、**MapleMono-NF**

---

## 八、给「重度 AI 用户」的进阶玩法

### 8.1 把 Vault 变成你的「外部大脑 RAG 库」

4. 用 **Copilot for Obsidian** 或 **Smart Connections** 对整个 Vault 建索引。
5. 关键设置：
    - 选择 Embedding 模型：本地 `nomic-embed-text` / `bge-m3`（通过 Ollama），或 OpenAI `text-embedding-3-large`。
    - 排除 `00-Inbox`、`80-Attachments` 以减小索引噪声。
6. 工作流：
    - **「问我自己」**：在任意笔记按下唤起 Copilot → Vault QA → 自动引用相关原文笔记。
    - **「写作助手」**：选中段落 → 改写 / 摘要 / 翻译 / 续写。
    - **「每日复盘」**：让 AI 读今天的 Daily Note + 链接笔记，输出三段式复盘。

### 8.2 Prompt as Note（提示词即笔记）

在 `30-Areas/AI/prompts/` 下每个 prompt 一个 md：

```markdown
---
type: prompt
tags: [prompt, role-architect]
model: claude-sonnet-4
temperature: 0.3
---

# 系统设计评审 Prompt

你是一名资深后端架构师……
```

用 Templater / QuickAdd 把它做成命令：选中一段需求 → ⌘P → `运行架构师评审` → 自动把 prompt + 选中内容发给 Copilot。**这就是个人版 Skill 库**。

### 8.3 AI 自动整理 Inbox

用 Templater + Copilot API 写一个脚本：

- 扫描 `00-Inbox/` 下今天的新笔记。
- 调 LLM 自动：①打标签 ②补摘要到 properties ③建议归档目录。
- 输出到 `99-Meta/Inbox-Review-date.md`，你只需要确认。

### 8.4 用 Canvas + AI 做架构图 / 复盘

- 把当前项目相关笔记拖到 Canvas。
- 用 **Advanced Canvas** 的 AI 节点连线生成「为什么 A → B」「下一步推演」。
- 适合你做车联网新旧平台切换的关键决策推演。

### 8.5 用 Bases / Dataview 做「个人 OKR Dashboard」

用 properties 把项目状态结构化，然后一张 Bases 表 / Dataview 面板就能看到：

- 进行中的项目（status、owner、due）
- 本周新增 Zettel 数量、AI 提示词增量
- 待办堆积 Top 5

---

## 九、避坑清单

<div class="callout callout-warning" markdown="1">
7. **不要把 Vault 放在多个云盘的交集里**（例如同时被 iCloud 和 OneDrive 同步），冲突会让你怀疑人生。
8. **不要一开始就装 30 个插件**。每装一个先问「它替代了我哪个流程？」。
9. **链接优先于文件夹**。一篇笔记可以只有 1 个文件夹，但应当有多个 `[[]]` 与 `#tag`。
10. **慎用「重命名 vault 根目录」**。某些插件用绝对路径会失效。
11. **AI 索引要排除附件目录**，不然 PDF 二进制会污染向量库。
12. **定期备份**：再香的本地优先，没有备份都是裸奔。建议 Git + Time Machine 双保险。
13. **移动端别期待和桌面端 1:1**。复杂 Dataview / Templater 在 iOS 上可能很慢。
14. **iCloud 在多设备同时编辑**容易出 `xxx (conflicted).md`，开 Linter 自动清理。
</div>

---

## 十、30 天上手路线图

| 阶段 | 目标 | 动作 |
| --- | --- | --- |
| Day 1 | 跑起来 | 安装、建 Vault、定目录结构、打开核心插件 |
| Day 2~3 | 会写 markdown + 链接 | 每天写 Daily Note，强迫自己用 [[]] 而不是层级目录 |
| Day 4~7 | 装第一批插件 | Templater / QuickAdd / Dataview / Calendar / Tasks |
| Week 2 | 项目页 + MOC | 把「车联网新旧平台切换」「HTTP 转发观测台」搬进 Vault |
| Week 3 | AI 接入 | 装 Copilot for Obsidian + Smart Connections + 本地 Ollama，做 Vault 问答 |
| Week 4 | 个人 Dashboard | 用 Dataview / Bases 做 OKR / Project / 学习面板，开始周复盘 |

---

## 十一、延伸阅读

- 官方文档：[https://help.obsidian.md](https://help.obsidian.md/)
- 官方论坛：[https://forum.obsidian.md](https://forum.obsidian.md/)
- 主题 / 插件市场：在应用内 `Settings → Community plugins / Appearance` 浏览
- 推荐 YouTube：Linking Your Thinking (LYT)、Nicole van der Hoeven、Bryan Jenks
- 推荐方法论：
    - 《How to Take Smart Notes》—— Zettelkasten 圣经
    - PARA 方法（Tiago Forte）
    - LYT / MOC 体系（Nick Milo）

---

<div class="callout callout-success" markdown="1">
**给你的下一步行动建议**
15. 今晚装好 Obsidian + 目录结构。
16. 把现有的 Notion 知识中**纯个人、纯本地、纯 markdown 友好**的部分（比如冒烟测试 Skill、SQL 速查、MacBook 环境笔记）逐步迁过去。
17. 一周后接入 Copilot + Ollama，把你的 AI Prompt / Skill 真正变成「随时可调用的能力」。
18. 一个月后回到这页文档，把你**实际在用**的插件 / 工作流再精简一次——能减则减。
</div>
