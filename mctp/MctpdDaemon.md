# mctpd 守護程式 (mctpd Daemon)

`mctpd` 是 MCTP 控制協議的守護程式，負責端點發現、EID 分配和 D-Bus 介面服務。

---

## 概述

`mctpd` 是 CodeConstruct/mctp 專案的核心元件，提供以下功能：

| 功能 | 說明 |
|------|------|
| **MCTP 控制協議** | 實作 DSP0236 定義的控制命令 |
| **EID 分配** | 作為 bus-owner 分配 EID 給遠端端點 |
| **端點發現** | 發現和列舉 MCTP 網路中的端點 |
| **D-Bus 服務** | 透過 D-Bus 暴露端點資訊和控制介面 |
| **路由管理** | 自動設定 MCTP 路由和鄰居表 |

---

## 運作模式

mctpd 支援兩種運作模式：

### Bus-Owner 模式

```
┌─────────────────────────────────────────────────────────────────┐
│                    Bus-Owner Mode                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  mctpd 作為 bus owner：                                         │
│                                                                 │
│  • 分配 EID 給遠端端點（SetupEndpoint, AssignEndpoint）          │
│  • 管理動態 EID 池                                               │
│  • 處理橋接器的 EID 池請求                                       │
│  • 發送 Set Endpoint ID 控制命令                                 │
│                                                                 │
│  配置：mode = "bus-owner"                                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Endpoint 模式

```
┌─────────────────────────────────────────────────────────────────┐
│                    Endpoint Mode                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  mctpd 作為 endpoint：                                          │
│                                                                 │
│  • 接受外部 bus owner 的 EID 分配                                │
│  • 回應 Get Endpoint ID、Get Endpoint UUID 等查詢               │
│  • 參與端點發現流程（Prepare/Endpoint Discovery）                │
│  • 不會主動分配 EID                                              │
│                                                                 │
│  配置：mode = "endpoint"                                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 命令行選項

```bash
mctpd [OPTIONS]
```

| 選項 | 說明 |
|------|------|
| `-c FILE`, `--config FILE` | 指定配置檔案（預設：`/etc/mctpd.conf`） |
| `-v`, `--verbose` | 啟用詳細日誌輸出 |

### 使用範例

```bash
# 使用預設配置啟動
mctpd

# 使用指定配置檔案
mctpd -c /path/to/mctpd.conf

# 詳細模式
mctpd -v
```

---

## 配置檔案

mctpd 使用 TOML 格式的配置檔案。預設路徑：`/etc/mctpd.conf`

### 完整配置範例

```toml
# mctpd 運作模式
mode = "bus-owner"

# MCTP 協議設定
[mctp]
message_timeout_ms = 250

# 可選：指定 UUID（通常自動取得系統 UUID）
# uuid = "21f0f554-7f7c-4211-9ca1-6d0f000ea9e7"

# Bus-owner 模式設定
[bus-owner]
dynamic_eid_range = [8, 254]
max_pool_size = 15
```

### 配置選項說明

#### 全域設定

| 選項 | 類型 | 預設值 | 說明 |
|------|------|--------|------|
| `mode` | string | `"bus-owner"` | 運作模式：`"bus-owner"` 或 `"endpoint"` |

#### [mctp] 區段

| 選項 | 類型 | 預設值 | 說明 |
|------|------|--------|------|
| `message_timeout_ms` | integer | 250 | MCTP 訊息逾時（毫秒） |
| `uuid` | string | 系統 UUID | 端點 UUID（RFC 4122 格式） |

#### [bus-owner] 區段

| 選項 | 類型 | 預設值 | 說明 |
|------|------|--------|------|
| `dynamic_eid_range` | array | `[8, 254]` | 動態 EID 分配範圍 |
| `max_pool_size` | integer | 15 | 橋接器最大 EID 池大小 |

---

## MCTP 控制協議實作

mctpd 實作以下 MCTP 控制協議命令：

### 作為請求發送者（Bus-Owner）

