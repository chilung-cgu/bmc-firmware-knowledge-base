# PLDMSensor（PLDM 感測器）

本文說明 OpenBIC 的 PLDM 感測器架構，包括 PDR 定義與感測器讀取流程。

---

## 概述

PLDM Platform Monitoring (Type 2) 提供標準化的感測器介面：

- **PDR (Platform Descriptor Record)**：描述感測器/Effecter 屬性
- **Numeric Sensor**：數值感測器（溫度、電壓等）
- **State Sensor**：狀態感測器（存在、健康狀態等）

---

## PDR 類型

### 常見 PDR 類型

| 類型 | 代碼 | 說明 |
|------|------|------|
| Terminus Locator | 1 | Terminus 位置資訊 |
| Numeric Sensor | 2 | 數值感測器 PDR |
| Numeric Sensor Aux Names | 3 | 感測器輔助名稱 |
| State Sensor | 4 | 狀態感測器 PDR |
| Numeric Effecter | 9 | 數值 Effecter PDR |
| State Effecter | 11 | 狀態 Effecter PDR |
| Entity Association | 15 | 實體關聯 |
| FRU Record Set | 20 | FRU 記錄集 |

---

## PLDM 感測器表

### 感測器定義

```c
// plat_pldm_sensor.c
pldm_sensor_info plat_pldm_sensor_table[] = {
    // 溫度感測器
    {
        .sensor_id = PLDM_SENSOR_ID_TEMP_CPU,
        .sensor_type = PLDM_SENSOR_TYPE_TEMPERATURE,
        .base_unit = PLDM_UNIT_CELSIUS,
        .unit_modifier = -3,  // milli-celsius
        .sensor_data_size = PLDM_SENSOR_DATA_SIZE_SINT32,
        .resolution = 1,
        .read_func = plat_pldm_sensor_temp_read,
    },
    
    // 電壓感測器
    {
        .sensor_id = PLDM_SENSOR_ID_VR_CPU0_VOUT,
        .sensor_type = PLDM_SENSOR_TYPE_VOLTAGE,
        .base_unit = PLDM_UNIT_VOLT,
        .unit_modifier = -3,  // milli-volt
        .sensor_data_size = PLDM_SENSOR_DATA_SIZE_SINT32,
        .read_func = plat_pldm_sensor_vr_read,
    },
    
    // 功率感測器
    {
        .sensor_id = PLDM_SENSOR_ID_CPU_POWER,
        .sensor_type = PLDM_SENSOR_TYPE_POWER,
        .base_unit = PLDM_UNIT_WATT,
        .unit_modifier = 0,
        .read_func = plat_pldm_sensor_power_read,
    },
};

const uint8_t PLAT_PLDM_SENSOR_COUNT = ARRAY_SIZE(plat_pldm_sensor_table);
```

---

## PDR 結構

### Numeric Sensor PDR

```c
// pldm_monitor.h
struct pldm_numeric_sensor_pdr {
    struct pldm_pdr_common_header hdr;
    uint16_t terminus_handle;
    uint16_t sensor_id;
    uint16_t entity_type;
    uint16_t entity_instance;
    uint16_t container_id;
    uint8_t sensor_init;
    uint8_t sensor_auxiliary_names_pdr;
    uint8_t base_unit;
    int8_t unit_modifier;
    uint8_t rate_unit;
    uint8_t base_oem_unit_handle;
    uint8_t aux_unit;
    int8_t aux_unit_modifier;
    uint8_t aux_rate_unit;
    uint8_t rel;
    uint8_t aux_oem_unit_handle;
    uint8_t is_linear;
    uint8_t sensor_data_size;
    float resolution;
    float offset;
    uint16_t accuracy;
    uint8_t plus_tolerance;
    uint8_t minus_tolerance;
    // 閾值和範圍資料...
};
```

### State Sensor PDR

```c
struct pldm_state_sensor_pdr {
    struct pldm_pdr_common_header hdr;
    uint16_t terminus_handle;
    uint16_t sensor_id;
    uint16_t entity_type;
    uint16_t entity_instance;
    uint16_t container_id;
    uint8_t sensor_init;
    uint8_t sensor_auxiliary_names_pdr;
    uint8_t composite_sensor_count;
    // 狀態集資料...
};
```

---

## 感測器讀取

### GetSensorReading 命令

