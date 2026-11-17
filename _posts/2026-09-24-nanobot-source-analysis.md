---
layout: post
title: "nanobot 项目源码分析（LLM · Plan · Memory · Tool）"
date: 2026-09-24
categories: [ai]
mermaid: false
math: false
toc: true
---
<div class="callout callout-note" markdown="1">
**nanobot**（HKUDS / 港大数据智能实验室）是一个超轻量级个人 AI Agent 框架：核心约 **4,000 行 Python**，对标 OpenClaw 却缩减约 99% 代码，MIT 许可、约 46k stars。本页面聚焦 **LLM / Plan / Memory / Tool** 四个模块的源码级解读，源码基于 `main` 分支。
</div>

## 一、整体架构定位

nanobot 的设计哲学是 **"小内核 + 可扩展外围"**，官方类比为 "AI Agent 界的 Linux 内核"。核心运行流：

```javascript
Channel(CLI/WebUI/聊天应用) → MessageBus(InboundMessage)
  → AgentLoop(会话/工作区/上下文)
    → AgentRunner(provider/tool 循环)
      → Provider(LLM 后端) ⇄ Tools(files/shell/web/MCP/cron)
  → MessageBus(OutboundMessage) → Channel
```

关键的**双层分离**：

- `agent/loop.py`（**AgentLoop**）：面向渠道的一个回合——接收入站消息、选定会话与工作区、构建上下文、挂接 hooks、发布出站消息。
- `agent/runner.py`（**AgentRunner**）：面向模型的循环——调用 provider、处理流式增量、执行工具、把结果喂回模型、在得到终答或触及限制时停止。

> 调试口诀：渠道路由/会话/工作区问题看 `loop.py`；provider/工具/流式/迭代限制问题看 `runner.py`。

---

## 二、LLM 模块解读

**核心文件**：`nanobot/providers/base.py`（Provider 抽象）、`nanobot/agent/runner.py`（调用循环）、`nanobot/providers/*`（各家实现）。

### 2.1 统一的 Provider 抽象

`providers/base.py` 定义了三个核心数据类 + 一个抽象基类：

- `ToolCallRequest`：LLM 发出的工具调用请求（`id` / `name` / `arguments`）。
- `LLMResponse`：统一响应（`content` / `finish_reason` / `tool_calls` / `thinking_blocks` 等）。
- `GenerationSettings`：默认生成参数（`temperature=0.7`、`max_tokens=4096`、`reasoning_effort`）。
- `LLMProvider(ABC)`：所有后端的基类，唯一必须实现的抽象方法是 `chat()`。

```python
class LLMProvider(ABC):
    """Base class for LLM providers."""
    supports_progress_deltas = False
    _CHAT_RETRY_DELAYS = (1, 2, 4)
    _TRANSIENT_ERROR_MARKERS = ("429", "rate limit", "500", "502",
        "503", "504", "overloaded", "timeout", "connection", "server error", ...)

    @abstractmethod
    async def chat(self, messages, tools=None, model=None,
                   max_tokens=4096, temperature=0.7,
                   reasoning_effort=None, tool_choice=None) -> LLMResponse: ...
```

**设计要点**：

1. **最小抽象面**：子类只需实现 `chat()`；流式 `chat_stream()` 有默认实现（退化为一次性调用后把整段内容当作单个 delta 推送），原生支持流式的 provider 才覆写。这让新增一个 OpenAI 兼容后端成本极低。
2. **重试与容错内建在基类**：`chat_with_retry` / `chat_stream_with_retry` 通过 `_run_with_retry` 统一处理瞬时错误。`_TRANSIENT_ERROR_MARKERS` 用字符串标记识别 429/5xx/超时/限流；退避序列 `(1,2,4)` 秒，持续同错最多 10 次、最大延迟 60s。
3. **异常转错误响应**：`_safe_chat` 把未预期异常包装成 `LLMResponse(finish_reason="error")` 而非抛出，避免单次模型故障击穿整个回合。
4. **参数缺省归一**：`_SENTINEL` 哨兵让上层无需层层透传 `temperature/max_tokens`，未显式传入或显式 `None` 都会回落到 `self.generation` 默认值。

多后端：大多数托管服务走 `openai_compat_provider.py`；Anthropic、Azure、Bedrock、OpenAI Codex、GitHub Copilot 有专用路径（`factory.py` + `registry.py` 负责选择）。

### 2.2 AgentRunner —— "Provider ⇄ Tool" 循环

