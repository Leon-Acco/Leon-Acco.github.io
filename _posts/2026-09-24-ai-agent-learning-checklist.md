---
layout: post
title: "AI Agent 学习清单：知识点 × 作用 × 成熟方案"
date: 2026-09-24
categories: [ai]
series: ai-learning
series_order: 4
mermaid: false
math: false
toc: true
---
<div class="callout callout-note" markdown="1">
这不是按周排的计划，而是一张**知识地图 / 清单**。每一项都写清楚：**它是什么 → 为什么需要它 → 目前市面上成熟的方案或框架**。建议当作 checklist 用，学完一项勾一项。
</div>

## 0. 全景：一个 Agent 由什么组成

```plain text
            ┌──────────────── 记忆 Memory ────────────────┐
            │  短期(上下文) / 长期(向量库) / 结构化(DB)    │
            └──────────────────────┬──────────────────────┘
                                   │
用户输入 ──▶ [ 规划 Planning ] ──▶ [ 推理决策 LLM Core ] ──▶ [ 工具调用 Tools ]
                   ▲                       │                        │
                   └──── 反思 Reflection ◀─┴── 观察 Observation ◀───┘
                                   │
            ┌──────────────────────┴──────────────────────┐
            │  运行时 Runtime：编排 / 状态 / 沙箱 / 权限   │
            └─────────────────────────────────────────────┘
                                   │
            ┌──────────────────────┴──────────────────────┐
            │  工程化：评测 / 可观测 / 成本 / 安全 / 部署  │
            └─────────────────────────────────────────────┘
```

一句话总结：**Agent = LLM（大脑）+ 工具（手脚）+ 记忆（经验）+ 循环控制（意志）+ 工程护栏（纪律）**。

---

## 1. 基座能力：LLM 本身

### 1.1 模型基础概念

- [ ] **Token 与上下文窗口** —— 决定一次能塞多少信息、成本怎么算。所有「上下文工程」问题的根源。
- [ ] **采样参数**（temperature / top_p / seed）—— 控制输出的稳定性。Agent 场景通常要低温度、可复现。
- [ ] **推理模型 vs 通用模型** —— 何时用 thinking 模型（复杂规划）、何时用小快模型（分类、路由）。
- [ ] **幻觉与知识截止** —— 明白为什么必须外挂检索和工具。

**成熟方案**：

| 类型 | 代表 |
| --- | --- |
| 闭源旗舰 | Claude（Sonnet / Opus）、GPT 系列、Gemini |
| 开源权重 | Llama、Qwen、DeepSeek、Mistral |
| 国内可用 | 通义千问 Qwen、DeepSeek、Kimi、豆包、智谱 GLM |
| 本地跑 | Ollama、LM Studio、vLLM、llama.cpp |
| 统一网关 | OpenRouter、LiteLLM、One-API |

### 1.2 Prompt 工程（Agent 版）

- [ ] **System Prompt 结构化写法**：角色 / 目标 / 可用工具 / 约束 / 输出格式 / 失败处理。
- [ ] **Few-shot 示例**：比长篇描述更能稳定行为。
- [ ] **思维链 CoT / ReAct 格式**：让模型「先想再做」。
- [ ] **结构化输出**：强制 JSON Schema，是工具调用可靠性的基础。

**成熟方案**：OpenAI Structured Outputs、Anthropic Tool Use、Instructor、Outlines、Guidance、Pydantic AI、BAML、DSPy（自动优化 prompt）。

---

## 2. 核心机制一：工具调用（Tool / Function Calling）

> **干啥的**：让模型从「只会说」变成「能做事」——查数据库、调 API、发邮件、跑代码。

- [ ] **Function Calling 原理**：模型输出结构化的函数名 + 参数，由你的代码执行后把结果回灌。
- [ ] **工具描述设计**：名称、描述、参数 Schema 写得好坏，直接决定调用准确率（这是最被低估的技能）。
- [ ] **并行调用与串行依赖**：能同时查的就并发，有依赖的必须排队。
- [ ] **错误处理与重试**：工具报错时把错误信息回灌给模型让它自愈。
- [ ] **工具过多怎么办**：工具检索（RAG over tools）、分组、子 Agent 分流。

