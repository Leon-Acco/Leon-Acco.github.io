---
layout: post
title: "AgentCore 产品功能详解"
date: 2026-10-27
categories: [ai]
series: agentcore
series_order: 3
tags: [AgentCore, 阿里云, Agent, MCP]
original: https://help.aliyun.com/zh/agentcore/operating-guide/
mermaid: false
math: false
toc: true
---
<div class="callout callout-tip" markdown="1">
**💡 返回 [AgentCore 学习指南（总览）](/2026/10/27/agentcore-00-overview/) · 概念基础见 [AgentCore 是什么：定位、作用与核心概念](/2026/10/27/agentcore-01-what-is/)**

</div>

控制台导航按「概览 / AGENT 构建 / 资产管理 / AGENT 协作 / 身份与权限 / 系统设置」组织（实拍结构见 [AgentCore 学习指南（总览）](/2026/10/27/agentcore-00-overview/) 截图）。

## 一、AGENT 构建

### 1. Agent 管理

创建入口分「托管 Harness」与「高代码」两条路，另有本地纳管形态。托管 Harness 创建时的配置区：

| 配置区 | 配置项与要点 |
| --- | --- |
| 基本信息 | Harness（QwenPaw/Qoder）、Agent 模板、名称、模型连接、模型、Token 配额（日/月用量上限，可选） |
| 文件配置 | 系统提示词 → 写入 SOUL.MD 与 AGENTS.MD（用模板时由模板提供） |
| Agent 资产配置 | Skill、Subagent（名称/角色/职责）、工具（MCP）、知识库（**托管 Harness 暂不支持**）、凭证 |
| 资源配置 | 实例规格（CPU/内存）、实例最大 Session 数、Session TTL（秒）、自动扩缩容（最小/最大实例数） |
| 网络配置 | 允许默认网卡访问公网 / 允许访问 VPC |
| 存储配置 | OSS 挂载（最多 10 个挂载点，路径必须在 `/mnt/oss/` 下且不重复） |

<div class="callout callout-note" markdown="1">
**📝 内置能力与热更新**

- 内置 Shell 和文件工具，无需额外添加；预装 Chromium，通过 mcporter 调用 MCP；内置搜索有免费额度，生产需另配
- 模型、提示词、Skill、Subagent、MCP、凭证修改**热更新**；实例规格、镜像、挂载点需**重启生效**
</div>

高代码 Agent 部署细节见 [AgentCore 实战：创建 Agent 与 AG-UI 调用](/2026/10/27/agentcore-05-practice/)。

### 2. Agent 模板

可复用的角色文件 + 资产配置。使用模板时可额外添加 Skill，但 Skill 名称不能重复；模板或 Skill 发布新版本后需核对已绑定版本。

### 3. 连接器

对接外部渠道（IM 等），详见「账号集成」。

## 二、资产管理

### 1. 模型连接

- 预制供应商（如阿里云百炼）或自定义供应商（自填 API 端点），目前仅支持手动输入 API Key
- 协议示例：OpenAI/v1
- 模型能力标签：工具调用 / 多工具调用 / 流式工具调用 / 视觉 / 音频 / 视频 / 文档 / 推理
- 调用参数：上下文长度、单次最大输出、流式输出
- 支持在线调试模型

### 2. Skill 管理

- **创建方式**：上传 ZIP 技能包 / Nacos CLI 导入 / Markdown 直接定义 / 从 Skill 广场导入三方 Skill
- **审核与发布**：新版本为草稿 → 提交审核（AI 安全护栏，独立产品需开通）→ 通过后发布；未开通护栏可直接发布
- **版本管理**（设计得比较完整）：

| 概念 | 可变性 | 用途 |
| --- | --- | --- |
| 工作区版本（HEAD） | 持续修改，保存即生效 | 开发调试 |
| 正式版本（如 1.0.0） | 创建后不可变 | 稳定使用、回滚、审计 |
| 版本别名 latest | 系统自动指向最新正式版本 | 自动跟随发布 |
| 自定义别名 | **尚未开放** | 多环境/灰度（开放前用 latest 或具体版本号） |