`runner.py` 是真正驱动 LLM 的地方。`AgentRunSpec` 封装单次执行配置（`initial_messages`、`tools`、`runtime`、`max_iterations`、`max_tool_result_chars` 等），`AgentRunResult` 返回结果（`final_content`、`tools_used`、`usage`、`stop_reason` 等）。

主循环骨架：

```python
for iteration in range(spec.max_iterations):
    # 1. 上下文治理：可修复/压缩历史但不移动持久化边界
    messages_for_model = self.context_governor(...) 
    # 2. 调用 provider（流式 / 进度流式 / 非流式三条路径）
    coro = spec.runtime.provider.chat_stream_with_retry(**kwargs, ...)  # 或 chat_with_retry
    # 3. 若有 tool_calls → 执行工具 → 结果作为 role="tool" 消息回填 → continue
    # 4. 否则产出最终回复 → 触发注入检查 → 结束
```

**值得学习的健壮性设计**：

- **迭代上限**：`max_iterations` 防止无限工具循环；预算耗尽时有 `finalization`（无工具的一次收尾调用）兜底给用户一个回答。
- **空响应/长度截断恢复**：`_MAX_EMPTY_RETRIES=2`、`_MAX_LENGTH_RECOVERIES=3`，对空内容或被 `length` 截断的响应自动重试/续写。
- **畸形工具调用清洗**：若 LLM 返回 `name` 缺失/非字符串的 tool_call，`_drop_malformed_tool_calls` 会**在执行前、写入历史前**统一丢弃，避免坏消息被回放而"永久卡死"会话（原注释明确指出会触发 `tool_use.name: Input should be a valid string`）。全部畸形则先重试、再退化为无工具请求。
- **超时**：`NANOBOT_LLM_TIMEOUT_S`（默认 300s，0 可关闭）防止 LLM 挂起导致会话锁饥饿；流式另有更宽松的 wall-clock 超时与 `NANOBOT_STREAM_IDLE_TIMEOUT_S` 空闲超时。
- **中途注入（injection）**：`_try_drain_injections` 允许在工具执行后、终答后等检查点插入新用户消息或"目标继续"提示，是长任务/多设备连续对话的关键机制（见 Plan 模块）。
- **并发工具**：`_partition_tool_batches` 按 `tool.concurrency_safe` 把只读工具分批并行。

---

## 三、Plan 模块解读

> ⚠️ 重要结论：**nanobot 没有一个独立的 "Planner" 模块**，它刻意不做显式的"任务规划树"。"规划"能力由三部分组合实现：**① AgentRunner 的迭代 ReAct 循环 ② 持续目标（Sustained Goal）机制 ③ 子代理（Subagent）委派**。这正是它"小核心"哲学的体现——把规划交给 LLM 自身，框架只提供"持续推进"与"任务分解"的脚手架。

### 3.1 隐式规划：ReAct 迭代循环

如上节所述，规划本质上是 `runner.py` 的 `for iteration in range(max_iterations)` 循环：模型自己决定"思考 → 调用工具 → 观察结果 → 再决策"，框架不预先生成计划，而是让每一步的工具结果驱动下一步。

### 3.2 持续目标（Sustained Goal）

`agent/loop.py` 里通过目标状态实现"长时间跨回合推进"。核心是一个惰性计算的 `_goal_continue()`：

```python
def _goal_continue() -> str | None:
    _goal_lines = goal_state_runtime_lines(session.metadata ...)
    if not _goal_lines:
        return None
    return ("You have an active sustained goal:\n\n"
        + "\n".join(_goal_lines)
        + "\n\nPlease continue working toward the objective using your tools, "
          "or call update_goal with action='complete' if the work is truly finished.")
```

配合 runner 的 `goal_active_predicate` 与 `goal_continue_message`：当一个回合结束但目标仍活跃时，框架**自动注入"继续朝目标推进"的提示**，让 agent 跨多回合持续工作，直到模型主动 `update_goal(action='complete')`。目标状态持久化在 `session.metadata`，因此重启/多回合仍然有效。用户侧通过 `/goal` 显式激活（`goal_permission.py` 负责激活与权限边界）。

### 3.3 子代理委派（Subagent）

`agent/subagent.py` 的 `SubagentManager` 实现"把子任务分解到后台独立执行"：

```python
async def spawn(self, task: str, label=None, ...) -> str:
    task_id = str(uuid.uuid4())[:8]
    bg_task = asyncio.create_task(self._run_subagent(task_id, task, ...))
    self._running_tasks[task_id] = bg_task
    return f"Subagent [{label}] started (id: {task_id}). I'll notify you when it completes."
```

**关键设计**：

