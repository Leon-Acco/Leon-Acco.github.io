---
layout: post
title: "Flink_SQL_系统化学习文档"
date: 2026-09-24
categories: [bigdata]
series: flink
series_order: 2
mermaid: false
math: false
toc: true
---
# Flink SQL 系统化学习文档（模块化精讲 + 业务案例库）

> 版本基线：Flink 1.19 / 1.20（1.x 末期语法，对齐 2.0）；涉及 1.15 的差异会标注。
> 使用方式：按篇顺序学习，每个模块的 SQL 都可以在 SQL Client 里直接跑（基于 datagen/kafka/jdbc 等 connector）。

## 总目录

**Part I 基础篇**
- 模块 1：Flink SQL 体系认知与环境
- 模块 2：DDL 完全指南
- 模块 3：数据类型与内置函数
- 模块 4：基础查询语法（含 CTE / UNION / CASE）

**Part II 核心语义篇**
- 模块 5：时间属性与 Watermark
- 模块 6：窗口 Window TVF（滚动/滑动/累积/会话）
- 模块 7：聚合（Group / Distinct / Over）
- 模块 8：Join 四种形态（Regular / Interval / Lookup / Temporal）
- 模块 9：去重与 TopN
- 模块 10：模式识别 MATCH_RECOGNIZE（CEP）

**Part III 工程实战篇**
- 模块 11：Connector 实战（Kafka / JDBC / ES / Filesystem / Paimon）
- 模块 12：CDC 与 Changelog 语义
- 模块 13：UDF 体系（Scalar / Table / Aggregate / Async）
- 模块 14：格式处理（JSON / Protobuf / 二进制自定义协议）
- 模块 15：状态、TTL 与性能调优
- 模块 16：Checkpoint、Savepoint 与作业运维

**Part IV 业务案例库（12 个典型场景）**
- 案例 1~12：实时大屏、订单超时、漏斗分析、UV 去重、会话分析、风控规则、实时入湖、双流打宽、TopN 榜单、迟到数据、多流合并、二进制报文

---

# Part I 基础篇

## 模块 1：Flink SQL 体系认知与环境

### 1.1 体系定位

Flink SQL = ANSI SQL 语法 + 流处理语义扩展，底层跑在和 DataStream API 相同的 runtime 上：

```
SQL → Calcite 优化器 → 物理计划 → Transformation → JobGraph → 集群
```

**关键认知：动态表（Dynamic Table）**

- 流就是一张不断变化的表；连续查询的结果也是动态表。
- 结果流的三种编码决定下游 connector 选型：

| 编码 | 含义 | 典型来源 | 可写入的 Sink |
|------|------|----------|--------------|
| Append-only | 只有 INSERT | 纯过滤/映射/窗口聚合（一行一次） | 任意（kafka、filesystem…） |
| Retract | INSERT + DELETE | group by 非窗口聚合、regular join | upsert-kafka、jdbc、ES、Paimon |
| Upsert | UPSERT + DELETE（需主键） | 带主键的聚合/join | upsert-kafka、jdbc、ES、Paimon |

> 判断口诀：**结果每行只发一次 → append；会更新旧结果 → retract/upsert，sink 必须支持主键更新**。

### 1.2 环境（最小可用）

```bash
# docker-compose 起 Flink 集群后
./bin/sql-client.sh          # 交互式
./bin/sql-client.sh -f job.sql   # 脚本模式（生产提交）
```

SQL Client 三种结果显示模式：`SET 'sql-client.execution.result-mode' = 'tableau' / 'table' / 'changelog';`

### 1.3 模块案例：第一个作业（造数 → 打印）

```sql
CREATE TABLE gen (
  id INT, name STRING,
  ts AS PROCTIME()    -- 处理时间列
) WITH (
  'connector' = 'datagen',
  'rows-per-second' = '5',
  'fields.id.min' = '1', 'fields.id.max' = '100',
  'fields.name.length' = '5'
);

SELECT * FROM gen WHERE id > 50;
```

**要点**：`datagen` connector 是学习期的万能数据源；`PROCTIME()` 生成处理时间计算列。

---

## 模块 2：DDL 完全指南

### 2.1 语法总览

```sql
CREATE TABLE   -- 映射外部系统（source/sink 同一张表，无区别）
CREATE VIEW    -- 逻辑封装，不落数据，可嵌套
CREATE FUNCTION  -- UDF 注册
CREATE CATALOG / DATABASE  -- 元数据组织
ALTER / DROP / SHOW / DESCRIBE
```

### 2.2 CREATE TABLE 三段式

```sql
CREATE TABLE orders (
  order_id    BIGINT,
  user_id     BIGINT,
  amount      DECIMAL(10,2),
  status      STRING,
  order_time  TIMESTAMP(3),
  -- ① 计算列：表达式从其他列派生
  order_date  AS CAST(order_time AS DATE),
  -- ② 元数据列：读取 connector 内部信息
  kafka_ts    TIMESTAMP_LTZ(3) METADATA FROM 'timestamp' VIRTUAL,
  -- ③ watermark 声明
  WATERMARK FOR order_time AS order_time - INTERVAL '5' SECOND,
  -- ④ 主键（仅约束语义，不强制校验，upsert 需要）
  PRIMARY KEY (order_id) NOT ENFORCED
) WITH (
  'connector' = 'kafka',
  'topic' = 'orders',
  'properties.bootstrap.servers' = 'kafka:9092',
  'scan.startup.mode' = 'earliest-offset',
  'format' = 'json',
  'json.ignore-parse-errors' = 'true'   -- 脏数据容错
);
```

四类列：**物理列**（来自数据）、**计算列**（`AS expr`，不落盘不占存储）、**元数据列**（`METADATA FROM`，如 Kafka 的 topic/partition/offset/timestamp/headers）、**主键**（NOT ENFORCED）。

### 2.3 LIKE 继承与 CTAS

```sql
-- 复用已有表定义（叠加/覆盖 WITH 参数）
CREATE TABLE orders_v2 (WATERMARK FOR order_time AS order_time) 
LIKE orders (INCLUDING ALL EXCLUDING CONSTRAINTS);

-- CTAS：建表+灌数据一体（1.18 起支持原子性，依赖 connector）
CREATE TABLE order_backup WITH ('connector'='filesystem','path'='s3://b/orders','format'='parquet')
AS SELECT * FROM orders;
```

### 2.4 Catalog 体系

```sql
CREATE CATALOG paimon WITH ('type'='paimon', 'warehouse'='hdfs:///warehouse');
USE CATALOG paimon;
CREATE DATABASE ods;
-- 完整路径：catalog.database.table
SELECT * FROM paimon.ods.orders;
```

