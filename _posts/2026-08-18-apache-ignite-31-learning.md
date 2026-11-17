---
layout: post
title: "Apache Ignite 3.1 学习文档"
date: 2026-08-18
categories: [bigdata]
mermaid: false
math: false
toc: true
---
> 本文档基于 Apache Ignite 3.1.0 官方文档（https://ignite.apache.org/docs/ignite3/3.1.0）全站内容整理编写，覆盖入门、架构原理、开发、配置运维、SQL、API 参考与工具等全部章节，适合作为系统学习 Ignite 3.1 的中文教程。

## 目录

- 第一章 快速入门
- 第二章 理解 Ignite：架构与核心概念
- 第三章 开发指南：客户端与数据操作
- 第四章 安装、配置与集群运维
- 第五章 SQL 详解
- 第六章 API 参考导览（Java / .NET / C++ / JDBC / ODBC / Python）
- 第七章 工具：CLI 命令、REST API 与术语表
- 第八章 从旧版本迁移（Ignite 2.x → 3.x，3.0 → 3.1）

---

# 第一章 快速入门

## 1.1 Apache Ignite 3 是什么

Apache Ignite 3 是一个面向高性能计算的**分布式数据库**，核心特性包括：

- 高性能的数据存储与处理能力（内存级速度）；
- 完整的 SQL 支持，带 **ACID 事务**；
- 跨多节点的**水平扩展**能力；
- **强一致性**保证（基于 Raft 协议与 MVCC）；
- 内建分布式计算（Compute）与分析能力。

### 运行环境要求

| 要求 | 版本 |
| --- | --- |
| JDK | 11 及以上（运行节点）；Java 客户端开发建议 JDK 17+ |
| 操作系统 | Linux（Debian / Red Hat 系）、Windows 10/11 |
| 指令集架构 | x86 或 x64 |
| Docker 环境 | 建议 8GB+ 可用内存 |

## 1.2 快速开始（Quick Start）

本节演示：下载 Ignite → 启动节点 → 用 CLI 初始化集群 → 执行 SQL。

### 1.2.1 下载与安装

1. 从官网下载页面获取 Ignite 发行包（数据库本体）；
2. 同时下载 **Ignite CLI**（命令行工具），它是与 Ignite 交互的主要方式；
3. 解压：

```bash
unzip ignite3-3.1.0.zip
```

解压后得到 `ignite3-db-3.1.0`（数据库）和 `ignite3-cli-3.1.0`（CLI）两个目录。

### 1.2.2 启动节点

Ignite 是分布式数据库，运行在**节点（node）**之上；多个节点组成**集群（cluster）**，数据均匀分布在各节点上。

```bash
cd ignite3-db-3.1.0
bin/ignite3db          # Linux
# Windows 下需要 bash 环境：bash bin\ignite3db
```

### 1.2.3 启动 CLI 并连接节点

```bash
cd ignite3-cli-3.1.0
bin/ignite3
```

CLI 启动后会提示是否连接默认节点（`http://127.0.0.1:10300`），确认即可；也可用 `connect` 命令手动连接：

```
connect http://127.0.0.1:10300
```

### 1.2.4 初始化集群

节点启动后只是互相发现，必须执行**初始化（initialization）**集群才开始工作：

```
cluster init --name=sampleCluster
```

可选参数 `--metastorage-group=node1,node2,node3` 指定元存储组（metastorage group）节点，生产环境建议 3、5 或 7 个。

> **注意**：集群配置（cluster config）与节点配置（node config）是分离的，不可混用。初始化集群时要提供的是**集群**配置。

### 1.2.5 执行 SQL

进入 SQL REPL 模式（有语法提示与自动补全）：

```
sql
```

然后执行：

```sql
CREATE TABLE IF NOT EXISTS Person (id INT PRIMARY KEY, city VARCHAR, name VARCHAR, age INT, company VARCHAR);

INSERT INTO Person (id, city, name, age, company) VALUES (1, 'London', 'John Doe', 42, 'Apache');
INSERT INTO Person (id, city, name, age, company) VALUES (2, 'New York', 'Jane Doe', 36, 'Apache');

SELECT * FROM Person;
```

`exit;` 退出 SQL 模式，`exit` 退出 CLI。停止节点：在节点进程按 `Ctrl+C`。**集群初始化状态会保留**，节点重启后无需再次初始化。

### 1.2.6 多节点集群的两种启动方式

**方式一：Docker Compose 起 3 个节点**（适合本地实验）：

```bash
docker pull apacheignite/ignite:3.1.0
docker compose -f docker-compose.yml up -d
# 然后初始化：
cluster init --name=sampleCluster
```

**方式二：多台主机分别启动**：

1. 在每个节点的配置文件（默认 `/etc/ignite-config.conf`，zip 安装在 `{IGNITE_HOME}/etc/ignite-config.conf`）中配置 `nodeFinder`，列出所有节点地址：

```hocon
ignite {
  network {
    port: 3344
    nodeFinder.netClusterNodes = ["host1:3344", "host2:3344", "host3:3344"]
  }
}
```

2. 确保每个节点名唯一（在 `etc/vars.env` 中修改 `NODE_NAME`）。
3. 逐台启动节点。
4. 用 CLI 检查**物理拓扑**（physical topology，节点互相可见但尚未成集群）：

```
cluster topology physical
```

5. 全部可见后执行 `cluster init` 初始化；初始化后节点形成**逻辑拓扑**（logical topology），可用 `cluster topology logical` 验证。

运行中的节点也可动态查看/修改配置：

```
node config show ignite.network.nodeFinder
node config update ignite.network.nodeFinder.netClusterNodes=["localhost:3344","otherHost:3344"]
```

> 修改 nodeFinder 等配置后需要**重启节点**生效。

## 1.3 用 Docker 启动三节点集群

创建 `docker-compose.yml`：

```yaml
name: ignite3

x-ignite-def: &ignite-def
  image: apacheignite/ignite:3.1.0
  environment:
    JVM_MAX_MEM: "4g"
    JVM_MIN_MEM: "4g"
  configs:
    - source: node_config
      target: /opt/ignite/etc/ignite-config.conf
      mode: 0644

services:
  node1:
    <<: *ignite-def
    command: --node-name node1
    ports:
      - "10300:10300"
      - "10800:10800"
  node2:
    <<: *ignite-def
    command: --node-name node2
    ports:
      - "10301:10300"
      - "10801:10800"
  node3:
    <<: *ignite-def
    command: --node-name node3
    ports:
      - "10302:10300"
      - "10802:10800"

configs:
  node_config:
    content: |
      ignite {
        network {
          port: 3344
          nodeFinder.netClusterNodes = ["node1:3344", "node2:3344", "node3:3344"]
        }
      }
```

启动与验证：

```bash
docker compose up -d
docker compose ps
```

在 Docker 中启动 CLI 并初始化：

```bash
docker run --rm -it --network=host -e LANG=C.UTF-8 -e LC_ALL=C.UTF-8 apacheignite/ignite:3.1.0 cli
```

```
connect http://localhost:10300
cluster init --name=ignite3 --metastorage-group=node1,node2,node3
cluster status
```

`cluster status` 输出示例：

```
[name: ignite3, nodes: 3, status: active, cmgNodes: [node1, node2, node3], msNodes: [node1, node2, node3]]
```

**端口说明**：`10300+` 为 REST API（管理操作）端口，`10800+` 为客户端连接端口。

**停止与删除**：`docker compose stop` 停止但保留数据；`docker compose down` 删除容器（未配置持久化存储时数据会丢失）。

## 1.4 SQL 实战：Chinook 示例库

官方教程使用 Chinook（数字音乐商店）示例数据演示 Ignite SQL 能力。以下是核心知识点。

### 1.4.1 分布区（Distribution Zone）

建表前先创建分布区，控制数据在集群中的分布与副本：

```sql
CREATE ZONE IF NOT EXISTS Chinook WITH replicas=2, storage_profiles='default';
CREATE ZONE IF NOT EXISTS ChinookReplicated WITH replicas=3, partitions=25, storage_profiles='default';
```

- `Chinook`：2 副本，用于大部分表；
- `ChinookReplicated`：3 副本、25 分区，用于频繁访问的参考数据（小表）。

### 1.4.2 建表与数据共置（Colocation）

```sql
CREATE TABLE Artist (
    ArtistId INT NOT NULL,
    Name VARCHAR(120),
    PRIMARY KEY (ArtistId)
) ZONE Chinook;

CREATE TABLE Album (
    AlbumId INT NOT NULL,
    Title VARCHAR(160) NOT NULL,
    ArtistId INT NOT NULL,
    ReleaseYear INT,
    PRIMARY KEY (AlbumId, ArtistId)
) COLOCATE BY (ArtistId) ZONE Chinook;
```

**`COLOCATE BY` 是关键设计**：Album 按 `ArtistId` 共置，保证同一艺术家的专辑与其 Artist 记录落在相同节点，Artist–Album 的 JOIN 无需跨节点传输数据。

Chinook 库的共置链设计：

- Track 按 `AlbumId` 共置（多数查询按专辑关联曲目）；
- Invoice 按 `CustomerId` 共置；形成 `Customer → Invoice` 局部性链；
- InvoiceLine 按 `TrackId` 共置（面向 Track 的关联查询）；
- 参考表（Genre、MediaType）放入多副本 zone，每个节点都有副本，读取性能最好。

验证表结构：

```sql
SELECT * FROM system.tables WHERE schema = 'PUBLIC';
```

### 1.4.3 批量导入数据

退出 SQL REPL 后用 `--file` 执行 SQL 脚本：

```
sql --file=/opt/ignite/downloads/current_catalog.sql
```

### 1.4.4 常用查询示例

```sql
-- 基础查询
SELECT * FROM Artist;
SELECT * FROM Album WHERE ArtistId = 3;

-- 多表 JOIN
SELECT t.Name AS TrackName, a.Title AS AlbumTitle, ar.Name AS ArtistName
FROM Track t
JOIN Album a ON t.AlbumId = a.AlbumId
JOIN Artist ar ON a.ArtistId = ar.ArtistId
LIMIT 10;

-- DML
INSERT INTO Artist (ArtistId, Name) VALUES (276, 'New Discovery Band');
UPDATE Album SET ReleaseYear = 2024 WHERE AlbumId = 348;
DELETE FROM Album WHERE AlbumId = 348;
```

> 分布式更新会自动同步到所有副本：先更新主副本（primary），再传播到其他节点上的备份副本。

### 1.4.5 系统视图与索引

```sql
SELECT * FROM system.tables;
SELECT * FROM system.zones;
SELECT * FROM system.table_columns WHERE TABLE_NAME = 'TRACK';

-- 普通（排序）索引
CREATE INDEX idx_track_name ON Track (Name);
-- 复合索引
CREATE INDEX idx_album_artist ON Album (ArtistId, Title);
-- HASH 索引（适合等值查找）
CREATE INDEX idx_customer_email ON Customer USING HASH (Email);

SELECT * FROM system.indexes;
```

> 索引提升查询性能但增加写入维护成本，只为最常见的查询模式建索引。

### 1.4.6 共置对执行计划的影响（EXPLAIN）

用 `EXPLAIN PLAN FOR ...` 可以观察共置设计的效果：

- 共置不匹配时：计划中出现大量 `Exchange(distribution=[single])`（节点间数据搬移），估算行数可能爆炸（数百万级）；
- 共置匹配后：出现 `ColocatedHashAggregate`（先在各节点本地聚合再合并），Exchange 搬运的行数降到极小。

执行计划中常见算子：`TableScan`、`IndexScan`（带 `searchBounds` 精确边界）、`HashJoin`、`MergeJoin`、`ColocatedHashAggregate`、`Exchange`（节点间数据交换）。

**分布式 SQL 优化的三条核心原则**：

1. 相关表之间正确共置；
2. 索引与 JOIN 模式对齐；
3. 查询结构遵循共置模型。

### 1.4.7 SQL 最佳实践

- **Schema 设计**：频繁 JOIN 的表使用合适的共置键；选择能让数据均匀分布的主键；
- **查询优化**：为 `WHERE`/`JOIN`/`ORDER BY` 涉及的列建索引；用 `EXPLAIN` 分析查询；避免笛卡尔积；
- **事务管理**：事务尽量短；不要在用户交互期间保持事务打开；相关操作打包进一个事务保证原子性；
- **资源管理**：生产环境监控查询性能；超大表考虑分区策略；用合适的数据类型减少存储。

## 1.5 使用 Java API（键值与表操作）

### 1.5.1 项目配置

Maven 依赖：

```xml
<dependency>
    <groupId>org.apache.ignite</groupId>
    <artifactId>ignite-client</artifactId>
    <version>3.1.0</version>
</dependency>
```

### 1.5.2 连接集群

```java
String[] addresses = {"localhost:10800", "localhost:10801", "localhost:10802"};

try (IgniteClient client = IgniteClient.builder()
        .addresses(addresses)   // 多地址用于故障转移
        .build()) {
    System.out.println("Connected: " + client.connections());
}
```

### 1.5.3 SQL API

```java
client.sql().execute(null, "SELECT * FROM Person")
    .forEachRemaining(row -> System.out.println(row.stringValue("name")));
```

### 1.5.4 用 Catalog API 建表

```java
Table table = client.catalog().createTable(
    TableDefinition.builder("Person2")
        .ifNotExists()
        .columns(
            ColumnDefinition.column("ID", ColumnType.INT32),
            ColumnDefinition.column("NAME", ColumnType.VARCHAR))
        .primaryKey("ID")
        .build());
```

### 1.5.5 四种表视图（Table View）模式

**1. RecordView + Tuple（无模式记录视图）**

```java
RecordView<Tuple> recordView = table.recordView();
recordView.upsert(null, Tuple.create().set("id", 2).set("name", "Jane"));
```

**2. RecordView + POJO（类型安全记录视图）**

```java
RecordView<Person> pojoView = table.recordView(Person.class);
pojoView.upsert(null, new Person(3, "Jack"));
```

POJO 需要无参构造器用于反序列化。

**3. KeyValueView + Tuple（键值视图）**

```java
KeyValueView<Tuple, Tuple> kvView = table.keyValueView();
kvView.put(null, Tuple.create().set("id", 4), Tuple.create().set("name", "Jill"));
```

**4. KeyValueView + 原生类型**

```java
KeyValueView<Integer, String> kvPojoView = table.keyValueView(Integer.class, String.class);
kvPojoView.put(null, 5, "Joe");
```

> 所有视图方法的第一个参数是事务（`null` 表示隐式事务自动提交）。

## 1.6 嵌入式模式（Embedded Mode）

嵌入式模式指从 Java 代码中直接启动和管理 Ignite 节点，而不是用 CLI/独立进程。

> **与 Ignite 2 的重要区别**：Ignite 3 不再区分 client/server 节点，嵌入式启动的节点默认就存储数据。

### 依赖

```xml
<dependency>
    <groupId>org.apache.ignite</groupId>
    <artifactId>ignite-api</artifactId>
    <version>3.1.0</version>
</dependency>
<dependency>
    <groupId>org.apache.ignite</groupId>
    <artifactId>ignite-runner</artifactId>
    <version>3.1.0</version>
</dependency>
```

### 必需的 JVM 参数

```
--add-opens java.base/java.lang=ALL-UNNAMED
--add-opens java.base/java.lang.invoke=ALL-UNNAMED
--add-opens java.base/java.lang.reflect=ALL-UNNAMED
--add-opens java.base/java.io=ALL-UNNAMED
--add-opens java.base/java.nio=ALL-UNNAMED
--add-opens java.base/java.math=ALL-UNNAMED
--add-opens java.base/java.util=ALL-UNNAMED
--add-opens java.base/java.time=ALL-UNNAMED
--add-opens java.base/jdk.internal.misc=ALL-UNNAMED
--add-opens java.base/jdk.internal.access=ALL-UNNAMED
--add-opens java.base/sun.nio.ch=ALL-UNNAMED
-Dio.netty.tryReflectionSetAccessible=true
```

### 启动节点并初始化集群

```java
// 启动节点：名称、配置文件路径、工作目录
IgniteServer node = IgniteServer.start("node1", configFilePath, workDir);

// 初始化集群
InitParameters initParameters = InitParameters.builder()
    .metaStorageNodeNames("node1")
    .clusterName("cluster")
    .build();
node.initCluster(initParameters);

// 获取 Ignite API 实例（集群激活后返回）
Ignite ignite = node.api();

// 使用 SQL API
ignite.sql().execute(null, "CREATE TABLE IF NOT EXISTS Person (id INT PRIMARY KEY, name VARCHAR, age INT)");
ignite.sql().execute(null, "INSERT INTO Person (id, name, age) VALUES (1, 'Person Man', 501)");
try (ResultSet<SqlRow> rs = ignite.sql().execute(null, "SELECT id, name, age FROM Person")) {
    while (rs.hasNext()) {
        SqlRow row = rs.next();
        System.out.println(row.value(0) + ", " + row.value(1));
    }
}
```

> Session 虽然可关闭，但 DDL/DML 查询不保持游标，省略 `close()` 是安全的。

## 1.7 配置要点（Configuration Tips）

### 1.7.1 默认集群存储

查看默认分布区配置：`cluster config show ignite.zone`。默认配置示例（JSON）：

