---
layout: post
title: "SpringAI 开发 MCP-Server（一）：0-1 的搭建与实现"
date: 2025-10-27
categories: [ai]
tags: [SpringAI, MCP, Java]
original: https://www.yuque.com/u22429903/kf0kog/fu1vkpl1h736km77
---

# SpringAI开发MCP-Server（一）：0-1的搭建与实现
## 一、前提概要

实现流程

- 使用 spring-ai-starter-mcp-server-webmvc 把普通 Spring Bean 变成 AI 可调用的 Tool；

- 在双数据源（含 Doris）场景下安全地暴露只读 SQL 查询能力，包含防注入与限流措施；

- 将 GB/T‑32960 报文解析逻辑封装为 Tool，让模型直接解析十六进制报文并返回结构化结果；

- 实现 MethodToolCallbackProvider 注册、SSE 流式响应、以及 Agent 侧的调用配置示例；

- 生产化建议：鉴权、配额、审计日志、异常策略与监控接入。

成果展示

## 二、什么是 MCP-Server?

1
什么是MCP?

MCP 全称 Model Context Protocol，直译就是 模型上下文协议。
它是一个 标准协议，用于让 大模型（LLM）和外部应用/工具 能够方便、安全地交互。

2
什么是MCP-Server

MCP Server 就是 MCP  协议的 具体实现端，负责把企业的 业务系统、数据库、工具 暴露出来，供大模型调用。

MCP Server = 把内部系统包装成标准接口的“翻译官”。

 模型通过 MCP Client 发送请求，MCP Server 把请求转化为业务系统能理解的操作，再把结果返回给模型。

如果想更加深入的了解MCP，可以参考篇文章

MCP
一、什么是 MCP？MCP 全称 Model Context Protocol，直译就是 模型上下文协议。它是一个 标准协议，用于让 大模型（LLM）和外部应用/工具 能够方便、安全地交互。可以简单理解为：👉 MCP 就像是 “大模型的 USB 接口”，通过统一的方式，把模型和各种系统（数据...
AI
## 三、开发前的准备

Spring AI 是一个面向人工智能工程的应用框架。其目标是将 Spring 生态系统的可移植性和模块化设计等设计原则应用于人工智能领域，并推广使用 POJO 作为人工智能领域应用程序的构建块。

a
Spring AI 简介与架构认知

Spring AI 是 Spring 团队推出的 AI 应用开发框架，目标是让开发者像写普通 Spring Boot 应用一样，轻松调用大模型、RAG、向量库、Prompt 模板等能力。

核心特性：

- 支持多家模型提供商（OpenAI、Azure、Ollama、Anthropic、百度、阿里、智谱等）

- 内置 ChatClient / EmbeddingClient / ImageClient 抽象层

- 统一配置体系（通过 application.yml）

- 提供 RAG、Prompt Template、Memory（会话上下文）、工具函数调用等扩展

- 未来版本将内置 MCP Server Starter

b
环境与工具准备

项目

要求/建议

JDK

17 或更高版本（Spring Boot 3+ 必须）

Spring Boot

3.3.x 或更新版本

Spring AI

建议 ≥ 1.0.0-M2

构建工具

Maven 或 Gradle（推荐 Maven）

IDE

IntelliJ IDEA / VS Code / Eclipse

网络代理

若访问 OpenAI / Azure 等国外服务，需要配置 HTTP/HTTPS 代理，建议直接使用国内的DeepSeek，通义千问，开发与学习已经足够

GitHub 访问

建议能拉取 Spring AI 官方 starter 源码（方便调试）

c
本项目MCP-Server采取的方式

本文将采取SSE的通信方式实现MCP-Server，Stdio与这个SSE的方式也是大差不差的

• SSE方式适用于客户端和服务器位于不同物理位置的场景。

• 适用于实时数据更新、消息推送、轻量级监控和实时日志流等场景

• 对于分布式或远程部署的场景，基于 HTTP 和 SSE 的传输方式则更为合适。

• 配置方式非常简单，基本上就一个链接就行，直接复制他的链接填上就行

目前分为了两种，一种是标准SSE,一种就是基于WebSocket SSE。

## 四、开发步骤

前面已经说明了相关的前期准备，JDK版本，Spring Boot版本，这两项是最重要的，一定要提前准备好，在做后面的开发

1
项目pom.xml配置

这是一个 基于 Spring Boot 3.5.5 + Spring AI 1.0.0 的 MCP Server 工程配置文件，用于实现一个通过 HTTP/SSE（Server-Sent Events）提供 AI 服务的 MCP-Server（Model Context Protocol Server）。

a
spring-ai-starter-mcp-server-webmvc 依赖

ⅰ
spring-ai-starter-mcp-server-webmvc 作用

这个依赖就是用来让你：

快速在 Spring Boot 中构建一个可供 AI 调用的 MCP Server

它提供了一整套自动化能力，包括：

功能

说明

MCP 协议实现

