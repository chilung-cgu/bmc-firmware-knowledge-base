# PLDMMonitor（PLDM 監控）

本文說明 OpenBIC 的 PLDM Platform Monitoring 功能，包括 State Effecter 與事件處理。

---

## 概述

PLDM Platform Monitoring (Type 2) 提供：

- **Numeric Sensor**：數值感測器讀取
- **State Sensor**：狀態感測器查詢
- **Numeric Effecter**：數值控制
- **State Effecter**：狀態設定
- **Event Message**：事件通知

---

## State Effecter

### 概念

State Effecter 用於控制平台狀態，例如：
- GPIO 輸出
- LED 控制
- 電源控制
- 風扇模式

### Effecter 定義

```c
// plat_pldm_monitor.c
pldm_state_effecter_info plat_state_effecter_table[] = {
    // GPIO Effecter
    {
        .effecter_id = EFFECTER_ID_GPIO_HIGH_BYTE | BIC_READY_R,
        .entity_type = PLDM_ENTITY_FIRMWARE_UPDATE_SERVICE,
        .state_set_id = PLDM_STATE_SET_GPIO_STATE,
        .possible_states = { GPIO_STATE_LOW, GPIO_STATE_HIGH },
        .set_func = plat_effecter_gpio_set,
    },
    
    // Power Control Effecter
    {
        .effecter_id = EFFECTER_ID_POWER_CONTROL,
        .entity_type = PLDM_ENTITY_POWER_SUPPLY,
        .state_set_id = PLDM_STATE_SET_OPERATIONAL_STATE,
        .set_func = plat_effecter_power_set,
    },
};

const uint8_t PLAT_PLDM_MAX_STATE_EFFECTER_IDX = 
    ARRAY_SIZE(plat_state_effecter_table);
```

### Set State Effecter States

```c
// pldm_monitor.c
uint8_t pldm_set_state_effecter_states(pldm_msg *msg)
{
    struct pldm_set_state_effecter_states_req *req = 
        (struct pldm_set_state_effecter_states_req *)msg->buf;
    
    uint16_t effecter_id = req->effecter_id;
    uint8_t composite_effecter_count = req->composite_effecter_count;
    
    // 查找 Effecter
    pldm_state_effecter_info *effecter = find_effecter_by_id(effecter_id);
    if (!effecter) {
        return PLDM_ERROR_INVALID_EFFECTER_ID;
    }
    
    // 設定狀態
    for (int i = 0; i < composite_effecter_count; i++) {
        uint8_t set_request = req->set_info[i].set_request;
        uint8_t state = req->set_info[i].effecter_state;
        
        if (set_request == PLDM_REQUEST_SET) {
            effecter->set_func(effecter, state);
        }
    }
    
    return PLDM_SUCCESS;
}
```

### GPIO Effecter 實作

```c
// plat_pldm_monitor.c
uint8_t plat_effecter_gpio_set(pldm_state_effecter_info *effecter, 
                                uint8_t state)
{
    // Effecter ID 包含 GPIO 編號
    uint8_t gpio_num = effecter->effecter_id & 0xFF;
    
    switch (state) {
    case GPIO_STATE_LOW:
        gpio_set(gpio_num, GPIO_LOW);
        break;
    case GPIO_STATE_HIGH:
        gpio_set(gpio_num, GPIO_HIGH);
        break;
    default:
        return PLDM_ERROR_INVALID_STATE_VALUE;
    }
    
    return PLDM_SUCCESS;
}
```

---

## Numeric Effecter

### 定義

```c
pldm_numeric_effecter_info plat_numeric_effecter_table[] = {
    // 風扇 PWM 控制
    {
        .effecter_id = EFFECTER_ID_FAN_PWM,
        .entity_type = PLDM_ENTITY_FAN,
        .effecter_data_size = PLDM_EFFECTER_DATA_SIZE_UINT8,
        .max_settable = 100,
        .min_settable = 0,
        .set_func = plat_effecter_fan_pwm_set,
    },
};
```

### Set Numeric Effecter Value

```c
uint8_t pldm_set_numeric_effecter_value(pldm_msg *msg)
{
    struct pldm_set_numeric_effecter_value_req *req = 
        (struct pldm_set_numeric_effecter_value_req *)msg->buf;
    
    uint16_t effecter_id = req->effecter_id;
    uint8_t effecter_data_size = req->effecter_data_size;
    
    pldm_numeric_effecter_info *effecter = 
        find_numeric_effecter_by_id(effecter_id);
    if (!effecter) {
        return PLDM_ERROR_INVALID_EFFECTER_ID;
    }
    
    // 根據資料大小解析數值
    int32_t value = 0;
    switch (effecter_data_size) {
    case PLDM_EFFECTER_DATA_SIZE_UINT8:
        value = req->effecter_value[0];
        break;
    case PLDM_EFFECTER_DATA_SIZE_SINT16:
        value = *(int16_t *)req->effecter_value;
        break;
    // ...
    }
    
    // 設定數值
    return effecter->set_func(effecter, value);
}
```