<div class="callout callout-warning" markdown="1">
**⚠️ 生产环境不要引用工作区版本**

HEAD 的任何修改立即影响使用方。生产建议引用 latest、具体版本号（自定义别名开放后再切换）。
</div>

### 3. 工具管理（MCP）

| 配置项 | 说明 |
| --- | --- |
| 工具类型 | MCP（远程连接 / HTTP 转 MCP）；虚拟 MCP（重组已有工具，即将上线） |
| URL | 服务地址，要求网络可达 |
| Transport | SSE 或 Streamable HTTP |
| 认证 | 可选启用：凭证类型 + 携带位置（自定义 Header）+ 值 |
| 描述 | 不超过 1000 字符 |

### 4. 知识库

连接**已有的**外部知识库（表格存储 OTS / 百炼知识库），不迁移文档；AgentCore 自动生成一个 MCP 工具提供检索能力，挂到 Agent 后生效。

| 约束 | 说明 |
| --- | --- |
| 一对一 | 一个连接对应一个外部知识库，多个库分别建连接 |
| 参数定死 | 模板托管状态下实例/业务空间/知识库在创建时确定，要换就删掉重建 |
| 单向转换 | 「转自由编辑」不可逆：可改工具名/参数 Schema/加自定义逻辑，但回不去模板托管 |
| 职责边界 | 文档上传、切片、索引、检索配置都在原知识库产品侧维护 |

### 5. 记忆空间

三层区分（容易混）：

| 概念 | 内容 |
| --- | --- |
| Agent 对话历史 | Agent 实际交互，**不会自动同步**到记忆空间 |
| 记忆空间中的历史 | 实际写入该空间的原始对话 |
| 长期记忆 | 系统从写入内容中**提炼**出的可持续信息 |

- 接入方式：MCP 挂载（标准工具）或 SDK 编排（代码控制读写）
- 三种使用模式：单 Agent 单空间 / 多 Agent 多空间（按业务域隔离）/ 统一入口 Agent 按场景路由
- 读写规则靠系统提示词或封装成 Skill 引导；挂载 ≠ 自动同步

### 6. 凭证管理

Key-Value 形式（Key 为环境变量名如 `OPENAI_API_KEY`），集中存放于 Workspace，被模型连接和工具引用，避免明文散落。安全建议：最小权限、定期轮换、泄露立即吊销。

### 7. 镜像缓存（公测）

加速容器镜像部署（详情见官方文档）。

## 三、AGENT 协作（Team）

- Team 是 Agent 之上的组织层；协作客户端（AgentTeams WEB）登录时需配置 Homeserver URL（协作 API 连接点公网地址）
- **任务看板**：待确认 / 已创建 / 执行中 / 已完成 / 已取消；任务有优先级（紧急/高/中/低）、指派 Agent、归属团队
- **定时任务**：基于 SchedulerX，支持周期（每天 19:00、每小时第 30 分）与手动执行，有执行记录

<div class="callout callout-note" markdown="1">
**📝 纳管 Agent 的协作限制**

本地纳管 Agent 只能作为 Team Worker，不能当 Leader，且每个纳管 Agent 仅可加入一个 Team。
</div>

## 四、身份与权限

- 用户管理：多成员多角色
- 账号集成：将 Agent 接入钉钉 / 飞书 / 企业微信——IM 平台建机器人拿凭证 → Agent 外部渠道配置绑定 → 测试群 @机器人验证
- RAM 访问控制：云账号级权限体系

## 五、观测大盘

前提：依赖云监控 CMS。四块分析视图：

| 视图 | 内容 |
| --- | --- |
| 请求概览 | 调用量、状态分布 |
| Token 分析 | Token 消耗构成（可归因到 Agent/任务） |
| 模型分析 | 各模型调用情况 |
| 工具调用分析 | MCP 工具调用情况 |

配套 AgentLoop Trace（托管 Harness 支持查看执行过程）。

## 相关笔记

- [AgentCore 实战：创建 Agent 与 AG-UI 调用](/2026/10/27/agentcore-05-practice/)
- [AgentCore 计费说明与使用约束清单](/2026/10/27/agentcore-08-billing/)
