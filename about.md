---
layout: default
title: 关于我
description: Acho — Java 后端工程师，专注车联网数据平台、实时计算与 AI 工程化。
---
<section class="page-head">
  <div class="wrap">
    <h1 class="page-title">关于我</h1>
    <p class="page-sub">Java 后端工程师 · 车联网数据平台 · AI 工程化实践者</p>
  </div>
</section>

<div class="wrap">
  <section class="about-intro">
    <img class="about-face" src="{{ '/assets/img/me/portrait.jpg' | relative_url }}"
         alt="Acho" width="760" height="1013" loading="lazy">
    <p class="about-lead">
      我是 Acho，一名在深圳工作的 Java 后端工程师。六年多时间都在车联网（IoV）
      这条链路上：设备接入、指令下发、国标协议解析、位置与车况数据管道、实时计算与数据仓库。
      近几年把相当一部分精力放在 AI 工程化上，做 Agent runtime、MCP 工具化与团队级 AI Coding 流程。
    </p>
  </section>

  <section class="about-block">
    <h2>我在做什么</h2>
    <div class="body">
      <p>
        日常工作是设计与维护车联网数据平台的后端链路：车辆与 T-Box 的海量接入、
        GB/T 32960 协议的解析与转发、设备影子与指令控制、里程与风险聚合的实时计算。
        关注的是吞吐、稳定性与数据口径的一致性——一个字段的口径错了，下游所有报表都得重做。
      </p>
      <p>
        技术之外，我习惯把做过的事写下来。这个站点是我从个人知识库整理出的公开部分，
        包含协议与架构设计、中间件原理、Flink 与 OLAP 实践，以及 AI 应用开发的学习路径。
      </p>
    </div>
  </section>

  <section class="about-block">
    <h2>技术方向</h2>
    <div class="body">
      <dl>
        <dt>语言与框架</dt>
        <dd>Java（JDK 21 生产实践，虚拟线程）· Spring Boot / Spring Cloud / Spring AI · MyBatis-Plus</dd>
        <dt>网络与协议</dt>
        <dd>Netty · TCP/IP · GB/T 32960、JT/T 808/809 车载通信协议</dd>
        <dt>消息与流计算</dt>
        <dd>Kafka · RocketMQ · RabbitMQ · Flink（状态、窗口、CDC 广播流）</dd>
        <dt>存储与查询</dt>
        <dd>MySQL / PostgreSQL · Redis / Redisson · Apache Ignite · Doris / SelectDB</dd>
        <dt>AI 工程化</dt>
        <dd>Agent runtime 与工具调用 · MCP-Server（Spring AI）· RAG · Claude Code 工作流与 Skills</dd>
      </dl>
      <p style="margin-top:1.4rem">
        <span class="stack-list">
          <span>Java</span><span>Spring Boot</span><span>Netty</span><span>Kafka</span>
          <span>RocketMQ</span><span>Flink</span><span>Doris</span><span>Redis</span>
          <span>Ignite</span><span>MySQL</span><span>Docker</span><span>Linux</span>
          <span>Spring AI</span><span>MCP</span><span>RAG</span><span>Claude Code</span>
        </span>
      </p>
    </div>
  </section>

  <section class="about-block">
    <h2>写作</h2>
    <div class="body">
      <p>
        文章先在个人知识库里长成笔记，成体系后再整理到这里。目前有
        <a href="{{ '/java/' | relative_url }}">Java 进阶</a>、
        <a href="{{ '/iot/' | relative_url }}">车联网</a>、
        <a href="{{ '/bigdata/' | relative_url }}">大数据</a>、
        <a href="{{ '/ai/' | relative_url }}">AI 与智能体</a>
        四个方向，另有若干成篇的专题系列。
      </p>
      <p>
        更早的、更随手的记录散在语雀与掘金；结构化程度高的内容，我都会往这里搬一次。
      </p>
    </div>
  </section>

  <section class="about-block">
    <h2>经历</h2>
    <div class="body">
      <p>
        航空维修专业出身，修过飞机，后来转做软件。这件事对我的影响是：
        更在意系统里那些"看不见但会要命"的细节——一次静默失败、一个没有边界的重试、一份没有监控的链路。
      </p>
      <p>
        现在在车联网平台做后端研发，从协议接入一直做到数据仓库与分析口径；
        同时负责团队内的 AI 开发工具链与规范落地。
      </p>
    </div>
  </section>

  <section class="about-block">
    <h2>联系</h2>
    <div class="body">
      <ul>
        <li>邮箱：<a href="mailto:acho_0778@163.com">acho_0778@163.com</a></li>
        <li>GitHub：<a href="https://github.com/Leon-Acco" target="_blank" rel="noopener">github.com/Leon-Acco</a></li>
        <li>语雀：<a href="https://www.yuque.com/u22429903" target="_blank" rel="noopener">yuque.com/u22429903</a></li>
        <li>掘金：<a href="https://juejin.cn/user/4191761160999496" target="_blank" rel="noopener">juejin.cn/user/4191761160999496</a></li>
      </ul>
      <p style="margin-top:1rem">
        如果你在这些平台上看到 Acho 这个 ID，大概率就是我。
        对文中的内容有疑问、发现错误，或者想聊车联网与 AI 工程化，都欢迎直接联系。
      </p>
      <p class="about-hiring">
        目前正在看新的机会，如果有老板感兴趣，麻烦联系我。
      </p>
    </div>
  </section>
</div>
