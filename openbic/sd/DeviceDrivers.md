# DeviceDrivers（裝置驅動程式）

本文說明 OpenBIC 的裝置驅動程式架構、常用驅動實作，以及如何新增自訂驅動。

---

## 驅動架構

### 目錄結構

```
common/dev/
├── include/               # 驅動標頭檔
│   ├── tmp75.h
│   ├── isl69259.h
│   ├── ina233.h
│   └── ...
├── tmp75.c                # 溫度感測器驅動
├── isl69259.c             # Renesas VR 驅動
├── mp2971.c               # MPS VR 驅動
├── xdpe12284c.c           # Infineon VR 驅動
├── ina233.c               # 電流/功率驅動
├── nvme.c                 # NVMe 驅動
├── pmic.c                 # PMIC 驅動
├── eeprom.c               # EEPROM 驅動
└── ...                    # 更多驅動 (~90+ 個)
```

### 驅動介面

每個驅動需實作讀取函數：

```c
// 驅動讀取函數原型
uint8_t sensor_dev_xxx_read(sensor_cfg *cfg, int *reading);

// 可選：初始化函數
uint8_t sensor_dev_xxx_init(sensor_cfg *cfg);
```

### 驅動註冊

```c
// sensor.c
SENSOR_DRIVE_INIT_DECLARE(tmp75);
SENSOR_DRIVE_INIT_DECLARE(isl69259);
SENSOR_DRIVE_INIT_DECLARE(mp2971);
// ...

// 驅動映射表
sensor_drive_api sensor_drive_tbl[] = {
    SENSOR_DRIVE_TYPE_INIT_MAP(tmp75),
    SENSOR_DRIVE_TYPE_INIT_MAP(isl69259),
    SENSOR_DRIVE_TYPE_INIT_MAP(mp2971),
    // ...
};
```

---

## 溫度感測器驅動

### TMP75

```c
// tmp75.c
uint8_t tmp75_read(sensor_cfg *cfg, int *reading)
{
    CHECK_NULL_ARG_WITH_RETURN(cfg, SENSOR_UNSPECIFIED_ERROR);
    CHECK_NULL_ARG_WITH_RETURN(reading, SENSOR_UNSPECIFIED_ERROR);

    uint8_t retry = 5;
    I2C_MSG msg = { 0 };

    msg.bus = cfg->port;
    msg.target_addr = cfg->target_addr;
    msg.tx_len = 1;
    msg.rx_len = 2;
    msg.data[0] = cfg->offset;  // Temperature register

    if (i2c_master_read(&msg, retry) != 0) {
        return SENSOR_FAIL_TO_ACCESS;
    }

    // TMP75: 12-bit resolution, 0.0625°C per LSB
    int16_t raw = (msg.data[0] << 8) | msg.data[1];
    raw >>= 4;  // 右移 4 bits
    
    // 轉換為 milli-celsius
    *reading = raw * 625 / 10;  // 0.0625 * 1000 / 10
    
    return SENSOR_READ_SUCCESS;
}
```

### TMP431（遠端溫度）

```c
// tmp431.c
uint8_t tmp431_read(sensor_cfg *cfg, int *reading)
{
    I2C_MSG msg = { 0 };
    
    msg.bus = cfg->port;
    msg.target_addr = cfg->target_addr;
    
    // 根據 offset 選擇 local 或 remote 溫度
    switch (cfg->offset) {
    case TMP431_LOCAL_TEMP:
        msg.data[0] = 0x00;
        break;
    case TMP431_REMOTE_TEMP:
        msg.data[0] = 0x01;
        break;
    }
    
    // ... 讀取邏輯
    return SENSOR_READ_SUCCESS;
}
```

---

## VR (Voltage Regulator) 驅動

### ISL69259 (Renesas)

