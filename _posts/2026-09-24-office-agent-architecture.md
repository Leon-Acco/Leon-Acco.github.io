---
layout: post
title: "Office_Agent 设计文档（架构与 Agent 实现细节）"
date: 2026-09-24
categories: [iot]
mermaid: true
math: false
toc: true
---
> Office_Agent 设计文档 · 对应代码快照 `4562f5c6` · 仓库 `Leon-Acco/office-agent`

这份文档写给需要接手、评审或参考这套系统的人。我会把架构、Agent 运行时、检索、记忆、规划这几块的实现讲清楚，包括我当时为什么这么选、代价是什么、哪些地方我自己也不满意。

<div class="callout callout-note" markdown="1">
**📝 📐**

**几个阅读约定**
- 写成「实现」的，都能在指定文件里直接对上，包括常量、正则、默认参数。
- 写成「取舍」的，是我当时的决策理由，不一定是最优解，但背后的权衡我会说明白。
- 写成「已知不足」的，是我自己已经识别、但还没改的问题，不必客气。
- 对比章节只讲机制差异，不给其他方案打分。我选了一条路，不代表别的路不对，只是场景不同。
- 性能相关的判断都是我从代码推的，没做过压测，看的人请按此折扣。
</div>

---

## 零、一张图看懂整个设计

先把脉络给出来。这套系统就是**一条主链路 + 四根横向支柱**：提问经分诊拿到身份，装配上下文，进 ReAct 循环，循环里靠工具去检索；而岗位包、作用域、Hook、记忆这四根支柱全程约束它。

```mermaid
flowchart LR
    U["用户提问"] --> D["① 分诊<br>L1 关键词 → L2 仓库名 → L3 LLM"]
    D --> C["② 上下文装配 build_context<br>历史 10 条 + 技能摘要<br>+ 仓库绑定 + 岗位包"]
    C --> L["③ ReAct 循环<br>思考 → 调工具 → 观察"]
    L --> R["④ 检索工具<br>知识库 / 资源 / 代码 / 仓库文档 / MCP"]
    R --> L
    L --> F["⑤ 收敛成答<br>预算耗尽则 finalize 兜底"]
    F --> O["SSE 流式输出 + 审计落库"]
    O --> COL{"跨领域？"}
    COL -- 是 --> RM["协作会议室<br>拆解 → 并行草案 → 互评 → 合成"]
    P["支柱一：岗位包 RolePack<br>工具白名单 + 四维预算"] -. 决定能调什么、能跑多久 .-> L
    S["支柱二：作用域收敛<br>_RepoScopeMixin"] -. 决定能查哪个仓库 .-> R
    HK["支柱三：Hooks<br>注入检测 / 脱敏 / 审计"] -. 每次工具调用前后 .-> L
    M["支柱四：记忆<br>Message 表 + 上下文压缩<br>+ 已审核知识库"] -. 供给与回写 .-> C
```

如果只看一张图，看上面那张就够了。下面这张是分层视角的同一件事，方便定位代码：

```mermaid
graph TB
    subgraph L5["接入层"]
        A1["routers/frontdesk.py<br>对话 + 分诊 + SSE"]
        A2["routers/tasks.py<br>协作会议室"]
        A3["routers/repos.py<br>仓库管理"]
    end
    subgraph L4["编排层"]
        B1["runtime/runner.py<br>ReAct 主循环"]
        B2["runtime/context.py<br>上下文 + 预算"]
        B3["runtime/consolidator.py<br>上下文压缩"]
        B4["runtime/hooks.py<br>安全 + 审计切面"]
    end
    subgraph L3L["能力层"]
        C1["tool_registry.py + tool_base.py<br>工具契约与校验"]
        C2["tools/ 内置十个工具"]
        C3["mcp_client.py<br>外部 MCP 工具"]
    end
    subgraph L2L["接入与存储"]
        D1["runtime/provider.py<br>协议翻译 / 重试 / 并发闸"]
        D2["services/git_service.py"]
        D3["MySQL / Redis / workspaces/"]
    end
    L5 --> L4 --> L3L --> L2L
```

---

## 一、系统概览

### 1.1 部署形态

我把整套东西做成了单进程 FastAPI 应用，固定 8000 端口，REST、SSE 和静态前端都在同一个进程里。没拆微服务，因为当前规模下拆开只会增加部署和调试成本。

```mermaid
graph TB
    BR["浏览器 · 无构建 SPA"] -->|"fetch / EventSource"| API
    subgraph API["FastAPI :8000 单进程 · 含静态文件托管"]
        E1["/api/frontdesk/*<br>对话与分诊（SSE）"]
        E2["/api/tasks/*<br>协作会议室"]
        E3["/api/repos/*<br>代码仓库管理"]
        E4["/api/knowledge/*<br>知识条目"]
        E5["/api/admin/*<br>组织与员工"]
        E6["/api/governance/*<br>岗位包 / 变更单 / 策略"]
        E7["/api/graph · /api/dashboard<br>/api/agents · /api/auth"]
    end
    API --> DB["MySQL 8<br>aiomysql + SQLAlchemy 2.0 async"]
    API --> RD["Redis<br>redis.asyncio"]
    API --> FS["本地文件系统<br>workspaces/repo_id/（git clone 落地）"]
    API --> LLM["HTTPS → open.bigmodel.cn/api/anthropic<br>Anthropic Messages 协议"]
    API -.->|"lifespan 拉起"| SCH["repo_scheduler<br>60s 轮询定时 pull"]
```

**实现**：`main.py` 的 lifespan 里我只做三件事——`init_db()` 建表、拉起后台任务 `repo_scheduler.auto_refresh_loop()`、关闭时释放 Redis 连接。

### 1.2 关键配置

当前版本 `APP_VERSION = "0.2.0"`，配置集中在 `backend/config.py`：

| `LLM_TEMPERATURE` | `0.7` | — |
| --- | --- | --- |
| `LLM_MAX_TOKENS` | `16384` | 推理模型的思考过程也吃输出 token，我设 4096 时碰到过正文被截成空的情况 |
| `LLM_BASE_URL` | `https://open.bigmodel.cn/api/anthropic` | 走 Anthropic 兼容端点 |
| `LLM_MODEL` | `glm-5.2` | — |
| 配置项 | 默认值 | 我为什么设这个值 |
| `LLM_TIMEOUT` | `60` 秒 | — |
| `LLM_CONCURRENCY` | `2` | 我宁可慢，不想被上游限流打断 |
| `WORKSPACE_ROOT` | `<项目根>/workspaces` | 仓库 clone 落地目录 |
| `USE_LANGCHAIN` / `USE_MULTI_AGENT` | `false` / `false` | LangGraph 旁路开关，默认关 |

**已知不足（优先级最高）**：我把 MySQL 密码、Redis 密码、LLM API Key 写成了明文默认值，而仓库是公开的。这三项已经算泄露，我需要轮换它们，并把逻辑改成「拿不到环境变量就启动失败」。本文不复述具体值。

---

## 二、代码结构

| 路径 | 职责 | 关键实现点 |
| --- | --- | --- |
| `backend/main.py` | 应用装配 | 10 个 router、静态托管、lifespan 启动调度器 |
| `backend/config.py` | 配置 | 环境变量读取 |
| `backend/database.py` | 数据库 | async engine、`async_session`、`get_db` |
| `backend/runtime/provider.py` | LLM 接入层 | 协议翻译、重试、并发闸、流式解析 |
| `backend/runtime/context.py` | 上下文装配 | `AgentContext`、`Budget`、`build_context`、`build_system_prompt` |
| `backend/runtime/runner.py` | ReAct 主循环 | `execute` / `execute_stream` |
| `backend/runtime/tool_registry.py` | 工具注册与调用 | 定义缓存、参数转换与校验 |
| `backend/runtime/tool_base.py` | 工具契约 | `Tool` 四要素 + `ToolResult` |
| `backend/runtime/tools/` | 内置工具 | 业务检索 + 代码检索 |
| `backend/runtime/mcp_client.py` | MCP 接入 | SSE transport、工具名前缀、会话生命周期 |
| `backend/runtime/hooks.py` | 切面 | `AuditHook`、`SecurityHook` |
| `backend/runtime/consolidator.py` | 上下文压缩 | token 估算、摘要、降级截断 |
| `backend/runtime/subagent.py` | 子代理 | 已实现，主路径未接入 |
| `backend/routers/frontdesk.py` | 对话入口 | 三级分诊 + SSE |
| `backend/routers/tasks.py` | 协作会议室 | 拆解 / 并行 / 互评 / 合成 |
| `backend/routers/repos.py` | 仓库 REST | clone / 搜索 / 分支 / 定时刷新 / 删除 |
| `backend/services/git_service.py` | Git 封装 | 子进程调 git CLI，路径安全 |
| `backend/services/repo_scheduler.py` | 定时任务 | 60s 轮询，到期 `git pull --ff-only` |
| `backend/agents/` | LangGraph 旁路 | 默认关闭 |