### 2.5 模块案例：埋点日志表（元数据 + 计算列实战）

```sql
CREATE TABLE click_log (
  user_id   BIGINT,
  page      STRING,
  action    STRING,
  log_ts    BIGINT,
  -- 计算列：毫秒转时间戳
  event_ts  AS TO_TIMESTAMP_LTZ(log_ts, 3),
  -- 元数据列：kafka 分区，排查数据倾斜时用
  partition_id INT METADATA FROM 'partition' VIRTUAL,
  WATERMARK FOR event_ts AS event_ts - INTERVAL '10' SECOND
) WITH (
  'connector'='kafka', 'topic'='click_log',
  'properties.bootstrap.servers'='kafka:9092',
  'scan.startup.mode'='earliest-offset', 'format'='json'
);

-- 检查各分区数据量（排查倾斜）
SELECT partition_id, COUNT(*) FROM click_log GROUP BY partition_id;
```

**常见坑**：计算列不可写（作为 sink 表时会报错，需 `EXCLUDING`）；元数据列加了 `VIRTUAL` 就不出现在 `SELECT *` 里。

---

## 模块 3：数据类型与内置函数

### 3.1 数据类型速查

| 类别 | 类型 | 备注 |
|------|------|------|
| 字符 | STRING / VARCHAR(n) | Flink 里 STRING 最常用 |
| 数值 | INT / BIGINT / DOUBLE / DECIMAL(p,s) | 金额用 DECIMAL |
| 时间 | DATE / TIME / TIMESTAMP(3) / TIMESTAMP_LTZ(3) | **带时区用 _LTZ**，跨系统推荐 |
| 布尔/二进制 | BOOLEAN / BYTES / VARBINARY | BYTES 对应裸字节（二进制协议场景） |
| 复合 | ARRAY<T> / MAP<K,V> / ROW<...> | 嵌套结构 |
| 其他 | INTERVAL、NULL | — |

转换函数：`CAST(x AS T)`（严格，失败报错）、`TRY_CAST`（1.15+，失败返回 NULL，清洗必备）。

### 3.2 高频内置函数分类记忆

```sql
-- 字符串
CONCAT(a,b) || SUBSTR(s,1,3) || UPPER/LOWER || TRIM || REGEXP_EXTRACT(s,'(\d+)',1)
-- 时间
DATE_FORMAT(ts,'yyyy-MM-dd HH:mm') || TO_TIMESTAMP_LTZ(bigint,3) || UNIX_TIMESTAMP()
-- 条件
COALESCE(a,b,'default') || NULLIF || CASE WHEN ... THEN ... END
-- 集合
ARRAY[1,2,3] || MAP['k',1] || element[1] || CARDINALITY(arr)
-- JSON（1.15+ 全量支持）
JSON_VALUE(j,'$.a') || JSON_QUERY(j,'$.arr') || JSON_EXISTS(j,'$.a') || JSON_OBJECT(KEY 'k' VALUE v)
```

### 3.3 模块案例：订单金额清洗

```sql
SELECT
  order_id,
  TRY_CAST(amount_str AS DECIMAL(10,2))          AS amount,        -- 脏数据→NULL
  COALESCE(TRY_CAST(amount_str AS DECIMAL(10,2)), 0) AS amount_safe,
  REGEXP_EXTRACT(page_url, '/product/(\d+)', 1)  AS product_id,   -- URL 提取商品ID
  DATE_FORMAT(order_time, 'yyyy-MM-dd')          AS dt
FROM raw_orders;
```

---

## 模块 4：基础查询语法

### 4.1 要点

- `WHERE / GROUP BY / HAVING / ORDER BY / LIMIT`：流模式下 `ORDER BY` 必须带时间属性列，否则状态无限增长；`LIMIT` 流模式慎用。
- **CTE（WITH 子句）**：复杂 SQL 分层，可读性必备。
- **UNION / UNION ALL / INTERSECT / EXCEPT**：流模式全部支持（EXCEPT/INTERSECT 有状态，注意 TTL）。
- **LATERAL TABLE + 炸裂函数**：一行转多行。

### 4.2 模块案例：订单打标 + 分层统计（CTE 实战）

```sql
WITH labeled AS (
  SELECT order_id, user_id, amount,
    CASE WHEN amount >= 1000 THEN 'big'
         WHEN amount >= 100  THEN 'mid'
         ELSE 'small' END AS level
  FROM orders
),
stat AS (
  SELECT level, COUNT(*) AS cnt, SUM(amount) AS total
  FROM labeled GROUP BY level
)
SELECT * FROM stat ORDER BY total DESC;
```

### 4.3 模块案例：JSON 数组炸裂（LATERAL TABLE）

```sql
-- 数据：{"order_id":1, "items":[{"sku":"a","num":2},{"sku":"b","num":1}]}
CREATE TABLE order_items_raw (
  order_id BIGINT,
  items STRING   -- json 数组
) WITH ('connector'='kafka', ...);

SELECT order_id, sku, num
FROM order_items_raw,
LATERAL TABLE(JSON_TUPLE... ) -- 老式
-- 现代写法：先用 JSON_QUERY 取出数组，再 CROSS JOIN UNNEST（2.0 起原生支持 UNNEST；1.x 用 TableFunction）
;
```

> 1.x 炸裂数组的实用方案：自研 TableFunction（模块 13 有完整代码），或用 `json format` 直接定义 `items ARRAY<ROW<sku STRING, num INT>>` 让 connector 自动反序列化，然后自定义 UDTF 炸开。

---

# Part II 核心语义篇

## 模块 5：时间属性与 Watermark

### 5.1 三种时间

| 时间 | 定义 | 使用场景 |
|------|------|----------|
| Processing Time | 机器时钟 | 对延迟敏感、不在乎乱序 |
| Event Time | 数据自带 | **生产默认**，可重放可恢复 |
| Ingestion Time | 进 Flink 时刻 | 折中，少用 |

### 5.2 Watermark 声明三板斧

```sql
-- ① 时间列已是 TIMESTAMP 类型
WATERMARK FOR ts AS ts - INTERVAL '5' SECOND

-- ② 数据是毫秒 BIGINT → 先计算列转换
event_ts AS TO_TIMESTAMP_LTZ(log_ms, 3),
WATERMARK FOR event_ts AS event_ts - INTERVAL '5' SECOND

-- ③ 严格递增场景
WATERMARK FOR ts AS ts
```

### 5.3 生产必知的三件事

