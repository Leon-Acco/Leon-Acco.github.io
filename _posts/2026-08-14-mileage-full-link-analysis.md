---
layout: post
title: "车辆里程计算与查询接口 — 全链路分析"
date: 2026-08-14
categories: [iot]
series: mileage
series_order: 3
mermaid: false
math: false
toc: true
---
> **接口**: `POST /vehicle/mileage/totalMileage`（dst-v2x-manager-service，端口 18522）
> **Flink 计算**: `dst-iot-car-calculator`（⚠️ 不是 `dst-iot-vehicle-mileage-calculate-job`，后者是空壳）
> **分析日期**: 2026-08-14

## 1. 结论（TL;DR）

`/totalMileage` 是**查 Doris 小时里程汇总表**的读接口。**Flink 只负责把"累计里程原始值"过滤后落库；"小时窗口的 min/max/里程增量"全靠 Doris SQL 聚合（XXL-Job 每小时定时任务）得到**，Flink 源码里的窗口聚合类并未在线上主链路启用。

```
车载终端(32960协议)
   │ 上报: 0x02实时 / 0x03补发(含累计里程 mileage_total)
   ▼
32960网关 → Kafka(topic, VIN塞在消息header的"vin"键)
   │
   ▼
Flink: VehicleMileageCalculatorJob (dst-iot-car-calculator)
   ├─ 解析32960 | 只留0x02/0x03 | 里程[>0, ≤maxMileage] | 时间[30天前, 未来1天]
   ├─ keyBy(uid) → MileageFilterFunction(固定阈值 + 中位数MAD + 设备复位识别)
   └─ Doris Sink ──────────► vehicle_mileage_data（原始表, DUPLICATE KEY, 月动态分区）
                               │
                               │ 每小时: XXL-Job VehicleMileageSummaryRefreshJob
                               │         = INSERT..SELECT 过去7天按小时聚合
                               │ 或手动: POST /vehicle/mileage/refresh
                               ▼
                    vehicle_mileage_summary_hourly（小时汇总, UNIQUE KEY + 写时合并）
                               │
                               ▼
        POST /vehicle/mileage/totalMileage
        ├─ 校验: 结束时间 ≤ 当前时间
        ├─ Doris: GROUP BY vin,uid → MIN/MAX(min/max_mileage), SUM(mileage)
        └─ Service层: 按 vin+startHour+endHour 去重, 保留 totalMileage 更大的那条
```

## 2. 接口调用链（逐层定位）

| 层级 | 文件 | 职责 |
|---|---|---|
| Controller | `module/business/mileage/controller/VehicleTotalMileageBusinessController.java` | `@PostMapping("/totalMileage")`，`@Validated` 收参 |
| 业务 Service | `module/business/mileage/service/VehicleTotalMileageBusinessService.java` | 校验"结束时间≤当前时间"后委托 |
| Doris Service | `infrastructure/biz/doris/mileage/service/MvVehicleHourlyMileageServiceImpl.java` | 查库 + 内存去重 |
| Doris Mapper | `infrastructure/biz/doris/mileage/mapper/MvVehicleHourlyMileageMapper.java` + XML | 执行聚合 SQL |

**请求参数**（`VehicleTotalMileageQuery`）：
- `vins`：车架号列表，**≤200 个**（`@Size(max=200)`）
- `startTime` / `endTime`：`yyyy-MM-dd HH:mm:ss`，开始/结束时间

**返回**（`VehicleTotalMileageRange`，List）：`uid` / `vin` / `minMileage` / `maxMileage` / `totalMileage` / `startHour` / `endHour`

### 2.1 核心查询 SQL（`MvVehicleHourlyMileageMapper.xml`）

```sql
SELECT vin, uid,
       MIN(min_mileage)  AS min_mileage,
       MAX(max_mileage)  AS max_mileage,
       ROUND(SUM(mileage), 1) AS total_mileage,
       MIN(window_hour)  AS start_hour,
       MAX(window_hour)  AS end_hour
FROM vehicle_mileage_summary_hourly
WHERE vin IN (...) AND window_hour BETWEEN #{startTime} AND #{endTime}
GROUP BY vin, uid
```

### 2.2 Service 层去重逻辑（容易忽略）

`MvVehicleHourlyMileageServiceImpl` 拿到 SQL 结果后，按 **`vin + startHour + endHour`** 分组，同一组内**保留 `totalMileage` 更大的那条**。这是对旧 `DUPLICATE KEY` 表结构下"刷新任务重复插入产生多行"的兜底（详见 §5 风险点）。

<div class="callout callout-note" markdown="1">
**📌 语义要点**