**关于 **`**backend/agents/**`：这里面是我早期用 LangGraph 跑通的一版实现，后来生产路径全部换成了自研运行时。我在 `multi_agent.py` 文件头写了废弃声明，但没删，因为它作为对照组还有参考价值。接手的人不要在它上面做新开发。

### 为什么最后没用 LangGraph

不是框架不好，是我需要的三件事恰好是它抽象掉的那一层：

| 我需要的 | 自研运行时里的做法 | 旁路版本的状态 |
| --- | --- | --- |
| 每个员工工具集不同 | 按岗位包白名单动态构建 Registry | `get_all_tools()` 全量给出 |
| 四维预算 | 步数/调用数/token/超时显式计数 | 只有 `recursion_limit` |
| 会话历史 | 从 `Message` 表按会话加载 | 我当时写的是 `history=[]  # TODO` |
| 仓库/领域作用域 | 下沉到工具实例构造期 | 无 |
| 自定义 SSE 事件 | 直接向外吐 thinking / tool.start / tool.result | 需适配 astream 事件 |

自己写循环的代价是要自己处理很多边界条件，这部分工作量不小。但因为权限边界和成本控制是这个产品的核心诉求，我认为这笔花得值。

---

## 三、LLM 接入层

这一层离外部不确定性最近，我在这里花的时间最多，坑也踩得最多。

### 3.1 协议翻译

**取舍**：内部历史统一用 OpenAI Chat Completions 结构，只在发 HTTP 前翻译成 Anthropic Messages。每次请求多一道转换，换来的是换模型供应商时只改 provider 一层，运行时和工具层完全无感知。

| `role: "system"` | 抽为请求体顶层 `system`，多条拼接 | `to_anthropic_messages` |
| --- | --- | --- |
| 内部（OpenAI） | 发送（Anthropic） | 处理函数 |
| `assistant.tool_calls[]` | `tool_use` 内容块（带 id / name / input） | 同上 |
| `role: "tool"` | user 消息内的 `tool_result` 块 | 同上 |
| 连续多条 tool 结果 | 合并进同一条 user 消息 | 同上 |
| 相邻同角色消息 | 强制合并（Anthropic 要求严格交替） | 同上 |
| `tools[].function` | `{name, description, input_schema}` | `to_anthropic_tools` |
| `stop_reason` | end_turn/stop_sequence→stop，tool_use→tool_calls，max_tokens→length | `_STOP_REASON_MAP` |

### 3.2 重试：一个我踩过的坑

**实现**：退避序列 `_RETRY_DELAYS = (1, 2, 4, 8)`，最多 4 次重试。`_is_transient_error()` 按异常**类型**判定：`httpx.TimeoutException` 和 `TransportError` 一律算瞬时，`HTTPStatusError` 仅 429 或 ≥500 算瞬时。

早期我是靠错误消息关键词匹配来判断能不能重试的，结果协作会议室总出现 `[LLM 调用失败: ]` 这种错误信息为空的情况。查下去才发现 httpx 的超时异常 `str(e)` 就是空字符串，所以它永远匹配不上任何关键词，被当成不可重试直接抛了。改成类型判定后这类失败基本消失。我同时加了 `_err_text()`，`str(e)` 为空时自动补上异常类名，不再让空错误流到前端。

这个坑值得写出来：靠异常消息文本做控制流判断本身就不可靠，应该一开始就用类型。

### 3.3 并发闸

**实现**：模块级 `_LLM_SEMAPHORE = asyncio.Semaphore(LLM_CONCURRENCY)`，`chat()`、`chat_stream()` 和 `services/llm.py` 的 `chat_completion()` 共用同一个信号量。另外用 `_get_shared_client()` 复用一个 `httpx.AsyncClient` 连接池。

**需要提醒看文档的人**：这个值默认是 2，它直接制约了协作会议室的真实并行度。就算 `asyncio.gather` 同时发起 5 个子任务，实际也是在信号量处排队，更像两路流水线。这是我为了不碰上游限流主动接受的，配额升级后应该同步放开。

### 3.4 流式解析

| 事件 | 我的处理 |
| --- | --- |
| `content_block_start` | 若为 `tool_use`，记下该 index 的 id 与 name |
| `content_block_delta` / `text_delta` | 逐 token 产出正文 |
| `content_block_delta` / `input_json_delta` | 按 index 累加 `partial_json` |
| `message_delta` | 读 `stop_reason` 与 usage |
| `error` | 转异常走重试判定 |

**取舍**：流式重试的边界我定在「是否已经向下游发过内容」。首个 chunk 产出前出错可以整次重试；已经发过内容再断就不重试了，宁可让用户看到一段不完整的回答，也不让他看到重复文本。

另外 `_drop_malformed_tool_calls()` 会在写回历史前丢掉 name 缺失或非字符串的 tool_call——这东西一旦进了历史，后面每一轮请求都会被它污染。

---

## 四、Agent 运行时

### 4.1 一次单聊请求走完的全链路

```mermaid
sequenceDiagram
    autonumber
    participant FE as 前端
    participant FD as frontdesk.py
    participant DB as MySQL
    participant CX as build_context
    participant RG as ToolRegistry
    participant AR as AgentRunner
    FE->>FD: POST /api/frontdesk/chat（SSE）
    FD->>FD: 分诊 _find_best_agent → _find_agent_by_repo → _llm_dispatch<br>（单聊入口跳过分诊，直接指定员工）
    FD->>DB: _get_or_create_session + _save_message(user)，立即 commit
    FD->>DB: _load_role_pack_spec：工具白名单 / 权限 / 预算
    FD->>CX: build_context
    CX-->>FD: history 最近 10 条（仅 user/assistant）<br>skills 直绑优先否则走 RolePack<br>repos 由 AgentRepoBinding 展开<br>docs 上限 20 · budget 四维
    FD->>RG: build_tool_registry（白名单 + 仓库作用域 + 默认领域）
    FD->>RG: load_mcp_tools(stack, allowed_tools)<br>stack 必须由调用方持有
    FD->>AR: execute_stream(...)
    AR-->>FE: thinking / tool.start / tool.result / answer.chunk
    AR-->>FD: answer.completed（可带 suggest_collab）
    FD->>DB: _save_message(assistant)，立即 commit
```

### 4.2 ReAct 主循环

这是整套系统的心脏，先看图：

```mermaid
flowchart TD
    S(["进入循环"]) --> B{"can_continue？<br>步数 / 调用数 / token / 超时"}
    B -- 否 --> FIN["finalize：追加收尾指令<br>tools=None，max_tokens=8000<br>再请求一次"]
    B -- 是 --> CON{"messages 超 32k token？"}
    CON -- 是 --> CMP["Consolidator 压缩<br>保留 system + 最近 6 条<br>中间段摘要为 500 token"]
    CON -- 否 --> CALL
    CMP --> CALL["provider.chat_stream<br>带上 registry.definitions"]
    CALL --> HAS{"返回含 tool_calls？"}
    HAS -- 是 --> HB["hooks.before_execute_tools<br>SecurityHook 可将参数改写为 _blocked"]
    HB --> EX["registry.execute 逐个执行<br>异常不上抛，转 ToolResult.error"]
    EX --> HA["hooks.after_execute_tools<br>截断 8000 字 + 脱敏 + 写 audit_log"]
    HA --> APP["messages += assistant（tool_calls）+ tool 结果"]
    APP --> B
    HAS -- 否 --> EMP{"正文为空？"}
    EMP -- 是 --> RT{"空响应重试 < 2？"}
    RT -- 是 --> CALL
    RT -- 否 --> FIN
    EMP -- 否 --> CLEAN["_strip_tool_name_spam 清洗"]
    CLEAN --> SHORT{"清洗后不足 200 字？"}
    SHORT -- 是 --> AGAIN["换更强指令再要一次<br>取更长的那份"]
    SHORT -- 否 --> OUT(["返回终答"])
    AGAIN --> OUT
    FIN --> OUT
```

对应的伪代码：

```javascript
for iteration in range(budget.max_steps):
    consolidator 检查是否需要压缩 messages
    resp = provider.chat/chat_stream(messages, tools=registry.definitions)

    if resp.tool_calls:
        hooks.before_execute_tools()      # SecurityHook 可改写参数为 _blocked
        for tc in tool_calls: registry.execute(tc)
        hooks.after_execute_tools()       # 脱敏 + 审计
        messages += [assistant(tool_calls), *tool_results]
        continue

    if 内容为空:
        空响应重试（上限 _MAX_EMPTY_RETRIES = 2）

    return 终答

# 循环跑满仍未终答：
finalize：追加收尾指令，tools=None、max_tokens=8000 再请求一次
```

几个我刻意定下的行为：

- 工具抛异常不往上传，统一转 `ToolResult.error(...)` 回填给模型。模型看到错误信息自己会换一种调法，比直接中断强。
- 预算耗尽不报错，而是走 finalize。用户宁可拿一段不完美的答案，不能拿一个错误页。
- 终答先过 `_strip_tool_name_spam()` 清洗。

