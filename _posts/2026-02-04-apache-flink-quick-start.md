---
layout: post
title: "Apache Flink 快速入门"
date: 2026-02-04
categories: [bigdata]
tags: [Flink, 流计算, Kafka]
original: https://www.yuque.com/u22429903/dymeak/cyc94t4gkotr6gk1
---

# Apache Flink 快速入门
目标

 	希望通过这份内容抛砖引玉，让你在阅读后不仅能独立编写 WordCount、接入 Kafka、使用 MySQL/Redis，对 Flink 作业有一个整体、清晰的认识。

## 一、第一部分：Flink 到底是什么？为什么要用它？

### 1.1 一句话定义

Flink 是一个“真·实时”流计算引擎，专门用来在数据不断产生的时候就立刻算出结果。

Flink核心目标，是“数据流上的有状态计算”(Stateful Computations over Data Streams )具体说明:Apache Flink是一个框架和分布式处理引擎，用于对无界和有界数据流进行有状态计算。（可以提一嘴，流批一体）

### 1.2 流处理的三个历史阶段

阶段

代表框架

延迟

状态管理

Exactly-once

典型问题

第一代

Storm / Heron

毫秒级

非常弱

非常麻烦

状态小、故障恢复难

第二代

Spark Streaming

秒级（微批）

有状态

有（但复杂）

延迟高、不能真毫秒级

第三代（当前最优）

Apache Flink

毫秒级

超大状态

原生支持

几乎没有短板

结论：Flink 是目前唯一一个同时满足下面 5 个要求的开源引擎

1
毫秒级延迟

2
高吞吐

3
超大状态（几十 TB 都行）

4
原生 Exactly-once

5
事件时间 + 水印（处理乱序）

所以一句话：现在公司里只要是“实时”两个字，基本默认就用 Flink。

### 1.3 Exactly-once

Exactly-once（精确一次）是流处理系统中最严格、最重要的“数据处理语义”保证。 简单来说，就是：不管程序挂了多少次、重启多少次、机器宕机多少次，最终每一条数据都恰好被正确处理一次，既不多也不少。

通俗的比喻解释

处理语义

比喻（你给女朋友发微信“到家了”）

后果

At-most-once

发了就不管了，网络断了女朋友也收不到

数据丢了（最轻松，但不能接受）

At-least-once

怕丢了就狂发，发 100 次确认她一定收到

数据重复了（女朋友炸了：你发 100 次干嘛）

Exactly-once

不管网络怎么抖、重连多少次，女朋友手机上永远只出现一次“到家了”

完美（既不丢也不重）

## 二、第二部分：Flink 核心概念

Flink分层API

有状态流处理：通过底层API(处理函数)，对最原始数据加工处理。底层API与DdaSteamAPI相集成，可以处理复杂的计算。

DataSteamAPI(流处理) 和 DataSetAPI(批处理)封装了底层处理函数，提供了通用的模块，比如转换(tansfonmations，包括fatmap等)，连接(joins)，聚合(aggregations)，窗口(windows)操作等。注意:Fnk1.12以后，DataSteamAPI已经实现map、真正的流批一体，所以DataSetAPI已经过时。

TableAPI是以表为中心的声明式编程，其中表可能会动态变化。TableAPI遵循关系模型:表有二维数据结构，类似于关系数据库中的表;	同时API提供可比较的操作，例如select、project、join、group-by、aggregate等。我们可以在表与 DataSteamDataset之间无缝切换，以允许程序将 TableAPl与DataStream 以及 DataSet 混合使用。

SOL这一层在语法与表达能力上与Table API类似，但是是以SOL查询表达式的形式表现程序。SOL抽象与tIabe API交互密切，同时SOL查询可以直接在Table API定义的表上执行.

Flink 运行时架构

这里以单列部署模式为列

作业管理器（JobManager）

JobManager = Flink 集群里的大脑 + 管家

一句话总结： JobManager 是整个 Flink 作业的“总指挥官”，负责接收你提交的代码、把作业拆成小任务、给每个 TaskManager 发命令、协调 Checkpoint、监控所有任务的生死，一旦出事就负责重启恢复。

它干的三件最核心的事：

1
把你的代码（JobGraph）变成可以并行执行的图（ExecutionGraph）

2
决定每个小任务跑在哪个 TaskManager 的哪个 slot 上

3
全局协调 Checkpoint 和故障恢复（Exactly-once 的真正指挥官）

一个 Flink 集群永远只有一个活跃的 JobManager（高可用时有 standby），所有作业都归它管，它挂了 = 整个集群所有作业全停。

（部署模式等细节可以去看官网教程，这里就不阐述部署了）

任务管理器（TaskManager）

TaskManager 是真正执行计算的 JVM 进程，负责接收 JobManager 派发的子任务（subtask），在自己的 Slot 里跑算子代码、管理本地状态、做 Checkpoint 快照、跟其他 TaskManager 交换数据，它是集群里“干体力活”的地方。

Flink 核心概念

1
并行度（Parallelism）

1.1
并行度（Parallelism）一句话概述

并行度 = 一个算子同时有多少个并行实例在干活，它直接决定你的作业能多快、能吃多少数据。

用大白话： 想象你要把 1000 个苹果削皮。

- 并行度 = 1 → 你一个人削，慢慢来

- 并行度 = 8 → 8 个人一起削，理论上快 8 倍 Flink 里每一个算子（source、map、keyBy/window、sink）都可以单独设置并行度。

1.2
并行度到底决定什么？

1
作业整体并行度 = 所有算子中最大的并行度

2
并行度越高，吞吐越大（前提是下游跟得上、集群 slot 够）

3
并行度必须 ≤ 集群总 slot 数，否则作业起不来

1.3
实际生产中最常用的 4 种设置方式（优先级从高到低）

设置方式

代码/命令示例

适用场景

优先级

### 1. 算子级别单独设置

.keyBy(...).window(...).sum(0).setParallelism(16) 最常用！

某个算子是热点，想单独给他开多线程

★★★★★

### 2. 提交作业时用 -p 指定

flink run -p 128 wordcount.jar

快速调整体并行度，不想改代码

★★★★

### 3. 代码里全局设置

env.setParallelism(64)

开发测试时用，生产不推荐

★★★

### 4. flink-conf.yaml 默认

parallelism.default: 64

整个集群默认值，最后手段

★★

1.4
推荐做法

1
代码里只给热点算子单独 setParallelism（比如 window、join、sink）Java

2
提交作业时用 -p 统一控制整体并行度Bash

1.5
生产常见值

- Kafka Source：一般和分区数对齐（比如 24 分区 → 并行度 24）

- 窗口聚合：CPU 核心数的 2~4 倍（比如 32 核机器，开 64~128）

- JDBC/MySQL Sink：3~12（MySQL 扛不住太多）

- ClickHouse/Doris Sink：8~32（看库性能）

一句话总结： 并行度就是你给算子雇了多少个工人干活，生产中永远是“热点算子单独 setParallelism + 提交时 -p 整体控制”。

2
算子链（Operator Chain）

3
任务槽（Task Slots）

## 三、第三部分：DataStreamAPI

DataStream API 是Flink的核心层API。一个Flink程序，其实就是对 DataStream (数据流) 的各种转换。具体来说，代码基本上都由以下几部分构成：

1、执行环境（必须第一步）

