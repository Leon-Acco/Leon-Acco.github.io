---
layout: post
title: "Kafka数据入库SelectDB方案"
date: 2026-10-09
categories: [iot]
series: iov-dw
series_order: 2
tags: [车联网, 数仓, SelectDB, ODS, Kafka, GB32960]
mermaid: false
math: false
toc: true
---
# Kafka → SelectDB：ODS 设备上报明细表（定稿方案）

把 `dst-iot-transpond-service` 转发的 GB/T 32960 报文（0x02 实时 / 0x03 补发）写入 SelectDB `dst_iot_dw.ods_vehicle_report`：**高频字段打平成列，长尾字段整条进 VARIANT（64 Subcolumn + Sparse 溢出），动态分区保留 90 天**。

```
Kafka Topic（SASL_SSL + snappy，无 key 无 header）
    │
    ▼
Kafka Connect Distributed ── doris-kafka-connector（Stream Load）
    │
    ▼
dst_iot_dw.ods_vehicle_report（SelectDB 5.7.99，DUPLICATE KEY 模型）
```

<div class="callout callout-note" markdown="1">
**📝 ODS 职责边界**

ODS 只负责原始明细存储，**不做里程等复杂业务计算**——那是 DWD 层的事。
</div>

## 一、工件清单

可直接执行的原始工件在 `Kafka数据入库selectDB/` 子目录（本文档为整合阅读版，内容与工件一字不差）：

| 文件 | 说明 |
| --- | --- |
| `01_建表语句_ods_vehicle_report.sql` | 建库建表（定稿版）+ 查询示例 + 运维命令 |
| `02_selectdb_kafka_connector.json` | Kafka Connect Distributed 模式的 Connector REST 提交体 |
| `03_worker消费者配置片段.properties` | SASL_SSL Kafka 集群在 connect-distributed.properties 里的片段 |

## 二、定稿设计决策

| # | 决策点 | 定稿取值 | 理由 |
| --- | --- | --- | --- |
| 1 | 热字段 | 85 列直接打平（含基础字段；2008/2009 储能与 2010 TBOX 段按码表全量） | 高频查询字段见名知意（snake_case），注释保留 counter id 溯源 |
| 2 | 长尾字段 | 整条报文落 `payload VARIANT` | 拒绝 200 列宽表；任何 counter 永远可查 |
| 3 | VARIANT 子列化 | `variant_max_subcolumns_count=64` + `variant_enable_typed_paths_to_sparse=true` | 最多 64 个 Path 转 Subcolumn，超出的进 Sparse |
| 4 | Key | `DUPLICATE KEY(ds, dev_code)` | 前缀索引按"日期 + 设备"裁剪扫描（Key 必须是建表列连续前缀） |
| 5 | 分区 | `RANGE(ds)` 按天，动态分区：保留 90 天、预建未来 3 天 | 生命周期自动化，过期分区自动删除 |
| 6 | 分桶 | `HASH(dev_code) BUCKETS AUTO` | 同设备数据落同桶，单车时间线查询少 shuffle |
| 7 | 副本 | 测试 1 / 生产 3（`replication_num`） | 建表默认 1，生产环境改 `"3"` |
| 8 | 存储格式 | `storage_format = V3` | VARIANT 子列较多的宽列场景推荐 V3 |
| 9 | 入库元数据 | `load_time DEFAULT CURRENT_TIMESTAMP` | 不经 connector 映射，由服务端自动填充 |

## 三、表结构速览

### 3.1 基础字段

| 列 | 类型 | 约束 | 说明 |
| --- | --- | --- | --- |
| `ds` | DATE | NOT NULL | 日期分区 |
| `dev_code` | VARCHAR(64) | NOT NULL | 设备号（终端 UID），源 JSON 字段 `device_no` |
| `vin_code` | VARCHAR(32) | NULL | 车架号，源 JSON 字段 `vin`；优先 Kafka 头 vinCode，缺失回退设备号 |
| `data_time` | DATETIME | NOT NULL | 数据采集时间（车载终端时间戳） |
| `cmd` | TINYINT | NULL | 2=实时数据 3=补发数据 |
| `source` | TINYINT | NULL | 3=平台转发 2=双链路 1=设备直连 |
| `load_time` | DATETIME | DEFAULT 当前时间 | 入库时间 |

<div class="callout callout-warning" markdown="1">
**⚠️ 字段动态缺失**

转发端缺失的数据域整段不出现在 JSON，connector 设 `strict_mode=false`，缺失字段自然为 NULL。下游判"无数据"用 `IS NULL`，不要用 `= ''`。
注意 `dev_code`/`data_time` 为 NOT NULL：源消息缺这两个字段时该行导入失败，进死信队列。
</div>

### 3.2 打平热字段（counter id → 列名）

