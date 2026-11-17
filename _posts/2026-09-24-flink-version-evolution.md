---
layout: post
title: "Flink_1.15-1.20_版本演进学习文档"
date: 2026-09-24
categories: [bigdata]
series: flink
series_order: 3
mermaid: false
math: false
toc: true
---
# Apache Flink 1.15 → 1.20 版本演进学习文档

> 面向使用者与开发者的版本变迁指南，重点梳理 **使用方式的变化** 与 **性能提升**。
> 资料来源：Apache Flink 官方发布公告、Release Notes、FLIP 设计文档。

## 目录

1. [总览：五年六版的演进主线](#一总览五年六版的演进主线)
2. [Flink 1.15（2022-05）](#二flink-1152022-05)
3. [Flink 1.16（2022-10）](#三flink-1162022-10)
4. [Flink 1.17（2023-03）](#四flink-1172023-03)
5. [Flink 1.18（2023-10）](#五flink-1182023-10)
6. [Flink 1.19（2024-03）](#六flink-1192024-03)
7. [Flink 1.20（2024-08，1.x 收官之作）](#七flink-1202024-081x-收官之作)
8. [横向总结：使用与性能的关键变化清单](#八横向总结使用与性能的关键变化清单)
9. [升级检查清单（1.15 → 1.20）](#九升级检查清单115--120)
10. [附录：版本时间线与参考链接](#十附录)

---

## 一、总览：五年六版的演进主线

| 版本 | 发布时间 | 一句话定位 | 最重要的关键词 |
|------|----------|------------|----------------|
| 1.15 | 2022-05 | Checkpoint 与弹性伸缩大年 | Changelog State Backend、Async Sink、Watermark 对齐 |
| 1.16 | 2022-10 | 批处理里程碑版本 | SQL Gateway、Hive 兼容 94%、动态分区裁剪、推测执行 |
| 1.17 | 2023-03 | 流式仓库（Streaming Warehouse） | TPC-DS 再提速 26%、删除/更新 API、预测执行增强 |
| 1.18 | 2023-10 | 为 2.0 铺路（弃用启动） | Runtime Filter、算子融合、SourceFunction 弃用、Pekko |
| 1.19 | 2024-03 | 可观测性 + 异步 SQL | config.yaml、AsyncScalarFunction、Web UI 内置 Profiler |
| 1.20 | 2024-08 | 1.x 最后一个版本 | Materialized Table、ForSt 状态后端（预览）、文件合并 Checkpoint |

三条贯穿主线：

1. **Checkpoint / 状态体系持续重构**：Changelog State Backend（1.15 MVP → 1.16 生产可用）→ 并行清理（1.19）→ ForSt 远程状态后端（1.20 预览，为 2.0 存算分离铺路）。
2. **批处理性能持续逼近 Spark**：动态分区裁剪（1.16，+30%）→ 连接重排/局部聚合（1.17，+26%）→ Runtime Filter + 算子融合（1.18，+14%），10T TPC-DS 累计相比 1.16 提升 54%。
3. **为 Flink 2.0 清理技术债**：1.18 起大规模弃用旧 API（DataSet、SourceFunction、Queryable State 等），1.19/1.20 继续推进配置体系与 Sink API 统一。

---

## 二、Flink 1.15（2022-05）

### 2.1 Checkpoint 与状态：通用增量 Checkpoint（FLIP-158）

这是 1.15 最重要的特性。引入 **Changelog State Backend**：

- 原理：状态更新**持续写入 changelog**（存储于 DFS），同时定期物化（materialization）到底层状态后端（RocksDB）。Checkpoint 时只需上传 changelog 增量部分。
- 收益：Checkpoint 间隔可从秒级降到亚秒级且**更加稳定可预测**；Failover 时数据回放量更小；事务型 Sink 的端到端延迟更低。
- 关键配置：

```yaml
state.backend.changelog.enabled: true
state.backend.changelog.storage: filesystem
execution.checkpointing.max-concurrent-checkpoints: 1
```

- 对齐/非对齐 Checkpoint 自动切换：`execution.checkpointing.aligned-checkpoint-timeout`，超过该时长自动从 Aligned 切换为 Unaligned。

### 2.2 Sink：Async Sink（FLIP-171）

- 新增 `AsyncSinkWriter` 抽象，简化"批量写出 + 限速重试"类 Sink 的开发（如 Elasticsearch 7/8 Sink 基于它构建）。
- 1.15 引入可配置的 `RateLimitingStrategy`（FLIP-242），自定义限速策略。

### 2.3 弹性伸缩与调度

- 跨源 Watermark 对齐（Watermark Alignment）：多源场景下对齐各分区的消费进度，防止快源把状态撑爆（`watermarkAlignment` 相关配置）。
- 自适应调度器增强：Reactive 模式下作业级指标可正常上报；增加异常历史记录；缩容速度提升。
- 引入**自适应批调度器（Adaptive Batch Scheduler）**雏形：根据节点数据量自动决定批作业各节点并行度。

### 2.4 其他使用面变化

- **增量 Savepoint**：RocksDB + 原生格式时，Savepoint 也可增量执行（此前 Savepoint 始终全量）。
- **作业结束时默认触发 Checkpoint**（`execution.checkpointing.checkpoints-after-tasks-finish`，解决无界窗口作业的收尾一致性问题）。
- SQL：批模式支持 Window TVF；新增 CAST、JSON 函数；SQL 拓扑保持稳定（升级版本后相同查询可恢复）。
- **去掉 Scala 版本绑定**：Java API 可与任意 Scala 版本（含 Scala 3）搭配使用（`flink-scala` 仅影响 Scala API 用户）。
- 移除 Mesos 支持；增加 OpenAPI 支持（REST API 文档化）。

### 2.5 小结（1.15）

对使用者最实际的三件事：**Changelog State Backend**（大状态作业 checkpoint 时长从秒级降到亚秒级）、**Watermark 对齐**（多 Kafka 分区/多源背压保护）、**Async Sink**（下游吞吐与一致性更好）。

---

## 三、Flink 1.16（2022-10）

官方定位：**Flink 批处理的里程碑版本**。230+ 贡献者、19 个 FLIP、900+ issue。

### 3.1 流处理：Changelog State Backend 生产可用

1.15 是 MVP，1.16 达到 production-ready：

- 支持状态迁移（升级/扩缩容时可转换）、本地恢复（local recovery）、文件缓存加速恢复；
- 支持按 checkpoint 开关切换、Web UI 暴露 changelog 指标与配置。

官方基准（value state，21 小时实验）：

| 百分位 | 端到端耗时（开启/关闭 changelog） | Checkpoint 数据量 |
|--------|----------------|-------------------|
| P90 | 664ms / 6s | 23.5MB / 4.52GB |

即 checkpoint 数据量降低约 **200 倍**，P90 耗时降低约 9 倍。

### 3.2 背压下 Checkpoint 改进（使用体验关键）

- **Overdraft Buffer（透支网络缓冲）**：为被背压的任务增加额外缓冲池，缓解 barrier 被卡在网络缓冲区导致的 checkpoint 超时。
- **非对齐 Checkpoint 级联启用**：此前 aligned-checkpoint-timeout 是每个 task 本地判断；1.16 起某个 task 检测到慢 checkpoint 后，会为上游算子也启用 unaligned checkpoint，背压下的 checkpoint 成功率明显提升。

### 3.3 SQL Gateway 与 Hive 兼容（批易用性）

- **SQL Gateway**（FLIP-91）：支持多租户、可插拔协议端点（REST API、HiveServer2 协议），可用 cURL/JDBC/Beeline 提交流、批甚至 OLAP 作业，可对接 DBeaver、Superset、DolphinScheduler 等生态。
- **HiveServer2 Endpoint**：SQL Gateway 可直接接受 Hive JDBC 连接，注册 Hive Catalog、切 Hive 方言、批模式执行，体验与 HiveServer2 一致。
- **Hive 语法兼容度达 94.1%**（Hive 2.3 全量 qtest），排除 ACID 查询达 97.3%，Hive SQL 平迁 Flink 成本大幅降低。
- HiveModule 支持原生聚合函数（SUM/COUNT/AVG/MIN/MAX 走 hash 聚合）。

### 3.4 批处理性能与稳定性

- **动态分区裁剪（FLIP-248）**：运行时根据维表数据裁剪事实表分区，10T TPC-DS 提升最高 **30%**。
- **Hybrid Shuffle Mode（FLIP-235，实验性）**：融合 blocking shuffle（资源需求小）与 pipelined shuffle（无需等上游完成）的优点，并支持不同 spill 策略。
- **Blocking Shuffle 进一步优化**：自适应网络缓冲分配、顺序 IO、结果分区复用，10T TPC-DS 整体再提 **7%**；新增 LZO/ZSTD 压缩算法。
- **Join Hint**：`/*+ BROADCAST(t) */`、`/*+ SHUFFLE_HASH(t) */` 等手动指定 join 策略，规避优化器选错计划。
- **自适应 Hash Join**：HashJoin 因数据倾斜失败时自动回退 Sort-Merge-Join，用户无感。
- **推测执行（Speculative Execution，FLIP-168/224）**：慢任务检测 + 黑名单机制，把慢 task 调度到其他节点重跑一份，先完成者胜出，消除长尾。（1.16 时 Sink 尚不支持。）

### 3.5 Lookup Join（维表关联）三连优化 —— 流处理高频痛点

| 能力 | 配置 | 解决问题 |
|------|------|----------|
| 缓存 | `lookup.cache`（全量/部分缓存） | 维表查询慢 |
| 异步模式 | `table.exec.async-lookup.output-mode` / lookup hint | 吞吐低（串行等待） |
| 可重试查询 | lookup hint 中 `retry-strategy` | 维表更新延迟导致关联不上 |

### 3.6 SQL 正确性：清理非确定性操作

长期存在的"某些 SQL 写法结果不正确/抛异常"问题（如非确定性函数用在需要确定性的场景），1.16 系统性解决了大部分。

### 3.7 小结（1.16）

大状态流作业：开 Changelog + Overdraft Buffer + 级联 unaligned checkpoint，容错体验质变。批作业：Hive 迁移 + 动态分区裁剪 + Join Hint，TPC-DS +30%。维表 join 三件套直接解决生产痛点。

---

## 四、Flink 1.17（2023-03）

主题：**迈向流式仓库（Streaming Warehouse）**，流批一体从"统一计算"走向"统一计算 + 统一存储"。

### 4.1 批处理性能再提升 26%（10T TPC-DS 相比 1.16）

- **动态规划连接重排**：join 数量多时自动选用更优的 DP 重排算法（搜索空间更大），且按 join 数自动选择算法，用户无感。（默认关闭，跑 TPC-DS 类查询建议开启 `table.optimizer.join-reorder-enabled`。）
- **动态本地哈希聚合**：根据数据分布动态决定是否做局部聚合，优化倾斜场景。
- 动态分区裁剪算法改进，覆盖更多查询；移除运行时部分不必要的虚函数调用。

### 4.2 推测执行与自适应批调度器增强

- 推测执行支持 Sink 算子：新增 `SupportsConcurrentExecutionAttempts` 接口，内置的 FileSink、PrintSink、HiveTableSink 等已支持（解决 1.16 Sink 不能推测执行的局限）。
- 慢任务检测考虑输入数据量，避免数据倾斜被误判为慢节点。
- 自适应批调度器对倾斜更健壮；**Hybrid Shuffle 成为批模式默认调度方式**。

### 4.3 流式仓库 API（FLIP-282）

- SQL 新增**行级 DELETE/UPDATE API**（批模式），外部存储（如 Flink Table Store / Paimon）可实现行级修改。
- `ALTER TABLE` 增强：加列/改列/删列、主键、水印，表结构维护更容易。

### 4.4 Hive 生态

- Hive 表 Sink 在**批模式也支持自动小文件合并**（此前仅流模式）。
- HiveModule 原生聚合函数（同 1.16 工作的延续）。

### 4.5 SQL Client / Gateway

- SQL Client 引入**网关模式**（gateway mode），可用 SQL 语句管理作业生命周期（SHOW JOBS、STOP JOB 等）。

### 4.6 PyFlink

- 支持 Python 3.10 与 Apple Silicon（M1/M2）；UDF 可用字符串声明数据类型；UDF 内可访问作业参数。

### 4.7 性能回归防护

- 引入每日性能基准监控（#flink-dev-benchmarks Slack 频道），性能回归进入"可主动发现"阶段，对使用者意味着后续小版本更稳。

### 4.8 小结（1.17）

对使用者：批查询自动变快（+26%）且更抗倾斜；流式仓库场景（Paimon/Table Store  CDC 入湖 + 行级更新）开始可用；SQL Gateway 进入实用期。

---

## 五、Flink 1.18（2023-10）

主题：**大量面向 2.0 的弃用 + 批性能两项杀手锏 + 运维体验跃升**。

### 5.1 ⚠️ 重大弃用（升级必读）

| 弃用项 | 替代方案 |
|--------|----------|
| `SourceFunction` | 统一 Source API（`Source`） |
| `SinkFunction`（未正式 deprecate 但 EOL） | SinkV2 |
| **DataSet API**（正式弃用） | DataStream API + `RuntimeExecutionMode.BATCH` |
| **Queryable State**（正式弃用） | 直接查状态后端 / 外部可查询存储 |

### 5.2 批处理性能：Runtime Filter + 算子融合（默认关闭，建议手动开启）

- **Runtime Filter（FLIP-? / FLINK-32486）**：对小表侧构建 Bloom Filter，广播到大表侧（probe 侧）过滤无关数据，减少扫描与 shuffle。部分 TPC-DS 查询提速最高 **3 倍**。
- **算子融合代码生成（Operator Fusion Codegen）**：把算子 DAG 融合成单个优化算子，消除虚函数调用、中间数据走 CPU 寄存器、降低指令缓存 miss。1.18 完成 Calc/HashAgg/HashJoin 三类算子。
- 两项合计：10T TPC-DS（分区表）相比 1.17 再提 **14%**，相比 1.16 累计 **+54%**。

```yaml
table.optimizer.runtime-filter.enabled: true
table.exec.operator-fusion-codegen.enabled: true
```

### 5.3 运行中动态调整并行度（FLIP-291，声明式资源管理外化）

- 通过 Web UI / REST API **在作业运行中修改任意算子的并行度**。
- 底层基于常规周期性 checkpoint（而非 savepoint）做 rescale，小状态作业几乎瞬时完成、停机极短。
- 配合 Web UI 的背压监控（红加蓝减）成为"找最优并行度"的官方推荐姿势。

### 5.4 SQL / Table API

- **算子级 State TTL（FLIP-292）**：Table API & SQL 可对**单个有状态算子**设置 TTL，例如双流 join 左右流设不同 TTL，此前只能全局 `table.exec.state.ttl`。
- **Watermark 对齐/空闲检测 SQL 化（FLIP-296）**：`OPTIONS` hint 里配置 watermark 对齐与 source 空闲超时。
- **存储过程（Stored Procedure）**：Catalog 可通过接口注册存储过程，用户用 `CALL` / `SHOW PROCEDURES` 调用（如 Paimon 的 compact 操作）。
- DDL 增强：`REPLACE TABLE AS SELECT`、`CREATE OR REPLACE TABLE AS SELECT`、批模式 `TRUNCATE TABLE`、`ALTER TABLE ADD/DROP PARTITION`、`SHOW PARTITIONS`；CTAS 支持原子性（取决于连接器）。
- SQL Gateway 获得 **JDBC Driver**（任何支持标准 JDBC 的工具可连 SQL Gateway）。

### 5.5 运行时与依赖

- **Akka → Pekko**：RPC 框架迁移到 Apache Pekko（Akka 许可证变更的应对），所有 Akka 依赖移除，裁剪集群依赖树。
- **支持 Java 17（beta）**编译与运行。
- **细粒度资源管理默认启用**（FLIP-31439）。
- **Watermark 对齐性能修复（FLINK-32420）**：5000 个 subtask 的场景下，JobManager 上 watermark 对齐耗时降低 **76 倍**，高并行度作业不再容易压垮 JM。
- Watermark 对齐在 1.18 起标记为**生产可用**。

### 5.6 已知坑（重要）

- 1.18.0/1.18.1 存在严重 bug（FLINK-34063）：**开启状态压缩（snappy 等）的作业升级到 1.18.0/1.18.1 可能丢数据**。使用状态压缩的用户应直接升到 1.18.2+。

### 5.7 小结（1.18）

对使用者：批 SQL 两个开关换取最多 3 倍加速；运行中调并行度成为常态运维手段；老 API（DataSet/SourceFunction）进入迁移倒计时。

---

## 六、Flink 1.19（2024-03）

主题：**可观测性大年 + 异步 SQL + 配置体系换代 + 又一批弃用**。

### 6.1 ⚠️ 配置体系换代（影响所有人）

- **标准 YAML 1.2 成为默认配置格式（FLIP-366）**：默认配置文件改为 `conf/config.yaml`；检测到旧 `flink-conf.yaml` 时优先使用旧文件（向后兼容），但 **2.0 起 flink-conf.yaml 彻底失效**。建议尽早迁移。
- `Configuration` 的 `setInteger/getInteger` 等类型化方法弃用，统一用 ConfigOption。
- AkkaOptions 更名 `RpcOptions`。
- 新增 `env.java.opts.administration`（管理员 JVM 参数，优先于用户参数）。

### 6.2 SQL 新能力（使用体验重点）

- **`STATE_TTL` hint（FLIP-373）**：join/group agg 上用 hint 直接写不同 TTL，无需改 compiled plan：

```sql
SELECT * FROM t1 JOIN t2 ON t1.id = t2.id
/*+ STATE_TTL('t1'='1h', 't2'='24h') */
```

- **MiniBatch Regular Join（FLINK-34219）**：常规 join 的 mini-batch 优化，减少中间结果、解决级联 join 的记录放大问题（`table.exec.mini-batch.allow-latency` 生态下又一利器）。
- **AsyncScalarFunction（FLIP-400）**：SQL/Table 终于有了异步 I/O UDF——此前只有 DataStream Java API 有 Async I/O，SQL 里调外部服务只能同步阻塞。仅流模式、当前仅保序输出。
- **Session Window TVF 流模式支持**；Window TVF 聚合支持 changelog 输入（CDC 直连窗口聚合）。
- 表源支持 `scan.parallelism` 设置并行度。
- 窗口 TVF 之外的 Group Window Aggregation 正式弃用（FLIP-392）。

### 6.3 可观测性（1.19 最大亮点之一）

- **Web UI 内置 Async-profiler（FLIP-375）**：在 JobManager/TaskManager 页面直接创建 profiling 实例（可指定时长与事件模式），完成后下载交互式 HTML 火焰图。不再需要自己挂 profiler。
- **Trace 体系（FLIP-384/385/386）**：checkpoint、作业初始化、恢复等产生 trace span，可导出到 OpenTelemetry；状态后端恢复时可附加自定义属性。配合指标系统，"checkpoint 为什么慢"第一次可以端到端追踪。

### 6.4 Checkpoint 与运行时

- **并行清理 checkpoint（FLINK-33090）**：过期 checkpoint 的 state 文件并行删除，大 checkpoint 清理时间从 ~10 分钟降到 **<1 分钟**（可关：`state.checkpoint.cleaner.parallel-mode: false`）。
- **ProcessingBacklog（FLINK-32514）**：source 标记 backlog 状态，运行时自动增大 checkpoint 间隔（追数据时高吞吐优先），追平后恢复低延迟间隔。
- **CLI 手动触发 checkpoint**：`./bin/flink checkpoint $JOB_ID [-full]`。
- `RestoreMode#LEGACY` 弃用（用 CLAIM / NO_CLAIM）。
- **指数退避重启策略成为默认（FLINK-33735）**：默认值优化、支持 max attempts、修复 region failover 的 attempts 计数不准。
- `System.out/err` 可重定向到日志（`taskmanager.system-out-redirect` 类配置），避免 TM 因 stdout 撑爆磁盘被杀。

### 6.5 API 迁移（又一批弃用）

- `RichFunction#open(Configuration)` → `open(OpenContext)`（旧的仍被回调，但建议迁移）。
- `RuntimeContext#getExecutionConfig` 弃用（FLIP-391，2.0 移除）。
- `Time` 类全面弃用，改用 `java.time.Duration`。
- SinkV2 接口大改（FLIP-372）：`TwoPhaseCommittingSink/StatefulSink/WithPre*Topology` 等弃用，改为 `CommitterInitContext/WriterInitContext/StatefulSinkWriter/CommittingSinkWriter`——**自己写 Sink 的开发者必须关注**。
- 批作业 source 动态并行度推断（FLIP-379）：FileSource 已实现，按数据量自动定并行度。

### 6.6 其他

- **支持 Java 21（beta）**；Python 3.7 移除、3.11 加入。
- 资源管理：`slotmanager.min-total-resource.*` 保证集群最小资源，加快作业启动。
- TM 位置在 REST API/Web UI 统一为 `endpoint`（host+port），`host` 字段弃用。

### 6.7 小结（1.19）

对使用者：**SQL 异步 UDF + STATE_TTL hint + MiniBatch join** 是流 SQL 三大实用新能力；**Web UI 一键火焰图 + checkpoint trace** 让性能排查成本骤降；**config.yaml** 迁移要排上日程。

---

## 七、Flink 1.20（2024-08，1.x 收官之作）

官方明确定位：**2.0 前最后一个小版本**，多项 2.0 高影响特性以 MVP 形态提前落地。

### 7.1 Materialized Table（FLIP-435）—— 数据管道开发范式变化

- 声明"查询 + 新鲜度（FRESHNESS）"，引擎自动推导 schema、**自动创建并调度刷新管道**，流/批由引擎按新鲜度决定，用户不再需要理解流批概念差异、不再需要直接维护 Flink 作业。
- 语法示例：

```sql
CREATE MATERIALIZED TABLE dwd_orders
PARTITIONED BY (ds)
FRESHNESS = INTERVAL '3' MINUTE
AS SELECT ... FROM orders o LEFT JOIN products FOR SYSTEM_TIME AS OF proctime() ...;

ALTER MATERIALIZED TABLE dwd_orders SUSPEND;
ALTER MATERIALIZED TABLE dwd_orders RESUME WITH ('sink.parallelism' = '10');
ALTER MATERIALIZED TABLE dwd_orders REFRESH PARTITION(ds='20231023');
```

- 生态：**Apache Paimon** 与内置 `test-filesystem` catalog 支持（Paimon 1.0+ 提供生产可用的物化表能力）。这是 Flink 向"流式湖仓 + 自动管道"迈出的标志性一步。

### 7.2 ForSt 状态后端 + 异步状态访问（MVP，2.0 架构预览）

- **ForStStateBackend**：基于 RocksDB 改造的 LSM-tree 状态存储，**sst 文件可放在 HDFS/S3 等远程文件系统**，本地盘只做缓存。状态大小不再受 TaskManager 本地盘限制，天然支持云原生部署。
- 配套 **State API V2（异步状态 API）**：`keyBy(...).enableAsyncState()` + `asyncGet/asyncUpdate`，非阻塞状态访问，隐藏远程 I/O 延迟。
- 1.20 状态：实验性。限制：仅增量快照，不支持 canonical savepoint / full snapshot / changelog / 文件合并 checkpoint；单 key/value 上限 2^31 字节。
- 意义：这是 2.0 **存算分离**（disaggregated state）架构的提前体验。后续 VLDB 论文数据（2.0 正式版对比 1.20）：checkpoint 耗时最高降 94%、恢复/扩缩容最高快 49 倍、成本节省最高 50%——1.20 引入的方向是 Flink 未来几年最大的架构红利。

### 7.3 统一文件合并 Checkpoint（FLIP-306，MVP）

- 把 checkpoint 产生的大量小文件合并写入大文件，减少文件创建/删除次数，缓解文件系统元数据压力（小文件风暴是 HDFS/S3 上大状态作业的著名痛点）。
- 配置：`execution.checkpointing.file-merging.enabled` 等。

### 7.4 运行时与运维

- **RocksDB 小 SST 文件合并修复（FLINK-26050）**：Flink 定期扫描本地 SST，小文件过多时主动触发 compaction，避免 processing-time 窗口等场景 SST 无限增长导致的稳定性事故（FLINK-34430 一类问题）。
- **批作业 JobManager 故障恢复（FLIP-383）**：此前 JM 挂了批作业只能从头重跑；1.20 起可恢复已完成 task 的状态，批作业长稳运行成为可能。
- `SOURCE_WATERMARK_MARKER` 等无关；重点是：统一 Sink API v2 转正（@Public），`SinkFunction` **正式弃用**。

### 7.5 API 与配置的最终清理（为 2.0 让路）

- **Unified Sink API v2 转正**（FLIP-197 毕业标准达成），旧 `SinkFunction` 弃用。
- `StreamExecutionEnvironment#setStateBackend()` / `getStateBackend()` 弃用 → 用 `state.backend.type` 配置。
- `TimeCharacteristic` 正式移除；大量 Kryo/类型注册相关旧方法标记弃用。
- 配置项全面梳理：一批配置项更名或弃用（详见 release notes 的 deprecated 列表）。

### 7.6 小结（1.20）

对使用者：Materialized Table + Paimon 是湖仓场景的"作业自动化"方案；ForSt + State API V2 是面向云原生大状态的未来投资；小文件合并与 RocksDB compaction 修复直接提升大状态作业稳定性。

---

## 八、横向总结：使用与性能的关键变化清单

### 8.1 性能提升全景（官方基准数据）

| 优化项 | 版本 | 量化收益 |
|--------|------|----------|
| Changelog State Backend | 1.15/1.16 | checkpoint 数据量降 ~200 倍（P90：4.52GB → 23.5MB），端到端耗时 6s → 664ms |
| 动态分区裁剪 | 1.16 | TPC-DS 10T 最高 +30% |
| Blocking shuffle 优化 | 1.16 | TPC-DS 10T +7% |
| 连接重排 + 动态局部聚合 | 1.17 | TPC-DS 10T +26%（对 1.16） |
| Runtime Filter | 1.18 | 部分 join 查询最高 3 倍 |
| 算子融合 codegen | 1.18 | TPC-DS 10T 对 1.17 再 +14%，对 1.16 累计 +54% |
| Watermark 对齐（高并行） | 1.18 | JM 对齐耗时降 76 倍（5000 subtask） |
| Checkpoint 并行清理 | 1.19 | 大 checkpoint 清理 10 分钟 → <1 分钟 |
| 批作业 JM 故障恢复 | 1.20 | 不再从头重跑（此前进度全损） |
| ForSt 存算分离（2.0 落地） | 1.20 预览 | checkpoint 耗时最高 -94%、恢复最快 49 倍、成本最高 -50% |

### 8.2 日常开发最需要上手的新能力（按推荐优先级）

1. **Changelog State Backend**（1.15+，1.16 生产可用）——大状态作业默认值得评估开启。
2. **算子级 / hint 级 State TTL**（1.18/1.19）——`table.exec.state.ttl` 全局一把梭成为过去式。
3. **SQL 异步 UDF（AsyncScalarFunction）**（1.19）——替代 SQL 里同步调外部系统的性能黑洞。
4. **MiniBatch Regular Join**（1.19）——级联 join 记录放大问题的官方解法。
5. **运行时调并行度 + 背压监控**（1.18）——弹性伸缩日常化。
6. **Web UI Profiler + Trace**（1.19）——性能排查标配。
7. **Materialized Table + Paimon**（1.20）——湖仓管道自动化。
8. **Runtime Filter / 算子融合两个开关**（1.18）——批 SQL 免费提速。
9. **Lookup Join 三件套**（1.16）——维表关联慢、旧、抖动的标准答案。
10. **Watermark 对齐 SQL 化**（1.18）——多源背压保护不再只能写 DataStream。

### 8.3 弃用与迁移速查表（1.18 → 1.20 累计）

| 类别 | 已弃用/移除 | 迁移目标 | 首次版本 |
|------|------------|----------|----------|
| 执行模型 | DataSet API | DataStream + BATCH 模式 | 1.18 |
| Source | SourceFunction | Source（FLIP-27） | 1.18 |
| Sink | SinkFunction | SinkV2（1.20 转正） | 1.20 |
| 状态 | Queryable State | 外部可查询存储 | 1.18 |
| 时间 | `Time` 类、`TimeCharacteristic` | `Duration`、事件时间默认 | 1.19/1.20 |
| 函数生命周期 | `open(Configuration)` | `open(OpenContext)` | 1.19 |
| 配置 | `flink-conf.yaml`（旧格式） | `config.yaml`（YAML 1.2） | 1.19 |
| 配置 | `setInteger/getInteger` 等 | ConfigOption | 1.19 |
| RPC | AkkaOptions | RpcOptions | 1.19 |
| 恢复模式 | RestoreMode#LEGACY | CLAIM / NO_CLAIM | 1.19 |
| 状态后端设置 | `env.setStateBackend()` | `state.backend.type` | 1.20 |
| 重启策略 | fixed-delay 默认 | exponential-delay 默认 | 1.19 |
| SQL | Group Window Aggregation | Window TVF | 1.19 弃用 |

### 8.4 兼容性注意事项（踩坑集锦）

- **1.18.0/1.18.1 状态压缩丢数据 bug（FLINK-34063）**：开了 snappy 等状态压缩的作业不要停在 1.18.0/1.18.1，直接 1.18.2+。
- **Savepoint 兼容性**：跨 1.x 升级通常可基于 savepoint 恢复，但状态结构变化（如 SQL 算子内部结构升级）可能导致无法恢复，升级前务必在预发验证 savepoint 恢复路径。
- **1.19 的 config.yaml 与 flink-conf.yaml 并存时优先旧文件**——迁移期配置"看起来没生效"先检查是否两个文件都在。
- **PyFlink 1.19 起不再支持 Python 3.7**；1.17 起支持 3.10，1.19 起支持 3.11。
- **Java 版本**：1.18 支持 Java 17（beta），1.19 支持 Java 21（beta）；生产建议 Java 11/17。
- **SinkV2 接口在 1.19 大改**：自建连接器维护者需按 FLIP-372 迁移，1.x 后期版本保持兼容但 2.0 移除旧接口。
- **ForSt（1.20）是实验特性**，不支持 changelog / 文件合并 checkpoint 等，生产大状态仍以 RocksDB + Changelog 为主。

---

## 九、升级检查清单（1.15 → 1.20）

**升级前：**

- [ ] 梳理作业是否使用：DataSet API、SourceFunction、SinkFunction、Queryable State、`Time` 类、`open(Configuration)`
- [ ] 确认状态压缩配置（避开 1.18.0/1.18.1）
- [ ] 预发环境验证 savepoint 跨版本恢复
- [ ] 连接器版本对照（Kafka/ES/JDBC 等连接器独立发版，注意与 Flink 版本的兼容矩阵）

**升级中：**

- [ ] 配置文件迁移到 `config.yaml`（可利用 YAML 语法校验工具）
- [ ] 替换弃用的配置项与方法（参考 8.3 表）
- [ ] 逐版本升级或跨版本升级均先小流量灰度

**升级后（建议开启的收益项）：**

- [ ] `state.backend.changelog.enabled: true`（大状态作业）
- [ ] `table.optimizer.runtime-filter.enabled: true`、`table.exec.operator-fusion-codegen.enabled: true`（批 SQL）
- [ ] `execution.checkpointing.file-merging.enabled: true`（小文件多的 HDFS/S3 场景，1.20）
- [ ] 背压高时确认 `execution.checkpointing.aligned-checkpoint-timeout` 已配置
- [ ] 用 Web UI Profiler 重新建立性能基线

**未来展望（2.0 已发布，2025-03）：** 存算分离状态管理（ForSt + 异步执行模型）正式落地、Materialized Table 生产化、Java 17 默认、Java 8 移除。1.20 是通往 2.0 的最后一块跳板——**现在升 1.20 就是在为 2.0 扫清弃用债务**。

---

## 十、附录

### 10.1 版本时间线

- Flink 1.15.0：2022-05-05
- Flink 1.16.0：2022-10-28
- Flink 1.17.0：2023-03-23
- Flink 1.18.0：2023-10-24（1.18.2 修复状态压缩 bug）
- Flink 1.19.0：2024-03-18
- Flink 1.20.0：2024-08-02（1.x 最后一个小版本；后续 1.20.x 为 bugfix 版本）
- Flink 2.0.0：2025-03-24

### 10.2 官方一手资料索引

- 各版本发布公告：`https://flink.apache.org/posts/`（1.16–1.20 均有专文）
- 各版本 Release Notes（升级必读）：`https://nightlies.apache.org/flink/flink-docs-release-{版本}/release-notes/`
- 关键 FLIP：
  - FLIP-158 Changelog State Backend
  - FLIP-171 Async Sink
  - FLIP-235 Hybrid Shuffle
  - FLIP-248 动态分区裁剪
  - FLIP-292 算子级 State TTL
  - FLIP-366 标准 YAML 配置
  - FLIP-372 Sink API 增强
  - FLIP-375 Web UI Profiler
  - FLIP-435 Materialized Table
  - FLIP-306 文件合并 Checkpoint
- 状态后端文档（含 ForSt 说明）：`https://nightlies.apache.org/flink/flink-docs-stable/docs/ops/state/state_backends/`
- 2.0 存算分离论文：VLDB 2025《Disaggregated State Management in Apache Flink 2.0》

---

*文档整理时间：2026-09。建议结合所用版本的具体 Release Notes 核对配置项名称。*
