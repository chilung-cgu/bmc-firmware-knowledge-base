# Redfish Chassis Service API

本文件說明 bmcweb 的 Chassis 服務 API，用於管理機箱、感測器、電源和散熱資訊。

---

## 📋 目錄

1. [服務概述](#服務概述)
2. [Chassis 資源](#chassis-資源)
3. [感測器 API](#感測器-api)
4. [電源資訊](#電源資訊)
5. [散熱資訊](#散熱資訊)
6. [LED 控制](#led-控制)

---

## 服務概述

Chassis 服務提供硬體機箱相關功能：

- **機箱資訊** - 類型、製造商、序號
- **感測器** - 溫度、電壓、風扇轉速
- **電源** - 電源供應器狀態
- **散熱** - 風扇和溫度管理
- **LED 控制** - 機箱識別燈

---

## Chassis 資源

### 列出所有機箱

```bash
curl -k -H "X-Auth-Token: $token" \
    https://${bmc}/redfish/v1/Chassis
```

**回應範例：**
```json
{
    "@odata.id": "/redfish/v1/Chassis",
    "@odata.type": "#ChassisCollection.ChassisCollection",
    "Members": [
        {"@odata.id": "/redfish/v1/Chassis/chassis"}
    ],
    "Members@odata.count": 1,
    "Name": "Chassis Collection"
}
```

### 查詢機箱詳情

```bash
curl -k -H "X-Auth-Token: $token" \
    https://${bmc}/redfish/v1/Chassis/chassis
```

**回應範例：**
```json
{
    "@odata.id": "/redfish/v1/Chassis/chassis",
    "@odata.type": "#Chassis.v1_21_0.Chassis",
    "Id": "chassis",
    "Name": "Chassis",
    "ChassisType": "RackMount",
    "Manufacturer": "OpenBMC",
    "Model": "Server",
    "SerialNumber": "1234567890",
    "PowerState": "On",
    "Status": {
        "State": "Enabled",
        "Health": "OK"
    },
    "Power": {
        "@odata.id": "/redfish/v1/Chassis/chassis/Power"
    },
    "Thermal": {
        "@odata.id": "/redfish/v1/Chassis/chassis/Thermal"
    },
    "Sensors": {
        "@odata.id": "/redfish/v1/Chassis/chassis/Sensors"
    },
    "Links": {
        "ComputerSystems": [
            {"@odata.id": "/redfish/v1/Systems/system"}
        ],
        "ManagedBy": [
            {"@odata.id": "/redfish/v1/Managers/bmc"}
        ]
    }
}
```

---

## 感測器 API

### 新版 Sensors API

bmcweb 支援新版 Redfish Sensors API：

```bash
# 列出所有感測器
curl -k -H "X-Auth-Token: $token" \
    https://${bmc}/redfish/v1/Chassis/chassis/Sensors
```

**回應範例：**
```json
{
    "@odata.id": "/redfish/v1/Chassis/chassis/Sensors",
    "@odata.type": "#SensorCollection.SensorCollection",
    "Members": [
        {"@odata.id": "/redfish/v1/Chassis/chassis/Sensors/temperature_CPU0"},
        {"@odata.id": "/redfish/v1/Chassis/chassis/Sensors/voltage_P12V"},
        {"@odata.id": "/redfish/v1/Chassis/chassis/Sensors/fan_Fan0"}
    ],
    "Members@odata.count": 3,
    "Name": "Sensor Collection"
}
```

### 查詢單一感測器

```bash
curl -k -H "X-Auth-Token: $token" \
    https://${bmc}/redfish/v1/Chassis/chassis/Sensors/temperature_CPU0
```

**回應範例：**
```json
{
    "@odata.id": "/redfish/v1/Chassis/chassis/Sensors/temperature_CPU0",
    "@odata.type": "#Sensor.v1_2_0.Sensor",
    "Id": "temperature_CPU0",
    "Name": "CPU0 Temperature",
    "ReadingType": "Temperature",
    "Reading": 45.5,
    "ReadingUnits": "Cel",
    "Status": {
        "State": "Enabled",
        "Health": "OK"
    },
    "Thresholds": {
        "UpperCritical": {
            "Reading": 95.0
        },
        "UpperCaution": {
            "Reading": 85.0
        }
    }
}
```

### 感測器類型

| 類型 | 說明 | 單位 |
|------|------|------|
| `Temperature` | 溫度 | Cel (攝氏) |
| `Voltage` | 電壓 | V (伏特) |
| `Current` | 電流 | A (安培) |
| `Power` | 功率 | W (瓦特) |
| `Rotational` | 轉速 | RPM |
| `Humidity` | 濕度 | % |

---

## 電源資訊

### 傳統 Power API

```bash
curl -k -H "X-Auth-Token: $token" \
    https://${bmc}/redfish/v1/Chassis/chassis/Power
```

**回應範例：**
```json
{
    "@odata.id": "/redfish/v1/Chassis/chassis/Power",
    "@odata.type": "#Power.v1_7_0.Power",
    "Id": "Power",
    "Name": "Power",
    "PowerControl": [
        {
            "@odata.id": "/redfish/v1/Chassis/chassis/Power#/PowerControl/0",
            "Name": "Chassis Power Control",
            "PowerConsumedWatts": 250,
            "PowerLimit": {
                "LimitInWatts": 500
            }
        }
    ],
    "Voltages": [
        {
            "@odata.id": "/redfish/v1/Chassis/chassis/Power#/Voltages/0",
            "Name": "12V Rail",
            "ReadingVolts": 12.1,
            "Status": {
                "State": "Enabled",
                "Health": "OK"
            }
        }
    ],
    "PowerSupplies": [
        {
            "@odata.id": "/redfish/v1/Chassis/chassis/Power#/PowerSupplies/0",
            "Name": "PSU 0",
            "Status": {
                "State": "Enabled",
                "Health": "OK"
            },
            "PowerOutputWatts": 250,
            "PowerCapacityWatts": 800
        }
    ]
}
```

---

## 散熱資訊

### 傳統 Thermal API

```bash
curl -k -H "X-Auth-Token: $token" \
    https://${bmc}/redfish/v1/Chassis/chassis/Thermal
```

**回應範例：**
```json
{
    "@odata.id": "/redfish/v1/Chassis/chassis/Thermal",
    "@odata.type": "#Thermal.v1_7_0.Thermal",
    "Id": "Thermal",
    "Name": "Thermal",
    "Temperatures": [
        {
            "@odata.id": "/redfish/v1/Chassis/chassis/Thermal#/Temperatures/0",
            "Name": "CPU0 Temp",
            "ReadingCelsius": 45.5,
            "UpperThresholdCritical": 95.0,
            "UpperThresholdNonCritical": 85.0,
            "Status": {
                "State": "Enabled",
                "Health": "OK"
            }
        }
    ],
    "Fans": [
        {
            "@odata.id": "/redfish/v1/Chassis/chassis/Thermal#/Fans/0",
            "Name": "Fan 0",
            "Reading": 5000,
            "ReadingUnits": "RPM",
            "Status": {
                "State": "Enabled",
                "Health": "OK"
            }
        }
    ]
}
```

### ThermalSubsystem API

新版 Redfish 提供更細緻的散熱子系統 API：

```bash
curl -k -H "X-Auth-Token: $token" \
    https://${bmc}/redfish/v1/Chassis/chassis/ThermalSubsystem
```

---

## LED 控制

### 查詢 LED 狀態

```bash
curl -k -H "X-Auth-Token: $token" \
    https://${bmc}/redfish/v1/Chassis/chassis | jq '.IndicatorLED'
```

### 控制識別燈

```bash
# 開啟識別燈 (閃爍)
curl -k -H "X-Auth-Token: $token" \
    -H "Content-Type: application/json" \
    -X PATCH https://${bmc}/redfish/v1/Chassis/chassis \
    -d '{"IndicatorLED": "Blinking"}'

# 關閉識別燈
curl -k -H "X-Auth-Token: $token" \
    -H "Content-Type: application/json" \
    -X PATCH https://${bmc}/redfish/v1/Chassis/chassis \
    -d '{"IndicatorLED": "Off"}'
```

**IndicatorLED 值：**
| 值 | 說明 |
|------|------|
| `Off` | 關閉 |
| `Lit` | 常亮 |
| `Blinking` | 閃爍 |

---

## D-Bus 對應

| Redfish 資源 | D-Bus 路徑 |
|--------------|------------|
| Chassis | `/xyz/openbmc_project/inventory/system/chassis` |
| Sensors | `/xyz/openbmc_project/sensors/*` |
| LED | `/xyz/openbmc_project/led/groups/enclosure_identify` |

---

## 相關文件

- [RedfishOverview](RedfishOverview.md) - Redfish 概述
- [RedfishSystemService](RedfishSystemService.md) - 系統服務
- [Architecture](Architecture.md) - 架構概述

---

*最後更新：2025-12-19*
