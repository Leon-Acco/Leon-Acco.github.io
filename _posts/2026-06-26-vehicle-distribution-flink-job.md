---
layout: post
title: "车辆全图分布实时作业 — 设计文档"
date: 2026-06-26
categories: [iot]
mermaid: false
math: false
toc: true
---
> **项目仓库**: `D:\code\flink_calculator\dst-iot-vehicle-realtime-flink-job`
> **版本**: V3 (2026-06-26)
> **状态**: 已编译通过, 待部署联调

## 1. 项目定位

**消费 Kafka 32960 协议原始报文, 实时计算每辆车的位置 + 在线状态, 写入 Doris (历史/OLAP) 和 Redis (实时在线判定)**。

- **上游**: Kafka 32960 二进制报文 (GB/T 32960 国标)
- **维度**: MySQL `basic_vehicle_info` (车辆基础信息)
- **下游**:
  - Doris 宽表 (按 VIN 主键的实时位置, 30-60s 入库)
  - Redis ZSET (按 VIN 的最新上报时间, 毫秒级)

## 2. 核心拓扑

```
                ┌─────────────────────────────────────┐
                │           main() 引导阶段            │
                │  (JobManager JVM, fail-fast)         │
                │                                     │
                │  1. Nacos 加载配置                   │
                │  2. validateGeoJson (R-tree 校验)    │
                │  3. loadVehicleInfoSnapshot (JDBC)   │
                └────────────────┬────────────────────┘
                                 │
       ┌─────────────────────────┴──────────────────────┐
       │                                                │
       ▼                                                ▼
┌──────────────┐                          ┌─────────────────────────┐
│  Kafka       │                          │  Bootstrap + CDC        │
│  (32960)     │                          │  (车辆信息全量 + 增量)   │
│              │                          │                         │
│  VehiclePosition                          │  VehicleInfoBroadcast   │
│  Deserializer                              │  Event                  │
└──────┬───────┘                          └────────────┬────────────┘
       │                                               │
       │  keyBy(vin)                         broadcast │
       │                                               │
       └──────────────► connect ◄──────────────────────┘
                       │
                       ▼
       ┌───────────────────────────────────┐
       │  VehiclePositionProcessFunction   │
       │  (KeyedBroadcastProcessFunction)  │
       │                                   │
       │  主流 → out.collect                │
       │  侧输出 → DORIS_OUTPUT_TAG         │
       └────────────┬──────────────────────┘
                    │
          ┌─────────┴─────────┐
          │                   │
          ▼ 主流               ▼ 侧输出
   ┌────────────────┐   ┌────────────────────┐
   │ VehicleOnline  │   │ VehiclePosition    │
   │ RedisSink      │   │ DorisSink          │
   │                │   │                    │
   │ ZADD time      │   │ Stream Load JSON   │
   │ vin score=ts   │   │ (VIN Unique Key)   │
   └────────────────┘   └────────────────────┘
```

## 3. 关键设计决策

### 3.1 在线判定 vs 位置记录 (V3 重要变更)

**核心语义**: "在线" 和 "有定位" 是两个不同的概念, 不能混为一谈。

| 场景 | Redis ZSet | Doris latestPosition | 说明 |
|------|-----------|---------------------|------|
| 坐标有效 + 时间有效 | ✅ 更新 | ✅ 更新 | 走完整富化流程 |
| **坐标无效 (GPS 丢失) + 时间有效** | ✅ **更新** | ❌ 跳过 | 设备仍在线 |
| 乱序 (旧时间戳) | ❌ 跳过 | ❌ 跳过 | 避免覆盖更新值 |
| reportTime 缺失 | ❌ 跳过 | ❌ 跳过 | ZSet 必须有 score |

**实现要点**:
- `VehiclePositionDeserializer` **不再丢弃**坐标无效的消息 (V3 改动)
- `VehiclePositionProcessFunction.processElement` 分流:
  - 第 2 步: `lastReportTime.update + out.collect` (在线判定, 无视坐标)
  - 第 3 步: `if (!isValidCoordinate) return` (Doris 跳过)

**踩坑场景**: 车辆进隧道 / 地下车库 / GPS 故障时, 上报 `(0,0)` 或境外坐标。
- V2 实现: 反序列化阶段直接丢弃 → Redis ZSet 不更新 → 车辆被误判为"离线"
- V3 实现: 反序列化器只丢弃 VIN 缺失 / 非 0x02 命令字, 坐标无效正常下发 → ProcessFunction 分流

### 3.2 Bootstrap vs CDC 分工

