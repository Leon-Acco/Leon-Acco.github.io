---
layout: post
title: "Claude Code 跨项目开发：一个项目依赖另一个项目的实操方案"
date: 2026-05-20
categories: [tools]
mermaid: false
math: false
toc: true
---
## 问题场景

你有两个项目（Project-A 和 Project-B），Project-B 的开发**依赖于 Project-A 的接口、数据模型或业务逻辑**。你需要在 Claude Code 中高效地完成 Project-B 的开发，同时确保与 Project-A 的契约保持一致。

---

## 核心挑战

1. **Claude Code 默认只感知当前工作目录**，无法自动"看到"另一个项目的代码
2. **跨项目的接口契约容易脱节**，手动对齐效率低且易出错
3. **两个项目可能需要联动修改**，但又要避免互相干扰

---

## 解决方案

### 方案一：`@` 绝对路径引用（最快上手）

**适用场景：** 只需要参考另一个项目的部分代码，不需要同时修改两个项目。

**操作方式：**

```bash
# 在 Project-B 中启动 Claude Code
cd /path/to/project-b
claude

# 对话中直接引用 Project-A 的文件或目录
> 请参考 @/path/to/project-a/src/api/UserService.java 的接口定义，
> 在当前项目中实现对应的 Feign 客户端
```

**优点：** 零配置，即用即走

**局限：** 每次都需要手动指定路径，Claude 不会自动记住依赖关系

---

### 方案二：`CLAUDE.md` 声明依赖关系（推荐日常开发）

**适用场景：** 长期开发中 Project-B 持续依赖 Project-A，需要 Claude 每次启动都知道上下文。

**操作方式：**

在 Project-B 根目录的 `CLAUDE.md` 中写入：

```markdown
# Project-B 项目上下文

## 技术栈
- Java 17 / Spring Boot 3.x / Maven
- 数据库：MySQL 8.0

## 依赖项目
- 本项目（Project-B）是消费端，依赖 Project-A（服务端）
- Project-A 本地路径：/home/user/project-a
- 核心接口定义：/home/user/project-a/src/main/java/com/example/api/
- 数据模型定义：/home/user/project-a/src/main/java/com/example/model/
- 接口文档（如有）：/home/user/project-a/docs/api.md

## 开发约束
- 调用 Project-A 的接口时，入参/出参必须与其契约严格一致
- 枚举值、错误码直接引用 Project-A 的定义，不要自行定义
- 新增接口调用前，先读取 Project-A 对应的 Controller/Service 确认行为
```

**效果：** 每次启动 Claude Code 时自动加载，Claude 在整个会话中都知道去哪里找依赖项目的代码。

---

### 方案三：Plan Mode 先规划再执行（复杂需求推荐）

**适用场景：** 需求涉及两个项目的多个模块，改动面较大，需要先理清思路再动手。

**操作步骤：**

```bash
# Step 1：进入 Plan Mode（只读，不做任何修改）
claude --permission-mode plan

# Step 2：让 Claude 同时分析两个项目
> 分析 @/path/to/project-a/src/api/ 的所有接口定义，
> 对比当前项目 @src/ 已有的客户端实现，
> 列出缺失的接口和数据模型差异，给出实现方案

# Step 3：确认方案后，按 Shift+Tab 切换到默认或自动接受模式

# Step 4：按方案逐步执行
> 按照刚才的方案，先实现 UserService 的 Feign 客户端
```

**优点：** 避免盲目修改，方案经确认后再执行，适合大改动

---

### 方案四：Subagent 并行研究（高效分析）

**适用场景：** 需要深入分析 Project-A 的代码，但不想让大量中间输出占用主会话上下文。

**操作方式：**

```bash
# 让 Explore 子代理（只读、独立上下文）去分析 Project-A
> 用 Explore 子代理分析 /path/to/project-a/src/api/ 下的所有接口，
> 整理出接口清单（方法签名、入参、出参、异常），
> 然后在当前项目中根据清单实现对应的客户端
```

**优点：** 子代理使用独立上下文，分析 Project-A 的大量文件不会"撑爆"主会话

---

### 方案五：Git Worktrees 联动修改（需要同时改两个项目）

**适用场景：** 两个项目在同一仓库（monorepo），或者需要在两个项目上同步做改动。

**操作方式：**

```bash
# 为两个项目各起一个终端
# 终端 1：Project-A（服务端）
cd /path/to/project-a && claude

# 终端 2：Project-B（消费端）
cd /path/to/project-b && claude

# 先在终端 1 中完成 Project-A 的接口变更
# 然后在终端 2 中让 Claude 读取最新的 Project-A 代码来适配
```

---

## 方案选型速查

| 场景 | 推荐方案 | 理由 |
| --- | --- | --- |
| 偶尔参考另一个项目的某个文件 | `@` 路径引用 | 最快，零配置 |
| 长期开发，持续依赖另一个项目 | `CLAUDE.md` 声明依赖 | 一次配置，自动生效 |
| 大改动，需要先出方案再执行 | Plan Mode | 先理清再动手，减少返工 |
| 需要深度分析依赖项目的大量代码 | Subagent | 隔离上下文，不污染主会话 |
| 两个项目需要同步修改 | Git Worktrees / 双终端 | 各自独立，互不干扰 |

---

## 实战建议

<div class="callout callout-tip" markdown="1">
**💡 💡**

**组合使用效果最佳：**
4. 在 `CLAUDE.md` 中声明依赖关系（长期有效）
5. 复杂需求先用 Plan Mode 规划
6. 深度分析用 Subagent 隔离
7. 确认方案后切换到自动接受模式高效执行

[!warning] ⚠️
**注意事项：**
- 确保 Project-A 的代码是最新的（先 `git pull`），否则 Claude 会基于过期代码开发
- 如果 Project-A 有接口文档（Swagger/OpenAPI），优先让 Claude 读文档而非源码，效率更高
- 大项目建议在 `CLAUDE.md` 中只列出**关键目录**，避免让 Claude 读取过多无关代码
</div>