接口名叫"总里程"，实际返回的是**查询时间窗口内**的行驶里程和 min/max 范围，**不是车辆终身累计里程**。
`totalMileage = SUM(hourly.mileage)`，而 `hourly.mileage = MAX(mileage_total) - MIN(mileage_total)`（小时窗口内累计值的最大最小差）。终端只上报累计里程（odometer），单点值不可信，必须用"窗口内最大最小差"还原真实行驶量。
</div>

## 3. 涉及的表（都在 Doris，不是 MySQL）

### 3.1 `vehicle_mileage_data` — 原始累计里程表（Flink 写入）

- 模型：`DUPLICATE KEY(uid, data_time)`，按月动态分区，只保留近约 12 个月

| 字段 | 含义 |
|---|---|
| `uid` | 设备唯一编码（32960 报文头终端 ID） |
| `data_time` | 上报时间 |
| `vin` | 车架号（**Flink 从 Kafka header 取，不在报文体内**） |
| `cmd` | 0x02 实时 / 0x03 补发 |
| `mileage_total` | 累计总里程（BIGINT，km） |

### 3.2 `vehicle_mileage_summary_hourly` — 小时汇总表（接口直接查的表）

| 字段 | 含义 |
|---|---|
| `uid` / `vin` | 设备 / 车架 |
| `window_hour` | 小时窗口（`data_time` 截断到 `%Y-%m-%d %H:00:00`） |
| `min_mileage` / `max_mileage` | 该小时内累计里程的最小/最大值 |
| `mileage` | 小时行驶增量 = `max - min` |

**表结构演进（重要）**：
- 老版本（`dst-iot-car-calculator` 内 SQL）：`DUPLICATE KEY` + BUCKETS 16
- 新版本（manager-service `sql/02_zhangchao_202603171600_doris_dst_v2x_db.sql`）：**`UNIQUE KEY(uid, vin, window_hour)` + `enable_unique_key_merge_on_write=true` + BUCKETS 64 + 3 副本 + zstd 压缩**，分区 p2023~p2028

<div class="callout callout-warning" markdown="1">
**⚠️ 为什么必须升级 UNIQUE KEY**

刷新任务用 `INSERT`（非 `INSERT OVERWRITE`）每小时跑一次，在 DUPLICATE KEY 下会**重复插入**同一窗口的多行 → 接口 SUM(mileage) 会翻倍，Service 层去重就是为此兜底。UNIQUE KEY + 写时合并后重复执行幂等，旧去重逻辑变成冗余保险。
</div>

## 4. Flink 计算任务（真正实现：`dst-iot-car-calculator`）

> ⚠️ **项目定位陷阱**：用户给的 `D:\code\flink_calculator\dst-iot-vehicle-mileage-calculate-job` **只有 `@SpringBootApplication` 启动类，无任何 Job 逻辑**（空壳）。真正的计算在 **`D:\code\flink_calculator\dst-iot-car-calculator`**，主类 `VehicleMileageCalculatorJob`。

### 4.1 主作业流水线

```
Kafka(32960原始二进制)
  → VehicleRecordDeserializer（协议解析, 用 protocol-32960-dst 库）
  → filter(mileage>0 且 ≤ maxMileage, Nacos配置)
  → keyBy(uid)
  → MileageFilterFunction（里程异常检测, 按uid维护Flink状态）
  → Doris Sink → vehicle_mileage_data
```

### 4.2 协议解析（`VehicleRecordDeserializer.java`）

- 用 `protocol-32960-dst` 库解析报文头，**只放行 `cmd=0x02` 实时、`cmd=0x03` 补发**
- 解析 02 数据包累计里程（`translationKilometer()`），非法值（≤0 或 ≥999999）丢弃
- **时间窗过滤**：只保留"过去 30 天 ~ 未来 1 天"的数据
- **VIN 来自 Kafka 消息 header 的 `vin` key**（上游网关投递时塞入）

### 4.3 里程异常检测（`MileageFilterFunction.java`）— 核心业务价值

按 `uid` 维护状态，**实时流(0x02) 严格、补发流(0x03) 宽松**，两层过滤：

| 策略 | 规则 | 目的 |
|---|---|---|
| 固定阈值 | 实时回退→丢弃；实时跳增 >100km→丢弃；补发跳增 >300km→丢弃 | 快速排除明显异常 |
| 设备复位识别 | 补发回退幅度 >上次值 90%，且当前值合理小，且上次值异常大 → 判定设备重置放行 | 避免把里程清零误判为异常 |
| 中位数 + MAD | 近 20 条样本中位数，偏离 >500km → 丢弃 | 捕捉 subtle 漂移 |

### 4.4 Doris Sink（`VehicleRawDataDorisSink.java`）

