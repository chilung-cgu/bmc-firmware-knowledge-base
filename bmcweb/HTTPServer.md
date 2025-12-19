# HTTP 伺服器實作

本文件說明 bmcweb 的 HTTP 伺服器實作細節。

---

## 📋 目錄

1. [概述](#概述)
2. [協議支援](#協議支援)
3. [TLS 配置](#tls-配置)
4. [壓縮支援](#壓縮支援)
5. [連線管理](#連線管理)

---

## 概述

bmcweb 使用 **Boost.Beast** 和 **Boost.Asio** 實作高效能 HTTP 伺服器：

- **非同步 I/O** - 單執行緒事件驅動
- **HTTP/1.1 & HTTP/2** - 完整協議支援
- **TLS** - OpenSSL 整合
- **壓縮** - gzip, zstd 支援

```
┌─────────────────────────────────────────────────────────────────┐
│                         bmcweb HTTP Server                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │                    Boost.Asio                                ││
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  ││
│  │  │ io_context  │  │   Timers    │  │  Socket Operations  │  ││
│  │  └─────────────┘  └─────────────┘  └─────────────────────┘  ││
│  └─────────────────────────────────────────────────────────────┘│
│                               │                                  │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │                    Boost.Beast                               ││
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  ││
│  │  │ HTTP Parser │  │  WebSocket  │  │  HTTP Serializer    │  ││
│  │  └─────────────┘  └─────────────┘  └─────────────────────┘  ││
│  └─────────────────────────────────────────────────────────────┘│
│                               │                                  │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │                    OpenSSL (TLS)                             ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 協議支援

### HTTP/1.1

標準 HTTP/1.1 支援：

- Keep-Alive 連線
- Chunked Transfer Encoding
- Range Requests

### HTTP/2

HTTP/2 透過以下方式啟用：

| 方式 | 說明 |
|------|------|
| **ALPN** | TLS 連線時協商 (h2) |
| **h2c Upgrade** | 非加密連線升級 |

### 協議協商

```
Client                              Server
  │                                    │
  │─── ClientHello (ALPN: h2, http/1.1) │
  │                                    │
  │◄── ServerHello (ALPN: h2) ─────────│
  │                                    │
  │════════ HTTP/2 Connection ═════════│
```

---

## TLS 配置

### 憑證管理

bmcweb 在以下位置尋找 TLS 憑證：

| 檔案 | 說明 |
|------|------|
| `/etc/ssl/certs/https/server.pem` | 伺服器憑證 |
| `/etc/ssl/private/https/server.key` | 私鑰 |

### 自動憑證生成

當找不到有效憑證時，bmcweb 會自動生成自簽憑證：

```cpp
// 憑證參數
- 金鑰長度: 2048 bits RSA
- 有效期: 10 年
- CN: BMC 主機名
```

### TLS 版本

| 版本 | 支援 |
|------|------|
| TLS 1.0 | ❌ 停用 |
| TLS 1.1 | ❌ 停用 |
| TLS 1.2 | ✅ |
| TLS 1.3 | ✅ |

### 透過 Redfish 管理憑證

```bash
# 列出憑證
curl -k -H "X-Auth-Token: $token" \
    https://${bmc}/redfish/v1/Managers/bmc/NetworkProtocol/HTTPS/Certificates

# 替換憑證
curl -k -H "X-Auth-Token: $token" \
    -H "Content-Type: application/json" \
    -X POST https://${bmc}/redfish/v1/CertificateService/Actions/CertificateService.ReplaceCertificate \
    -d '{
        "CertificateUri": "/redfish/v1/Managers/bmc/NetworkProtocol/HTTPS/Certificates/1",
        "CertificateType": "PEM",
        "CertificateString": "-----BEGIN CERTIFICATE-----\n...\n-----END CERTIFICATE-----"
    }'
```

---

## 壓縮支援

### 支援的格式

| 格式 | Content-Encoding |
|------|------------------|
| gzip | `gzip` |
| zstd | `zstd` |

### 壓縮協商

根據客戶端 `Accept-Encoding` header 選擇：

```http
GET /redfish/v1/Systems HTTP/1.1
Accept-Encoding: gzip, deflate, zstd

HTTP/1.1 200 OK
Content-Encoding: zstd
```

### 壓縮行為

- 只壓縮 text/json 等可壓縮類型
- 小回應不壓縮 (overhead 過大)
- 串流回應支援壓縮

---

## 連線管理

### 逾時設定

| 參數 | 預設值 | 說明 |
|------|--------|------|
| 連線逾時 | 30s | 無活動斷線 |
| 請求逾時 | 30s | 請求處理超時 |
| Keep-Alive | 啟用 | 持久連線 |

### 並行連線

bmcweb 使用單執行緒處理所有連線：

```
┌─────────────────────────────────────────────────────┐
│                    io_context                        │
│  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐           │
│  │Conn1│ │Conn2│ │Conn3│ │Conn4│ │Conn5│ ...       │
│  └──┬──┘ └──┬──┘ └──┬──┘ └──┬──┘ └──┬──┘           │
│     │       │       │       │       │               │
│     └───────┴───────┴───────┴───────┘               │
│                     │                                │
│              Event Loop (run())                      │
└─────────────────────────────────────────────────────┘
```

### 監聽端口

| 端口 | 用途 |
|------|------|
| 443 | HTTPS (生產環境) |
| 80 | HTTP 重導向到 HTTPS |
| 18080 | 開發/測試用 HTTP |

---

## 請求處理

### 請求生命週期

```
1. Accept Connection
       │
2. TLS Handshake (if HTTPS)
       │
3. Read HTTP Request
       │
4. Parse Headers & Body
       │
5. Authentication
       │
6. Route Matching
       │
7. Handler Execution
       │
8. Send Response
       │
9. Keep-Alive or Close
```

### 錯誤處理

| HTTP 狀態碼 | 情況 |
|-------------|------|
| 400 | 請求格式錯誤 |
| 401 | 未認證 |
| 403 | 權限不足 |
| 404 | 路徑不存在 |
| 405 | 方法不允許 |
| 500 | 內部錯誤 |
| 503 | 服務暫時不可用 |

---

## 持久化資料

bmcweb 儲存持久化資料於：

| 資料 | 位置 |
|------|------|
| Session | `/var/lib/bmcweb/sessions.json` |
| 配置 | `/var/lib/bmcweb/persistent_data.json` |

### 暫存檔案

多部分上傳使用：
```
/tmp/bmcweb/
```

此目錄由 systemd 在服務重啟時清理。

---

## 相關文件

- [Architecture](Architecture.md) - 架構概述
- [WebSocketDBus](WebSocketDBus.md) - WebSocket 處理
- [Authentication](Authentication.md) - 認證機制
- [Configuration](Configuration.md) - 配置選項

---

*最後更新：2025-12-19*