```json
{
  "ignite" : {
    "zone" : {
      "defaultDataStorage" : "aipersist",
      "defaultDistributionZone" : {
        "dataNodesAutoAdjustScaleDown" : 2147483647,
        "dataNodesAutoAdjustScaleUp" : 0,
        "dataStorage" : { "dataRegion" : "default", "name" : "aipersist" },
        "filter" : "$..*",
        "partitions" : 25,
        "replicas" : 1,
        "zoneId" : 0
      }
    }
  }
}
```

- 新分布区默认使用 `aipersist`（持久化存储引擎），可改为 `aimem`（内存）或 `rocksdb`；
- 配置文件支持 **JSON 或 HOCON** 两种格式；
- 查看/修改数据区（data region）：

```
cluster config show ignite.aipersist
cluster config update ignite.aipersist.defaultRegion.size=9999999
```

> 修改数据区参数后需要重启集群。

### 1.7.2 本地路径配置

`{IGNITE_HOME}/etc/vars.env` 可改：工作目录（数据存储位置）、日志目录、库加载目录、默认节点配置文件路径。

节点配置中还可设置：

- `ignite.system.cmgPath`：CMG（集群管理组）信息存储位置；
- `ignite.system.metastoragePath`：元存储信息位置；
- `ignite.system.partitionsBasePath`：数据分区存储位置；
- `ignite.system.partitionsLogPath`：RAFT 日志位置（与节点日志分开）。

### 1.7.3 堆内存配置

Ignite 把数据存在**堆外内存**（每个存储引擎单独保留），Java 堆只用于中间对象：集群/节点元数据（表版本链、事务锁信息等）、查询中间结果、Compute 作业数据。

默认堆为 **16GB**，通过 `etc/vars.env` 中的 `JVM_MAX_MEM` / `JVM_MIN_MEM`（等价于 `-Xmx`/`-Xms`）调整。堆小 → GC 快、占用少；堆大 → 可容纳更多对象但 GC 时间长。

### 1.7.4 日志配置

默认使用 `java.util.logging`（JUL），配置文件为 `etc/ignite.java.util.logging.properties`，日志输出到 `LOG_DIR` 指向的目录（默认 `{IGNITE_HOME}/log`）。可用 `java.util.logging.config.file` 指定自定义配置。

---

# 第二章 理解 Ignite：架构与核心概念

## 2.1 Apache Ignite 3 概览

Apache Ignite 3 是为高性能事务与分析型负载设计的分布式数据库：**内存级速度 + 磁盘持久化**，同一份数据同时支持 SQL 与键值访问，并提供完整 ACID 事务。

### 适用场景

| 负载类型 | Ignite 的价值 |
| --- | --- |
| 高吞吐 OLTP | 内存处理 + 持久化存储，支撑每秒数百万事务 |
| 实时分析 | 分布式 SQL 直接在分区数据上执行，无需 ETL |
| 缓存层 | 用可 SQL 查询、带事务的数据层替代外部缓存 |
| 微服务数据 | 强一致的共享分布式状态 |
| 事件处理 | 共置计算在数据所在处执行业务逻辑 |

### 核心特性一览

1. **统一数据模型**：表（Table）是唯一数据结构，SQL 和 Key-Value API 操作同一份 schema 和同一份数据。
2. **ACID 事务**：所有表默认支持事务，基于 MVCC；读写事务为可串行化（Serializable）隔离，只读事务为快照隔离且不加锁。
3. **分布区（Distribution Zones）**：控制分区数、副本因子、节点放置规则。
4. **分布式计算（Compute）**：作业可发往任意节点、与数据共置执行、或全集群广播。
5. **分布式 SQL**：基于 Apache Calcite 的 ANSI SQL 引擎，支持标准 JDBC。
6. **多存储引擎**：`aimem`（纯内存易失）、`aipersist`（内存+磁盘持久化，默认）、`rocksdb`（磁盘 LSM，实验性）。
7. **多语言客户端**：Java（11+）、.NET（6+）、C++17 原生瘦客户端（二进制协议），以及 JDBC、ODBC 标准连接。

### 与 Ignite 2 的架构差异（速览）

- **数据模型**：Ignite 2 用 cache + Binary Object；Ignite 3 统一为「表」，SQL 与 KV 共用同一 schema。
- **事务**：Ignite 2 需配置 cache 原子性模式；Ignite 3 全部默认事务化（MVCC），用 WAIT_DIE 算法免检测地防死锁。
- **分布配置**：Ignite 2 的 affinity 函数、backup、baseline topology 合并为 Ignite 3 的分布区；自定义 affinity 被确定性的 rendezvous hashing 取代。
- **客户端模型**：Ignite 2 胖客户端加入集群拓扑；Ignite 3 只用瘦客户端，嵌入式模式保留给特殊场景。
- **Compute API**：新增作业状态跟踪、优先级队列、节点离开自动故障转移；全部返回 `CompletableFuture`。
- **管理工具**：多个脚本合并为单一 `ignite3` CLI（交互式 + 自动补全）；配置从 XML 改为 HOCON。

## 2.2 系统架构

Ignite 3 采用模块化组件架构。**每个节点运行完全相同的服务集合**，软件层面没有 server/coordinator 之分——节点角色由 RAFT 组成员身份和 Placement Driver 的租约（lease）分配决定。

### 2.2.1 节点组件

**核心服务**

| 组件 | 职责 |
| --- | --- |
| Cluster Service | 网络通信、节点发现、消息路由 |
| Vault Manager | 节点级本地持久化存储 |
| Hybrid Clock | 为 MVCC 生成分布式混合时间戳 |
| Failure Manager | 节点故障检测与处理 |

**集群协调**

| 组件 | 职责 |
| --- | --- |
| CMG（Cluster Management Group） | 集群初始化、节点准入、逻辑拓扑 |
| MetaStorage | 分布式元数据（schema、配置、租约） |
| Placement Driver | 通过限时租约选择主副本 |

**数据管理**

| 组件 | 职责 |
| --- | --- |
| Table Manager | 表生命周期与分布式操作 |
| Replica Manager | 分区副本生命周期与请求路由 |
| Distribution Zone Manager | 数据放置策略与分区指派 |
| Index Manager | 索引创建、维护与异步构建 |

**查询与事务处理**

| 组件 | 职责 |
| --- | --- |
| SQL Query Processor | 查询解析、优化、分布式执行 |
| Transaction Manager | ACID 事务协调（2PC 协议） |
| Compute Component | 分布式作业执行 |

### 2.2.2 集群协调机制

- **CMG（集群管理组）**：专用 RAFT 组，负责集群级协调。初始化时确定成员，通常 3~5 个投票节点，其余节点作为 learner 参与。新节点经 CMG 验证后才能加入集群。
- **MetaStorage**：跨节点 RAFT 复制的分布式 KV 存储。所有集群元数据变更都经过它；节点维护本地副本，通过 RAFT 日志复制接收更新；watch 机制提供元数据变更的实时通知。
- **Placement Driver**：通过**限时租约**管理主副本选择。租约默认 5 秒，每半个周期自动续租；选择优先级：当前持有者 → RAFT leader → 稳定指派；租约持久化在 MetaStorage 中全集群可见。

### 2.2.3 RAFT 复制体系

| RAFT 组 | 用途 | 成员 |
| --- | --- | --- |
| CMG | 集群管理 | 3~5 个投票节点 |
| MetaStorage | 集群元数据 | 3~5 个投票节点 + learner |
| Placement Driver | 租约管理 | placement driver 节点 |
| Partition Groups | 数据复制 | 分区指派到的节点 |

**每个表分区都是一个独立 RAFT 组**，提供：写线性化（所有写经 leader）、日志复制到 follower、故障自动选主、基于 B+ 树的多版本状态机存储。

### 2.2.4 事务处理

- **事务协调**：Transaction Manager 用两阶段提交（2PC）协调分布式事务。
- **混合并发控制**：读写事务 = 基于锁 + MVCC（可串行化）；只读事务 = 基于时间戳的快照读（快照隔离，不加锁）。读写事务通过主副本获取锁；只读事务沿 MVCC 版本链按时间戳读取。

### 2.2.5 SQL 执行流水线

SQL 引擎基于 **Apache Calcite**，查询分 6 个阶段：

1. **解析**（Parsing）：SQL 文本 → 抽象语法树；
2. **校验**（Validation）：针对系统 catalog 做语义校验；
3. **优化**（Optimization）：基于成本的 Calcite 规则优化；
4. **物理规划**（Physical Planning）：转换为物理算子（IgniteRel）；
5. **映射**（Mapping）：计划片段（fragment）分配到集群节点；
6. **执行**（Execution）：分布式执行，节点间通过 Exchange（Sender/Receiver）交换数据。

### 2.2.6 组件启动顺序

```
VaultManager → ClusterService → CMG → MetaStorage → PlacementDriver →
ReplicaManager → TxManager → TableManager → SqlQueryProcessor
```

## 2.3 存储架构

Ignite 3 把**逻辑数据组织**与**物理存储实现**分离为分层架构：表定义数据模型 → 分布区控制分区与副本 → 存储 profile 配置引擎参数 → 存储引擎处理物理读写。

两个配置作用域：

- **集群级**：表 schema、分布区配置、存储 profile 名称在所有节点一致；
- **节点级**：profile 的具体实现（内存大小、文件路径）各节点独立配置——支持异构集群，不同硬件的节点可在同一分布区内使用各自合适的存储容量。

### 2.3.1 三大存储引擎

| 引擎 | 数据结构 | 持久化 | 适用场景 |
| --- | --- | --- | --- |
| `aimem` | B+ 树 | 无（易失） | 缓存、临时数据、最低延迟 |
| `aipersist` | B+ 树 | 基于 checkpoint | 通用、读写均衡（**默认**） |
| `rocksdb` | LSM 树 | 预写日志（WAL） | 写密集负载（实验性） |

**aimem（AIMemory）**：全部数据存于堆外内存 B+ 树，节点关闭数据即丢失。内存按段（segment）分配，从 `initSizeBytes` 起步按需扩展；每个数据区（region）最多 16 段，每次扩段最小 256MB；默认页 16KB（可配 1KB~16KB，须为 2 的幂），每页 32 字节头部/锁开销。

Profile 参数：

| 属性 | 默认 | 说明 |
| --- | --- | --- |
| `engine` | - | 必须为 `aimem` |
| `initSizeBytes` | 动态 | 初始内存，默认等于 `maxSizeBytes` |
| `maxSizeBytes` | 动态 | 最大内存，默认 `max(256MB, 物理内存的 20%)` |

引擎级参数：`ignite.storage.engines.aimem.pageSizeBytes`（默认 16384）。

**aipersist（AIPersist）**：数据存于磁盘分区文件（`table-{tableId}/part-{partitionId}.bin`），内存中有页缓存；checkpoint 进程定期把脏页刷盘以保证持久性与崩溃恢复；delta 文件记录两次 checkpoint 间的增量变化；B+ 树存储版本链（行数据）、排序索引、哈希索引与元数据。

Checkpoint 参数：

| 属性 | 默认 | 说明 |
| --- | --- | --- |
| `intervalMillis` | 180000 | checkpoint 间隔（3 分钟） |
| `intervalDeviationPercent` | 40 | 随机偏移，防止多节点同步 checkpoint |
| `checkpointThreads` | - | checkpoint 写线程数 |
| `compactionThreads` | - | 压缩线程数 |

**写节流（Write Throttling）**：checkpoint 缓冲达到 2/3（66.7%）时启动写节流——66% 以下正常；66%~90% 渐进节流、checkpoint 优先级提升；90% 以上重度节流、更新显著延迟。出现节流说明磁盘 I/O 慢或写入速率过高。

Profile 参数：`engine=aipersist`、`sizeBytes`（默认 `max(256MB, 20% RAM)`）。

**rocksdb**：基于 RocksDB 库的 LSM 树。写路径：内存写缓冲 → 刷为 L0 层 SST 文件；后台 compaction 合并 SST 并下沉到更低层；读路径：先查写缓冲，再从 L0 逐层向下。存在写放大与空间放大。适合写密集、追加型时序数据；读密集或要求稳定点查延迟的场景建议用 aipersist。

Profile 参数：`engine=rocksdb`、`sizeBytes`、`writeBufferSizeBytes`（默认 64MB）。引擎级参数：`flushDelayMillis`（RAFT 触发 flush 前的延迟，默认 100ms）。

### 2.3.2 引擎级配置

```
node config show ignite.storage.engines
node config update ignite.storage.engines.aipersist.checkpoint.intervalMillis=180000
```

> 修改引擎配置后需重启节点。

### 2.3.3 存储 Profile

Profile 把存储引擎绑定到具体参数；分布区按名称引用 profile，zone 内的表使用该 zone 指派的 profile。

- 每个节点自动创建名为 `default` 的 profile（aipersist 引擎）；
- 各节点必须有**同名、同引擎类型**的 profile，但容量可以不同；
- 新增 profile 后需重启节点。

```
node config show ignite.storage.profiles
node config update "ignite.storage.profiles:{cache_profile{engine:aimem,maxSizeBytes:536870912}}"
node config update "ignite.storage.profiles:{rocks_profile{engine:rocksdb,sizeBytes:1073741824}}"
```

### 2.3.4 在表上使用 Profile

```sql
CREATE ZONE financial_zone
    WITH PARTITIONS=25, REPLICAS=3,
    STORAGE_PROFILES = ['default', 'fast_cache'];

CREATE TABLE accounts (
    id INT PRIMARY KEY,
    balance DECIMAL(15,2)
) ZONE financial_zone STORAGE PROFILE 'default';

CREATE TABLE rate_limits (
    client_id INT PRIMARY KEY,
    requests INT
) ZONE financial_zone STORAGE PROFILE 'fast_cache';
```

> **约束**：表创建后不能更换 zone 或 storage profile；若某节点缺少所需 profile，则不会存储使用该 profile 的表的分区。

## 2.4 安全架构

安全默认**关闭**，需在集群配置中显式启用。两个独立层次：**认证**（验证客户端身份）与**传输加密**（SSL/TLS），可独立启用，但生产环境应同时启用（否则凭证明文传输）。

> **重要安全模型警告**：任何能访问集群节点的人都能对集群执行恶意操作，没有机制防护。因此所有 Ignite 服务端端口应只在受保护子网（DMZ）内开放；若必须暴露到 DMZ 之外，应使用受信 CA 签发的 SSL 证书控制访问。

### 2.4.1 认证

内置 basic 用户名/密码认证器（集群配置中定义用户）：

```json
{
  "ignite": {
    "security": {
      "enabled": true,
      "authentication": {
        "providers": [
          {
            "type": "basic",
            "name": "default",
            "users": [
              { "username": "admin", "password": "admin123" }
            ]
          }
        ]
      }
    }
  }
}
```

客户端连接：

```java
IgniteClient client = IgniteClient.builder()
    .addresses("localhost:10800")
    .authenticator(BasicAuthenticator.builder()
        .username("admin").password("admin123").build())
    .build();
```

认证管理器按顺序尝试多个 provider，直到一个成功。自定义认证需实现：服务端 `Authenticator`、扩展 `AuthenticationProviderConfigurationSchema` 的配置 schema、客户端 `IgniteClientAuthenticator`——可对接外部身份系统 / LDAP。

### 2.4.2 SSL/TLS 传输加密

三条独立通道分别配置：

| 配置路径 | 保护对象 |
| --- | --- |
| `ignite.network.ssl` | 节点间通信 |
| `ignite.clientConnector.ssl` | 客户端到节点 |
| `ignite.rest.ssl` | REST API |

SSL 属性：`enabled`（默认 false）、`clientAuth`（none/optional/require，用于 mTLS）、`ciphers`、`keyStore.{type,path,password}`、`trustStore.{type,path,password}`（默认 PKCS12）。

配置示例：

```
# 节点间 SSL
node config update ignite.network.ssl.enabled=true
node config update ignite.network.ssl.keyStore.path=/path/to/keystore.p12
node config update ignite.network.ssl.keyStore.password=keystorepass
node config update ignite.network.ssl.trustStore.path=/path/to/truststore.p12
node config update ignite.network.ssl.trustStore.password=truststorepass
# 双向 TLS
node config update ignite.network.ssl.clientAuth=require

# 客户端连接器 SSL
node config update ignite.clientConnector.ssl.enabled=true

# REST API SSL
node config update ignite.rest.ssl.enabled=true
node config update ignite.rest.ssl.port=10400
```

客户端 SSL：

```java
SslConfiguration sslConfig = SslConfiguration.builder()
    .enabled(true)
    .trustStorePath("/path/to/truststore.p12")
    .trustStorePassword("truststorepass")
    .build();
IgniteClient client = IgniteClient.builder()
    .addresses("localhost:10800").ssl(sslConfig)
    .authenticator(...).build();
```

### 2.4.3 安全事件与限制

- 认证事件：`USER_AUTHENTICATION_SUCCESS` / `USER_AUTHENTICATION_FAILURE`，可用于安全审计；
- **当前限制**：只提供认证，尚无授权（RBAC、权限）——所有认证用户对集群资源拥有同等访问权。

## 2.5 表与 Schema

表是 Ignite 3 的主数据结构，统一了 SQL 与 KV API。三层架构：**Catalog 层**（schema/表/索引/分布区的版本化元数据）→ **Schema 层**（列类型、二进制序列化、schema 演进）→ **存储层**（二进制行格式的分区存储）。

### 2.5.1 Schema 结构

每个表有 `SchemaDescriptor`，维护 4 种列序：行序（整行序列化顺序）、键序（主键列）、值序（非键列）、共置序（分区指派用列）。不在某序列中的列位置为 -1。