**空闲分区/空闲 source**（窗口不触发的头号原因）：某 Kafka 分区没数据 → 该分区 watermark 不推进 → 整体 watermark 卡住。

   ```sql
   -- DDL WITH 参数（1.18+ 也可用 hint）
   'table.exec.source.idle-timeout' = '30s'   -- 全局配置
   -- 或 SQL hint（1.18+）
   SELECT /*+ OPTIONS('source.idle-timeout'='30s') */ * FROM kafka_src
   ```

2. **多源 watermark 对齐**（快慢源差距过大撑爆状态）：

   ```sql
   SELECT /*+ OPTIONS('watermark-align.group'='g1',
                     'watermark-align.max-drift'='1m',
                     'watermark-align.update-interval'='10s') */
   FROM source_a;
   ```

3. **迟到数据**：窗口默认 watermark 过了就触发销毁，迟到数据丢弃。处理方案见案例库「案例 10」。

### 5.4 模块案例：乱序日志的 watermark 验证

```sql
CREATE TABLE events (
  id BIGINT,
  event_ts AS TO_TIMESTAMP_LTZ(log_ms, 3),
  WATERMARK FOR event_ts AS event_ts - INTERVAL '10' SECOND
) WITH ('connector'='kafka', ...);

-- 观察 watermark 是否推进：当前窗口最大事件时间
SELECT MAX(event_ts), COUNT(*) FROM events;
-- 同时观察 Web UI → Task 的 Watermark 指标
```

---

## 模块 6：窗口 Window TVF

> 1.19 起老的 `TUMBLE(...)` Group Window 语法已弃用，**统一使用 Window TVF**（表值函数写法）。

### 6.1 四种窗口语法骨架

```sql
-- 滚动窗口：无重叠，每 1 分钟一个
SELECT window_start, window_end, SUM(amount)
FROM TABLE(TUMBLE(TABLE orders, DESCRIPTOR(order_time), INTERVAL '1' MINUTE))
GROUP BY window_start, window_end;

-- 滑动窗口：窗口 10 分钟，每 1 分钟滑一次（数据会重复计算）
FROM TABLE(HOP(TABLE orders, DESCRIPTOR(order_time),
               INTERVAL '1' MINUTE, INTERVAL '10' MINUTE))

-- 累积窗口：1 天大窗，每 10 分钟出一次截至当前的累计值（实时大屏神器）
FROM TABLE(CUMULATE(TABLE orders, DESCRIPTOR(order_time),
                    INTERVAL '10' MINUTE, INTERVAL '1' DAY))

-- 会话窗口：间隔 5 分钟无数据则断开（1.19 起流模式支持）
FROM TABLE(SESSION(TABLE events, DESCRIPTOR(event_ts), INTERVAL '5' MINUTE))
```

### 6.2 Window TopN 与 Window Deduplication（1.15+ 隐藏神器）

```sql
-- 每个窗口内取销售额前 3 的商品
SELECT * FROM (
  SELECT *, ROW_NUMBER() OVER (
    PARTITION BY window_start, window_end ORDER BY gmv DESC) AS rn
  FROM (
    SELECT window_start, window_end, sku, SUM(amount) AS gmv
    FROM TABLE(TUMBLE(TABLE orders, DESCRIPTOR(order_time), INTERVAL '5' MINUTE))
    GROUP BY window_start, window_end, sku
  )
) WHERE rn <= 3;
```

### 6.3 模块案例：实时 GMV 大屏（累积窗口 + upsert 输出）

```sql
CREATE TABLE gmv_board (
  window_start TIMESTAMP(3), window_end TIMESTAMP(3),
  gmv DECIMAL(18,2), order_cnt BIGINT,
  PRIMARY KEY (window_start) NOT ENFORCED
) WITH ('connector'='jdbc', 'url'='jdbc:mysql://...', 'table-name'='gmv_board', ...);

INSERT INTO gmv_board
SELECT window_start, window_end,
       SUM(amount) AS gmv, COUNT(*) AS order_cnt
FROM TABLE(CUMULATE(TABLE orders, DESCRIPTOR(order_time),
                    INTERVAL '1' MINUTE, INTERVAL '1' DAY))
GROUP BY window_start, window_end;
```

**要点**：CUMULATE 每分钟输出"今天截至当前的累计值"，jdbc upsert 不停覆盖同一行——大屏前端直接 `SELECT` 最新值即可，无需前端自己累加。

**常见坑**：窗口聚合输出是 append 流（每个 key+window 最终一条），但要写 jdbc 需要主键；window_start 通常作为主键一部分。

---

## 模块 7：聚合

### 7.1 三种聚合

| 类型 | 写法 | 输出流 | 状态 |
|------|------|--------|------|
| Group Agg | `GROUP BY k` | retract/upsert（结果会变） | **无限增长，必须 TTL** |
| Window Agg | Window TVF | append | 窗口结束即清理 |
| Over Agg | `OVER (PARTITION BY k ORDER BY ts)` | append，每行都出结果 | 按时间范围存 |

### 7.2 模块案例：三种聚合对比（同一份订单数据）

```sql
-- ① 分组聚合：每个用户的累计消费（结果不断更新 → upsert 流）
SELECT user_id, SUM(amount) AS total FROM orders GROUP BY user_id;

-- ② 窗口聚合：每分钟总销售额（窗口结束出最终结果 → append 流）
SELECT window_start, SUM(amount) FROM TABLE(
  TUMBLE(TABLE orders, DESCRIPTOR(order_time), INTERVAL '1' MINUTE))
GROUP BY window_start, window_end;

-- ③ Over 聚合：每条订单带上"该用户前 10 分钟的累计金额"（逐行输出）
SELECT order_id, user_id, amount,
  SUM(amount) OVER (
    PARTITION BY user_id ORDER BY order_time
    RANGE BETWEEN INTERVAL '10' MINUTE PRECEDING AND CURRENT ROW
  ) AS amount_10min
FROM orders;
```

### 7.3 DISTINCT 聚合与倾斜

```sql
-- 近 1 小时 UV
SELECT COUNT(DISTINCT user_id) FROM TABLE(
  CUMULATE(TABLE click_log, DESCRIPTOR(event_ts), INTERVAL '5' MINUTE, INTERVAL '1' HOUR))
GROUP BY window_start, window_end;

-- 倾斜时开 SPLIT（把 distinct 桶化两阶段聚合）
SET 'table.optimizer.distinct-agg.split.enabled' = 'true';
```

**常见坑**：非窗口的 `GROUP BY`（如 ①）状态会随 key 无限增长，生产上必须配 TTL（模块 15）。