### 4.3 三个专门处理模型行为异常的补丁

这三处都是被真实故障逼出来的，不是提前设计的：

| finalize 轮已经 `tools=None` 了，GLM 还想调工具，就把工具名当正文重复输出几百次 | 正则 `([A-Za-z_][\w.-]*)(?:\s+\1){5,}` 匹配并截断，只匹配标识符形 token，避免误伤回答里的 ASCII 图 | `runner.py` |
| --- | --- | --- |
| 现象 | 我的处理 | 位置 |
| 推理模型思考链吃满输出配额，正文为空 | `LLM_MAX_TOKENS` 提到 16384，结构化调用不低于 2000，并加空返回重试 | `config.py` / `runner.py` |
| 清洗完发现正文不足 200 字 | 换一条更强的指令再要一次，取更长的那份 | `runner.py` |

**说清楚一件事**：这三个补丁是针对 glm-5.2 的具体行为的，不是通用能力。换模型时必须重新验证，尤其是那个正则——如果回答本身就包含重复标识符的代码片段，它是会误伤的。

### 4.4 预算模型

| 字段 | 默认值 | 加载自 |
| --- | --- | --- |
| `max_steps` | 8 | `RolePack.spec.permission.budget.steps` |
| `max_calls` | 6 | `.calls` |
| `timeout_sec` | 60 | `.timeout` |
| `max_token` | 48000 | `.token` |

运行时用 `current_step` / `current_calls` / `current_token` 计数，统一由 `can_continue()` 判定。

**已知不足**：预算现在只拦截不反馈。模型不知道自己还剩几步，所以不会在剩一步时主动收束。把剩余预算写进每轮 system 提示是个成本很低的改进，我还没做。

### 4.5 System Prompt 的组装

我把它分成 Stable 和 Volatile 两段，前者尽量不变以利于上游 prompt 缓存。

```mermaid
graph LR
    subgraph ST["Stable 段 · 尽量不变，利于缓存"]
        direction TB
        S1["1 岗位身份"] --> S2["2 岗位职责"] --> S3["3 授权资源<br>仓库由绑定关系动态渲染"]
        S3 --> S4["4 AGENTS.md 原文<br>截断 4000 字"] --> S5["5 可用工具描述表"]
        S5 --> S6["6 技能摘要<br>上限 10 条，只放 name+description"]
        S6 --> S7["7 权限边界与执行预算"] --> S8["8 七条行动规则"] --> S9["9 工具调用 few-shot"]
    end
    subgraph VO["Volatile 段 · 每次变"]
        direction TB
        V1["已收集证据（最近 5 条）"] --> V2["会话上下文提示"]
    end
    ST --> P["最终 System Prompt<br>约 3000-6000 token（未实测）"]
    VO --> P
    SK["技能全文"] -. 要时才拉 loadSkill .-> P
```

**Stable 段**

1. 岗位身份：姓名、职称、部门、领域
2. 岗位职责
3. 授权资源：绑定仓库（`Repository.name` + `description` 动态渲染）+ 已上传文档清单（上限 20）
4. `AGENTS.md` 行为准则原文（截断 4000 字）
5. 可用工具描述表
6. 技能摘要：上限 10 条，只放 name + description；全文由 `loadSkill` 按需拉。只有 1 个技能且 instructions 不足 600 字时直接内联
7. 权限边界与执行预算
8. 七条行动规则：先查后答 / 工具优先 / 数据准确 / 领域边界 / 信息不足明确告知 / 回答详尽 / 整合加工
9. 工具调用 few-shot 示例

**Volatile 段**

10. 已收集证据（最近 5 条）
11. 会话上下文提示

**取舍**：仓库清单我没写进 `AGENTS.md`，而是从绑定关系动态渲染。手写的清单一定会随着绑定变更而漂移，而模型完全依赖这份清单判断自己能查什么。同理，技能只放摘要也是为了控住 prompt 长度。按这个结构，一个绑 2 仓库、5 技能的员工大致在 3000–6000 token 量级，没实测过。

### 4.6 工具契约

- `Tool` 抽象基类四要素：`name` / `description` / `parameters`（JSON Schema）/ `execute`。
- `ToolResult` 直接继承 `str`，额外带 `is_error`。这样结果可以当字符串直接喂回模型，又不丢错误语义。
- 并发属性：`read_only`（默认 True）、`exclusive`（默认 False）、`concurrency_safe = read_only and not exclusive`。
- `ToolRegistry.prepare_call()` 三步：查表 → `_cast_params()` 字符串转 bool/int/float → `_validate_params()` 必填校验。
- 工具不存在时我返回文本提示（附可用工具名）而不抛异常，让模型下一轮自己纠正。
- 定义列表按 name 排序后缓存，顺序稳定才能命中 prompt 缓存。

**已知不足**：`concurrency_safe` 我定义了但还没用上，runner 目前是顺序执行工具的。

### 4.7 Hook 切面

`AgentHook` 基类给了三个钩子点：`before_execute_tools` / `after_execute_tools` / `after_iteration`。

| `SecurityHook` | before | `_check_injection()` 检测 `ignore previous`、`system:`、两种路径遍历写法，命中就把参数整体换成 `{_blocked: True, _reason: ...}`，runner 见到就跳过 |
| --- | --- | --- |
| Hook | 时机 | 行为 |
| `SecurityHook` | after | 结果截断 8000 字；对 password/secret/token/api_key/apikey/access_key/private_key/credential 命中处，把其后 50 字符换成 `[REDACTED]` |
| `AuditHook` | after | 用独立 DB 会话写 `audit_log`：actor / action / target / detail / trace_id / decision |

**已知不足（两处）**：

12. Hook 目前只在 `execute_stream` 挂载，非流式 `execute` 路径既没审计也没脱敏。这是一致性缺口，应该补上。
13. 注入检测是英文字符串黑名单，中文表述的越权指令、编码变形、以及从工具返回内容里来的间接注入都拦不住。我不把这层当安全边界，它的主要价值是审计留痕。真正的边界在下一节。

---

## 五、工具层

### 5.1 业务工具

| `getEmployeeInfo` | `Agent` 及组织表 | — |
| --- | --- | --- |
| `searchKnowledge` | `KnowledgeCandidate` | SQL 侧状态过滤 + 领域双向包含，按 `created_at` 倒序取 5 条；Python 侧关键词过滤后取前 3 条；正文截 500 字 |
| 工具 | 数据来源 | 硬限制 |
| `searchResource` | `Resource` | 先名称命中，返回正文预览 2000 字；名称未命中则正文 LIKE，返回命中点前后各 150 字 |
| `loadSkill` | `Skill` | 按需返回全文 |

领域过滤用的是双向包含 `_domain_bidir_filter`，也就是 `domain.contains(d) OR literal(d).contains(domain)`，为的是让「后端」和「后端域」能互相匹配。

`searchResource` 里有一个防幻觉的处理：文档存在但正文读不出来时，我会显式回一段警告，要求模型不得根据文档名称或描述推测内容。加这段是因为我实测碰到过模型凭文件名把正文编出来。

### 5.2 代码工具

| `getProjectStructure` | 目录树 + 文件类型 Top5 + 技术栈推断 | 深度默认 2 上限 4；每层最多 15 项 |
| --- | --- | --- |
| `listFiles` | 列文件（`pattern="*"` 时只返回代码类扩展名） | 默认 50，接口上限 200，展示 30 |
| `getCodeExcerpt` | 按行号区间读 | 超 500KB 拒绝；默认 1–50 行 |
| `searchCode` | 遍历仓库逐行子串匹配（大小写不敏感） | 跳 `.git`、跳二进制、跳超 200KB 文件；默认 `max_results=20`，展示上限 15 |
| 工具 | 行为 | 限制 |
| `searchInDocs` | 只在文档类扩展名里搜 | 14 种扩展名白名单；超 200KB 跳过；命中 15、展示 10 |
| `cloneRepo` | 唯一写工具，`read_only=False` 且 `exclusive=True` | 超时 300s；成功后补写 `Repository` 表让它进入绑定体系，写表失败会 rollback 但不影响 clone 结果 |

技术栈推断是阈值启发式：`.py` 超 3 判 Python，`.java` 超 3 判 Java，`.js+.ts+.tsx` 超 3 判 JS/TS，`.vue` 有就判 Vue，`.go` 有就判 Go。不解析依赖清单，够用就行。

### 5.3 作用域收敛：我认为这是系统里最重要的一个设计