最简单的方式，就是直接调用getExecutionEnvironment方法。它会根据当前运行的上下文直接得到正确的结果：如果程序是独立运行的，就返回一个本地执行环境；如果是创建了jar包，然后从命令行调用它并提交到集群执行，那么就返回集群的执行环境。也就是说，这个方法会根据当前运行的方式，自行决定该返回什么样的运行环境。

（如果生产使用的是云产品部署，这里代码里面就不要写死处理Checkpoint）

2、Source（从哪儿读数据）

Source 是整个 Flink 作业的入口，决定了数据从哪儿来、怎么来、是否带事件时间。 生产中 90% 的 Source 都是 Kafka，剩下 10% 是测试/文件/自定义。

### 1. Source 分类 & 使用频率

类型

使用场景

生产频率

推荐度

Kafka

生产实时数据入口

★★★★★

必学

文件（File）

批处理或小数据测试

★★

了解

集合（Collection）

本地测试、单元测试

★★★

常用

Socket

快速验证、nc 测试

★★

了解

DataGen

性能测试、压测

★★★

推荐

自定义 Source

特殊数据源（MySQL/Redis 等）

★★★

进阶

生产推荐：Kafka + WatermarkStrategy（事件时间 + 水印）。

### 2. Kafka Source (生产主力)

### 3. Source 层生产必需考虑的 3 件事

配置项

代码位置

为什么必须配

Checkpoint（Exactly-once）

env.enableCheckpointing(10000L, CheckpointingMode.EXACTLY_ONCE);

Kafka Source 才能实现端到端精确一次

水印 + 事件时间

WatermarkStrategy.forBoundedOutOfOrderness(...) + withTimestampAssigner

窗口才能基于事件时间正确计算，处理乱序

并行度对齐 Kafka 分区

Source 并行度 = Kafka 分区数

否则吞吐打折，某些分区消费慢

### 4. 一句话总结 Source 层

生产作业 99% 都是从 Kafka Source 开始，配好 Checkpoint + 水印 + 并行度对齐分区，就能稳稳吃下 10w+ QPS，剩下就是 Transform 和 Sink 的事了。

3、Transform （怎么算）

Transform 是 DataStream API 的核心，负责对数据流进行各种转换、过滤、分组、窗口、聚合等操作。 生产中 90% 的业务逻辑都在这里实现。

1
Transform 基本分类

类型

作用

常见操作

生产频率

逐条处理

一进一出（或过滤）

map、flatMap、filter

★★★★★

分组

按 key 分组

keyBy

★★★★★

窗口

把无限流切成有限段

window（Tumbling/Sliding/Session）

★★★★★

聚合

窗口内/全局聚合

sum、reduce、aggregate、process

★★★★★

时间 & 水印

处理乱序 + 事件时间

assignTimestampsAndWatermarks

★★★★★

高级

定时器、状态、侧输出、双流连接

process、connect、sideOutput

★★★★

2
常用的 Transform

2.1
map / flatMap / filter

2.2
keyBy（分组，必须在窗口前用，这一步操作也可以叫做开窗操作）

计算的结果不仅依赖当前数据，还跟之前的数据有关，相当于要把所有数据聚在一起进行汇总合并——这就是所谓的“聚合”（Aggregation），类似于MapReduce中的reduce操作。

在内部，是通过计算key的哈希值（hash code），对分区数进行取模运算来实现的。所以这里key如果是POJO的话，必须要重写hashCode()方法。

2.3
window（窗口，实时计算灵魂）

（这个后面统一讲，这个比较重要）

2.4
聚合（sum/reduce/aggregate）

3
物理分区算子（Physical Partitioning）

4
分流

所谓“分流”，就是将一条数据流拆分成完全独立的两条、甚至多条流。也就是基于一个DataStream，定义一些筛选条件，将符合条件的数据拣选出来放到对应的流里。

4.1
简单实现

其实根据条件筛选数据的需求，本身非常容易实现：只要针对同一条流多次独立调用.filter()方法进行筛选，就可以得到拆分之后的流了。

案例需求：读取一个整数数字流，将数据流划分为奇数流和偶数流。

这种实现非常简单，但代码显得有些冗余——我们的处理逻辑对拆分出的三条流其实是一样的，却重复写了三次。而且这段代码背后的含义，是将原始数据流stream复制三份，然后对每一份分别做筛选；这明显是不够高效的。我们自然想到，能不能不用复制流，直接用一个算子就把它们都拆分开呢？

4.2
侧输出流

分流 = 把一条主流里“不想要”或“特殊处理”的数据，单独拎出来走另一条支流。

最常见的两个场景：

1
窗口迟到数据（超出了 allowedLateness）

2
脏数据/异常数据（解析失败、字段为空等）

不分流的话，这些数据要么丢掉，要么污染主流结果；分流后可以单独打印、写死库、告警，完全不影响主流。

### 4.2.1
分流核心代码

### 4.2.2
两种常见分流方式

方式一：窗口自动分流迟到数据

方式二：ProcessFunction 手动分流

5
合流

5.1
union（简单河流操作）

最简单的合流操作，就是直接将多条流合在一起，叫作流的“联合”（union）。联合操作要求必须流中的数据类型必须相同，合并之后的新流会包括所有流中的元素，数据类型不变。

特点：

- 多条流合并成一条

- 元素类型必须一致

- 不保证顺序（只是把所有数据混在一起）

- 并行度取最大值

适用场景：

- 多个 Kafka topic 的同构数据

- 多个文件/多个分区数据合并统计

- 测试多个来源的相同格式数据

生产注意：union 不改变数据类型，也不提供额外处理逻辑，适合“纯合并”场景。

5.2
connect（类型可不同，最强大合流方式）

流的联合虽然简单，不过受限于数据类型不能改变，灵活性大打折扣，所以实际应用较少出现。除了联合（union），Flink还提供了另外一种方便的合流操作——连接（connect）。

特点：

- 两条流（只能两条）合并成 ConnectedStreams

- 类型可以不同（左流 A，右流 B）

- 能分别处理两条流（CoMap / CoFlatMap / CoProcess）

- 可以共享状态（广播状态、Keyed 状态共享）

代码示例（三种常见用法）：

### 5.2.1
用法1：CoMap / CoFlatMap（简单分别映射）

### 5.2.2
用法2：CoProcessFunction（最灵活，能访问状态、定时器）

### 5.2.3
广播状态（Broadcast Connect，常用于规则配置下发）

5.3
### 3. union vs connect 对比

对比项

union

connect

流数量

2条或多条

仅2条

类型要求

必须相同

可以不同

处理方式

自动合并，无自定义逻辑

需要 CoMap / CoProcess 手动处理

共享状态

不支持

支持（尤其是广播状态）

生产常见场景

同类型多源合并

双流 join、规则广播、订单支付匹配

一句话总结：

- 想简单合并同类型多流 → 用 union

- 想分别处理两条流、共享状态、做 join/匹配 → 用 connect + CoProcess

4、Sink（把结果写出去）

Flink作为数据处理框架，最终还是要把计算处理的结果写入外部存储，为外部应用提供支持。

Sink 是 Flink 作业的出口，负责将计算结果写入外部系统（如数据库、消息队列、文件、Redis 等）。 生产中 80% 的 Sink 都是 Kafka、JDBC、ClickHouse/Doris、Redis、ES 等，剩余是文件/打印（测试用）或自定义。

### 1. Sink 分类 & 使用频率