| 数据域 | counter → 列名 |
| --- | --- |
| 整车 2001xxx | 2001001 `vehicle_status` ｜ 2001002 `charge_status` ｜ 2001004 `speed` ｜ 2001005 `total_mileage` ｜ 2001006 `total_voltage` ｜ 2001007 `total_current`（(原始-10000)/10）｜ 2001008 `soc` ｜ 2001009 `dcdc_status` ｜ 2001011 `insulation_resistance` ｜ 2001012 `accel_pedal_position` ｜ 2001013 `brake_pedal_status` ｜ 200101001 `gear` ｜ 200101002 `has_braking_force` ｜ 200101003 `has_driving_force` |
| 驱动电机 2002xxx（多电机列表字符串） | 2002002 `motor_status_list` ｜ 2002003 `motor_controller_temp_list` ｜ 2002004 `motor_speed_list` ｜ 2002005 `motor_torque_list` ｜ 2002006 `motor_temp_list` ｜ 2002007 `motor_controller_voltage_list` ｜ 2002008 `motor_controller_current_list`（多电机维度分析在 DWD 拆明细） |
| 车辆位置 2005xxx | 2005002 `longitude` ｜ 2005003 `latitude` ｜ 200500101 `positioning_status` ｜ 200500102 `latitude_hemisphere` ｜ 200500103 `longitude_hemisphere` |
| 极值数据 2006xxx | 2006001~3 `max_voltage_subsystem_no` / `max_voltage_cell_no` / `max_cell_voltage` ｜ 2006004~6 `min_voltage_*` ｜ 2006007~9 `max_temp_*` ｜ 2006010~12 `min_temp_*`（温度偏移 -40） |
| 储能装置电压/温度 2008/2009xxx | 2008001 `ess_voltage_subsystem_count` ｜ 2008002 `ess_voltage_list`（电压值 ÷1000 得 V）｜ 2009001 `ess_temp_subsystem_count` ｜ 2009002 `ess_temp_list`（子系统明细在 DWD 拆） |
| TBOX 2010xxx（码表全量，2010022/23 留 payload） | 2010003 `power_supply_flag` ｜ 2010004 `tbox_version` ｜ 2010005 `relay_status` ｜ 2010006 `terminal_bind_status` ｜ 2010007 `controller_version_status` ｜ 2010008 `lock_status` ｜ 2010009 `charge_energy` ｜ 2010010 `vcu_version` ｜ 2010011 `charge_mode_flag` ｜ 2010012 `lock_flag` ｜ 2010013 `fault_count` ｜ 2010014 `vehicle_model_id` ｜ 2010015 `tbox_work_status` ｜ 2010016 `vehicle_on_signal` ｜ 2010017 `vehicle_charge_signal` ｜ 2010018 `can_wakeup_status` ｜ 2010020 `low_battery_voltage` ｜ 2010021 `tbox_battery_voltage` ｜ 2010024 `dual_link_status` ｜ 2010025 `maintenance_tip` ｜ 2010026 `payment_tip` ｜ 2010027 `inspection_tip` ｜ 2010044 `battery_removal_status` ｜ 2010045 `gps_mode` ｜ 2010046 `gnss_satellite_count` ｜ 2010051 `battery_soh` ｜ 2010052 `ac_work_status` ｜ 2010053 `water_pump_fault` ｜ 2010054 `fast_charge_relay_fault` ｜ 2010055 `vacuum_degree` ｜ 2010056 `gnss_work_status` ｜ 2010057 `emmc_work_status` ｜ 2010058 `flash_work_status` ｜ 2010059 `ethernet_status` |
| 通用报警 2013xxx | 2013001 `max_alarm_level` ｜ 2013002 `general_alarm_flags`（64bit） |

### 3.3 长尾字段（仅存 payload，用时提取）

| 数据域 | counter 范围 | 提取示例 |
| --- | --- | --- |
| TBOX 双链路IP端口（运维诊断） | 2010022/2010023 | `payload -> '$.2010023'`（IP端口列表字符串） |
| 冷藏车冷机 | 2011002~2011021 | `CAST(payload -> '$.2011011' AS SMALLINT)` |
| 电池故障报警等级 | 2012002~2012012 | `CAST(payload -> '$.2012011' AS CHAR(2))`，"00"=正常 |
| 故障代码列表 | 2013003~2013010 | `payload -> '$.2013004'` |

## 四、VARIANT 机制：64 是子列化上限，不是字段数上限

```
payload 里的 Path：
  Path 1  ~ Path 64   → Subcolumn 化（独立类型 + 统计信息，查询像普通列）
  Path 65 ~ Path N    → Sparse（数据照常保存，只是不做子列化）
```

- 热字段**双写**：既在打平列、也在 payload 里。代价是少量冗余存储，换来新增热字段可在线升级、历史可用 `UPDATE ... CAST(payload -> ...)` 回填，**永不重建表**。
- 长尾字段查询语法统一：`payload -> '$.<counter>'`，必要时 `CAST` 成目标类型。