自动处理模型发来的 JSON-RPC / WebSocket 消息，符合 Model Context Protocol 规范。

Tool 注册机制

允许你编写 @McpTool

 或实现 McpTool

 接口的类，自动暴露为 AI 可用的“工具”。

WebMVC 适配

支持基于 HTTP 的 MCP 通信（即 spring-boot-starter-web

 兼容）。

工具描述生成

自动生成 MCP 元数据（Tool 名称、参数描述、功能定义），供 AI 模型理解如何使用。

与 LLM 集成

可与 Spring AI 的 OpenAI

、Ollama

、Anthropic

 等客户端整合，实现端到端 AI 能力。

ⅱ
典型应用场景

在车联网（V2X）或 IoT 系统中，你可以通过 MCP Server 让 AI：

场景

AI 可以做的事情

数据查询

“请帮我查询今天所有在线车辆的数目” → AI 调用 query()

 工具，执行 SQL 返回结果。

告警分析

“显示当前所有一级风险车辆” → AI 通过 MCP 调用你的告警服务接口。

报表生成

“帮我生成上周车辆能耗分析报告” → AI 从 Doris / MySQL 拉取聚合数据。

这些都由 spring-ai-starter-mcp-server-webmvc 帮你处理协议层通信。

ⅲ
它与其他依赖的关系

依赖

功能

spring-ai-starter-mcp-server-webmvc

提供 AI 调用接口（你是服务端）

spring-ai-starter-mcp-client

让你的应用去访问别的 MCP 服务（你是客户端）

spring-ai-openai-spring-boot-starter

提供与 OpenAI 模型交互的能力（AI 模型调用 MCP Server）

依赖定位是让外部 AI（比如 Cursor,Hiagent,Dify）可以调用你的业务逻辑的 Server

b
完整配置

2
开发代码

ⅰ
编写mcp-tool

1
总体结构

模块

涉及知识点

Spring Boot 服务层

使用 @Service

 标注业务组件，配合依赖注入（@Qualifier

）实现解耦。

Spring AI MCP Tool

通过 @Tool

 / @ToolParam

 注解暴露 AI 可调用的工具方法。

JDBC 操作 Doris

通过 JdbcTemplate

 操作 Doris 数据库，实现表信息与数据查询。

日志与异常处理

使用 Lombok 的 @Slf4j

 打印日志，异常捕获后返回结构化错误信息。

安全防护

限制 SQL 类型（仅 SELECT）、添加默认 LIMIT、防止 SQL 注入与爆量查询。

协议解析模块

调用 32960 协议解析工具类 Analysis32960

，验证、解析报文。

响应封装

使用统一的 Response

 对象（成功/失败），返回给上层或 AI 模型。

2
Spring AI (MCP Server) 相关知识点

注解

含义

@Tool

将方法暴露为 AI 可调用的工具函数（Tool），MCP Server 启动时会自动扫描注册。

@ToolParam

用于说明参数的语义和用途，帮助 AI 模型理解参数（用于自动生成 JSON Schema）。

MCP Server 框架

自动暴露这些工具为可被 ChatGPT / Claude 调用的 MCP 接口，比如 /mcp/tools/query

。

关键点：AI 模型并不是调用传统 API，而是通过 MCP 协议调用“工具（Tool）”。
@Tool 注解让普通业务方法变成“AI 能调用的函数”。

ⅱ
注册Mcp工具

1
@Bean

- 表示这是一个 Spring Bean，由 Spring 容器管理。

- MCP Server 会扫描容器中的 ToolCallbackProvider Bean，用来注册可被 AI 调用的工具方法。

2
ToolCallbackProvider

- Spring AI MCP 的接口，用于 提供工具回调对象。

- 核心作用：告诉 MCP Server 哪些对象的方法可以被 AI 调用。

- MCP Server 在收到模型请求时，会通过这个回调查找对应的工具方法。

3
MethodToolCallbackProvider.builder()

- MethodToolCallbackProvider 是 ToolCallbackProvider 的实现类。

- 它会扫描你提供的对象中的 所有带 @Tool 注解的方法。

- 然后生成 MCP 的工具映射（工具名称、方法、参数信息、描述信息）。

4
.toolObjects(parseTool)

- 将 AiBusinessTool 实例传给 MCP Server。

- MCP Server 会扫描这个对象中所有 @Tool 方法，例如：

  - query(String sql)

  - listAllTables()

  - getTableSchema(String tableName)

  - parseHeader(String hex)

  - parseRealTimeData(String hex)

- 这样 AI 模型就可以通过 MCP 协议调用这些方法。

5
.build()

- 构建 MethodToolCallbackProvider 对象，并注册到 Spring 容器中。

3
Sping AI 配置文件

写在 bootstrap.properties 中，这是SpringBoot框架的根配置文件

## 五、验证测试