类型

典型场景

生产频率

推荐写法 & 注意点

Print / PrintToErr

测试、调试

★★★

.print()

 或 .printToErr()

，并行度自动设为1

文件（FileSink）

批式输出、归档

★★

FileSink.forRowFormat(...)

，支持分桶/滚动

Kafka

实时数仓下游、消息转发

★★★★★

FlinkKafkaProducer

，支持 exactly-once

JDBC (MySQL/PostgreSQL)

写入关系型数据库

★★★★

JdbcSink

，批量 + 防反压

Redis

缓存、实时特征

★★★

自定义 RichSinkFunction 或社区 connector

Elasticsearch

日志/搜索写入

★★★

ElasticsearchSink

，支持 bulk

ClickHouse / Doris

实时数仓主流

★★★★★

社区/官方 connector，批量 upsert

自定义 Sink

特殊系统（MQ、HTTP、HBase）

★★★★

继承 RichSinkFunction

### 2. 最常用的 6 种 Sink 代码示例（直接复制）

### 2.1 Print（测试神器）

### 2.2 文件 Sink（StreamingFileSink，新版推荐）

### 2.3 Kafka Sink（exactly-once 生产标配）

### 2.4 JDBC Sink（MySQL/PostgreSQL 批量写入）

### 2.5 Redis Sink（自定义示例，生产常用）

### 2.6 ClickHouse Sink（实时数仓主流，社区 connector 示例）

### 3. Sink 层生产（减少出问题的可能）

必做项

为什么必须做

操作方式

慢 Sink 前断链

防止下游慢导致全链反压

.disableChaining()

热点/慢 Sink 单独 slot 组

避免抢占上游 slot，隔离故障

.slotSharingGroup("slow-sink")

并行度合理设置

数据库/Kafka 扛不住太高并行

JDBC 3~8，Kafka 跟分区对齐

Exactly-once 配置（Kafka 等）

端到端精确一次

Semantic.EXACTLY_ONCE + Checkpoint

批量 + 超时优化

提升吞吐，减少网络开销

JdbcExecutionOptions batchSize / interval

### 4. 一句话总结 Sink 层

Sink 就是“把结果写出去”，生产核心是： Kafka（消息转发） → JDBC/ClickHouse（数仓落盘） → Redis/ES（缓存/搜索） → 自定义（特殊系统）。 永远记住：慢 Sink 必须 disableChaining + slotSharingGroup + 合理并行度，否则整个作业反压挂掉。

(当前咱们用到的Sink类型有：Kafka，Doris，InfluDB)

## 四、第四部分：Flink 中的时间和窗口

时间和窗口是 Flink 实时计算的灵魂：时间语义决定“什么时候算”，窗口决定“算哪些数据”。 Flink 是唯一原生支持事件时间 + 水印的开源引擎，这让它在处理乱序数据时远超 Spark Streaming（Spark 只支持处理时间，窗口不灵活）。

1、窗口

正确理解:在 Fink 中，窗口其实并不是一个“框”，应该把窗口理解成一个“桶”。在Fink中，窗口可以把流切割成有限大小的多个“存储桶””(bucket); 每个数据都会分发到对应的桶中，当到达窗口结束时间时，就对每个桶中收集的数据进行计算处理。

注意：Flink中窗口并不是静态准备好的，而是动态创建——当有落在这个窗口区间范围的数据达到时，才创建对应的窗口。

为什么要有窗口？
	无界流永远不会结束，无法直接做 sum、count、group by 等聚合操作。 窗口人为地给数据流划定边界，让聚合有意义。

### 1.1 窗口的分类

按照驱动类型分的话，可以分为时间类型窗口和计数类型窗口以及自定义类型窗口，这些类型的窗口里面，Flink会有具体的窗口实现。

a. 时间类型窗口（Time Window） —— 最常用

时间窗口以时间的维度去定义窗口的开始和结束，所以截取的就是某一个时间段的数据。到达结束时间的时候，窗口不在手机数据，触发计算输出结果，并将窗口销毁，所以可以理解为“定点发车”

### 1.1.1
滚动时间窗口（Tumbling  Time Window）

滚动窗口固定大小，均匀切片，窗口之间没有重叠，也不会有间隔，是首尾相连的状态，每个数据都会分配到一个窗口，而且只会属于一个窗口

滚动窗口最重要的参数就是，窗口的大小（Window Size ），但是这里的大小可以是时间也可以是个数，但是咱们这里说的是时间滚动窗口，这里的 Window Size 就是时间。

### 1.1.2
滑动时间窗口（Sliding Time Window）

滑动窗口的大小也是固定的，但是窗口之间并不是首位相连的，而是可以错开一定的位置。

定义窗口的参数有两个：除了窗口大小（Window Size）以外，还有滑动步长(Window Slide)，他其实就是代表窗口计算的频率，滑动步长就代表了计算频率，类似加速度。

滑动步长小于窗口的大小那么就会出现重合的情况，这时候数据就有可能出现分配到多个窗口的情况，而具体的个数，就是窗口大小和滑动步长的比值决定的，滑动窗口适合计算结果更新频率非常高的场景。

### 1.1.3
会话时间窗口（Session Time Window）

会话窗口，是基于"会话（session）"来对数据进行分组的。会话窗口只能基于事件维度来定义。

会话窗口中，最重要的参数就是会话的超时时间，也就是两个会话之间的最小距离，如果相邻的两个数据到来的时间间隔小于指定的大小，那么说明还在保持会话，那么属于同一个窗口，如果大于那么新来的数据就应该属于新的窗口，而前一个窗口就应该关闭了

会话窗口的长度不是固定的，开始事件和结束时间也不是固定的，各个分区之间的窗口也没有关联，会话窗口一定不会重合，而且留有至少为size的间隔，在一些保持会话的场景中可以使用这个窗口来进行数据处理

b. 计数类型窗口（Count Window） —— 条数驱动，生产中约 5~10%

### 1.1.1
滚动计数窗口（Tumbling Count Window）

和时间滚动窗口类似，不过这里取得不是时间维度，取得是数据个数，也就是 Window Size 取得是数据个数。

### 1.1.2
滑动计数窗口（Sliding Count Window）

### 1.2 窗口 API 概览

Flink 窗口 API 的核心调用链路非常清晰，大致可以分为以下几个层次：

最核心的几个组成部分：

层级

作用

是否必须

常见实现类 / 方法

keyBy

是否按键分区（决定窗口是全局还是分组）

视情况

keyBy(KeySelector)

window

指定窗口分配器（怎么切窗口）

必须

window(WindowAssigner)

trigger

自定义触发计算时机

选填

trigger(Trigger)

evictor

在触发前剔除部分元素

选填（极少用）

evictor(Evictor)

窗口函数

在窗口内怎么聚合/处理数据

必须

reduce / aggregate / process / fold（已废弃）

allowedLateness

允许迟到数据继续参与计算

选填

allowedLateness(Time)

sideOutputLateData

超过 allowedLateness 的数据分流

选填

sideOutputLateData(OutputTag)

最常见的调用模板

1. 按键分区（Keyed Windows） vs 非按键分区（Non-Keyed Windows）

这是窗口 API 最重要的一个分水岭，直接决定窗口的计算范围和状态存储方式。

(1) 按键分区窗口（Keyed Windows）—— 绝大多数生产场景使用

- 必须先 keyBy

