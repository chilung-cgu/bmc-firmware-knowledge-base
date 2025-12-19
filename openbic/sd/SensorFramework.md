# SensorFramework（感測器框架）

本文說明 OpenBIC 的感測器框架設計，包括 sensor_cfg 結構、感測器輪詢機制，以及支援的感測器類型。

---

## 框架概述

OpenBIC 感測器框架提供統一的介面管理各類感測器：

```
┌─────────────────────────────────────────────────────────────────┐
│                     Sensor Framework                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────┐    ┌─────────────────┐                    │
│  │  Sensor Poll    │    │  Sensor Config  │                    │
│  │  Thread         │───►│  Table          │                    │
│  └────────┬────────┘    └────────┬────────┘                    │
│           │                      │                              │
│           ▼                      ▼                              │
│  ┌─────────────────────────────────────┐                       │
│  │          Sensor Driver Map          │                       │
│  │  ┌───────┐ ┌───────┐ ┌───────┐     │                       │
│  │  │ TMP75 │ │ INA233│ │ VR    │ ... │                       │
│  │  └───────┘ └───────┘ └───────┘     │                       │
│  └─────────────────────────────────────┘                       │
│                      │                                          │
│                      ▼                                          │
│  ┌─────────────────────────────────────┐                       │
│  │           HAL Layer (I2C/I3C)       │                       │
│  └─────────────────────────────────────┘                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## sensor_cfg 結構

### 定義

```c
// sensor.h
typedef struct _sensor_cfg_ {
    uint8_t num;                    // 感測器編號
    uint8_t type;                   // 感測器類型
    uint8_t port;                   // I2C/I3C 匯流排
    uint8_t target_addr;            // 目標地址
    uint8_t offset;                 // 暫存器偏移
    bool (*access_checker)(uint8_t); // 存取檢查函數
    uint8_t arg0;                   // 額外參數 0
    uint8_t arg1;                   // 額外參數 1
    int sample_count;               // 取樣計數
    int cache;                      // 快取值
    uint8_t cache_status;           // 快取狀態
    bool is_init;                   // 是否已初始化
    void *init_args;                // 初始化參數
    void *priv_data;                // 私有資料
    uint8_t retry;                  // 重試次數
    int (*pre_sensor_read_hook)(struct _sensor_cfg_ *, void *);  // 讀取前回調
    void *pre_sensor_read_args;     // 回調參數
    int (*post_sensor_read_hook)(struct _sensor_cfg_ *, void *, int *); // 讀取後回調
    void *post_sensor_read_args;    // 回調參數
} sensor_cfg;
```

### 欄位說明

| 欄位 | 說明 |
|------|------|
| `num` | 唯一感測器編號 |
| `type` | 驅動類型 (sensor_dev_xxx) |
| `port` | I2C/I3C bus 編號 |
| `target_addr` | I2C 7-bit 地址 |
| `offset` | 暫存器偏移或通道 |
| `access_checker` | 確認是否可讀取 |
| `pre_sensor_read_hook` | 讀取前需執行的操作 |
| `post_sensor_read_hook` | 讀取後處理 |

---

## 感測器類型

### 支援的驅動類型

```c
// sensor.c
const char *const sensor_type_name[] = {
    sensor_name_to_num(tmp75)
    sensor_name_to_num(adc)
    sensor_name_to_num(peci)
    sensor_name_to_num(isl69259)
    sensor_name_to_num(hsc)
    sensor_name_to_num(nvme)
    sensor_name_to_num(mp5990)
    sensor_name_to_num(ina233)
    sensor_name_to_num(tmp431)
    sensor_name_to_num(pmic)
    sensor_name_to_num(mp2971)
    sensor_name_to_num(xdpe12284c)
    sensor_name_to_num(raa229621)
    sensor_name_to_num(nct7718w)
    sensor_name_to_num(mp2985)
    sensor_name_to_num(ina238)
    // ... 更多驅動
};
```

### 常用感測器類型

| 類型 | 說明 | 驅動檔案 |
|------|------|----------|
| `sensor_dev_tmp75` | 溫度感測器 | tmp75.c |
| `sensor_dev_ast_adc` | AST1030 ADC | ast_adc.c |
| `sensor_dev_isl69259` | Renesas VR | isl69259.c |
| `sensor_dev_mp2971` | MPS VR | mp2971.c |
| `sensor_dev_xdpe12284c` | Infineon VR | xdpe12284c.c |
| `sensor_dev_ina233` | 電流/功率 | ina233.c |
| `sensor_dev_nvme` | NVMe 溫度 | nvme.c |
| `sensor_dev_pmic` | DDR5 PMIC | pmic.c |
| `sensor_dev_tmp431` | 遠端溫度 | tmp431.c |

---

## 感測器配置表

### 範例配置

```c
// plat_sensor_table.c
sensor_cfg plat_sensor_config[] = {
    // 溫度感測器
    {
        .num = SENSOR_NUM_TEMP_TMP75_1,
        .type = sensor_dev_tmp75,
        .port = I2C_BUS0,
        .target_addr = 0x48,
        .offset = 0x00,
        .access_checker = stby_access,
    },
    
    // VR 感測器 (電壓)
    {
        .num = SENSOR_NUM_VR_CPU0_VOUT,
        .type = sensor_dev_mp2971,
        .port = I2C_BUS4,
        .target_addr = ADDR_VR_CPU0,
        .offset = PMBUS_READ_VOUT,
        .access_checker = dc_access,
        .pre_sensor_read_hook = pre_vr_read,
        .pre_sensor_read_args = &vr_pre_read_args[0],
    },
    
    // VR 感測器 (電流)
    {
        .num = SENSOR_NUM_VR_CPU0_IOUT,
        .type = sensor_dev_mp2971,
        .port = I2C_BUS4,
        .target_addr = ADDR_VR_CPU0,
        .offset = PMBUS_READ_IOUT,
        .access_checker = dc_access,
    },
    
    // ADC 感測器
    {
        .num = SENSOR_NUM_VOL_ADC_P3V3_STBY,
        .type = sensor_dev_ast_adc,
        .port = ADC_PORT0,
        .target_addr = 0,
        .offset = ADC_CHANNEL_2,
        .access_checker = stby_access,
    },
};