<div class="callout callout-tip" markdown="1">
**💡 热字段晋升路径**

某 payload 字段查询变高频后：
1. `ALTER TABLE dst_iot_dw.ods_vehicle_report ADD COLUMN xxx TYPE DEFAULT NULL;`
2. 同步改 connector 的 `columns` + `jsonpaths`（见第七节强耦合警告）；
3. `UPDATE` 回填历史：`UPDATE ... SET xxx = CAST(payload -> '$.2011xxx' AS ...)`;
4. 必要时 `ADD INDEX ... USING INVERTED` 建倒排索引。
</div>

## 五、完整建表 SQL（01 号工件）

```sql
-- ============================================================================
-- ODS 设备上报事实明细表（定稿版）
-- SelectDB 5.7.99
--
-- 设计目标：
--   1. 高频字段：直接打平为普通列
--   2. 长尾字段：使用 VARIANT 保存
--   3. VARIANT：最多 64 个 Path 进行 Subcolumn 化，超过 64 个的低频 Path 进入 Sparse
--   4. ODS 只负责原始明细存储，不在 ODS 做里程等复杂业务计算
--   5. 生命周期：动态分区保留最近 90 天
--   6. 分区：ds 按天分区
--   7. Key：DUPLICATE KEY(ds, dev_code)，前缀索引按"日期 + 设备"裁剪扫描
--   8. 副本：测试环境 1 副本，生产环境改 3 副本
--   9. 存储格式：V3，适合 VARIANT 宽列场景
--
-- 来源：dst-iot-transpond-service Kafka Topic
--       GB/T 32960，0x02 实时数据 / 0x03 补发数据
-- ============================================================================

-- ============================================================================
-- 1. Database
-- ============================================================================

CREATE DATABASE IF NOT EXISTS dst_iot_dw;

-- ============================================================================
-- 2. Table
-- ============================================================================

CREATE TABLE IF NOT EXISTS dst_iot_dw.ods_vehicle_report
(
    -- ========================================================================
    -- 基础字段
    -- ========================================================================

    ds                          DATE        NOT NULL
                                COMMENT "日期分区 yyyy-MM-dd",

    dev_code                    VARCHAR(64) NOT NULL
                                COMMENT "设备号(终端UID,报文头识别码)",

    vin_code                    VARCHAR(32) NULL
                                COMMENT "车架号(优先Kafka头vinCode,缺失回退设备号)",

    data_time                   DATETIME    NOT NULL
                                COMMENT "数据采集时间(车载终端时间戳)",

    cmd                         TINYINT     NULL
                                COMMENT "命令码 2=实时数据 3=补发数据",

    source                      TINYINT     NULL
                                COMMENT "数据来源 3=平台转发 2=双链路 1=设备直连",

    -- ========================================================================
    -- 整车数据 2001xxx
    -- ========================================================================

    vehicle_status              TINYINT     NULL
                                COMMENT "车辆状态 1启动 2熄火 3其他 (2001001)",

    charge_status               TINYINT     NULL
                                COMMENT "充电状态 1停车充电 2行驶充电 3未充电 4其他 (2001002)",

    speed                       DOUBLE      NULL
                                COMMENT "车速 km/h 精度0.1 (2001004)",

    total_mileage               DOUBLE      NULL
                                COMMENT "累计里程 km 精度0.1 有效0~999999.9 (2001005)",

    total_voltage               DOUBLE      NULL
                                COMMENT "总电压 V 精度0.1 (2001006)",

    total_current               DOUBLE      NULL
                                COMMENT "总电流 A 精度0.1 (原始-10000)/10 (2001007)",

    soc                         TINYINT     NULL
                                COMMENT "SOC % 0~100 (2001008)",

    dcdc_status                 TINYINT     NULL
                                COMMENT "DC-DC状态 1工作 2断开 (2001009)",

    insulation_resistance       INT         NULL
                                COMMENT "绝缘电阻 kΩ 原始值 (2001011)",

    accel_pedal_position        TINYINT     NULL
                                COMMENT "加速踏板行程 % (2001012)",

    brake_pedal_status          SMALLINT    NULL
                                COMMENT "制动踏板状态 原始值 (2001013)",

    gear                        TINYINT     NULL
                                COMMENT "档位 低4位 (200101001)",

    has_braking_force           TINYINT     NULL
                                COMMENT "是否有制动力 bit4 (200101002)",

    has_driving_force           TINYINT     NULL
                                COMMENT "是否有驱动力 bit5 (200101003)",

    -- ========================================================================
    -- 驱动电机 2002xxx
    --
    -- 当前协议为多电机列表字符串。
    -- 如果未来需要单电机维度分析，在 DWD 层拆明细。
    -- ========================================================================

    motor_status_list           STRING      NULL
                                COMMENT "驱动电机状态列表 1耗电 2发电 3停机 (2002002)",

    motor_controller_temp_list  STRING      NULL
                                COMMENT "电机控制器温度℃列表 偏移-40 (2002003)",

    motor_speed_list            STRING      NULL
                                COMMENT "电机转速r/min列表 偏移-20000 (2002004)",

    motor_torque_list           STRING      NULL
                                COMMENT "电机转矩N·m列表 0.1精度 (2002005)",

    motor_temp_list             STRING      NULL
                                COMMENT "电机温度℃列表 偏移-40 (2002006)",

    motor_controller_voltage_list
                                STRING      NULL
                                COMMENT "控制器输入电压V列表 0.1精度 (2002007)",

    motor_controller_current_list
                                STRING      NULL
                                COMMENT "控制器母线电流A列表 (原始-10000)/10 (2002008)",

    -- ========================================================================
    -- 车辆位置 2005xxx
    -- ========================================================================

    longitude                   DOUBLE      NULL
                                COMMENT "经度 度 6位小数 (2005002)",

    latitude                    DOUBLE      NULL
                                COMMENT "纬度 度 6位小数 (2005003)",

    positioning_status          TINYINT     NULL
                                COMMENT "定位状态 bit0 0=未定位 1=已定位 (200500101)",

    latitude_hemisphere         TINYINT     NULL
                                COMMENT "纬度类型 bit1 0=北纬 1=南纬 (200500102)",

    longitude_hemisphere        TINYINT     NULL
                                COMMENT "经度类型 bit2 0=东经 1=西经 (200500103)",

    -- ========================================================================
    -- 极值数据 2006xxx
    -- ========================================================================

    max_voltage_subsystem_no    SMALLINT    NULL
                                COMMENT "最高电压电池子系统号 (2006001)",

    max_voltage_cell_no         SMALLINT    NULL
                                COMMENT "最高电压电池单体代号 (2006002)",

    max_cell_voltage            DOUBLE      NULL
                                COMMENT "单体电压最高值 V 0.001精度 (2006003)",

    min_voltage_subsystem_no    SMALLINT    NULL
                                COMMENT "最低电压电池子系统号 (2006004)",

    min_voltage_cell_no         SMALLINT    NULL
                                COMMENT "最低电压电池单体代号 (2006005)",

    min_cell_voltage            DOUBLE      NULL
                                COMMENT "单体电压最低值 V 0.001精度 (2006006)",

    max_temp_subsystem_no       SMALLINT    NULL
                                COMMENT "最高温度子系统号 (2006007)",

    max_temp_probe_no           SMALLINT    NULL
                                COMMENT "最高温度探针序号 (2006008)",

    max_temperature             SMALLINT    NULL
                                COMMENT "最高温度值 ℃ 偏移-40 (2006009)",

    min_temp_subsystem_no       SMALLINT    NULL
                                COMMENT "最低温度子系统号 (2006010)",

    min_temp_probe_no           SMALLINT    NULL
                                COMMENT "最低温度探针序号 (2006011)",

    min_temperature             SMALLINT    NULL
                                COMMENT "最低温度值 ℃ 偏移-40 (2006012)",

    -- ========================================================================
    -- 可充电储能装置电压 2008xxx
    --
    -- 子系统电压信息列表为大字符串（每子系统：编号+电压值），
    -- 单子系统维度分析在 DWD 层拆明细。
    -- ========================================================================

    ess_voltage_subsystem_count SMALLINT    NULL
                                COMMENT "可充电储能子系统个数(电压) (2008001)",

    ess_voltage_list            STRING      NULL
                                COMMENT "可充电储能子系统电压信息列表 原始字符串 (2008002)",

    -- ========================================================================
    -- 可充电储能装置温度 2009xxx
    -- ========================================================================

    ess_temp_subsystem_count    SMALLINT    NULL
                                COMMENT "可充电储能子系统个数(温度) (2009001)",

    ess_temp_list               STRING      NULL
                                COMMENT "可充电储能子系统温度信息列表 原始字符串 (2009002)",

    -- ========================================================================
    -- TBOX 自定义 2010xxx（码表全量打平，双链路IP端口 2010022/2010023 留 payload）
    -- ========================================================================

    power_supply_flag           TINYINT     NULL
                                COMMENT "设备供电标识 (2010003)",

    tbox_version                STRING      NULL
                                COMMENT "TBOX版本 原始值 (2010004)",

    relay_status                TINYINT     NULL
                                COMMENT "继电器状态 (2010005)",

    terminal_bind_status        TINYINT     NULL
                                COMMENT "终端绑定状态 (2010006)",

    controller_version_status   TINYINT     NULL
                                COMMENT "车辆控制器版本状态 (2010007)",

    lock_status                 TINYINT     NULL
                                COMMENT "锁车状态 (2010008)",

    charge_energy               INT         NULL
                                COMMENT "本次充电电量 kWh (2010009)",

    vcu_version                 STRING      NULL
                                COMMENT "VCU版本号 原始值 (2010010)",

    charge_mode_flag            TINYINT     NULL
                                COMMENT "车辆快慢充标志 (2010011)",

    lock_flag                   TINYINT     NULL
                                COMMENT "锁车标志 原始值 (2010012)",

    fault_count                 SMALLINT    NULL
                                COMMENT "故障数 (2010013)",

    vehicle_model_id            STRING      NULL
                                COMMENT "车型识别(车型ID) 原始值 (2010014)",

    tbox_work_status            TINYINT     NULL
                                COMMENT "TBOX工作状态 (2010015)",

    vehicle_on_signal           TINYINT     NULL
                                COMMENT "车辆ON信号 0/1 (2010016)",

    vehicle_charge_signal       TINYINT     NULL
                                COMMENT "车辆CHARGE信号 0/1 (2010017)",

    can_wakeup_status           TINYINT     NULL
                                COMMENT "TBOX CAN唤醒状态 (2010018)",

    low_battery_voltage         DOUBLE      NULL
                                COMMENT "车辆小电瓶电压 V (2010020)",

    tbox_battery_voltage        DOUBLE      NULL
                                COMMENT "TBOX内部电池电压 V (2010021)",

    dual_link_status            TINYINT     NULL
                                COMMENT "双链路状态 (2010024)",

    maintenance_tip             TINYINT     NULL
                                COMMENT "保养提示 (2010025)",

    payment_tip                 TINYINT     NULL
                                COMMENT "缴费提示 (2010026)",

    inspection_tip              TINYINT     NULL
                                COMMENT "年审提示 (2010027)",

    battery_removal_status      TINYINT     NULL
                                COMMENT "动力电池拆除状态 (2010044)",

    gps_mode                    TINYINT     NULL
                                COMMENT "GPS模式定位 (2010045)",

    gnss_satellite_count        SMALLINT    NULL
                                COMMENT "GNSS可用卫星数 (2010046)",

    battery_soh                 TINYINT     NULL
                                COMMENT "动力电池SOH % (2010051)",

    ac_work_status              TINYINT     NULL
                                COMMENT "空调工作状态 (2010052)",

    water_pump_fault            TINYINT     NULL
                                COMMENT "水泵故障 (2010053)",

    fast_charge_relay_fault     TINYINT     NULL
                                COMMENT "快充继电器粘连故障 (2010054)",

    vacuum_degree               INT         NULL
                                COMMENT "真空度 原始值 (2010055)",

    gnss_work_status            TINYINT     NULL
                                COMMENT "GNSS工作状态 (2010056)",

    emmc_work_status            TINYINT     NULL
                                COMMENT "EMMC工作状态 (2010057)",

    flash_work_status           TINYINT     NULL
                                COMMENT "Flash工作状态 (2010058)",

    ethernet_status             TINYINT     NULL
                                COMMENT "以太网通讯状态 (2010059)",

    -- ========================================================================
    -- 通用报警 2013xxx
    -- ========================================================================

    max_alarm_level             TINYINT     NULL
                                COMMENT "最高报警等级 0=无故障 1~3级 (2013001)",

    general_alarm_flags         BIGINT      NULL
                                COMMENT "通用报警标志位 64bit (2013002)",

    -- ========================================================================
    -- 长尾字段 VARIANT Sparse
    --
    --   Path 1 ~ Path 64   → Subcolumn 化（有独立类型和统计信息，查询快）
    --   Path 65 ~ Path N   → Sparse（数据仍然保存，只是不做子列化）
    --
    -- 注意：64 是 Subcolumn 化上限，不是 payload 只能存 64 个字段。
    -- ========================================================================

    payload VARIANT<
        PROPERTIES(
            'variant_max_subcolumns_count' = '64',
            'variant_enable_typed_paths_to_sparse' = 'true'
        )
    >
                                COMMENT "长尾字段JSON，最多64个Path进行Subcolumn化，其余进入Sparse",

    -- ========================================================================
    -- 入库元数据
    -- ========================================================================

    load_time                   DATETIME    NULL
                                DEFAULT CURRENT_TIMESTAMP
                                COMMENT "入库时间"
)

-- ============================================================================
-- 3. Key（必须是建表列的连续前缀：ds → dev_code）
-- ============================================================================

DUPLICATE KEY(ds, dev_code)

-- ============================================================================
-- 4. Partition（分区由动态分区策略自动管理）
-- ============================================================================

PARTITION BY RANGE(ds) ()

-- ============================================================================
-- 5. Distribution（同设备数据落同桶，单车时间线查询少 shuffle）
-- ============================================================================

DISTRIBUTED BY HASH(dev_code) BUCKETS AUTO

-- ============================================================================
-- 6. Properties
-- ============================================================================

PROPERTIES
(
    -- 动态分区：每天一个分区，保留最近 90 天，预建未来 3 天
    "dynamic_partition.enable"    = "true",
    "dynamic_partition.time_unit" = "DAY",
    "dynamic_partition.start"     = "-90",
    "dynamic_partition.end"       = "3",
    "dynamic_partition.prefix"    = "p",

    -- 副本：测试环境 1 副本；生产环境改为 "3"
    "replication_num"             = "1",

    -- V3 存储格式：VARIANT 子列较多时推荐
    "storage_format"              = "V3"
);

-- ============================================================================
-- 7. 查询示例
-- ============================================================================

-- 7.1 冷藏车字段（长尾，payload 提取）
-- SELECT
--     vin_code,
--     data_time,
--     CAST(payload -> '$.2011010' AS SMALLINT) AS reefer_set_temp,
--     CAST(payload -> '$.2011011' AS SMALLINT) AS reefer_actual_temp
-- FROM dst_iot_dw.ods_vehicle_report
-- WHERE ds = '2026-09-28'
--   AND payload -> '$.2011011' IS NOT NULL;

-- 7.2 电池报警等级（长尾，payload 提取）
-- SELECT
--     vin_code,
--     data_time,
--     CAST(payload -> '$.2012011' AS CHAR(2)) AS overheat_alarm_level
-- FROM dst_iot_dw.ods_vehicle_report
-- WHERE ds = '2026-09-28'
--   AND CAST(payload -> '$.2012011' AS CHAR(2)) IN ('01', '02', '03');

-- 7.3 故障代码（长尾，payload 提取）
-- SELECT
--     vin_code,
--     data_time,
--     payload -> '$.2013004' AS fault_code
-- FROM dst_iot_dw.ods_vehicle_report
-- WHERE ds = '2026-09-28'
--   AND payload -> '$.2013004' IS NOT NULL;

-- 7.4 位置 + 热字段（打平列直查）
-- SELECT
--     vin_code,
--     data_time,
--     longitude,
--     latitude,
--     speed,
--     total_mileage,
--     soc
-- FROM dst_iot_dw.ods_vehicle_report
-- WHERE ds = CURRENT_DATE();

-- ============================================================================
-- 8. 运维查询
-- ============================================================================

-- 查看动态分区状态：
-- SHOW DYNAMIC PARTITION TABLES;

-- 查看最终表结构（含 VARIANT 属性）：
-- SHOW CREATE TABLE dst_iot_dw.ods_vehicle_report;
```

