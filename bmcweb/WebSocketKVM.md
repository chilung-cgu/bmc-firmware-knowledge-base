# KVM WebSocket 介面

本文件說明 bmcweb 的 KVM (Keyboard, Video, Mouse) WebSocket 介面。

---

## 📋 目錄

1. [概述](#概述)
2. [連線方式](#連線方式)
3. [RFB 協議](#rfb-協議)
4. [與 WebUI 整合](#與-webui-整合)

---

## 概述

KVM WebSocket 提供遠端桌面存取功能：

- **協議** - RFB (Remote Framebuffer Protocol)，又稱 VNC 協議
- **傳輸** - WebSocket over HTTPS
- **客戶端** - webui-vue 或任何 RFB-over-WebSocket 客戶端

```
┌─────────────────┐                    ┌─────────────────┐
│   Browser       │                    │     bmcweb       │
│  ┌───────────┐  │                    │                  │
│  │ noVNC /   │◄═══ WebSocket/RFB ══►│  KVM Handler     │
│  │ webui-vue │  │                    │        │         │
│  └───────────┘  │                    │        ▼         │
└─────────────────┘                    │  Video/Input     │
                                       │    Driver        │
                                       └────────┬─────────┘
                                                │
                                       ┌────────▼─────────┐
                                       │    ASPEED/       │
                                       │  Video Capture   │
                                       │    Hardware      │
                                       └──────────────────┘
```

---

## 連線方式

### WebSocket 端點

```
wss://{bmc}/kvm/0
```

### RFB 握手

連線後需要執行 RFB 協議握手：

1. **版本協商** - 交換 RFB 版本
2. **安全類型** - 選擇認證方式
3. **ClientInit** - 客戶端初始化
4. **ServerInit** - 伺服器發送桌面資訊

### 使用 noVNC

[noVNC](https://github.com/novnc/noVNC) 是常用的 Web VNC 客戶端：

```html
<!DOCTYPE html>
<html>
<head>
    <script src="noVNC/vnc.js"></script>
</head>
<body>
    <div id="screen"></div>
    <script>
        const rfb = new RFB(
            document.getElementById('screen'),
            `wss://${bmc}/kvm/0`,
            {
                credentials: { token: authToken }
            }
        );
    </script>
</body>
</html>
```

---

## RFB 協議

### 支援的編碼

| 編碼 | 說明 |
|------|------|
| Raw | 原始像素 |
| Tight | 壓縮編碼 |
| ZRLE | Zlib Run-Length Encoding |

### 支援的像素格式

| 格式 | 說明 |
|------|------|
| 32-bit | BGRA/RGBA |
| 16-bit | RGB565 |
| 8-bit | 調色盤 |

### 客戶端到伺服器訊息

| 類型 | 說明 |
|------|------|
| KeyEvent | 鍵盤事件 |
| PointerEvent | 滑鼠事件 |
| FramebufferUpdateRequest | 請求畫面更新 |
| ClientCutText | 剪貼簿文字 |

### 伺服器到客戶端訊息

| 類型 | 說明 |
|------|------|
| FramebufferUpdate | 畫面更新 |
| ServerCutText | 剪貼簿文字 |

---

## 與 WebUI 整合

### webui-vue KVM 元件

`webui-vue` 內建 KVM 功能，透過 bmcweb 的 WebSocket 端點提供遠端桌面：

1. 導航到 **Operations** → **KVM**
2. 自動連線到 `/kvm/0`
3. 全螢幕模式可用

### KVM 操作

| 功能 | 說明 |
|------|------|
| 全螢幕 | 放大 KVM 視窗 |
| 特殊按鍵 | Ctrl+Alt+Del 等 |
| 畫質調整 | 調整壓縮等級 |

---

## 硬體要求

KVM 功能需要硬體支援：

### ASPEED AST2500/2600

ASPEED BMC SoC 內建視訊擷取功能：

- Video Engine 擷取 VGA 輸出
- 支援 H.264 編碼 (AST2600)
- 低延遲 USB HID 模擬

### 驅動程式

```
/dev/video0  - 視訊擷取設備
/dev/hidg0   - USB HID Gadget (鍵盤)
/dev/hidg1   - USB HID Gadget (滑鼠)
```

---

## 配置選項

### 啟用 KVM

```bash
EXTRA_OEMESON:pn-bmcweb:append = "-Dkvm='enabled'"
```

### 視訊品質

可透過 RFB 協議調整：

- 壓縮等級
- JPEG 品質
- 編碼選擇

---

## 故障排除

### 無法連線

1. 確認 KVM 功能已啟用
2. 檢查硬體支援
3. 驗證認證 token

### 黑畫面

1. 確認主機已開機
2. 檢查 `/dev/video0` 是否存在
3. 查看 bmcweb 日誌

```bash
journalctl -u bmcweb | grep -i kvm
```

### 延遲過高

1. 降低畫質設定
2. 使用有線網路
3. 選擇較高效的編碼

---

## 相關文件

- [WebSocketSerial](WebSocketSerial.md) - Serial Console
- [Architecture](Architecture.md) - 架構概述
- [Troubleshooting](Troubleshooting.md) - 故障排除

---

*最後更新：2025-12-19*