| 命令 | 說明 |
|------|------|
| Set Endpoint ID | 分配 EID 給遠端端點 |
| Get Endpoint ID | 查詢端點當前 EID |
| Get Endpoint UUID | 查詢端點 UUID |
| Get Message Type Support | 查詢支援的訊息類型 |
| Allocate Endpoint IDs | 分配 EID 池給橋接器 |

### 作為回應者（兩種模式）

| 命令 | 說明 |
|------|------|
| Get Endpoint ID | 回報本機 EID |
| Get Endpoint UUID | 回報本機 UUID |
| Get MCTP Version Support | 回報支援的 MCTP 版本 |
| Get Message Type Support | 回報支援的訊息類型 |
| Get Vendor Defined Msg Support | 回報供應商定義訊息支援 |

### 僅 Endpoint 模式

| 命令 | 說明 |
|------|------|
| Set Endpoint ID | 接受 bus owner 分配的 EID |
| Prepare for Endpoint Discovery | 準備發現流程 |
| Endpoint Discovery | 執行發現流程 |

---

## D-Bus 服務

mctpd 提供完整的 D-Bus 服務：

```
服務名稱：au.com.codeconstruct.MCTP1
根物件：/au/com/codeconstruct/mctp1
```

### 物件樹結構

```
/au/com/codeconstruct/mctp1
├── interfaces/
│   ├── mctpi2c1          (au.com.codeconstruct.MCTP.Interface1)
│   │                     (au.com.codeconstruct.MCTP.BusOwner1, if bus-owner)
│   └── mctpi2c2          ...
└── networks/
    └── 1/
        └── endpoints/
            ├── 8         (xyz.openbmc_project.MCTP.Endpoint)
            │             (xyz.openbmc_project.Common.UUID)
            │             (au.com.codeconstruct.MCTP.Endpoint1)
            ├── 10        ...
            └── 11        ...
```

### 主要 D-Bus 介面

| 介面 | 說明 |
|------|------|
| `au.com.codeconstruct.MCTP1` | 頂層介面，訊息類型註冊 |
| `au.com.codeconstruct.MCTP.Interface1` | MCTP 介面屬性 |
| `au.com.codeconstruct.MCTP.BusOwner1` | Bus-owner 操作方法 |
| `au.com.codeconstruct.MCTP.Network1` | 網路層操作 |
| `xyz.openbmc_project.MCTP.Endpoint` | OpenBMC 標準端點介面 |
| `xyz.openbmc_project.Common.UUID` | 端點 UUID |
| `au.com.codeconstruct.MCTP.Endpoint1` | 端點控制方法 |
| `au.com.codeconstruct.MCTP.Bridge1` | 橋接器 EID 池資訊 |

詳細 D-Bus API 說明請參閱：

- [DBusOverview](DBusOverview.md) - D-Bus 介面總覽
- [InterfaceAPI](InterfaceAPI.md) - Interface1 / BusOwner1
- [NetworkAPI](NetworkAPI.md) - Network1
- [EndpointAPI](EndpointAPI.md) - Endpoint / Endpoint1
- [BridgeAPI](BridgeAPI.md) - Bridge1

---

## 端點發現流程

### SetupEndpoint 流程

```mermaid
sequenceDiagram
    participant Client as D-Bus Client
    participant mctpd as mctpd
    participant Device as MCTP Device

    Client->>mctpd: SetupEndpoint(hwaddr)
    
    mctpd->>Device: Get Endpoint ID
    Device-->>mctpd: Response (current EID or null)
    
    alt Device has no EID
        mctpd->>mctpd: Allocate EID from pool
        mctpd->>Device: Set Endpoint ID (new EID)
        Device-->>mctpd: Response (accepted)
    end
    
    mctpd->>Device: Get Endpoint UUID
    Device-->>mctpd: Response (UUID)
    
    mctpd->>Device: Get Message Type Support
    Device-->>mctpd: Response (types)
    
    mctpd->>mctpd: Add route & neigh entry
    mctpd->>mctpd: Create D-Bus object
    
    mctpd-->>Client: (eid, net, path, new)
```

### AssignEndpoint vs SetupEndpoint vs LearnEndpoint