### 2.5.2 列类型

**定长类型**：BOOLEAN(1B)、INT8(1B)、INT16(2B)、INT32(4B)、INT64(8B)、FLOAT(4B)、DOUBLE(8B)、UUID(16B)、DATE(3B, `LocalDate`)。

**变长类型**：STRING（默认上限 65536）、BYTES、DECIMAL（精度/标度，`BigDecimal`）、TIME（`LocalTime`）、DATETIME（`LocalDateTime`）、TIMESTAMP（`Instant`，精度 0-9）。

> 时间类型**必须用 java.time API**，不支持 `java.util.Date`、`java.sql.Date/Time/Timestamp`。

### 2.5.3 主键

- 每个表必须有主键，支持 **Hash 主键**（哈希分区）与 **Sorted 主键**（范围分区 + 排序）；
- 约束：主键列全部非空、不允许重复列、列须存在于 schema、自动创建主键索引；
- 主键创建后不可修改；共置列必须是主键的一部分。

### 2.5.4 Schema 版本化

追加式（append-only）版本化：每次 `ALTER TABLE` 使 catalog 版本 +1 并产生新 schema 版本。版本不可变、连续编号、由 `ColumnMapper` 自动完成新旧版本数据迁移（读取旧版本行时自动升级，新列填默认值）、每个存储的行携带其 schema 版本。

### 2.5.5 二进制行格式

紧凑二进制格式，零拷贝读取：null 位图跟踪空列（不占存储）、定长列按偏移直接访问（无需反序列化）、变长列通过偏移表直达。

### 2.5.6 设计约束

1. 建表/改表/删表用 SQL DDL（Table API 只做读写，另有 Catalog API 可建表）；
2. schema 版本必须连续递增；
3. 主键不可变；
4. 变长类型有长度上限，DECIMAL 需显式精度/标度。

## 2.6 数据分区

分区把表数据切成固定大小的块分布到各节点，实现水平扩展：加节点 → 存储容量与查询吞吐同时增加。

- 表属于分布区；`PARTITIONS` 决定分区数，`REPLICAS` 决定每个分区的副本数；
- 分区编号 0 ~ PARTITIONS-1；同一分区的副本尽量放在不同节点；
- 同一 zone 内的表共享「分区→节点」映射（这是共置的基础）；
- Fair 分布算法把放置决策存入 metastorage，保证一致复用。

相关节点配置：`ignite.system.partitionsBasePath`（分区存储目录，默认 `work/partitions`）、`ignite.system.partitionsLogPath`（分区 RAFT 日志）、`ignite.system.metastoragePath`（元数据，**建议与分区放在不同设备**）。

### 2.6.1 分区数选择

推荐：**总可用核数的 2~4 倍 ÷ 副本数**。

- 3 节点 × 8 核、3 副本 → 16/24/32 个分区；
- 7 节点 × 16 核、3 副本 → 75/112/150 个分区。

不指定时自动计算：`dataNodesCount * coresOnNode * 2 / replicas`（至少 1 个）。不建议显著超出推荐值——维护分区及其分布会拖累集群性能。每表分区数上限 65000。

### 2.6.2 副本数、共识组与法定人数

- 每个分区的副本组成一个 RAFT 组，**法定人数（quorum）**才能写入；
- 默认 quorum：1 副本 → 1；2~4 副本 → 2；5+ 副本 → 3；
- 副本分两类：**投票成员（peer）**参与选举与写确认；**learner** 被动接收数据、不参与选举；
- 共识组大小 = `quorumSize * 2 - 1`（如 5 副本 quorum 2 → 3 peer + 2 learner）；
- 失去共识组多数 → 分区进入**只读状态**（仅显式只读事务可读）；
- **建议副本数恒为奇数且至少 3**：2 副本丢 1 个即失多数；3/5 副本可在网络分区场景保持可用；`REPLICAS ALL` 可在每个节点都放副本。

### 2.6.3 主副本与租约

- 每个分区一个**主副本（primary）**，处理全部读写事务；由 Placement Driver 通过租约指派；
- 租约机制：写线性化（只有 primary 处理读写事务）、周期性续租、租约过期自动故障转移（协商新 primary）；
- 只读事务可从备份副本读取；读写事务的读也走主副本；
- 新 primary 上任前先与复制组同步数据；复制组不可用（失多数）时需走灾难恢复流程，可能要手动 reset 分区。

### 2.6.4 版本存储与低水位线（Low Watermark）

MVCC 为每行保留多个版本组成的**版本链**：

- 读写事务看到最新版本；只读事务看到其开始时间戳时的版本；
- 早于**低水位线**的版本可被垃圾回收；低水位线默认 **600000ms（10 分钟）**；
- 已 DROP 的表在低水位线之前也不从磁盘删除，只读事务仍可按旧时间戳读它；
- 有活跃事务需要旧版本时，GC 推迟。

### 2.6.5 分布重置与再平衡

- 长期运行 + 拓扑多次变化后，共置可能变差、SQL 性能退化，可通过 CLI/REST API **重置（重算）分区分布**；重置通常触发分区再平衡，可能耗时较长；
- 集群规模变化时，Ignite 等待 zone 的 `AUTO SCALE UP/DOWN` 超时后按分布算法重新分配分区并传输数据，即**数据再平衡（rebalance）**。

## 2.7 分布与共置（Distribution & Colocation）

### 2.7.1 分布区定义

```sql
CREATE ZONE my_zone WITH
    PARTITIONS = 25,
    REPLICAS = 3,
    DATA_NODES_FILTER = '$[?(@.region == "us-east")]',
    DATA_NODES_AUTO_ADJUST_SCALE_UP = 300,
    DATA_NODES_AUTO_ADJUST_SCALE_DOWN = 600,
    STORAGE_PROFILES = 'default';
```

每个表恰好属于一个分布区，zone 控制：分区数、副本因子、可存数据的节点、扩缩容行为。

### 2.7.2 Rendezvous Hashing（约 rendezvous/最高随机权重哈希）

对每个分区：把每个节点 ID 与分区号组合计算哈希 → 按哈希值降序排序 → 指派给得分最高的节点。优点：**确定性放置**（同拓扑同映射）、**最小再平衡**（加减节点只影响必须移动的分区）、**稳定副本顺序**。

### 2.7.3 数据共置

共置保证不同表的相关数据落在同一分区，从而实现：无跨节点传输的 JOIN、无需分布式协调的事务、数据本地化的计算作业。

共置条件（缺一不可）：

1. 同一分布区；
2. 共置键列类型兼容；
3. **共置列必须是主键的一部分**；
4. 所有共置表对共置键使用同一哈希函数。

```sql
CREATE TABLE customers (
    customer_id INT PRIMARY KEY, name VARCHAR(100)
) WITH PRIMARY_ZONE = 'my_zone';

CREATE TABLE orders (
    order_id INT, customer_id INT, total DECIMAL(10,2),
    PRIMARY KEY (order_id, customer_id)
) COLOCATE BY (customer_id) WITH PRIMARY_ZONE = 'my_zone';
```

SQL 引擎会识别共置表并本地化 JOIN 执行；未共置时同一查询需要节点间搬数据。

### 2.7.4 再平衡控制与节点过滤

- `DATA_NODES_AUTO_ADJUST_SCALE_UP`：新节点加入数据分布前的延迟（默认立即）；
- `DATA_NODES_AUTO_ADJUST_SCALE_DOWN`：离线节点移出分布前的延迟（默认立即）；
- 设置延迟可避免滚动重启或短暂网络抖动引发不必要的再平衡；
- `DATA_NODES_FILTER` 用 JSONPath 表达式按节点属性筛选参与节点，如 `$[?(@.storage == "ssd" && @.memory >= 32)]`。

### 2.7.5 设计约束

1. 表指派到 zone 后不可更换；
2. zone 创建后分区数不可改；
3. 共置必须建表时指定；
4. 同一时刻一个分区只有一个租约持有者；
5. 失去共识多数 → 分区只读。

## 2.8 事务与 MVCC

### 2.8.1 事务模型

- 所有表默认事务化；读写事务**可串行化**；只读事务**快照隔离**且不加锁；
- **WAIT_DIE** 算法防死锁（无需死锁检测）；
- 分布式事务用**两阶段提交（2PC）**协调。

### 2.8.2 MVCC 版本链与写意向

每行的版本链按新到旧存储全部版本。每个版本含：提交时间戳（未提交为 null）、所属事务 ID、行数据（删除为 tombstone）、指向前版本的链接。

未提交的修改以**写意向（write intent）**形式挂在版本链头部，记录：写者事务 ID、提交分区 ID（用于查事务状态）、未提交的值。提交时写意向获得时间戳变成正式版本；中止时移除写意向。

**可见性规则**：只读事务固定在其创建时间戳读取一致快照；读写事务可见自己的写意向，遇他人写意向时等待或中止；只读事务总是跳过写意向读旧的已提交版本。

### 2.8.3 混合时间戳（Hybrid Timestamp）

混合逻辑时钟 = 物理时间 + 逻辑计数器：保证同节点事件的因果序、近似墙钟时间（支持基于时间的查询）、无协调本地生成。纪元起点为 2021-01-01；同毫秒内的多事件靠逻辑计数器排序。

> 节点间时钟必须同步（误差须在 `schemaSync.maxClockSkewMillis` 内）。

### 2.8.4 隔离级别与锁

读写事务用两阶段锁（2PL）：生长阶段先取锁再访问，收缩阶段在提交/中止时统一放锁。锁模式兼容矩阵：

| 锁 | IS | IX | S | SIX | X |
| --- | --- | --- | --- | --- | --- |
| IS | ✓ | ✓ | ✓ | ✓ | ✗ |
| IX | ✓ | ✓ | ✗ | ✗ | ✗ |
| S | ✓ | ✗ | ✓ | ✗ | ✗ |
| SIX | ✓ | ✗ | ✗ | ✗ | ✗ |
| X | ✗ | ✗ | ✗ | ✗ | ✗ |

IS=意向共享、IX=意向排他、S=共享（读锁）、SIX=共享意向排他、X=排他（写锁）。锁粒度为**按 key**。

只读事务完全绕开锁：创建时分配固定读时间戳、可从任意副本读、无需分区登记与协调、不能改数据。

```java
var tx = client.transactions().begin(new TransactionOptions().readOnly(true));
Account account = accounts.get(tx, 42);
tx.commit(); // 只读事务 commit 为 no-op
```

### 2.8.5 死锁预防（WAIT_DIE）

事务请求被他人持有的锁时：

| 请求方年龄 | 动作 |
| --- | --- |
| 比持有方老 | 等待锁释放 |
| 比持有方年轻 | 立即中止，以**相同时间戳**重试 |

年轻事务总是让位给年老事务，杜绝循环等待；同时间戳重试保证事务最终变老并成功。

### 2.8.6 事务生命周期与 2PC

状态：PENDING（活跃）→ FINISHING（协调者发起提交/回滚）→ COMMITTED / ABORTED；协调者丢失则进入 ABANDONED，超时后孤儿检测触发，根据提交分区状态解决写意向并释放锁。

发起事务的节点即**协调者**：跟踪登记的分区、指派一个提交分区存事务状态、编排 2PC、处理失败与超时。任一参与分区 prepare 失败，协调者中止所有参与者。

### 2.8.7 版本垃圾回收

低水位线默认 10 分钟：调大可支持更长的只读事务，但旧版本占用更多存储；活跃事务会阻止其可能需要的版本被回收。

### 2.8.8 事务选项与约束

```java
var rwTx = client.transactions().begin(new TransactionOptions().timeoutMillis(30000));
var roTx = client.transactions().begin(new TransactionOptions().readOnly(true));
```

| 选项 | 默认 | 说明 |
| --- | --- | --- |
| `readOnly` | false | 只读模式 |
| `timeoutMillis` | 0（无） | 超时自动回滚 |

约束：读写走主副本；锁按 key；**不支持 savepoint**（无部分回滚）；时钟须同步；每事务一个提交分区。

## 2.9 分布式计算与事件

### 2.9.1 计算模型

- 全异步，返回 `CompletableFuture<R>`；
- 作业放置：任意节点（AnyNode）/ 与数据共置（Colocated）/ 广播（Broadcast）；
- 优先级队列 + 可配线程池；节点离开自动故障转移；支持 MapReduce。

作业实现 `ComputeJob<T, R>`：

```java
public class AccountBalanceJob implements ComputeJob<Long, Double> {
    @Override
    public CompletableFuture<Double> executeAsync(JobExecutionContext context, Long accountId) {
        Table accounts = context.ignite().tables().table("accounts");
        Tuple record = accounts.recordView().get(null, Tuple.create().set("id", accountId));
        return CompletableFuture.completedFuture(record.doubleValue("balance"));
    }
}
```

`JobExecutionContext` 提供：`ignite()`（集群操作入口）、`isCancelled()`、`partition()`（共置作业的分区信息）。

### 2.9.2 三种作业目标

```java
// 任意节点
JobExecution<Double> exec = client.compute().submit(
    JobTarget.anyNode(client.clusterNodes()), descriptor, accountId);

// 共置：在 account 42 数据所在节点执行，消除网络传输
client.compute().submit(
    JobTarget.colocated("accounts", Tuple.create().set("id", 42L)), descriptor, 42L);

// 广播：所有节点执行，收全部结果
BroadcastExecution<String> bexec = client.compute().submitBroadcast(
    client.clusterNodes(), JobDescriptor.builder(NodeInfoJob.class).build(), null);
Map<ClusterNode, String> results = bexec.resultsAsync().join();
```

### 2.9.3 调度、优先级与故障转移

配置：`threadPoolSize`（默认 max(CPU 核数, 8)）、`queueMaxSize`（默认 Integer.MAX_VALUE）、`statesLifetimeMillis`（默认 60000，作业状态保留时长）。

```java
JobDescriptor.<String,String>builder(MyJob.class).priority(5).build(); // 数字越大优先级越低
execution.changePriorityAsync(1); // 执行中调整优先级
```

同优先级按 FIFO 执行。故障转移只在**节点离开**时触发（应用异常不重试，直接抛给调用方）；应用级重试用 `.maxRetries(3)`。

### 2.9.4 作业状态

QUEUED → EXECUTING → COMPLETED / FAILED / CANCELING → CANCELED。

```java
JobState state = execution.stateAsync().join();
state.status(); state.createTime(); state.startTime();
```

### 2.9.5 MapReduce 任务

实现 `MapReduceTask`（`splitAsync` 拆分 + `reduceAsync` 聚合），用 `client.compute().submitMapReduce(descriptor, input)` 提交。典型例子：词频统计——把文档按段落拆成多个 `MapReduceJob` 并行计数，再合并各作业结果。

### 2.9.6 事件系统

计算事件：COMPUTE_JOB_QUEUED / EXECUTING / COMPLETED / FAILED / CANCELING / CANCELED。

```java
EventListener<ComputeEventParameters> listener = params -> {
    System.out.println("Job " + params.jobId() + " status: " + params.status());
    return CompletableFuture.completedFuture(false); // false=继续监听，true=触发后自动移除
};
client.compute().listen(IgniteEventType.COMPUTE_JOB_COMPLETED, listener);
```

### 2.9.7 代码部署

作业类必须在执行节点可用，通过**部署单元（Deployment Unit）**分发：

```java
DeploymentUnit unit = DeploymentUnit.fromPath("my-jobs", "1.0.0", Path.of("my-jobs.jar"));
client.deployment().deployAsync(unit).join();

JobDescriptor<String, String> descriptor = JobDescriptor
    .<String, String>builder("com.example.MyJob")
    .deploymentUnits(List.of(new DeploymentUnit("my-jobs", "1.0.0")))
    .build();
```

### 2.9.8 设计约束

1. 作业应无状态（状态存表里）；
2. 参数与结果必须可序列化；
3. 自动故障转移只覆盖基础设施故障；
4. 监听器按注册顺序执行，跨节点无全局顺序；
5. 执行线程池有界，长作业会阻塞其他作业。

## 2.10 性能分析：EXPLAIN 详解

### 2.10.1 命令语法

```sql
EXPLAIN [PLAN | MAPPING] FOR <query>
```

- `PLAN`（默认）：以关系算子树展示计划，适合排查优化器相关性能问题；
- `MAPPING`：展示查询片段到集群节点的映射，适合排查数据共置相关问题。

执行计划揭示：表访问顺序与方式（全扫 vs 索引扫）、JOIN 算法、索引使用与搜索边界、每阶段行数估算、数据在节点间的分布。

### 2.10.2 算子速查

| 算子 | 类别 | 说明 |
| --- | --- | --- |
| TableScan | 访问 | 全表扫描 |
| IndexScan | 访问 | 索引扫描，可带 searchBounds |
| KeyValueGet / KeyValueModify | 访问 | 按主键直查 / DML 优化算子 |
| SystemViewScan / TableFunctionScan | 访问 | 系统视图 / 表函数扫描 |
| HashJoin / MergeJoin / NestedLoopJoin | 连接 | 哈希 / 排序归并 / 嵌套循环 |
| CorrelatedNestedLoopJoin | 连接 | 相关子查询嵌套循环（**慢**，应改写） |
| ColocatedHashAggregate / ColocatedSortAggregate | 聚合 | 共置单遍聚合 |
| MapHash/SortAggregate + ReduceHash/SortAggregate | 聚合 | 两阶段分布式聚合（Map 预聚合 + Reduce 合并） |
| Map/ReduceIntersect、Map/ReduceMinus、UnionAll | 集合 | INTERSECT / EXCEPT / UNION ALL |
| Project / Filter / Sort / Limit | 变换 | 投影 / 过滤 / 排序 / 限量 |
| Exchange / TrimExchange | 分布 | 节点间数据重分布 |
| Sender / Receiver | 分布 | 片段间数据收发 |
| SelectCount | 优化 | 非事务 `SELECT COUNT(*)` 专用优化 |