const int SENSOR_CONFIG_SIZE = ARRAY_SIZE(plat_sensor_config);
```

---

## 存取檢查器

### 常用檢查函數

```c
// 始終可存取 (Standby 電源)
bool stby_access(uint8_t sensor_num)
{
    return true;
}

// 僅在 DC 電源開啟時可存取
bool dc_access(uint8_t sensor_num)
{
    return get_DC_status();
}

// DC 電源開啟且 POST 完成後可存取
bool post_access(uint8_t sensor_num)
{
    return (get_DC_status() && get_post_status());
}

// VR 感測器存取檢查
bool vr_access(uint8_t sensor_num)
{
    return (get_DC_status() && !is_vr_fault());
}
```

---

## 感測器輪詢

### 輪詢機制

```c
// sensor.c
void sensor_poll(void *arug0, void *arug1, void *arug2)
{
    k_msleep(5000);  // 等待系統穩定
    
    while (1) {
        for (int i = 0; i < sensor_config_count; i++) {
            sensor_cfg *cfg = &sensor_config[i];
            
            // 檢查是否可存取
            if (cfg->access_checker && !cfg->access_checker(cfg->num)) {
                cfg->cache_status = SENSOR_NOT_ACCESSIBLE;
                continue;
            }
            
            // 執行 pre-read hook
            if (cfg->pre_sensor_read_hook) {
                cfg->pre_sensor_read_hook(cfg, cfg->pre_sensor_read_args);
            }
            
            // 讀取感測器
            int reading = 0;
            uint8_t status = sensor_read(cfg, &reading);
            
            // 執行 post-read hook
            if (cfg->post_sensor_read_hook) {
                cfg->post_sensor_read_hook(cfg, cfg->post_sensor_read_args, &reading);
            }
            
            // 更新快取
            cfg->cache = reading;
            cfg->cache_status = status;
        }
        
        k_msleep(sensor_poll_interval);
    }
}
```

### 輪詢控制

```c
// 啟用/禁用感測器輪詢
void control_sensor_polling(uint8_t sensor_num, uint8_t enable)
{
    for (int i = 0; i < sensor_config_count; i++) {
        if (sensor_config[i].num == sensor_num) {
            if (enable) {
                // 啟用輪詢
            } else {
                // 禁用輪詢
            }
            break;
        }
    }
}
```

---

## Pre/Post Hook

### Pre-Read Hook

在讀取感測器前執行，通常用於：
- 切換 I2C Mux
- 選擇 VR 頁面
- 準備裝置

```c
// plat_hook.c
int pre_vr_read(sensor_cfg *cfg, void *args)
{
    vr_pre_read_arg *vr_args = (vr_pre_read_arg *)args;
    
    // 切換 VR 頁面
    if (vr_args->vr_page != 0xFF) {
        I2C_MSG msg = { 0 };
        msg.bus = cfg->port;
        msg.target_addr = cfg->target_addr;
        msg.tx_len = 2;
        msg.data[0] = PMBUS_PAGE;
        msg.data[1] = vr_args->vr_page;
        i2c_master_write(&msg, 3);
    }
    
    return 0;
}
```

### Post-Read Hook

在讀取後處理數據，通常用於：
- 單位轉換
- 數值校正
- 多值計算

```c
int post_adc_read(sensor_cfg *cfg, void *args, int *reading)
{
    adc_post_arg *adc_args = (adc_post_arg *)args;
    
    // 應用電壓分壓比
    *reading = (*reading) * adc_args->divisor;
    
    return 0;
}
```

---

## PLDM 感測器整合

OpenBIC 支援 PLDM 感測器介面：

```c
// pldm_sensor.c
void pldm_sensor_monitor_init(void)
{
    // 初始化 PLDM 感測器監控
}