```c
// pldm_monitor.c
uint8_t pldm_get_sensor_reading(pldm_msg *msg)
{
    struct pldm_get_sensor_reading_req *req = 
        (struct pldm_get_sensor_reading_req *)msg->buf;
    
    uint16_t sensor_id = req->sensor_id;
    
    // 查找感測器
    pldm_sensor_info *sensor = find_sensor_by_id(sensor_id);
    if (!sensor) {
        return PLDM_ERROR_INVALID_SENSOR_ID;
    }
    
    // 讀取感測器值
    int reading = 0;
    uint8_t status = sensor->read_func(sensor, &reading);
    
    // 填充回應
    struct pldm_get_sensor_reading_resp resp = {
        .completion_code = PLDM_SUCCESS,
        .sensor_data_size = sensor->sensor_data_size,
        .sensor_operational_state = PLDM_SENSOR_ENABLED,
        .sensor_event_message_enable = PLDM_NO_EVENT_GENERATION,
        .present_state = PLDM_SENSOR_NORMAL,
        .previous_state = PLDM_SENSOR_UNKNOWN,
        .event_state = PLDM_SENSOR_UNKNOWN,
    };
    
    // 根據資料大小填充讀數
    memcpy(resp.reading, &reading, sizeof(reading));
    
    return PLDM_SUCCESS;
}
```

### 平台讀取函數

```c
// plat_pldm_sensor.c
uint8_t plat_pldm_sensor_temp_read(pldm_sensor_info *sensor, int *reading)
{
    // 從本地感測器快取讀取
    sensor_cfg *cfg = find_sensor_config(sensor->local_sensor_num);
    if (!cfg) {
        return PLDM_SENSOR_NOT_PRESENT;
    }
    
    if (cfg->cache_status != SENSOR_READ_SUCCESS) {
        return PLDM_SENSOR_FAILED;
    }
    
    *reading = cfg->cache;
    return PLDM_SUCCESS;
}
```

---

## 感測器表初始化

```c
// plat_init.c
void plat_init_pldm_sensor_table(void)
{
    // 載入 PDR 表
    pldm_load_numeric_sensor_pdr(plat_pldm_sensor_table, 
                                  PLAT_PLDM_SENSOR_COUNT);
    
    // 建立感測器 ID 到本地感測器的映射
    for (int i = 0; i < PLAT_PLDM_SENSOR_COUNT; i++) {
        pldm_sensor_info *sensor = &plat_pldm_sensor_table[i];
        sensor->local_sensor_cfg = find_sensor_by_num(sensor->local_sensor_num);
    }
}
```

---

## 感測器事件

### Event Generation

```c
// pldm_monitor.c
void pldm_generate_sensor_event(uint16_t sensor_id, 
                                 uint8_t sensor_event_class,
                                 uint8_t *event_data,
                                 uint16_t event_data_len)
{
    pldm_msg msg = { 0 };
    msg.hdr.pldm_type = PLDM_TYPE_PLAT_MON;
    msg.hdr.cmd = PLDM_PLATFORM_EVENT_MESSAGE;
    msg.hdr.rq = PLDM_REQUEST;
    
    struct pldm_platform_event_message_req req = {
        .format_version = 0x01,
        .tid = plat_pldm_get_tid(),
        .event_class = PLDM_SENSOR_EVENT,
    };
    
    struct pldm_sensor_event_data event = {
        .sensor_id = sensor_id,
        .sensor_event_class = sensor_event_class,
    };
    
    // 發送事件到 BMC
    mctp_pldm_send_msg(find_mctp_by_bus(bmc_bus), &msg);
}
```

---

## 感測器與本地框架整合

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   BMC (PLDM Requester)                                         │
│        │                                                        │
│        │ GetSensorReading                                       │
│        ▼                                                        │
│   ┌────────────────────────────────────────────────────────┐   │
│   │              PLDM Sensor Interface                      │   │
│   │                                                         │   │
│   │   pldm_sensor_info[] ◄─────► Sensor ID Mapping         │   │
│   │        │                                                │   │
│   │        ▼                                                │   │
│   │   plat_pldm_sensor_read()                               │   │
│   │        │                                                │   │
│   └────────┼────────────────────────────────────────────────┘   │
│            │                                                     │
│            ▼                                                     │
│   ┌────────────────────────────────────────────────────────┐   │
│   │           Local Sensor Framework                        │   │
│   │                                                         │   │
│   │   sensor_cfg[]  ◄─────►  Sensor Poll Thread            │   │
│   │        │                                                │   │
│   │        ▼                                                │   │
│   │   sensor_dev_xxx_read() ──► I2C/I3C ──► Hardware       │   │
│   └────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 相關文件

- [SensorFramework](SensorFramework.md) - 本地感測器框架
- [PLDMOverview](PLDMOverview.md) - PLDM 協議
- [PLDMMonitor](PLDMMonitor.md) - PLDM 監控

---

*返回 [Home](Home.md)*