Exchange 的 distribution 策略：`single`（汇聚单节点）、`broadcast`（每节点全量副本）、`random`（随机分区）、`hash`（按指定列哈希分区）、`table`（按某表分布对齐）、`identity`（按列值分布）。

### 2.10.3 常见性能问题与对策

**1. 全表扫描代替索引扫描**：优化器选了 TableScan，可用 hint 强制索引：

```sql
SELECT /*+ FORCE_INDEX(t_col1_idx) */ id FROM t WHERE col1 = '1';
```

强制后计划出现 `IndexScan` 且带 `searchBounds: [ExactBounds ...]`。

**2. 选中次优索引**：有 `t_col1_col2_idx` 却用了 `t_col1_idx`，同样用 `FORCE_INDEX` 纠正。

**3. 意外 Sort**：索引排序方向（如 DESC）与查询所需（ASC）不一致时优化器会插入 Sort；建一个排序方向匹配的索引即可消除。

**4. 相关子查询（CorrelatedNestedLoopJoin）**：性能差、CPU 高，应改写为普通 JOIN：

```sql
-- 慢：SELECT emp.name, (SELECT dept.name FROM dept WHERE emp.dept_id=dept.id) FROM emp ...
-- 快：
SELECT emp.name, dept.name FROM emp, dept WHERE emp.salary > 1000 AND emp.dept_id = dept.id;
```

**5. 冗余排序**：`WHERE salary=1 ORDER BY name` 对 `(salary, name)` 索引仍需 Sort；改写为 `ORDER BY salary, name`（等值列在前不改变语义）即可直接走 IndexScan。

**6. COUNT 优化失效**：显式事务中 `SELECT COUNT(*)` 可能退化为 MapSortAggregate+TableScan；非事务场景才有 SelectCount 优化。

**7. 索引扫描缺 searchBounds**：谓词类型不匹配（如 `DECIMAL(5,3)` 列与整数字面量比较产生 CAST）时，IndexScan 只剩 predicate 没有 searchBounds——退化为全索引扫描。对策：`NO_INDEX` hint 禁用该索引，或**显式类型对齐**：

```sql
SELECT id FROM t WHERE col1 = 43::DECIMAL(5, 3);  -- 出现 ExactBounds
-- 表达式一侧做转换：col1 = SUBSTR(...)::int 而不是 col1::varchar = ...
```

**8. 共置使用**：两张表都未共置 → 计划出现两个 Exchange（全量搬到单节点再 JOIN）；一方共置 → 只剩一个 Exchange（只搬非共置方）；双方按 JOIN 键共置 → **零 Exchange**，完全本地 JOIN。

### 2.10.4 EXPLAIN MAPPING 输出解读

MAPPING 输出按片段（Fragment）组织：

- `Fragment#0 root`：根片段，产生最终结果（用户游标所在）；
- `distribution`：该片段的映射策略；
- `executionNodes`：执行该片段的节点列表；
- `partitions`：从哪些节点读哪些表的哪些分区；
- `tree`：该片段对应的关系子树，片段间用 Sender/Receiver 连接。

---

---

# 第三章　开发指南：客户端与数据操作

本章对应官方文档 Develop 部分，覆盖瘦客户端（Java/.NET/C++）、Table API、事务、数据流、分布式计算、代码部署、序列化、事件，以及 JDBC/ODBC/Python 连接与 Spring 集成。

## 3.1 瘦客户端概述

Ignite 3 的客户端（瘦客户端，thin client）通过**客户端连接器（client connector）**与集群通信，默认端口 **10800**。客户端的特点：

- 通过标准 socket 连接集群，**不加入集群拓扑**，不持有数据，也不会被选为计算任务的执行节点；
- 连接任意一个节点即可，该节点负责查询分发与结果聚合；
- 客户端具备分区感知（partition awareness）能力，可将键值操作直接路由到持有对应分区的节点；
- 支持自动重连、心跳保活、请求重试与负载均衡；
- 支持基础认证（basic authentication）与 SSL/TLS 加密。

官方提供三种语言的瘦客户端：**Java、.NET（C#）、C++**；另有 JDBC 驱动、ODBC 驱动与 Python DB API 驱动。

## 3.2 Java 客户端

### 3.2.1 引入依赖与建立连接

Maven 依赖：

```xml
<dependency>
    <groupId>org.apache.ignite</groupId>
    <artifactId>ignite-client</artifactId>
    <version>3.1.0</version>
</dependency>
```

建立连接：

```java
try (IgniteClient client = IgniteClient.builder()
        .addresses("127.0.0.1:10800")
        .build()) {
    // 使用 client.sql()、client.tables()、client.transactions()、client.compute() 等 API
}
```

`IgniteClient` 实现了 `AutoCloseable`，推荐使用 try-with-resources 管理生命周期。可以传入多个地址，客户端会随机选择并自动故障转移。

### 3.2.2 认证

集群开启基础认证后，通过 `BasicAuthenticator` 提供凭据：

```java
IgniteClient client = IgniteClient.builder()
    .addresses("127.0.0.1:10800")
    .authenticator(BasicAuthenticator.builder()
        .username("ignite")
        .password("ignite")
        .build())
    .build();
```

### 3.2.3 主要配置参数

| 参数 | 说明 | 默认值 |
| --- | --- | --- |
| `addresses` | 集群节点地址列表（host:port） | 无（必填） |
| `connectTimeout` | 建立连接的超时（毫秒） | 5000 |
| `heartbeatInterval` | 心跳消息间隔（毫秒） | 30000 |
| `heartbeatTimeout` | 心跳超时，超时判定连接失效（毫秒） | 5000 |
| `operationTimeout` | 单次操作超时（毫秒），0 表示不限 | 0 |
| `backgroundReconnectInterval` | 后台重连间隔（毫秒） | 30000 |
| `retryPolicy` | 重试策略 | 读操作默认重试 16 次，写操作默认不重试 |
| `metricsEnabled` | 是否启用客户端指标 | false |
| `loggerFactory` | 日志工厂 | `System::getLogger` |
| `addressFinder` | 动态地址发现 | 无 |
| `asyncContinuationExecutor` | 异步回调执行器 | ForkJoinPool.commonPool |

启用指标后可获取客户端侧约 17 项指标（连接数、活跃请求数、请求/响应字节数、各类型操作计数与耗时直方图、重试次数、事务提交/回滚计数等）。

### 3.2.4 客户端日志

Java 瘦客户端使用 Java 9 引入的 `System.Logger` API：

- 默认通过 `System::getLogger` 委托给 **JUL（java.util.logging）**，INFO 级别输出到控制台；
- 注意日志级别名称映射：System.Logger 的 `DEBUG`/`TRACE` 对应 JUL 的 `FINE`/`FINER`、Logback/Log4j2 的 `DEBUG`/`TRACE`。**在 JUL 的 logging.properties 里写 `DEBUG` 会抛 `IllegalArgumentException: Bad level "DEBUG"`**，这是最常见的配置错误；
- 使用 Logback/Log4j2 时需要实现 `LoggerFactory` 接口做 SLF4J 适配器，传入 builder 的 `.loggerFactory(...)`；
- 常用日志类目：`org.apache.ignite.internal.client.TcpClientChannel`（连接、心跳、请求响应）、`ReliableChannel`（故障转移与重试）、`ClientSql`（SQL 与分区路由）、`ClientTable`（表操作与 schema 版本）、`ClientDataStreamer`（批量流）、`ClientTimeoutWorker`（超时检测）。父类目 `org.apache.ignite.internal.client` 可统一控制；
- 生产建议设置为 WARN；排查连接/故障转移问题时临时开 DEBUG；TRACE 仅在短期诊断时使用。

## 3.3 .NET 客户端

通过 NuGet 安装 `Apache.Ignite` 包：

```csharp
using Apache.Ignite;

var cfg = new IgniteClientConfiguration
{
    Endpoints = { new IPEndPoint(IPAddress.Loopback, 10800) },
    Authenticator = new BasicAuthenticator { Username = "ignite", Password = "ignite" }
};
using IIgniteClient client = await IgniteClient.StartAsync(cfg);
```

要点：

- 全异步 API（`Task`/`ValueTask`），如 `ExecuteBatchAsync`、`ExecuteScriptAsync`；
- 配置对象 `IgniteClientConfiguration` 与 Java 端参数基本对应（连接/心跳/操作超时、重试、SSL 等）；
- 支持与 ASP.NET Core 依赖注入集成，可注册 `IgniteClientGroup`（多个命名客户端）；
- 指标通过 `System.Diagnostics.Metrics` 暴露，meter 名称为 `Apache.Ignite`，可直接对接 OpenTelemetry；
- 支持 LINQ 风格的表查询与 POJO 映射；
- 可以运行用 C# 编写的计算作业（见 3.8 节的 DotNetSidecar 机制）。

## 3.4 C++ 客户端

C++ 客户端需要从源码用 CMake 构建（要求 CMake 3.18+ 与支持 C++17 的编译器）：

```cpp
#include <ignite/client/ignite_client.h>

ignite::IgniteClientConfiguration cfg{"127.0.0.1:10800"};
auto client = ignite::IgniteClient::Start(cfg, std::chrono::seconds(5));
```

要点：

- POJO 与 tuple 之间的转换需要实现 `convert_to_tuple` / `convert_from_tuple` 两个 marshalling 函数模板特化；
- 同样走客户端连接器（默认 10800 端口），支持认证与 SSL；
- API 风格与 Java 客户端对应：`GetTables()`、`GetSql()`、`GetCompute()`、`GetTransactions()` 等。

## 3.5 Table API 与表视图

Table API 是 Ignite 3 中以编程方式操作数据的核心 API，通过 `client.tables()` 获取表，再选择视图：

```java
Table table = client.tables().table("Person");
```

### 3.5.1 四种视图组合

| 视图 | Tuple 模式 | POJO 模式 |
| --- | --- | --- |
| RecordView（整行） | `RecordView<Tuple>` | `RecordView<Person>` |
| KeyValueView（键值分离） | `KeyValueView<Tuple, Tuple>` | `KeyValueView<Integer, Person>` |

- `RecordView`：把行作为整体读写，类似关系表；
- `KeyValueView`：把主键列与其余列拆开为 key/value 两部分，类似 Ignite 2 的 Cache API；
- Tuple 模式无需预先定义类，适合通用工具与动态结构；POJO 模式类型安全，适合业务代码。

常用操作（两种视图一致）：`get / getAll / put / putAll / insert / insertAll / upsert / upsertAll / replace / getAndPut / getAndReplace / delete / deleteExact / deleteAll / remove（KV）/ contains`，以及对应的 `*Async` 异步版本。`deleteExact` 和 `replace` 实现了乐观并发语义（仅当行与给定值完全一致时才删除/替换）。

### 3.5.2 限定名与大小写规则

`QualifiedName.parse("PUBLIC.Person")` 遵循 SQL 标识符规则：不带引号的名称不区分大小写（统一转为大写），带双引号的名称区分大小写。也可以用 `QualifiedName.of(schema, name)` 显式构造。

### 3.5.3 POJO 映射与 Mapper

- 默认按字段名/getter 与列名对应（大小写不敏感）；
- 复杂映射可用 `Mapper.builder(Person.class).map("field", "COLUMN", TypeConverter...)` 自定义，包括字段与列名不同、类型转换（如 `LocalDate` ↔ `DATE` 的自定义格式）；
- POJO 需要满足：有公开无参构造（或匹配字段的构造），字段可访问。

### 3.5.4 条件查询（Criteria 查询）

RecordView/KeyValueView 支持不下推 SQL 的轻量条件查询：

```java
try (Cursor<Person> cur = view.query(null, null, null, null,
        columnValue("countryCode", equalTo("US"))
            .and(columnValue("salary", greaterThan(50000))))) {
    // ...
}
```

支持的条件：`columnValue(col, equalTo / notEqualTo / greaterThan / greaterThanOrEqualTo / lessThan / lessThanOrEqualTo / in / notIn / isNull / isNotNull)`，组合运算 `not / and / or`。适合简单的按键范围过滤，复杂查询仍应使用 SQL。

### 3.5.5 分区分布 API

`PartitionDistribution` API（替代已废弃的 `PartitionManager`）可以查询某个 key 属于哪个分区、分区到节点的映射等，用于诊断数据分布问题。

## 3.6 事务

推荐用法是 `runInTransaction`，它自动处理提交与失败重试（如 leader 切换导致的可重试错误）：

```java
client.transactions().runInTransaction(tx -> {
    view.put(tx, key, value);
    client.sql().execute(tx, "UPDATE Account SET balance = balance - ? WHERE id = ?", amount, id);
});

// 异步版本
client.transactions().runInTransactionAsync(tx -> ...);

// 只读事务
client.transactions().runInTransaction(tx -> {
    // 只读操作，基于快照，不阻塞写
}, new TransactionOptions().readOnly(true).timeoutMillis(10_000));
```

要点：

- `TransactionOptions` 支持 `readOnly(true)`（快照读，不加锁、可读取稍旧的一致快照）与 `timeoutMillis`；
- 也可以手动 `begin()` → `commit()/rollback()`，但需要自己处理可重试异常；
- SQL 脚本、Table API、KV 操作可以混在同一个事务中；
- 事务语义见第二章 2.8（可序列化 RW、快照 RO、WAIT_DIE）。

## 3.7 数据流（Data Streamer）

DataStreamer 用于大批量写入（初始加载、CDC、迁移），吞吐远高于逐条 put/INSERT：

```java
try (var publisher = new SubmissionPublisher<DataStreamerItem<Person>>()) {
    CompletableFuture<Void> streamerFut = view.streamData(
        publisher,
        DataStreamerOptions.builder()
            .pageSize(1000)                       // 每批条数
            .perPartitionParallelOperations(1)    // 每分区并发批数
            .autoFlushInterval(5000)              // 攒批超时毫秒
            .retryLimit(16)                       // 失败重试次数
            .build(),
        null,                                   // receiver（见下）
        null,
        null);

    for (Person p : data) {
        publisher.submit(DataStreamerItem.of(p));           // upsert
        // publisher.submit(DataStreamerItem.removed(key)); // 删除
    }
}
```

要点：

- 基于 Reactive Streams Flow API，自带**背压**；`SubmissionPublisher` 可缓冲，数据来自数据库游标等"不能暂停"的源时应实现 demand-driven publisher（按 request(n) 拉取）；
- 语义为 **at-least-once**：失败重试可能导致重复，业务上需幂等（主键相同覆盖写即天然幂等）；
- 批失败时抛 `DataStreamerException`，其中 `failedItems()` 给出失败条目集合；
- **DataStreamerReceiver**：`streamData` 的重载支持在数据所在节点上执行自定义接收逻辑 `DataStreamerReceiver<T, A, R>`（如服务器端校验、过滤、富化、跨表写），配合结果订阅者接收每个批次的结果（注意结果订阅没有背压，receiver 返回的结果集不宜过大）；
- .NET / C++ 客户端同样提供 DataStreamer。

## 3.8 分布式计算（Compute）

Ignite 3 的计算模型以 **ComputeJob** 为基本单元，配合灵活的 **JobTarget** 指定执行位置：

### 3.8.1 提交作业

```java
JobDescriptor<String, String> job = JobDescriptor.builder(WordCountJob.class)
    .units(new DeploymentUnit("my-unit", "1.0.0"))   // 代码所在部署单元
    .resultClass(String.class)
    .build();

// 单节点执行（任选一个节点）
CompletableFuture<String> fut = client.compute()
    .submitAsync(JobTarget.anyNode(client.clusterNodes()), job, "args");

// 指定节点
client.compute().executeAsync(JobTarget.node("node1"), job, "args");

// 共置执行：在持有指定 key 的节点上执行（数据本地性）
client.compute().executeAsync(JobTarget.colocated("Person", keyTuple), job, "args");
```

### 3.8.2 广播与 MapReduce

- `BroadcastJobTarget.nodes(...)`：在多个节点上同时执行同一作业并收集各自结果；
- `BroadcastJobTarget.table("Person")`：按表分区广播，每个分区在其主节点执行，天然共置；
- `MapReduceTask`：经典 split → map（多节点并行执行子作业）→ reduce（汇总）三段式。

### 3.8.3 作业控制与容错

- `JobDescriptor` 可设置 `priority`（队列优先级）、`maxRetries`（失败重试）；
- `CancelHandle` 用于取消作业，作业内可检查取消标记实现协作式取消；
- SQL 侧可用 `KILL COMPUTE '<jobId>'` 终止作业；
- 故障转移：工作节点宕机时作业按 `maxRetries` 重投；协调节点或客户端断开时的行为可配置；
- 作业事件（排队/执行/完成/失败/取消）可通过事件系统订阅（见 3.11）。

### 3.8.4 .NET 作业

通过 **DotNetSidecar**（随节点部署的 .NET 运行时旁车进程）执行 C# 作业：作业程序集打包为部署单元，sidecar 以 `AssemblyLoadContext` 隔离加载。部署目录**不能有子目录、不能包含 Ignite 自身的 dll**。