---

## 模块 8：Join 四种形态

### 8.1 总览表（背下来）

| Join | 语法 | 状态开销 | 语义 | 典型场景 |
|------|------|----------|------|----------|
| Regular | `a JOIN b ON a.k=b.k` | 双边全量存 | 任意时刻匹配 | 双流关联（慎用，TTL 必配） |
| Interval | `ON a.k=b.k AND a.ts BETWEEN b.ts ± x` | 只存时间窗 | 时间有界匹配 | 曝光→点击归因 |
| Lookup | `JOIN dim FOR SYSTEM_TIME AS OF proc_time` | 无状态（查外部） | 查当前维表 | MySQL/HBase 维表 |
| Temporal | `JOIN dim FOR SYSTEM_TIME AS OF event_ts` | 维表侧存版本 | 查历史时刻维度 | 汇率/价格回溯 |

### 8.2 模块案例①：Interval Join —— 订单与支付 15 分钟匹配

```sql
SELECT o.order_id, o.amount, p.pay_time
FROM orders o
JOIN payments p
  ON o.order_id = p.order_id
 AND p.pay_time BETWEEN o.order_time AND o.order_time + INTERVAL '15' MINUTE;
```

**要点**：状态只保留 15 分钟窗口内的数据，TTL 由时间边界自动推导——比 regular join 安全得多。**能用 interval join 就不要用 regular join**。

### 8.3 模块案例②：Lookup Join + 三件套 —— 订单关联用户维表

```sql
CREATE TABLE dim_user (
  user_id BIGINT, city STRING, vip_level INT,
  PRIMARY KEY (user_id) NOT ENFORCED
) WITH (
  'connector'='jdbc', 'url'='jdbc:mysql://...', 'table-name'='dim_user', ...,
  -- 三件套：缓存 + 异步 + 重试
  'lookup.cache'='PARTIAL',
  'lookup.partial-cache.max-rows'='50000',
  'lookup.partial-cache.expire-after-write'='10min'
);

SELECT /*+ LOOKUP('table'='dim_user',
                  'output-mode'='allow_unordered',
                  'retry-strategy'='fixed_delay',
                  'fixed-delay'='1s', 'max-attempts'='3') */
  o.order_id, u.city, u.vip_level
FROM orders o
JOIN dim_user FOR SYSTEM_TIME AS OF PROCTIME() AS u
  ON o.user_id = u.user_id;
```

### 8.4 模块案例③：Temporal Join —— 按下单时刻的汇率折算

```sql
-- 汇率表：CDC changelog 流（带主键+更新时间）
CREATE TABLE rates (
  currency STRING, rate DECIMAL(10,4), update_ts TIMESTAMP(3),
  WATERMARK FOR update_ts AS update_ts,
  PRIMARY KEY (currency) NOT ENFORCED
) WITH ('connector'='mysql-cdc', ...);

SELECT o.order_id, o.amount * r.rate AS amount_cny
FROM orders o
JOIN rates FOR SYSTEM_TIME AS OF o.order_time AS r
  ON o.currency = r.currency;
```

### 8.5 模块案例④：Regular Join（万不得已时用）

```sql
-- 必须配 TTL！1.19+ 用 hint 精细控制两边不同 TTL
SELECT /*+ STATE_TTL('clicks'='1h', 'orders'='24h') */
  c.click_id, o.order_id
FROM clicks c JOIN orders o ON c.user_id = o.user_id;
```

**常见坑**：regular join 输出是 retract 流，写 kafka 需 `upsert-kafka` 或带主键的 sink；join 两边 key 倾斜 → 配置 `table.exec.skew-optimization` 或拆分 hot key。

---

## 模块 9：去重与 TopN

### 9.1 两种去重

```sql
-- ① 普通去重（流式 DISTINCT，状态存所有见过的 key）
SELECT DISTINCT user_id FROM click_log;

-- ② ROW_NUMBER 保留第一行/最后一行（常用：CDC 数据按主键取最新）
SELECT order_id, status FROM (
  SELECT *, ROW_NUMBER() OVER (PARTITION BY order_id ORDER BY update_ts DESC) AS rn
  FROM order_cdc
) WHERE rn = 1;
```

### 9.2 TopN（无窗口，持续输出榜单）

```sql
-- 实时热销商品 TOP 10（retract 流，写 upsert sink）
SELECT * FROM (
  SELECT sku, gmv,
         ROW_NUMBER() OVER (ORDER BY gmv DESC) AS rn
  FROM (SELECT sku, SUM(amount) AS gmv FROM orders GROUP BY sku)
) WHERE rn <= 10;
```

### 9.3 模块案例：日志按 trace_id 取最新状态

```sql
-- 同一 trace 多次上报状态，保留最新一条
CREATE VIEW latest_span AS
SELECT trace_id, span_id, status, event_ts
FROM (
  SELECT *, ROW_NUMBER() OVER (
    PARTITION BY trace_id, span_id ORDER BY event_ts DESC) AS rn
  FROM trace_log
) WHERE rn = 1;
```

**要点**：`ROW_NUMBER` TopN/去重底层有专门优化（状态按 key 淘汰 + rank 变化才输出）；配 TTL 防状态膨胀（默认按第一行进入起算，1.15+ 行为已优化）。

---

## 模块 10：模式识别 MATCH_RECOGNIZE（CEP）

### 10.1 语法骨架

```sql
SELECT ... FROM events
MATCH_RECOGNIZE (
  PARTITION BY device_id          -- 分组
  ORDER BY event_ts               -- 事件时间排序（必须）
  MEASURES                        -- 输出哪些列
    FIRST(A.event_ts) AS start_ts,
    LAST(B.event_ts)  AS end_ts
  ONE ROW PER MATCH               -- 或 ALL ROWS PER MATCH
  AFTER MATCH SKIP PAST LAST ROW  -- 匹配后跳到哪里继续
  PATTERN (A B+ C)                -- 模式：A 开头，B 至少一次，C 结尾
  WITHIN INTERVAL '10' MINUTE     -- 整体时限（1.15+）
  DEFINE                          -- 各模式变量条件
    A AS A.status = 'login_fail',
    B AS B.status = 'login_fail',
    C AS C.status = 'login_succ'
) AS m;
```

模式量词：`A{3}` 恰好3次、`A+` 一次以上、`A?` 零或一、`A{2,5}` 范围、`(A|B)` 或、`^A` 开头。

### 10.2 模块案例：账户盗用检测（5 分钟内 3 次失败登录后成功）