```c
// isl69259.c
uint8_t isl69259_read(sensor_cfg *cfg, int *reading)
{
    CHECK_NULL_ARG_WITH_RETURN(cfg, SENSOR_UNSPECIFIED_ERROR);

    uint8_t retry = 5;
    I2C_MSG msg = { 0 };

    msg.bus = cfg->port;
    msg.target_addr = cfg->target_addr;
    msg.tx_len = 1;
    msg.rx_len = 2;
    msg.data[0] = cfg->offset;  // PMBus command

    if (i2c_master_read(&msg, retry) != 0) {
        return SENSOR_FAIL_TO_ACCESS;
    }

    // PMBus Linear 格式解碼
    switch (cfg->offset) {
    case PMBUS_READ_VOUT:
        *reading = decode_linear11(msg.data);
        break;
    case PMBUS_READ_IOUT:
        *reading = decode_linear11(msg.data);
        break;
    case PMBUS_READ_TEMPERATURE_1:
        *reading = decode_linear11(msg.data);
        break;
    case PMBUS_READ_POUT:
        *reading = decode_linear11(msg.data);
        break;
    }

    return SENSOR_READ_SUCCESS;
}
```

### MP2971 (MPS)

```c
// mp2971.c
uint8_t mp2971_read(sensor_cfg *cfg, int *reading)
{
    // MPS VR 特定處理
    // 支援多頁面（PAGE 0/1 對應雙軌道）
    
    I2C_MSG msg = { 0 };
    
    // 設定頁面
    if (cfg->arg0 != 0xFF) {
        msg.bus = cfg->port;
        msg.target_addr = cfg->target_addr;
        msg.tx_len = 2;
        msg.data[0] = PMBUS_PAGE;
        msg.data[1] = cfg->arg0;
        i2c_master_write(&msg, 3);
    }

    // 讀取數據
    msg.tx_len = 1;
    msg.rx_len = 2;
    msg.data[0] = cfg->offset;
    
    if (i2c_master_read(&msg, 5) != 0) {
        return SENSOR_FAIL_TO_ACCESS;
    }

    // 解碼
    *reading = mp2971_decode(cfg->offset, msg.data);
    
    return SENSOR_READ_SUCCESS;
}
```

### XDPE12284C (Infineon)

```c
// xdpe12284c.c
uint8_t xdpe12284c_read(sensor_cfg *cfg, int *reading)
{
    // Infineon VR 實作
    // 支援 XDPE12284 雙軌 VR
    
    // ... 實作細節
    return SENSOR_READ_SUCCESS;
}
```

---

## 電流/功率監控驅動

### INA233

```c
// ina233.c
uint8_t ina233_read(sensor_cfg *cfg, int *reading)
{
    I2C_MSG msg = { 0 };
    
    msg.bus = cfg->port;
    msg.target_addr = cfg->target_addr;
    msg.tx_len = 1;
    msg.rx_len = 2;
    msg.data[0] = cfg->offset;

    if (i2c_master_read(&msg, 5) != 0) {
        return SENSOR_FAIL_TO_ACCESS;
    }

    int16_t raw = (msg.data[1] << 8) | msg.data[0];
    
    switch (cfg->offset) {
    case INA233_BUS_VOLTAGE:
        // 1.25 mV/LSB
        *reading = raw * 125 / 100;
        break;
    case INA233_CURRENT:
        // 根據 calibration 設定
        *reading = raw * current_lsb;
        break;
    case INA233_POWER:
        // 功率計算
        *reading = raw * power_lsb;
        break;
    }

    return SENSOR_READ_SUCCESS;
}
```

### INA238

```c
// ina238.c
uint8_t ina238_read(sensor_cfg *cfg, int *reading)
{
    // INA238 16-bit ADC 版本
    // ... 實作細節
    return SENSOR_READ_SUCCESS;
}
```

---

## PMIC 驅動

### DDR5 PMIC

```c
// pmic.c
uint8_t pmic_read(sensor_cfg *cfg, int *reading)
{
    // JEDEC DDR5 PMIC 規範
    I2C_MSG msg = { 0 };
    
    msg.bus = cfg->port;
    msg.target_addr = cfg->target_addr;
    msg.tx_len = 1;
    msg.rx_len = 1;
    msg.data[0] = cfg->offset;

    if (i2c_master_read(&msg, 5) != 0) {
        return SENSOR_FAIL_TO_ACCESS;
    }

    switch (cfg->offset) {
    case PMIC_PWR_STATUS:
        *reading = msg.data[0];
        break;
    case PMIC_TEMP:
        // 溫度轉換
        *reading = pmic_temp_convert(msg.data[0]);
        break;
    }

    return SENSOR_READ_SUCCESS;
}
```

---

## AST1030 ADC 驅動