| 维度 | `validateGeoJson` (R-tree) | `loadVehicleInfoSnapshot` (车辆信息) |
|------|----------------------------|--------------------------------------|
| 加载什么 | GeoJSON 文件 → R-tree 索引 | MySQL 表 → POJO 列表 |
| 数据性质 | 静态资源 (地理边界, 几乎不变) | 业务数据 (会变, 需要 CDC) |
| 加载结果可序列化 | ❌ R-tree 不可序列化 | ✅ POJO 列表可序列化 |
| main 加载的实例去向 | 🗑️ 丢弃 (纯 fail-fast) | 📤 通过 `env.fromCollection` 传到 subtask |
| subtask 怎么拿到 | 自己重新加载 (JVM 单例) | Flink 框架自动分发广播流 |
| 运行时有增量吗 | ❌ 没有 | ✅ 有 (CDC `latest-offset`) |

**关键限制**: Flink 跨 JVM 不共享内存。R-tree 100MB+ 大对象图无法跨 TM 共享, JVM 单例已是最佳实践。

### 3.3 主流 + 侧输出双通道

| 输出通道 | 触发时机 | 走的 Sink | 频率 | 用途 |
|---------|---------|----------|------|------|
| 主流 `out.collect` | `processElement` 每条 | Redis ZSet | 毫秒级 | 在线判定 |
| 侧输出 `ctx.output(DORIS_OUTPUT_TAG)` | `onTimer` 触发 | Doris Stream Load | 30-60s/辆 | 历史 / OLAP |

**为什么不直接内联 Redis 写**: 同步 Jedis 调用会阻塞主流 (1-2ms RTT), 33k 条/秒直接吞吐崩溃。**未来优化方向**: AsyncFunction 异步写 Redis (吞吐 5-10x)。

### 3.4 Doris 表模型

**Unique Key 模型**, 主键 `vin_code`, 自动 upsert:

```sql
CREATE TABLE vehicle_realtime_position (
    `vin_code`         VARCHAR(32)    NOT NULL COMMENT '车架号 (主键)',
    `car_no`           VARCHAR(32)    NULL,
    `brand_id`         BIGINT         NULL,
    `car_brand_name`   VARCHAR(64)    NULL,
    `type_id`          BIGINT         NULL,
    `car_model_name`   VARCHAR(64)    NULL,
    `lng`              DOUBLE         NULL COMMENT 'WGS-84 经度',
    `lat`              DOUBLE         NULL,
    `gcj02_lng`        DOUBLE         NULL COMMENT 'GCJ-02 经度 (高德)',
    `gcj02_lat`        DOUBLE         NULL,
    `report_time`      DATETIME       NULL,
    `recv_time`        DATETIME       NULL,
    `etl_time`         DATETIME       NULL,
    `province_code`    VARCHAR(16)    NULL,
    `province_name`    VARCHAR(32)    NULL,
    `city_code`        VARCHAR(16)    NULL,
    `city_name`        VARCHAR(32)    NULL,
    `district_code`    VARCHAR(16)    NULL,
    `district_name`    VARCHAR(32)    NULL,
    `tile_id_z4`       VARCHAR(32)    NULL COMMENT '国家级行政区划',
    `tile_id_z5`       VARCHAR(32)    NULL,
    `tile_id_z6`       VARCHAR(32)    NULL COMMENT '省级',
    `tile_id_z7`       VARCHAR(32)    NULL,
    `tile_id_z8`       VARCHAR(32)    NULL COMMENT '市级',
    `tile_id_z9`       VARCHAR(32)    NULL,
    `tile_id_z10`      VARCHAR(32)    NULL COMMENT '街道级'
)
ENGINE=OLAP
UNIQUE KEY(`vin_code`)
DISTRIBUTED BY HASH(`vin_code`) BUCKETS 10
PROPERTIES (
    "replication_allocation" = "tag.location.default: 1",
    "enable_unique_key_merge_on_write" = "true",
    "light_schema_change" = "true"
);
```

**关键参数决策**:
- **不分区**: 以车为维度的实时表, 不需要按时间分区 (Unique 模型自动按 VIN 覆盖)
- **BUCKETS 10**: 100w 车辆合适, 1000w 车 32+
- **MoW (Merge on Write)**: 高频 upsert 性能更好, 读性能好

## 4. 数据流速对照

| 通道 | 延迟 | 用途 | 查询方式 |
|------|------|------|---------|
| Redis ZSet | 毫秒级 | 在线判定 / 实时分布 | `ZCOUNT vehicle:realtime:time (now-300) +inf` |
| Doris 宽表 | 30-60s | 历史轨迹 / OLAP | `SELECT * FROM vehicle_realtime_position WHERE vin_code = ?` |

**为什么不能只用 Doris**: Doris 30-60s 入库延迟, 车辆刚上线 / 离线需要毫秒级响应。
**为什么不能只用 Redis**: Redis 不擅长历史轨迹查询 (按时间范围扫), 也不擅长 OLAP (GROUP BY 聚合)。