## 3.9 代码部署（Deployment Units）

计算作业代码通过部署单元（unit）分发到集群：

```bash
# CLI 部署：--nodes ALL 或 MAJORITY（多数派即可，未部署节点后续自动补齐）
cluster unit deploy my-unit --version 1.0.0 --path ./jobs.jar --nodes MAJORITY

# 查看单元状态
cluster unit list

# 卸载
cluster unit undeploy my-unit --version 1.0.0
```

REST 方式：

```bash
curl -X POST "http://localhost:10300/management/v1/deployment/units/my-unit/1.0.0" \
     -H "Content-Type: application/octet-stream" --data-binary @jobs.jar
curl -X DELETE "http://localhost:10300/management/v1/deployment/units/my-unit/1.0.0"
```

要点：

- 单元状态流转：`UPLOADING` → `DEPLOYED` → `OBSOLETE` → `REMOVING`；
- 文件落地在各节点 `{IGNITE_HOME}/work/deployment` 目录；
- REST 方式还可以直接上传 ZIP（自动解压）；
- 手工把 jar 拷贝进部署目录属于"手动部署"，需要**重启节点**才生效，生产不推荐；
- 单元版本号建议语义化；同一单元可并存多个版本，作业描述符里指定使用哪个版本。

## 3.10 序列化

- 原生类型、Tuple、常见 JDK 类型（String、数值、日期时间、UUID、byte[] 等）自动序列化，无需任何配置；
- 自定义类型通过 `Marshaller<T, byte[]>` 实现，注册到 `JobDescriptor` 的 `inputMarshaller()` / `resultMarshaller()`，可按作业粒度覆盖；
- POJO 表记录由 Mapper 负责行列映射，不涉及显式序列化器。

## 3.11 事件

Ignite 3 提供集群级**事件日志（event log）**：按通道（channel）订阅事件，输出到日志 sink（JSON 格式）。

启用示例（CLI）：

```bash
cluster config update ignite.eventlog.channels.audit.events=[USER_AUTHENTICATION_SUCCESS,USER_AUTHORIZATION_FAILURE]
```

主要事件族：

| 类别 | 事件 |
| --- | --- |
| 连接 | `CLIENT_CONNECTION_ESTABLISHED`、`CLIENT_CONNECTION_CLOSED` |
| 认证 | `USER_AUTHENTICATION_SUCCESS`、`USER_AUTHENTICATION_FAILURE` |
| 授权 | `USER_AUTHORIZATION_SUCCESS`、`USER_AUTHORIZATION_FAILURE`（事件体含 privileges 列表） |
| 查询 | `QUERY_STARTED`、`QUERY_FINISHED`（含 initiator、queryId、schema、sql、txId、耗时、错误等字段） |
| 计算作业 | `COMPUTE_JOB_QUEUED/EXECUTING/FAILED/COMPLETED/CANCELING/CANCELED`，type 区分 `SINGLE/BROADCAST/MAP_REDUCE/DATA_RECEIVER` |
| MapReduce | `MAP_REDUCE_TASK_QUEUED/EXECUTING/FAILED/COMPLETED/CANCELED` |

典型用途：安全审计（认证/授权失败追踪）、慢查询追踪、作业生命周期监控。

## 3.12 从 Java 类建表（Annotations）

可以直接用注解标注 Java 类，让 Ignite 据此在 catalog 中建表，省掉手写 DDL：

```java
@Table("Person")
@Zone("MyZone")
@Index(columns = @ColumnRef("cityId"))
public class Person {
    @Id
    private int id;
    @Column("full_name")
    private String name;
    private Integer cityId;
}
```

```java
client.catalog().createTable(PersonKey.class, Person.class);   // key 类 + value 类
```

注解包括 `@Table`、`@Zone`、`@Id`、`@Column`、`@Index`、`@ColumnRef`。也可以用 `TableDefinition.builder(...)` 以编程方式定义表结构，与 DDL 等价。

## 3.13 JDBC 驱动

依赖 `org.apache.ignite:ignite-jdbc:3.1.0`，驱动类 `org.apache.ignite.jdbc.IgniteJdbcDriver`：

```java
Connection conn = DriverManager.getConnection("jdbc:ignite:thin://127.0.0.1:10800");
```

连接串格式：

```
jdbc:ignite:thin://host[:port][,host[:port]...][/schema][?param1=v1&param2=v2]
```

常用参数：`schema`（默认 PUBLIC）、`connectionTimeZone`、`queryTimeout`、`connectionTimeout`、`reconnectThrottlingPeriod`（默认 30000ms）、`reconnectThrottlingRetries`（默认 3）、`username/password`（基础认证）、`sslEnabled` 及 trustStore/keyStore 系列、`clientAuth`、`ciphers`。

参数优先级：**Connection API 入参 > 连接串中最后出现的同名参数 > Properties 对象**。

事务：`conn.setAutoCommit(false)` 后用 `commit()/rollback()` 手动控制。

已知限制：不支持连接池、不支持多 endpoint 的高级语义；一大批 JDBC 规范接口未实现（流式 setter、Blob/Clob、Savepoint、warnings 等）；`prepareStatement`/`execute*` 仅支持前向只读结果集、不返回生成键。接入 BI 工具前建议先核对支持矩阵。

## 3.14 ODBC 驱动

- 实现 ODBC API 3.8；仅在**连接级**线程安全（一个连接不要跨线程并发使用）；
- Windows：安装器直装（需 VC++ 2017 运行库）；Linux：提供 rpm/deb，需先装 UnixODBC 与支持 C++14 的 libstdc++；
- 支持的 SQL 类型：CHAR/VARCHAR/LONGVARCHAR、SMALLINT/INTEGER/TINYINT/BIGINT、FLOAT/DOUBLE/DECIMAL、BIT、BINARY/VARBINARY/LONGVARBINARY、GUID、DATE/TIME/TIMESTAMP；
- 可配合 pyodbc 使用：

```python
import pyodbc
conn = pyodbc.connect('Driver={Apache Ignite 3};Address=127.0.0.1:10800;')
conn.setencoding(encoding='utf-8')
cursor = conn.cursor()
cursor.execute('SELECT * FROM Person')
```

## 3.15 Python DB API 驱动

遵循 PEP-0249 的 Python 驱动：

```bash
pip install pyignite3_dbapi
```

```python
import pyignite3_dbapi as pyignite_dbapi

with pyignite_dbapi.connect(address=['127.0.0.1:10800'], timeout=10,
                            identity='user', secret='password',   # 认证（可选）
                            use_ssl=True, ssl_certfile='...') as conn:  # SSL（可选）
    with conn.cursor() as cur:
        cur.execute('CREATE TABLE IF NOT EXISTS Person(id INT PRIMARY KEY, name VARCHAR, age INT)')
        cur.executemany('INSERT INTO Person VALUES(?, ?, ?)',
                        [[1, 'John', 30], [2, 'Jane', 32]])
        cur.execute('SELECT * FROM Person ORDER BY id')
        for row in cur.fetchall():
            print(row)
```

要点：

- 事务：默认自动提交；`conn.autocommit = False` 后用 `conn.commit()/conn.rollback()` 手动控制（rollback 回滚所有未提交操作）；
- 连接参数：`schema`（默认 PUBLIC）、`page_size`（单请求最大行数，默认 1024）；
- 要求 Python 3.9–3.12；当前版本不支持多语句脚本执行。

## 3.16 Spring 集成

### 3.16.1 Spring Boot Starter

`spring-boot-starter-ignite-client` 自动装配 `IgniteClient` Bean（starter 版本与集群版本一致，均为 3.1.0）：

```properties
ignite.client.addresses=node1:10800,node2:10800
ignite.client.connectTimeout=5000
ignite.client.operationTimeout=3000
ignite.client.auth.basic.username=ignite
ignite.client.auth.basic.password=ignite
ignite.client.sslConfiguration.enabled=true
ignite.client.sslConfiguration.trustStorePath=/path/to/truststore.jks
ignite.client.metricsEnabled=true
```

- 任何地方 `@Autowired IgniteClient` 即可使用；
- 属性表达不了的配置（重试策略、日志工厂、动态地址发现、异步执行器等）实现 `IgniteClientPropertiesCustomizer` Bean 编程定制；编程式 authenticator 优先于 `auth.basic.*` 属性。

### 3.16.2 Spring Data JDBC 方言

通过 `spring-data-ignite` 方言 + `ignite-jdbc` 驱动，可以直接使用 Spring Data 的 `CrudRepository` 与派生查询方法：

依赖三件套：`spring-boot-starter-data-jdbc` + `spring-data-ignite:3.1.0` + `ignite-jdbc:3.1.0`。

```properties
spring.datasource.url=jdbc:ignite:thin://localhost:10800
spring.datasource.driver-class-name=org.apache.ignite.jdbc.IgniteJdbcDriver
```

还需在 `src/main/resources/META-INF/spring.factories` 注册方言（否则回退为通用 ANSI SQL，部分数据库特定操作会失败）：

```
org.springframework.data.jdbc.repository.config.DialectResolver$JdbcDialectProvider=org.apache.ignite.data.IgniteDialectProvider
```

之后即可常规使用：`@EnableJdbcRepositories` + `@Table("PERSON")` 实体（`@Id`/`@Column` 映射）+ `interface PersonRepository extends CrudRepository<Person, Long>`。注意：**表需要提前在 Ignite 中建好**，Spring Data 不负责建表。

---

# 第四章　安装、配置与集群运维

本章对应官方文档 Configure and Operate 部分，覆盖安装方式、配置体系、存储配置、安全配置、集群生命周期、监控指标、灾难恢复与常见异常处理。

## 4.1 安装方式

### 4.1.1 ZIP 压缩包安装（最通用）

要求：Linux/macOS/Windows，**Java 11+**（生产建议 17 或 21）。

Ignite 3 发行版分两个压缩包：

- `ignite3-db-{version}`：数据库本体，解压后即可启动节点；
- `ignite3-cli-{version}`：CLI 管理工具（`ignite3`），是与集群交互的主要方式，不必装在每台机器上（也可走 REST）。

```bash
unzip ignite3-db-3.1.0.zip && cd ignite3-db-3.1.0
export IGNITE_HOME=$(pwd)
bin/ignite3db          # 启动节点，默认读取 etc/ignite-config.conf
```

节点启动后：若集群已初始化则自动加入；否则等待 `cluster init`。环境变量、JVM 参数等在 `etc/vars.env` 中配置（如 `IGNITE3_EXTRA_JVM_ARGS`、`JVM_MAX_MEM`）。

### 4.1.2 Docker 安装

```bash
docker run -d -p 10300:10300 -p 10800:10800 -p 3344:3344 apache/ignite3:3.1.0
```

- 三个端口：10300（管理 REST）、10800（瘦客户端/JDBC/ODBC）、3344（节点间通信）；
- 持久化数据务必挂载 `/opt/ignite/work` 卷，否则容器删除后数据丢失；
- 多节点可用官方示例 docker-compose（要求 Docker Compose 2.23.1+），启动后记得先 `cluster init`；
- **不建议在 Docker 里跑 CLI 工具**（网络隔离问题），CLI 装在宿主机上；确需容器内运行时要自建 network 并把各容器接入同一网络。

### 4.1.3 Kubernetes 安装

要求 Kubernetes 1.20+；生产推荐 Helm chart，也可手工部署，步骤如下：

1. **ConfigMap**：存放 `ignite-config.conf`（含 `network.nodeFinder.netClusterNodes` 指向 headless service、存储 profile 定义等）。注意 ConfigMap 只读挂载，`node config update` 对它不生效——改配置要 `kubectl edit configmap` 后逐 pod 重启。
2. **Service**：内部通信用 **headless service**（`clusterIP: None`，`publishNotReadyAddresses: True`），让客户端能发现每个 pod 的 IP，保留分区感知能力；外部客户端访问可用 `LoadBalancer`（放弃分区感知）或 `NodePort`。
3. **StatefulSet**：关键点——`IGNITE_NODE_NAME` 从 `metadata.name` 注入（保证节点名稳定）、`IGNITE_WORK_DIR` 指向 PVC 挂载目录（如 `/ai3-work`）、`terminationGracePeriodSeconds` 设大（示例 60000）以便优雅停机、资源 requests/limits 示例为 4C4G。
4. **初始化 Job**：用 CLI 镜像执行 `cluster init --name=ignite --url=http://ignite-svc-headless:10300`。
5. **验证**：`kubectl exec -it ignite-cluster-0 -- /opt/ignite3cli/bin/ignite3 cluster status`，状态应为 `ACTIVE`。

### 4.1.4 DEB/RPM 安装

官方同时提供 deb 与 rpm 包（适用于 Ubuntu/Debian 与 RHEL 系）。安装后二进制、配置、工作目录分别落在标准系统路径，可用 systemd 管理 ignite3db 服务。适合传统虚拟机/物理机部署。

## 4.2 配置体系

### 4.2.1 两级配置

Ignite 3 的配置采用 **HOCON** 格式（也兼容 JSON），单一根节点 `ignite`，分两个层级：

| 层级 | 作用范围 | 存储位置 | 管理方式 |
| --- | --- | --- | --- |
| **集群配置** | 全集群共享，自动传播到所有节点 | 集群内（经 CMG/MetaStorage 复制） | `cluster config show/update`，运行时即可生效（个别参数需重启） |
| **节点配置** | 仅本节点 | 本地 `etc/ignite-config.conf` | `node config show/update`，多数修改需**重启节点** |

```bash
node config show ignite.clientConnector                 # 查看局部配置
node config update ignite.network.shutdownTimeoutMillis=20000
cluster config update --file ../ignite-config.conf      # 用文件批量更新
cluster config update ignite.system.idleSafeTimeSyncIntervalMillis=600
node config update -n node1 <配置>                       # 更新非默认节点
```

注意：**CLI 直接设置的值优先于配置文件**；更新多参数时传入一段合法 HOCON，CLI 会解析并一次性应用。

### 4.2.2 配置文件与环境变量

- `etc/ignite-config.conf`：节点启动配置，改动后重启生效；
- `etc/vars.env`：工作目录路径、JVM 参数（`IGNITE3_EXTRA_JVM_ARGS`）等与"节点/集群配置"无关的环境项；
- 集群配置只能经 CLI/REST 管理，不存在于本地文件中。

### 4.2.3 常用节点配置项速查

| 配置路径 | 默认 | 说明 |
| --- | --- | --- |
| `ignite.network.port` | 3344 | 节点间通信端口 |
| `ignite.network.nodeFinder.netClusterNodes` | localhost:3344 | 节点发现地址列表 |
| `ignite.clientConnector.port` | 10800 | 客户端连接端口 |
| `ignite.rest.port` | 10300 | 管理 REST 端口 |
| `ignite.compute.threadPoolSize` | 10 | 计算作业线程池大小 |
| `ignite.compute.queueMaxSize` | Integer.MAX | 计算队列上限 |
| `ignite.compute.statesLifetimeMillis` | 60000 | 作业状态保留时长 |
| `ignite.system.cmgPath` / `metastoragePath` | work 目录下 | CMG / MetaStorage 数据存放路径 |
| `ignite.storage.profiles` | default (aipersist) | 存储 profile 列表 |

完整的节点/集群配置参数清单见官方参考页（Node Configuration / Cluster Configuration / Storage Profiles / CLI Configuration），本文不一一罗列。

## 4.3 存储配置

### 4.3.1 存储 Profile 机制

- **存储 profile = 存储引擎 + 引擎参数** 的命名组合，是表与底层引擎之间的桥梁；
- 每个节点可定义多个 profile；**一张表只用一个 profile**；
- 建 zone 时用 `STORAGE PROFILES ['p1','p2']` 限定该 zone 内表可用的 profile 集合；
- 建表时 `STORAGE PROFILE 'p1'` 指定具体 profile；zone 与 profile **建表后不可更改**；
- 各节点上同名 profile 的参数可以不同（按本地配置存储）；若某节点没有该 profile，表数据不会落在该节点。

```bash
# 查看当前 profile
node config show ignite.storage.profiles

# 新增 rocksdb profile（改完需重启节点）
node config update "ignite.storage.profiles:{rocksProfile{engine:rocksdb,size:10000}}"
```

```sql
CREATE ZONE exampleZone (PARTITIONS 2, REPLICAS 3) STORAGE PROFILES ['profile1', 'profile3'];
CREATE TABLE exampleTable (key INT PRIMARY KEY, my_value VARCHAR)
  ZONE exampleZone STORAGE PROFILE 'profile1';
```

### 4.3.2 三种引擎的配置要点

- **aimem**（纯内存）：参数主要有 `pageSizeBytes`、`initSizeBytes`、`maxSizeBytes`、`replacementMode`（`CLOCK`/`RANDOM_LRU`/`SEGMENTED_LRU`，达到上限时的页淘汰策略）。断电丢数据，靠多副本保证可用性；
- **aipersist**（默认 profile 使用的持久化引擎）：在 aimem 参数基础上增加 `checkpointIntervalMillis`、`checkpointPageBufSize`、`fsyncMode`（`LOG_ONLY`/`SYNC`/`BACKGROUND`）、写节流阈值（`throttlingStartRatio`≈0.66、`throttlingStopRatio`≈0.9）、`dataRegionSize` 等；数据先写内存页 + WAL，checkpoint 时刷盘；
- **rocksdb**：`size`（写缓冲）、`writeBufferSize`、`flushOnCompaction` 等 RocksDB 风格参数；数据主体在磁盘 LSM 树中，内存需求小。