JSON stream load，**批模式：1 万行或 3 秒 flush 一次**，写 `vehicle_mileage_data`。

<div class="callout callout-note" markdown="1">
**📌 流批分治设计**

Flink 主 job **没有**使用 `MileageAggregator` / `MileageWindowFunction` / `HourlyMileage` 这几个窗口聚合类。它们是"小时窗口实时聚合直接写汇总表"的另一套备选设计。线上实际采用 **"原始表落库 + Doris SQL 聚合"**：实时链路只做清洗，聚合交给 Doris，便于补数、重算、冷热分离。这是典型的**流批分治**——牺牲一点实时性，换来历史可回溯性。
</div>

### 4.5 配置来源

`NacosConfigUtils`：从 Nacos 拉取 `FlinkJobConfig`（`kafka.bootstrapServers` / `kafka.topic` / `kafka.groupId` / `maxMileage` / `doris.host` / `doris.database` / `doris.table` 等）。

## 5. 小时汇总表如何产生（定时任务 + 手动 + 历史回填）

### 5.1 定时任务：`VehicleMileageSummaryRefreshJob`（XXL-Job）

- 位置：`infrastructure/job/VehicleMileageSummaryRefreshJob.java`
- JobHandler 名：`VehicleMileageSummaryRefreshJob`
- 调度策略：**每小时执行一次**（XXL-Job 控制台配置）
- 逻辑：调用 `MvVehicleHourlyMileageServiceImpl.refreshLastSevenDays()`

```sql
-- refreshLastSevenDays 核心 SQL
INSERT INTO vehicle_mileage_summary_hourly
SELECT uid, vin, DATE_FORMAT(data_time,'%Y-%m-%d %H:00:00') AS window_hour,
       CAST(MIN(mileage_total) AS DECIMAL(18,1)) AS min_mileage,
       CAST(MAX(mileage_total) AS DECIMAL(18,1)) AS max_mileage,
       CAST(MAX(mileage_total)-MIN(mileage_total) AS DECIMAL(18,1)) AS mileage
FROM vehicle_mileage_data
WHERE data_time >= DATE_SUB(CURDATE(), INTERVAL 7 DAY)   -- 只刷过去7天
GROUP BY uid, vin, DATE_FORMAT(data_time,'%Y-%m-%d %H:00:00')
```

### 5.2 手动入口

`POST /vehicle/mileage/refresh` → 同一个 `refreshLastSevenDays()`（供临时补数）。

### 5.3 历史全量回填

`sql/02_zhangchao_202603171600_doris_dst_v2x_db.sql`：无条件 `INSERT..SELECT` 全量数据（用于上线/建表后一次性回填）。

## 6. 现状问题与隐患 — 设备乱上报双里程链案例（2026-08-14 实测）

> 真实数据暴露：同 `uid`(=vin=`HGCH1KB68RZ015679`) 在 13:01~13:18 的 17 分钟内**并行上报两条差异 ~25000km 的累计里程链**，且**全部成功入库**，导致小时汇总被污染。

### 6.1 异常数据形态（节选）

| uid | data_time | cmd | mileage_total |
|---|---|---|---|
| HGCH1KB68RZ015679 | 13:01:51 | **3(补发)** | 50857.8 |
| HGCH1KB68RZ015679 | 13:01:54 | **2(实时)** | 75827.9 |
| ... | 13:02~13:15 | 3 | 50858 → 50870.8（+13.0km） |
| ... | 13:02~13:18 | 2 | 75828 → 75835.8（+7.9km） |

- **补发链(cmd=3)**：50,857.8 → 50,870.8，单调缓增，链内"合法"
- **实时链(cmd=2)**：75,827.9 → 75,835.8，单调缓增，链内"合法"
- **两链基准差 ≈ 24,970 km**

### 6.2 计算错误的直接后果

```
13:00-14:00 小时窗口聚合：
  MIN(mileage_total) = 50857.8   ← 补发链最小值
  MAX(mileage_total) = 75835.8   ← 实时链最大值
  mileage = MAX - MIN = 24978.0 km   ← 该小时被算成行驶 24978 km（真实仅 8~13km，虚增 2000~3000 倍）
```

### 6.3 根因（为什么能入库 + 为什么算错）

**Flink 过滤全部失守的原因：**
1. `keyBy(uid)` 分区，但实时/补发用**两个独立状态**（`lastRealtimeMileage` / `lastHistoryMileage`），**互相不知晓**
2. 链内单调递增 + 增量 <100km（`MAX_HOURLY_MILEAGE`）→ 固定阈值全绿
3. 中位数过滤器是**每链独立**的，只检验"链内一致性"，检验不了"跨 cmd 基准不一致"
4. 入口粗筛（`>0 && <999999`、`≤ maxMileage`）两条链都在"合法区间"

