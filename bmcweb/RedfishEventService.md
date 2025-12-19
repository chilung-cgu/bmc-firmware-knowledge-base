# Redfish EventService API

本文件說明 bmcweb 的事件服務 API，用於事件訂閱和通知。

---

## 📋 目錄

1. [服務概述](#服務概述)
2. [事件訂閱](#事件訂閱)
3. [訂閱管理](#訂閱管理)
4. [SSE 支援](#sse-支援)
5. [事件類型](#事件類型)

---

## 服務概述

EventService 提供非同步事件通知機制：

- **事件訂閱** - 訂閱特定類型的事件
- **Push 通知** - HTTP POST 到訂閱者 URL
- **SSE** - Server-Sent Events 串流

---

## 事件訂閱

### 查詢 EventService

```bash
curl -k -H "X-Auth-Token: $token" \
    https://${bmc}/redfish/v1/EventService
```

**回應範例：**
```json
{
    "@odata.id": "/redfish/v1/EventService",
    "@odata.type": "#EventService.v1_7_0.EventService",
    "Id": "EventService",
    "Name": "Event Service",
    "ServiceEnabled": true,
    "DeliveryRetryAttempts": 3,
    "DeliveryRetryIntervalSeconds": 30,
    "EventTypesForSubscription": [
        "StatusChange",
        "ResourceAdded",
        "ResourceRemoved",
        "ResourceUpdated",
        "Alert"
    ],
    "Subscriptions": {
        "@odata.id": "/redfish/v1/EventService/Subscriptions"
    },
    "Actions": {
        "#EventService.SubmitTestEvent": {
            "target": "/redfish/v1/EventService/Actions/EventService.SubmitTestEvent"
        }
    },
    "ServerSentEventUri": "/redfish/v1/EventService/SSE"
}
```

### 建立訂閱

```bash
curl -k -H "X-Auth-Token: $token" \
    -H "Content-Type: application/json" \
    -X POST https://${bmc}/redfish/v1/EventService/Subscriptions \
    -d '{
        "Destination": "https://192.168.1.50:8888/events",
        "Protocol": "Redfish",
        "SubscriptionType": "RedfishEvent",
        "Context": "MyEventSubscription",
        "EventFormatType": "Event"
    }'
```

### 訂閱參數

| 參數 | 說明 | 範例 |
|------|------|------|
| `Destination` | 接收事件的 URL | `https://server/events` |
| `Protocol` | 通知協議 | `Redfish` |
| `SubscriptionType` | 訂閱類型 | `RedfishEvent`, `SSE` |
| `Context` | 自定義上下文字串 | `MyContext` |
| `EventFormatType` | 事件格式 | `Event`, `MetricReport` |
| `RegistryPrefixes` | 過濾的 Registry 前綴 | `["OpenBMC"]` |
| `ResourceTypes` | 過濾的資源類型 | `["ComputerSystem"]` |
| `MessageIds` | 過濾的訊息 ID | `["Alert.1.0.LanDisconnect"]` |

---

## 訂閱管理

### 列出所有訂閱

```bash
curl -k -H "X-Auth-Token: $token" \
    https://${bmc}/redfish/v1/EventService/Subscriptions
```

### 查詢特定訂閱

```bash
curl -k -H "X-Auth-Token: $token" \
    https://${bmc}/redfish/v1/EventService/Subscriptions/{SubscriptionId}
```

**回應範例：**
```json
{
    "@odata.id": "/redfish/v1/EventService/Subscriptions/1",
    "@odata.type": "#EventDestination.v1_11_0.EventDestination",
    "Id": "1",
    "Name": "Event Subscription",
    "Destination": "https://192.168.1.50:8888/events",
    "Protocol": "Redfish",
    "SubscriptionType": "RedfishEvent",
    "Context": "MyEventSubscription",
    "EventFormatType": "Event",
    "DeliveryRetryPolicy": "RetryForever"
}
```

### 更新訂閱

```bash
curl -k -H "X-Auth-Token: $token" \
    -H "Content-Type: application/json" \
    -X PATCH https://${bmc}/redfish/v1/EventService/Subscriptions/1 \
    -d '{
        "Context": "UpdatedContext",
        "DeliveryRetryPolicy": "SuspendRetries"
    }'
```

### 刪除訂閱

```bash
curl -k -H "X-Auth-Token: $token" \
    -X DELETE https://${bmc}/redfish/v1/EventService/Subscriptions/1
```

---

## SSE 支援

### 使用 SSE

bmcweb 支援 Server-Sent Events 作為替代的事件接收方式：

```bash
# 使用 curl 接收 SSE
curl -k -H "X-Auth-Token: $token" \
    -H "Accept: text/event-stream" \
    https://${bmc}/redfish/v1/EventService/SSE
```

### SSE 過濾

可透過 query string 過濾事件：

```bash
# 僅接收 Alert 類型事件
curl -k -H "X-Auth-Token: $token" \
    -H "Accept: text/event-stream" \
    "https://${bmc}/redfish/v1/EventService/SSE?$filter=EventType eq 'Alert'"
```

### SSE 事件格式

```
event: message
data: {
data:   "@odata.type": "#Event.v1_6_0.Event",
data:   "Events": [{
data:     "EventType": "Alert",
data:     "Message": "Temperature threshold exceeded"
data:   }]
data: }

```

---

## 事件類型

### EventType

| 類型 | 說明 |
|------|------|
| `StatusChange` | 資源狀態變更 |
| `ResourceAdded` | 新增資源 |
| `ResourceRemoved` | 移除資源 |
| `ResourceUpdated` | 資源更新 |
| `Alert` | 告警事件 |

### 事件格式範例

```json
{
    "@odata.type": "#Event.v1_6_0.Event",
    "Id": "1",
    "Name": "Event Array",
    "Context": "MyContext",
    "Events": [
        {
            "EventType": "Alert",
            "Severity": "Warning",
            "Message": "Temperature threshold exceeded on sensor CPU0_Temp",
            "MessageId": "OpenBMC.0.2.SensorThresholdWarning",
            "MessageArgs": ["CPU0_Temp", "85", "80"],
            "OriginOfCondition": {
                "@odata.id": "/redfish/v1/Chassis/chassis/Sensors/temperature_CPU0"
            },
            "EventTimestamp": "2025-12-19T05:00:00+00:00"
        }
    ]
}
```

---

## 測試事件

### 發送測試事件

```bash
curl -k -H "X-Auth-Token: $token" \
    -H "Content-Type: application/json" \
    -X POST https://${bmc}/redfish/v1/EventService/Actions/EventService.SubmitTestEvent \
    -d '{
        "EventId": "TestEvent1",
        "EventType": "Alert",
        "Message": "This is a test event",
        "MessageId": "OpenBMC.0.1.TestEvent",
        "Severity": "OK"
    }'
```

---

## 完整訂閱範例

```bash
# 1. 建立訂閱
curl -k -H "X-Auth-Token: $token" \
    -H "Content-Type: application/json" \
    -X POST https://${bmc}/redfish/v1/EventService/Subscriptions \
    -d '{
        "Context": "Production Server Monitor",
        "Destination": "https://monitor.example.com:8443/redfish/events",
        "Protocol": "Redfish",
        "SubscriptionType": "RedfishEvent",
        "EventFormatType": "Event",
        "DeliveryRetryPolicy": "RetryForever",
        "RegistryPrefixes": ["OpenBMC"],
        "ResourceTypes": ["ComputerSystem", "Chassis"]
    }'

# 2. 儲存訂閱 ID
export sub_id=$(curl -k -H "X-Auth-Token: $token" \
    https://${bmc}/redfish/v1/EventService/Subscriptions \
    | jq -r '.Members[0]."@odata.id"' | cut -d'/' -f6)

# 3. 驗證訂閱
curl -k -H "X-Auth-Token: $token" \
    https://${bmc}/redfish/v1/EventService/Subscriptions/${sub_id}

# 4. 發送測試事件
curl -k -H "X-Auth-Token: $token" \
    -H "Content-Type: application/json" \
    -X POST https://${bmc}/redfish/v1/EventService/Actions/EventService.SubmitTestEvent

# 5. 刪除訂閱
curl -k -H "X-Auth-Token: $token" \
    -X DELETE https://${bmc}/redfish/v1/EventService/Subscriptions/${sub_id}
```

---

## 相關文件

- [RedfishOverview](RedfishOverview.md) - Redfish 概述
- [WebSocketDBus](WebSocketDBus.md) - D-Bus 事件 WebSocket
- [Troubleshooting](Troubleshooting.md) - 故障排除

---

*最後更新：2025-12-19*