- **隔离的工具环境**：`_build_tools` 通过 `ToolLoader().load(ctx, registry, scope="subagent")` 为子代理构建独立、受限的 `ToolRegistry`（工作区沙箱），避免子任务越权。
- **后台异步**：每个子代理是一个 `asyncio.Task`，主 agent 不阻塞；`max_concurrent_subagents` 限制并发数。
- **结果回注**：子代理完成后不是直接返回，而是把结果作为一条 `injected_event: subagent_result` 的 system 消息 **publish 回主 agent 的会话队列**（用 `session_key_override` 对齐统一会话），从而作为"中途注入"触发主 agent 继续——与 3.2 的注入机制共用同一条通道。
- **可按会话取消**：`cancel_by_session` 支持整会话级别的子代理取消。

---

## 四、Memory 模块解读

**核心文件**：`nanobot/agent/memory.py`。开头注释直接点明三个角色：**"纯文件 I/O 存储 + 轻量 Consolidator + Dream 处理器"**。

### 4.1 两级存储

| 存储 | 位置 | 作用 |
| --- | --- | --- |
| Session（短期） | `<workspace>/sessions/*.jsonl` | 近期对话轮次，回放进上下文 |
| Memory（长期） | `<workspace>/memory/MEMORY.md` | 长期事实，人类可读的 Markdown |
| 合并源历史 | `<workspace>/memory/history.jsonl` | 追加式、供整理的原始历史 |

`MEMORY.md` 通过 `read_memory()` 读取，并以 `## Long-term Memory\n...` 段落注入上下文。`history.jsonl` 通过 `append_history()` 追加，单条有硬上限 `_HISTORY_ENTRY_HARD_CAP = 64_000` 字符防止爆炸。

### 4.2 Consolidator —— 轻量、token 预算触发

```python
class Consolidator:
    """Lightweight consolidation: summarizes evicted messages into history.jsonl."""
    _MAX_CONSOLIDATION_ROUNDS = 5
    _SAFETY_BUFFER = 1024  # tokenizer 估算漂移的额外余量
```

当会话上下文超出 token 预算时，被"驱逐"的旧消息会被总结后写入 `history.jsonl`，而不是直接丢弃。若总结失败，会**降级为 "[RAW] 原始归档"**（`Memory consolidation degraded: raw-archived N messages`）——保证信息不丢、宁可原样存。此外注意 `strip_think` 的处理：清洗后即使内容为空也持久化空串，避免思考内容在历史回放中泄漏。

### 4.3 Dream —— 重量级、cron 调度的两阶段整理

```python
class Dream:
    """Two-phase memory processor: analyze history.jsonl, then edit files via AgentRunner.
    Phase 1 produces an analysis summary (plain LLM call).
    Phase 2 delegates to AgentRunner with read_file / edit_file tools ...
    """
```

Dream 是 nanobot 记忆系统最有特色的设计：

- **阶段一（分析）**：对 `history.jsonl` 做一次普通 LLM 调用，产出分析摘要。
- **阶段二（落盘）**：把摘要**交给一个带 **`**read_file**`** / **`**edit_file**`** 工具的 AgentRunner**，让 agent 自己去编辑 `MEMORY.md` 等记忆文件。也就是说——**记忆整理本身就是一次 Agent 运行**，而非硬编码的字符串拼接。
- **调度**：由运行时在 `agents.defaults.dream.enabled` 为真时按 cron 周期触发（gateway 后台任务）。

> 借鉴点：Consolidator（实时、预算触发、保底原样归档）+ Dream（周期、LLM 驱动、可读文件）是一套"短期不丢 + 长期沉淀"的分层记忆范式。

---

## 五、Tool 模块解读

**核心文件**：`nanobot/agent/tools/base.py`（`Tool` / `Schema` / `ToolResult` 抽象）、`registry.py`（注册与校验）、以及 `shell.py` / `filesystem.py` / `web.py` / `cron.py` / `spawn.py` 等具体工具。

### 5.1 Tool 抽象基类

```python
class Tool(ABC):
    @property @abstractmethod
    def name(self) -> str: ...
    @property @abstractmethod
    def description(self) -> str: ...
    @property @abstractmethod
    def parameters(self) -> dict: ...          # JSON Schema
    @abstractmethod
    async def execute(self, **kwargs) -> Any: ...

    @property
    def read_only(self) -> bool: return False  # 无副作用 → 可并行
    @property
    def concurrency_safe(self) -> bool:
        return self.read_only and not self.exclusive
```

**设计要点**：

