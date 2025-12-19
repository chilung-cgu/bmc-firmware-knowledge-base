# Redfish TelemetryService API

本文件說明 bmcweb 的遙測服務 API，用於指標收集和報告。

---

## 📋 目錄

1. [服務概述](#服務概述)
2. [指標報告定義](#指標報告定義)
3. [指標報告](#指標報告)
4. [觸發器](#觸發器)

---

## 服務概述

TelemetryService 提供遙測和指標收集功能：

- **指標報告定義** - 定義要收集的指標
- **指標報告** - 實際的指標數據
- **觸發器** - 基於閾值的告警

---

## 查詢 TelemetryService

```bash
curl -k -H "X-Auth-Token: $token" \
    https://${bmc}/redfish/v1/TelemetryService
```

**回應範例：**
```json
{
    "@odata.id": "/redfish/v1/TelemetryService",
    "@odata.type": "#TelemetryService.v1_3_0.TelemetryService",
    "Id": "TelemetryService",
    "Name": "Telemetry Service",
    "Status": {
        "State": "Enabled",
        "Health": "OK"
    },
    "MaxReports": 20,
    "MinCollectionInterval": "PT1S",
    "MetricReportDefinitions": {
        "@odata.id": "/redfish/v1/TelemetryService/MetricReportDefinitions"
    },
    "MetricReports": {
        "@odata.id": "/redfish/v1/TelemetryService/MetricReports"
    },
    "Triggers": {
        "@odata.id": "/redfish/v1/TelemetryService/Triggers"
    }
}
```

---

## 指標報告定義

### 列出定義

```bash
curl -k -H "X-Auth-Token: $token" \
    https://${bmc}/redfish/v1/TelemetryService/MetricReportDefinitions
```

### 建立報告定義

```bash
curl -k -H "X-Auth-Token: $token" \
    -H "Content-Type: application/json" \
    -X POST https://${bmc}/redfish/v1/TelemetryService/MetricReportDefinitions \
    -d '{
        "Id": "CPUTempReport",
        "Name": "CPU Temperature Report",
        "MetricReportDefinitionType": "Periodic",
        "ReportActions": ["RedfishEvent"],
        "Schedule": {
            "RecurrenceInterval": "PT30S"
        },
        "Metrics": [
            {
                "MetricId": "CPU0Temp",
                "MetricProperties": [
                    "/redfish/v1/Chassis/chassis/Sensors/temperature_CPU0#Reading"
                ]
            }
        ]
    }'
```

### 定義參數

| 參數 | 說明 |
|------|------|
| `MetricReportDefinitionType` | `Periodic` - 週期性, `OnChange` - 變更時, `OnRequest` - 請求時 |
| `ReportActions` | `RedfishEvent` - 發送事件, `LogToMetricReportsCollection` - 記錄 |
| `Schedule.RecurrenceInterval` | 收集間隔 (ISO 8601 duration) |
| `Metrics` | 要收集的指標陣列 |

### 查詢定義詳情

```bash
curl -k -H "X-Auth-Token: $token" \
    https://${bmc}/redfish/v1/TelemetryService/MetricReportDefinitions/CPUTempReport
```

### 刪除定義

```bash
curl -k -H "X-Auth-Token: $token" \
    -X DELETE https://${bmc}/redfish/v1/TelemetryService/MetricReportDefinitions/CPUTempReport
```

---

## 指標報告

### 列出報告

```bash
curl -k -H "X-Auth-Token: $token" \
    https://${bmc}/redfish/v1/TelemetryService/MetricReports
```

### 查詢報告

```bash
curl -k -H "X-Auth-Token: $token" \
    https://${bmc}/redfish/v1/TelemetryService/MetricReports/CPUTempReport
```

**回應範例：**
```json
{
    "@odata.id": "/redfish/v1/TelemetryService/MetricReports/CPUTempReport",
    "@odata.type": "#MetricReport.v1_4_0.MetricReport",
    "Id": "CPUTempReport",
    "Name": "CPU Temperature Report",
    "Timestamp": "2025-12-19T05:00:00+00:00",
    "MetricReportDefinition": {
        "@odata.id": "/redfish/v1/TelemetryService/MetricReportDefinitions/CPUTempReport"
    },
    "MetricValues": [
        {
            "MetricId": "CPU0Temp",
            "MetricValue": "45.5",
            "Timestamp": "2025-12-19T05:00:00+00:00",
            "MetricProperty": "/redfish/v1/Chassis/chassis/Sensors/temperature_CPU0#Reading"
        }
    ]
}
```

---

## 觸發器

### 列出觸發器

```bash
curl -k -H "X-Auth-Token: $token" \
    https://${bmc}/redfish/v1/TelemetryService/Triggers
```

### 建立觸發器

```bash
curl -k -H "X-Auth-Token: $token" \
    -H "Content-Type: application/json" \
    -X POST https://${bmc}/redfish/v1/TelemetryService/Triggers \
    -d '{
        "Id": "CPUTempTrigger",
        "Name": "CPU Temperature Trigger",
        "TriggerActions": ["RedfishEvent"],
        "NumericThresholds": {
            "UpperCritical": {
                "Reading": 95,
                "Activation": "Increasing"
            },
            "UpperWarning": {
                "Reading": 85,
                "Activation": "Increasing"
            }
        },
        "MetricProperties": [
            "/redfish/v1/Chassis/chassis/Sensors/temperature_CPU0#Reading"
        ]
    }'
```

### 觸發器參數

| 參數 | 說明 |
|------|------|
| `TriggerActions` | 觸發動作: `RedfishEvent`, `LogToLogService` |
| `NumericThresholds` | 數值閾值 |
| `DiscreteTriggerCondition` | 離散觸發條件 |

### 閾值類型

| 閾值 | 說明 |
|------|------|
| `UpperCritical` | 上限嚴重 |
| `UpperWarning` | 上限警告 |
| `LowerWarning` | 下限警告 |
| `LowerCritical` | 下限嚴重 |

---

## 使用範例

### 完整監控設置

```bash
# 1. 建立多指標報告定義
curl -k -H "X-Auth-Token: $token" \
    -H "Content-Type: application/json" \
    -X POST https://${bmc}/redfish/v1/TelemetryService/MetricReportDefinitions \
    -d '{
        "Id": "SystemHealthReport",
        "Name": "System Health Report",
        "MetricReportDefinitionType": "Periodic",
        "ReportActions": ["LogToMetricReportsCollection", "RedfishEvent"],
        "Schedule": {
            "RecurrenceInterval": "PT60S"
        },
        "Metrics": [
            {
                "MetricId": "CPU0Temp",
                "MetricProperties": [
                    "/redfish/v1/Chassis/chassis/Sensors/temperature_CPU0#Reading"
                ]
            },
            {
                "MetricId": "SystemPower",
                "MetricProperties": [
                    "/redfish/v1/Chassis/chassis/Power#PowerControl/0/PowerConsumedWatts"
                ]
            },
            {
                "MetricId": "Fan0Speed",
                "MetricProperties": [
                    "/redfish/v1/Chassis/chassis/Thermal#Fans/0/Reading"
                ]
            }
        ]
    }'

# 2. 建立溫度觸發器
curl -k -H "X-Auth-Token: $token" \
    -H "Content-Type: application/json" \
    -X POST https://${bmc}/redfish/v1/TelemetryService/Triggers \
    -d '{
        "Id": "HighTempAlert",
        "Name": "High Temperature Alert",
        "TriggerActions": ["RedfishEvent"],
        "NumericThresholds": {
            "UpperCritical": {
                "Reading": 95,
                "Activation": "Increasing"
            }
        },
        "MetricProperties": [
            "/redfish/v1/Chassis/chassis/Sensors/temperature_CPU0#Reading"
        ]
    }'

# 3. 查詢最新報告
curl -k -H "X-Auth-Token: $token" \
    https://${bmc}/redfish/v1/TelemetryService/MetricReports/SystemHealthReport
```

---

## 相關文件

- [RedfishChassisService](RedfishChassisService.md) - 感測器 API
- [RedfishEventService](RedfishEventService.md) - 事件服務
- [Architecture](Architecture.md) - 架構概述

---

*最後更新：2025-12-19*
