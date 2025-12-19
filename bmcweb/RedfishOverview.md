# Redfish 協議概述

本文件介紹 Redfish 標準以及 bmcweb 對 Redfish 的實作方式。

---

## 📋 目錄

1. [什麼是 Redfish](#什麼是-redfish)
2. [Redfish 資料模型](#redfish-資料模型)
3. [bmcweb Redfish 實作](#bmcweb-redfish-實作)
4. [D-Bus 到 Redfish 轉譯](#d-bus-到-redfish-轉譯)
5. [Redfish 合規性](#redfish-合規性)

---

## 什麼是 Redfish

**Redfish** 是由 DMTF (Distributed Management Task Force) 定義的 RESTful API 標準，用於資料中心硬體管理。

### 核心特色

| 特色 | 說明 |
|------|------|
| **RESTful** | 標準 HTTP 方法 (GET, POST, PATCH, DELETE) |
| **JSON** | 使用 JSON 格式進行資料交換 |
| **OData** | 遵循 OData 協議規範 |
| **安全性** | 支援 HTTPS、認證、授權 |
| **可擴展** | 支援 OEM 擴展 |

### DMTF 規格文件

| 文件編號 | 名稱 | 說明 |
|----------|------|------|
| DSP0266 | Redfish Specification | 核心協議規格 |
| DSP0268 | Redfish Data Model | 資料模型規格 |
| DSP8010 | Redfish Schema Bundle | JSON Schema 定義 |
| DSP2046 | Redfish Resource and Schema Guide | 資源與 Schema 指南 |

---

## Redfish 資料模型

### 資源樹結構

Redfish 使用階層式資源樹：

```
/redfish/v1/                          ← Service Root
├── /redfish/v1/Systems/              ← 系統集合
│   └── /redfish/v1/Systems/system/   ← 單一系統
│       ├── Processors/
│       ├── Memory/
│       ├── Storage/
│       └── LogServices/
├── /redfish/v1/Chassis/              ← 機箱集合
│   └── /redfish/v1/Chassis/chassis/
│       ├── Power/
│       ├── Thermal/
│       └── Sensors/
├── /redfish/v1/Managers/             ← 管理器集合
│   └── /redfish/v1/Managers/bmc/
│       ├── EthernetInterfaces/
│       ├── NetworkProtocol/
│       └── LogServices/
├── /redfish/v1/AccountService/       ← 帳戶服務
├── /redfish/v1/SessionService/       ← Session 服務
├── /redfish/v1/EventService/         ← 事件服務
├── /redfish/v1/UpdateService/        ← 更新服務
└── /redfish/v1/TelemetryService/     ← 遙測服務
```

### 通用屬性

所有 Redfish 資源都包含以下通用屬性：

```json
{
    "@odata.id": "/redfish/v1/Systems/system",
    "@odata.type": "#ComputerSystem.v1_20_0.ComputerSystem",
    "Id": "system",
    "Name": "system"
}
```

| 屬性 | 說明 |
|------|------|
| `@odata.id` | 資源的唯一 URI |
| `@odata.type` | 資源的 Schema 類型 |
| `Id` | 資源識別碼 |
| `Name` | 人類可讀名稱 |

### 集合格式

```json
{
    "@odata.id": "/redfish/v1/Systems",
    "@odata.type": "#ComputerSystemCollection.ComputerSystemCollection",
    "Members": [
        {"@odata.id": "/redfish/v1/Systems/system"}
    ],
    "Members@odata.count": 1,
    "Name": "Computer System Collection"
}
```

---

## bmcweb Redfish 實作

### 處理架構

bmcweb 的 Redfish 實作位於 `redfish-core/` 目錄：

```
redfish-core/
├── include/
│   ├── node.hpp              ← 基礎 Node 類別
│   ├── privileges.hpp        ← 權限定義
│   └── utils/                ← 工具函數
├── lib/
│   ├── systems.hpp           ← /Systems 處理
│   ├── chassis.hpp           ← /Chassis 處理
│   ├── managers.hpp          ← /Managers 處理
│   ├── account_service.hpp   ← /AccountService 處理
│   └── ...
└── src/
    └── redfish.cpp           ← 路由註冊
```

### 路由註冊

```cpp
// 典型的 Redfish 路由註冊
BMCWEB_ROUTE(app, "/redfish/v1/Systems/system/")
    .privileges(redfish::privileges::getComputerSystem)
    .methods(boost::beast::http::verb::get)(
        [&app](const crow::Request& req,
               const std::shared_ptr<bmcweb::AsyncResp>& asyncResp) {
            handleComputerSystemGet(app, req, asyncResp);
        });
```

### AsyncResp 模式

bmcweb 使用 `AsyncResp` 處理非同步回應：

```cpp
void handleSystemGet(const crow::Request& req,
                     const std::shared_ptr<bmcweb::AsyncResp>& asyncResp) {
    // 設定基本屬性
    asyncResp->res.jsonValue["@odata.id"] = "/redfish/v1/Systems/system";
    asyncResp->res.jsonValue["@odata.type"] = 
        "#ComputerSystem.v1_20_0.ComputerSystem";
    asyncResp->res.jsonValue["Id"] = "system";
    
    // 非同步查詢 D-Bus 屬性
    sdbusplus::asio::getProperty<std::string>(
        *crow::connections::systemBus,
        "xyz.openbmc_project.State.Host",
        "/xyz/openbmc_project/state/host0",
        "xyz.openbmc_project.State.Host",
        "CurrentHostState",
        [asyncResp](const boost::system::error_code& ec,
                    const std::string& state) {
            if (!ec) {
                asyncResp->res.jsonValue["PowerState"] = 
                    translatePowerState(state);
            }
        });
}
```

---

## D-Bus 到 Redfish 轉譯

bmcweb 作為 **D-Bus 到 Redfish 的轉譯器**，將 D-Bus 物件轉換為 Redfish 資源：

### 轉譯對照

| Redfish 路徑 | D-Bus 服務 | D-Bus 路徑 |
|--------------|------------|------------|
| `/redfish/v1/Systems/system` | `xyz.openbmc_project.State.Host` | `/xyz/openbmc_project/state/host0` |
| `/redfish/v1/Chassis/chassis/Sensors` | `xyz.openbmc_project.Sensor.*` | `/xyz/openbmc_project/sensors/*` |
| `/redfish/v1/Managers/bmc` | `xyz.openbmc_project.State.BMC` | `/xyz/openbmc_project/state/bmc0` |

### 屬性轉換範例

```
D-Bus Property                        Redfish Property
─────────────                        ────────────────
xyz.openbmc_project.State.Host       PowerState
  CurrentHostState = "Running"    →    = "On"
  CurrentHostState = "Off"        →    = "Off"

xyz.openbmc_project.Sensor.Value     Reading
  Value = 45.5                    →    = 45.5
  Unit = "DegreesC"               →  ReadingUnits = "Cel"
```

### 介面探索

bmcweb 使用 D-Bus Introspection 動態發現可用資源：

```cpp
// 列舉實作特定介面的物件
crow::connections::systemBus->async_method_call(
    callback,
    "xyz.openbmc_project.ObjectMapper",
    "/xyz/openbmc_project/object_mapper",
    "xyz.openbmc_project.ObjectMapper",
    "GetSubTree",
    "/xyz/openbmc_project/sensors", 0,
    std::array<const char*, 1>{"xyz.openbmc_project.Sensor.Value"});
```

---

## Redfish 合規性

### Redfish Service Validator

bmcweb 必須通過 [Redfish Service Validator](https://github.com/DMTF/Redfish-Service-Validator) 測試：

```bash
# 安裝 validator
pip3 install redfish_service_validator

# 執行驗證
rf_service_validator \
    --auth Session \
    -i https://bmc-ip:443 \
    -u root -p 0penBmc
```

### 合規性要求

| 要求 | 說明 |
|------|------|
| **無錯誤** | Validator 不得報告 ERROR |
| **無警告** | Validator 不應報告 WARNING |
| **Schema 符合** | 所有屬性符合 Redfish Schema |

### 支援的 Schema 版本

bmcweb 支援的 Schema 版本列表記錄於 `scripts/update_schemas.py`。

更新 Schema：
```bash
cd bmcweb
python3 scripts/update_schemas.py
```

---

## 相關文件

- [RedfishSchema](RedfishSchema.md) - 支援的 Schema 清單
- [RedfishAccountService](RedfishAccountService.md) - 帳戶服務
- [RedfishSystemService](RedfishSystemService.md) - 系統服務
- [Authentication](Authentication.md) - 認證機制

---

*最後更新：2025-12-19*