```mermaid
flowchart TD
    A["模型发起代码类工具调用"] --> B{"allowed_repos 为空？"}
    B -- 是 --> C["不限制<br>总前台自答 / 未绑定员工保持全局检索"]
    B -- 否 --> D{"传了 repo_id 吗？"}
    D -- 没传 --> E{"有 default_repo？"}
    E -- 有 --> F["自动填入默认仓库"]
    E -- 没有 --> G["返回提示，列出可访问仓库"]
    D -- 传了 --> H{"在白名单内？"}
    H -- 在 --> I["正常执行"]
    H -- 不在 --> J["返回 ToolResult.ok 纠错提示<br>引导改用授权仓库（不用 error）"]
    F --> I
    subgraph TRI["同一份授权信息出现在三处"]
        T1["工具描述"]
        T2["参数 schema（_repo_param_schema）"]
        T3["执行期校验（_resolve_repo）"]
    end
```

`_RepoScopeMixin` 在工具**实例化时**就注入 `allowed_repos` 与 `default_repo`，`_resolve_repo()` 三条分支：

14. `allowed_repos` 为空 → 不限制（总前台自答、未绑定员工保持全局检索）
15. 没传 `repo_id` 但有 `default_repo` → 自动填；没默认仓库就返回提示，列出可访问仓库
16. `repo_id` 不在白名单 → 返回一条纠错提示，引导它改用授权仓库

同时 `_repo_param_schema()` 会把当前授权仓库写进参数描述，`_required_with_repo()` 在有默认仓库时把 `repo_id` 从必填里拿掉。授权信息同时出现在工具描述、参数 schema、执行期校验三个地方。

**取舍**：越权时我返回的是 `ToolResult.ok(...)` 而不是 `error`。错误会消耗模型的容错预算，甚至让它直接放弃；提示则能让它在同一轮里改用正确仓库。代价是「越权」这个事实不会被标记为异常，审计里看不到 DENY。这一点我需要再确认一下 `AuditHook` 当前是怎么记的。

### 5.4 MCP 集成

- 用官方 `mcp` 包的 SSE transport，支持自定义 headers 鉴权。
- `Tool` 表一行 = 一个 MCP Server，筛选条件 `type='mcp' AND state='APPROVED' AND name IN allowed_tools`。白名单里出现 server 名就启用它全部远端工具，粒度到 server 不到单个工具。
- 工具名加 `mcp__<server>__<tool>` 前缀，非法字符用 `_FN_NAME_RE` 清掉，总长截 64，满足 `^[a-zA-Z0-9_-]{1,64}$`。
- **会话生命周期**：`load_mcp_tools(stack, ...)` 内部绝不自建 `async with`，会话必须由调用方传入的 `AsyncExitStack` 持有，且该 stack 要覆盖整个流式消费过程。否则 `list_tools` 一返回会话就关了，后面 `call_tool` 全打在死会话上。这是接 MCP 时最容易踩的坑，单拿出来提一句。
- 单个 server 连不上只记日志跳过，不拖垮整场对话。
- `MCPTool.read_only = False`：远端语义未知，我保守取值。代价是它们都不具备 `concurrency_safe`，将来做工具并发时会受限。

---

## 六、检索（RAG）

### 6.1 先把话说明白

这套系统**没有向量库，也没有 embedding 调用**。检索完全由模型在 ReAct 循环里主动调工具完成，匹配方式是 SQL `LIKE` 和 Python 子串比较。叫它 RAG 不太准确，准确的说法是 **Agentic 检索 + 关键词匹配**。

### 6.2 四条检索路径

```mermaid
flowchart LR
    M["模型在 ReAct 循环里自己决定查不查、查几次"] --> K & R & C & DD
    K["searchKnowledge"] --> K1["SQL 状态+领域过滤<br>→ created_at 倒序取 5 条<br>→ 内存子串过滤 → 3 条 × 500 字"]
    R["searchResource"] --> R1["先名称命中（2000 字预览）<br>否则正文 LIKE（前后各 150 字）"]
    C["searchCode"] --> C1["全仓库 rglob 逐行匹配<br>跳 .git / 二进制 / 超 200KB<br>→ 15 处 × 200 字"]
    DD["searchInDocs"] --> D1["限 14 种文档扩展名<br>命中 15 · 展示 10"]
    K1 --> W["❗ 已知不足：先取 5 条再过滤<br>第 6 条及更早的知识永远召不回"]
```

| 知识库 | SQL 状态+领域过滤 → 取最近 5 条 → 内存子串过滤 | `created_at` 倒序（不是相关性） | 3 条 × 500 字 |
| --- | --- | --- | --- |
| 路径 | 召回算法 | 排序依据 | 截断 |
| 资源文档 | 名称命中，失败降级为正文 LIKE | 无显式排序 | 2000 字预览或 150+150 字片段 |
| 代码 | 全仓库遍历逐行匹配 | 文件遍历顺序（不是相关性） | 15 处 × 200 字/行 |
| 仓库文档 | 同上，限定文档扩展名 | 同上 | 15 处，展示 10 |

**已知不足（我认为是当前最应该先修的一个）**：知识库检索是先按时间取 5 条、再做关键词过滤，而不是先过滤再排序取 top-k。后果是一旦某个领域下已审核知识超过 5 条，第 6 条及更早的在任何关键词下都召不回来。这不是范式选择的问题，就是逻辑顺序写反了，改回来十行代码的事。

### 6.3 代码检索的成本

`search_in_repo` 是 Python 层的 `rglob` + 逐行遍历，无索引无缓存。另外 `_attach_authors()` 会为结果里**每个唯一文件**额外跑两次 `git log` 子进程（创建人一次、最后修改人一次），带 15 秒超时。

加作者信息是为了回答「这文件谁写的」这类问题，产品上确实有用。但实现上我现在是对每个命中文件都查，而且走的是同步子进程，这两点都应该改。

### 6.4 与其他检索范式的机制差异

| 多跳问题 | 可多轮迭代 | 单轮，需额外设计 | 单轮 | 图遍历原生支持 | 依赖模型 |
| --- | --- | --- | --- | --- | --- |
| 精确标识符 | 直接命中 | 看分词与模型 | 能命中 | 一般 | 不适用 |
| 同义词查询 | 命中不了 | 能命中 | 能命中 | 能命中 | 不适用 |
| 匹配原理 | 字面子串 | 向量近邻 | 词频 + 向量加权 | 实体关系图遍历 | 无 |
| 触发方式 | 模型自己决定查不查、查几次 | 每次提问固定前置检索 | 同向量 RAG | 同向量 RAG | 无检索 |
| 维度 | 本项目 | 向量 RAG | 混合检索 | GraphRAG | 长上下文直塞 |
| 索引构建 | 无 | 需切块+向量化 | 需两套索引 | 需实体抽取与建图 | 无 |
| 内容更新延迟 | `git pull` 后立即生效 | 需重建索引 | 需重建索引 | 需重建图 | 无 |
| 单次查询成本 | 多轮 LLM 调用，token 高 | 1 次检索，token 低 | 低 | 中 | 输入 token 极高 |
| 结果可追溯 | 工具链与文件行号可见 | 相似度分数 | 同左 | 路径可见 | 不可追溯 |
| 主要失效模式 | 用户用了文档里没有的说法 | 语义相近但答案错的片段 | 权重调参 | 建图质量 | 长上下文信息遗漏 |

**我的判断，分两个场景说**：

- **代码问答**：字面匹配在这里确实合适。找 `createOrder`，grep 比 embedding 准，而且仓库一 pull 就生效，不用维护索引。当前的短板不在范式，在无索引的耗时。
- **业务知识问答**：字面匹配的短板很明显。用户不会用和知识条目一模一样的措辞，而知识库条目本来就少（因为要过人工审核），两相叠加召回失败率高。这块是我接下来要动的。

### 6.5 我的改进计划

| 优先级 | 改进 | 预估改动 |
| --- | --- | --- |
| 1 | 知识库改为「SQL 带关键词条件全表过滤 → 排序 → top-k」 | 十行内 |
| 2 | 代码检索改走 `git grep` 或 ripgrep 子进程 | 单函数 |
| 3 | `_attach_authors` 改异步，且只对展示的前 N 条富化 | 单函数 |
| 4 | 知识库加 MySQL 全文索引（ngram 分词器支持中文），现有过滤作兜底 | 需 DDL 变更 |
| 5 | 引入轻量向量检索做混合召回 | 新依赖，待评估 |
| 6 | 把 `commit_sha` 锁定真正用起来（`AgentContext` 预留了字段但检索结果还没与具体 commit 绑定） | 中等 |

---

## 七、记忆（Memory）

```mermaid
flowchart TB
    subgraph SHORT["短期 · 单次请求内"]
        W["内存 messages[]<br>包含完整工具轨迹"]
        EV["AgentContext.evidence_cards"]
    end
    subgraph MID["中期 · 单会话"]
        MSG["Message 表<br>role / content / evidence_ids / feedback"]
    end
    subgraph LONG["长期 · 全局"]
        KC["KnowledgeCandidate 表<br>人工提交 + 人工审核"]
    end
    W -->|"超 32k token 触发压缩<br>保留 system + 最近 6 条<br>中间段摘要 500 token"| W
    W -->|"只持久化 user / assistant 正文<br>工具轨迹丢弃"| MSG
    MSG -->|"下次请求取最近 10 条反转为正序"| W
    MSG -.->|"人工提交，无自动沉淀路径"| KC
    KC -->|"只有 APPROVED 参与检索"| W
    N1["❗ 压缩结果不写回数据库"] -.- W
    N2["❗ 10 条上限硬编码，超出无声丢失"] -.- MSG
```