**成熟方案**：

- **MCP（Model Context Protocol）** —— Anthropic 提出、目前事实标准的工具/数据源接入协议，生态最丰富，优先学这个。
- OpenAI Function Calling / Responses API、Anthropic Tool Use
- LangChain Tools、LlamaIndex Tools、Composio（几百个 SaaS 工具开箱即用）、Arcade
- 代码执行沙箱：**E2B**、Daytona、Modal、Docker + Firecracker、Pyodide（浏览器内）

---

## 3. 核心机制二：记忆（Memory）

> **干啥的**：让 Agent 跨轮次、跨会话地「记得住」，而不是每次从零开始。

| 层级 | 作用 | 典型实现 |
| --- | --- | --- |
| 短期 / 工作记忆 | 当前会话上下文 | 消息数组、滑动窗口、摘要压缩 |
| 长期语义记忆 | 记住事实、偏好 | 向量库检索 |
| 情节记忆 | 记住「上次怎么做的」 | 会话日志 + 检索 |
| 程序性记忆 | 记住技能/流程 | Skill 文件、SOP 文档 |
| 结构化记忆 | 精确数据 | SQL / KV / 知识图谱 |

- [ ] **上下文压缩策略**：滚动摘要、要点提取、分层摘要。
- [ ] **写入时机**：什么该记、什么不该记（避免记忆污染）。
- [ ] **检索时机**：每轮都检索 vs 按需检索。

**成熟方案**：Mem0、Zep、Letta（原 MemGPT）、LangMem、Cognee；底层向量库用 pgvector、Qdrant、Milvus、Weaviate、Chroma、LanceDB。

---

## 4. 核心机制三：检索增强（RAG）

> **干啥的**：把私有知识、实时资料喂给模型，降幻觉、提准确率。RAG 不是 Agent 的对立面，而是 Agent 的一个工具。

- [ ] **文档解析**：PDF / 表格 / 扫描件的坑最多。
- [ ] **切分策略**：固定长度、语义切分、按结构切分、父子块。
- [ ] **Embedding 与向量检索**：模型选型、维度、相似度度量。
- [ ] **混合检索**：向量 + BM25 关键词，再融合（RRF）。
- [ ] **重排 Rerank**：召回 50 条 → 精排 5 条，性价比最高的一步优化。
- [ ] **查询改写**：HyDE、多查询扩展、子问题拆解。
- [ ] **Agentic RAG**：让 Agent 自己决定检索几次、检索什么。
- [ ] **GraphRAG**：需要跨文档全局推理时用。

**成熟方案**：

| 环节 | 方案 |
| --- | --- |
| 解析 | Unstructured、LlamaParse、Docling、MinerU、PyMuPDF |
| 框架 | LlamaIndex（首选）、LangChain、Haystack、RAGFlow、Dify |
| Embedding | OpenAI text-embedding-3、BGE-M3、Jina、Voyage、Cohere Embed |
| Rerank | Cohere Rerank、BGE-Reranker、Jina Reranker |
| 图谱 | Microsoft GraphRAG、LightRAG、Neo4j |

---

## 5. 核心机制四：规划与推理循环

> **干啥的**：决定 Agent「下一步做什么」，是 Agent 区别于普通对话机器人的灵魂。

- [ ] **ReAct**（Reason + Act 交替循环）—— 最基础也最常用的范式，必学。
- [ ] **Plan-and-Execute** —— 先出全局计划再逐步执行，适合长任务。
- [ ] **Reflection / Self-Critique** —— 执行完自我审查再修正（Reflexion、Self-Refine）。
- [ ] **Tree of Thoughts / 多路径搜索** —— 需要探索多解时用，成本高。
- [ ] **任务分解与子任务管理** —— TODO list 模式，现在主流 Coding Agent 都这么干。
- [ ] **终止条件与循环上限** —— 防止无限循环烧钱，工程必备。

