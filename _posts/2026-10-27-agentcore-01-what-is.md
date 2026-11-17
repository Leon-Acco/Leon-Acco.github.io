---
layout: post
title: "AgentCore 是什么：定位、作用与核心概念"
date: 2026-10-27
categories: [ai]
series: agentcore
series_order: 2
tags: [AgentCore, 阿里云, Agent]
original: https://help.aliyun.com/zh/agentcore/agentcore-product-overview
mermaid: false
math: false
toc: true
---
<div class="callout callout-tip" markdown="1">
**💡 返回 [AgentCore 学习指南（总览）](/2026/10/27/agentcore-00-overview/)**

</div>

## 一、是什么

AgentCore 是阿里云提供的**智能体构建和治理平台**，支持在统一控制台中创建和管理 Agent：接入多家模型供应商、集成外部工具（MCP 服务）、实时监控 Agent 运行状态，并支持多成员、多角色的团队协作与权限管理，满足企业级安全合规要求。

官方将其能力归为三组：

| 能力组 | 覆盖内容 |
| --- | --- |
| Agent 构建 | Agent 管理、Agent 模板、模型连接、Skill 管理、工具管理（MCP）、凭证管理 |
| Agent 协作 | Team 协作（多 Agent 组队）、账号集成（钉钉/微信/飞书） |
| 身份与权限 | 用户管理、RAM 访问控制 |

<div class="callout callout-note" markdown="1">
**📝 与官网表述的对应**

官网产品页：「Agent 构建与运行、协作与治理、观测与评估」三大核心能力。帮助中心拆为「Agent 构建 / 资产管理 / Agent 协作 / 身份与权限」四组，两者是同一能力的不同切分。
</div>

## 二、有什么作用（解决什么问题）

平台定位是**统管多来源智能体**：自研 Agent、开源 Harness（Claude Code / Codex / OpenClaw / Qoder CLI 等）、商业 SaaS Agent 都能纳到一个平台下统一构建、运行、协作、观测和治理。

### 官方应用场景

| 场景 | 玩法 |
| --- | --- |
| 数字员工 | 岗位型智能体，Harness 配置化创建即可上线，IM 渠道直连办公场景（入职办理、单据审核、工单值守） |
| 智能客服 | 单 Agent 承接对外咨询，沙箱运行时弹性伸缩抗高峰，身份权限约束数据访问 |
| 智能问数 | 自然语言提问 → Agent Team 拆任务 → 多 Agent 经 MCP 接数据源分别取数/建模/归因 |
| AI 编程助手 | 存量编程 Agent 原样纳管、按需组队；研发全链路可观测可干预，Token 消耗可归因可审计 |

### 平台边界（重要）

<div class="callout callout-warning" markdown="1">
**⚠️ AgentCore 不管这些**

- **不存模型本身**：模型来自外部供应商（百炼、OpenAI 兼容端点等），Token 费用由供应商计
- **不管理知识库文档**：知识库连接只做「接入与检索」，文档上传/切片/索引由 OTS 或百炼负责
- **不自动保存对话历史**：会话标识只表示请求归属，历史保存由应用或 Harness 实现（详见 [AgentCore 实战：创建 Agent 与 AG-UI 调用](/2026/10/27/agentcore-05-practice/) 的会话机制）
</div>

## 三、核心概念

资源以 **Workspace** 为边界组织，Workspace 内独立维护模型配置、Agent、Team 和 MCP 工具。

| 概念 | 定义 |
| --- | --- |
| Workspace | 资源隔离和管理的基本单元，按项目/业务线分配 |
| Agent | 执行 AI 任务的单元，运行时关联一个模型，可绑定 Skill、MCP、凭证 |
| Agent 模板 | 可复用的 Agent 配置模板，快速批量部署同类型智能体 |
| Team | 多个 Agent 组成的组织形式，在 Agent 之上再加一层编排 |
| Skill | Agent 的专业技能，内容以 SKILL.md 定义，支持 ZIP / Nacos CLI / Markdown / 预置模板导入 |
| MCP | Model Context Protocol，为 Agent 扩展标准化外部工具调用（远程连接 / HTTP 转 MCP） |
| 模型连接 | 接入模型供应商的通道：供应商 + API 端点 + 协议 + 访问凭证 |
| 模型 | 连接下可选的推理模型，记录上下文长度、单次最大输出及能力项 |
| 凭证 | 访问外部服务的密钥，Key-Value（环境变量名→值）集中保存，创建后 Value 默认隐藏 |
| 记忆空间 | 跨运行持续保存信息的空间，系统从写入内容提炼长期记忆（MCP 或 SDK 接入） |
| 知识库连接 | AgentCore 与已有外部知识库（OTS / 百炼）之间的访问关系，检索能力以 MCP 工具提供 |

### Agent 的四种形态

| 形态 | 说明 | 适合 |
| --- | --- | --- |
| Harness Agent | 平台提供运行框架（QwenPaw / Qoder），控制台配置即建即用，无需开发部署 | 快速搭建、数字员工 |
| 高代码 Agent | 容器镜像或代码包部署自己的 HTTP 服务，自定义业务逻辑 | 精细控制、复杂逻辑 |
| 纳管 Agent | 把已部署在**本地**的 Agent 纳入平台管理，无需改造即参与 Team 编排 | 存量 Agent（如本地 Claude Code） |
| 注册 Agent（待上线） | 部署在平台之外的 Agent 注册进来统一调度 | 未来形态 |

### 两种托管 Harness：QwenPaw vs Qoder

| 能力项 | QwenPaw | Qoder |
| --- | --- | --- |
| 模型来源 | 平台配置切换模型与供应商 | 仅 Qoder 原生模型 |
| BYOK（自带密钥）/ 通用供应商 | 支持 | 不支持 |
| 上下文压缩 | QwenPaw 原生机制 | Qoder 原生机制 |
| 其余能力 | 两者一致（Shell/文件工具、Subagent、Skill、AG-UI、IM 渠道等） | 同左 |

<div class="callout callout-tip" markdown="1">
**💡 选型口径**

要在平台侧切换模型或供应商 → QwenPaw；用 Qoder 原生模型 → Qoder。
</div>

## 四、运行底座与依赖

| 依赖产品 | 用途 |
| --- | --- |
| 云监控 CMS | 写入 Tracing 数据 |
| PrivateLink | 私网访问 |
| SchedulerX3 | 分布式任务调度（Team 定时任务），需购买 |

支持地域：华东1（杭州）、华东2（上海）、华南1（深圳）、华北2（北京）、新加坡。

## 相关笔记

- [AgentCore 产品功能详解](/2026/10/27/agentcore-02-features/)
- [AgentCore 实战：创建 Agent 与 AG-UI 调用](/2026/10/27/agentcore-05-practice/)
- [AgentCore 计费说明与使用约束清单](/2026/10/27/agentcore-08-billing/)
- [AgentCore 与主流 Agent 平台对比](/2026/10/27/agentcore-03-comparison/)
