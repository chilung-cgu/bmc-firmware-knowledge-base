# IPMB 感測器

## 簡介

IpmbSensor 守護程式透過 IPMB（Intelligent Platform Management Bus）協定讀取遠端感測器數值。IPMB 是 IPMI 架構中用於 BMC 間通訊的 I2C 協定。

---

## 守護程式資訊

| 項目 | 值 |
|------|-----|
| 執行檔 | `ipmbsensor` |
| D-Bus 服務 | `xyz.openbmc_project.IpmbSensor` |
| 配置類型 | IpmbSensor |
| 資料來源 | IPMB |

---

## Entity-Manager 配置

### 基本配置

```json
{
    "Exposes": [
        {
            "Address": "0x20",
            "Class": "METemp",
            "Name": "ME Temp",
            "Type": "IpmbSensor"
        }
    ],
    "Name": "IPMB Sensors",
    "Probe": "TRUE",
    "Type": "Board"
}
```

### 配置欄位說明

| 欄位 | 必填 | 類型 | 說明 |
|------|------|------|------|
| `Name` | ✅ | string | 感測器名稱 |
| `Type` | ✅ | string | 必須為 `"IpmbSensor"` |
| `Address` | ✅ | string | IPMB 從機位址 |
| `Class` | ✅ | string | 感測器類別 |
| `HostSMbusIndex` | ❌ | number | 主機 SMBus 索引 |
| `PowerState` | ❌ | string | 電源狀態條件 |
| `SensorType` | ❌ | string | 感測器測量類型 |
| `ScaleValue` | ❌ | number | 縮放值 |
| `OffsetValue` | ❌ | number | 偏移值 |
| `Thresholds` | ❌ | array | 閾值配置 |

---

## Class 類別

| Class 值 | 說明 |
|----------|------|
| `METemp` | Intel ME（Management Engine）溫度 |
| `temp` | 一般溫度感測器 |
| `PxeBridgeTemp` | PXE 橋接溫度 |
| `i2c` | 一般 I2C 感測器 |
| `HSCBridge` | 熱插拔控制器橋接 |

---

## SensorType 類型

| SensorType 值 | D-Bus 路徑 | 說明 |
|---------------|------------|------|
| `current` | `/sensors/current/` | 電流 |
| `power` | `/sensors/power/` | 功率 |
| `voltage` | `/sensors/voltage/` | 電壓 |
| （預設） | `/sensors/temperature/` | 溫度 |

---

## 數值轉換

使用 `ScaleValue` 和 `OffsetValue` 進行數值轉換：

```
實際值 = 原始值 × ScaleValue + OffsetValue
```

```json
{
    "Address": "0x20",
    "Class": "temp",
    "Name": "Remote Temp",
    "Type": "IpmbSensor",
    "ScaleValue": 0.5,
    "OffsetValue": -10
}
```

---

## 完整配置範例

### Intel ME 溫度監測

```json
{
    "Exposes": [
        {
            "Address": "0x2C",
            "Class": "METemp",
            "Name": "ME Temp",
            "Type": "IpmbSensor",
            "PowerState": "On",
            "Thresholds": [
                {
                    "Direction": "greater than",
                    "Name": "upper critical",
                    "Severity": 1,
                    "Value": 100
                },
                {
                    "Direction": "greater than",
                    "Name": "upper warning",
                    "Severity": 0,
                    "Value": 90
                }
            ]
        }
    ],
    "Name": "ME IPMB Sensor",
    "Probe": "TRUE",
    "Type": "Board"
}
```

### 多個 IPMB 感測器

```json
{
    "Exposes": [
        {
            "Address": "0x20",
            "Class": "temp",
            "HostSMbusIndex": 0,
            "Name": "Riser Temp 1",
            "Type": "IpmbSensor",
            "PowerState": "On"
        },
        {
            "Address": "0x20",
            "Class": "temp",
            "HostSMbusIndex": 1,
            "Name": "Riser Temp 2",
            "Type": "IpmbSensor",
            "PowerState": "On",
            "SensorType": "voltage",
            "ScaleValue": 0.001
        }
    ],
    "Name": "Riser IPMB Sensors",
    "Probe": "TRUE",
    "Type": "Board"
}
```

---

## D-Bus 輸出

```bash
$ busctl tree xyz.openbmc_project.IpmbSensor
└─/xyz
  └─/xyz/openbmc_project
    └─/xyz/openbmc_project/sensors
      └─/xyz/openbmc_project/sensors/temperature
        ├─/xyz/openbmc_project/sensors/temperature/ME_Temp
        └─/xyz/openbmc_project/sensors/temperature/Riser_Temp_1
```

---

## IPMB 通訊

IpmbSensor 使用 phosphor-ipmi-ipmb 服務進行 IPMB 通訊：

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│ IpmbSensor  │────▶│ ipmb-host   │────▶│ 遠端 BMC    │
│ 守護程式    │     │ 服務        │     │ 或控制器    │
└─────────────┘     └─────────────┘     └─────────────┘
      │                   │                   │
      ▼                   ▼                   ▼
    D-Bus              I2C/SMBus           感測器
    請求               訊息                 數值
```

---

## 除錯指令

```bash
# 檢查 D-Bus 感測器
busctl introspect xyz.openbmc_project.IpmbSensor \
    /xyz/openbmc_project/sensors/temperature/ME_Temp

# 查看 IPMB 服務狀態
systemctl status xyz.openbmc_project.Ipmb.service

# 查看日誌
journalctl -u xyz.openbmc_project.ipmbsensor.service
```

---

## 相關文件

- [感測器類型總覽](SensorTypes.md)
- [設定指南](ConfigurationGuide.md)
- [IPMI 架構](https://github.com/openbmc/docs/blob/master/architecture/ipmi-architecture.md)
