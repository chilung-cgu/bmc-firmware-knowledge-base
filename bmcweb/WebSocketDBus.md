# D-Bus WebSocket 介面

本文件說明 bmcweb 的 D-Bus 事件 WebSocket 介面。

---

## 📋 目錄

1. [概述](#概述)
2. [連線方式](#連線方式)
3. [訊息格式](#訊息格式)
4. [訂閱過濾](#訂閱過濾)
5. [使用範例](#使用範例)

---

## 概述

D-Bus WebSocket 提供即時的 D-Bus 事件串流：

- **即時通知** - D-Bus 物件/屬性變更
- **過濾機制** - 只接收感興趣的事件
- **雙向通訊** - 動態調整訂閱

```
┌─────────────┐                    ┌─────────────┐
│   Client    │◄═══ WebSocket ═══►│   bmcweb    │
│             │                    │      │      │
└─────────────┘                    │      ▼      │
                                   │   D-Bus     │
                                   │  Monitor    │
                                   └──────┬──────┘
                                          │
                                   ┌──────▼──────┐
                                   │ OpenBMC     │
                                   │ D-Bus       │
                                   └─────────────┘
```

---

## 連線方式

### WebSocket 端點

```
wss://{bmc}/subscribe
```

### 建立連線

使用 `websocat` 工具：

```bash
websocat -k --header "X-Auth-Token:$token" \
    wss://${bmc}/subscribe
```

使用 Python：

```python
import websocket
import json

def on_message(ws, message):
    print(f"Received: {message}")

def on_open(ws):
    # 訂閱特定路徑
    ws.send(json.dumps({
        "paths": ["/xyz/openbmc_project/sensors"],
        "interfaces": ["xyz.openbmc_project.Sensor.Value"]
    }))

ws = websocket.WebSocketApp(
    f"wss://{bmc}/subscribe",
    header={"X-Auth-Token": token},
    on_message=on_message,
    on_open=on_open
)
ws.run_forever(sslopt={"cert_reqs": 0})
```

---

## 訊息格式

### 訂閱請求

客戶端發送 JSON 格式的訂閱請求：

```json
{
    "paths": ["/xyz/openbmc_project/sensors"],
    "interfaces": ["xyz.openbmc_project.Sensor.Value"]
}
```

### 事件通知

伺服器推送的事件格式：

```json
{
    "event": "PropertiesChanged",
    "path": "/xyz/openbmc_project/sensors/temperature/CPU0_Temp",
    "interface": "xyz.openbmc_project.Sensor.Value",
    "properties": {
        "Value": 45.5
    }
}
```

### 事件類型

| 事件 | 說明 |
|------|------|
| `PropertiesChanged` | 屬性變更 |
| `InterfacesAdded` | 新增介面 |
| `InterfacesRemoved` | 移除介面 |

---

## 訂閱過濾

### 路徑過濾

訂閱特定 D-Bus 路徑或子樹：

```json
{
    "paths": [
        "/xyz/openbmc_project/sensors",
        "/xyz/openbmc_project/state/host0"
    ]
}
```

### 介面過濾

訂閱特定 D-Bus 介面：

```json
{
    "interfaces": [
        "xyz.openbmc_project.Sensor.Value",
        "xyz.openbmc_project.State.Host"
    ]
}
```

### 組合過濾

路徑和介面可組合使用：

```json
{
    "paths": ["/xyz/openbmc_project/sensors"],
    "interfaces": ["xyz.openbmc_project.Sensor.Value"]
}
```

這將只接收 `/xyz/openbmc_project/sensors` 路徑下實作 `xyz.openbmc_project.Sensor.Value` 介面的物件變更。

---

## 使用範例

### 監控感測器

```python
import websocket
import json
import ssl

bmc = "192.168.1.100"
token = "your-auth-token"

def on_message(ws, message):
    data = json.loads(message)
    if data.get("event") == "PropertiesChanged":
        path = data.get("path", "")
        props = data.get("properties", {})
        if "Value" in props:
            sensor_name = path.split("/")[-1]
            print(f"{sensor_name}: {props['Value']}")

def on_open(ws):
    subscription = {
        "paths": ["/xyz/openbmc_project/sensors"],
        "interfaces": ["xyz.openbmc_project.Sensor.Value"]
    }
    ws.send(json.dumps(subscription))
    print("Subscribed to sensor updates")

ws = websocket.WebSocketApp(
    f"wss://{bmc}/subscribe",
    header={"X-Auth-Token": token},
    on_message=on_message,
    on_open=on_open
)
ws.run_forever(sslopt={"cert_reqs": ssl.CERT_NONE})
```

### 監控電源狀態

```python
subscription = {
    "paths": ["/xyz/openbmc_project/state"],
    "interfaces": [
        "xyz.openbmc_project.State.Host",
        "xyz.openbmc_project.State.BMC",
        "xyz.openbmc_project.State.Chassis"
    ]
}
```

### 使用 websocat 測試

```bash
# 安裝 websocat
cargo install websocat

# 連線並訂閱
echo '{"paths":["/xyz/openbmc_project/sensors"]}' | \
    websocat -k -H "X-Auth-Token:$token" wss://${bmc}/subscribe
```

---

## 啟用 WebSocket

WebSocket 功能需要在編譯時啟用 `rest` 選項：

```bash
meson setup builddir -Drest=enabled
```

或在 local.conf：

```bash
EXTRA_OEMESON:pn-bmcweb:append = "-Drest='enabled'"
```

---

## 與 Redfish EventService 比較

| 功能 | WebSocket | EventService |
|------|-----------|--------------|
| 協議 | WebSocket | HTTP Push / SSE |
| 事件來源 | D-Bus 原生 | Redfish 事件 |
| 過濾粒度 | D-Bus 路徑/介面 | Redfish 資源 |
| 適用場景 | 開發/除錯 | 生產環境整合 |

---

## 相關文件

- [Architecture](Architecture.md) - 架構概述
- [DBusRESTAPI](DBusRESTAPI.md) - D-Bus REST API
- [RedfishEventService](RedfishEventService.md) - Redfish 事件服務
- [WebSocketSerial](WebSocketSerial.md) - Serial Console

---

*最後更新：2025-12-19*