**成熟方案**：LangGraph（状态机式编排，工业界最主流）、OpenAI Agents SDK、Anthropic Claude Agent SDK、Pydantic AI、Google ADK、Smolagents（极简，代码即行动）。

---

## 6. 多 Agent 与编排

> **干啥的**：单个 Agent 上下文塞不下、职责太杂时，拆成多个专职 Agent 协作。

- [ ] **常见拓扑**：Supervisor（主管分发）、Pipeline（流水线）、Group Chat（群聊辩论）、Hierarchical（层级）。
- [ ] **Handoff / 交接机制**：怎么把控制权和上下文传给下一个 Agent。
- [ ] **共享状态与黑板模式**：多 Agent 之间怎么同步信息。
- [ ] **什么时候不该上多 Agent**：绝大多数场景，单 Agent + 好工具就够了。先别过度设计。

**成熟方案**：LangGraph（Supervisor / Swarm 模板）、AutoGen / AG2、CrewAI、OpenAI Swarm & Agents SDK、MetaGPT、Camel-AI；跨 Agent 通信协议关注 **A2A（Agent2Agent）**。

---

## 7. 上下文工程（Context Engineering）

> **干啥的**：2025 年后被公认为比 Prompt 工程更重要的技能——在有限窗口里放「刚好够用」的信息。

- [ ] **上下文预算分配**：system / 工具定义 / 记忆 / 检索结果 / 历史，各占多少。
- [ ] **压缩与卸载**：把长内容写进文件，上下文里只留引用（file system as context）。
- [ ] **上下文腐化（Context Rot）**：太长反而变笨，要主动裁剪。
- [ ] **Just-in-time 检索**：需要时才拉，而不是一次性全塞。
- [ ] **Prompt Caching**：把稳定前缀缓存住，成本能降一个数量级。

**成熟方案**：Anthropic Prompt Caching / Context Editing、OpenAI Prompt Caching、LangGraph 的 state 裁剪、Claude Skills（按需加载能力包）。

---

## 8. 评测（Evaluation）

> **干啥的**：没有评测的 Agent 迭代 = 盲人开车。这是从 Demo 走向生产的分水岭。

- [ ] **构建评测集**：从真实流量里挑 50–200 条典型 + 边缘案例。
- [ ] **指标体系**：任务成功率、工具调用准确率、步数/成本、延迟、幻觉率。
- [ ] **LLM-as-Judge**：用模型打分，注意校准和偏差。
- [ ] **轨迹评测（Trajectory Eval）**：不只看最终答案，还看中间每一步走得对不对。
- [ ] **回归测试**：改 prompt / 换模型后，跑一遍确保没退化。

**成熟方案**：LangSmith、Braintrust、Langfuse、Ragas（RAG 专用）、DeepEval、Promptfoo、OpenAI Evals、Phoenix (Arize)。

---

## 9. 可观测性与调试

> **干啥的**：Agent 是黑盒多步循环，出问题时你必须能看到「它第 7 步为什么调错了工具」。

- [ ] **Tracing / Span**：完整记录每次 LLM 调用、工具调用的输入输出。
- [ ] **Token 与成本追踪**：按会话、按用户、按功能维度算账。
- [ ] **回放与断点续跑**：从失败那一步重来，而不是整个重跑。
- [ ] **OpenTelemetry GenAI 语义约定**：标准化埋点。

**成熟方案**：LangSmith、Langfuse（开源自托管首选）、Arize Phoenix、Helicone、W&B Weave、OpenLLMetry。

---

## 10. 安全、权限与护栏

> **干啥的**：Agent 能动手，就意味着能闯祸。这一块是上线前的硬门槛。

- [ ] **提示词注入（Prompt Injection）**：外部网页/文档里藏指令让 Agent 干坏事。核心防线是「内容≠指令」。
- [ ] **最小权限原则**：每个工具只给必要权限，写操作和读操作分级。
- [ ] **Human-in-the-loop**：高风险动作（付款、删除、发送）必须人工确认。
- [ ] **沙箱隔离**：代码执行、浏览器操作一律隔离环境。
- [ ] **输入输出过滤**：PII 脱敏、敏感词、越狱检测。
- [ ] **审计日志**：谁在什么时候让 Agent 做了什么。

