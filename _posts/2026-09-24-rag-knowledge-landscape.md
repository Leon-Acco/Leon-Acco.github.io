---
layout: post
title: "RAG 知识体系全景图：从原理到生产落地"
date: 2026-09-24
categories: [ai]
mermaid: false
math: false
toc: true
---
<div class="callout callout-note" markdown="1">
RAG（Retrieval-Augmented Generation，检索增强生成）= 检索系统 + 生成模型。核心思想：**不要让模型凭记忆答题，让它开卷考试。**本页按「概念 → 链路 → 每个环节的技术点 → 进阶范式 → 评测 → 工程 → 学习路径」组织，可作为系统学习清单。
</div>

## 一、为什么需要 RAG

| 大模型的问题 | RAG 如何缓解 |
| --- | --- |
| 幻觉：编造事实 | 答案锚定在检索到的原文上，可溯源 |
| 知识截止：不知道新事情 | 知识库随时更新，无需重训模型 |
| 不懂私域知识：企业内部文档 | 把私有数据放进检索库 |
| 微调成本高、周期长 | 改数据即可，成本低、迭代快 |
| 无法引用出处 | 天然可给出 citation |
| 上下文窗口有限 | 只塞入最相关的片段 |

**RAG vs 微调 vs 长上下文**

- **RAG**：知识频繁变化、要求可溯源、数据量大 → 首选
- **微调**：改变风格、格式、领域术语表达、固化行为模式 → 补充
- **长上下文直接塞**：文档少（几十页内）、一次性任务 → 更简单
- 现实做法通常是三者组合：RAG 供事实 + 微调调风格 + 长上下文兜底

---

## 二、RAG 的标准链路

```plain text
[离线] 数据源 → 解析 → 清洗 → 分块 → 向量化 → 写入索引
                                              ↓
[在线] 用户 Query → 查询改写 → 混合检索 → 重排 → 上下文压缩
                                              ↓
                              Prompt 组装 → LLM 生成 → 引用标注 → 后校验
                                              ↓
                                        评测 / 反馈 / 迭代
```

---

## 三、离线索引阶段（Indexing）

### 3.1 数据接入与解析

- **文档类型**：PDF、Word、PPT、Excel、HTML、Markdown、扫描件、代码库、数据库表、API
- **解析难点**：PDF 双栏排版、表格、公式、页眉页脚、图片中的文字
- **常用工具**：Unstructured、PyMuPDF、pdfplumber、MinerU、Marker、Docling、LlamaParse、Tesseract/PaddleOCR
- **版面分析（Layout Analysis）**：先识别标题/正文/表格/图，再按结构提取，比纯文本抽取质量高一个档次
- **表格处理**：转成 Markdown 表或结构化 JSON，必要时为每张表生成一段自然语言摘要一起入库
- **多模态**：图片走 VLM 生成描述文本；或用 CLIP 类模型做图文统一向量

### 3.2 清洗与规范化

- 去重（精确去重 + MinHash/SimHash 近似去重）
- 去噪：广告、导航栏、免责声明、重复页眉
- 编码统一、全半角、繁简、日期与单位规范化
- 敏感信息脱敏（PII）

### 3.3 分块（Chunking）——RAG 效果的第一杀手

| 策略 | 说明 | 适用 |
| --- | --- | --- |
| 固定长度 | 按 token/字符切，带 overlap（10-20%） | 基线做法 |
| 递归分块 | 按 段落→句子→字符 逐级切 | 通用默认 |
| 结构感知 | 按 Markdown 标题、HTML 标签、代码函数切 | 结构化文档 |
| 语义分块 | 相邻句向量相似度骤降处断开 | 长散文 |
| 命题分块 | 拆成独立可理解的原子命题 | 高精度问答 |
| Late Chunking | 先整篇编码再切分，保留全局上下文 | 长文档 |
| Agentic Chunking | 用 LLM 判断切分点 | 质量优先、成本高 |

**关键技巧**

- **Small-to-Big / Parent-Child**：用小块做检索保证精度，命中后返回父块或整节给 LLM 保证信息完整
- **上下文增强（Contextual Retrieval）**：给每个块前置一句「本块属于 X 文档 Y 章节，讲的是……」，Anthropic 实验显示可显著降低检索失败率
- **元数据注入**：标题路径、来源、日期、作者、权限标签、版本号
- 分块大小经验值：中文 300–800 字，英文 200–500 token，视问答粒度调整

### 3.4 向量化（Embedding）

- **模型选择**：BGE-M3、GTE、Conan、Qwen3-Embedding（中文强）；OpenAI text-embedding-3、Cohere Embed、Voyage、E5、Jina（多语/英文）
- **关键参数**：维度、最大输入长度、是否支持多语言、是否区分 query/passage 前缀
- **对称 vs 非对称检索**：问答场景是非对称的（短 query 找长文档），要用支持 instruction 前缀的模型
- **维度压缩**：Matryoshka 表示学习可截断维度省存储
- **领域微调**：用业务问答对做对比学习（InfoNCE），配合难负例挖掘，通常是收益最大的单点优化

