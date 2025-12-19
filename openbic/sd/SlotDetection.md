# SlotDetection（Slot 偵測機制）

本文說明 OpenBIC yv4-sd 的 Slot 偵測機制，包括 ADC 電壓讀取與 EID 映射。

---

## 概述

Yosemite V4 系統支援最多 8 個 Blade Slot，每個 Slot 的 SD BIC 需要：
1. 偵測所在的 Slot 位置
2. 根據位置設定唯一的 Endpoint ID (EID)
3. 配置對應的 Platform ID (PID)

---

## 偵測原理

### ADC 電壓分壓

每個 Slot 透過不同的電阻分壓比產生唯一電壓：

```
     VCC (2.5V)
       │
       ├── R_upper
       │
       ├─────────► ADC Input
       │
       ├── R_lower (Slot 特定)
       │
      GND
```

### Slot 電壓對應表

| Slot | 電壓範圍 | 說明 |
|------|----------|------|
| Slot 1 | < 0.15V | 接近 GND |
| Slot 2 | 0.15V ~ 0.45V | - |
| Slot 3 | 0.45V ~ 0.75V | - |
| Slot 4 | 0.75V ~ 1.05V | - |
| Slot 5 | 1.05V ~ 1.35V | - |
| Slot 6 | 1.35V ~ 1.625V | - |
| Slot 7 | 1.625V ~ 1.875V | - |
| Slot 8 | > 1.875V | 接近 VCC |

---

## 程式碼實作

### EID 映射表

```c
// plat_class.c
struct _SLOT_EID_MAPPING_TABLE {
    float low_voltage;
    float high_voltage;
    uint8_t condition;
    uint8_t eid;
    uint8_t pid;
};

struct _SLOT_EID_MAPPING_TABLE _slot_eid_mapping_table[] = {
    { 0, 0.15, LOWER, SLOT1_EID, SLOT1_PID },
    { 0.15, 0.45, RANGE, SLOT2_EID, SLOT2_PID },
    { 0.45, 0.75, RANGE, SLOT3_EID, SLOT3_PID },
    { 0.75, 1.05, RANGE, SLOT4_EID, SLOT4_PID },
    { 1.05, 1.35, RANGE, SLOT5_EID, SLOT5_PID },
    { 1.35, 1.625, RANGE, SLOT6_EID, SLOT6_PID },
    { 1.625, 1.875, RANGE, SLOT7_EID, SLOT7_PID },
    { 1.875, 2.5, HIGHER, SLOT8_EID, SLOT8_PID },
};
```

### 條件定義

```c
enum {
    LOWER = 0,   // 電壓 < high_voltage
    HIGHER = 1,  // 電壓 > low_voltage
    RANGE = 2,   // low_voltage <= 電壓 <= high_voltage
};
```

### ADC 讀取

```c
// plat_class.c
bool get_adc_voltage(int channel, float *voltage)
{
    const struct device *dev = device_get_binding("ADC");
    if (!dev) {
        return false;
    }
    
    struct adc_sequence sequence = { ... };
    int ret = adc_read(dev, &sequence);
    if (ret != 0) {
        return false;
    }
    
    // 讀取參考電壓設定
    uint32_t reg_value = sys_read32(0x7e6e20d0);
    float reference_voltage;
    
    switch (reg_value & (BIT(7) | BIT(6))) {
    case REF_VOL_2_5V:
        reference_voltage = 2.5;
        break;
    case REF_VOL_1_2V:
        reference_voltage = 1.2;
        break;
    case REF_VOL_EXT_HIGH:
    case REF_VOL_EXT_LOW:
    default:
        reference_voltage = 2.5;
        break;
    }
    
    // 計算實際電壓
    *voltage = (float)(adc_buffer * reference_voltage) / 1024.0f;
    
    return true;
}
```

### Slot EID 查詢

```c
// plat_class.c
uint8_t get_slot_eid(void)
{
    return slot_eid;
}

uint8_t get_slot_id(void)
{
    return slot_id;
}

static void init_slot_config(void)
{
    float voltage = 0.0f;
    
    // 讀取 ADC Channel 13
    if (!get_adc_voltage(ADC_CHANNEL_13, &voltage)) {
        LOG_ERR("Failed to read slot ADC");
        return;
    }
    
    LOG_INF("Slot ADC voltage: %.2fV", voltage);
    
    // 查找匹配的 Slot
    for (int i = 0; i < ARRAY_SIZE(_slot_eid_mapping_table); i++) {
        struct _SLOT_EID_MAPPING_TABLE *entry = &_slot_eid_mapping_table[i];
        bool match = false;
        
        switch (entry->condition) {
        case LOWER:
            match = (voltage < entry->high_voltage);
            break;
        case HIGHER:
            match = (voltage > entry->low_voltage);
            break;
        case RANGE:
            match = (voltage >= entry->low_voltage && 
                     voltage <= entry->high_voltage);
            break;
        }
        
        if (match) {
            slot_eid = entry->eid;
            slot_pid = entry->pid;
            slot_id = i + 1;  // Slot 1-8
            LOG_INF("Detected Slot %d, EID: 0x%02x", slot_id, slot_eid);
            return;
        }
    }
    
    // 未找到匹配 - 使用預設值
    LOG_WRN("Unknown slot configuration, using default");
    slot_eid = SLOT1_EID;
    slot_pid = SLOT1_PID;
    slot_id = 1;
}
```