## 5. KeyedState 设计

```
VehiclePositionProcessFunction:
├── lastReportTime    (Long)              乱序检测
├── latestPosition    (VehiclePosition)   onTimer 时输出到 Doris
└── timerRegistered   (Boolean)           避免重复注册 timer

BroadcastState:
└── vehicleInfoBroadcast (Map<vin, BasicVehicleInfo>)
```

**状态规模估算** (100w 车):
- lastReportTime: 8 bytes × 100w = 8 MB
- latestPosition: ~200 bytes × 100w = 200 MB
- timerRegistered: 1 byte × 100w = 1 MB
- 合计 KeyedState: ~210 MB / subtask
- BroadcastState: ~120 MB / subtask (broadcast 每子任务一份)

## 6. 性能基线

**单条耗时拆解**:

| 步骤 | 耗时 |
|------|------|
| 反序列化 32960 协议 | ~10 μs |
| 乱序检测 (RocksDB read) | ~50 μs |
| 坐标系转换 WGS84→GCJ02 | <1 μs |
| 行政区划匹配 R-tree | 0.1-1 ms |
| 网格瓦片 7 级计算 | <5 μs |
| KeyedState 读写 (8 次) | ~400 μs |
| BroadcastState 读 | ~30 μs |
| Redis Pipeline (2 命令) | 1-2 ms |
| **总计** | **1.5-3.5 ms / 条** |

**吞吐估算** (33k 条/秒 = 100w 车 / 30s):
- 单 subtask: 500 条/秒
- 70 个 subtask: 35k 条/秒 ✅
- 集群规模经验值: parallelism × 单 subtask 吞吐 > 输入吞吐 × 1.5

**优化方向** (按收益排序):
1. ⭐⭐⭐⭐⭐ AsyncFunction 异步写 Redis (吞吐 5-10x)
2. ⭐⭐ HeapStateBackend (状态 < 2GB 时更快)
3. ⭐ 行政区匹配从 R-tree 改 Grid Index (省 0.5ms/条)

## 7. 已知陷阱与踩坑记录

### 7.1 V1 陷阱: CDC 启动前 Kafka 早期数据漏富化
**问题**: CDC `initial` 模式启动后, Kafka 主流立即消费, 但 CDC 全表扫描需要数分钟, 期间早期位置数据无业务字段。
**解决**: main() 阶段同步 `bootstrapVehicleInfo` 加载全量, 通过 `env.fromCollection` union 到 CDC 流前面。

### 7.2 V2 陷阱: 反序列化器丢弃坐标无效消息 → 在线判定错误
**问题**: 车辆 GPS 丢失时上报 `(0,0)`, 反序列化器直接 return, Redis ZSet 不更新, 车辆被误判为"离线"。
**解决** (V3): 反序列化器不再过滤坐标, ProcessFunction 分流处理 (坐标无效只写 Redis)。

### 7.3 V2 陷阱: Redis Sink 写了 7 类结构, 单 subtask 吞吐低
**问题**: Hash + GEO + ZSET + 4 类统计 HINCRBY, 单条 20 个 Redis 命令, 吞吐仅 500 条/秒。
**解决** (V3): Redis 只保留 ZSET (在线判定), 其他迁移到 Doris。

### 7.4 陷阱: Doris ZSet 跟 Redis ZSet 是两回事
- Redis ZSet = Sorted Set (member=vin, score=reportTime)
- Doris 没有 ZSet 概念, 用 `vehicle_realtime_position` Unique 表存最新位置

### 7.5 陷阱: JSON 序列化时区
Doris DATETIME 不带时区, 必须显式按 `Asia/Shanghai` 转换 (VehiclePositionDorisSink.formatEpochMillis)。

## 8. 文件结构

```
src/main/java/com/dst/iot/car/vehiclemap/
├── VehicleRealtimeJob.java                 主入口
├── config/
│   ├── FlinkJobConfig.java                 配置 POJO
│   └── NacosConfigUtils.java               Nacos 加载
├── source/
│   ├── VehicleKafkaSourceFactory.java      Kafka Source 构建
│   └── VehiclePositionDeserializer.java    32960 协议反序列化器 (V3 不再丢坐标)
├── cdc/
│   ├── JsonCdcDeserializer.java            通用 Debezium JSON 反序列化
│   ├── BasicVehicleInfoParser.java         JSON → POJO
│   ├── VehicleInfoBootstrapLoader.java     JDBC 全量加载
│   └── VehicleInfoCdcSource.java           Bootstrap + CDC union → broadcast
├── process/
│   └── VehiclePositionProcessFunction.java KeyedBroadcastProcessFunction (V3 分流)
├── sink/
│   ├── VehicleOnlineRedisSink.java         Redis ZSet 写入 (单条实时)
│   └── VehiclePositionDorisSink.java       Doris Stream Load (onTimer 批量)
├── pojo/
│   ├── VehiclePosition.java                位置 POJO
│   ├── BasicVehicleInfo.java               车辆信息 POJO (7 个业务字段)
│   ├── VehicleInfoBroadcastEvent.java      广播事件包装
│   ├── DistrictInfo.java                   行政区划信息
│   └── RedisKeys.java                      Redis Key 计算器
├── district/                               R-tree 行政区匹配器
│   ├── DistrictMatcher.java                接口
│   ├── DefaultDistrictMatcher.java         默认实现
│   ├── DistrictRTree.java                  R-tree 索引
│   └── ...
└── utils/
    └── PositionUtil.java                   WGS84 ↔ GCJ02 转换
```