### 7.1 四类状态的载体

| 类型 | 载体 | 作用域 | 持久化 |
| --- | --- | --- | --- |
| 会话历史 | `Message` 表 | 单 `session_id` | 是 |
| 本轮工作记忆 | 内存 `messages[]` | 单次 ReAct 循环 | 否 |
| 证据卡 | `AgentContext.evidence_cards` | 单次请求 | 否（`Message.evidence_ids` 有字段） |
| 长期知识 | `KnowledgeCandidate` 表 | 全局 | 是 |

`Message` 表字段：`id`、`session_id`、`role`、`content`、`agent_id`、`evidence_ids`(JSON)、`confidence`、`feedback`（up/down/空）、`created_at`。

历史装配：`build_context` 按 `session_id` 查、`created_at` 倒序取 `history_limit`（默认 10）条后反转为正序，只注入 user 和 assistant。

**这个选择带来三个后果，看文档的人需要知道**：

17. 工具调用轨迹不进下一轮上下文。模型第 N 轮查到的东西，第 N+1 轮只能靠自己上一条回答的文本回忆，看不到原始工具返回。多轮追问时它会重复检索。
18. 10 条上限是硬编码的，不随岗位包变。超过 10 轮必然丢早期信息，而且丢得惄无声息。
19. `Message.role` 允许 `system`，但注入时被我过滤掉了。

第 1 和第 2 都是已知不足，`history_limit` 应该做成岗位包可配，工具轨迹至少应该以摘要形式保留。

### 7.2 上下文压缩

| 触发阈值 | `max_tokens = 32000` |
| --- | --- |
| token 估算 | `estimate_tokens()`：中文字符 ×2，其他 ×0.3，每条 +4。纯启发式，不调 tokenizer |
| 环节 | 实现 |
| 分区 | system 全保留；最近 `_KEEP_RECENT = 6` 条全保留；中间部分送压缩 |
| 压缩 | 调 LLM 生成摘要，`_SUMMARY_MAX_TOKENS = 500`，`temperature = 0.1` |
| 注入 | 摘要作为一条消息插回，前缀 `[consolidated] ` |
| 失败降级 | 不重试不报错，改成每条截断至 200 字并附 `...[truncated]` |

**取舍**：压缩本身也要调 LLM，而调 LLM 就可能失败。我宁可在失败时退化成笨办法（截断），也不让整个对话因为压缩失败而中断。

**补充两点**：一是压缩发生在 runner 循环内、作用于内存的 `messages[]`，**结果不写回数据库**，下次请求仍从 `Message` 表重新拉原文。二是「中文 ×2」这个估算偏保守，会让压缩比实际需要更早触发；但因为会话历史本来只有 10 条，32000 在单聊里很难达到，压缩实际主要在工具返回体积大的时候生效。

### 7.3 长期记忆：我选了人工审核制

`KnowledgeCandidate` 的治理态机是 `SUBMITTED → IN_REVIEW → APPROVED / REJECTED → EXPIRED`，字段含 `body_md`、`evidence_ids`、`commit_sha`、`expires_at`、`scope`（PROJECT/DEPARTMENT/COMPANY）、`confidence`、`conflict_warning`。检索工具只查已通过审核的条目。

**取舍**：系统里没有任何自动沉淀路径，Agent 不会把对话结论自动写成知识。这是我故意的。企业内部知识一旦沉淀错了，后面所有人都会被误导，而且很难发现。我宁愿知识库长得慢一点。

代价也很实在：沉淀速度完全取决于人工。如果审核环节没人做，长期记忆就是空的，系统会退化成「每次都从代码里现查」。这套机制能不能成立，依赖的是配套的审核工作流，不单是代码问题。

**需要再确认**：`routers/knowledge.py` 的列表接口按 `status` 过滤，取值是 `published` / `pending_review` / `expired`，和检索工具用的 `state = APPROVED` 不是一套口径。可能是两个独立字段（治理态 + 展示态），也可能是我早期遗留的不一致。如果是后者，会出现「知识管理页显示已发布、Agent 却检索不到」这种很难排查的现象。我会先把这个核完。

### 7.4 与其他记忆方案的机制差异

| 维度 | 本项目 | LangChain BufferWindow | LangChain SummaryBuffer | MemGPT / Letta | Mem0 / Zep |
| --- | --- | --- | --- | --- | --- |
| 短期窗口 | 固定 10 条，硬编码 | 固定 k 条 | 固定 token 预算 | 主上下文分页 | 滑窗 |
| 含工具轨迹 | 否 | 是（看配置） | 是 | 是 | 视实现 |
| 压缩时机 | 超 32k token | 不压缩 | 超预算即摘要 | 上下文将满时换出 | 异步抽取 |
| 压缩结果持久化 | 否 | — | 否 | 是 | 是 |
| 长期记忆写入 | 人工提交 + 人工审核 | 无 | 无 | 模型自主写入 | 自动抽取事实 |
| 长期记忆召回 | 关键词检索工具 | — | — | 模型主动检索 | 向量召回 |
| 跨会话延续 | 仅通过已审核知识 | 无 | 无 | 有 | 有 |
| 用户级画像 | 无（`user_id` 默认 `guest`） | 无 | 无 | 有 | 有 |
| 错误信息扩散风险 | 低 | 不适用 | 不适用 | 较高 | 较高 |
| 知识沉淀吞吐 | 受人工限制 | — | — | 高 | 高 |

---

## 八、规划（Plan）

我写了三套规划机制，两套在生产路径上。

### 8.1 请求级：三级分诊链

```mermaid
flowchart TD
    Q["用户提问"] --> L1{"L1 _find_best_agent<br>_KEYWORD_MAP 命中？<br>0 次 LLM"}
    L1 -- 命中 --> A["确定员工"]
    L1 -- 未命中 --> L2{"L2 _find_agent_by_repo<br>出现仓库名？长名优先<br>0 次 LLM"}
    L2 -- 命中 --> A
    L2 -- 未命中 --> L3{"L3 _llm_dispatch<br>temperature=0.1 max_tokens=2000<br>空返重试 1 次，1-2 次 LLM"}
    L3 -- 输出 agent_id --> A
    L3 -- 都未命中 --> F["总前台自答<br>给全部内置工具 + 所有 APPROVED 的 MCP"]
    A --> MD["_match_domains 数领域个数"]
    MD --> C{"≥ 2 个领域？"}
    C -- 是 --> SG["answer.completed 带 suggest_collab<br>前端提示转协作任务"]
    N["❗ L1 命中即返回，误命中时 L2/L3 无机会纠正"] -.- L1
```

| 级别 | 函数 | 机制 | LLM 调用 |
| --- | --- | --- | --- |
| L1 | `_find_best_agent` | 关键词表 `_KEYWORD_MAP` 映射领域（产品/架构/后端/前端/测试/管理/通用） | 0 次 |
| L2 | `_find_agent_by_repo` | 问题里出现仓库名 → 反查 `AgentRepoBinding`，长名优先 | 0 次 |
| L3 | `_llm_dispatch` | 员工列表（含 repos、skills）拼进 prompt，要求输出 `{agent_id, reason}`；`temperature=0.1`、`max_tokens=2000`；空返重试 1 次；`extract_llm_json` 容错剥围栏 | 1–2 次 |
| 兜底 | — | 都没命中就总前台自答，此时给全部内置工具 + 所有 APPROVED 的 MCP Server | — |

另外 `_match_domains` 会数问题涉及几个领域，≥2 个时在 `answer.completed` 里带上 `suggest_collab`，前端据此提示用户转协作任务。

**取舍**：便宜的先跑，昂贵的兜底。大多数问题在 L1/L2 就完事了，不用为分诊多花一次 LLM 调用。

**已知不足**：L1 的关键词表是硬编码中文词表，新增领域得改代码；而且它优先级最高，一旦误命中（比如「前端接口怎么调」里的「接口」命中后端域），L2/L3 没有机会纠正。把 L1 从「命中即返回」降为「候选打分」可以缓解，代价是必然要跑 L3。

### 8.2 任务级：协作会议室

