# D-Bus REST API

本文件說明 bmcweb 的 D-Bus REST API，提供對 OpenBMC D-Bus 物件的直接存取。

---

## 📋 目錄

1. [概述](#概述)
2. [API 端點](#api-端點)
3. [物件操作](#物件操作)
4. [使用範例](#使用範例)
5. [與 Redfish 比較](#與-redfish-比較)

---

## 概述

D-Bus REST API 提供對 OpenBMC D-Bus 匯流排的直接 HTTP 存取：

- **低階存取** - 直接操作 D-Bus 物件
- **高保真度** - 完整的 D-Bus 語義
- **開發友善** - 適合除錯和探索

> [!NOTE]
> 此 API 主要用於開發和除錯。生產環境建議使用 Redfish API。

---

## 啟用 D-Bus REST API

需要在編譯時啟用 `rest` 選項：

```bash
meson setup builddir -Drest=enabled
```

或在 local.conf：

```bash
EXTRA_OEMESON:pn-bmcweb:append = "-Drest='enabled'"
```

---

## API 端點

### 根端點

```
/xyz/
```

### 物件路徑映射

D-Bus 物件路徑直接映射到 URL：

| D-Bus 路徑 | REST URL |
|------------|----------|
| `/` | `/xyz/` |
| `/xyz/openbmc_project` | `/xyz/openbmc_project` |
| `/xyz/openbmc_project/sensors/temperature/CPU0` | `/xyz/openbmc_project/sensors/temperature/CPU0` |

---

## 物件操作

### GET - 讀取物件

```bash
# 讀取物件及其所有介面和屬性
curl -k -H "X-Auth-Token: $token" \
    https://${bmc}/xyz/openbmc_project/state/host0
```

**回應範例：**
```json
{
    "data": {
        "CurrentHostState": "xyz.openbmc_project.State.Host.HostState.Running",
        "RequestedHostTransition": "xyz.openbmc_project.State.Host.Transition.On"
    },
    "message": "200 OK",
    "status": "ok"
}
```

### GET - 列出子物件

```bash
# 加上 list 參數列出子物件
curl -k -H "X-Auth-Token: $token" \
    "https://${bmc}/xyz/openbmc_project/sensors/list"
```

### GET - 列舉所有物件

```bash
# enumerate 列出所有子物件及其屬性
curl -k -H "X-Auth-Token: $token" \
    "https://${bmc}/xyz/openbmc_project/sensors/enumerate"
```

### PUT - 設定屬性

```bash
# 設定屬性值
curl -k -H "X-Auth-Token: $token" \
    -H "Content-Type: application/json" \
    -X PUT https://${bmc}/xyz/openbmc_project/state/host0/attr/RequestedHostTransition \
    -d '{"data": "xyz.openbmc_project.State.Host.Transition.On"}'
```

### POST - 呼叫方法

```bash
# 呼叫 D-Bus 方法
curl -k -H "X-Auth-Token: $token" \
    -H "Content-Type: application/json" \
    -X POST https://${bmc}/xyz/openbmc_project/software/bmc/action/Activate \
    -d '{"data": []}'
```

### DELETE - 刪除物件

```bash
# 刪除物件（如果支援）
curl -k -H "X-Auth-Token: $token" \
    -X DELETE https://${bmc}/xyz/openbmc_project/logging/entry/1
```

---

## 使用範例

### 查詢主機電源狀態

```bash
curl -k -H "X-Auth-Token: $token" \
    https://${bmc}/xyz/openbmc_project/state/host0 \
    | jq '.data.CurrentHostState'
```

### 電源操作

```bash
# 開機
curl -k -H "X-Auth-Token: $token" \
    -H "Content-Type: application/json" \
    -X PUT https://${bmc}/xyz/openbmc_project/state/host0/attr/RequestedHostTransition \
    -d '{"data": "xyz.openbmc_project.State.Host.Transition.On"}'

# 關機
curl -k -H "X-Auth-Token: $token" \
    -H "Content-Type: application/json" \
    -X PUT https://${bmc}/xyz/openbmc_project/state/host0/attr/RequestedHostTransition \
    -d '{"data": "xyz.openbmc_project.State.Host.Transition.Off"}'
```

### 讀取感測器

```bash
# 列出所有感測器
curl -k -H "X-Auth-Token: $token" \
    "https://${bmc}/xyz/openbmc_project/sensors/enumerate"

# 讀取特定感測器
curl -k -H "X-Auth-Token: $token" \
    https://${bmc}/xyz/openbmc_project/sensors/temperature/CPU0_Temp
```

### 登入 (取得 Token)

```bash
# 傳統登入方式
curl -k -c cjar -b cjar \
    -H "Content-Type: application/json" \
    -X POST https://${bmc}/login \
    -d '{"data": ["root", "0penBmc"]}'
```

---

## 回應格式

### 成功回應

```json
{
    "data": { ... },
    "message": "200 OK",
    "status": "ok"
}
```

### 錯誤回應

```json
{
    "data": { "description": "Error message" },
    "message": "404 Not Found",
    "status": "error"
}
```

---

## 與 Redfish 比較

| 特性 | D-Bus REST | Redfish |
|------|------------|---------|
| 標準化 | OpenBMC 專屬 | DMTF 標準 |
| 抽象層級 | 低階 (D-Bus 原生) | 高階 (硬體抽象) |
| 互通性 | 僅 OpenBMC | 跨廠商 |
| 適用場景 | 開發/除錯 | 生產環境 |
| Schema | 無正式 Schema | 完整 JSON Schema |

### 何時使用 D-Bus REST

- 開發和除錯
- 存取 Redfish 未暴露的功能
- 探索 D-Bus 物件結構
- 快速原型開發

### 何時使用 Redfish

- 生產環境整合
- 跨 BMC 廠商相容性
- 標準化管理
- 需要正式 API 契約

---

## 安全考量

> [!WARNING]
> D-Bus REST API 提供低階存取，可能暴露敏感操作。生產環境應謹慎使用。

- 需要適當認證
- 權限對應 D-Bus 策略
- 建議在開發環境使用

---

## 相關文件

- [WebSocketDBus](WebSocketDBus.md) - D-Bus 事件訂閱
- [RedfishOverview](RedfishOverview.md) - Redfish API
- [Architecture](Architecture.md) - 架構概述

---

*最後更新：2025-12-19*