- 每个 key 都有自己独立的窗口（每个 key 独立维护窗口状态）

- 状态按 key 分散存储（RocksDB 等状态后端按 key 存储）

- 支持所有窗口类型（时间窗口、计数窗口、全局窗口）

- 支持增量聚合（reduce/aggregate）效率最高

- 典型场景：按用户 ID、设备 ID、商品 ID 等分组统计

代码标志： 必须有 .keyBy(...)

(2) 非按键分区窗口（Non-Keyed Windows）—— 全局窗口，所有数据共享同一个窗口

- 不能 keyBy（或者 keyBy 后又 shuffle 成全局）

- 整个流只有一个窗口，所有数据都进入同一个窗口

- 状态是全局的（所有并行度共享一个状态，容易成为瓶颈）

- 只能使用全局窗口（GlobalWindow） + 自定义 Trigger

- 不支持增量聚合（reduce/aggregate），只能用 process 全量计算

- 典型场景：全局去重、全局 TopN、全局唯一计数、所有数据累加到一个阈值才触发

代码标志： 没有 .keyBy(...)，直接 .window(GlobalWindows.create())

生产对比总结：

维度

Keyed Windows（按键分区）

Non-Keyed Windows（非按键分区）

是否 keyBy

必须

禁止

窗口数量

每个 key 一个（成千上万）

全流只有一个

状态存储

按 key 分散（可扩展）

全局单一（容易单点瓶颈）

支持聚合函数

reduce / aggregate / process

只能 process（全量计算）

生产使用比例

95%+

极少（<5%）

典型场景

按用户/设备/商品统计

全局 TopN、全局唯一计数

2. 非按键分区窗口（Non-Keyed Windows）详细说明

非按键窗口几乎只使用 GlobalWindow，配合自定义 Trigger 来实现“全局聚合”。

典型使用场景（生产中遇到的几种）：

1
全局累计计数，直到达到某个阈值才输出一次

2
全局去重（所有数据中不重复的元素）

3
全局 TopN（所有数据中的前 N 名）

4
所有数据累加到一个计数器，直到收到“结束标记”才输出

核心代码模板

自定义 Trigger 示例（最常见：每 100 条触发一次）：

使用：

注意：

- 非按键窗口状态是全局的，数据量大时容易 OOM 或单点瓶颈。

- 只能用 process 全量计算，不能用 reduce/aggregate（因为没有 key 分散状态）。

- 除非业务明确要求“全局聚合”，否则强烈建议使用 Keyed Window。

### 3. 代码中窗口 API 的典型调用方式

模板 1：最常见 Keyed 时间窗口

模板 2：Keyed 滑动窗口

模板 3：Keyed 会话窗口

模板 4：Non-Keyed 全局窗口（少见）

模板 5：带侧输出的完整生产写法

总结一句话：

窗口 API 的本质就是：先决定是否 keyBy（分组还是全局），再选 window 分配器切桶，最后用聚合函数计算，生产中 95% 是 Keyed + Tumbling/Sliding + aggregate/process + 迟到处理。

### 4.关键点总结

问题

答案

可以先 keyBy 再加 window 再加 trigger 吗？

可以，而且是推荐的写法

trigger 是作用在哪个层面？

作用在每个 key 的每个窗口实例上（Keyed Window）

非 keyBy 的流可以用 trigger 吗？

可以，但必须用 GlobalWindows.create()，而且状态不按 key 分散

trigger 和 allowedLateness 冲突吗？

不冲突，trigger 决定“什么时候计算一次”，allowedLateness 决定“迟到数据还能参与多久”

自定义 trigger 最常见的目的

1. 条数阈值触发 2. 外部信号触发 3. 组合触发（时间+条数+信号） 4. 延迟保底触发

### 1.3 窗口分配器

定义窗口分配器（Window Assigners）是构建窗口算子的第一步，它的作用就是定义数据应该被“分配”到哪个窗口。所以可以说，窗口分配器其实就是在指定窗口的类型。

（在上述窗口类型已经做具体案列使用了）

### 1. 时间计数的窗口分配器（最常用，基于时间戳计数）

这些分配器以时间长度作为窗口的划分依据，是生产中 90%+ 的场景所用。

分配器类名

中文名称

计数方式说明

是否重叠

生产使用频率

典型代码写法

TumblingEventTimeWindows

滚动事件时间窗

固定时间长度，不重叠

否

★★★★★

.window(TumblingEventTimeWindows.of(Time.minutes(10)))

TumblingProcessingTimeWindows

滚动处理时间窗

固定时间长度，不重叠（基于机器时间）

否

★★

.window(TumblingProcessingTimeWindows.of(Time.seconds(30)))

SlidingEventTimeWindows

滑动事件时间窗

固定窗口大小 + 滑动步长，可重叠

是

★★★★

.window(SlidingEventTimeWindows.of(Time.hours(1), Time.minutes(5)))

SlidingProcessingTimeWindows

滑动处理时间窗

固定窗口大小 + 滑动步长，可重叠（机器时间）

是

★★

.window(SlidingProcessingTimeWindows.of(Time.minutes(10), Time.seconds(30)))

EventTimeSessionWindows

事件时间会话窗

根据不活跃间隔（gap）动态计数时间

—

★★★

.window(EventTimeSessionWindows.withGap(Time.minutes(30)))

ProcessingTimeSessionWindows

处理时间会话窗

根据不活跃间隔动态计数（机器时间）

—

★

.window(ProcessingTimeSessionWindows.withGap(Time.seconds(60)))

小结： 时间计数的分配器占主导地位，其中 TumblingEventTimeWindows 和 SlidingEventTimeWindows 是绝对主力。

### 2. 元素计数的窗口分配器（基于数据条数计数）

这些分配器不看时间，只看来了多少条数据。

分配器类名

中文名称

计数方式说明

是否重叠

生产使用频率

典型代码写法

CountWindow

计数窗口

滚动或滑动，基于元素个数

可选

★★

.countWindow(1000)

 或 .countWindow(5000, 1000)

GlobalWindows

全局窗口

不自动切分，所有元素进同一个窗口

—

★

.window(GlobalWindows.create())

说明：

- .countWindow(size) → 滚动计数窗口，每 size 条一个窗口

- .countWindow(size, slide) → 滑动计数窗口，每 slide 条触发一次，窗口包含最近 size 条

### 3. 其他 / 特殊计数的窗口分配器（高级 / 自定义）

分配器类名

中文名称

计数方式说明

生产使用频率

典型代码写法

MergingWindowAssigner

可合并窗口分配器

内部类，主要用于 Session 等可合并窗口

—

（一般不直接使用）

WindowAssigner（自定义）

自定义窗口分配器

完全自己决定一条数据属于哪些窗口

★（极少）

继承 WindowAssigner 并实现 assignWindows 方法

### 4.总结：按“计数”维度

- 时间计数（主流） → Tumbling / Sliding / Session（EventTime 优先）

- 元素计数（次主流） → countWindow（size）或 countWindow（size, slide）

- 无固定计数 → GlobalWindows + 自定义 Trigger

- 完全自由计数 → 自己写 WindowAssigner（狗都不用）

### 5.快速对照表（常见的 5 种）

你想实现的效果

推荐分配器

是否需要 keyBy

计数依据

每 10 分钟统计一次，不重叠

TumblingEventTimeWindows

必须

时间