```mermaid
flowchart TD
    IN["任务标题 + 描述 + deadline_minutes"] --> R0["[0] 预留时间<br>reserve_sec = min(90, deadline × 0.1)"]
    R0 --> P["[1] Plan · _decompose_task<br>temperature=0.2 max_tokens=4000<br>护栏：agent_id 必在有效集内 · 数量截至 5 · confidence 白名单"]
    P --> R1["[2] Round 1 · 并行草案<br>asyncio.gather · 完整 ReAct<br>session=task-id · history_limit=0<br>时间预算 55%"]
    R1 --> ST1{"按时交付？"}
    ST1 -- 是 --> SUB["状态 analyzing → submitted"]
    ST1 -- 超时 --> CLA["状态 clarify（未交付）"]
    SUB --> CK{"草案 ≥ 2 份？"}
    CK -- 否 --> SY
    CK -- 是 --> R2["[3] Round 2 · _run_discussion<br>全部草案可见（单份截 6000 字）<br>session=task-id-r2 · 时间预算 35%"]
    R2 -- 成功 --> DIS["submitted → discussing → discussed"]
    R2 -- 失败 --> BACK["回退 submitted<br>绝不覆写 subtask_detail"]
    DIS --> SY["[4] Synth · _synthesize_doc<br>temperature=0.3 max_tokens=16000<br>固定六章节"]
    BACK --> SY
    CLA --> SY
    SY -- 失败 --> FB["_fallback_doc 代码拼接兜底"]
    SY --> SM["[5] Summary · _summarize ≤200 字<br>首行强制 ✅ 或 ⚠️<br>冲突逐条列出，禁止淡化"]
    FB --> SM
    SM --> OUT["落盘 workspaces/collab_docs/<br>写入 TaskCard.result_doc_path 与 conflict_note"]
```

```javascript
输入：任务标题 + 描述 + deadline_minutes

[0] 预留时间   reserve_sec = min(90, deadline * 0.1)

[1] Plan       _decompose_task
                 LLM 输出 {"subtasks":[{agent_id,title,detail,confidence}]}
                 temperature=0.2, max_tokens=4000
                 护栏：agent_id 必须在有效员工集合内
                       数量截断至 5
                       confidence 只接受白名单取值

[2] Round 1    _run_subtask × N，asyncio.gather 并行
                 每个子任务跑完整 ReAct，可调全部授权工具
                 独立 session_id = "task-{id}"，history_limit = 0
                 时间预算 = 总时长 × 55%，asyncio.wait_for 包裹
                 状态：analyzing → submitted；超时未交付 → clarify

[3] Round 2    _run_discussion，触发条件：草案 ≥ 2 份
                 每人可见全部草案（单份截 6000 字），站在本岗位互评
                 独立 session_id = "task-{id}-r2"
                 时间预算 = 总时长 × 35%
                 状态：submitted → discussing → discussed
                 失败 → 回退 submitted，绝不覆写 subtask_detail

[4] Synth      _synthesize_doc，temperature=0.3, max_tokens=16000
                 固定六章节：任务背景与目标 / 总体方案 / 各部门要点与衔接
                             / 分歧点与裁决 / 风险与待确认 / 行动清单与分工
                 LLM 失败 → _fallback_doc 用代码拼接草案兜底
                 产物落盘 workspaces/collab_docs/，路径写入 TaskCard.result_doc_path

[5] Summary    _summarize，≤200 字
                 首行强制 ✅（无冲突）或 ⚠️（有冲突/未交付）
                 prompt 要求冲突逐条显式列出，禁止合并淡化
                 写入 TaskCard.conflict_note
```

**重试机制**：`_run_agent_question_with_retry` 把两种情况都当失败并整轮重试——输出为空，或输出命中 `_LLM_FAIL_MARKERS = ("[LLM 调用失败", "[LLM 重试耗尽]")`。退避 3 秒、8 秒，`max_retries = 2`。`CancelledError` 直接上抛，不参与重试。

**取舍**：`TaskAssignment` 里我把草案（`subtask_detail`）和互评（`discussion_note`）分了两列。这样第二轮无论怎么失败，第一轮的成果都不会被覆写。多 Agent 流程里最让人难受的就是后面一环报错把前面跑出来的东西也带没了。

### 8.3 SubagentManager（已实现，主路径未接入）

`runtime/subagent.py` 提供 `spawn` / `wait_all(timeout=60)` / `get_results_summary()`，`max_concurrent = 3`，子任务用独立 AgentRunner 与工具白名单，`history_limit = 0`，默认预算 `{steps:5, calls:3, timeout:30, token:24000}`。

**已知不足**：协作会议室现在走的是 `tasks.py` 自己的 `asyncio.gather`，没用这个模块。同一个进程里因此存在两套并发编排，再加上 provider 层的全局信号量（默认 2），实际并发受最紧的那一层约束。这是应该合并的重复实现。

### 8.4 与其他规划范式的机制差异

| 维度 | 本项目·单 Agent | 本项目·协作会议室 | ReAct 原始形态 | Plan-and-Solve | Tree of Thoughts | AutoGen GroupChat | CrewAI |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 计划表示 | 无显式计划，逐步决策 | 一次性拆解为 ≤5 子任务 | 无显式计划 | 先出完整计划再执行 | 搜索树 | 无计划，靠发言轮转 | 显式 task 列表 |
| 执行拓扑 | 串行 | 全并行，无依赖建模 | 串行 | 多为串行 | 分支并行 | 串行发言 | 串行或层级 |
| 重规划 | 每轮隐式调整 | 无 | 每步隐式 | 无 | 回溯 | 有 | 有限 |
| 角色差异来源 | 单角色 | 绑定的仓库/领域/技能/工具白名单 | 无角色 | 无角色 | 无角色 | 仅 system prompt | prompt + 工具 |
| 交叉评审 | 无 | 固定两轮，草案全可见 | 无 | 无 | 评估节点 | 自由讨论 | 可配置 |
| 分歧处理 | 不适用 | 强制单列「分歧点与裁决」章节 | 不适用 | 不适用 | 剪枝 | 趋向共识 | 趋向共识 |
| 终止条件 | 预算四维 + 模型自判 | 时间比例切分 | 模型自判 | 计划走完 | 深度/宽度上限 | 轮数上限 | 任务完成 |
| 失败处理 | finalize 兜底应答 | 每环独立降级 | 无 | 无 | 无 | 有限 | 有限 |
| 成本上界 | 可预测 | 可预测 | 不确定 | 较确定 | 高且难预测 | 易发散 | 中 |

### 8.5 这套规划真正不一样的地方，以及它的天花板

**两处我认为有实质差异的设计**

20. **角色差异不只在 prompt 里**。后端域员工的工具实例在构造期就注入了仓库白名单，越界会被 `_resolve_repo` 拦下，不是靠 prompt 请求它自觉。常见框架里角色差异只体现在 system message 上，这是不同层级的机制。
21. **合成阶段我刻意保留分歧**。合成 prompt 硬性要求「分歧点与裁决」章节，汇总首行强制 ✅/⚠️ 标记，并明确要求不得合并淡化。多数多 Agent 框架的目标是收敛到共识，我这里反过来。真实评审会上，被掩盖的分歧比没有共识更危险。

**三处结构性限制（已知不足）**

22. 没有依赖建模。所有子任务无条件并行，但真实任务常有顺序约束（先定架构再定接口）。现在只能靠拆解 prompt 里的描述文字暗示顺序，执行层不保证。子任务支持 `depends_on` 做拓扑分层是下一步。
23. 没有重规划。第一轮拆解质量决定了上限，就算第二轮互评暴露出「漏了一个关键角色」，也不会重拆或补人。
24. 讨论轮数固定为 2，与分歧程度无关。分歧大时不会追加讨论，分歧小时也要付第二轮的成本。

**关于时间比例切分，有一个我自己复盘时才意识到的问题**：把 deadline 按 55%/35%/剩余切分，能保证整体不超时，但它切的是墙钟时间，而真正的瓶颈是 LLM 并发。在 `LLM_CONCURRENCY=2` 下，子任务越多，每个子任务实际能拿到的执行时间越少（排队时间也算在 `wait_for` 内），所以**子任务数量与超时率是正相关的**。我现在没有对应的保护，合理的做法是根据并发度反推子任务数上限，而不是固定截断到 5。

---

## 九、Git 仓库管理

### 9.1 git_service

所有 Git 操作都是子进程调系统 `git` CLI，没用 GitPython。启动时 `check_git_available()` 检一次并缓存结果，没装时返回按发行版分类的安装指引。

| 操作 | 实现 | 超时 |
| --- | --- | --- |
| clone | `asyncio.create_subprocess_exec`，**全量克隆** | 300s |
| pull | `git pull --ff-only` | 60s |
| 切分支 | `fetch origin` → 本地有则 `checkout`，无则 `checkout -B <b> origin/<b>` → `pull --ff-only` | 各 60–120s |
| 分支列表 / 作者查询 | `subprocess.run`，**同步阻塞** | 15s |

**取舍**：我没用浅克隆。`--depth 1` 只有一条提交，支撑不了 `git log` 查文件创建人/最后改动人，也切不了分支。磁盘多占一点换这两个能力，我认为划算。

路径安全：`resolve_safe_path()` 把目标路径 `resolve()` 后校验是否以仓库根为前缀，防路径遍历。