| 方法 | 說明 |
|------|------|
| **SetupEndpoint** | 查詢現有 EID，若無則分配新的 |
| **AssignEndpoint** | 總是分配新 EID |
| **AssignEndpointStatic** | 指定特定 EID |
| **LearnEndpoint** | 僅查詢，不分配 EID |

---

## 連接狀態管理

mctpd 追蹤端點的連接狀態：

### 連接狀態

```
┌─────────────────────────────────────────────────────────────────┐
│                    Endpoint Connectivity                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  正常狀態                                                        │
│  • 端點回應正常                                                  │
│  • D-Bus Connectivity 屬性顯示正常                               │
│                                                                 │
│  降級狀態（degraded）                                            │
│  • 端點無回應                                                    │
│  • mctpd 啟動恢復程序                                            │
│  • 定期輪詢嘗試重新連接                                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 端點恢復

當端點無回應時，mctpd 會：

1. 標記端點為 degraded
2. 定期發送 Get Endpoint ID 嘗試恢復
3. 驗證回應的 UUID 是否匹配
4. 若恢復成功，更新連接狀態

---

## Systemd 整合

### 服務定義

```ini
# /lib/systemd/system/mctpd.service
[Unit]
Description=MCTP control protocol daemon
Wants=mctp-local.target
After=mctp-local.target

[Service]
Type=dbus
BusName=au.com.codeconstruct.MCTP1
ExecStart=/usr/sbin/mctpd

[Install]
WantedBy=mctp.target
```

### 相關 Target

```ini
# /lib/systemd/system/mctp.target
[Unit]
Description=MCTP infrastructure active
Wants=mctp-local.target

[Install]
WantedBy=multi-user.target
```

### 啟動順序

```
multi-user.target
    └── mctp.target
        ├── mctp-local.target
        │   └── mctp-local-setup.service (if exists)
        └── mctpd.service
```

---

## 日誌與除錯

### 查看日誌

```bash
# 透過 journalctl
journalctl -u mctpd -f

# 詳細模式啟動
mctpd -v
```

### 常見日誌訊息

```
# 發現新端點
mctpd[1234]: Found endpoint EID 10 on mctpi2c1

# EID 分配
mctpd[1234]: Assigned EID 10 to device at hwaddr 0x1d

# 端點 UUID
mctpd[1234]: Endpoint 10 UUID: 12345678-1234-1234-1234-123456789abc

# 連接問題
mctpd[1234]: Endpoint 10 not responding, marking degraded

# 恢復
mctpd[1234]: Endpoint 10 recovered
```

---

## 建構選項

### Meson 選項

| 選項 | 類型 | 預設值 | 說明 |
|------|------|--------|------|
| `tests` | boolean | true | 啟用測試 |
| `unsafe-recover-nil-uuid` | boolean | false | 允許 nil UUID 恢復（開發用） |
| `unsafe-writable-connectivity` | boolean | false | 允許寫入 Connectivity 屬性（開發用） |

```bash
# 禁用測試（減少相依性）
meson setup obj -Dtests=false

# 開發選項
meson setup obj -Dunsafe-recover-nil-uuid=true
```

### 相依性

| 相依 | 版本 | 說明 |
|------|------|------|
| libsystemd | ≥247 | sd-bus 和 sd-event |
| meson | ≥0.59.0 | 建構系統 |

---

## 常見問題

### mctpd 無法啟動

1. 檢查 libsystemd 版本
2. 確認配置檔案語法正確
3. 檢查 D-Bus 權限

### 無法發現端點

1. 確認 MCTP 介面已啟用（`mctp link show`）
2. 確認硬體連接正常
3. 檢查 I2C 地址是否正確

### D-Bus 連線失敗

1. 確認 mctpd 正在運行
2. 檢查 D-Bus 配置
3. 確認服務名稱正確

---

## 相關文件

- [Configuration](Configuration.md) - 配置指南
- [DBusOverview](DBusOverview.md) - D-Bus 介面總覽
- [EndpointDiscovery](EndpointDiscovery.md) - 端點發現
- [SystemdIntegration](SystemdIntegration.md) - Systemd 整合

---

[← 返回首頁](Home.md)