```sql
CREATE TABLE login_alert (
  user_id BIGINT, first_fail TIMESTAMP_LTZ(3), succ_ts TIMESTAMP_LTZ(3)
) WITH ('connector'='kafka', 'topic'='login_alert', ...);

INSERT INTO login_alert
SELECT user_id, first_fail, succ_ts
FROM login_log
MATCH_RECOGNIZE (
  PARTITION BY user_id
  ORDER BY event_ts
  MEASURES
    FIRST(F.event_ts) AS first_fail,
    S.event_ts AS succ_ts
  ONE ROW PER MATCH
  AFTER MATCH SKIP PAST LAST ROW
  PATTERN (F{3,} S)
  WITHIN INTERVAL '5' MINUTE
  DEFINE
    F AS F.result = 'fail',
    S AS S.result = 'success'
);
```

---

# Part III 工程实战篇

## 模块 11：Connector 实战

### 11.1 选型速查

| Connector | 角色 | 流编码 | 关键 WITH 参数 |
|-----------|------|--------|----------------|
| kafka | source+sink | append/retract | topic、scan.startup.mode、format |
| upsert-kafka | source+sink | upsert（需主键） | 自动 compact |
| jdbc | sink+维表 | upsert | 必须 PRIMARY KEY；lookup 三件套 |
| elasticsearch | sink | upsert | index、flush 间隔 |
| filesystem | source+sink | append | path、format（parquet/orc）、分区提交 |
| mysql-cdc | source | changelog | tables、server-id、scan.startup.mode |
| paimon | source+sink | upsert/changelog | 湖仓一体，1.17+ 主推 |
| datagen | source | append | 学习/压测造数 |
| blackhole | sink | — | 性能压测 |

### 11.2 模块案例：Kafka JSON 源 + 幂等写 ES

```sql
CREATE TABLE docs (
  doc_id STRING, title STRING, content STRING, ts TIMESTAMP(3),
  PRIMARY KEY (doc_id) NOT ENFORCED
) WITH ('connector'='kafka','topic'='docs',...,'format'='json');

CREATE TABLE es_sink (
  doc_id STRING, title STRING, content STRING,
  PRIMARY KEY (doc_id) NOT ENFORCED
) WITH (
  'connector'='elasticsearch-7',
  'hosts'='http://es:9200', 'index'='docs',
  'sink.bulk-flush.max-size'='1000',
  'sink.bulk-flush.interval'='2s'
);

INSERT INTO es_sink SELECT doc_id, title, content FROM docs;
-- ES 以 doc_id 为 _id 覆盖写 → 天然幂等，故障恢复无重复
```

### 11.3 模块案例：filesystem 分区写 + Hive 分区自动提交

```sql
CREATE TABLE log_archive (
  user_id BIGINT, page STRING, event_ts TIMESTAMP(3),
  dt STRING, hh STRING
) PARTITIONED BY (dt, hh)
WITH (
  'connector'='filesystem',
  'path'='s3://warehouse/click_log',
  'format'='parquet',
  'partition.time-extractor.timestamp-pattern'='$dt $hh:00:00',
  'sink.partition-commit.trigger'='partition-time',
  'sink.partition-commit.delay'='1 min',
  'sink.partition-commit.policy.kind'='metastore,success-file'  -- 提交 Hive 分区+_SUCCESS
);
```

---

## 模块 12：CDC 与 Changelog 语义

### 12.1 概念

CDC connector 把数据库 binlog 翻译成 changelog 流（+I/-U/+U/-D 四种消息）。Flink SQL 原生理解这些消息——聚合、join 都能正确处理更新。

### 12.2 模块案例：MySQL 订单表实时同步到 Kafka（upsert）

```sql
CREATE TABLE orders_cdc (
  order_id BIGINT, user_id BIGINT, amount DECIMAL(10,2), status STRING,
  update_ts TIMESTAMP(3),
  PRIMARY KEY (order_id) NOT ENFORCED
) WITH (
  'connector'='mysql-cdc',
  'hostname'='mysql','port'='3306',
  'username'='flink','password'='xxx',
  'database-name'='shop','table-name'='orders',
  'server-id'='5400-5404'
);

CREATE TABLE orders_kafka (
  order_id BIGINT, user_id BIGINT, amount DECIMAL(10,2), status STRING,
  PRIMARY KEY (order_id) NOT ENFORCED
) WITH ('connector'='upsert-kafka','topic'='orders_upsert',...);

INSERT INTO orders_kafka SELECT * FROM orders_cdc;
```

### 12.3 模块案例：CDC 维表做 Temporal Join（版本维度）

```sql
-- 商品价格会调整，按"下单时刻"的价格算金额
CREATE TABLE product_cdc (
  sku STRING, price DECIMAL(10,2), update_ts TIMESTAMP(3),
  WATERMARK FOR update_ts AS update_ts,
  PRIMARY KEY (sku) NOT ENFORCED
) WITH ('connector'='mysql-cdc', ..., 'table-name'='product');

SELECT o.order_id, o.qty * p.price AS amount
FROM orders o
JOIN product_cdc FOR SYSTEM_TIME AS OF o.order_time AS p
  ON o.sku = p.sku;
```

**常见坑**：mysql-cdc 的 `server-id` 多作业必须不同（或给范围）；Temporal join 要求维表流有 watermark 和主键。

---

## 模块 13：UDF 体系

### 13.1 四种 UDF 对比

| 类型 | 基类 | 输入→输出 | 典型用途 |
|------|------|-----------|----------|
| 标量函数 | ScalarFunction | 1行→1值 | 解析、加解密、格式转换 |
| 表函数 | TableFunction | 1行→N行 | 数组炸裂、分词 |
| 聚合函数 | AggregateFunction | N行→1值 | 自定义统计（如分位数） |
| 异步标量 | AsyncScalarFunction（1.19+） | 1行→1值（非阻塞） | 调外部服务 |

### 13.2 模块案例①：ScalarFunction —— 手机号脱敏

```java
public class MaskPhone extends ScalarFunction {
    public String eval(String phone) {
        if (phone == null || phone.length() != 11) return phone;
        return phone.substring(0, 3) + "****" + phone.substring(7);
    }
}
```

```sql
CREATE FUNCTION mask_phone AS 'com.example.MaskPhone';
SELECT user_id, mask_phone(phone) FROM users;
```

### 13.3 模块案例②：TableFunction —— 炸开逗号分隔标签

