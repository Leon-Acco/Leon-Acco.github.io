---
layout: post
title: "AgentCore SDK 开发指南（Java 为主）"
date: 2026-10-27
categories: [ai]
series: agentcore
series_order: 5
tags: [AgentCore, 阿里云, Java, SpringAI, LangChain4j, SDK]
original: https://help.aliyun.com/zh/agentcore/agentcore-java-sdk-usage-guide
mermaid: false
math: false
toc: true
---
<div class="callout callout-tip" markdown="1">
**💡 返回 [AgentCore 学习指南（总览）](/2026/10/27/agentcore-00-overview/) · 部署上线见 [AgentCore 高代码与纳管 Agent 部署手册](/2026/10/27/agentcore-07-deployment/)**

</div>

`alibabacloud-agentcore-sdk` 用于开发 Agent **应用**（调用模型/MCP/Skill/记忆/凭证），与调用平台管理 API 的 OpenAPI SDK **不是同一个包**。SDK 不负责创建 Workspace、上传代码、构建镜像或部署应用。

## 一、SDK 模块与依赖（Maven Central）

```xml
<dependency>
    <groupId>io.github.ai-agentcore</groupId>
    <artifactId>agentcore-sdk</artifactId>
    <version>0.1.0</version>
</dependency>
```

| 模块（artifactId） | 能力 | 对应框架版本 |
| --- | --- | --- |
| agentcore-sdk | 基础 SDK：Model / MCP / Skill / Memory / Credential 接口 | — |
| agentcore-sdk-server | AG-UI、OpenAI Chat Completions、自定义协议服务 | — |
| agentcore-sdk-spring-ai | Spring AI 适配 | Spring AI 2.0.1 |
| agentcore-sdk-langchain4j | LangChain4j 适配 | LangChain4j 1.20.0 |
| agentcore-sdk-agentscope | AgentScope Java 适配 | AgentScope Java 2.0.3 |

<div class="callout callout-note" markdown="1">
**📝 坐标差异**

Maven groupId 是 `io.github.ai-agentcore`，Java 包名却是 `io.agentcore` 开头。
</div>

环境要求：JDK 17+、Maven 3.9+。Python 版需 3.10+，Node.js 版需 20.3+。

## 二、初始化客户端

```java
try (var core = AgentCore.auto()) {
    var model = core.model("my-model-connection", "my-model").block();
    // ...
}
```

- `AgentCore.auto()` 从 **AgentCore 运行环境**获取访问配置——业务代码**无需填写平台访问密钥**（云端资源示例必须跑在托管环境内）
- 实例在**应用生命周期内复用**，退出时 `close()`；不要在单次 HTTP 请求后关闭共享实例
- 基础 API 返回 Reactor 的 `Mono`/`Flux`，必须订阅才执行；`block()` 仅用于普通主线程，**WebFlux 事件循环里禁止 block**

## 三、五大云端资源调用

| 资源 | 调用 | 要点 |
| --- | --- | --- |
| 模型 | `core.model(连接名, 模型名)` → `completion()` / `stream()` | 免拼接地址；OpenAI 协议另有 `responses()`（不支持时**不会**自动降级到 Chat Completions）；无 Embedding 能力 |
| MCP | `core.mcp("my-mcp")` → `listTools()` / `callTool(名, 参数)` | listTools 只发现不执行；模型自动用工具需注册到框架 |
| Skill | `core.skills().load("my-skill", "1.0.0")` → `skill.tools()` | 版本须已发布，省略则取最新；同实例固定首次加载版本，**建议显式指定**；本地 Skill：`core.skills().local(Path.of("./skills"))` |
| 记忆 | `core.memory("my-memory")` → `addMemories` / `searchMemories` 等 | MemoryScope(userId, agentId, sessionId)，sessionId 传 null = 跨会话检索；新写入需时间才可检索 |
| 凭证 | `core.credentials().get("my-api-key")` → `value()` / `asHeaders()` | 为 MCP 绑定：`core.mcp("my-mcp", "my-mcp-header", Map.of(...))` |

### 记忆的 Scope 与 Metadata

- **Scope**（业务归属，应用自定义维护）：`userId` / `agentId` / `sessionId`，查询时精确过滤，不指定则不过滤
- **Metadata**（业务属性）：字符串键值对，如 `category=dietary-preference`、`source=conversation`
- 长期记忆 ≠ 会话历史：输入与生成的记忆**不一定一一对应**（可能多条或零条）

### 自定义资源（不注册平台资源，应用直连）

```java
core.directModel(name, URI.create(baseUrl), ModelClient.Protocol.OPENAI, headerProvider);
core.directMcp(URI.create(url), MCPClient.Transport.STREAMABLE_HTTP, headerProvider);
```

- 直连模型支持 `Protocol.ANTHROPIC`（Base URL 不带 `/v1`，须指定 max_tokens）
- 直连 MCP 支持 SSE 与 stdio（stdio 不承载 HTTP Header）
- API Key 由运行环境或密钥管理服务提供，不硬编码

## 四、三大框架适配

三框架能力范围一致（模型、工具、事件、记忆），选型看**已有技术栈**：