### 3.5 索引与存储

- **向量索引算法**：Flat（精确）、IVF、HNSW（主流）、IVF-PQ / ScaNN（大规模压缩）、DiskANN（超大规模落盘）
- **量化**：PQ、SQ、二值量化（BQ），用精度换内存
- **向量数据库**：Milvus、Qdrant、Weaviate、Chroma、LanceDB、Vespa、Elasticsearch / OpenSearch、pgvector、Redis Vector、Pinecone、Zilliz、阿里云 DashVector
- **稀疏索引**：BM25 / SPLADE，与稠密向量互补
- **选型维度**：数据规模、QPS、过滤能力（metadata filtering）、更新频率、是否需要多租户隔离、是否需要标量+向量混合查询

---

## 四、在线检索阶段（Retrieval）

### 4.1 查询理解与改写

- **Query Rewriting**：口语化→检索友好表达
- **多轮指代消解**：把「它的价格呢」补全成完整问题（对话式 RAG 必做）
- **Multi-Query**：一个问题生成 3–5 个变体并行检索再合并
- **RAG-Fusion**：多查询结果用 RRF（倒数排名融合）重排
- **Query Decomposition**：复杂问题拆成子问题分别检索
- **Step-Back Prompting**：先抽象成上位问题，再检索背景知识
- **HyDE**：让 LLM 先假想一个答案，用假答案的向量去检索
- **意图路由**：判断走知识库 / 走 SQL / 走 Web / 直接回答

### 4.2 检索方式

- **稠密检索**：语义相似，抗同义词，但对专有名词、编号、型号弱
- **稀疏检索（BM25）**：精确关键词命中，对术语和代码强
- **混合检索（Hybrid）**：两路召回 + RRF 或加权融合 —— **生产环境几乎必选**
- **元数据过滤**：时间范围、部门、权限、文档类型（前置过滤 vs 后置过滤有性能差异）
- **图检索**：知识图谱上做多跳关系推理

### 4.3 重排（Reranking）

- **为什么需要**：向量检索是双塔近似，Cross-Encoder 让 query 和文档做全交互，精度显著更高
- **常用模型**：BGE-Reranker、Cohere Rerank、Jina Reranker、Qwen3-Reranker、ColBERT（延迟交互，折中方案）
- **典型配置**：召回 top 50–100 → rerank 取 top 3–8
- **LLM-as-Reranker**：用小模型打分，质量高但贵
- **多样性控制**：MMR 去冗余，避免 top-k 全是同一段的复述

### 4.4 上下文构建

- **压缩**：LLMLingua、抽取式压缩、句子级过滤，去掉无关句
- **排序位置**：注意「Lost in the Middle」，把最相关的放开头和结尾
- **去重合并**：相邻块拼接还原上下文
- **预算管理**：给检索内容设 token 上限，留足生成空间
- **Prompt 模板要点**：明确「只依据以下资料回答」「不确定就说不知道」「引用编号」

---

## 五、生成与可信度

- **引用与溯源**：让模型输出 `[1][2]` 并映射回原文位置，是企业场景的硬需求
- **忠实性约束**：约束生成不超出资料范围（Groundedness）
- **拒答机制**：检索分数低于阈值时明确说「资料不足」
- **后校验**：用第二个模型或规则校验答案中的每句话是否有原文支撑（NLI 蕴含检验）
- **结构化输出**：JSON schema 约束，便于下游消费
- **流式与延迟**：先出检索结果占位，再流式生成，改善体感

---

## 六、进阶范式

| 范式 | 核心思想 | 适用场景 |
| --- | --- | --- |
| **Naive RAG** | 检索一次 → 生成 | 基线 |
| **Advanced RAG** | 加查询改写、混合检索、重排 | 生产标配 |
| **Modular RAG** | 各模块可插拔、可编排 | 复杂系统 |
| **Self-RAG** | 模型自己判断要不要检索、检索够不够、答案对不对 | 提升可靠性 |
| **Corrective RAG (CRAG)** | 评估检索质量，差则触发 Web 搜索兜底 | 开放域 |
| **Adaptive RAG** | 按问题难度选择路径（直答/单检索/多跳） | 成本优化 |
| **Agentic RAG** | Agent 自主规划多轮检索、调用多种工具 | 复杂研究型任务 |
| **GraphRAG** | 构建实体-关系图与社区摘要，支持全局性问题 | 「整个语料的主题是什么」这类问题 |
| **LightRAG / HippoRAG** | 轻量图索引，降低 GraphRAG 成本 | 中等规模 |
| **RAPTOR** | 递归聚类+摘要建成树，多粒度检索 | 长文档、跨章节综合 |
| **Memory-augmented RAG** | 引入长期记忆（Mem0、ReMe） | 个人助理、长期对话 |
| **多模态 RAG** | 文本+图表+视频统一检索（ColPali 等） | 财报、说明书、幻灯片 |

---

## 七、评测（做不好评测就没法优化）

### 7.1 分层指标

