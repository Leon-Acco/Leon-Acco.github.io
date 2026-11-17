---
layout: post
title: "Agent Skills：从“提示词驱动”到“能力驱动”的智能体架构演进"
date: 2026-05-20
categories: [ai]
mermaid: false
math: false
toc: true
---
---

# 一、Agent Skills 是怎么来的？（背景与由来）

### 1️⃣ 先看「Agent」本身的问题

在 LLM Agent（智能体）出现早期，大家的典型做法是：

- 一个 Agent
- 一大段 Prompt
- 外加一堆 Tool（函数 / API）
- 让模型自己“想办法用工具”

**问题很快就暴露了：**

| 问题 | 表现 |
| --- | --- |
| Prompt 越来越大 | 上千行 system prompt |
| 行为不稳定 | 同样输入，有时用 A 工具，有时用 B |
| 复用性差 | 一个 Agent 的能力很难拆出来给别的 Agent 用 |
| 可测试性极差 | 你根本不知道是哪一段 Prompt 出了问题 |
| 升级困难 | 改一个能力，可能整个 Agent 都崩 |

👉 **Agent 变成了一个“超级脚本怪物”**

---

### 2️⃣ Agent Skills 的思想来源

Agent Skills 的思想，本质上来自 **三个成熟领域** 的交汇：

**软件工程**
   - 单一职责
   - 模块化
   - 可测试、可复用
2. **人类技能模型**
   - 人不是每次都从零思考
   - 而是调用“技能”：写邮件、做总结、查资料
3. **Agent 架构进化**
   - 从 “Prompt 驱动”
   - → “能力驱动（Capability-based Agent）”

👉 于是就出现了一个自然的抽象：

**把 Agent 的“能力”显式拆出来，称为 Skill（技能）**

---

# 二、Agent Skills 解决了什么问题？

一句话总结：

**它解决的是：Agent 不可控、不可维护、不可规模化的问题**

### 1️⃣ Skill 是什么？

**Skill ≠ Tool ≠ Prompt**

你可以这样理解：

```plain text
Skill = 可复用的、目标明确的、受约束的 Agent 行为单元
```

一个 Skill 通常包含：

- 明确的输入
- 明确的输出
- 内部可能用 Prompt + Tool + 推理
- 对外是一个“能力黑盒”

---

### 2️⃣ 它解决的核心痛点

### ❌ 以前（没有 Skills）

```plain text
用户需求
  ↓
一个超级 Agent
  ↓
模型自己想
  ↓
随机用工具
```

- 行为不可预测
- 很难 debug
- 很难复用

### ✅ 现在（有 Skills）

```plain text
用户需求
  ↓
Planner / Router
  ↓
Skill A / Skill B / Skill C
```

- 能力边界清晰
- 行为稳定
- 可单独测试、升级

---

# 三、Agent Skills 的核心优势

### 1️⃣ 工程级优势（最重要）

| 优势 | 说明 |
| --- | --- |
| 模块化 | 一个 Skill = 一个能力 |
| 可复用 | 多 Agent 共享 Skill |
| 可测试 | Skill 可以单测 |
| 可维护 | 升级只影响一个 Skill |
| 可控 | 限定模型“能做什么” |

---

### 2️⃣ 对 LLM 本身也友好

- Prompt 变短
- 上下文更聚焦
- 减少幻觉
- 降低 Token 成本

---

### 3️⃣ 更接近“真正的智能体”

人做事是：

判断 → 调用技能 → 执行 → 返回结果

而不是：

每次都即兴推理一遍

---

# 四、Agent Skills vs MCP（重点对比）

这个对比**非常关键**，很多人会混。

### 先一句话结论：

**MCP 解决的是“Agent 怎么用外部能力”Agent Skills 解决的是“Agent 本身有哪些内在能力”**

---

### 1️⃣ 定位对比

| 维度 | Agent Skills | MCP |
| --- | --- | --- |
| 层级 | Agent 内部能力层 | Agent ↔ 外部系统 |
| 关注点 | 行为与能力建模 | 工具/服务标准化 |
| 是否涉及推理 | ✅ 是 | ❌ 否 |
| 是否是协议 | ❌ 架构思想 | ✅ 协议 |
| 是否 LLM 参与 | 强参与 | 弱/不参与 |

---

### 2️⃣ 本质差异（非常重要）

### MCP 是什么？

```plain text
一个统一的“工具接入协议”
```

- 标准化 tool 描述
- 跨平台
- 跨语言
- 类似：USB / HTTP / gRPC

👉 **MCP 本身不“智能”**

---

### Agent Skill 是什么？

```plain text
一个“可思考、可规划、可组合”的能力单元
```

- 内部可能用 MCP Tool
- 但 Skill 本身是 Agent 行为

---

### 3️⃣ 它们是对立的吗？

❌ 不是✅ **是上下游关系**

```plain text
Agent
 ├── Planner
 ├── Skill：数据分析
 │     └── MCP Tool：数据库查询
 ├── Skill：报告生成
 │     └── MCP Tool：文档系统
```

👉 **MCP 是 Skill 的“肌肉”Agent Skills 是 Agent 的“大脑模块”**

---

# 五、一个完整的 Agent Skills 实现案例（实战）

### 🎯 场景：自动生成市场分析报告的 Agent

---

### 1️⃣ 拆 Skill（这是关键一步）

### Skill 1：需求理解（Requirement Analysis）

- 输入：用户一句话
- 输出：结构化需求

```json
{
  "industry": "新能源",
  "region": "中国",
  "time_range": "2024",
  "depth": "高层汇报"
}
```

---

### Skill 2：数据收集（Data Collection）

- 内部：
- 调用 MCP：数据库 / API
- 输出：原始数据摘要

---

### Skill 3：分析推理（Analysis）

- 纯 LLM 推理
- 不直接对外

---

### Skill 4：报告生成（Report Writing）

- 输入：分析结果
- 输出：结构化报告（Markdown / PPT）

---

### 2️⃣ Skill 的代码抽象（示意）

```python
class MarketAnalysisSkill(Skill):
    name = "market_analysis"

    def run(self, requirement):
        data = self.collect_data(requirement)
        insight = self.analyze(data)
        report = self.write_report(insight)
        return report
```

---

### 3️⃣ Agent 的整体流程

```plain text
用户输入
  ↓
Planner 判断需要哪些 Skills
  ↓
依次调用：
  - RequirementSkill
  - DataCollectionSkill
  - AnalysisSkill
  - ReportSkill
  ↓
最终输出
```

---

### 4️⃣ Skill 的好处立刻体现

- 数据源换了？只改 DataCollectionSkill
- 报告风格换了？只改 ReportSkill
- 想加“竞品对比”？加一个新 Skill

---

# 六、什么时候你“真的需要”学 Agent Skills？

如果你符合以下任一条，**就该学**：

- ✅ 你在做 **多步 Agent**
- ✅ 你觉得 Prompt 越写越乱
- ✅ 你想做可长期演进的 AI 系统
- ✅ 你在考虑企业级 / 产品级 Agent

如果只是玩 Demo，**可以先不拆 Skill**。

---

# 七、一句总结（给你一个“脑内模型”）

**Agent Skills = 把“会做什么”变成一等公民MCP = 把“能用什么工具”变成标准接口**

（最后给大家提供一个快速获取Skills的网站：[Skills MAP](https://skillsmp.com/)）