引擎与 profile 的选择原则：热数据、低延迟优先 aimem/aipersist；海量温冷数据、内存受限选 rocksdb。三者可在同一集群内按表混用。

## 4.4 安全配置

### 4.4.1 基础认证

默认情况下**安全是关闭的**——任何用户都能修改集群、上传代码并远程执行。开启认证的推荐流程：在初始化时就带上安全配置：

```hocon
ignite {
  security {
    enabled: true
    authentication {
      providers = [
        {
          name = default
          type = basic
          users = [ { username = ignite, password = "ignite", displayName = administrator, roles = [system] } ]
        }
      ]
    }
  }
}
```

```bash
cluster init --name=sampleCluster --config-files=/cluster-config.conf
# 若初始化时未开启，也可事后开启（开启后所有连接会被断开，需带凭据重连）：
cluster config update ignite.security.enabled=true
connect http://127.0.0.1:10300 --username ignite --password ignite
# 修改默认用户密码：
cluster config update ignite.security.authentication.providers.default.users.ignite.password=myPass
```

⚠️ 警告：**丢失所有 system 角色的账号将永久失去集群管理员权限**。3.1 的授权模型：认证 + 角色（system/admin 等），授权粒度较粗，细粒度权限依赖事件审计兜底。

### 4.4.2 SSL/TLS

三类通信通道都可启用 SSL，均在**节点级**配置，且只支持 **PKCS12/JKS keystore**（不支持裸证书路径）：

| 通道 | 配置路径 | 说明 |
| --- | --- | --- |
| REST（用户↔集群） | `ignite.rest.ssl`（SSL 端口默认 10400，支持 `dualProtocol`、`httpToHttpsRedirection`） | 管理接口加密 |
| 客户端（客户端↔集群） | `ignite.clientConnector.ssl` | 瘦客户端/JDBC/ODBC 加密 |
| 节点间 | `ignite.network.ssl` | Messaging/Scalecube 加密 |

各通道通用参数：`enabled`、`ciphers`、`clientAuth`（`none`/`optional`/`require`，即 mTLS）、`keyStore{type,path,password}`、`trustStore{...}`。

客户端侧对应配置：Java 用 `SslConfiguration.builder()`；JDBC 在连接串加 `sslEnabled=true&trustStorePath=...&clientAuth=require&keyStorePath=...`；.NET 用 `IgniteClientConfiguration.SslStreamFactory`；CLI 用 `cli config set cli.trust-store.*`。

安全基线建议：节点间通信跑在隔离网络并启用 `network.ssl`；对外只暴露 10800（客户端）与 10300（REST）并启用 SSL + 认证。

## 4.5 集群生命周期

### 4.5.1 物理拓扑与逻辑拓扑

- **物理拓扑**：通过 `network.nodeFinder` 互相发现的节点集合（成员关系会传播：只要某节点的发现列表里有你，你最终会知道所有节点）；
- **逻辑拓扑**：经 CMG 验证并接受、真正组成集群的节点集合。

未初始化的集群里节点只做信息交换，不处理请求，直到收到 `cluster init`。

时钟要求：所有节点时钟偏差不得超过 `schemaSync.maxClockSkewMillis`（默认 500ms），否则影响事务正确性；`schemaSync.delayDurationMillis` 控制 schema 变更的传播等待（同时影响 DDL 生效延迟）。

### 4.5.2 初始化过程

```bash
cluster init --name=myCluster \
  --cluster-management-group=node1,node2,node3 \
  --metastorage-group=node1,node2,node3
```

- CMG 与 MetaStorage 组各形成一个 **Raft 组**并选主，然后通知其余节点加入；
- 只指定一个参数时另一组复用同一批节点；都不指定则按节点名字母序自动选：集群 ≤3 节点全部入选；4 节点选 3 个（保持奇数）；≥5 节点选 5 个；
- **推荐 3/5/7 个成员**：越多越稳但略有性能开销。

CMG 多数派丢失：集群仍可读写在途事务、处理用户请求，但**不能加新节点、不能让离群节点重归、CREATE INDEX 会挂起**。
MetaStorage 多数派丢失：**集群不可用，可能丢数据**（MetaStorage 存有 catalog——全部表结构、索引、zone 信息等）。

### 4.5.3 节点加入与重归

- 新节点：进入物理拓扑 → CMG 发出邀请 → 节点发起验证请求 → CMG 验证通过 → 从 MetaStorage 拉取集群元信息 → 加入逻辑拓扑；
- 节点离开物理拓扑后会**立即被移出逻辑拓扑，且不能以同一节点 ID 重归**——必须重启节点生成新 ID 后按新节点流程加入（重启中会先做本地恢复）。

## 4.6 监控与指标

### 4.6.1 架构与开销

- 指标按 **source**（来源，如 jvm、storage、sql、raft、placement-driver）组织，**默认全部关闭**（避免性能开销），按需启用；
- 通过 **exporter** 导出到外部工具：JMX、日志、OpenTelemetry；
- 也可以直接用 SQL 查 **SYSTEM schema 的系统视图**，无需外部工具。

```bash
node metric source list          # 列出可用 source
node metric source enable jvm    # 启用（cluster metric source enable jvm 全集群启用）
node metric list                 # 查看指标
```

### 4.6.2 导出器配置

```bash
# JMX
cluster config update ignite.metrics.exporters.myJmxExporter.exporterName=jmx

# 日志导出（periodMillis 默认 30000；enabledMetrics 支持 jvm.* 通配）
cluster config update ignite.metrics.exporters.logPush \
  '{"exporterName":"logPush","periodMillis":30000,"oneLinePerMetricSource":true,"enabledMetrics":[]}'

# OpenTelemetry → Prometheus（Prometheus 需开启 OTLP receiver）
cluster config update ignite.metrics.exporters.test: \
  {exporterName:otlp, endpoint:"http://localhost:9090/api/v1/otlp/v1/metrics", protocol:"http/protobuf"}

# OpenTelemetry → Grafana Cloud（带 Authorization 头）
cluster config update ignite.metrics.exporters.test: \
  {exporterName:otlp, endpoint:"https://otlp-gateway-prod-xxx.grafana.net/otlp", protocol:"http/protobuf", headers{Authorization.header="Basic ..."}}
```

OTLP exporter 参数：`endpoint`、`protocol`（`grpc`/`http/protobuf`）、`compression`（默认 gzip）、`periodMillis`（默认 30000）、`headers`、SSL 系列。每个节点独立向 endpoint 推送。

### 4.6.3 重要指标速览

| Source | 代表指标 | 用途 |
| --- | --- | --- |
| `client.handler` | SessionsActive/Rejected、RequestsActive/Processed/Failed、TransactionsActive、CursorsActive | 客户端连接与请求健康度 |
| `jvm` | memory.heap.Used/Max、gc.CollectionTime、UpTime | 内存与 GC |
| `os` | CpuLoad、LoadAverage | 主机负载 |
| `placement-driver` | ActiveLeasesCount、LeasesCreated/Prolonged、CurrentStable/PendingAssignmentsSize | 租约与分区分配 |
| `raft` | 各 disruptor 的 Batch/Stripes 直方图 | Raft 流水线效率 |
| `metastorage` | SafeTimeLag、IdempotentCacheSize | MetaStorage 健康 |
| `clock.service` | ClockSkewExceedingMaxClockSkew | **时钟偏移超标告警（事务正确性红线）** |
| `resource.vacuum` | 各类 transaction meta vacuum 计数 | 事务元数据清理 |
| `storage.aipersist.{profile}` | CurrDirtyRatio/TargetDirtyRatio、CpWrittenPages/CpWriteSpeed、SpeedBasedThrottlingPercentage、ThrottleParkTime | 持久化引擎写放大与节流监控 |

### 4.6.4 系统视图

`SYSTEM` schema 下的内置视图可用普通 SQL 查询：

```sql
SELECT * FROM system.system_views;    -- 列出全部系统视图
```

重点视图：

- `COMPUTE_TASKS`：计算任务状态与时间线；
- `GLOBAL_PARTITION_STATES` / `GLOBAL_ZONE_PARTITION_STATES`：分区全局状态（`AVAILABLE/DEGRADED/READ_ONLY/UNAVAILABLE`）——**日常巡检和灾排第一入口**；
- `LOCAL_ZONE_PARTITION_STATES`：本节点视角的分区状态；
- `INDEXES` / `INDEX_COLUMNS`：索引类型（HASH/SORTED）、唯一性、构建状态（`REGISTERED/BUILDING/AVAILABLE/STOPPING`）；
- 其余还有表、zone、事务、SQL 查询、锁、schema 等相关视图。

## 4.7 灾难恢复

### 4.7.1 分区状态模型

**本地状态**（单副本视角）：`Healthy` / `Initializing`（Raft 组初始化中）/ `Snapshot installation`（全量状态传输中）/ `Catching-up`（追日志，落后约 100 条以内）/ `Broken`（状态机异常，需人工介入）/ `Unavailable`。

**全局状态**（用户视角）：`Available`（可读写）/ `Degraded`（可用但副本数低于配置，风险升高）/ `Read-only`（多数派不在，仅剩副本可读历史数据）/ `Unavailable`（不可服务）。

### 4.7.2 分区恢复流程

先用 `recovery partition states --global` 定位问题分区，然后：

**少数派离线**（如 3 副本挂 1 个）：
1. 尽快把节点拉回——在 scale-down 超时前回来则自动增量复制（日志复制或全量状态传输）；超时未归则触发 rebalance 把副本发到新节点（可能耗时几十分钟）；
2. 状态回到 `Healthy/Available` 即完成；
3. 若为 `Broken`：`recovery partitions restart`；仍不行则 `recovery partitions restart --with-cleanup`（删本地数据后从集群恢复）；
4. 若为 `Read-only/Degraded/Unavailable`：`recovery partitions reset`。

**多数派离线**（如 2 副本挂 1 个、3 副本挂 2 个）：主副本还在则分区变只读（租约过期前可读），不在则不可用。流程同上，但 `reset` **通常会丢失该分区的最新数据**，并可能因跨分区事务波及其它分区的部分最新写入。

**分区全部副本丢失**：`reset` 后该分区数据全丢。优先尽力把原节点拉回。

### 4.7.3 系统组恢复（CMG / MetaStorage）

征兆：日志出现 `Send with retry timed out [retryCount = 11, groupId = cmg_group]`（或 `metastorage_group`）；节点 `restart` 后停在 `Local CMG state recovered, starting the CMG` 而没有 `Successfully joined the cluster`。

> ⚠️ 系统组恢复命令**有脑裂风险**，仅在多数派**永久**丢失时作为最后手段使用。

CMG 多数派丢失：
1. 先尝试重启 CMG 节点；
2. 失败则强制指定新多数派：`recovery cluster reset --url=<node-url> --cluster-management-group=<新CMG节点列表>`（`--url` 指向的节点必须在新组内，它成为 Repair Conductor）；
3. 对当时宕机/被隔离、未参与修复的节点：恢复后用 `recovery cluster migrate --old-cluster-url=... --new-cluster-url=...` 迁入新集群。

MetaStorage 多数派丢失：流程类似，`recovery cluster reset --url=<node> --metastorage-replication-factor=N`（省略 CMG 参数则从 CMG leader 取当前投票成员）。若某节点 MetaStorage 版本与修复后的 leader 分叉，该节点启动会报 `MetastorageDivergedException (META-7)`——**清空该节点数据，以空白节点重新加入**。

## 4.8 常见异常代码速查

Ignite 异常只返回 UUID + 错误码，完整堆栈在节点日志。按类别记忆常用码：

| 错误码 | 含义与处理 |
| --- | --- |
| `IGN-CLIENT-1/2/3/6/8` | 连接失败/协议不兼容/版本不匹配/连到了不同集群/连错端口（如 REST 端口） |
| `IGN-AUTHENTICATION-1/2/3` | 认证类型不支持/凭据错误/未配置 basic provider |
| `IGN-SQL-3/4/5/6/8/9` | 语法错/语义校验错/约束违反/语句被取消/无法映射到当前拓扑（稍候重试）/外部事务内禁止事务控制语句 |
| `IGN-TX-4/5` | 锁冲突（WAIT_DIE 中止，重试）/锁等待超时 |
| `IGN-TX-10/15` | 只读事务时间戳早于低水位线，数据已被 GC——用更新的时间戳重试 |
| `IGN-TX-12` | 主副本租约过期，重试；频繁出现要查节点故障/GC 停顿 |
| `IGN-REP-5/6` | 副本未就绪/请求打到非主副本（客户端会自动重定向，重试即可） |
| `IGN-COMPUTE-4` | 计算队列溢出，调大 `compute.queueMaxSize` |
| `IGN-CODEDEPLOY-1/2` | 部署单元不存在/重复 |
| `IGN-NETWORK-2` | 端口被占用 |
| `IGN-STORAGE-1/2` | 灾后索引不一致（等自动重建）/持久化数据损坏（从备份或其它副本恢复） |
| `IGN-WORKERS-1/2` | 系统 worker 心跳超时/关键操作超时——节点疑似卡死，重启 |
| `IGN-CMN-65535` | 内部错误，大概率是 bug，提 issue |
| `IGN-META-7` | MetaStorage 分叉，清数据后重新加入（见 4.7.3） |

## 4.9 本章小结

- 安装四选一：ZIP（最通用）、Docker（快速试验）、K8s（生产推荐 Helm）、DEB/RPM（传统主机）；
- 配置两级模型：集群配置走 CLI 即时生效，节点配置改后重启；记住 `cluster/node config show|update`；
- 存储按 profile 抽象，zone 圈定可用集合，表绑定其一，建表后不可改；
- 安全三件套：认证（`security.enabled`）、三通道 SSL、最小暴露面；
- 运维三板斧：`cluster status`、系统视图（尤其 `GLOBAL_ZONE_PARTITION_STATES`）、指标（OTLP 接 Prometheus/Grafana）；
- 灾难恢复分级处理：分区问题用 `recovery partitions *`，系统组问题用 `recovery cluster reset/migrate`——后者是核按钮，慎用。

---

# 第五章　SQL 详解

本章对应官方文档 SQL 部分：引擎架构、查询执行、数据类型、DDL/DML、分布区、运维命令、事务、函数与性能调优。

## 5.1 SQL 引擎架构

Ignite 3 的 SQL 引擎基于 **Apache Calcite** 构建，具有以下特点：

- **为分布式环境优化**：不限于单一 map-reduce 阶段，支持复杂的多阶段数据归集；
- **事务化 SQL**：所有表都支持带完整事务保证的 SQL；
- **集群级系统视图**：动态更新的集群全景信息（见 4.6.4）；
- **多索引查询**：同一查询可同时使用多个索引加速；
- **贴近 SQL 标准**：高度遵循现代 SQL 标准；
- **改进的优化算法**：基于规则（planner rules）反复改写关系表达式求最优；
- **整体高性能**：执行灵活，内存与 CPU 效率高。

执行流水线六阶段（第二章已详述）：解析 → 校验 → 优化 → 分段 → 分布式执行 → 归并。SQL 的执行最终落在各分区主副本所在的节点上完成。

## 5.2 执行查询（Java SQL API）

### 5.2.1 基本操作

```java
// 建表（executeScript 可一次执行多条语句）
client.sql().executeScript(
    "CREATE TABLE CITIES (ID INT PRIMARY KEY, NAME VARCHAR);"
  + "CREATE TABLE ACCOUNTS (ACCOUNT_ID INT PRIMARY KEY, CITY_ID INT, "
  + "FIRST_NAME VARCHAR, LAST_NAME VARCHAR, BALANCE DOUBLE)");

// 参数化批量插入
long rowsAdded = Arrays.stream(client.sql().executeBatch(tx,
        "INSERT INTO ACCOUNTS VALUES (?, ?, ?, ?, ?)",
        BatchedArguments.of(1, 1, "John", "Doe", 1000.0d)
                        .add(2, 1, "Jane", "Roe", 2000.0d)))
    .sum();

// 查询：ResultSet 必须关闭（try-with-resources）
try (ResultSet<SqlRow> rs = client.sql().execute(null,
        "SELECT a.FIRST_NAME, c.NAME FROM ACCOUNTS a JOIN CITIES c ON c.ID = a.CITY_ID")) {
    while (rs.hasNext()) {
        SqlRow row = rs.next();
        row.stringValue(0);   // 按序号或列名取值
    }
}
```

### 5.2.2 序列（Sequence）生成主键

```java
client.sql().execute(null, "CREATE SEQUENCE IF NOT EXISTS defaultSequence;");
client.sql().execute(null,
    "CREATE TABLE IF NOT EXISTS Person (ID BIGINT DEFAULT NEXTVAL('defaultSequence') PRIMARY KEY, NAME VARCHAR)");
```

### 5.2.3 按分区查询（`__part` 系统列）

```sql
SELECT city_id, id, "__part" FROM Person;        -- 查出每行所在分区
SELECT city_id, id FROM Person WHERE "__part"=23; -- 只扫指定分区
```

### 5.2.4 脚本与取消