// plat_pldm_sensor.c
// 定義 Platform Descriptor Records (PDR)
pldm_sensor_info plat_pldm_sensor_table[] = {
    {
        .sensor_id = PLDM_SENSOR_ID_TEMP_CPU,
        .sensor_type = PLDM_SENSOR_TYPE_TEMPERATURE,
        .base_unit = PLDM_UNIT_CELSIUS,
        // ...
    },
};
```

---

## 感測器讀取狀態

```c
// 讀取狀態定義
#define SENSOR_READ_SUCCESS         0x00
#define SENSOR_NOT_ACCESSIBLE       0x01
#define SENSOR_INIT_STATUS          0x02
#define SENSOR_NOT_FOUND            0x03
#define SENSOR_FAIL_TO_ACCESS       0x04
#define SENSOR_UNSPECIFIED_ERROR    0x05
#define SENSOR_POLLING_DISABLE      0x06
```

---

## Shell 命令

```bash
# 列出所有感測器
uart:~$ sensor list

# 讀取特定感測器
uart:~$ sensor read <sensor_num>

# 感測器狀態
uart:~$ sensor status
```

---

## 新增感測器

### 步驟

1. **新增驅動**（若需要）
2. **定義感測器編號**
3. **新增配置到 plat_sensor_config[]**
4. **實作存取檢查器**（若需要）
5. **實作 Hook 函數**（若需要）

### 範例

```c
// 1. 定義編號
#define SENSOR_NUM_NEW_TEMP  0x50

// 2. 新增配置
{
    .num = SENSOR_NUM_NEW_TEMP,
    .type = sensor_dev_tmp75,
    .port = I2C_BUS0,
    .target_addr = 0x49,
    .offset = 0x00,
    .access_checker = stby_access,
},
```

---

## 相關文件

- [DeviceDrivers](DeviceDrivers.md) - 裝置驅動
- [PLDMSensor](PLDMSensor.md) - PLDM 感測器
- [I2C_I3C](I2C_I3C.md) - 通訊介面

---

*返回 [Home](Home.md)*