**已知不足（影响面最大的一个）**：分支查询和作者查询用的是同步 `subprocess.run`，在 async 事件循环里会阻塞整个进程。单次 15 秒超时意味着最坏情况下一次请求可以卡住整个服务 15 秒。改成 `create_subprocess_exec` 就行，这是我优先级很靠前的一项。

### 9.2 定时刷新

`auto_refresh_loop()` 每 60 秒轮询一次 `repository` 表，跳过 `auto_refresh_minutes` 为 NULL/0 的、未到间隔的、本地目录不存在的，其余跑 `pull_repo` 并更新 `last_sync_at`。时间比较统一用 naive UTC，因为 MySQL DATETIME 不带时区。循环内捕获所有异常后继续，只有 `CancelledError` 直接上抛，以便随服务正常关停。

### 9.3 仓库 REST 接口

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| GET | `/env` | 环境自检：git CLI、磁盘（要求剩余超 1GB）、workspaces 可写性、git 全局配置、Gitee/GitHub/GitLab 连通性 |
| POST | `/clone` | clone 并写入 `Repository` 表 |
| GET | 根路径 | 列表；先调 `sync_workspace_repos` 把本地目录增量登记入库 |
| GET | `/{id}/files`、`/read`、`/structure`、`/branches`、`/file-author` | 只读查询 |
| POST | `/{id}/search`、`/pull`、`/checkout` | 检索与同步 |
| PATCH | `/{id}` | 改名（同步重命名本地目录）、改地址、改默认分支、改刷新间隔（0 关闭，上限 7 天） |
| DELETE | `/{id}` | 删本地目录 + 删表记录 + 清理 `AgentRepoBinding` |

两处我特意处理过的地方：

25. Windows 下 `.git` 内只读文件会让 `shutil.rmtree` 抛 `PermissionError`。我用 `onexc` 回调去掉只读位后重试，并明确不用 `ignore_errors=True`——静默失败后列表接口的 `sync_workspace_repos` 会把残留目录重新登记，用户看到的就是「删不掉」。
26. 删完再校验一次目录是否仍在，仍在就报 500，不拿假成功糊弄前端。

另外，`sync_workspace_repos` 与列表接口的兜底逻辑都要求目录下存在 `.git` 才登记，以此排除 `collab_docs`、`uploads` 这些非仓库目录。

---

## 十、数据模型

| 表 | 关键字段 | 状态取值 |
| --- | --- | --- |
| `agent` | `agents_md`、`skills`、`resources`、`role_pack_id`、`adoption_rate`、`session_count` | draft → indexing → pending_check → trial → online → maintenance / retired |
| `role_pack` | `config`（JSON：tools / skills / permission / budget） | — |
| `role_pack_version` | `spec`、`checksum`(SHA-256)、`rollout`（灰度百分比） | DRAFT → IN_REVIEW → CANARY → RELEASED → ROLLED_BACK / RETIRED |
| `session` | `user_id`（默认 `guest`）、`title`、`state` | active / closed |
| `message` | `role`、`content`、`agent_id`、`evidence_ids`、`confidence`、`feedback` | — |
| `knowledge_candidate` | `body_md`、`evidence_ids`、`commit_sha`、`expires_at`、`scope`、`conflict_warning` | SUBMITTED → IN_REVIEW → APPROVED / REJECTED → EXPIRED |
| `task_card` | `deadline_minutes`（默认 30）、`tags`、`conflict_note`、`result_doc_path` | in_progress / completed / failed |
| `task_assignment` | `subtask_detail`、`discussion_note`、`confidence`、`agent_emoji` | analyzing → submitted → discussing → discussed；clarify 为失败/超时态 |
| `change_request` | `diff`、`approval_policy` | PENDING → APPROVED / REJECTED → APPLIED / FAILED；策略 SINGLE / DUAL / QUORUM |
| `policy_rule` | `category`、`version`（追加式） | ACL / REDACTION / BUDGET / TOOL_RISK；ACTIVE / DISABLED |
| `gov_role` | — | platform-admin / rolepack-owner / knowledge-reviewer / security-reviewer / auditor |
| `repository` | `clone_url`、`default_branch`、`auto_refresh_minutes`、`last_sync_at`、`owner` | CONNECTING → CONNECTED → INDEXING → READY → DEGRADED / FAILED |
| `agent_repo_binding` | `agent_id` × `repo_id`（引用 `Repository.id` 而非 name，所以改名不断绑定） | — |
| `audit_log` | actor / action / target / detail / trace_id / decision | ALLOW / DENY |

**已知不足**：`session.user_id` 默认是 `guest`，对话链路上没有认证依赖注入。所以现在所有会话在数据层面都归属同一个虚拟用户，`gov_role` 定义的角色体系在对话侧还没接入。治理模型是齐备的，但权限还没落到人头上。

---

## 十一、并发、超时与失败模式

先用一张图看清楚超时是怎么嵌套的，下面的表是完整参数清单：

```mermaid
flowchart TB
    T1["协作任务 deadline_minutes（默认 30 分钟）"] --> T2["Round 1 = 55% · Round 2 = 35% · 合成保底 90s"]
    T2 --> T3["单个 Agent 执行 Budget.timeout_sec = 60s<br>max_steps=8 · max_calls=6 · max_token=48000"]
    T3 --> T4["LLM 单次请求 LLM_TIMEOUT = 60s"]
    T4 --> T5["全局并发闸 LLM_CONCURRENCY = 2<br>排队时间计入上层超时"]
    T5 --> N["❗ 真正的瓶颈在最内层：<br>子任务越多，每个拿到的实际执行时间越少"]
```

排查线上问题时看这张表就够了，我把散在各处的时间与并发参数集中在一起：

| 环节 | 参数 | 值 | 超时后的行为 |
| --- | --- | --- | --- |
| LLM 单次请求 | `LLM_TIMEOUT` | 60s | 按类型判为瞬时错误 → 退避 1/2/4/8 重试 |
| LLM 全局并发 | `LLM_CONCURRENCY` | 2 | 排队，等待时间计入上层超时 |
| Agent 单次执行 | `Budget.timeout_sec` | 60s | finalize 兜底应答 |
| Agent 步数 | `Budget.max_steps` | 8 | 同上 |
| 上下文压缩 | 阈值 32000 token | — | 压缩失败降级为截断 200 字 |
| 协作 Round 1 | deadline × 55% | — | 标记 clarify |
| 协作 Round 2 | deadline × 35% | — | 回退 submitted，保留草案 |
| 协作整轮重试 | 退避 3s / 8s，2 次 | — | 记为未交付 |
| Subagent | `max_concurrent=3`，`wait_all(timeout=60)` | — | 主路径未启用 |
| git clone | 300s | — | 返回失败 |
| git pull / checkout | 60–120s | — | 返回失败 |
| git 分支/作者查询 | 15s，**同步阻塞** | — | 返回空 |
| 仓库定时刷新 | 60s 轮询 | — | 异常记日志继续 |

### 三个我在线上碰到过的真实故障

| 故障 | 根因 | 现状 |
| --- | --- | --- |
| 协作会议室出现 `[LLM 调用失败: ]`，错误信息是空的 | httpx 超时异常 `str(e)` 为空串，我早期靠消息匹配判断可重试性 | 已改成按异常类型判定 |
| 草案里出现工具名重复数百次 | finalize 轮 `tools=None` 但模型仍试图调工具 | 已加正则清洗 + 短回答重试 |
| 模型凭文档名编造正文 | 资源工具在正文不可读时没作说明 | 已加显式禁止提示 |

### SSE 场景下的一条数据库纪律

流式接口内的所有写库必须立即 `commit`：`_get_or_create_session`、`_save_message`、标题更新、反馈记录都是。不这么做的话，事务会横跨整个 LLM 调用（数十秒），MySQL 在 REPEATABLE READ 下会阻塞其他请求的 INSERT 直到锁等待超时。接手的人在流式链路上加写库逻辑时，请继续保持这个习惯。

---

## 十二、总结

### 12.1 我认为这套系统做对的五件事

27. **权限边界落在执行层，不在提示层**。仓库白名单在工具构造期就注入，越界由代码拦，不靠模型自觉。
28. **外部调用几乎都有降级分支**。LLM、MCP、git、DB 写入、文档合成各自都有明确的失败路径，单点失败不会把整个流程带倒。
29. **MCP 会话生命周期处理对了**。`AsyncExitStack` 由调用方持有并覆盖整个消费过程，这是这类集成里最容易出错的一点。
30. **协议翻译层隔离了模型供应商**。换 LLM 只需改 provider 一层。
31. **注释里写了「为什么」而不只是「写了什么」**。很多发现很难从代码重新推出来，写下来才不会丢。

### 12.2 我自己列的待办清单

