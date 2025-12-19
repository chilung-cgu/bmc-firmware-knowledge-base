# 原始碼結構

本文件說明 bmcweb 的原始碼組織結構。

---

## 📋 目錄

1. [目錄結構](#目錄結構)
2. [核心元件](#核心元件)
3. [Redfish 實作](#redfish-實作)
4. [擴展方法](#擴展方法)

---

## 目錄結構

```
bmcweb/
├── http/                      # HTTP 伺服器核心
│   ├── http_connection.hpp    # HTTP 連線處理
│   ├── http_request.hpp       # 請求封裝
│   ├── http_response.hpp      # 回應封裝
│   ├── http_server.hpp        # 伺服器核心
│   ├── routing.hpp            # 路由系統
│   ├── websocket.hpp          # WebSocket 處理
│   └── ...
├── include/                   # 通用標頭檔
│   ├── async_resp.hpp         # 非同步回應
│   ├── dbus_utility.hpp       # D-Bus 工具
│   ├── persistent_data.hpp    # 持久化資料
│   ├── sessions.hpp           # Session 管理
│   └── ...
├── redfish-core/              # Redfish 實作
│   ├── include/
│   │   ├── node.hpp           # 基礎節點類別
│   │   ├── privileges.hpp     # 權限定義
│   │   └── utils/             # 工具函數
│   ├── lib/                   # Redfish 資源處理
│   │   ├── account_service.hpp
│   │   ├── chassis.hpp
│   │   ├── managers.hpp
│   │   ├── systems.hpp
│   │   └── ...
│   └── src/
│       └── redfish.cpp        # 路由註冊
├── scripts/                   # 工具腳本
│   ├── update_schemas.py      # Schema 更新
│   └── ...
├── src/                       # 主程式
│   └── webserver_main.cpp     # 程式入口
├── subprojects/               # 依賴子專案
├── meson.build                # 建置定義
└── meson_options.txt          # 編譯選項
```

---

## 核心元件

### http/ 目錄

HTTP 伺服器核心實作：

| 檔案 | 說明 |
|------|------|
| `http_server.hpp` | HTTP 伺服器類別 |
| `http_connection.hpp` | 連線管理 |
| `http_request.hpp` | Request 封裝 |
| `http_response.hpp` | Response 封裝 |
| `routing.hpp` | 路由引擎 |
| `websocket.hpp` | WebSocket 支援 |

### include/ 目錄

通用功能：

| 檔案 | 說明 |
|------|------|
| `async_resp.hpp` | 非同步回應類別 |
| `dbus_utility.hpp` | D-Bus 工具函數 |
| `sessions.hpp` | Session 資料結構 |
| `persistent_data.hpp` | 持久化儲存 |
| `json_formatters.hpp` | JSON 格式化 |

---

## Redfish 實作

### redfish-core/lib/ 結構

每個 Redfish 資源對應一個檔案：

| 檔案 | 資源 |
|------|------|
| `service_root.hpp` | `/redfish/v1/` |
| `systems.hpp` | `/redfish/v1/Systems` |
| `chassis.hpp` | `/redfish/v1/Chassis` |
| `managers.hpp` | `/redfish/v1/Managers` |
| `account_service.hpp` | `/redfish/v1/AccountService` |
| `session_service.hpp` | `/redfish/v1/SessionService` |
| `event_service.hpp` | `/redfish/v1/EventService` |
| `update_service.hpp` | `/redfish/v1/UpdateService` |
| `telemetry_service.hpp` | `/redfish/v1/TelemetryService` |
| `sensors.hpp` | Sensor 相關 |
| `power.hpp` | Power 相關 |
| `thermal.hpp` | Thermal 相關 |
| `log_services.hpp` | LogServices 相關 |

### 路由註冊模式

```cpp
// redfish-core/src/redfish.cpp
void requestRoutes(App& app) {
    // 註冊各個資源的路由
    requestRoutesServiceRoot(app);
    requestRoutesSystemsCollection(app);
    requestRoutesChassisCollection(app);
    requestRoutesManagerCollection(app);
    // ...
}
```

### 典型的 Handler 結構

```cpp
// redfish-core/lib/systems.hpp

inline void requestRoutesSystemsCollection(App& app) {
    BMCWEB_ROUTE(app, "/redfish/v1/Systems/")
        .privileges(redfish::privileges::getComputerSystemCollection)
        .methods(boost::beast::http::verb::get)(
            [&app](const crow::Request& req,
                   const std::shared_ptr<bmcweb::AsyncResp>& asyncResp) {
                handleSystemsCollection(app, req, asyncResp);
            });
}

inline void handleSystemsCollection(
    App& app,
    const crow::Request& req,
    const std::shared_ptr<bmcweb::AsyncResp>& asyncResp) {
    
    asyncResp->res.jsonValue["@odata.type"] = 
        "#ComputerSystemCollection.ComputerSystemCollection";
    asyncResp->res.jsonValue["@odata.id"] = "/redfish/v1/Systems";
    asyncResp->res.jsonValue["Name"] = "Computer System Collection";
    
    // 填充 Members 陣列...
}
```

---

## 擴展方法

### 新增 Redfish 資源

1. **建立 Handler 檔案**
```cpp
// redfish-core/lib/my_resource.hpp
inline void requestRoutesMyResource(App& app) {
    BMCWEB_ROUTE(app, "/redfish/v1/MyResource/")
        .privileges(redfish::privileges::getMyResource)
        .methods(boost::beast::http::verb::get)(handleMyResourceGet);
}
```

2. **定義權限**
```cpp
// redfish-core/include/privileges.hpp
inline const std::array<Privileges, 1> getMyResource = {
    Privileges({"Login"})
};
```

3. **註冊路由**
```cpp
// redfish-core/src/redfish.cpp
void requestRoutes(App& app) {
    // ...
    requestRoutesMyResource(app);
}
```

### 新增 WebSocket 端點

```cpp
// include/my_websocket.hpp
void requestRoutesMyWebSocket(App& app) {
    BMCWEB_ROUTE(app, "/myws")
        .privileges(redfish::privileges::login)
        .websocket()
        .onopen([](crow::websocket::Connection& conn) {
            // 連線開啟
        })
        .onmessage([](crow::websocket::Connection& conn,
                      const std::string& data,
                      bool isBinary) {
            // 處理訊息
        })
        .onclose([](crow::websocket::Connection& conn,
                    const std::string& reason) {
            // 連線關閉
        });
}
```

---

## 重要類別

### crow::Request

```cpp
class Request {
public:
    std::string_view url();
    std::string_view method();
    std::string getHeaderValue(std::string_view key);
    std::string body;
    // ...
};
```

### crow::Response

```cpp
class Response {
public:
    void result(http::status s);
    void addHeader(std::string_view key, std::string_view value);
    nlohmann::json jsonValue;
    std::string body();
    // ...
};
```

### bmcweb::AsyncResp

```cpp
class AsyncResp {
public:
    crow::Response res;
    
    AsyncResp() = default;
    ~AsyncResp() {
        // 解構時自動發送回應
    }
};
```

---

## 相關文件

- [Architecture](Architecture.md) - 架構概述
- [Configuration](Configuration.md) - 編譯配置
- [RedfishOverview](RedfishOverview.md) - Redfish 概述

---

*最後更新：2025-12-19*