- **四要素契约**：`name` / `description` / `parameters(JSON Schema)` / `execute`。`to_schema()` 直接产出 OpenAI function-calling 格式，`name/description/参数` 就是给模型看的 "接口文档"。
- **并发标记**：`read_only` + `exclusive` 推导出 `concurrency_safe`，供 runner 的 `_partition_tool_batches` 决定能否并行执行——只读工具批量并发，写工具串行。
- **结果类型 **`**ToolResult**`：继承自 `str`，但带 `is_error` 结构化状态；`ToolResult.error(...)` 用于失败。既兼容"字符串直接喂回模型"，又能携带错误语义。
- **声明式 Schema**：`@tool_parameters({...})` 类装饰器把 JSON Schema 绑到类上并自动注入 `parameters` 属性，免去手写 property，且每次访问返回深拷贝防止污染。

### 5.2 参数校验与类型转换

`base.py` 里的 `Schema` 类实现了一个**自研的轻量 JSON Schema 校验器** `validate_json_schema_value`：支持 `type` / `enum` / `minimum/maximum` / `minLength/maxLength` / `minItems/maxItems` / 嵌套 `object.properties` / `array.items` / `required` / `additionalProperties`，并对 `bool` 伪装成 `int` 等边界做了处理。

配合两步预处理：

5. `cast_params` → `_cast_value`：按 schema 做**安全类型转换**（如字符串 `"true"` → bool、嵌套对象递归转换）。
6. `validate_params`：转换后再校验，返回错误列表（空即通过）。

### 5.3 ToolRegistry —— 注册、排序、校验、执行

```python
class ToolRegistry:
    def register(self, tool): self._tools[tool.name] = tool; self._cached_definitions = None
    def get_definitions(self):  # 内建工具排序在前，MCP 工具排序在后 → 缓存
        ...
    def prepare_call(self, name, params):
        """Resolve, cast, and validate one tool call."""
        tool = self._tools.get(name)
        if not tool: return None, params, f"Error: Tool '{name}' not found. Available: ..."
        cast_params = tool.cast_params(params)
        errors = tool.validate_params(cast_params)
        if errors: return tool, cast_params, f"Error: Invalid parameters ..."
        return tool, cast_params, None
```

**设计要点**：

- **稳定排序 + 缓存**：`get_definitions()` 把内建工具作为稳定前缀排序、MCP 工具排在后面并缓存，目的是让**发送给模型的工具列表顺序稳定**，利于 provider 侧的 prompt 缓存命中。任何 register/unregister 都会使缓存失效。
- **prepare_call 三步走**：解析 → 转换 → 校验，错误直接返回给模型的**文本提示**（含可用工具名/正确调用格式），而不是抛异常——让模型能自我纠错重试。
- **发现机制**：内建工具从 `agent/tools/` 扫描注册（`ToolLoader`），外加 MCP 工具、插件 entry point；子代理用独立 registry 做能力隔离。

内建工具家族：`filesystem.py`（读写/patch，限定工作区）、`shell.py`（带安全护栏的命令执行）、`web.py`（搜索/抓取，带 SSRF 校验的 `_validate_url`）、`cron.py`（定时，支持一次性 `at`）、`spawn.py`（子代理）、`message.py` 等。

---

## 六、对 OfficeAgent 的可借鉴点

- **LLM 层的 Provider 抽象 + 内建重试/超时/畸形调用清洗**：值得直接借鉴到你"虚拟员工"调用底座，尤其是"畸形 tool_call 在写入历史前丢弃"这一防卡死细节。
- **Plan 的"不做显式 Planner"哲学**：用 ReAct 迭代 + Sustained Goal + Subagent 组合，非常契合你"总前台/对接人分诊 + 领域员工执行"的两级路由——子代理的"隔离工具 registry + 结果回注"可对应你的"领域员工受限能力 + 结果汇总"。
- **Memory 的 Consolidator + Dream 分层**：契合你"代码作为事实源 + Markdown 补充业务意图"的记忆沉淀；Dream"用带 edit_file 工具的 Agent 自己整理记忆文件"的思路尤其新颖。
- **Tool 的四要素契约 + 声明式 Schema + 稳定排序缓存**：直接支撑你 P0"只读代码/受控文档问答"——用 `read_only` 标记即可天然实现"只读能力边界"与并发安全。

---

<div class="callout callout-note" markdown="1">
**一句话总结**：nanobot 的四个模块都贯彻同一条原则——*框架只提供最小、稳定、可组合的脚手架，把"智能"尽量交还给 LLM 本身*。LLM 层做统一抽象与容错，Plan 层用迭代+目标+子代理代替显式规划，Memory 层用"实时保底 + 周期 Agent 整理"分层沉淀，Tool 层用严格 Schema 契约 + 并发标记保证安全与可扩展。
</div>