最近 30 分钟，每 1 分钟更新一次

SlidingEventTimeWindows

必须

时间

用户不操作超过 20 分钟就算新会话

EventTimeSessionWindows

必须

时间间隔

每 10000 条数据聚合一次

countWindow(10000)

必须

元素条数

所有数据累积到 5000 条才输出一次

GlobalWindows + 自定义 Trigger

不需要

自定义计数

### 1.4 窗口函数

窗口函数是窗口切好之后，在窗口内部对数据进行真正计算的部分。 简单说：窗口分配器负责“切桶”，窗口函数负责“桶里怎么算”。

窗口函数（Window Functions）分为三大类，这也是生产中最常用、最清晰的划分方式：

1
增量聚合函数（Incremental Aggregation Functions） —— 代表：ReduceFunction / AggregateFunction

2
全窗口函数（Full Window Functions） —— 代表：ProcessWindowFunction / 全量 WindowFunction

3
增量聚合 + 全窗口函数的结合使用 —— 生产中最推荐的混合模式（先增量聚合、再全窗口处理）

1.1
增量聚合函数（ReduceFunction / AggregateFunction）

核心特点： 每来一条数据，就和之前的中间结果进行一次合并（增量计算），不需要把窗口内所有数据都保存在内存里，内存占用极低，计算效率最高。

适用场景：

- 简单聚合（sum、max、min、count）

- 可以用二元操作合并的指标（如字符串拼接、向量加法）

- 对内存敏感的大窗口场景

### 1.1.1
ReduceFunction（最简单增量聚合）

优点：

- 代码最短

- 状态只存一个元素（极省内存）

缺点：

- 输入和输出类型必须完全一致

- 无法输出复杂类型（如带窗口开始/结束时间的元组）

### 1.1.2
AggregateFunction（更灵活的增量聚合，生产主力）

优点：

- 累加器类型可以和输入/输出类型完全不同（非常灵活）

- 支持 merge（状态后端 checkpoint 时合并状态）

- 内存占用仍然很低

推荐指数：生产中最常用增量聚合方式。

1.2
全窗口函数（Full Window Functions）

核心特点： 窗口触发时，会把窗口内所有原始数据（Iterable）都传递给函数，属于全量计算。 可以访问窗口的元数据（开始/结束时间、当前 key 等）。

适用场景：

- 需要窗口开始/结束时间

- 需要对窗口内所有数据做排序、去重、TopN 等全量操作

- 需要注册定时器、侧输出等高级功能

代表：ProcessWindowFunction（最常用）

优点：

- 能拿到所有原始数据 + 窗口上下文（Context）

- 支持定时器、侧输出、状态操作

缺点：

- 全量计算 → 窗口内数据量大时内存压力大

- 不适合超大窗口

1.3
增量聚合 + 全窗口函数的结合使用（生产最推荐模式）

为什么结合使用？

- 先用 AggregateFunction / ReduceFunction 增量计算，把中间结果存下来（省内存、高效）

- 再用 ProcessWindowFunction 拿到最终聚合结果 + 窗口元数据，输出格式化结果或做额外逻辑

典型代码（先 aggregate 求和，再 process 加窗口信息）

这种写法的优势（生产中最常见的原因）：

- 内存占用低（aggregate 增量计算，只存中间结果）

- 功能强大（process 能拿到窗口元数据、key、定时器等）

- 计算效率高 + 输出信息丰富

一句话总结：

- 简单聚合 → 用 reduce

- 复杂累加指标 → 用 aggregate

- 需要窗口元数据 / 定时器 / 侧输出 → 用 process

- 大多数生产场景 → 先 aggregate 增量聚合，再 process 加元数据（兼顾性能和功能）

2、时间语义

想象你在统计“每个小时进店的顾客数量”，但顾客进店的时间和你的监控摄像头拍到他们的时间可能不一样：

- 事件时间（Event Time） 顾客真正踏进店门的那一刻（刷门禁卡的时间戳）。 → 这才是业务真正关心的“顾客进店时间”。

- 处理时间（Processing Time） 监控摄像头把画面传到你的电脑，你电脑收到画面并开始计数的时刻。 → 如果网络卡了 5 秒，这个顾客就被算到下一个小时去了，结果就不准。

- 摄取时间（Ingestion Time） 摄像头画面刚进入你的视频分析系统时，系统自动打上的时间戳。 → 比处理时间早一点，但仍然不是真正的进店时间。

一句话总结比喻： 业务要的是“顾客真正进店的时间”（事件时间），而不是“摄像头拍到他/系统处理他”的时间。

Flink 是目前唯一一个原生端到端支持事件时间 + 水印（Watermark）的开源流计算引擎，这也是它能做实时数仓、实时对账、实时特征计算的核心原因。

Flink 三种时间语义详细对比（带生产场景）

时间语义

定义

优点

缺点 / 风险

生产适用场景

Flink 版本支持度

Event Time

数据自带的时间戳（通常是业务发生时刻，如订单创建时间、日志产生时间）

结果准确、不受延迟/乱序影响、可重放

需要 Watermark 处理乱序，配置稍复杂

实时数仓、实时报表、订单统计、行为分析

全版本支持，最推荐

Processing Time

数据到达算子时的机器系统时间（wall-clock time）

最简单、延迟最低

乱序数据会导致结果错误、不可重放

低准确性要求的监控、告警、测试环境

全版本支持

Ingestion Time

数据进入 Source 时 Flink 自动分配的系统时间戳

比 Processing Time 更早分配时间戳

仍然无法处理 Source 之前的乱序

极少数从 Processing Time 过渡到 Event Time 的场景

全版本支持，但很少用

选择分布（2025 年主流公司实践）：

- 95%+ 的实时指标作业 → 必须用 Event Time

- 剩余 5% → Processing Time（内部监控、延迟敏感但准确性不重要的场景）

- Ingestion Time → 基本没人用（几乎可以忽略）

为什么 Spark Streaming 做不到事件时间（文档对比的关键点）

文档中对比表里有一行特别重要：

项目

Flink

Spark Streaming

时间语义

事件时间 + 处理时间

只支持处理时间

窗口

多、灵活

少、不灵活（必须是批次整数倍）

Spark Streaming 是微批（micro-batch）架构，本质上是把流切成小批次，每批次用批处理引擎跑，所以天然只能用“批次到达时间”（处理时间），无法真正基于事件时间做窗口聚合。

Flink 是真正的“连续流”引擎，支持在算子之间插入 Watermark 来推进事件时间，所以能完美处理乱序和延迟。

如何在代码中正确指定时间语义（最常用写法）

几乎所有生产代码都会在 Source 层或第一步 Transform 就指定事件时间 + 水印：

关键参数解释：

- forBoundedOutOfOrderness：允许的最大乱序时间（最常用）

- withTimestampAssigner：告诉 Flink 从哪取时间戳（支持 lambda 或 MapFunction）

- withIdleness：如果某个 key 分区很久没数据，水印不会卡死（生产必加）

常见踩坑总结（面试 / 生产常问）

坑

表现

解决办法

忘记 assignTimestampsAndWatermarks

窗口永远不触发、数据堆积

必须在 Source 后尽快加

Watermark 推进太慢

窗口延迟很高

检查最大乱序设置是否合理 + 加 withIdleness

用 Processing Time 做报表

结果每天都不一样、无法回溯

改成 Event Time + Watermark