```java
@FunctionHint(output = @DataTypeHint("ROW<tag STRING>"))
public class SplitTags extends TableFunction<Row> {
    public void eval(String tags) {
        if (tags == null) return;
        for (String t : tags.split(",")) collect(Row.of(t));
    }
}
```

```sql
CREATE FUNCTION split_tags AS 'com.example.SplitTags';
SELECT user_id, tag
FROM users, LATERAL TABLE(split_tags(tags)) AS T(tag);
```

### 13.4 模块案例③：AggregateFunction —— 加权平均

```java
public class WeightedAvg extends AggregateFunction<Double, WeightedAvg.Acc> {
    public static class Acc { public double sum; public long cnt; }
    @Override public Acc createAccumulator() { return new Acc(); }
    @Override public Double getValue(Acc acc) {
        return acc.cnt == 0 ? null : acc.sum / acc.cnt;
    }
    public void accumulate(Acc acc, Double value, Long weight) {
        if (value != null && weight != null) { acc.sum += value * weight; acc.cnt += weight; }
    }
    public void retract(Acc acc, Double value, Long weight) {
        if (value != null && weight != null) { acc.sum -= value * weight; acc.cnt -= weight; }
    }
    public void merge(Acc acc, Iterable<Acc> it) {
        for (Acc o : it) { acc.sum += o.sum; acc.cnt += o.cnt; }
    }
}
```

**要点**：实现 `retract` 才能用于会产生撤回的聚合（如 group agg 含更新）；`merge` 用于会话窗口/批优化。

### 13.5 模块案例④：AsyncScalarFunction（1.19+）—— 调用风控接口

```java
public class RiskCheck extends AsyncScalarFunction {
    private transient HttpClient client;
    @Override public void open(FunctionContext ctx) { client = HttpClient.newHttpClient(); }
    public void eval(CompletableFuture<String> out, String userId) {
        client.sendAsync(buildReq(userId), ofString())
              .thenAccept(r -> out.complete(parseRiskLevel(r.body())))
              .exceptionally(e -> { out.complete("UNKNOWN"); return null; });
    }
}
```

**要点**：SQL 里调用外部服务**必须**用异步 UDF（1.19 之前只能同步阻塞拖垮吞吐）；保序输出。

---

## 模块 14：格式处理

### 14.1 format 一览

`json` / `csv` / `avro` / `parquet` / `orc` / `protobuf` / `debezium-json` / `canal-json` / `maxwell-json` / `raw` / `ogg-json`

### 14.2 模块案例①：嵌套 JSON 直接映射复合类型

```json
{"user":{"id":1001,"name":"tom"},"items":[{"sku":"a","num":2}],"ts":1727000000000}
```

```sql
CREATE TABLE nested_orders (
  user  ROW<id BIGINT, name STRING>,
  items ARRAY<ROW<sku STRING, num INT>>,
  ts    BIGINT,
  event_ts AS TO_TIMESTAMP_LTZ(ts, 3),
  WATERMARK FOR event_ts AS event_ts - INTERVAL '5' SECOND
) WITH ('connector'='kafka', ..., 'format'='json');

-- 直接点出来用
SELECT user.id, user.name, items[1].sku FROM nested_orders;
```

**要点**：schema 固定时让 connector 反序列化比 `JSON_VALUE` 快得多（不用每条重复解析）。

### 14.3 模块案例②：schema 不固定的 JSON → JSON_VALUE 兜底

```sql
CREATE TABLE flex_events (
  payload STRING  -- 整段 json 存一列
) WITH ('connector'='kafka', ..., 'format'='raw', 'raw.charset'='UTF-8');

SELECT
  JSON_VALUE(payload, '$.type') AS event_type,
  COALESCE(
    TRY_CAST(JSON_VALUE(payload, '$.amount') AS DECIMAL(10,2)),
    TRY_CAST(JSON_VALUE(payload, '$.money') AS DECIMAL(10,2))  -- 字段名漂移兜底
  ) AS amount
FROM flex_events
WHERE JSON_EXISTS(payload, '$.type');
```

### 14.4 模块案例③：自定义二进制协议

参考配套文档《Flink_SQL_学习文档_二进制报文实战.md》：`format='raw'` 读 BYTES → ScalarFunction UDF 解码（CRC 校验）→ JSON_VALUE 或直接 `@DataTypeHint("ROW<...>")` 返回结构化行（高性能方案）。

---

## 模块 15：状态、TTL 与性能调优

### 15.1 TTL 三级体系（演进史）

```sql
-- L1 全局（1.15 时代，一把梭，粗）
SET 'table.exec.state.ttl' = '1h';

-- L2 算子级（1.18+，compiled plan 指定单个算子）
-- L3 hint 级（1.19+，推荐）
SELECT /*+ STATE_TTL('a'='1h', 'b'='24h') */ ... FROM a JOIN b ON ...
```

### 15.2 聚合/Join 调优决策树

```
GROUP BY 慢/倾斜？
├─ key 本身倾斜（少数 key 占大头）
│   ├─ distinct → table.optimizer.distinct-agg.split.enabled=true
│   └─ 普通 agg → 两阶段聚合 AUTO + 手动加盐拆分（见案例库11变体）
├─ 状态访问频繁 → MiniBatch：
│   table.exec.mini-batch.enabled=true
│   table.exec.mini-batch.allow-latency=5s
│   table.exec.mini-batch.size=5000
└─ join 记录放大（级联 join 中间结果爆炸）→ MiniBatch Regular Join（1.19+ 自动支持）

窗口聚合 CPU 高？
├─ 开 local-global（1.15+ 优化器 AUTO 大多自动开）
└─ 检查 watermark 是否过慢导致状态滞留
```

### 15.3 Checkpoint 与状态后端（承接版本演进文档）

```yaml
state.backend.type: rocksdb
state.backend.changelog.enabled: true              # 1.16 生产可用，checkpoint 数据量降 ~200x
execution.checkpointing.interval: 1min
execution.checkpointing.aligned-checkpoint-timeout: 30s   # 背压自动切 unaligned
execution.checkpointing.file-merging.enabled: true        # 1.20 小文件合并
```

### 15.4 模块案例：倾斜订单表的 distinct UV 优化前后

```sql
-- 优化前：COUNT(DISTINCT) 单个 key 热点，TM 之间负载差 10 倍
SELECT dt, COUNT(DISTINCT buyer_id) FROM orders GROUP BY dt;

-- 优化后
SET 'table.optimizer.distinct-agg.split.enabled' = 'true';
SET 'table.exec.mini-batch.enabled' = 'true';
SET 'table.exec.mini-batch.allow-latency' = '5s';
-- 同一 SQL，观察 Web UI 各 subtask 的 recordsReceived 趋于均衡
```