| 框架 | 入口类 | 工具注册 | 记忆接入 | 编排特点 |
| --- | --- | --- | --- | --- |
| Spring AI | AgentCoreSpringAI、SpringAIEvents、MemoryAdvisor | 转 `ToolCallback[]` 经 `ToolCallingChatOptions` 绑定 | ChatClient 注册 MemoryAdvisor | 资源初始化加载，Prompt 按请求构造 |
| LangChain4j | AgentCoreLangChain4j、LangChain4jEvents、MemoryAdapter | `AgentCoreLangChain4j.tools(...)` 注册到 AiServices | `MemoryAdapter.retriever()` + call/stream | `LangChain4jEvents.from` 启动 TokenStream 并转事件 |
| AgentScope Java | AgentCoreAgentScope、AgentScopeEvents、MemoryMiddleware | `toolkit.registerAgentTool(...)` | ReActAgent 注册 MemoryMiddleware | 每请求独立 ReActAgent，用完释放（`Flux.using` 包装） |

### LangChain4j 端到端骨架

```java
var model = AgentCoreLangChain4j.streamingModel(core.model(...).block());
var tools = core.mcp(...).listTools() + core.skills().load(...).tools();
var assistant = AiServices.builder(Assistant.class)
        .streamingChatModel(model)
        .tools(AgentCoreLangChain4j.tools(tools))
        .build();
new AgentCoreServer(request -> ... LangChain4jEvents.from(() -> assistant.chat(input))).start(9000);
```

<div class="callout callout-warning" markdown="1">
**⚠️ 两个坑**

- `LangChain4jEvents.from` 负责启动 TokenStream 并转换工具调用事件——**不要**手动调 `start()` 或只转发文本回调，否则丢失工具执行过程
- Spring AI 的 MemoryAdvisor 只作用于 **ChatClient 调用链**，不作用于直接调用 `SpringAIEvents.stream` 的独立流程
</div>

### 记忆自动回写（默认关闭）

- LangChain4j 开启：`new MemoryAdapter(memory, true)`
- 回写只保存**本轮用户输入 + 正常完成的最终文本答案**，不重复写历史消息和工具结果；开启后不要手动写回同一轮内容

## 五、构建与部署（Maven）

```bash
mvn -B -ntp package
# 用 dependency 插件导出运行期依赖，target/classes 与 target/dependency 一起进镜像
java -cp 'target/classes:target/dependency/*' App
```

容器镜像要求（Linux amd64、/bin/bash、/bin/sh、ACR 可达）见 [AgentCore 高代码与纳管 Agent 部署手册](/2026/10/27/agentcore-07-deployment/)。

## 六、对外协议选择

| 对比项 | AG-UI | OpenAI Chat Completions |
| --- | --- | --- |
| 文本输出 | ✅ | ✅ |
| 消息边界 | 保留开始/内容/结束事件 | 聚合为一条 assistant 消息 |
| 工具调用 | 保留调用 ID 与参数 | 以 tool_calls 返回 |
| 工具结果 | **独立事件返回** | **不提供** |
| 适用 | 展示多步执行过程 | 对接现有 Chat Completions 客户端 |

- AG-UI 请求体：`threadId`/`runId` 必须非空，每次运行换新 runId；**tools 传空不会禁用已注册工具**；正常流 RUN_STARTED → … → RUN_FINISHED，失败 RUN_ERROR，工具调用与结果靠 `toolCallId` 关联
- 两个 SSE 接口**空闲 15 秒发心跳注释**——心跳不是模型输出，也不代表生成完成
- 服务端已执行工具，客户端**不要**再执行收到的工具调用轨迹

## 七、安全约束（上线前逐条确认）

1. **凭证保护**：凭证值不进源码、日志、对外响应；自有模型 API Key 由运行环境/密钥管理提供
2. **Skill 脚本**：以应用进程权限执行，仅加载可信来源；不需要命令执行时设 `ALLOW_EXECUTE_COMMAND=false`
3. **记忆范围字段**：Scope 由可信业务上下文提供，不由模型任意指定，不替代访问控制
4. **日志**：不记录密钥和敏感对话内容

## 八、FAQ 精选

| 问题 | 答案 |
| --- | --- |
| 必须用 AgentCoreServer 吗？ | 不必，现有 Web 服务可直接调资源接口；用它只为复用协议封装 |
| 加载了 MCP/Skill 模型不调用工具？ | 还需注册到框架 + 模型支持工具调用；简单问题模型也可能判断不需要 |
| 相同会话 ID 自动恢复历史吗？ | **不会**。AgentCoreServer 不持久化历史，多轮对话由应用/框架管理 |
| 自动记忆写全部历史吗？ | 不会。默认关，开启后只写本轮输入+最终答案 |
| 需要手动关客户端吗？ | 应用退出时 close()；生命周期内复用，勿按请求开关 |
| 跨域？ | CORS 由应用或网关配置；SDK 用 SLF4J，日志脱敏按安全约束 |

## 相关笔记

- [AgentCore 高代码与纳管 Agent 部署手册](/2026/10/27/agentcore-07-deployment/)
- [AgentCore 实战：创建 Agent 与 AG-UI 调用](/2026/10/27/agentcore-05-practice/)