---

## PLDM Event Message

### 事件類型

| 類型 | 代碼 | 說明 |
|------|------|------|
| Sensor Event | 0x00 | 感測器事件 |
| Effecter Event | 0x01 | Effecter 事件 |
| Redfish Task Event | 0x02 | Redfish 任務 |
| Heartbeat Timer | 0x03 | 心跳計時器 |
| CPER Event | 0x04 | 平台錯誤記錄 |

### 發送事件

```c
// pldm_monitor.c
uint8_t pldm_platform_event_message(uint8_t tid,
                                     uint8_t event_class,
                                     uint8_t *event_data,
                                     uint16_t event_data_len)
{
    pldm_msg msg = { 0 };
    msg.ext_params.ep = MCTP_EID_BMC;
    msg.hdr.rq = PLDM_REQUEST;
    msg.hdr.pldm_type = PLDM_TYPE_PLAT_MON;
    msg.hdr.cmd = PLDM_PLATFORM_EVENT_MESSAGE;
    
    struct pldm_platform_event_message_req req = {
        .format_version = 0x01,
        .tid = tid,
        .event_class = event_class,
    };
    
    // 組裝訊息
    uint16_t total_len = sizeof(req) + event_data_len;
    uint8_t *buf = malloc(total_len);
    memcpy(buf, &req, sizeof(req));
    memcpy(buf + sizeof(req), event_data, event_data_len);
    
    msg.buf = buf;
    msg.len = total_len;
    
    // 發送到 BMC
    mctp_pldm_send_msg(find_mctp_by_bus(bmc_bus), &msg);
    
    free(buf);
    return PLDM_SUCCESS;
}
```

### 感測器事件格式

```c
struct pldm_sensor_event_data {
    uint16_t sensor_id;
    uint8_t sensor_event_class;
    union {
        // Numeric Sensor
        struct {
            uint8_t event_state;
            uint8_t previous_event_state;
            uint8_t sensor_data_size;
            uint8_t present_reading[4];
        } numeric;
        
        // State Sensor
        struct {
            uint8_t sensor_offset;
            uint8_t event_state;
            uint8_t previous_event_state;
        } state;
    };
};
```

---

## Sensor 監控線程

```c
// pldm_sensor.c
void pldm_sensor_monitor_thread(void *arg0, void *arg1, void *arg2)
{
    while (1) {
        for (int i = 0; i < sensor_count; i++) {
            pldm_sensor_info *sensor = &sensor_table[i];
            
            // 讀取感測器
            int reading = 0;
            uint8_t status = sensor->read_func(sensor, &reading);
            
            // 檢查閾值
            if (sensor->threshold_enabled) {
                uint8_t event = check_threshold(sensor, reading);
                if (event != PLDM_SENSOR_NORMAL && 
                    event != sensor->current_event_state) {
                    // 發送事件
                    pldm_generate_sensor_event(
                        sensor->sensor_id,
                        PLDM_NUMERIC_SENSOR_STATE,
                        &event,
                        sizeof(event));
                }
                sensor->current_event_state = event;
            }
        }
        
        k_msleep(sensor_monitor_interval);
    }
}
```

---

## 閾值檢查

```c
uint8_t check_threshold(pldm_sensor_info *sensor, int reading)
{
    // 檢查上限臨界
    if (sensor->upper_critical_enabled && 
        reading >= sensor->upper_critical) {
        return PLDM_SENSOR_UPPERCRITICAL;
    }
    
    // 檢查上限警告
    if (sensor->upper_warning_enabled && 
        reading >= sensor->upper_warning) {
        return PLDM_SENSOR_UPPERWARNING;
    }
    
    // 檢查下限警告
    if (sensor->lower_warning_enabled && 
        reading <= sensor->lower_warning) {
        return PLDM_SENSOR_LOWERWARNING;
    }
    
    // 檢查下限臨界
    if (sensor->lower_critical_enabled && 
        reading <= sensor->lower_critical) {
        return PLDM_SENSOR_LOWERCRITICAL;
    }
    
    return PLDM_SENSOR_NORMAL;
}
```

---

## 相關文件

- [PLDMSensor](PLDMSensor.md) - PLDM 感測器
- [PLDMOverview](PLDMOverview.md) - PLDM 協議
- [InterruptHandling](InterruptHandling.md) - 事件處理

---

*返回 [Home](Home.md)*