Ingestion Time 误以为是 Event Time

Source 前有延迟时结果错

直接换成 Event Time

多 Source 不同时间字段

水印混乱

可以用侧输出或 union 后统一处理

总结一句话

Flink 的时间语义核心优势在于：它真正支持“事件时间 + Watermark”，让窗口计算基于业务发生时间，而不是数据到达时间，所以结果准确、可回溯，这是 Spark Streaming 做不到的，也是实时数仓、实时对账必须用 Flink 的根本原因。

如果你想继续深挖某个具体点，比如：

- Watermark 推进机制的详细图解

- 乱序数据 + 迟到数据的完整处理流程

- 多 Source 不同时间语义的混合写法

- 生产中 Watermark 参数怎么调优

3、水位线

在窗口的处理过程中，我们可以基于数据的时间戳，自定义一个“逻辑时钟”。这个时钟的时间不会自动流逝;它的时间进展，就是靠着新到数据的时间戳来推动的。

这样的好处在于，计算的过程可以完全不依赖处理时间(系统时间)，不论什么时候进行统计处理，得到的结果都是正确的。而一般实时流处理的场景中，事件时间可以基本与处理时间保持同步，只是略微有一点延迟，同时保证了窗口计算的正确性。

## 一、水位线的核心作用

- 没有 Watermark → 窗口永远不会触发（因为系统不知道什么时候数据到齐）

- 有 Watermark → 系统根据 Watermark 推进事件时间，窗口结束时间 ≤ 当前 Watermark 时，窗口关闭并输出结果

- Watermark 本质上是一种特殊的事件，它随着数据流向下游传播，携带一个时间戳，表示“进度”

比喻： 水位线就像河道里的浮标，浮标漂到哪里，就代表“上游到这个位置的水都流过来了，可以开始统计这段河道的水量了”。

## 二、水位线的生成方式

允许固定乱序（Bounded Out-of-Orderness） —— 最常用

  - 含义：假设数据最大延迟 5 秒，系统会等 5 秒后才推进 Watermark

  - Watermark 时间 = 当前最大事件时间 - 5 秒

单调递增时间（Monotonous Timestamps） —— 数据严格按时间顺序到达

  - 含义：每来一条数据，Watermark 立即推进到这条数据的时间戳

  - 适合 Kafka 等分区内有序的场景

生产额外推荐（防卡住）：

## 三、水位线如何推进

1
Source 读取数据 → 提取时间戳

2
每条数据到达时，更新“当前最大事件时间”

3
Watermark = 当前最大事件时间 - 最大乱序容忍度

4
Watermark 随着数据流向下游传播

5
当 Watermark > 某个窗口的结束时间 → 该窗口触发计算、输出结果、清空状态

示例流程（假设最大乱序 5 秒）：

- 来了一条 ts=10000 的数据 → 最大事件时间=10000

- Watermark = 10000 - 5000 = 5000

- 窗口 [0-10000) 不会触发（因为 Watermark 还没超过 10000）

- 继续来 ts=15000 的数据 → 最大事件时间=15000

- Watermark = 15000 - 5000 = 10000 → 窗口 [0-10000) 触发计算

## 四、水位线与迟到数据的关系

- 迟到数据：时间戳 < 当前 Watermark，但数据晚到

- 默认：迟到数据直接丢弃

- 处理方式（生产必须加）：

a
allowedLateness：允许迟到数据继续参与窗口计算（更新结果）

b
sideOutputLateData：超过 allowedLateness 的数据分流到侧输出（不丢，但不影响主流）

完整生产代码模板（文档风格）：

## 五、水位线常见问题 & 解决（生产经验总结）

问题

表现

原因 & 解决办法

窗口永远不触发

结果不输出

忘记 assignTimestampsAndWatermarks → 必须加

水位线推进太慢

延迟很高

最大乱序设置太大 → 调小容忍度（2~10 秒常见）

水位线卡死

某个分区没数据后全作业卡住

加 .withIdleness(Duration.ofSeconds(60))

迟到数据全丢

结果不准

加 allowedLateness + sideOutputLateData

多 Source 水位线不一致

水印混乱

统一在 union 或 connect 后重新 assign Watermark

一句话总结：

Watermark 是事件时间窗口能正确触发、处理乱序的关键。 没有它，事件时间就退化成处理时间；有了它，Flink 才能真正做到“基于业务发生时间计算，结果准确、可回溯”，这也是 Flink 比 Spark Streaming 更适合实时数仓的根本原因。

## 五、第五部分：处理函数

### 1. 基本处理函数（ProcessFunction）

作用：最底层的流处理函数，能对每条数据进行任意处理，支持：

- 访问事件时间 / 处理时间

- 注册 / 删除定时器

- 发出侧输出（Side Output）

- 修改输出类型

适用场景：需要定时器、侧输出、复杂过滤逻辑，但不需要窗口聚合的场景。

代码模板（最常用写法）：

关键点：

- processElement：每条数据调用一次

- onTimer：定时器触发时调用

- 支持 ctx.timerService() 注册事件时间 / 处理时间定时器

2. 按键分区处理函数（KeyedProcessFunction）

作用：在 keyBy 之后使用，每个 key 拥有独立的处理逻辑和状态。

与 ProcessFunction 的区别：

- KeyedProcessFunction 必须在 keyBy 之后使用

- 每个 key 独立维护状态和定时器

- 支持 keyed state（ValueState、ListState 等）

代码模板：

关键点：

- 状态和定时器都是 keyed 的（每个 key 独立）

- 常用于：每个用户/设备独立维护状态、超时检测、会话管理

3. 窗口处理函数（ProcessWindowFunction）

作用：窗口触发时，对窗口内所有数据进行全量处理，能访问窗口元数据（开始/结束时间、key 等）。

两种常见形式：

- 单独使用（全量计算）

- 与增量聚合结合（推荐，性能更好）

代码模板（单独使用，全量计算）：

关键点：

- Iterable<WaterSensor> elements：窗口内所有原始数据

- Context：可获取窗口开始/结束时间、当前 key、侧输出、定时器

### 4. 应用案例——Top N（窗口内 Top N）

场景：统计每分钟 vc 值最高的 Top 3 传感器

推荐写法：增量 + 全窗口结合（性能最佳）

生产真实 Top N 写法建议：

- 用 ListState 收集窗口内所有数据（小窗口）

- 或用 RocksDB + 定时器排序

- 或用 Flink SQL 的 OVER 窗口 + ROW_NUMBER()

### 5. 侧输出流（Side Output）

作用：将不符合条件的数据或迟到数据输出到另一条流，不影响主流计算。

代码模板（最常见两种用法）：

用法1：ProcessFunction 中手动侧输出（脏数据 / 异常）

用法2：窗口迟到数据自动侧输出

关键点：

- OutputTag 必须是匿名内部类或静态字段（泛型擦除问题）

- getSideOutput 获取侧输出流，可继续处理（print / sink / union 等）

- 生产中常用于：脏数据监控、迟到数据补救、异常告警

## 六、第六部分：状态管理

### 1. Flink 中的状态（State）

状态是 Flink 流计算的核心概念，指的是算子在处理无界流时需要记住的历史信息。

为什么需要状态？ 无界流是连续的，很多计算需要“记忆”之前的数据，例如：

- 累计求和（sum）

- 去重计数

- 窗口聚合

- 超时检测（上一次事件的时间）

