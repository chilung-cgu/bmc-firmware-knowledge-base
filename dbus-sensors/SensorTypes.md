# 感測器類型總覽

## 簡介

dbus-sensors 支援多種感測器類型，每種類型由專門的守護程式處理。本文件提供所有支援的感測器類型概覽。

---

## 感測器守護程式

| 守護程式 | 服務名稱 | 配置類型 | 資料來源 |
|----------|----------|----------|----------|
| ADCSensor | `xyz.openbmc_project.ADCSensor` | ADC | hwmon (IIO) |
| HwmonTempSensor | `xyz.openbmc_project.HwmonTempSensor` | TMP75, TMP441 等 | hwmon |
| PSUSensor | `xyz.openbmc_project.PSUSensor` | INA219, ADM1275 等 | hwmon (PMBus) |
| FanSensor | `xyz.openbmc_project.FanSensor` | AspeedFan, I2CFan | hwmon |
| CPUSensor | `xyz.openbmc_project.CPUSensor` | XeonCPU | PECI |
| NVMeSensor | `xyz.openbmc_project.NVMeSensor` | NVME1000 | SMBus |
| ExternalSensor | `xyz.openbmc_project.ExternalSensor` | ExternalSensor | D-Bus |
| IpmbSensor | `xyz.openbmc_project.IpmbSensor` | IpmbSensor | IPMB |
| IntrusionSensor | `xyz.openbmc_project.IntrusionSensor` | ChassisIntrusionSensor | GPIO/I2C |
| ExitAirTempSensor | `xyz.openbmc_project.ExitAirTempSensor` | ExitAirTempSensor | 計算值 |

---

## 感測器單位類型

dbus-sensors 支援以下單位類型，對應到不同的 D-Bus 路徑：

| 單位 | D-Bus 路徑 | 說明 |
|------|------------|------|
| DegreesC | `/sensors/temperature/` | 攝氏溫度 |
| Volts | `/sensors/voltage/` | 電壓 |
| Amperes | `/sensors/current/` | 電流 |
| Watts | `/sensors/power/` | 功率 |
| RPMS | `/sensors/fan_tach/` | 風扇轉速 |
| Joules | `/sensors/energy/` | 能量 |
| Percent | `/sensors/utilization/` | 使用率 |
| CFM | `/sensors/airflow/` | 氣流（立方英尺/分鐘） |
| Pascals | `/sensors/pressure/` | 壓力 |
| Meters | `/sensors/altitude/` | 高度 |
| Hertz | `/sensors/frequency/` | 頻率 |
| PercentRH | `/sensors/humidity/` | 相對濕度 |
| LPM | `/sensors/liquidflow/` | 液體流量（公升/分鐘） |

---

## 溫度感測器晶片

HwmonTempSensor 支援的晶片類型：

| 晶片型號 | 通道數 | 說明 |
|----------|--------|------|
| TMP75 | 1 | 基本 I2C 溫度感測器 |
| TMP441 | 2 | 本地 + 遠端溫度 |
| TMP464 | 5 | 多通道溫度感測器 |
| LM75A | 1 | 標準溫度感測器 |
| LM95234 | 5 | 多通道遠端二極體 |
| MAX31725 | 1 | 高精度溫度感測器 |
| MAX6581 | 8 | 多通道溫度監控 |
| EMC1403 | 3 | 三通道溫度感測器 |
| JC42 | 1 | JEDEC DIMM 溫度 |
| SBTSI | 1 | AMD CPU 側頻溫度 |
| NCT7802 | 5 | 硬體監控 IC |
| BME280 | 1 | 溫度 + 濕度 + 氣壓 |

> 📋 **完整列表**：詳見 [HwmonTempSensor](HwmonTempSensor.md)

---

## PMBus 感測器晶片

PSUSensor 支援的 PMBus 裝置：

| 類別 | 晶片型號 |
|------|----------|
| 電流/電壓監控 | INA219, INA230, INA233, INA238 |
| 熱插拔控制器 | ADM1272, ADM1275, ADM1278, LM25066 |
| 多相調節器 | TPS53679, XDPE12284, RAA228228 |
| 電源供應器 | DPS800, cffps, IPSPS1 |

> 📋 **完整列表**：詳見 [PSUSensor](PSUSensor.md)

---

## 風扇感測器類型

| 類型 | 平台 | 說明 |
|------|------|------|
| AspeedFan | Aspeed AST2500/2600 | 內建 PWM/轉速計 |
| NuvotonFan | Nuvoton | Nuvoton 風扇控制器 |
| I2CFan | I2C | I2C 風扇控制晶片 |
| HPEFan | HPE 平台 | HPE 特定風扇 |

---

## 特殊感測器

### ExternalSensor

- 無實體硬體連結
- 數值由外部來源推送（IPMI、Redfish）
- 支援 Timeout 機制偵測過期數值

### ExitAirTempSensor / CFMSensor

- 計算型感測器
- 基於其他感測器數值計算

### ChassisIntrusionSensor

- 機箱入侵偵測
- GPIO 或 I2C 輸入

---

## 資料來源分類

```
┌─────────────────────────────────────────────────────────┐
│                    dbus-sensors                          │
├─────────────────┬─────────────────┬─────────────────────┤
│   hwmon 來源    │   直接存取      │    D-Bus 來源       │
├─────────────────┼─────────────────┼─────────────────────┤
│ • ADCSensor     │ • CPUSensor     │ • ExternalSensor    │
│ • HwmonTemp     │   (PECI)        │ • ExitAirTemp       │
│ • PSUSensor     │ • NVMeSensor    │   (計算值)          │
│ • FanSensor     │   (SMBus)       │                     │
│                 │ • IpmbSensor    │                     │
│                 │ • Intrusion     │                     │
└─────────────────┴─────────────────┴─────────────────────┘
```

---

## 相關文件

- [ADC 感測器](ADCSensor.md)
- [Hwmon 溫度感測器](HwmonTempSensor.md)
- [PSU 感測器](PSUSensor.md)
- [風扇感測器](FanSensor.md)
- [CPU 感測器](CPUSensor.md)
- [外部感測器](ExternalSensor.md)
- [設定指南](ConfigurationGuide.md)
