---
layout: post
title: "AgentCore 高代码与纳管 Agent 部署手册"
date: 2026-10-27
categories: [ai]
series: agentcore
series_order: 8
tags: [AgentCore, 阿里云, 部署, Docker, ACR]
original: https://help.aliyun.com/zh/agentcore/high-code-agent-deployment-guide
mermaid: false
math: false
toc: true
---
<div class="callout callout-tip" markdown="1">
**💡 返回 [AgentCore 学习指南（总览）](/2026/10/27/agentcore-00-overview/) · 基础流程见 [AgentCore 实战：创建 Agent 与 AG-UI 调用](/2026/10/27/agentcore-05-practice/) · 代码侧开发见 [AgentCore SDK 开发指南（Java 为主）](/2026/10/27/agentcore-04-sdk-java/)**

</div>

高代码 Agent 有两条部署路径：**容器镜像**（自定义环境最灵活）与**代码包**（平台运行时，上手快）。另有**本地纳管**形态，把机器上已有的 CLI Agent 接入 Team 协作。

## 一、容器镜像部署

### 镜像硬性要求

| 检查项 | 要求 |
| --- | --- |
| 架构 | Linux x86_64（linux/amd64） |
| Shell | 必须提供可执行的 `/bin/bash` 和 `/bin/sh` |
| 运行产物 | 镜像内包含应用、依赖和启动所需文件 |
| 镜像仓库 | 企业版 ACR，开启实例公开匿名拉取 + 公开仓库 |
| 网络 | Agent 运行环境能访问镜像仓库（与公网/VPC 路径匹配） |
| 应用监听 | `0.0.0.0`，并实现健康检查与对话接口 |

<div class="callout callout-warning" markdown="1">
**⚠️ 镜像里不要放密钥和业务数据（公开仓库可被匿名拉取）。**

</div>

### 八步流程

1. 准备容器镜像（对照上表）
2. 创建 Agent 并选择镜像
3. 配置启动方式和执行角色
4. 声明对话协议（Chat Completions → `/openai/v1/chat/completions`；AGUI → `/ag-ui/agent`；可同时选，Path 以 `/` 开头不含域名）
5. 配置运行资源和健康检查
6. 配置网络（公网 / VPC）和日志
7. 提交创建并验证
8. 更新镜像（迭代）

<div class="callout callout-note" markdown="1">
**📝 勾选协议不会自动生成接口——实际路由、响应格式必须与声明一致，否则控制台调试失败。**

</div>

## 二、代码包部署

流程与镜像部署一致（步骤 2 选运行时、步骤 3 提供代码包），平台负责构建运行环境。首次体验可直接用平台示例（官方教程：使用 Python SDK 快速创建高代码 Agent）。

## 三、服务端运行约定（框架无关）

以 Java SDK 的 `AgentCoreServer` 为例（详见 [AgentCore SDK 开发指南（Java 为主）](/2026/10/27/agentcore-04-sdk-java/)）：

| 配置项 | 值 |
| --- | --- |
| 运行时 | JRE 17+（Java 场景） |
| 监听地址 | `0.0.0.0` |
| 端口 | 9000（显式指定） |
| AG-UI 接口 | `POST /agui`（**Java SDK 0.1.0 默认路径，勿照搬其他语言的 `/ag-ui/agent`**） |
| OpenAI 兼容接口 | `POST /openai/v1/chat/completions` |
| 存活探针 | `GET /healthz`（不判断下游资源状态） |
| 就绪探针 | `GET /readyz`（资源初始化完成后才就绪） |

验证顺序：`/healthz` → `/readyz` → 发一次真实请求，确认正常结束事件而非 RUN_ERROR。

## 四、镜像缓存（公测，暂不计费）

加速 Agent 启动时拉取镜像，属于 Workspace 级资源。

| 要点 | 说明 |
| --- | --- |
| 匹配规则 | 按镜像 **Digest** 匹配复用；同一普通 Tag 不支持覆盖推送（latest 除外） |
| 前置授权 | 需授予 AgentCore 读源镜像的权限（与 Agent 执行角色无关） |
| ACR 网络 | 公网拉取：为 Workspace 配公网白名单（面板会提示出口 IP）；VPC 拉取：配 VPC 访问 + 开「允许访问 VPC」 |
| 计费 | 公测期免费，正式商用后计费 |

## 五、本地纳管 Agent：零改造接入 Team

### 前提

- Workspace 已开启 Team 协作；已创建 Team（纳管 Agent 只能当 **Worker**、不能当 Leader、仅可加入一个 Team）
- 本地已装运行时并满足版本要求：

| 组件 | 最低版本 |
| --- | --- |
| Claude Code | **2.1.141+**（低版本不支持纳管配置参数） |
| Codex / OpenClaw / Qoder CLI | 满足文档版本要求 |
| LoongSuite Pilot | 无需预装，启动命令自动安装/升级（迁移机器、轮换凭据后须重新完整执行） |

### 流程

1. **创建纳管 Agent**：控制台创建，选模型来源（平台或本地）、运行时，可绑系统提示词、Skill 和 MCP
2. **生成启动命令**：Agent 运行中后在详情页生成——
      - 选择访问网络：内网（同 VPC / 专线 / VPN）或公网（办公电脑、跨网络服务器）
      - 选择操作系统（Linux/macOS/Windows）与运行时（须与本机实际一致）
      - 命令包含连接凭据 + 云监控配置；Trace 上报至云监控 2.0 工作空间
3. **本地执行完整命令**：两步连续执行（Pilot 安装/升级 + 启动）
4. **首次接入验证**：验证 Pilot 与服务状态；连接正常但运行数据缺失时，检查网络选择与目标地址连通性

<div class="callout callout-warning" markdown="1">
**⚠️ 令牌安全**

启动命令中的令牌**明文仅在生成页面展示一次**，关闭页面无法再查看，需要时重新签发。重新接入、迁移机器或轮换凭据 → 重新生成启动命令。

**网络/数据上报**
网络选择同时影响 Agent 连接与运行数据上报；本地 Skill/MCP 继承功能依赖当前启动命令安装的 Pilot 版本，不能只看版本号。
</div>

## 相关笔记

- [AgentCore SDK 开发指南（Java 为主）](/2026/10/27/agentcore-04-sdk-java/)
- [AgentCore 产品功能详解](/2026/10/27/agentcore-02-features/)
- [AgentCore 计费说明与使用约束清单](/2026/10/27/agentcore-08-billing/)