---

## 模块 16：Checkpoint、Savepoint 与作业运维

### 16.1 操作命令

```bash
# 触发 checkpoint（1.19+ CLI 支持）
./bin/flink checkpoint $JOB_ID [-full]

# savepoint 停止/恢复（升级、迁移必用）
./bin/flink stop --savepointPath /sp $JOB_ID
./bin/flink run -s /sp/xxx -c ... job.jar
./bin/sql-client.sh 提交的作业同样走 savepoint 恢复

# 运行中调并行度（1.18+，Web UI 或 REST）
curl -X PATCH .../jobs/$JOB_ID -d '{"parallelism": 8}'
```

### 16.2 升级 SQL 作业的正确姿势

1. savepoint 停止旧作业；
2. 同 SQL 提交新版本（1.16 起官方保证 SQL 作业升级兼容性，但算子结构变化的改写可能破坏）；
3. 预发环境先验证 `-s savepoint` 恢复；
4. 灰度观察 watermark/checkpoint/延迟指标。

### 16.3 模块案例：排障三板斧

```sql
-- ① 数据进没进来？
SELECT COUNT(*), MAX(event_ts) FROM src;
-- ② watermark 卡在哪？看 Web UI watermark 指标 + 空闲分区
-- ③ 慢在哪？1.19+ Web UI 直接创建 async-profiler 火焰图，看热点函数
```

---

# Part IV 业务案例库

> 每个案例：业务背景 → 数据模型 → 完整 SQL → 要点解析。所有案例可独立改造成你的业务。

## 案例 1：实时大屏 GMV（累积窗口 + upsert）

**背景**：双11 大屏，每分钟刷新"今日累计 GMV / 订单数 / 客单价"。

```sql
CREATE TABLE orders (... WATERMARK FOR order_time AS order_time - INTERVAL '5' SECOND)
WITH ('connector'='kafka','topic'='orders',...);

CREATE TABLE board (
  stat_date STRING, gmv DECIMAL(18,2), order_cnt BIGINT, avg_amount DECIMAL(18,2),
  PRIMARY KEY (stat_date) NOT ENFORCED
) WITH ('connector'='jdbc','url'='jdbc:mysql://.../dashboard','table-name'='board',...);

INSERT INTO board
SELECT
  DATE_FORMAT(window_start,'yyyy-MM-dd') AS stat_date,
  SUM(amount), COUNT(*), SUM(amount)/COUNT(*)
FROM TABLE(CUMULATE(TABLE orders, DESCRIPTOR(order_time),
                    INTERVAL '1' MINUTE, INTERVAL '1' DAY))
GROUP BY window_start, window_end;
```

**要点**：CUMULATE 输出"当天截至当前的累计"，jdbc upsert 按 stat_date 覆盖——大屏只查一行。

## 案例 2：订单支付超时告警（Interval Join 反向用法）

**背景**：下单 15 分钟未支付 → 告警催付。

```sql
-- 思路：LEFT JOIN + IS NULL 找出时间窗内没匹配的订单
INSERT INTO unpaid_alert
SELECT o.order_id, o.user_id, o.order_time
FROM orders o
LEFT JOIN payments p
  ON o.order_id = p.order_id
 AND p.pay_time BETWEEN o.order_time AND o.order_time + INTERVAL '15' MINUTE
WHERE p.order_id IS NULL;
-- 该写法输出在窗口结束后确认未支付；需要"到点即告警"语义可用 CEP（案例3变体）
```

**要点**：interval join 的状态自动按 15 分钟清理，是"超时检测"类需求的标准 SQL 模式。

## 案例 3：登录风控 CEP（连续失败后成功）

见模块 10.2 完整代码。**变体**：`PATTERN (F{5,})`（连续5次失败即锁号告警，不管后续）；`WITHIN` 控制时间窗；`AFTER MATCH SKIP TO NEXT F` 做滑动检测。

## 案例 4：电商转化漏斗（多步骤模式）

**背景**：浏览 → 加购 → 下单 → 支付，30 分钟内完成四步视为完整转化。

```sql
SELECT user_id, view_ts, pay_ts
FROM user_behavior
MATCH_RECOGNIZE (
  PARTITION BY user_id
  ORDER BY event_ts
  MEASURES V.event_ts AS view_ts, P.event_ts AS pay_ts
  ONE ROW PER MATCH
  AFTER MATCH SKIP PAST LAST ROW
  PATTERN (V A O P)
  WITHIN INTERVAL '30' MINUTE
  DEFINE
    V AS action = 'view',
    A AS action = 'cart',
    O AS action = 'order',
    P AS action = 'pay'
);
-- 再对输出做 COUNT 与 UV 统计即得漏斗各级转化率
```

## 案例 5：实时 UV 去重（两级方案）

```sql
-- 方案A：COUNT DISTINCT（精确，状态大）
SELECT window_start, COUNT(DISTINCT user_id)
FROM TABLE(TUMBLE(TABLE click_log, DESCRIPTOR(event_ts), INTERVAL '5' MINUTE))
GROUP BY window_start, window_end;

-- 方案B：先 ROW_NUMBER 去重再 COUNT（状态等价，语义更清晰可控）
SELECT window_start, COUNT(*) AS uv
FROM TABLE(TUMBLE(
  TABLE (SELECT user_id, event_ts FROM (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY event_ts) rn
    FROM click_log) WHERE rn = 1),
  DESCRIPTOR(event_ts), INTERVAL '5' MINUTE))
GROUP BY window_start, window_end;
```

## 案例 6：用户会话分析（Session Window）

**背景**：App 埋点，5 分钟无操作视为会话结束，统计每个会话的时长、页面数、入口页。

```sql
SELECT user_id, window_start AS session_start, window_end AS session_end,
       TIMESTAMPDIFF(SECOND, window_start, window_end) AS duration_sec,
       COUNT(*) AS page_cnt,
       MIN_BY(page, event_ts) AS entry_page     -- 1.15+ 支持 MIN_BY/MAX_BY
FROM TABLE(SESSION(TABLE app_events, DESCRIPTOR(event_ts), INTERVAL '5' MINUTE))
GROUP BY user_id, window_start, window_end;
```

## 案例 7：实时风控（Lookup Join 实时名单 + 规则过滤）

