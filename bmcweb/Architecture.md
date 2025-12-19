# bmcweb 架構概述

本文件說明 bmcweb 的系統架構、核心元件以及處理流程。

---

## 📋 目錄

1. [架構總覽](#架構總覽)
2. [核心元件](#核心元件)
3. [請求處理流程](#請求處理流程)
4. [D-Bus 整合](#d-bus-整合)
5. [協議支援](#協議支援)

---

## 架構總覽

bmcweb 採用事件驅動的非同步架構，基於 Boost.Beast 和 Boost.Asio 實作。主要特色：

- **單執行緒事件循環** - 使用 io_context 處理所有 I/O
- **非同步處理** - 所有 D-Bus 呼叫和 HTTP 處理皆為非同步
- **路由分發** - 根據 URL 路徑將請求分發至對應 handler

```
┌───────────────────────────────────────────────────────────────────┐
│                          bmcweb                                    │
├───────────────────────────────────────────────────────────────────┤
│                                                                    │
│  ┌──────────────────────────────────────────────────────────────┐ │
│  │                    HTTP Server Layer                          │ │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐   │ │
│  │  │  HTTP/1.1   │  │   HTTP/2    │  │   TLS (OpenSSL)     │   │ │
│  │  └─────────────┘  └─────────────┘  └─────────────────────┘   │ │
│  └──────────────────────────────────────────────────────────────┘ │
│                               │                                    │
│  ┌──────────────────────────────────────────────────────────────┐ │
│  │                    Router Layer                               │ │
│  │  ┌─────────────────────────────────────────────────────────┐ │ │
│  │  │  URL Pattern Matching → Handler Dispatch                 │ │ │
│  │  └─────────────────────────────────────────────────────────┘ │ │
│  └──────────────────────────────────────────────────────────────┘ │
│                               │                                    │
│  ┌──────────────────────────────────────────────────────────────┐ │
│  │                   Handler Layer                               │ │
│  │  ┌───────────┐ ┌───────────┐ ┌───────────┐ ┌──────────────┐  │ │
│  │  │  Redfish  │ │ D-Bus REST│ │ WebSocket │ │ Static Files │  │ │
│  │  └───────────┘ └───────────┘ └───────────┘ └──────────────┘  │ │
│  └──────────────────────────────────────────────────────────────┘ │
│                               │                                    │
│  ┌──────────────────────────────────────────────────────────────┐ │
│  │                    D-Bus Layer                                │ │
│  │  ┌─────────────────────────────────────────────────────────┐ │ │
│  │  │              sdbusplus (Async D-Bus)                     │ │ │
│  │  └─────────────────────────────────────────────────────────┘ │ │
│  └──────────────────────────────────────────────────────────────┘ │
│                                                                    │
└───────────────────────────────────────────────────────────────────┘
```

---

## 核心元件

### 1. App 類別

`App` 是 bmcweb 的核心類別，管理整個應用程式生命週期：

| 職責 | 說明 |
|------|------|
| **HTTP 監聽** | 建立 HTTP/HTTPS 監聽器 |
| **路由註冊** | 提供 `BMCWEB_ROUTE` 巨集註冊路由 |
| **連線管理** | 管理客戶端連線 |
| **事件循環** | 執行 io_context |

### 2. Router

Router 負責 URL 匹配和請求分發：

```cpp
// 路由註冊範例
BMCWEB_ROUTE(app, "/redfish/v1/Systems/<str>/")
    .privileges(redfish::privileges::getComputerSystem)
    .methods(boost::beast::http::verb::get)(
        std::bind_front(handleSystemGet, std::ref(app)));
```

**路由特色：**
- 支援 URL 參數 (`<str>`, `<int>`)
- 權限檢查
- HTTP 方法限制

### 3. Request/Response

| 類別 | 說明 |
|------|------|
| `crow::Request` | 封裝 HTTP 請求，包含 headers、body、URL |
| `crow::Response` | 封裝 HTTP 回應，支援 JSON、檔案、串流 |

### 4. 認證中介軟體

認證請求在路由處理前進行驗證：

```
Request → AuthN Middleware → Authorization Check → Route Handler
```

支援的認證方式：
- Basic Authentication (RFC 7617)
- Session/Cookie Authentication
- Mutual TLS (mTLS)
- X-Auth-Token (Redfish DSP0266)

---

## 請求處理流程

### HTTP 請求處理

```
┌─────────────┐
│   Client    │
└──────┬──────┘
       │ 1. HTTPS Request
       ▼
┌──────────────────┐
│  TLS Handshake   │
│   (OpenSSL)      │
└────────┬─────────┘
         │ 2. Decrypt
         ▼
┌──────────────────┐
│  HTTP Parser     │
│ (Boost.Beast)    │
└────────┬─────────┘
         │ 3. Parse
         ▼
┌──────────────────┐
│  Authentication  │
│   Middleware     │
└────────┬─────────┘
         │ 4. Validate
         ▼
┌──────────────────┐
│     Router       │
│  (URL Matching)  │
└────────┬─────────┘
         │ 5. Dispatch
         ▼
┌──────────────────┐
│  Route Handler   │
│ (e.g., Redfish)  │
└────────┬─────────┘
         │ 6. D-Bus calls
         ▼
┌──────────────────┐
│   D-Bus Layer    │
│  (sdbusplus)     │
└────────┬─────────┘
         │ 7. Response
         ▼
┌──────────────────┐
│  JSON Response   │
└──────────────────┘
```

### WebSocket 處理

WebSocket 連線建立後維持長連線：

1. **握手** - HTTP Upgrade 請求
2. **認證** - 驗證使用者權限
3. **訊息迴圈** - 雙向訊息傳遞
4. **關閉** - 清理資源

---

## D-Bus 整合

bmcweb 透過 **sdbusplus** 與 OpenBMC 服務通訊：

### D-Bus 到 Redfish 轉譯

```
Redfish URI                    D-Bus Object Path
────────────                   ─────────────────
/redfish/v1/Systems/system  →  /xyz/openbmc_project/state/host0
/redfish/v1/Chassis/chassis →  /xyz/openbmc_project/inventory/...
/redfish/v1/Managers/bmc    →  /xyz/openbmc_project/software/...
```

### 非同步 D-Bus 呼叫模式

```cpp
// 典型的非同步 D-Bus 呼叫
sdbusplus::asio::getProperty<std::string>(
    *crow::connections::systemBus,
    "xyz.openbmc_project.State.Host",
    "/xyz/openbmc_project/state/host0",
    "xyz.openbmc_project.State.Host",
    "CurrentHostState",
    [asyncResp](const boost::system::error_code& ec,
                const std::string& hostState) {
        if (ec) {
            // 錯誤處理
            return;
        }
        // 處理結果
        asyncResp->res.jsonValue["PowerState"] = hostState;
    });
```

### 常用 D-Bus 服務

| D-Bus 服務 | 用途 |
|------------|------|
| `xyz.openbmc_project.State.Host` | Host 電源狀態 |
| `xyz.openbmc_project.State.BMC` | BMC 狀態 |
| `xyz.openbmc_project.Sensor.*` | 感測器數值 |
| `xyz.openbmc_project.Logging` | 日誌服務 |
| `xyz.openbmc_project.Software.*` | 韌體管理 |

---

## 協議支援

### HTTP 協議

| 協議 | 支援 | 說明 |
|------|------|------|
| HTTP/1.1 | ✅ | 標準 HTTP |
| HTTP/2 | ✅ | ALPN (TLS) / h2c upgrade |
| TLS 1.2+ | ✅ | OpenSSL |

### 壓縮

bmcweb 支援回應壓縮：

| 格式 | 支援 |
|------|------|
| gzip | ✅ |
| zstd | ✅ |

根據 `Accept-Encoding` header 自動選擇。

### WebSocket

| 用途 | 端點 |
|------|------|
| D-Bus 事件 | `/subscribe` |
| Serial Console | `/console0` |
| KVM | `/kvm/0` |

---

## 相關文件

- [CodeOrganization](CodeOrganization.md) - 原始碼結構
- [HTTPServer](HTTPServer.md) - HTTP 伺服器詳細說明
- [Authentication](Authentication.md) - 認證機制
- [RedfishOverview](RedfishOverview.md) - Redfish 實作

---

*最後更新：2025-12-19*