- 每个用户的最后登录时间

Flink 状态的三大特性（官方核心）：

- Exactly-once 语义支持（Checkpoint + 状态恢复）

- 大规模（TB 级状态，RocksDB 后端）

- 按键分区（Keyed State）与算子级别（Operator State）两种类型

状态必须持久化：通过 Checkpoint 和 Savepoint 保存到外部存储（HDFS/S3 等），故障恢复时从最近 Checkpoint 恢复。

### 2. 按键分区状态（Keyed State）

Keyed State 是最常用的状态类型，必须在 keyBy 之后使用。 每个 key 拥有自己独立的状态实例，状态按 key 分散存储（天然支持大规模）。

支持的 5 种基本 Keyed State 类型（官方文档分类）：

类型

描述

代码示例（ValueState 最常用）

ValueState<T>

每个 key 存单个值

ValueState<Integer> lastVc;

ListState<T>

每个 key 存一个列表

ListState<WaterSensor> history;

MapState<UK, UV>

每个 key 存一个 Map

MapState<String, Integer> userCount;

ReducingState<T>

每个 key 存一个 reduce 后的值

ReducingState<Integer> sumVc;

AggregatingState<IN, ACC, OUT>

更灵活的聚合状态（类似 AggregateFunction）

AggregatingState<WaterSensor, Integer, Double>

代码示例（KeyedProcessFunction 中使用 ValueState + ListState）

生产注意：

- 状态名称（Descriptor）必须全局唯一

- 状态后端默认 RocksDB（支持大状态）

### 3. 算子状态（Operator State）

Operator State 不依赖 key，是算子级别的状态，每个并行子任务（subtask）拥有自己独立的一份状态。

典型场景：

- Source 算子记录已消费的 offset（Kafka Source 内部使用）

- Sink 算子记录已写出的记录数

- 自定义 Source / Sink 需要跨重启记住位置

- 全局计数器（所有数据共用一个计数）

支持的类型：

- ListState<T>（最常用）

- UnionListState<T>（checkpoint 时合并所有 subtask 的列表）

- BroadcastState<K, V>（广播状态，常用于规则下发）

代码示例（自定义 Source 使用 Operator State 记录 offset）

关键点：

- Operator State 不按 key 分区，每个 subtask 独立一份

- Checkpoint 时会把所有 subtask 的状态都保存

- UnionListState 用于需要合并所有 subtask 状态的场景（如广播）

### 4. 状态后端（State Backends）

状态后端决定状态如何存储、Checkpoint 如何实现。

Flink 提供了三种主要状态后端（文档中隐含提及 RocksDB 是生产推荐）：

状态后端类型

存储位置

Checkpoint 方式

适用场景

生产推荐度

MemoryStateBackend

JVM 堆内存

全量 Checkpoint 到 JobManager

小规模测试、状态很小

★（不推荐生产）

FsStateBackend

文件系统（HDFS/S3）

全量 / 增量（文件拷贝）

中等规模、状态不大

★★

RocksDBStateBackend

RocksDB（嵌入式 KV 存储）

增量 Checkpoint（非常高效）

大状态（GB~TB 级）、生产主流

★★★★★（生产默认）

生产配置推荐（flink-conf.yaml）：

RocksDB 优势（为什么生产必选）：

- 支持超大状态（TB 级）

- 增量 Checkpoint（只存变化部分，速度快）

- 内存占用可控（溢写到磁盘）

总结一句话：

- Keyed State：按 key 分散，生产 95% 用它（ValueState / ListState / MapState）

- Operator State：算子级别，全局或每个 subtask 一份（offset、广播规则）

- 状态后端：生产必须用 RocksDBStateBackend + 增量 Checkpoint

## 七、第七部分：容错机制

Flink 的容错机制是其 Exactly-Once 语义的核心支撑，主要通过检查点（Checkpoint）、状态一致性和端到端精确一次三个层面实现。

### 1. 检查点（Checkpoint）

检查点是 Flink 容错的核心机制，它周期性地为整个作业“拍照”，保存当前的状态快照和输入流的位置信息。

检查点的本质作用：

- 当作业发生故障（TaskManager 宕机、进程崩溃、网络中断等）时，从最近一次成功的 Checkpoint 恢复

- 恢复后，作业从 Checkpoint 保存的“输入位置 + 状态”继续计算，避免重复计算或丢失数据

检查点工作流程（生产中最常见理解）：

1
JobManager 定期发起 Checkpoint（默认每 60 秒，可配置）

2
向所有 Source 插入特殊的 Checkpoint Barrier（屏障）

3
Barrier 随着数据流向下游传播

4
每个算子收到 Barrier 后：

  - 暂停处理新数据

  - 将当前状态快照写入外部存储（RocksDB / HDFS 等）

  - 确认完成后向下游发送 Barrier

5
当所有 Sink 确认 Barrier 到达，Checkpoint 完成

关键配置（生产必调）：

Checkpoint 类型：

- 全量 Checkpoint：每次都完整保存状态（FsStateBackend 默认）

- 增量 Checkpoint：只保存变化部分（RocksDBStateBackend 支持，生产主流）

### 2. 状态一致性

Flink 提供了三种状态一致性语义（Consistency Guarantees），文档中隐含强调 Exactly-Once 是核心优势：

一致性级别

含义

是否有重复/丢失

Flink 支持方式

生产适用场景

At-most-once

最多处理一次，可能丢失数据

可能丢失

不做任何 Checkpoint

延迟极低、允许丢数据的监控

At-least-once

至少处理一次，可能重复

可能重复

Checkpoint + Barrier（默认）

大多数场景（Spark Streaming 级别）

Exactly-once

精确一次，既不丢也不重

无重复、无丢失

Checkpoint + Barrier + 两阶段提交

实时数仓、订单、对账、财务场景

Flink 如何实现状态一致性：

- Checkpoint Barrier 将流分成“已确认”和“未确认”两部分

- 状态快照只包含 Barrier 之前的计算结果

- Sink 只有在 Checkpoint 成功后才真正提交输出（两阶段提交）

一句话： Flink 通过 Checkpoint + Barrier 机制，确保状态和输入流位置一致，从而实现 Exactly-Once。

3. 端到端精确一次（End-To-End Exactly-Once）

端到端 Exactly-Once 是指：从 Source 到 Sink，整个链路数据既不丢失也不重复，即使发生故障恢复后也保持一致。

实现条件（必须同时满足）：

1
Source 支持：能重置消费位置（Kafka 支持 offset 重置）

2
Flink 内部：Checkpoint + Barrier + Exactly-Once 模式

3
Sink 支持：支持幂等写或两阶段提交（Two-Phase Commit）

常见 Sink 的 Exactly-Once 支持情况：

Sink 类型

是否支持 Exactly-Once

实现方式

生产注意事项

Kafka Sink

是

FlinkKafkaProducer + Semantic.EXACTLY_ONCE

必须开启事务 + Checkpoint

JDBC Sink

部分支持

幂等写（upsert）或两阶段提交

推荐用 upsert-kafka 或自定义两阶段

ClickHouse

部分支持

社区 connector + 幂等写入

需业务层去重或幂等设计

Redis

部分支持

自定义幂等写入

依赖业务主键幂等

File Sink

是

FileSink + exactly-once 模式

Flink 1.15+ 原生支持

生产代码示例（Kafka Sink Exactly-Once）：