## 六、Connector 完整配置（02 号工件）

```json
{
  "name": "selectdb-sink-ods-vehicle-report",
  "config": {
    "connector.class": "org.apache.doris.kafka.connector.DorisSinkConnector",
    "tasks.max": "4",

    "topics": "iot-vehicle-report",
    "doris.topic2table.map": "iot-vehicle-report:ods_vehicle_report",

    "buffer.count.records": "20000",
    "buffer.flush.time": "30",
    "buffer.size.bytes": "67108864",

    "doris.urls": "selectdb-cn-xxxx.selectdbfe.rds.aliyuncs.com",
    "doris.http.port": "8080",
    "doris.query.port": "9030",
    "doris.user": "admin",
    "doris.password": "YOUR_PASSWORD",
    "doris.database": "dst_iot_dw",

    "key.converter": "org.apache.kafka.connect.storage.StringConverter",
    "value.converter": "org.apache.kafka.connect.json.JsonConverter",
    "value.converter.schemas.enable": "false",

    "sink.properties.format": "json",
    "sink.properties.read_json_by_line": "true",
    "sink.properties.strip_outer_array": "false",
    "sink.properties.strict_mode": "false",
    "sink.properties.max_filter_ratio": "0.01",
    "sink.properties.columns": "ds,dev_code,vin_code,data_time,cmd,source,vehicle_status,charge_status,speed,total_mileage,total_voltage,total_current,soc,dcdc_status,insulation_resistance,accel_pedal_position,brake_pedal_status,gear,has_braking_force,has_driving_force,motor_status_list,motor_controller_temp_list,motor_speed_list,motor_torque_list,motor_temp_list,motor_controller_voltage_list,motor_controller_current_list,longitude,latitude,positioning_status,latitude_hemisphere,longitude_hemisphere,max_voltage_subsystem_no,max_voltage_cell_no,max_cell_voltage,min_voltage_subsystem_no,min_voltage_cell_no,min_cell_voltage,max_temp_subsystem_no,max_temp_probe_no,max_temperature,min_temp_subsystem_no,min_temp_probe_no,min_temperature,ess_voltage_subsystem_count,ess_voltage_list,ess_temp_subsystem_count,ess_temp_list,power_supply_flag,tbox_version,relay_status,terminal_bind_status,controller_version_status,lock_status,charge_energy,vcu_version,charge_mode_flag,lock_flag,fault_count,vehicle_model_id,tbox_work_status,vehicle_on_signal,vehicle_charge_signal,can_wakeup_status,low_battery_voltage,tbox_battery_voltage,dual_link_status,maintenance_tip,payment_tip,inspection_tip,battery_removal_status,gps_mode,gnss_satellite_count,battery_soh,ac_work_status,water_pump_fault,fast_charge_relay_fault,vacuum_degree,gnss_work_status,emmc_work_status,flash_work_status,ethernet_status,max_alarm_level,general_alarm_flags,payload",
    "sink.properties.jsonpaths": "[\"$.ds\",\"$.device_no\",\"$.vin\",\"$.data_time\",\"$.cmd\",\"$.source\",\"$.2001001\",\"$.2001002\",\"$.2001004\",\"$.2001005\",\"$.2001006\",\"$.2001007\",\"$.2001008\",\"$.2001009\",\"$.2001011\",\"$.2001012\",\"$.2001013\",\"$.200101001\",\"$.200101002\",\"$.200101003\",\"$.2002002\",\"$.2002003\",\"$.2002004\",\"$.2002005\",\"$.2002006\",\"$.2002007\",\"$.2002008\",\"$.2005002\",\"$.2005003\",\"$.200500101\",\"$.200500102\",\"$.200500103\",\"$.2006001\",\"$.2006002\",\"$.2006003\",\"$.2006004\",\"$.2006005\",\"$.2006006\",\"$.2006007\",\"$.2006008\",\"$.2006009\",\"$.2006010\",\"$.2006011\",\"$.2006012\",\"$.2008001\",\"$.2008002\",\"$.2009001\",\"$.2009002\",\"$.2010003\",\"$.2010004\",\"$.2010005\",\"$.2010006\",\"$.2010007\",\"$.2010008\",\"$.2010009\",\"$.2010010\",\"$.2010011\",\"$.2010012\",\"$.2010013\",\"$.2010014\",\"$.2010015\",\"$.2010016\",\"$.2010017\",\"$.2010018\",\"$.2010020\",\"$.2010021\",\"$.2010024\",\"$.2010025\",\"$.2010026\",\"$.2010027\",\"$.2010044\",\"$.2010045\",\"$.2010046\",\"$.2010051\",\"$.2010052\",\"$.2010053\",\"$.2010054\",\"$.2010055\",\"$.2010056\",\"$.2010057\",\"$.2010058\",\"$.2010059\",\"$.2013001\",\"$.2013002\",\"$\"]",

    "errors.tolerance": "all",
    "errors.deadletterqueue.topic.name": "ods-vehicle-report-dlt",
    "errors.deadletterqueue.context.headers.enable": "true",
    "errors.deadletterqueue.topic.replication.factor": "2",

    "consumer.security.protocol": "SASL_SSL",
    "consumer.sasl.mechanism": "SCRAM-SHA-512",
    "consumer.sasl.jaas.config": "org.apache.kafka.common.security.scram.ScramLoginModule required username=\"kafka-user\" password=\"YOUR_KAFKA_PASSWORD\";"
  }
}
```