## 9. 配置项 (Nacos)

```yaml
# Kafka
kafka.bootstrapServers: "..."
kafka.topic: "vehicle-32960"
kafka.groupId: "vehicle-realtime-flink-job"

# Redis (只写在线 ZSet)
redis.host: "..."
redis.port: 6379
redis.password: "..."
redis.database: 0
redis.pool.maxTotal: 32
redis.key.realtimePrefix: "vehicle:realtime"

# MySQL CDC (basic_vehicle_info)
vehicle.mysqlHost: "..."
vehicle.mysqlPort: 3306
vehicle.mysqlUsername: "..."
vehicle.mysqlPassword: "..."
vehicle.mysqlDatabase: "dst_v2x_db"
vehicle.mysqlTable: "basic_vehicle_info"
vehicle.mysqlStartupMode: "latest-offset"   # bootstrap 已覆盖全量, CDC 只读增量

# Doris
doris.host: "..."
doris.username: "root"
doris.password: "..."
doris.database: "dst_iot"
doris.table: "vehicle_realtime_position"
doris.flushIntervalMs: 30000                # 30 秒/辆

# 网格瓦片
grid.zoomMin: 4
grid.zoomMax: 10

# GeoJSON
geojson.path: ""
geojson.classpathResource: "data/amap_districts_final.geojson"

# 作业名
jobName: "Vehicle Realtime Distribution Job"
```

## 10. 在线统计查询 (Redis 端)

```bash
# 5 分钟内活跃车辆数
ZCOUNT vehicle:realtime:time (now-300) +inf

# 5 分钟内活跃 VIN 列表
ZRANGEBYSCORE vehicle:realtime:time (now-300) +inf

# 某车最后上报时间
ZSCORE vehicle:realtime:time {vin}

# 全部车辆最后上报时间 (前 100)
ZREVRANGE vehicle:realtime:time 0 100 WITHSCORES
```

**过期策略**: ZSet 整体 7 天 TTL, 长期不上报的车辆自动清理。

## 11. 历史轨迹查询 (Doris 端)

```sql
-- 某车辆当前最新位置
SELECT * FROM vehicle_realtime_position WHERE vin_code = 'LRDASRE2K1D0000001';

-- 某省份所有在线车辆 (5 分钟内)
SELECT vin_code, car_no, lng, lat, report_time
FROM vehicle_realtime_position
WHERE province_code = '110000'
  AND report_time > NOW() - INTERVAL 5 MINUTE;

-- 某网格 z6 内的车辆数 (地图聚合)
SELECT tile_id_z6, COUNT(*) AS car_count
FROM vehicle_realtime_position
WHERE report_time > NOW() - INTERVAL 5 MINUTE
GROUP BY tile_id_z6;
```

## 12. 待办与扩展方向

- [ ] Doris DDL 在测试库创建
- [ ] Nacos 配置补全 (vehicle.mysql*, doris.*)
- [ ] 联调: Kafka → ProcessFunction → Redis ZSet 验证毫秒级延迟
- [ ] 联调: CDC 更新 basic_vehicle_info → ProcessFunction 富化生效
- [ ] 压测: 100w 车 / 30s 上报场景下吞吐能否扛住
- [ ] **优化**: AsyncFunction 异步写 Redis (未来扩到 500w 车时必做)
- [ ] **优化**: Doris 端物化视图 (按省市聚合时加速)
- [ ] 监控: Redis writeFailCount + Doris Stream Load 失败率告警

## 13. 参考

- **Flink realtime-service 消费堆积与车辆状态高负载分析** — 老 Flink 作业的踩坑记录
- [车辆轨迹按区域查询系统 — 详细设计方案](/2026/10/23/trajectory-region-system-design/) — 区域查询的设计思路
- 参考项目: `D:\code\flink_calculator\dst-iot-adas-risk-aggregation-job` (CDC + Doris 范例)