- `executeScript()`：多条语句按序执行、自动攒批（类似 Ignite 2 的 `SET STREAMING`），大量语句时性能显著更好。注意：**语句在首页结果就绪时即视为完成**，脚本中靠后的语句可能影响前面 SELECT 的大数据量结果；
- 取消查询：`CancelHandle.create()` → 把 token 传给 `executeAsync` → `cancelHandle.cancelAsync()`（同 handle 的 token 可批量取消）；.NET 用 `CancellationTokenSource`；C++ 用 `cancel_handle`；
- 也可以用 SQL `KILL QUERY '<id>'`，query id 从 `SYSTEM.SQL_QUERIES` 视图获取。

## 5.3 数据类型

### 5.3.1 类型总表与语言映射

| SQL 类型 | ColumnType | Java | .NET | C++ | 备注 |
| --- | --- | --- | --- | --- | --- |
| BOOLEAN | BOOLEAN | Boolean | bool | bool | |
| TINYINT | INT8 | Byte | sbyte | int8_t | -128~127 |
| SMALLINT | INT16 | Short | short | int16_t | |
| INT | INT32 | Integer | int | int32_t | 别名 INTEGER |
| BIGINT | INT64 | Long | long | int64_t | |
| DECIMAL(p,s) | DECIMAL | BigDecimal | BigDecimal | big_decimal | 默认精度 32767、标度 0；最大标度 16383 |
| REAL | FLOAT | Float | float | float | IEEE 754 单精度 |
| DOUBLE | DOUBLE | Double | double | double | 别名 DOUBLE PRECISION |
| VARCHAR(n) | STRING | String | string | std::string | 默认长 65536，最大 2^31 |
| VARBINARY(n) | BYTE_ARRAY | byte[] | byte[] | vector<byte> | 别名 BINARY |
| TIME | TIME | LocalTime | LocalTime | ignite_time | 精度 0~3 |
| DATE | DATE | LocalDate | LocalDate | ignite_date | |
| TIMESTAMP | DATETIME | LocalDateTime | LocalDateTime | ignite_date_time | 毫秒以上精度被忽略 |
| TIMESTAMP WITH LOCAL TIME ZONE | DATETIME | LocalDateTime/Instant | | | 不存时区，读出时按会话时区转换 |
| UUID | UUID | UUID | Guid | uuid | |
| NULL | NULL | Void | Null | nullptr | |

### 5.3.2 重要注意事项

- **CHAR 受限**：只能用于表达式（如 `CAST('a' AS CHAR(3))`），**不能用于 DDL**——建表请用 VARCHAR；
- **DECIMAL 特例**：允许标度大于精度（如 `DECIMAL(3,6)` 只能存 ±0.000999 之间的小数）；Java `BigDecimal` 默认推导为 `DECIMAL(28,6)`，超出需 `CAST(? AS DECIMAL(100,50))`；
- **TIMESTAMP 精度**：只支持到毫秒（3 位），更靠后的位被忽略；
- **Java 旧类型不支持**：`java.sql.Time`、`java.util.Date` 不能用于 Table API，请用 `java.time` 包；
- **隐式类型转换仅限同族**：Numeric 族（TINYINT→…→DOUBLE）内部可转，跨族（如 VARCHAR↔INT）必须显式 CAST。

## 5.4 DDL 详解

### 5.4.1 CREATE TABLE

```sql
CREATE TABLE IF NOT EXISTS Person (
  id int primary key,
  city_id int default 1,
  name varchar,
  company varchar
) COLOCATE BY (city_id)            -- 共置键（必须是主键的子集，顺序与主表主键一致）
  ZONE MYZONE                       -- 分布区（可先不存在的 zone，写数据前建好即可）
  STORAGE PROFILE 'default';        -- 存储 profile（大小写敏感字符串）
```

### 5.4.2 ALTER TABLE

```sql
ALTER TABLE Person ADD COLUMN (code varchar, gdp double);
ALTER TABLE Person DROP COLUMN (code, gdp);
ALTER TABLE Person ALTER COLUMN age SET DATA TYPE BIGINT;
ALTER TABLE Person ALTER COLUMN Name SET DATA TYPE varchar(11);
ALTER TABLE Person ALTER COLUMN city DROP NOT NULL;   -- 允许（非主键列）
ALTER TABLE Person ALTER COLUMN city SET DEFAULT 'unknown';
ALTER TABLE Person ALTER COLUMN city DROP DEFAULT;
```

类型转换限制：**只允许放大**——`FLOAT→DOUBLE`、`INT8/16/32→INT64`、DECIMAL 非主键列可增精度、STRING/BYTE_ARRAY 非主键列可增长度；**禁止**改标度、禁止 NULLABLE→NOT NULL、主键列一律不可改类型。DROP COLUMN 前需先手动 DROP 相关索引。

### 5.4.3 DROP TABLE

表被标记删除后，数据在低水位线（LWM）推进到位前仍对**只读事务**可见（可时间旅行读），之后由垃圾回收清除。

### 5.4.4 索引

```sql
CREATE INDEX IF NOT EXISTS dept_name_idx ON Person (department_id DESC, name ASC);  -- 默认 TREE
CREATE INDEX name_surname_idx ON Person USING HASH (name, surname);                  -- 哈希索引
DROP INDEX IF EXISTS dept_name_idx;
```

- TREE（排序）索引支持范围扫描与排序，HASH 索引只支持等值；
- 同一列不能在一个索引里出现两次；
- **索引构建要等所有"先于建索引"的事务结束才开始**；有挂起事务时构建不会启动，状态（含等待原因）可见于 `SYSTEM.INDEXES` 视图；
- DROP INDEX 后索引处于 STOPPING 直到存量事务结束，空间在 LWM 越过后才释放。

## 5.5 分布区（Distribution Zones）DDL

```sql
CREATE ZONE IF NOT EXISTS exampleZone
  (PARTITIONS 25,                      -- 分区数
   REPLICAS 3,                         -- 每分区副本数
   AUTO SCALE UP 300,                  -- 新节点加入后延迟 300s 再扩容调整（默认 5；OFF 关闭）
   AUTO SCALE DOWN 600,                -- 拓扑变化后延迟 600s 再缩容（默认 OFF）
   NODES FILTER '$[?(@.storage == "SSD")]',  -- JSONPath 节点属性过滤
   CONSISTENCY MODE 'STRONG_CONSISTENCY',     -- 或 HIGH_AVAILABILITY
   QUORUM SIZE 2)                      -- 复制组多数派大小
STORAGE PROFILES ['default', 'rocksProfile'];
```

- `CONSISTENCY MODE`：多数派丢失时，`STRONG_CONSISTENCY`（默认）数据不可用直到多数派恢复；`HIGH_AVAILABILITY` 继续读写剩余节点（可能丢数据）；
- `QUORUM SIZE` 取值范围：单副本时为 1，多副本时最小 2、最大为副本数的一半向上取整；默认 5+ 副本时为 3，否则取 min(2, 副本数)；
- zone 名用标识符（大小写不敏感）或字符串（大小写敏感）均可；
- `ALTER ZONE ... SET (...)` 可改 PARTITIONS/REPLICAS/FILTER/SCALE 参数/QUORUM，`RENAME TO` 改名，`DROP ZONE` 删除。

## 5.6 DML 与 SELECT

- 标准四件套：`INSERT`、`UPDATE`、`DELETE`、`MERGE`（`WHEN MATCHED` / `WHEN NOT MATCHED` 至少出现一个）；
- `SELECT` 支持完整的现代 SQL：各类 JOIN（共置与非共置分布式 JOIN、跨表 JOIN）、子查询、聚合、窗口函数、CTE 等；
- 支持优化器 hint 注释（见 5.10）。

## 5.7 运维命令

### 5.7.1 COPY INTO（批量导入导出）

```sql
-- CSV 导入（带表头、自定义分隔与引号）
COPY FROM '/path/data.csv' INTO Table1 (name, age) FORMAT CSV
WITH 'delimiter'='|', 'quoteChar'='~', 'batchSize'='2048', 'null'='no data';

-- 导出查询结果为 CSV
COPY FROM (SELECT name, age FROM Table1) INTO '/path/data.csv' FORMAT CSV WITH 'header'='true';

-- 从 S3 导入
COPY FROM 's3://mybucket/data.csv' INTO Table1 (name, age) FORMAT CSV
WITH 's3.access-key-id'='keyid', 's3.secret-access-key'='secret';

-- Parquet（支持分区目录 + 正则 pattern）
COPY FROM '/tmp/partitioned_table_dir' INTO city (id, name, population)
FORMAT PARQUET WITH 'pattern'='.*';

-- Iceberg（本地 HadoopCatalog / S3 GlueCatalog）
COPY FROM Table1 (id,name,height) INTO '/tmp/person.i/' FORMAT ICEBERG
WITH 'table-identifier'='person', 'catalog-impl'='org.apache.iceberg.hadoop.HadoopCatalog';
```

支持格式：**CSV、PARQUET、ICEBERG**；支持 S3（`s3.client-region/access-key-id/secret-access-key`）；Iceberg 支持 catalog 属性与 `table-identifier`（Glue 要求 `db.table` 全小写）。目标表必须已存在。

### 5.7.2 KILL 系列

```sql
KILL QUERY '<query_id>' [NO WAIT];      -- 取消查询（全集群级联取消）
KILL TRANSACTION '<tx_id>' [NO WAIT];   -- 中止事务
KILL COMPUTE '<job_id>' [NO WAIT];      -- 取消计算作业
```

ID 分别从 `SYSTEM.SQL_QUERIES`、`SYSTEM.TRANSACTIONS`、`SYSTEM.COMPUTE_JOBS` 视图获取；`NO WAIT` 立即返回，状态经视图跟踪。

## 5.8 SQL 事务控制

```sql
START TRANSACTION READ WRITE;   -- 或 READ ONLY
INSERT INTO Person VALUES (1, 'John', 'Smith');
UPDATE Account SET balance = balance - 100 WHERE id = 1;
COMMIT;                          -- 或 ROLLBACK
```

⚠️ 限制：**事务控制语句只能出现在脚本（executeScript）中**；**事务内不允许 DDL**；在外部事务（如 JDBC 关闭 autoCommit）里使用会报 `IGN-SQL-9`。

## 5.9 函数速览

官方 Operators and Functions 参考页提供完整清单，按类别记忆：

- **聚合**：`AVG/COUNT/MAX/MIN/SUM/ANY_VALUE/EVERY/SOME/GROUPING`；
- **JSON**：`JSON_VALUE/JSON_QUERY/JSON_EXISTS/JSON_OBJECT/JSON_ARRAY/JSON_KEYS/JSON_LENGTH/JSON_REMOVE/JSON_PRETTY/JSON_TYPE/IS JSON ...` 等一整套；
- **正则**：POSIX 正则匹配（大小写敏感/不敏感）、`REGEXP_REPLACE`；
- **数值**：`MOD/EXP/POWER/LN/LOG10/ABS/RAND/RAND_INTEGER/三角函数/SQRT/CBRT/ROUND/SIGN/TRUNCATE/PI` 等；
- **字符串**：`CONCAT(||)/SUBSTRING/UPPER/LOWER/LENGTH/TRIM/REPLACE/POSITION/OCTET_LENGTH` 等；
- **日期时间**：`CURRENT_DATE/CURRENT_TIME/CURRENT_TIMESTAMP/EXTRACT/DATEADD/DATEDIFF` 等；
- **条件与系统**：`CASE/COALESCE/NULLIF/CAST/SYSTEM_USER` 等。

SQL Conformance 参考页列出了与 SQL:2016 标准的符合度以及保留关键字清单（建表命名时避让，如非用不可则加双引号）。

## 5.10 SQL 性能调优

### 5.10.1 优化器 Hint

格式为特殊注释 `/*+ HINT */`（**hint 名前后必须有空格**，紧跟在操作符之后，一个操作符只能带一个 hint）：

```sql
SELECT /*+ NO_INDEX */ T1.* FROM TBL1 T1 WHERE T1.V1=? AND T1.V2=?;
SELECT /*+ FORCE_INDEX(TBL1_IDX2, TBL2_IDX1) */ T1.V1, T2.V1 FROM TBL1 T1, TBL2 T2 WHERE ...;
SELECT /*+ FORCE_INDEX('TBL2_idx1') */ ...   -- 引号参数大小写敏感
```

- `FORCE_INDEX`：强制走索引（空参=任意可用索引；可指定一个或多个）；
- `NO_INDEX`：禁用索引（同理）；
- 两者不能同时出现在同一查询；引用不存在的索引会报错 `Hints mentioned indexes "..." were not found`；
- hint 是"建议"，优化器在不适用的场景会跳过。

### 5.10.2 EXPLAIN 两种形态

```sql
EXPLAIN PLAN FOR SELECT ...;      -- 关系算子树：查优化器问题
EXPLAIN MAPPING FOR SELECT ...;   -- 查询分段到节点的映射：查共置/分布问题
```

- PLAN 输出算子树：`TableScan`（全表）、`IndexScan`（含索引名/类型/排序方向）、`Sort`、`HashJoin/MergeJoin/NestedLoopJoin`、`Project`、`Limit`、`Exchange`（节点间数据重分布）；每个节点带 `est: (rows=N)` 估算行数；
- MAPPING 输出 Fragment 结构：`Fragment#0 root` 是用户游标所在（通常单节点），含 `Receiver` 算子；其它 Fragment 含 `Sender` 与各表的 `TableScan`；`executionNodes` 列出执行节点、`partitions` 列出每表分区到节点的映射——**如果预期共置的 JOIN 却出现跨节点 Exchange/Sender，就是共置没生效的信号**。

### 5.10.3 查询批处理

一次调用执行多条语句时相似请求自动攒批。大脚本推荐顺序：**先 DDL → 再授权 → 最后灌数据**。

### 5.10.4 相关子查询陷阱（重点）

相关子查询（引用外层行值的子查询）会**对外层每一行重新执行一次**，Ignite 3 不做自动缓存优化：

- 官方实测：带主键过滤的高选择性外层 + 便宜标量子查询没问题（0.007s）；去掉过滤的全表版本飙到 2.4s；
- 改写为 JOIN 后通常有数量级提升：`NOT EXISTS` 子查询（3.2s）→ `LEFT JOIN ... GROUP BY ... HAVING COUNT(...)=0`（0.04s）；部门最低工资相关子查询（18s）→ 先 `GROUP BY` 预聚合再 JOIN（0.02s）；
- 改写要点：GROUP BY 必须包含外表唯一键（不确定就带上全部主键列）；`HAVING COUNT()` 要统计 JOIN 右侧保证非空的列；需要保留无匹配行用 LEFT JOIN，否则 INNER JOIN 更快。

### 5.10.5 计划缓存

优化好的计划会缓存复用，默认 **1800 秒过期**（`ignite.planner.planCacheExpiresAfterSeconds`）。数据分布大变后可手动失效（实验性 API）：

```bash
sql planner invalidate-cache --tables=PUBLIC.Person
```

## 5.11 本章小结

- SQL 是 Ignite 3 的一等公民接口：Calcite 引擎 + 全事务保证 + 分布式 JOIN；
- 写路径优先批处理：`executeScript`/`executeBatch`/DataStreamer/`COPY INTO`；
- 类型系统注意三个坑：CHAR 不能进 DDL、TIMESTAMP 只到毫秒、隐式转换不跨族；
- DDL 变更遵循"只放大不缩小"；索引构建等存量事务，DROP 对象等 LWM；
- 分布区是数据放置的总开关：分区数、副本数、节点过滤、扩缩容延迟、一致性模式、quorum；
- 调优三板斧：EXPLAIN PLAN 看算子、EXPLAIN MAPPING 看分布、相关子查询一律改 JOIN。

---

# 第六章　API 参考导览

官方 API Reference 部分是方法级文档，本章给出结构化导览，帮助你在需要时快速定位。完整方法签名请对照官方站点与 Javadoc。

## 6.1 Java 客户端 API

入口均为 `IgniteClient` 的子系统门面（subsystem facade）：

| 门面 | 获取方式 | 覆盖内容 |
| --- | --- | --- |
| SQL API | `client.sql()` | 查询执行、PreparedStatement（Statement 配置）、ResultSet 处理、DML/DDL、批操作、类型化结果映射（`execute(..., Class<T>)`）、异步执行、查询取消、事务集成、SqlRow 按名/按序号访问 |
| Tables API | `client.tables()` | 表发现（`table()/tables()`）、RecordView/KeyValueView（Tuple 与 POJO 两种）、批量操作、条件操作（replace/deleteExact 等乐观并发）、异步操作、分区信息、Tuple 构造与取值 |
| Criteria API | `view.query(...)` | 比较/NULL 检查/IN/AND/OR/复杂谓词、类型化 criteria、KV 视图查询、范围查询、游标迭代、事务集成 |
| Transactions API | `client.transactions()` | 隐式事务（`runInTransaction`）、显式 `begin()/commit()`、只读事务、超时、跨表事务、异步事务闭包、SQL 集成、事务状态、错误处理与异常类型 |
| Compute API | `client.compute()` | 作业实现（ComputeJob）、提交与 JobTarget（anyNode/node/colocated）、作业上下文（`JobExecutionContext` 可拿到本地表的 RecordView）、取消与 token、优先级、状态查询、广播执行与结果收集、部署单元、自定义序列化、MapReduceTask、错误处理 |
| Data Streamer API | `view.streamData(...)` | 基础流、StreamOptions、操作类型（upsert/remove）、自定义 publisher、receiver 流、错误处理、auto-flush、KV 视图流、性能注意事项 |
| Network API | `client.cluster()` | 集群节点访问、异步节点发现、节点标识与地址（`NetworkAddress` 解析/格式化）、节点元数据、客户端连接信息、计算目标节点选择 |
| Catalog API | `client.catalog()` | 注解建表、KV 表定义、列配置、复合主键与排序、索引定义、共置配置、zone 配置、流式 `TableDefinition/ZoneDefinition` builder、条件创建、表/zone 删除、schema 名 |
| Security API | `BasicAuthenticator` 等 | 认证配置、认证类型（`AuthenticationType` 解析）、失败处理、TLS 组合、自定义 `IgniteClientAuthenticator`、凭据管理、嵌入式节点认证、最佳实践 |
| Server API（嵌入式） | `IgniteServer` | 节点初始化、资源访问、表/SQL/事务/计算/Catalog 操作、集群信息、生命周期管理 |