## 七、Worker 消费者配置片段（03 号工件）

`connect-distributed.properties` 需补充的片段（SASL_SSL Kafka 集群）：

```properties
# ============================================================================
# connect-distributed.properties 需要补充的片段（SASL_SSL Kafka 集群）
#
# 说明：本场景生产端无 Message Key、无 Header，消息体为 JSON 字符串，
# 所以 key.converter 用 StringConverter 即可；value 用 JsonConverter 且不启用 schema
# （生产端是裸 fastjson JSON，不是 Kafka Connect 带 schema 的 envelope）。
# snappy 是 broker 端压缩，客户端无需显式配置。
# ============================================================================

# ---- 基础 ----
bootstrap.servers=kafka-broker1:9093,kafka-broker2:9093,kafka-broker3:9093
group.id=connect-cluster
plugin.path=/opt/kafka/plugins
listeners=http://0.0.0.0:8083

# ---- 内嵌消费者的 SASL_SSL 认证（也可按 02 号 connector 配置用 consumer.* 前缀在任务级覆盖）----
security.protocol=SASL_SSL
sasl.mechanism=SCRAM-SHA-512
sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required username="kafka-user" password="YOUR_KAFKA_PASSWORD";
# 如 broker 使用自签证书，追加：
# ssl.truststore.location=/path/to/kafka.client.truststore.jks
# ssl.truststore.password=truststore密码

# ---- 序列化 ----
key.converter=org.apache.kafka.connect.storage.StringConverter
value.converter=org.apache.kafka.connect.json.JsonConverter
value.converter.schemas.enable=false

# ---- 稳定性 ----
# Stream Load 攒批期间消费者不许被踢出消费组，把 rebalance 间隔调大
max.poll.interval.ms=1800000
consumer.max.poll.interval.ms=1800000
offset.flush.interval.ms=10000

# ---- 存储 topic（生产环境副本因子建议 >= 3）----
config.storage.topic=connect-configs
offset.storage.topic=connect-offsets
status.storage.topic=connect-status
config.storage.replication.factor=3
offset.storage.replication.factor=3
status.storage.replication.factor=3
```