---

## EID 分配

### 基礎 EID 定義

```c
// 每個 Slot 的基礎 EID
#define SLOT1_EID  0x10
#define SLOT2_EID  0x18
#define SLOT3_EID  0x20
#define SLOT4_EID  0x28
#define SLOT5_EID  0x30
#define SLOT6_EID  0x38
#define SLOT7_EID  0x40
#define SLOT8_EID  0x48

// 對應的 PID
#define SLOT1_PID  1
#define SLOT2_PID  2
// ...
```

### 相關裝置 EID

```
Slot N EID 架構：
├── SD BIC:   Base EID
├── FF BIC:   Base EID + 1
├── WF BIC:   Base EID + 2
├── FF CXL:   Base EID + 3
├── WF CXL1:  Base EID + 4
└── WF CXL2:  Base EID + 5
```

### 路由表更新

```c
// plat_mctp.c
void set_routing_table_eid()
{
    uint8_t base_eid = get_slot_eid();
    
    // 更新路由表中的 EID
    for (uint8_t i = 2, j = 1; i < ARRAY_SIZE(plat_mctp_route_tbl); i++, j++) {
        mctp_route_entry *p = plat_mctp_route_tbl + i;
        p->endpoint = base_eid + j;
    }
    
    LOG_INF("Routing table updated with base EID: 0x%02x", base_eid);
}
```

---

## 平台配置初始化

```c
// plat_class.c
void init_platform_config(void)
{
    // 初始化 Slot 配置
    init_slot_config();
    
    // 初始化 Blade 配置類型
    init_blade_config();
    
    // 初始化 Board Revision
    init_board_revision();
    
    // 初始化 Retimer 類型
    init_retimer_type();
    
    // 初始化 VR 類型
    init_vr_type();
}
```

---

## Blade 配置

### 類型偵測

```c
enum {
    BLADE_CONFIG_UNKNOWN = 0,
    BLADE_CONFIG_T1C,     // Type 1 Compute
    BLADE_CONFIG_T1M,     // Type 1 Memory
};

bool get_blade_config(uint8_t *blade_config)
{
    float voltage = 0.0f;
    
    if (!get_adc_voltage(ADC_CHANNEL_12, &voltage)) {
        *blade_config = BLADE_CONFIG_UNKNOWN;
        return false;
    }
    
    // 根據電壓判斷 Blade 類型
    if (voltage >= 0.0f && voltage <= 1.05f) {
        *blade_config = BLADE_CONFIG_T1C;
        LOG_INF("Blade config: Type 1 Compute");
    } else if (voltage >= 1.2f && voltage <= 1.7f) {
        *blade_config = BLADE_CONFIG_T1M;
        LOG_INF("Blade config: Type 1 Memory");
    } else {
        *blade_config = BLADE_CONFIG_UNKNOWN;
        LOG_WRN("Unknown blade configuration: %.2fV", voltage);
        return false;
    }
    
    return true;
}
```

---

## 除錯與驗證

### Shell 命令

```bash
# 顯示目前 Slot 資訊
uart:~$ platform info
Slot ID: 3
Slot EID: 0x20
Blade Config: T1C
Board Rev: DVT

# 讀取原始 ADC 電壓
uart:~$ adc read 13
Channel 13: 0.65V
```

### IPMI 命令

```c
// OEM_1S_GET_BOARD_ID
void OEM_1S_GET_BOARD_ID(ipmi_msg *msg)
{
    msg->data[0] = BOARD_ID;
    msg->data[1] = get_slot_id();
    msg->data_len = 2;
    msg->completion_code = CC_SUCCESS;
}
```

---

## 相關文件

- [YV4SDPlatform](YV4SDPlatform.md) - 平台概述
- [PlatformInit](PlatformInit.md) - 初始化流程
- [MCTPOverview](MCTPOverview.md) - EID 路由

---

*返回 [Home](Home.md)*