怎么验证MCP-Server正确的开发完成，可以自己开发MCP clients ，也可以采用市面上成熟的AI集成应用比如HiAgent,Dify,Cursor, Claude等，它们已经包含了MCP clients 能力了，这里就不开发MCP clients 了，这里将采用Cursor和HiAgent做测试

1、Cursor测试Mcp-Server

配置cursor

Cursor创建一个新的项目，创建文件 .cursor ,在其下面再创建文件mcp.json，mcp.json就是放置mcp-server配置的地方，我们刚刚创建的Mcp-Server项目的名称为 dst-v2x-manager-service ，启动端口为：18522

1️⃣ "mcpServers"

- 顶层字段，表示 MCP Server 的注册列表。

- 可以同时配置多个 MCP Server，每个服务通过唯一名称区分。

2️⃣ "dst-v2x-manager-service"

- 这是 MCP Server 的 逻辑名称 / 服务名称，可自定义。

- 在客户端或者 Spring AI Agent 中调用时，可以通过这个名称引用对应的服务。

3️⃣ "type": "mcp"

- 指明这是一个 MCP Server 类型。

- Spring AI 支持多种连接方式（比如 openai、mcp），这里使用 mcp 表示标准 Model Context Protocol Server。

4️⃣ "url": "http://127.0.0.1:18522/sse"

- MCP Server 的 访问地址。

- http://127.0.0.1:18522/sse 表示 本地服务的 SSE（Server-Sent Events）端点。

- 这说明 MCP Server 使用 SSE 协议与客户端/模型通信。

- SSE 模式适合 持续推送事件 / 实时响应 的场景，比如 AI 模型订阅工具输出。

结果验证

打开 cusor 设置页面，点击 Tools / MCP ,会发现刚刚配置的Mcp-Server

打开启用

会发现底下的红点变为了绿点平且刚刚开发的工具都已经显示，代表配置成功

2、Hiagent测试Mcp-Server

配置MCP插件

接入mcp插件

地址就是http://127.0.0.1:18522/sse这个地址，暂时选用无认证方式，线上环境需要鉴权。

同步工具

这些都是自动生成的，同步成果代表mcp-server正常

发布MCP

## 六、成果展示

这里就用Hiagent做一个成果展示，因为我的Cusor账号无法登录了（国内封号很严重，如果不封号可以直接在聊天框选择Agent模式聊天，让他调用你的Mcp工具）

利用这个MCP创建一个Agent

a
创建Agent

 这里需要维护好提示词，以及配置刚刚MCP工具，这样一个简单的Agent就完成了

b
聊天测试

如果需要更清晰的回复，这里就需要配置模型，知识库，RAG等，这里就不过多赘述，主要先完成MCP-Server

,到这里就结束了，希望能给你带来帮助。

​

6 人点赞

Acho2025-11-06 18:111249
0
IP 属地广东
分享到：
正文
⌘ + ⇧ + F全屏编辑

回复
4181字
关于语雀使用帮助数据安全服务协议English
个人知识库
AI
搜索
⌘ + J
首页
AI工具
Claude国内使用攻略
AI学习
MCP
智能体搭建
如何快速实现AI助手
Agent学习：如何构建知识库
SpringAI开发MCP-Server（一）：0-1的搭建与实现
SpringAI开发MCP-Server（二）：安全验证以及工具调优
Agent Skills：从“提示词驱动”到“能力驱动”的智能体架构演进
skills / spec / 多-Agent
大纲
## 一、前提概要
实现流程
成果展示
## 二、什么是 MCP-Server?
### 1. 什么是MCP?
### 2. 什么是MCP-Server
## 三、开发前的准备
a. Spring AI 简介与架构认知
b. 环境与工具准备
c. 本项目MCP-Server采取的方式
## 四、开发步骤
### 1. 项目pom.xml配置
a. spring-ai-starter-mcp-server-webmvc 依赖
ⅰ. spring-ai-starter-mcp-server-webmvc 作用
ⅱ. 典型应用场景
ⅲ. 它与其他依赖的关系
b. 完整配置
### 2. 开发代码
ⅰ. 编写mcp-tool
### 1. 总体结构
2. Spring AI (MCP Server) 相关知识点
ⅱ. 注册Mcp工具
### 1. @Bean
### 2. ToolCallbackProvider
3. MethodToolCallbackProvider.builder()
### 4. .toolObjects(parseTool)
### 5. .build()
### 3. Sping AI 配置文件
## 五、验证测试
1、Cursor测试Mcp-Server
配置cursor
1️⃣ "mcpServers"
2️⃣ "dst-v2x-manager-service"
3️⃣ "type": "mcp"
4️⃣ "url": "http://127.0.0.1:18522/sse"
结果验证
2、Hiagent测试Mcp-Server
配置MCP插件
接入mcp插件
同步工具
发布MCP
## 六、成果展示
利用这个MCP创建一个Agent
a. 创建Agent
b. 聊天测试

> 本文整理自个人知识库，[查看语雀原文](https://www.yuque.com/u22429903/kf0kog/fu1vkpl1h736km77)。