## 八、三处强耦合警告

<div class="callout callout-warning" markdown="1">
**⚠️ 改表必同步**

建表列序、connector `sink.properties.columns`、`sink.properties.jsonpaths` 三者一一对应（columns 按位对应 jsonpaths，最后一项 `"$"` 把整条 JSON 根对象映射进 `payload`）。本次定稿已把表列名从 `device_no`/`vin` 改为 `dev_code`/`vin_code`——**源 JSON 字段名没变**（jsonpaths 仍是 `$.device_no`/`$.vin`），只有表列名变了。`load_time` 不进映射，由 DEFAULT 自动填充。
一致性可用脚本自检：`99-Meta/tmp/verify_connector.py`（校验列数对齐 + 列名归属 + NOT NULL 覆盖）。
</div>

## 九、部署步骤

```bash
# 1. 放置插件：下载 doris-kafka-connector jar（SelectDB 控制台或 Doris 社区）到 KAFKA_HOME/libs 或 plugin.path

# 2. 按 03 片段修改 connect-distributed.properties，启动 Connect 集群
$KAFKA_HOME/bin/connect-distributed.sh -daemon $KAFKA_HOME/config/connect-distributed.properties

# 3. 提交 connector（先改 02 文件里的 topic、连接地址、账号密码）
curl -i http://127.0.0.1:8083/connectors -H "Content-Type: application/json" -X POST \
  -d @02_selectdb_kafka_connector.json

# 4. 验证
curl -s http://127.0.0.1:8083/connectors/selectdb-sink-ods-vehicle-report/status
mysql -h<fe> -P9030 -uroot -e "SELECT count(*) FROM dst_iot_dw.ods_vehicle_report WHERE ds = CURRENT_DATE();"
```