- **检索层**：Recall@k、Precision@k、MRR、NDCG、Hit Rate、Context Relevance
- **生成层**：Faithfulness（忠实度）、Answer Relevance、Answer Correctness、Completeness
- **端到端**：人工满意度、任务成功率、拒答准确率
- **工程层**：P95 延迟、单次成本、缓存命中率、吞吐

### 7.2 方法与工具

- **RAGAS**：无参考答案也能算忠实度/相关性，最常用
- **TruLens、DeepEval、Phoenix/Arize、LangSmith、Langfuse**：追踪 + 评测
- **BEIR / MTEB / C-MTEB**：embedding 与检索模型选型基准
- **合成评测集**：用 LLM 从自己的文档生成 QA 对，快速搭出几百条黄金集（务必人工抽检）
- **消融实验**：一次只改一个变量（分块大小 / 是否重排 / top-k），否则无法归因

<div class="callout callout-warning" markdown="1">
经验法则：**RAG 效果差，80% 出在检索而不是生成。**排查顺序应为：能不能召回正确片段 → 排序对不对 → 上下文全不全 → 最后才怀疑 prompt 和模型。
</div>

---

## 八、工程与生产问题

- **增量更新**：文档变更如何做增量 embedding、软删除、版本共存
- **权限控制**：行级/文档级 ACL，检索时按用户权限过滤（不能只在生成时过滤）
- **多租户隔离**：分 collection、分 partition 还是分 namespace
- **缓存**：语义缓存（相似 query 直接复用答案）、embedding 缓存、检索结果缓存
- **成本**：embedding 调用量、向量存储费用、rerank 与 LLM token 成本
- **延迟优化**：并行召回、异步 rerank、小模型预筛、提前流式返回
- **可观测**：全链路 tracing，记录 query→召回列表→rerank 分数→最终 prompt
- **安全**：Prompt Injection（文档里藏指令）、数据投毒、越权检索、输出泄漏
- **冷启动**：没有真实问题时先用合成问题跑通评测闭环

---

## 九、常见坑清单

- [ ] 只用向量检索，专有名词/型号/编号检索不到 → 加 BM25 混合
- [ ] 分块太大导致噪声多，太小导致语义断裂 → 用 Small-to-Big
- [ ] 表格被切碎，数字对不上 → 单独处理表格并加摘要
- [ ] 没有重排，top-k 里正确答案排在第 20 位 → 加 Reranker
- [ ] 多轮对话不做指代消解，第二轮就检索失败
- [ ] 检索不到也硬答 → 缺拒答阈值
- [ ] 中文用了英文为主的 embedding 模型
- [ ] 元数据没入库，无法按时间/部门过滤
- [ ] 文档更新后旧向量没删，答案自相矛盾
- [ ] 没有评测集，全靠「感觉变好了」

---

## 十、技术栈速查

| 环节 | 可选方案 |
| --- | --- |
| 编排框架 | LangChain / LlamaIndex / Haystack / AgentScope / DSPy |
| 文档解析 | Unstructured、MinerU、Docling、LlamaParse、PaddleOCR |
| Embedding | BGE-M3、Qwen3-Embedding、GTE、OpenAI v3、Voyage、Jina |
| 向量库 | Milvus、Qdrant、Weaviate、pgvector、Elasticsearch、Chroma |
| Rerank | BGE-Reranker、Cohere Rerank、Jina、ColBERT |
| 评测 | RAGAS、DeepEval、TruLens、Langfuse |
| 开箱平台 | Dify、RAGFlow、FastGPT、Coze、阿里云百炼、Anything LLM |

---

## 十一、建议的学习路径

1. **跑通最小闭环**（1 天）：几十个 PDF + 递归分块 + 一个 embedding 模型 + Chroma + 一个 prompt，先有能问答的东西。
2. **建评测集**（2 天）：合成 100–200 条 QA，接 RAGAS，得到基线分数。这一步比任何优化都重要。
3. **依次做单点优化并测量收益**：
    1. 加混合检索（BM25 + 向量）
    2. 加 Reranker
    3. 优化分块策略 + 元数据
    4. 加查询改写 / 多查询
    5. 加上下文压缩与引用标注
4. **上难度**：处理表格与扫描件、多轮对话、权限过滤、增量更新。
5. **进阶范式**：按需引入 GraphRAG（全局性问题）、RAPTOR（长文档综合）、Agentic RAG（多跳推理）。
6. **读经典材料**：RAG 综述（*Retrieval-Augmented Generation for LLMs: A Survey*）、原始 RAG 论文（Lewis et al., 2020）、DPR、ColBERT、Self-RAG、CRAG、RAPTOR、GraphRAG 论文，以及 Anthropic 的 Contextual Retrieval 博客。

<div class="callout callout-success" markdown="1">
一句话总结：RAG 的门槛在入门（一下午能跑通），天花板在检索质量与评测体系（几个月持续打磨）。把精力放在**分块、混合检索、重排、评测**这四件事上，就能覆盖大部分收益。
</div>
