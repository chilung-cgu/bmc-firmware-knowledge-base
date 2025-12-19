# Redfish Schema 參考

本文件列出 bmcweb 支援的 Redfish Schema 及其端點。

---

## 📋 目錄

1. [Schema 概述](#schema-概述)
2. [核心 Schema](#核心-schema)
3. [服務端點](#服務端點)
4. [Schema 管理](#schema-管理)

---

## Schema 概述

bmcweb 實作 DMTF Redfish Schema，透過 **D-Bus 物件轉譯**提供標準化 API。

### 通用欄位

所有 Redfish 資源包含以下通用欄位：

| 欄位 | 說明 | 範例 |
|------|------|------|
| `@odata.id` | 資源 URI | `/redfish/v1/Systems/system` |
| `@odata.type` | Schema 類型 | `#ComputerSystem.v1_20_0.ComputerSystem` |
| `Id` | 資源識別碼 | `system` |
| `Name` | 顯示名稱 | `Computer System` |

---

## 核心 Schema

### ComputerSystem

管理主機系統資源。

| 端點 | 方法 | 說明 |
|------|------|------|
| `/redfish/v1/Systems` | GET | 系統集合 |
| `/redfish/v1/Systems/{SystemId}` | GET, PATCH | 單一系統 |
| `/redfish/v1/Systems/{SystemId}/Actions/ComputerSystem.Reset` | POST | 電源控制 |

**主要屬性：**
- `PowerState` - 電源狀態 (On, Off, PoweringOn, PoweringOff)
- `Boot` - 開機設定
- `ProcessorSummary` - 處理器摘要
- `MemorySummary` - 記憶體摘要

### Chassis

管理機箱及其包含的元件。

| 端點 | 方法 | 說明 |
|------|------|------|
| `/redfish/v1/Chassis` | GET | 機箱集合 |
| `/redfish/v1/Chassis/{ChassisId}` | GET, PATCH | 單一機箱 |
| `/redfish/v1/Chassis/{ChassisId}/Power` | GET | 電源資訊 |
| `/redfish/v1/Chassis/{ChassisId}/Thermal` | GET | 散熱資訊 |
| `/redfish/v1/Chassis/{ChassisId}/Sensors` | GET | 感測器集合 |

### Manager

管理 BMC 本身。

| 端點 | 方法 | 說明 |
|------|------|------|
| `/redfish/v1/Managers` | GET | 管理器集合 |
| `/redfish/v1/Managers/{ManagerId}` | GET, PATCH | 單一管理器 |
| `/redfish/v1/Managers/{ManagerId}/EthernetInterfaces` | GET | 網路介面 |
| `/redfish/v1/Managers/{ManagerId}/NetworkProtocol` | GET, PATCH | 網路協議 |
| `/redfish/v1/Managers/{ManagerId}/LogServices` | GET | 日誌服務 |

---

## 服務端點

### AccountService

| 端點 | 說明 |
|------|------|
| `/redfish/v1/AccountService` | 帳戶服務配置 |
| `/redfish/v1/AccountService/Accounts` | 使用者帳戶集合 |
| `/redfish/v1/AccountService/Accounts/{AccountId}` | 單一帳戶 |
| `/redfish/v1/AccountService/Roles` | 角色集合 |
| `/redfish/v1/AccountService/Roles/{RoleId}` | 單一角色 |
| `/redfish/v1/AccountService/LDAP/Certificates` | LDAP 憑證 |

### SessionService

| 端點 | 說明 |
|------|------|
| `/redfish/v1/SessionService` | Session 服務配置 |
| `/redfish/v1/SessionService/Sessions` | Session 集合 |
| `/redfish/v1/SessionService/Sessions/{SessionId}` | 單一 Session |

### EventService

| 端點 | 說明 |
|------|------|
| `/redfish/v1/EventService` | 事件服務配置 |
| `/redfish/v1/EventService/Subscriptions` | 訂閱集合 |
| `/redfish/v1/EventService/Subscriptions/{SubscriptionId}` | 單一訂閱 |

### UpdateService

| 端點 | 說明 |
|------|------|
| `/redfish/v1/UpdateService` | 更新服務配置 |
| `/redfish/v1/UpdateService/FirmwareInventory` | 韌體清單 |
| `/redfish/v1/UpdateService/Actions/UpdateService.SimpleUpdate` | 執行更新 |

### TelemetryService

| 端點 | 說明 |
|------|------|
| `/redfish/v1/TelemetryService` | 遙測服務配置 |
| `/redfish/v1/TelemetryService/MetricReportDefinitions` | 報告定義 |
| `/redfish/v1/TelemetryService/MetricReports` | 實際報告 |
| `/redfish/v1/TelemetryService/Triggers` | 觸發器 |

### CertificateService

| 端點 | 說明 |
|------|------|
| `/redfish/v1/CertificateService` | 憑證服務 |
| `/redfish/v1/CertificateService/CertificateLocations` | 憑證位置 |
| `/redfish/v1/Managers/bmc/NetworkProtocol/HTTPS/Certificates` | HTTPS 憑證 |
| `/redfish/v1/Managers/bmc/Truststore/Certificates` | 信任儲存區 |

### TaskService

| 端點 | 說明 |
|------|------|
| `/redfish/v1/TaskService` | 任務服務配置 |
| `/redfish/v1/TaskService/Tasks` | 任務集合 |
| `/redfish/v1/TaskService/Tasks/{TaskId}` | 單一任務 |

---

## Schema 管理

### 查詢可用 Schema

```bash
curl -k -H "X-Auth-Token: $token" \
    https://${bmc}/redfish/v1/JsonSchemas
```

### 取得特定 Schema

```bash
curl -k -H "X-Auth-Token: $token" \
    https://${bmc}/redfish/v1/JsonSchemas/ComputerSystem
```

### 更新 Schema

bmcweb 的 Schema 由 `scripts/update_schemas.py` 管理：

```python
# 在 update_schemas.py 中新增 Schema
SCHEMAS = [
    "ComputerSystem",
    "Chassis",
    "Manager",
    # ... 其他 Schema
]
```

執行更新：
```bash
python3 scripts/update_schemas.py
```

### Schema 版本

bmcweb 支援的 Schema 版本記錄於程式碼中，例如：

```cpp
asyncResp->res.jsonValue["@odata.type"] = 
    "#ComputerSystem.v1_20_0.ComputerSystem";
```

---

## 相關文件

- [RedfishOverview](RedfishOverview.md) - Redfish 協議概述
- [RedfishAccountService](RedfishAccountService.md) - 帳戶服務
- [RedfishSystemService](RedfishSystemService.md) - 系統服務

---

*最後更新：2025-12-19*