**成熟方案**：NeMo Guardrails、Guardrails AI、Llama Guard、OpenAI Moderation、Rebuff / Lakera（注入检测）、E2B 沙箱、LangGraph 的 interrupt / human approval 节点；参考 **OWASP Top 10 for LLM Applications**。

---

## 11. 工程化与部署

> **干啥的**：让 Agent 从「本地脚本」变成「能被别人用的服务」。

- [ ] **长任务执行**：异步任务队列、断点续跑、超时控制。
- [ ] **状态持久化（Checkpointing）**：进程挂了能恢复。
- [ ] **流式输出**：SSE / WebSocket，让用户看到中间过程。
- [ ] **并发与限流**：模型 API 有 RPM/TPM 限制，要做退避重试。
- [ ] **成本优化**：模型分级路由（简单任务用小模型）、缓存、批处理。
- [ ] **多租户与隔离**。

**成熟方案**：LangGraph Platform、Temporal（长流程编排）、Celery / Redis Queue、Inngest、Modal、Ray Serve、vLLM（自托管推理）、LiteLLM（路由 + 限流 + 计费）。

---

## 12. 典型应用形态（选一个深挖）

| 形态 | 干啥的 | 代表方案 |
| --- | --- | --- |
| **Coding Agent** | 读写代码库、跑测试、提 PR | Claude Code、Cursor、Codex、Aider、OpenHands、SWE-agent |
| **Deep Research** | 多轮检索 + 综合成报告 | OpenAI/Gemini Deep Research、GPT-Researcher、Open Deep Research |
| **Computer / Browser Use** | 操作浏览器和桌面 | Claude Computer Use、Browser Use、Playwright MCP、Stagehand |
| **RPA / 工作流自动化** | 连接 SaaS 做业务流 | n8n、Dify、Coze、Zapier Agents、Make |
| **客服 / 销售 Agent** | 对话式业务处理 | Sierra、Intercom Fin、自建 LangGraph |
| **数据分析 Agent** | Text-to-SQL、图表 | Vanna、Julius、PandasAI |

---

## 13. 进阶（有余力再看）

- [ ] **微调与后训练**：LoRA / QLoRA、SFT、DPO —— 什么时候值得微调（通常是最后手段）。工具：Unsloth、LLaMA-Factory、Axolotl、TRL。
- [ ] **强化学习训 Agent**：GRPO、RLVR、环境构建。工具：verl、OpenRLHF、ART。
- [ ] **小模型蒸馏**：把大模型的 Agent 行为蒸馏到小模型降本。
- [ ] **协议生态**：MCP、A2A、AG-UI —— 关注标准化走向。

---

## 14. 学习顺序建议（把清单串起来）

1. **先跑通**：用 OpenAI/Anthropic 原生 API 手写一个 ReAct 循环（不用框架），彻底理解「模型输出工具调用 → 你执行 → 回灌」这个闭环。**这一步别跳过。**
2. **再用框架**：换成 LangGraph 或 OpenAI Agents SDK 重写一遍，体会框架帮你解决了什么。
3. **加记忆和 RAG**：接一个向量库，做一个能记住你偏好的知识问答 Agent。
4. **接 MCP**：把外部工具（文件系统、GitHub、数据库）接进来。
5. **上评测和可观测**：接 Langfuse/LangSmith，建 30 条评测集，量化你的改动。
6. **做安全和部署**：加 human-in-the-loop、沙箱、日志，部署成一个真实可用的服务。

<div class="callout callout-warning" markdown="1">
**三个最常见的坑**：① 一上来就上多 Agent（90% 的场景单 Agent 更好）；② 只调 prompt 不建评测集，改到后面全靠感觉；③ 工具描述写得太随意，然后怪模型笨。
</div>

[AI Agent 工程师面试题库（大厂面试官视角）](/2026/09/24/ai-agent-interview-questions/)