此外官方还提供 **Java API Reference（Javadoc）** 在线版，与上述分页互补。

## 6.2 .NET 客户端 API

`IIgniteClient` 体系，结构与 Java 对齐：

- **Client API**：`IgniteClientConfiguration`、`IgniteClient.StartAsync`、连接弹性与重试；
- **SQL API**：`client.Sql.ExecuteAsync` 系列，结果集 `IIgniteResultSet<T>`，支持取消令牌；
- **Tables API**：`ITable`、`IRecordView<T>`/`IKeyValueView<K,V>`、Tuple（`IIgniteTuple`）与 POJO；
- **Transactions API**：`ITransactions`，`RunInTransactionAsync` 等；
- **Compute API**：`ICompute`，`SubmitAsync`/`ExecuteAsync`/`ExecuteColocatedAsync`/广播；
- **Data Streamer API**：`DataStreamer.StreamDataAsync`，`DataStreamerOptions`；
- **Network API**：`ICluster`、`IClusterNode`；
- **ADO.NET API**：`IgniteDbConnection/IgniteDbCommand/IgniteDbDataReader`——走 ADO.NET 习惯的连接串、参数化命令、事务；
- **LINQ API**：表视图可直接作为 `IQueryable` 使用，支持 Where/Select/OrderBy/GroupBy/Join/聚合等标准算子子集，表达式树翻译为 SQL 下推执行。

## 6.3 C++ 客户端 API

`ignite::IgniteClient` 体系，同样与 Java 对齐：

- **Client API**：`IgniteClientConfiguration`、`IgniteClient::Start`、配置选项、错误处理、连接管理；
- **SQL API**：`execute`/`execute_script`、结果集、事务集成、查询取消；
- **Tables API**：`get_tables()`、`record_view`/`key_value_view`、`ignite_tuple`、异步操作；
- **Transactions API**：`transactions().begin()`、表与 SQL 操作、事务选项（只读/超时）、可见性、异步；
- **Compute API**：`compute().submit/execute`、JobTarget、JobDescriptor、作业控制、广播；
- **Network API**：`cluster_node`、节点发现与生命周期、与 Compute 配合选择目标节点。

## 6.4 SQL-only API

三条纯 SQL 通道，细节见第三章 3.13–3.15：

- **JDBC Driver**：`org.apache.ignite.jdbc.IgniteJdbcDriver`，连接串 `jdbc:ignite:thin://host:10800`；
- **ODBC Driver**：ODBC 3.8，Windows/Linux 安装包，支持 pyodbc；
- **Python DB-API**：`pip install pyignite3_dbapi`，PEP-0249 风格。

## 6.5 选型建议

- Java 服务：瘦客户端 + Table API（热路径 KV）+ SQL API（复杂查询）；
- Spring 应用：Spring Boot Starter（瘦客户端）或 Spring Data（JDBC 方言）；
- BI/报表/即席查询：JDBC 或 ODBC；
- 数据科学/脚本：Python DB-API；
- 极致批量写入：DataStreamer（各语言均有）；
- 服务器内嵌场景（测试、单机工具）：嵌入式 Server API。

---

# 第七章　工具：CLI、REST API 与术语表

## 7.1 CLI 工具（ignite3）

CLI 是管理 Ignite 3 的主力工具，两种使用模式：

- **交互模式（REPL）**：`bin/ignite3` 启动后 `connect http://localhost:10300`，支持补全与历史；
- **非交互模式**：`bin/ignite3 cluster config show` 直接执行单条命令，适合脚本化；
- `--verbose` 获取详细输出；CLI 自身日志可配置。

### 7.1.1 命令全景

**连接与杂项**：`connect`（`--username/--password` 带认证）、`disconnect`、`clear`/`cls`、`exit`、`help`、`version`。

**SQL**：

```bash
sql "SELECT * FROM Person"                     # 直接执行 SQL
sql planner invalidate-cache --tables=PUBLIC.Person   # 失效计划缓存
```

**CLI 自身配置（profile 机制）**：

```bash
cli config profile create myProfile
cli config profile activate myProfile
cli config profile list / show
cli config get cli.trust-store.path
cli config set cli.trust-store.password=secret
cli config show / remove
```

**集群管理**：

```bash
cluster init --name=myCluster --cluster-management-group=node1,node2,node3
cluster status
cluster topology physical / logical
cluster config show [路径]
cluster config update <HOCON> | --file conf.conf
cluster unit deploy <unitId> --version 1.0.0 --path ./jobs.jar --nodes MAJORITY
cluster unit undeploy <unitId> --version 1.0.0
cluster unit list
cluster metric source enable|disable|list <source>
```

**节点管理**：

```bash
node status
node version
node config show [路径]
node config update [-n node1] <HOCON>
node metric list
node metric source enable|disable|list <source>
node unit list
```

**灾难恢复**：

```bash
recovery partitions states [--global] [--zone ...] [--partition ...]
recovery partitions restart [--with-cleanup] --zone ... --partition ...
recovery partitions reset --zone ... --partition ...
recovery cluster reset --url=<node-url> --cluster-management-group=<nodes>
recovery cluster migrate --old-cluster-url=<url> --new-cluster-url=<url>
```

**分布/zone**：`distribution reset`、`zone datanodes reset`——强制 zone 的数据节点分配立即重置（用于卡住的分配异常）。

### 7.1.2 使用提示

- 多数命令都有 `--url`（指定节点 REST 地址）或 `-n`（节点名）参数；
- 认证集群上先 `connect --username ... --password ...`，后续命令沿用会话；
- 脚本化时建议用 profile 管理多套环境（dev/test/prod）的连接与凭据。

## 7.2 REST API

集群暴露 **OpenAPI** 规范的 REST 接口，默认端口 10300（`ignite.rest` 可改），可用于监控与管理：

```bash
# 集群状态
curl 'http://localhost:10300/management/v1/cluster/state'

# 创建快照
curl -H "Content-Type: application/json" \
     -d '{"snapshotType":"FULL","tableNames":"table1,table2","startTimeEpochMilli":0}' \
     http://localhost:10300/management/v1/snapshot/create

# 部署单元
curl -X POST --data-binary @jobs.jar \
     "http://localhost:10300/management/v1/deployment/units/my-unit/1.0.0"
```

CLI 能做的事 REST 基本都能做（配置读写、初始化、拓扑、部署、恢复等）——CLI 本身就是 REST 的封装。要在 Java 应用里集成，推荐用 **OpenAPI Generator** 生成类型化客户端：

```java
ApiClient client = Configuration.getDefaultApiClient();
client.setBasePath("http://localhost:10300");
ClusterConfigurationApi api = new ClusterConfigurationApi(client);
String configuration = api.getClusterConfiguration();
```

启用 SSL 后 REST 走 10400（HTTPS）；开启认证后请求需带基础认证头。

## 7.3 术语表（Glossary）

| 术语 | 含义 |
| --- | --- |
| Cluster management group（CMG） | Raft 组形式的集群管理组，leader 负责节点的加入与离开 |
| Metastorage group | 存储集群权威元数据（catalog、拓扑历史等）的 Raft 组 |
| Cluster node | 集群节点，计算与存储的基本单元 |
| Coordinator node | 接收计算作业、管理执行并向客户端汇报结果的节点 |
| Distribution zone | 分布区：控制数据如何分区、分区如何落在节点上，属集群配置，用 SQL 管理 |
| Data colocation | 数据共置：相关数据放在同一节点 |
| Data region | 数据区：控制存储可用内存量的逻辑区域 |
| Data rebalance | 再平衡：在节点间重新均匀分布分区的过程 |
| Logical topology | 逻辑拓扑：经 CMG 验证、组成集群的节点集合 |
| Physical topology | 物理拓扑：互相发现连通的节点集合，未必都属于集群 |
| Persistent Storage | 持久化存储：无论集群状态如何都保留数据的存储 |
| Primary Storage | 主存储：数据写入与常规读取的目标库 |
| Snapshot | 快照：某时点数据的一致副本（REST 可创建） |
| Storage engine / Storage profile | 存储引擎（aimem/aipersist/rocksdb）与其参数化命名实例（profile） |
| Low watermark（LWM） | 低水位线：GC 清理 MVCC 旧版本的时间阈值（默认 600s） |
| Lease / Primary replica | 租约/主副本：Placement Driver 授予的分区写入口 |
| Version storage | 版本存储：MVCC 保存行多版本的区域 |

---

# 第八章　迁移指南

## 8.1 从 Ignite 2 迁移

Ignite 2 与 Ignite 3 是**架构重写级**的差异，代码不能直接复用，但多数概念可对应，迁移工作量可控。

### 8.1.1 配置迁移（XML → HOCON）

Ignite 2 的 XML bean 配置在 Ignite 3 中变为 HOCON（或 JSON），且结构重新划分——拆为**集群配置、节点配置、分布区配置**三层：

```hocon
ignite {
  network {
    nodeFinder { netClusterNodes = ["localhost:3344"] }
    port = 3344
  }
  storage {
    profiles = [ { name = persistent, engine = aipersist } ]
  }
  nodeAttributes.nodeAttributes = { region = "US", storage = "SSD" }
}
```

存储配置的概念映射：

| Ignite 2 | Ignite 3 |
| --- | --- |
| Data region + 持久化开关 | 存储引擎（aimem/aipersist/rocksdb）+ 存储 profile |
| Cache + Affinity function | 表 + 分布区（zone）+ COLOCATE BY |
| Cache store（外部存储） | 通过 SQL 配置的 cache storage |
| 胖客户端/瘦客户端之分 | **全部是瘦客户端**，统一 `clientConnector` 配置 |
| 事件 SPI | 事件通道（channel，定义收集什么）+ sink（定义发到哪，当前仅 log） |
| 指标 SPI（默认开启） | 指标 source（**默认关闭**，按 source 启用）+ exporter（JMX/log/OTLP） |

注意：存储 profile 与引擎只能通过节点配置 + 重启来定义；只有表和分布区能在代码/SQL 中创建。网络、REST 等原 XML 中的 bean 现在都在节点配置的对应段落里。

### 8.1.2 代码迁移要点

- `IgniteCache<K,V>` → `KeyValueView<K,V>` 或 `RecordView<T>`；
- `Ignite#cache()` → `IgniteClient#tables().table()`；
- ScanQuery → criteria 查询或 SQL；
- `IgniteCompute` → `ComputeJob` + `JobTarget`，任务类打包进部署单元；
- 事务 API 语义类似（`runInTransaction` 更省心）；
- `SET STREAMING ON` → `executeScript` 自动攒批 / DataStreamer。

## 8.2 从 Ignite 3.0 迁移到 3.1

> ⚠️ **3.1 引入了 zone-based replication 架构变化，无法原地滚动升级**——必须新建 3.1 集群、导出导入数据，**需要停机窗口**。

### 8.2.1 架构变化：表级复制 → zone 级复制

| 方面 | 3.0 表级 | 3.1 zone 级 |
| --- | --- | --- |
| Raft 组 | 每张表独立创建 | 同 zone 内的表共享 |
| 100 张表的集群 | 100 套 Raft 组 | 1 套共享 Raft 组 |
| 内存占用 | 随表数升高 | 显著降低 |
| 线程开销 | 高 | 低 |

对用户透明：API 与查询行为不变，无需改代码（个别废弃 API 除外，如 `ignite.clusterNodes()` → `ignite.cluster().nodes()`）。表级复制在 3.1 仍兼容，但后续版本将移除。

配置格式也有调整，常见的两处：

| 类型 | 3.0 | 3.1 |
| --- | --- | --- |
| 超时属性 | `timeout=5000` | `timeoutMillis=5000` |
| 建 zone | `CREATE ZONE z WITH STORAGE_PROFILES='default', REPLICAS=3` | `CREATE ZONE z (REPLICAS 3) STORAGE PROFILES['default']` |

### 8.2.2 六阶段迁移流程

**阶段 1：记录现状**（连接 3.0 集群，CLI 进 `sql` 模式）

```sql
SELECT * FROM SYSTEM.SCHEMAS;                 -- 所有 schema
SELECT SCHEMA_NAME, TABLE_NAME FROM SYSTEM.TABLES WHERE TABLE_TYPE='TABLE';
SELECT SCHEMA_NAME, TABLE_NAME, COLUMN_NAME, TYPE, NULLABLE, COLUMN_DEFAULT
  FROM SYSTEM.TABLE_COLUMNS ORDER BY TABLE_NAME, ORDINAL_POSITION;
SELECT * FROM SYSTEM.ZONES;                   -- 分布区配置
SELECT COUNT(*) FROM your_table;              -- 每表行数（迁移后核对用）
```

把完整 CREATE TABLE 语句（含约束、索引、表选项）保存为 `schema-recreation.sql`。

**阶段 2：从 3.0 导出数据**

推荐 **Parquet**（压缩、IO 快，适合大数据量）；CSV 可读性好，适合中小数据量与排错：

```sql
-- Parquet 导出（推荐）
COPY FROM analytics.events (id, event_time, user_id, event_type, payload)
INTO '/backup/ignite-3.0-export/analytics_events.parquet' FORMAT PARQUET;

-- CSV 导出（带表头）；大表可按范围分片导出
COPY FROM (SELECT * FROM analytics.events WHERE id BETWEEN 0 AND 1000000)
INTO '/backup/.../events_part1.csv' FORMAT CSV WITH 'header'='true';
```

官方迁移指南提供了 `export-all-tables.sh` 自动化脚本模式（循环表清单 → 导出 → 同时保存行数到 `.count` 文件）。**导出未全部验证前不要进入下一阶段**；停机后也不要删除 3.0 的数据。

**阶段 3：搭建 3.1 集群**

下载 3.1 → 按新格式调整配置 → 逐节点启动 → `cluster init --name=ignite-cluster` → `cluster topology` 确认全部 ACTIVE → 重建 schema、zone、表（结构必须与 3.0 **完全一致**，否则导入失败）。

**阶段 4：导入数据**

```sql
COPY FROM '/backup/ignite-3.0-export/analytics_events.parquet'
INTO analytics.events (id, event_time, user_id, event_type, payload)
FORMAT PARQUET WITH 'batchSize'='2048';
```

配套的 `import-all-tables.sh` 模式：逐表导入后立即比对行数（导出时存的 `.count`），不一致即中止。**行数核对之外再做抽样**：抽查若干行、NOT NULL 列无 NULL、日期范围（MIN/MAX）保持。

**阶段 5：切换客户端**

更新连接地址；检查废弃 API；先用连通性测试程序验证（连集群、读一条样本数据），再逐步切流量。

**阶段 6：迁移后验证**

确认 zone 级复制已生效：`grep "Zone based replication" logs/*.log` 应输出 `Zone based replication: true`；`SELECT * FROM SYSTEM.ZONES` 确认 zone 配置正确。全部验证通过前**不要做应用正式切换**。

## 8.3 迁移检查清单

- [ ] 已保存全部 schema/表/zone 定义与行数基线
- [ ] 已选定导出格式（大数据量用 Parquet）
- [ ] 导出文件全部生成且非 0 字节
- [ ] 3.1 集群初始化成功、拓扑完整
- [ ] 表结构与 3.0 完全一致（含索引与约束）
- [ ] 导入后行数逐表核对通过、抽样检查通过
- [ ] 客户端连接测试通过、废弃 API 已替换
- [ ] 3.0 数据保留至 3.1 验证全部完成

---

# 结语与参考资源

至此，本文覆盖了 Ignite 3.1.0 官方文档的全部主要内容：入门 → 核心概念与架构 → 开发（客户端/Table API/事务/流/计算/部署/事件/连接与集成）→ 安装配置运维 → SQL 详解 → API 参考 → 工具 → 迁移。

**推荐学习路径**：
1. 先跑通第一章 Quick Start，建立手感；
2. 第二章理解核心概念（分区、共置、事务、Raft）——这是用好 Ignite 的关键；
3. 按技术栈选读第三章的客户端部分，动手写增删改查与一个计算作业；
4. 需要上生产时精读第四章（配置/安全/监控/灾备）与第五章的调优部分；
5. 第六、七章作为日常速查手册。

**官方资源**：

- 文档总站：https://ignite.apache.org/docs/ignite3/3.1.0
- 下载：https://ignite.apache.org/download.cgi
- 3.1.0 Release Notes：https://ignite.apache.org/releases/3.1.0/release_notes.html
- GitHub：https://github.com/apache/ignite
- Slack 社区：https://ignite.apache.org/community/resources.html

---

> 本文档基于 https://ignite.apache.org/docs/ignite3/3.1.0 全部文档页面（约 155 页）整理编写，内容版权归 Apache 软件基金会所有，采用 Apache License 2.0。本文档仅供学习参考，细节以官方文档为准。