```c
// ast_adc.c
uint8_t ast_adc_read(sensor_cfg *cfg, int *reading)
{
    uint8_t channel = cfg->offset;
    
    if (channel >= NUMBER_OF_ADC_CHANNEL) {
        return SENSOR_UNSPECIFIED_ERROR;
    }

    // 讀取 ADC 暫存器
    uint32_t reg_value = sys_read32(AST1030_ADC_BASE_ADDR + adc_info[channel].offset);
    uint32_t raw_value = (reg_value >> adc_info[channel].shift) & 0x3FF;

    // 取得參考電壓
    float ref_voltage = 2.5;  // 預設 2.5V
    
    // 計算實際電壓 (mV)
    *reading = (raw_value * (int)(ref_voltage * 1000)) / 1024;

    return SENSOR_READ_SUCCESS;
}
```

---

## NVMe 驅動

```c
// nvme.c
uint8_t nvme_read(sensor_cfg *cfg, int *reading)
{
    // NVMe MI 規範讀取溫度
    I2C_MSG msg = { 0 };
    
    msg.bus = cfg->port;
    msg.target_addr = cfg->target_addr;
    msg.tx_len = 1;
    msg.rx_len = 8;
    msg.data[0] = NVME_MI_TEMP_SENSOR;

    if (i2c_master_read(&msg, 5) != 0) {
        return SENSOR_FAIL_TO_ACCESS;
    }

    // NVMe 溫度格式：Kelvin 減 273
    int16_t temp_k = (msg.data[2] << 8) | msg.data[1];
    *reading = (temp_k - 273) * 1000;  // milli-celsius

    return SENSOR_READ_SUCCESS;
}
```

---

## PMBus 工具函數

### Linear11 解碼

```c
// util_pmbus.c
int decode_linear11(uint8_t *data)
{
    int16_t mantissa = ((data[1] & 0x07) << 8) | data[0];
    int8_t exponent = (data[1] >> 3) & 0x1F;
    
    // Sign extension
    if (exponent & 0x10) {
        exponent |= 0xE0;
    }
    if (mantissa & 0x400) {
        mantissa |= 0xF800;
    }
    
    // 計算實際值
    if (exponent >= 0) {
        return mantissa << exponent;
    } else {
        return mantissa >> (-exponent);
    }
}

// VID 轉電壓
int vid_to_voltage(uint8_t vid, uint8_t vid_mode)
{
    switch (vid_mode) {
    case VID_MODE_AMD:
        return 1550 - vid * 625 / 100;
    case VID_MODE_INTEL:
        return vid * 5;
    default:
        return 0;
    }
}
```

---

## 新增驅動

### 步驟

1. **建立驅動檔案**：`common/dev/my_sensor.c`
2. **建立標頭檔**：`common/dev/include/my_sensor.h`
3. **實作讀取函數**
4. **註冊驅動**

### 範例

```c
// my_sensor.h
#ifndef MY_SENSOR_H
#define MY_SENSOR_H

#include "sensor.h"

uint8_t my_sensor_read(sensor_cfg *cfg, int *reading);
uint8_t my_sensor_init(sensor_cfg *cfg);

#endif

// my_sensor.c
#include "my_sensor.h"
#include "hal_i2c.h"

uint8_t my_sensor_read(sensor_cfg *cfg, int *reading)
{
    CHECK_NULL_ARG_WITH_RETURN(cfg, SENSOR_UNSPECIFIED_ERROR);
    
    I2C_MSG msg = { 0 };
    msg.bus = cfg->port;
    msg.target_addr = cfg->target_addr;
    msg.tx_len = 1;
    msg.rx_len = 2;
    msg.data[0] = cfg->offset;

    if (i2c_master_read(&msg, 5) != 0) {
        return SENSOR_FAIL_TO_ACCESS;
    }

    // 處理讀取數據
    *reading = (msg.data[0] << 8) | msg.data[1];
    
    return SENSOR_READ_SUCCESS;
}

// sensor.c 中註冊
SENSOR_DRIVE_INIT_DECLARE(my_sensor);
// 在 sensor_drive_tbl 中新增
SENSOR_DRIVE_TYPE_INIT_MAP(my_sensor),
```

---

## 相關文件

- [SensorFramework](SensorFramework.md) - 感測器框架
- [I2C_I3C](I2C_I3C.md) - 通訊介面
- [PLDMSensor](PLDMSensor.md) - PLDM 感測器

---

*返回 [Home](Home.md)*