吞吐调优（按需追加到 connector 配置）：

| 参数 | 作用 |
| --- | --- |
| `"sink.properties.group_commit": "async_mode"` | 服务端攒批，削峰明显 |
| `"load.model": "copy_into"` | 海量数据先落对象存储再批量导入（云实例适用） |
| `"auto.redirect": "true"` | Stream Load 经 FE 重定向到 BE，减少元数据往返 |

死信队列：已配 `errors.tolerance=all` + DLT topic（`ods-vehicle-report-dlt`），脏数据不阻塞主链路，消费 DLT 做告警即可。

## 十、常用查询模板

```sql
-- 单车时间线（热字段，走 ds+dev_code 前缀索引；注意列名是 vin_code）
SELECT data_time, speed, soc, total_voltage, total_current, longitude, latitude
FROM dst_iot_dw.ods_vehicle_report
WHERE vin_code = 'LXXXXXXXXXXXX' AND ds BETWEEN '2026-09-01' AND '2026-09-28'
ORDER BY data_time;

-- 冷藏车温度合规（长尾，payload 提取）
SELECT vin_code, data_time,
       CAST(payload -> '$.2011010' AS SMALLINT) AS set_temp,
       CAST(payload -> '$.2011011' AS SMALLINT) AS actual_temp
FROM dst_iot_dw.ods_vehicle_report
WHERE ds = CURRENT_DATE()
  AND payload -> '$.2011011' IS NOT NULL
  AND abs(CAST(payload -> '$.2011011' AS SMALLINT) - CAST(payload -> '$.2011010' AS SMALLINT)) > 5;

-- 故障车辆清单（长尾 + 热字段混查）
SELECT vin_code, data_time, max_alarm_level, fault_count,
       payload -> '$.2013004' AS fault_code
FROM dst_iot_dw.ods_vehicle_report
WHERE ds = CURRENT_DATE()
  AND payload -> '$.2013004' IS NOT NULL;
```

## 十一、运维速查

```sql
SHOW DYNAMIC PARTITION TABLES;                 -- 动态分区状态（90 天保留是否生效）
SHOW CREATE TABLE dst_iot_dw.ods_vehicle_report;      -- 最终表结构（含 VARIANT 属性）
SELECT ds, count(*) FROM dst_iot_dw.ods_vehicle_report GROUP BY ds ORDER BY ds DESC;  -- 各分区行数
```

## 参考

- [车联网数据仓库建设方案](/2026/09/23/iov-data-warehouse-design/)（总方案 v1.1）
- **车联网里程数据处理与计算方案**