端到端 Exactly-Once 完整链路： Source（Kafka offset） → Checkpoint 保存 offset + 状态 → Sink 预提交（两阶段） → Checkpoint 成功 → Sink 正式提交 → 故障恢复时回滚未提交的事务

一句话总结：

Flink 的容错机制通过 Checkpoint（拍照） 保证状态一致性，通过 Barrier + 两阶段提交 实现 端到端 Exactly-Once，这是 Flink 能做实时数仓、实时对账、财务级精确计算的核心原因。

​

1 人点赞

1
Acho02-09 11:4291
0
IP 属地广东
分享到：
正文
⌘ + Enter提交

回复
20557字
关于语雀使用帮助数据安全服务协议English
个人知识库
Flink
搜索
⌘ + J
首页
Apache Flink 快速入门
大纲
目标
## 一、第一部分：Flink 到底是什么？为什么要用它？
### 1.1 一句话定义
### 1.2 流处理的三个历史阶段
### 1.3 Exactly-once
## 二、第二部分：Flink 核心概念
Flink分层API
Flink 运行时架构
作业管理器（JobManager）
任务管理器（TaskManager）
Flink 核心概念
### 1. 并行度（Parallelism）
### 1.1. 并行度（Parallelism）一句话概述
### 1.2. 并行度到底决定什么？
### 1.3. 实际生产中最常用的 4 种设置方式（优先级从高到低）
### 1.4. 推荐做法
### 1.5. 生产常见值
### 2. 算子链（Operator Chain）
### 2.1. 什么时候能链？什么时候不能链？
### 2.2. 实际生产中怎么操作算子链
### 2.3. 生产中最常见的 3 个算子链黄金写法
### 3. 任务槽（Task Slots）
### 3.1. 任务槽（Task Slots）一句话概述
### 3.2. Task Slot 和 Parallelism 的终极区别（背下来）
### 3.3. 生产中最常用的 3 种操作（直接背代码和命令）
### 3.4. 生产黄金配置模板
### 3.5. 生产中最常见的 3 个 slot 配置经验值
### 3.6. 一句话总结
## 三、第三部分：DataStreamAPI
1、执行环境（必须第一步）
2、Source（从哪儿读数据）
### 1. Source 分类 & 使用频率
### 2. Kafka Source (生产主力)
### 3. Source 层生产必需考虑的 3 件事
### 4. 一句话总结 Source 层
3、Transform （怎么算）
### 1. Transform 基本分类
### 2. 常用的 Transform
### 2.1. map / flatMap / filter
### 2.1.1. 映射（map）
### 2.1.2. 过滤（filter）
### 2.1.3. 扁平映射（flatMap）
### 2.2. keyBy（分组，必须在窗口前用，这一步操作也可以叫做开窗操作）
### 2.3. window（窗口，实时计算灵魂）
### 2.4. 聚合（sum/reduce/aggregate）
### 2.4.1. 简单聚合
### 2.4.2. 归约聚合（reduce）
3. 物理分区算子（Physical Partitioning）
### 3.1. 随机分区（shuffle）
### 3.2. 轮询分区（Round-Robin）
### 3.3. 重缩放分区（rescale）
### 3.4. 广播（broadcast）
### 3.5. 全局分区（global）
### 3.6. 自定义分区（Custom）
### 4. 分流
### 4.1. 简单实现
### 4.2. 侧输出流
### 4.2.1. 分流核心代码
### 4.2.2. 两种常见分流方式
### 5. 合流
### 5.1. union（简单河流操作）
### 5.2. connect（类型可不同，最强大合流方式）
### 5.2.1. 用法1：CoMap / CoFlatMap（简单分别映射）
### 5.2.2. 用法2：CoProcessFunction（最灵活，能访问状态、定时器）
### 5.2.3. 广播状态（Broadcast Connect，常用于规则配置下发）
### 5.3. 3. union vs connect 对比
4、Sink（把结果写出去）
### 1. Sink 分类 & 使用频率
### 2. 最常用的 6 种 Sink 代码示例（直接复制）
### 2.1 Print（测试神器）
### 2.2 文件 Sink（StreamingFileSink，新版推荐）
### 2.3 Kafka Sink（exactly-once 生产标配）
### 2.4 JDBC Sink（MySQL/PostgreSQL 批量写入）
### 2.5 Redis Sink（自定义示例，生产常用）
### 2.6 ClickHouse Sink（实时数仓主流，社区 connector 示例）
### 3. Sink 层生产（减少出问题的可能）
### 4. 一句话总结 Sink 层
## 四、第四部分：Flink 中的时间和窗口
1、窗口
### 1.1 窗口的分类
a. 时间类型窗口（Time Window） —— 最常用
### 1.1.1. 滚动时间窗口（Tumbling Time Window）
### 1.1.2. 滑动时间窗口（Sliding Time Window）
### 1.1.3. 会话时间窗口（Session Time Window）
b. 计数类型窗口（Count Window） —— 条数驱动，生产中约 5~10%
### 1.1.1. 滚动计数窗口（Tumbling Count Window）
### 1.1.2. 滑动计数窗口（Sliding Count Window）
### 1.2 窗口 API 概览
1. 按键分区（Keyed Windows） vs 非按键分区（Non-Keyed Windows）
(1) 按键分区窗口（Keyed Windows）—— 绝大多数生产场景使用
(2) 非按键分区窗口（Non-Keyed Windows）—— 全局窗口，所有数据共享同一个窗口
2. 非按键分区窗口（Non-Keyed Windows）详细说明
### 3. 代码中窗口 API 的典型调用方式
### 4.关键点总结
### 1.3 窗口分配器
### 1. 时间计数的窗口分配器（最常用，基于时间戳计数）
### 2. 元素计数的窗口分配器（基于数据条数计数）
### 3. 其他 / 特殊计数的窗口分配器（高级 / 自定义）
### 4.总结：按“计数”维度
### 5.快速对照表（常见的 5 种）
### 1.4 窗口函数
1.1. 增量聚合函数（ReduceFunction / AggregateFunction）
### 1.1.1. ReduceFunction（最简单增量聚合）
### 1.1.2. AggregateFunction（更灵活的增量聚合，生产主力）
### 1.2. 全窗口函数（Full Window Functions）
### 1.3. 增量聚合 + 全窗口函数的结合使用（生产最推荐模式）
2、时间语义
3、水位线
## 一、水位线的核心作用
## 二、水位线的生成方式
## 三、水位线如何推进
## 四、水位线与迟到数据的关系
## 五、水位线常见问题 & 解决（生产经验总结）
## 五、第五部分：处理函数
### 1. 基本处理函数（ProcessFunction）
2. 按键分区处理函数（KeyedProcessFunction）
3. 窗口处理函数（ProcessWindowFunction）
### 4. 应用案例——Top N（窗口内 Top N）
### 5. 侧输出流（Side Output）
## 六、第六部分：状态管理
### 1. Flink 中的状态（State）
### 2. 按键分区状态（Keyed State）
### 3. 算子状态（Operator State）
### 4. 状态后端（State Backends）
## 七、第七部分：容错机制
### 1. 检查点（Checkpoint）
### 2. 状态一致性
3. 端到端精确一次（End-To-End Exactly-Once）

> 本文整理自个人知识库，[查看语雀原文](https://www.yuque.com/u22429903/dymeak/cyc94t4gkotr6gk1)。
