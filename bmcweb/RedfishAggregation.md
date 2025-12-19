# Redfish 聚合功能

本文件說明 bmcweb 的 Redfish 聚合功能，用於整合多個 BMC 的資源。

---

## 📋 目錄

1. [功能概述](#功能概述)
2. [配置方法](#配置方法)
3. [運作機制](#運作機制)
4. [使用範例](#使用範例)
5. [限制與注意事項](#限制與注意事項)

---

## 功能概述

Redfish 聚合允許主 BMC 整合來自衛星 BMC 的資源：

```
┌─────────────────────────────────────────────────────────────┐
│                     Redfish Client                           │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                   主 BMC (Aggregator)                        │
│  ┌───────────────────────────────────────────────────────┐  │
│  │                    bmcweb                              │  │
│  │  本地資源 + 聚合的衛星 BMC 資源                         │  │
│  └───────────────────────────────────────────────────────┘  │
└────────────────────┬───────────────┬────────────────────────┘
                     │               │
        ┌────────────┘               └────────────┐
        ▼                                         ▼
┌───────────────────┐                   ┌───────────────────┐
│  衛星 BMC (sat0)   │                   │  衛星 BMC (sat1)   │
│  /redfish/v1/...  │                   │  /redfish/v1/...  │
└───────────────────┘                   └───────────────────┘
```

聚合後的資源會加上前綴以區分來源：

| 原始 URI | 聚合後 URI |
|----------|------------|
| `/redfish/v1/Systems/system` (本地) | `/redfish/v1/Systems/system` |
| `/redfish/v1/Systems/system` (sat0) | `/redfish/v1/Systems/sat0_system` |
| `/redfish/v1/Systems/system` (sat1) | `/redfish/v1/Systems/sat1_system` |

---

## 配置方法

### 啟用聚合功能

在 local.conf 中加入：

```bash
EXTRA_OEMESON:pn-bmcweb:append = "-Dredfish-aggregation='enabled'"
```

### 配置衛星 BMC

透過 Entity-Manager 定義衛星 BMC：

```json
{
    "Exposes": [
        {
            "Hostname": "192.168.1.10",
            "Port": "80",
            "Name": "sat0",
            "Type": "SatelliteController",
            "AuthType": "None"
        },
        {
            "Hostname": "192.168.1.11",
            "Port": "80",
            "Name": "sat1",
            "Type": "SatelliteController",
            "AuthType": "None"
        }
    ],
    "Name": "Satellite BMC Configuration",
    "Type": "Board"
}
```

### 配置參數

| 參數 | 說明 | 必要 |
|------|------|------|
| `Hostname` | 衛星 BMC IP 或主機名 | ✅ |
| `Port` | HTTP 端口 | ✅ |
| `Name` | 衛星識別名稱 (用於 URI 前綴) | ✅ |
| `Type` | 必須為 `SatelliteController` | ✅ |
| `AuthType` | 認證類型 (目前僅支援 `None`) | ✅ |

---

## 運作機制

### 集合聚合

當查詢頂層集合時，主 BMC 會：

1. 處理本地請求
2. 同時轉發 GET 請求到所有衛星
3. 合併 `Members` 陣列
4. 在衛星資源 URI 添加前綴

**範例：**
```bash
curl -k -H "X-Auth-Token: $token" https://${bmc}/redfish/v1/Systems
```

**回應：**
```json
{
    "@odata.id": "/redfish/v1/Systems",
    "Members": [
        {"@odata.id": "/redfish/v1/Systems/system"},
        {"@odata.id": "/redfish/v1/Systems/sat0_system"},
        {"@odata.id": "/redfish/v1/Systems/sat1_system"}
    ],
    "Members@odata.count": 3
}
```

### 單一資源存取

當存取帶有衛星前綴的資源時：

1. 識別前綴判斷目標衛星
2. 移除前綴後轉發請求
3. 在回應中添加前綴
4. 返回修改後的回應

**範例：**
```bash
curl -k -H "X-Auth-Token: $token" https://${bmc}/redfish/v1/Systems/sat0_system
```

**處理流程：**
```
1. 識別 sat0_ 前綴 → 目標為 sat0
2. 移除前綴 → /redfish/v1/Systems/system
3. 轉發到 sat0: GET http://192.168.1.10/redfish/v1/Systems/system
4. 在回應 URI 添加 sat0_ 前綴
5. 返回修改後的回應
```

---

## 使用範例

### 查詢聚合的系統集合

```bash
curl -k -H "X-Auth-Token: $token" https://${bmc}/redfish/v1/Systems
```

### 查詢衛星系統

```bash
curl -k -H "X-Auth-Token: $token" https://${bmc}/redfish/v1/Systems/sat0_system
```

### 控制衛星系統電源

```bash
curl -k -H "X-Auth-Token: $token" \
    -H "Content-Type: application/json" \
    -X POST https://${bmc}/redfish/v1/Systems/sat0_system/Actions/ComputerSystem.Reset \
    -d '{"ResetType": "On"}'
```

### 查詢衛星 Chassis

```bash
curl -k -H "X-Auth-Token: $token" https://${bmc}/redfish/v1/Chassis/sat0_chassis
```

---

## 支援的資源

聚合支援所有頂層集合下的資源：

| 集合 | 支援 |
|------|------|
| `/redfish/v1/Systems` | ✅ |
| `/redfish/v1/Chassis` | ✅ |
| `/redfish/v1/Managers` | ✅ |
| `/redfish/v1/TaskService/Tasks` | ✅ |
| `/redfish/v1/UpdateService/FirmwareInventory` | ✅ |

### 不聚合的資源

| 資源 | 原因 |
|------|------|
| `/redfish/v1/JsonSchemas` | Schema 版本可能不同 |
| `/redfish/v1/$metadata` | 假設與主 BMC 相容 |

---

## 限制與注意事項

> [!WARNING]
> **目前限制：**
> - 僅支援 HTTP (非 HTTPS) 連接到衛星
> - 衛星 BMC 不支援認證
> - 每個衛星必須有唯一的 `Name`

> [!IMPORTANT]
> **錯誤處理：**
> - 衛星無回應會重試 (依照重試策略)
> - 重試失敗返回 502 錯誤
> - 部分衛星失敗不影響其他資源

### 安全性考量

由於目前僅支援無認證 HTTP：

- 建議將衛星 BMC 放在隔離網路
- 使用防火牆限制存取
- 考慮使用 VPN 加密流量

---

## 疑難排解

### 衛星未出現在集合中

1. 檢查 Entity-Manager 配置
2. 確認衛星 BMC 可達
3. 檢查 D-Bus 上的 SatelliteController 物件

```bash
busctl tree xyz.openbmc_project.EntityManager | grep -i satellite
```

### 502 錯誤

表示與衛星通訊失敗：

1. 確認網路連通性
2. 檢查衛星 BMC 服務狀態
3. 查看 bmcweb 日誌

```bash
journalctl -u bmcweb -f
```

---

## 相關文件

- [Architecture](Architecture.md) - 架構概述
- [RedfishOverview](RedfishOverview.md) - Redfish 概述
- [Configuration](Configuration.md) - 配置選項

---

*最後更新：2025-12-19*