| 级别 | 问题 | 位置 |
| --- | --- | --- |
| P0 | 三项凭据以明文写在公开仓库的默认值里，需轮换 + 改强制环境变量 | `config.py` |
| P0 | 知识检索「先取最近 5 条再过滤」，超出 5 条的知识永不可召回 | `runtime/tools/__init__.py` |
| P1 | git 分支/作者查询用同步 `subprocess.run`，阻塞事件循环最长 15s | `git_service.py` |
| P1 | 代码检索无索引，且每个命中文件额外两次 `git log` | `git_service.py` |
| P1 | 无认证，`user_id` 恒为 `guest`，`gov_role` 未接入对话侧 | `frontdesk.py` |
| P1 | 会话历史丢弃工具消息，且 10 条上限硬编码 | `context.py` |
| P1 | 项目未纳入 Git 版本管理，靠 `.bak` 文件链回滚 | 全局 |
| P2 | Hook 仅在流式路径挂载，非流式无审计无脱敏 | `runner.py` |
| P2 | 注入检测为英文黑名单，不覆盖中文与间接注入 | `hooks.py` |
| P2 | 两套 Agent 实现并存，应彻底隔离或删除旁路 | `runtime/` vs `agents/` |
| P2 | 两套并发编排并存，应合并 | `subagent.py` vs `tasks.py` |
| P2 | 协作子任务数与超时率正相关，无保护 | `tasks.py` |
| P2 | 仓库存本机文件系统，多实例部署需共享存储 | `git_service.py` |
| P2 | 知识条目 `state` 与 `status` 两套口径待核 | `models/knowledge.py` |

### 12.3 这套系统的取向

回头看，我的技术选择有一个一致的倾向：**在能力上限和行为可控之间，我稳定地选后者**。

- 不用向量检索，接受召回率损失，换零索引维护与结果可追溯；
- 不做自动知识沉淀，接受沉淀慢，换不会写入错误结论；
- 不用通用 Agent 框架，接受自己写循环，换工具边界与预算的精确控制；
- 并发只开 2，接受慢，换不被上游限流打断。

对「企业内部、答错代价高于答慢代价」这个场景，我认为这个取向是对的，现在也不打算推翻。

但取向对不等于实现到位。真正的风险不在这些取舍上，而在两处执行层面的问题：**知识召回的逻辑顺序写反了**，让长期记忆形同虚设；**同步阻塞的 git 调用**，让「慢」可能变成「卡住整个服务」。这两项修复成本都很低，是我接下来先动的。

最后说明一句：本文所有性能相关的判断都是静态分析得出的，没经过实测。仓库规模、知识条目数量、并发用户数这三个变量会明显改变结论权重。真要上规模，建议先补一轮压测再定优先级。

[https://prod-files-secure.s3.us-west-2.amazonaws.com/f73c3176-1336-4063-b578-c33cc8f51fe2/5873045b-accd-4678-82e3-75aef36ff9a5/office_agent_design.html?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=ASIAZI2LB46663CWBNXU%2F20260807%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20260807T064420Z&X-Amz-Expires=3600&X-Amz-Security-Token=IQoJb3JpZ2luX2VjEIX%2F%2F%2F%2F%2F%2F%2F%2F%2F%2FwEaCXVzLXdlc3QtMiJHMEUCIA9q6UX36Lty12eovHP8xqSAqINUn8wEbf0RbyzWmDZCAiEAjqL4Vy27TX%2F0T9LOPPuNqOFFZCcvZQf98W0F1CEtcrMq%2FwMIThAAGgw2Mzc0MjMxODM4MDUiDI2Tzrz0E%2FUi2bY8bSrcA2%2BNEpanaQNR7CG1EVSCJ%2Fbct710YWR1bq3%2Fh87Um35mBz4AwD4OadvxfAvRJk4q4A2gD87Xq%2BzdAOPehqnZdCCuT9uJHDxX6OnV8K6sETzmOn3L99yHjiFBFSqm8wa2KRigpS6N2fS3WhnODsDgyqg3pb32348M1N5bLYFNLFxvOGQwnM1eNIbJDkWpwbSA1foK7HbE33Ous2Z2rBpzZ1MmRtmjklI1JDPGqSlVPuc%2FsoS4qQDd8s64NnVEMITcVHDTemdt0ABQzg5HRROccTvtobj4vhzEAkWkOekZoJLToAKTaR6E7H88%2FaDpwxjXCSc34fFKwNwnocdG8YBgdY%2Bd3kfbmSTKWIpftyXFHx1xYDCS9DgU064ZPVqDhTulKTeoIHUbwIYA98H8QMRjqvLxM%2FN775FpCYnbvw9C6d%2Fe9NRxMG52tmS1Br8h5JEyVw51sj7NDO4SbXzxlWGz0riZawwEVIPXbOIsL6T8TIvzMiLdXK0YekspNc%2B4zOhLZFr8m9NJiGUDlxrL3uLqCMBgVX0CHL8in3LRN3JUvXAjdjZTYDO4I7nxrW779IeEq6G4RBPLHRk1pJ5PnxRruPZ3z7zK%2BYO5Gh%2B9vv%2BRHOVJhNK%2BGL00nLYsDSKfMJzR1dMGOqUBopoPNHt0JIHVB8n73b3Nqz%2BEVwsIyGwfccbg3Fd2eLw%2FtWi6P9QvlZH8ZmRLgWKZIu3YqouogwHasiWhJRE0DcxrHSwJ7dUjPU%2FGEeFx1N%2Bj5i7OcqSvuSwopMj9%2B1cjXit1jhJtFrUjQv34AGKu3TY4rOuCTlkECtGtV9KsalGMXVwTdw5t8ej1QE%2FxemD%2BLIO2iLa%2FcjUoIcctGbAym4%2FczSTK&X-Amz-Signature=b2aaf9beb1407651fdf1e3fbf568c4139d2b483dab63ab9acac9013dc0ea4069&X-Amz-SignedHeaders=host&x-amz-checksum-mode=ENABLED&x-id=GetObject](https://prod-files-secure.s3.us-west-2.amazonaws.com/f73c3176-1336-4063-b578-c33cc8f51fe2/5873045b-accd-4678-82e3-75aef36ff9a5/office_agent_design.html?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=ASIAZI2LB46663CWBNXU%2F20260807%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20260807T064420Z&X-Amz-Expires=3600&X-Amz-Security-Token=IQoJb3JpZ2luX2VjEIX%2F%2F%2F%2F%2F%2F%2F%2F%2F%2FwEaCXVzLXdlc3QtMiJHMEUCIA9q6UX36Lty12eovHP8xqSAqINUn8wEbf0RbyzWmDZCAiEAjqL4Vy27TX%2F0T9LOPPuNqOFFZCcvZQf98W0F1CEtcrMq%2FwMIThAAGgw2Mzc0MjMxODM4MDUiDI2Tzrz0E%2FUi2bY8bSrcA2%2BNEpanaQNR7CG1EVSCJ%2Fbct710YWR1bq3%2Fh87Um35mBz4AwD4OadvxfAvRJk4q4A2gD87Xq%2BzdAOPehqnZdCCuT9uJHDxX6OnV8K6sETzmOn3L99yHjiFBFSqm8wa2KRigpS6N2fS3WhnODsDgyqg3pb32348M1N5bLYFNLFxvOGQwnM1eNIbJDkWpwbSA1foK7HbE33Ous2Z2rBpzZ1MmRtmjklI1JDPGqSlVPuc%2FsoS4qQDd8s64NnVEMITcVHDTemdt0ABQzg5HRROccTvtobj4vhzEAkWkOekZoJLToAKTaR6E7H88%2FaDpwxjXCSc34fFKwNwnocdG8YBgdY%2Bd3kfbmSTKWIpftyXFHx1xYDCS9DgU064ZPVqDhTulKTeoIHUbwIYA98H8QMRjqvLxM%2FN775FpCYnbvw9C6d%2Fe9NRxMG52tmS1Br8h5JEyVw51sj7NDO4SbXzxlWGz0riZawwEVIPXbOIsL6T8TIvzMiLdXK0YekspNc%2B4zOhLZFr8m9NJiGUDlxrL3uLqCMBgVX0CHL8in3LRN3JUvXAjdjZTYDO4I7nxrW779IeEq6G4RBPLHRk1pJ5PnxRruPZ3z7zK%2BYO5Gh%2B9vv%2BRHOVJhNK%2BGL00nLYsDSKfMJzR1dMGOqUBopoPNHt0JIHVB8n73b3Nqz%2BEVwsIyGwfccbg3Fd2eLw%2FtWi6P9QvlZH8ZmRLgWKZIu3YqouogwHasiWhJRE0DcxrHSwJ7dUjPU%2FGEeFx1N%2Bj5i7OcqSvuSwopMj9%2B1cjXit1jhJtFrUjQv34AGKu3TY4rOuCTlkECtGtV9KsalGMXVwTdw5t8ej1QE%2FxemD%2BLIO2iLa%2FcjUoIcctGbAym4%2FczSTK&X-Amz-Signature=b2aaf9beb1407651fdf1e3fbf568c4139d2b483dab63ab9acac9013dc0ea4069&X-Amz-SignedHeaders=host&x-amz-checksum-mode=ENABLED&x-id=GetObject)
