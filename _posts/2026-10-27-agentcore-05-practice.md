---
layout: post
title: "AgentCore 实战：创建 Agent 与 AG-UI 调用"
date: 2026-10-27
categories: [ai]
series: agentcore
series_order: 6
tags: [AgentCore, 阿里云, Agent, AG-UI, SSE]
original: https://help.aliyun.com/zh/agentcore/agent-instructions-managed-harness
mermaid: false
math: false
toc: true
---
<div class="callout callout-tip" markdown="1">
**💡 返回 [AgentCore 学习指南（总览）](/2026/10/27/agentcore-00-overview/) · 功能全景见 [AgentCore 产品功能详解](/2026/10/27/agentcore-02-features/)**

</div>

## 一、快速开始：10 分钟跑通第一个 Agent

1. **准备**：已创建 Workspace；模型连接已验证可用；本地有 curl
2. **创建 Agent**：控制台 → Agent 创建 → 托管 Harness → 选 QwenPaw，填名称、模型连接、模型；资源配置保留预填的实例最大 Session 数和 Session TTL，**关闭自动扩缩容**；网络开启「允许默认网卡访问公网」；提交创建
3. **等就绪**：详情页 → 实例与 Session，等实例和 Session 就绪
4. **拿凭证**：详情页 → 外部渠道配置 → 直接访问，获取 **Endpoint** 与 **API Key**
5. **发起对话**（服务端执行）：

   ```bash
   export AGENTCORE_ENDPOINT="<Endpoint>"
   export AGENTCORE_ENDPOINT_API_KEY="<API Key>"

   curl --no-buffer --request POST "${AGENTCORE_ENDPOINT%/}/ag-ui/agent" \
     --header "x-api-key: ${AGENTCORE_ENDPOINT_API_KEY}" \
     --header 'Content-Type: application/json' \
     --header 'Accept: text/event-stream' \
     --data '{"threadId":"session-001","runId":"run-001","messages":[{"id":"message-001","role":"user","content":"请用三个要点介绍你的功能。"}],"state":{},"tools":[],"context":[],"forwardedProps":{}}'
   ```

<div class="callout callout-warning" markdown="1">
**⚠️ 安全红线**

API Key 只能保存在**应用服务端**，不要写入前端代码或前端配置。`AGENTCORE_ENDPOINT` 不包含 `/ag-ui/agent` 路径。
</div>

### SSE 事件判断

| 事件 | 含义 |
| --- | --- |
| RUN_STARTED | 对话开始 |
| TEXT_MESSAGE_CONTENT | 返回文本片段（delta 字段） |
| RUN_FINISHED 且 outcome.type=success | 成功结束 |
| RUN_ERROR | 失败或取消 |
| 连接关闭但无结束事件 | 本次中断 |

续聊：**threadId 不变**，换新的 runId 和消息 ID。

## 二、AG-UI 协议速查

| 项 | 值 |
| --- | --- |
| 发起/重连 | `POST {agent_endpoint}/ag-ui/agent`（SSE 事件流） |
| 停止执行 | `POST {agent_endpoint}/ag-ui/agent/stop`（204） |
| 认证头 | `x-api-key`（API Key 认证时必填） |
| 会话头 | `X-AgentCore-Session-ID`（必填，必须等于 threadId） |
| Content-Type / Accept | `application/json` / `text/event-stream` |

请求体字段：`threadId`（会话 ID，同对话复用）、`runId`（每次执行换新）、`messages`（至少含本轮用户输入）、`state` / `tools` / `context` / `forwardedProps`（无需时填空容器）。

## 三、高代码 Agent 的两种部署

前提：已有可运行的 HTTP 服务，实际接口与控制台**声明的协议和路径匹配**。

| 方式 | 要点 |
| --- | --- |
| 容器镜像 | 准备镜像 → 创建 Agent 选镜像 → 配置启动方式与执行角色 → 声明协议 → 资源与健康检查 → 网络与日志 → 提交验证 |
| 代码包 | 准备代码包 → 创建 Agent 选运行时 → 其余流程同上（首次体验可直接用平台示例） |

**协议声明**（可同时选两种，Path 以 `/` 开头、不含域名）：

| 协议 | 用途 | 预填 Path |
| --- | --- | --- |
| Chat Completions | 接收 OpenAI Chat Completions 格式请求 | `/openai/v1/chat/completions` |
| AGUI | 接收 AG-UI 请求并返回流式事件 | `/ag-ui/agent` |

<div class="callout callout-warning" markdown="1">
**⚠️ 声明 ≠ 实现**

勾选协议不会自动生成接口；协议与实际路由/响应格式不一致会影响控制台调试。也不要把下游模型的 API Key 填进协议 Header。
</div>

## 四、本地纳管 Agent（存量 Agent 零改造入队）

- **控制台前提**：Workspace 已开启 Team 协作；已创建 Team；只能当 Worker、仅可加入一个 Team
- **本地环境**：Linux/macOS/Windows；已安装 Claude Code / Codex / OpenClaw / Qoder CLI（满足版本要求）；模型来源可选平台或本地；网络可达 AgentCore、模型和 MCP 服务
- **接入方式**：控制台生成**完整启动命令**（选网络、OS、运行时、工作目录），本地执行即可，命令会自动安装/升级 LoongSuite Pilot；首次接入验证 Pilot 与服务状态

## 五、会话机制与会话隔离（面试级重点）

### 1. 三个概念

| 概念 | 解决的问题 |
| --- | --- |
| 会话标识（如 demo-session-01） | 这次请求属于哪个会话（**由调用方生成维护**） |
| 运行实例 | 由哪个运行环境执行（一个实例按容量承载多会话） |
| 对话历史与业务状态 | 本次执行需要哪些上下文、结果存哪 |

### 2. 责任边界（平台不背锅清单）

- 会话标识 ≠ 平台已保存历史：**历史与状态的保存恢复由应用或 Harness 实现**
- 控制台声明协议 ≠ 自动实现接口
- 两个协议用相同会话标识 ≠ 自动跨协议共享历史
- 会话标识不表示会话独占实例；同会话路由保持连续性但**不是永久绑定同一进程**的承诺

### 3. 传递方式

基于 HTTP Header 识别，Header 名称在创建 Agent 时设置（默认 `X-AgentCore-Session-ID`，可改为业务既有字段名）。

### 4. 实例与 Session 容量模型

- 一个 Agent 多实例 × 一个实例多 Session；**同实例 Session 共享 CPU/内存**——提高「实例最大 Session 数」不增加算力
- 同一 Session 同时只处理一个请求，后续排队；不同 Session 并发
- 自动扩缩容按 Session 数计算所需实例数（例：每实例 5 Session、min=1/max=3 → 0~5 Session 用 1 实例，6~10 用 2，11~15 用 3）；所有实例到顶后新 Session 报**容量不足**
- Session TTL 决定空闲回收；TTL 长 → 占容量久、缩容晚
- 官方提醒：会话槽位 2×5=10 不等于能同时跑 10 个高耗时请求——模型配额、工具耗时、内存、实例健康都影响

### 5. 备份与文件

- 原生 Session 定期备份、启动时恢复；TTL 不决定历史保存时长，回收 Session 不删历史
- 异常中断时，**最近 5 秒内未备份的内容可能丢失**
- 每个 Session 独立工作目录；同 Agent 的 Session 经 `shared/` 共享文件；工作文件不进备份，持久化走 OSS 挂载

## 相关笔记

- [AgentCore 产品功能详解](/2026/10/27/agentcore-02-features/)
- [AgentCore 计费说明与使用约束清单](/2026/10/27/agentcore-08-billing/)
