# Serial Console WebSocket

本文件說明 bmcweb 的 Serial Console WebSocket 介面。

---

## 📋 目錄

1. [概述](#概述)
2. [連線方式](#連線方式)
3. [使用範例](#使用範例)
4. [與 obmc-console 整合](#與-obmc-console-整合)

---

## 概述

Serial Console WebSocket 提供透過瀏覽器存取主機序列主控台的功能：

```
┌─────────────┐                    ┌─────────────┐
│  Browser/   │                    │   bmcweb    │
│  Terminal   │◄═══ WebSocket ═══►│      │      │
│             │                    │      ▼      │
└─────────────┘                    │ obmc-console│
                                   │    client   │
                                   └──────┬──────┘
                                          │ Unix Socket
                                   ┌──────▼──────┐
                                   │obmc-console │
                                   │   server    │
                                   └──────┬──────┘
                                          │ TTY
                                   ┌──────▼──────┐
                                   │ Host Serial │
                                   │    Port     │
                                   └─────────────┘
```

---

## 連線方式

### WebSocket 端點

| 端點 | 說明 |
|------|------|
| `wss://{bmc}/console0` | 主要 Serial Console |
| `wss://{bmc}/console/{id}` | 指定 Console ID |

### 使用 websocat

```bash
websocat -k -H "X-Auth-Token:$token" wss://${bmc}/console0
```

### 使用 JavaScript

```javascript
const ws = new WebSocket(`wss://${bmc}/console0`, [], {
    headers: {
        'X-Auth-Token': token
    }
});

ws.onopen = () => {
    console.log('Connected to serial console');
};

ws.onmessage = (event) => {
    // 處理來自 serial console 的輸出
    process.stdout.write(event.data);
};

// 發送輸入到 console
document.addEventListener('keypress', (e) => {
    ws.send(e.key);
});
```

---

## 使用範例

### 基本連線

```bash
# 使用 websocat 連線
websocat -k --header "X-Auth-Token:$token" wss://${bmc}/console0

# 連線後可直接與主機互動
# 輸入會傳送到主機
# 輸出會顯示在終端
```

### Python 客戶端

```python
import websocket
import sys
import threading

bmc = "192.168.1.100"
token = "your-auth-token"

def on_message(ws, message):
    sys.stdout.write(message)
    sys.stdout.flush()

def on_open(ws):
    print("Connected to serial console")
    
    def send_input():
        while True:
            char = sys.stdin.read(1)
            ws.send(char)
    
    thread = threading.Thread(target=send_input)
    thread.daemon = True
    thread.start()

ws = websocket.WebSocketApp(
    f"wss://{bmc}/console0",
    header={"X-Auth-Token": token},
    on_message=on_message,
    on_open=on_open
)
ws.run_forever(sslopt={"cert_reqs": 0})
```

### 在 WebUI 中使用

`webui-vue` 提供整合的 Serial Console 介面，透過此 WebSocket 端點連線。

---

## 與 obmc-console 整合

### 架構

bmcweb 透過 Unix socket 與 `obmc-console` 服務通訊：

```
/run/obmc-console.sock
```

### obmc-console 服務

`obmc-console` 是獨立的服務，管理實際的 TTY 設備：

```ini
# /lib/systemd/system/obmc-console@.service
[Service]
ExecStart=/usr/bin/obmc-console-server --socket-id %i
```

### 多 Console 支援

若系統有多個 Serial Console：

```bash
# Console 0
wss://${bmc}/console0

# Console 1  
wss://${bmc}/console1
```

---

## 權限要求

存取 Serial Console 需要適當的權限：

| 角色 | 存取權限 |
|------|----------|
| Administrator | ✅ |
| Operator | ✅ |
| ReadOnly | ❌ |

---

## 相關文件

- [WebSocketKVM](WebSocketKVM.md) - KVM WebSocket
- [WebSocketDBus](WebSocketDBus.md) - D-Bus WebSocket
- [Authentication](Authentication.md) - 認證機制

---

*最後更新：2025-12-19*