**Doris 聚合出错的原因：**
- 汇总 SQL 按 `uid+vin+hour` GROUP BY，**不区分 cmd**，把两条里程链的 min/max 混在一起做差

<div class="callout callout-warning" markdown="1">
**⚠️ 一句话根因**

**Flink 只做"单链增量"校验，不做"跨 cmd 基准一致性"校验**；Doris 汇总 SQL 又不区分 cmd，同一小时窗口内两条里程链的 min/max 混做差。入口假设"一辆车只有一个累计里程源"，现实是设备会给两个——假设崩塌，过滤与聚合全失效。
</div>

### 6.4 危害传播链路

```
设备双里程源(cmd=2≈75800 / cmd=3≈50800，差~25000km)
   │ 两条链内部都"合法" → Flink 全放行
   ▼
vehicle_mileage_data   ← 两条链全部入库（原始表被污染）
   │
   ▼ 每小时刷新 SQL: MAX-MIN 跨链取差
vehicle_mileage_summary_hourly  ← 单小时 mileage≈25000km（污染）
   │
   ├─ /totalMileage   → SUM(mileage) 虚增
   ├─ 日报 dailyList  → daily_mileage 虚增
   └─ 里程分布/报表   → 全部失真
```

**为什么刷新救不了**：`refreshLastSevenDays` 从已被污染的 `vehicle_mileage_data` 重算，源头脏，怎么刷都脏。

### 6.5 修复方向（按性价比排序）

1. **入口交叉校验（治本之一，改动最小收益最大）**：`MileageFilterFunction` 给实时链/补发链做**基准交叉比对**——同一 uid 两链基准差 > 阈值（如 500km）时判定异常，只保留与主流一致的一条或整体告警。脏数据在入口拦截，不再污染原始表。
2. **汇总 SQL 防御（治本之二）**：a) 聚合时**区分 cmd**，实时/补发分别取 min/max，选增量更合理的一条；或 b) 给 `mileage = MAX-MIN` 加**合理性上限**（超阈值置 0/标记异常，宁缺毋滥）；或 c) 改用**逐条增量累加**（按时间排序累计相邻合法增量），天然绕过跨链跳变。
3. **存量数据修复**：修正 Flink 后，对受影响时段**按新口径重建** `vehicle_mileage_summary_hourly`（不是简单再刷一次）。
4. **可观测性**：对"单 uid 双里程链基准差超阈值"打日志/告警，尽早发现设备固件问题。

## 7. 几个易踩坑细节

1. **两个"阈值"别混淆**：
   - Flink 主 job 的 `maxMileage`（Nacos 配置）→ 过滤**单条上报**累计里程合理性（如 999999）
   - `MileageFilterFunction` 的 `MAX_HOURLY_MILEAGE=100` / `MEDIAN_THRESHOLD=500` → 过滤**相邻上报增量**异常
   - 汇总 SQL 的 `mileage = max - min` → 小时窗口行驶增量

2. **`totalMileage` ≠ 终身总里程**：是查询窗口内的行驶里程累加 + min/max 范围。

3. **接口校验双轨**：`VehicleTotalMileageQuery` 注解 `@Size(max=200)` 限制 vins≤200；Service 内 `VIN_LIMIT=1000` 是日报类接口用的，`totalMileage` 走的是注解校验。

4. **空壳项目**：`dst-iot-vehicle-mileage-calculate-job` 仅含启动类 + 依赖（Flink/doris connector），建议与 `dst-iot-car-calculator` 二选一维护，避免歧义。

5. **数据分区**：`vehicle_mileage_summary_hourly` 手工分 p2023~p2028 年分区；`vehicle_mileage_data` 用月动态分区（`dynamic_partition`）。查询 `window_hour` 必须带区间，避免全分区扫描。

## 8. 相关笔记

- [车辆里程计算 — 业务科普版](/2026/09/17/mileage-business-explained/) — 面向产品/运营的业务说明（架构、流程、FAQ、新口径）
- [dst-iot-car-calculator 车辆里程计算作业 — 详细设计文档](/2026/09/10/mileage-calculator-design/) — Flink 作业本身的详细设计与计算逻辑（本文档的姊妹篇）
- [车辆全图分布实时作业 — 设计文档](/2026/06/26/vehicle-distribution-flink-job/) — 同类的 32960→Kafka→Flink→Doris 实时作业
- **Flink realtime-service 消费堆积与车辆状态高负载分析** — Flink 消费性能问题
- **32960数据分发逻辑梳理** — 上游 32960 数据如何进 Kafka