```sql
-- 黑名单维表（运营后台实时维护）
CREATE TABLE blacklist (
  user_id BIGINT, reason STRING, PRIMARY KEY (user_id) NOT ENFORCED
) WITH ('connector'='jdbc', ..., 'lookup.cache'='PARTIAL',
        'lookup.partial-cache.expire-after-write'='1min');

INSERT INTO risk_orders
SELECT /*+ LOOKUP('table'='blacklist') */
  o.*, b.reason
FROM orders o
JOIN blacklist FOR SYSTEM_TIME AS OF PROCTIME() AS b
  ON o.user_id = b.user_id
WHERE o.amount > 5000;   -- 大额+黑名单 → 拦截流
```

## 案例 8：MySQL → Paimon 实时入湖（CDC 管道）

```sql
CREATE CATALOG paimon WITH ('type'='paimon','warehouse'='hdfs:///warehouse');
USE CATALOG paimon;

CREATE TABLE ods_orders (
  order_id BIGINT, user_id BIGINT, amount DECIMAL(10,2), status STRING,
  update_ts TIMESTAMP(3),
  PRIMARY KEY (order_id) NOT ENFORCED
) WITH (
  'connector'='mysql-cdc', 'hostname'='mysql', 'database-name'='shop',
  'table-name'='orders', 'username'='flink', 'password'='xxx', 'server-id'='5501'
);

-- Paimon 目标表（主键表，自动处理 changelog）
CREATE TABLE dwd_orders (
  order_id BIGINT, user_id BIGINT, amount DECIMAL(10,2), status STRING,
  dt STRING,
  PRIMARY KEY (order_id) NOT ENFORCED
) PARTITIONED BY (dt) WITH ('changelog-producer'='input');

INSERT INTO dwd_orders
SELECT order_id, user_id, amount, status,
       DATE_FORMAT(update_ts,'yyyy-MM-dd')
FROM ods_orders;
```

**要点**：Paimon 主键表天然消费 changelog；下游可以用 Flink 批 SQL 直接查湖表做 T+0 报表；1.20 起可用 Materialized Table 把 ODS→DWD→DWS 管道全部托管。

## 案例 9：双流打宽（订单流 + 物流流，Interval Join）

```sql
CREATE TABLE order_wide (
  order_id BIGINT, amount DECIMAL(10,2),
  order_time TIMESTAMP(3), ship_time TIMESTAMP(3),
  lead_hours DOUBLE,
  PRIMARY KEY (order_id) NOT ENFORCED
) WITH ('connector'='upsert-kafka','topic'='order_wide',...);

INSERT INTO order_wide
SELECT o.order_id, o.amount, o.order_time, s.ship_time,
       TIMESTAMPDIFF(HOUR, o.order_time, s.ship_time) AS lead_hours
FROM orders o
JOIN shipments s
  ON o.order_id = s.order_id
 AND s.ship_time BETWEEN o.order_time AND o.order_time + INTERVAL '7' DAY;
```

## 案例 10：迟到数据处理（两级窗口）

**背景**：1 分钟窗口出快报，迟到的数据用 1 小时窗口出终报，下游合并。

```sql
-- 快报：正常 watermark（5s 容忍）
INSERT INTO metric_fast
SELECT window_start, device_id, SUM(val)
FROM TABLE(TUMBLE(TABLE telemetry, DESCRIPTOR(event_ts), INTERVAL '1' MINUTE))
GROUP BY window_start, window_end, device_id;

-- 终报：另一张源表 watermark 放宽到 1 小时（或批模式重算）
-- 下游报表系统：先展示快报，终报覆盖订正
```

**要点**：SQL 层没有 DataStream 的 allowedLateness + side output 直译，工程上就是"两条不同 watermark 容忍度的链路"。

## 案例 11：多流合并 + 分流（UNION ALL + STATEMENT SET）

```sql
-- 合并：App/小程序/H5 三端日志统一口径
CREATE VIEW all_events AS
SELECT 'app' AS src, * FROM app_log
UNION ALL SELECT 'h5', * FROM h5_log
UNION ALL SELECT 'mini', * FROM mini_log;

-- 分流：一次读取三路写出
BEGIN STATEMENT SET;
INSERT INTO sink_error  SELECT * FROM all_events WHERE level = 'ERROR';
INSERT INTO sink_biz    SELECT * FROM all_events WHERE level = 'INFO' AND type='biz';
INSERT INTO sink_others SELECT * FROM all_events WHERE level NOT IN ('ERROR','INFO') OR type<>'biz';
END;
```

## 案例 12：二进制报文解析全链路（UDF + JSON + 聚合 + 分流 + 存储）

**本案例为独立完整文档**：自定义二进制协议 → `format='raw'` 读 BYTES → `DecodeTelemetry` UDF（含 CRC 校验的完整 Java 代码）→ JSON_VALUE 抽取 → 三路分流 → 1 分钟窗口聚合写 MySQL → lookup 维表写 ES → CEP 高温告警 → Paimon 入湖 → Materialized Table 托管。

详见配套文件《Flink_SQL_学习文档_二进制报文实战.md》第四、五章。

---

## 附录 A：高频配置速查

```yaml
# config.yaml（1.19+ 格式）
execution.checkpointing.interval: 1min
execution.checkpointing.aligned-checkpoint-timeout: 30s
state.backend.type: rocksdb
state.backend.changelog.enabled: true
table.exec.mini-batch.enabled: true
table.exec.mini-batch.allow-latency: 5s
table.exec.source.idle-timeout: 30s
table.optimizer.distinct-agg.split.enabled: true
```

## 附录 B：模块学习顺序与自检问题

| 顺序 | 模块 | 自检问题 |
|------|------|----------|
| 1 | 模块 1-4 | 计算列和物理列区别？retract 流能写普通 kafka 吗？ |
| 2 | 模块 5-6 | 窗口不触发怎么排查？CUMULATE 和 HOP 区别？ |
| 3 | 模块 7-9 | 非窗口 GROUP BY 为什么必须 TTL？Window TopN 怎么写？ |
| 4 | 模块 8 | 四种 join 各存多少状态？什么时候必须 temporal join？ |
| 5 | 模块 10 | PATTERN 量词怎么组合？WITHIN 的作用？ |
| 6 | 模块 11-14 | 维表缓存怎么配？JSON 固定 schema 与动态 schema 两种路线？ |
| 7 | 模块 15-16 | 倾斜怎么治？升级作业的标准流程？ |
| 8 | 案例库 | 能独立改造 3 个以上案例到自己业务吗？ |

---

*文档整理时间：2026-09。语法基线 Flink 1.19/1.20；1.15 差异已在文中标注。配合《Flink 1.15-1.20 版本演进学习文档》《Flink SQL 二进制报文实战》三件套学习效果最佳。*
