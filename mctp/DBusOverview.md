# D-Bus 介面總覽 (D-Bus Overview)

本文提供 mctpd D-Bus 服務的完整總覽，包含物件樹結構、介面列表和基本使用方式。

---

## 服務資訊

| 項目           | 值                            |
| -------------- | ----------------------------- |
| **服務名稱**   | `au.com.codeconstruct.MCTP1`  |
| **根物件路徑** | `/au/com/codeconstruct/mctp1` |
| **類型**       | D-Bus 服務（Type=dbus）       |

---

## 物件樹結構

```
Service: au.com.codeconstruct.MCTP1
│
└── /au/com/codeconstruct/mctp1
    │
    │   介面：
    │   • org.freedesktop.DBus.ObjectManager
    │   • au.com.codeconstruct.MCTP1
    │
    ├── interfaces/
    │   │
    │   ├── mctpi2c1
    │   │   介面：
    │   │   • au.com.codeconstruct.MCTP.Interface1
    │   │   • au.com.codeconstruct.MCTP.BusOwner1 (if bus-owner)
    │   │
    │   └── mctpi2c2
    │       介面：
    │       • au.com.codeconstruct.MCTP.Interface1
    │       • au.com.codeconstruct.MCTP.BusOwner1 (if bus-owner)
    │
    └── networks/
        │
        └── 1/                    (網路 ID)
            │
            │   介面：
            │   • au.com.codeconstruct.MCTP.Network1
            │
            └── endpoints/
                │
                ├── 8             (本地 EID)
                │   介面：
                │   • xyz.openbmc_project.MCTP.Endpoint
                │   • xyz.openbmc_project.Common.UUID
                │   • au.com.codeconstruct.MCTP.Endpoint1
                │
                ├── 10            (遠端 EID)
                │   介面：
                │   • xyz.openbmc_project.MCTP.Endpoint
                │   • xyz.openbmc_project.Common.UUID
                │   • au.com.codeconstruct.MCTP.Endpoint1
                │
                └── 12            (橋接器 EID)
                    介面：
                    • xyz.openbmc_project.MCTP.Endpoint
                    • xyz.openbmc_project.Common.UUID
                    • au.com.codeconstruct.MCTP.Endpoint1
                    • au.com.codeconstruct.MCTP.Bridge1
```

---

## 介面摘要

### 頂層介面

| 介面                                 | 物件                          | 說明         |
| ------------------------------------ | ----------------------------- | ------------ |
| `au.com.codeconstruct.MCTP1`         | `/au/com/codeconstruct/mctp1` | 頂層服務介面 |
| `org.freedesktop.DBus.ObjectManager` | `/au/com/codeconstruct/mctp1` | 物件枚舉     |

### 介面物件介面

| 介面                                   | 物件                    | 說明           |
| -------------------------------------- | ----------------------- | -------------- |
| `au.com.codeconstruct.MCTP.Interface1` | `.../interfaces/<name>` | 介面屬性       |
| `au.com.codeconstruct.MCTP.BusOwner1`  | `.../interfaces/<name>` | Bus-owner 操作 |

### 網路物件介面

| 介面                                 | 物件                 | 說明     |
| ------------------------------------ | -------------------- | -------- |
| `au.com.codeconstruct.MCTP.Network1` | `.../networks/<net>` | 網路操作 |

### 端點物件介面

| 介面                                  | 物件                  | 說明             |
| ------------------------------------- | --------------------- | ---------------- |
| `xyz.openbmc_project.MCTP.Endpoint`   | `.../endpoints/<eid>` | OpenBMC 端點介面 |
| `xyz.openbmc_project.Common.UUID`     | `.../endpoints/<eid>` | 端點 UUID        |
| `au.com.codeconstruct.MCTP.Endpoint1` | `.../endpoints/<eid>` | 端點控制         |
| `au.com.codeconstruct.MCTP.Bridge1`   | `.../endpoints/<eid>` | 橋接器 EID 池    |

---

## 介面詳細文件

| 文件                            | 涵蓋介面                              |
| ------------------------------- | ------------------------------------- |
| [InterfaceAPI](InterfaceAPI.md) | Interface1、BusOwner1                 |
| [NetworkAPI](NetworkAPI.md)     | Network1                              |
| [EndpointAPI](EndpointAPI.md)   | MCTP.Endpoint、Common.UUID、Endpoint1 |
| [BridgeAPI](BridgeAPI.md)       | Bridge1                               |

---

## 使用 busctl 查詢

### 列出服務

```bash
$ busctl list | grep MCTP
au.com.codeconstruct.MCTP1       1234 mctpd         - -
```

### 列出物件樹

```bash
$ busctl tree au.com.codeconstruct.MCTP1
└─/au
  └─/au/com
    └─/au/com/codeconstruct
      └─/au/com/codeconstruct/mctp1
        ├─/au/com/codeconstruct/mctp1/interfaces
        │ └─/au/com/codeconstruct/mctp1/interfaces/mctpi2c1
        └─/au/com/codeconstruct/mctp1/networks
          └─/au/com/codeconstruct/mctp1/networks/1
            └─/au/com/codeconstruct/mctp1/networks/1/endpoints
              ├─/au/com/codeconstruct/mctp1/networks/1/endpoints/8
              └─/au/com/codeconstruct/mctp1/networks/1/endpoints/10
```

### 內省介面

```bash
# 頂層物件
$ busctl introspect au.com.codeconstruct.MCTP1 /au/com/codeconstruct/mctp1
NAME                                TYPE      SIGNATURE  RESULT/VALUE  FLAGS
au.com.codeconstruct.MCTP1          interface -          -             -
.RegisterTypeSupport                method    yau        -             -
org.freedesktop.DBus.ObjectManager  interface -          -             -
.GetManagedObjects                  method    -          a{oa{sa{sv}}} -
.InterfacesAdded                    signal    oa{sa{sv}} -             -
.InterfacesRemoved                  signal    oas        -             -
```

```bash
# 介面物件（bus-owner）
$ busctl introspect au.com.codeconstruct.MCTP1 \
    /au/com/codeconstruct/mctp1/interfaces/mctpi2c1
NAME                                 TYPE      SIGNATURE RESULT/VALUE FLAGS
au.com.codeconstruct.MCTP.BusOwner1  interface -         -            -
.AssignEndpoint                      method    ay        yisb         -
.AssignEndpointStatic                method    ayy       yisb         -
.LearnEndpoint                       method    ay        yisb         -
.SetupEndpoint                       method    ay        yisb         -
au.com.codeconstruct.MCTP.Interface1 interface -         -            -
.NetworkId                           property  u         1            emits-change
.Role                                property  s         "BusOwner"   emits-change writable
```

```bash
# 端點物件
$ busctl introspect au.com.codeconstruct.MCTP1 \
    /au/com/codeconstruct/mctp1/networks/1/endpoints/10
NAME                                   TYPE      SIGNATURE RESULT/VALUE FLAGS
au.com.codeconstruct.MCTP.Endpoint1    interface -         -            -
.Remove                                method    -         -            -
.Recover                               method    -         -            -
.SetMTU                                method    u         -            -
.Connectivity                          property  s         "Available"  emits-change
xyz.openbmc_project.Common.UUID        interface -         -            -
.UUID                                  property  s         "..."        const
xyz.openbmc_project.MCTP.Endpoint      interface -         -            -
.EID                                   property  y         10           const
.NetworkId                             property  u         1            const
.SupportedMessageTypes                 property  ay        2 0 1        const
```

---

## 使用 ObjectManager

### 取得所有受管物件

```bash
$ busctl call au.com.codeconstruct.MCTP1 /au/com/codeconstruct/mctp1 \
    org.freedesktop.DBus.ObjectManager GetManagedObjects
```

### 監聽新增/移除訊號

```bash
# 監聽端點新增/移除
$ busctl monitor au.com.codeconstruct.MCTP1
```

**InterfacesAdded 訊號**：當新端點被發現時發出

```
signal oa{sa{sv}} {
    object_path: "/au/com/codeconstruct/mctp1/networks/1/endpoints/10"
    interfaces: {
        "xyz.openbmc_project.MCTP.Endpoint": {...},
        "xyz.openbmc_project.Common.UUID": {...},
        ...
    }
}
```

**InterfacesRemoved 訊號**：當端點被移除時發出

```
signal oas {
    object_path: "/au/com/codeconstruct/mctp1/networks/1/endpoints/10"
    interfaces: ["xyz.openbmc_project.MCTP.Endpoint", ...]
}
```

---

## 常用操作範例

### 發現新端點

```bash
# 使用 SetupEndpoint（查詢現有 EID 或分配新的）
busctl call au.com.codeconstruct.MCTP1 \
    /au/com/codeconstruct/mctp1/interfaces/mctpi2c1 \
    au.com.codeconstruct.MCTP.BusOwner1 \
    SetupEndpoint ay 1 0x1d
```

**回應格式**：`yisb`

- `y`: EID（byte）
- `i`: 網路 ID（integer）
- `s`: D-Bus 物件路徑（string）
- `b`: 是否為新分配（boolean）

### 讀取端點 EID

```bash
busctl get-property au.com.codeconstruct.MCTP1 \
    /au/com/codeconstruct/mctp1/networks/1/endpoints/10 \
    xyz.openbmc_project.MCTP.Endpoint EID
```

### 讀取端點 UUID

```bash
busctl get-property au.com.codeconstruct.MCTP1 \
    /au/com/codeconstruct/mctp1/networks/1/endpoints/10 \
    xyz.openbmc_project.Common.UUID UUID
```

### 設定 MTU

```bash
busctl call au.com.codeconstruct.MCTP1 \
    /au/com/codeconstruct/mctp1/networks/1/endpoints/10 \
    au.com.codeconstruct.MCTP.Endpoint1 \
    SetMTU u 128
```

### 移除端點

```bash
busctl call au.com.codeconstruct.MCTP1 \
    /au/com/codeconstruct/mctp1/networks/1/endpoints/10 \
    au.com.codeconstruct.MCTP.Endpoint1 \
    Remove
```

---

## D-Bus 型別簽名

常見型別簽名：

| 簽名 | 說明            | 範例                   |
| ---- | --------------- | ---------------------- |
| `y`  | byte (uint8)    | EID、訊息類型          |
| `u`  | uint32          | 網路 ID、MTU           |
| `i`  | int32           | （已棄用的網路 ID）    |
| `s`  | string          | 路徑、UUID、Role       |
| `b`  | boolean         | 是否為新端點           |
| `ay` | array of bytes  | 硬體地址、訊息類型列表 |
| `au` | array of uint32 | 版本列表               |
| `v`  | variant         | 供應商 ID              |

### 方法簽名範例

| 方法                   | 輸入  | 輸出   | 說明                                   |
| ---------------------- | ----- | ------ | -------------------------------------- |
| `SetupEndpoint`        | `ay`  | `yisb` | 硬體地址 → (EID, net, path, new)       |
| `AssignEndpointStatic` | `ayy` | `yisb` | 硬體地址 + EID → (EID, net, path, new) |
| `SetMTU`               | `u`   | -      | MTU 值                                 |
| `RegisterTypeSupport`  | `yau` | -      | 訊息類型 + 版本列表                    |

---

## 訊息類型註冊

應用程式可以向 mctpd 註冊支援的 MCTP 訊息類型：

### RegisterTypeSupport

```bash
# 註冊 PLDM (0x01) 支援，版本 1.0.0 和 1.1.0
busctl call au.com.codeconstruct.MCTP1 /au/com/codeconstruct/mctp1 \
    au.com.codeconstruct.MCTP1 \
    RegisterTypeSupport yau 1 2 0xF1F0F000 0xF1F1F000
```

### RegisterVDMTypeSupport

> ❓ **尚未實作**：當前 upstream `mctpd.c` 中不存在此方法。以下為規劃中的 API 格式，尚未實裝。

```bash
# 註冊供應商定義訊息支援（PCIe VID 格式）- 尚未實作
# busctl call au.com.codeconstruct.MCTP1 /au/com/codeconstruct/mctp1 \
#     au.com.codeconstruct.MCTP1 \
#     RegisterVDMTypeSupport yvq 0 v:q:0x8086 0x0001
```

> [!NOTE]
> 註冊會在 D-Bus 連線斷開時自動取消。

---

## 與 OpenBMC 標準的相容性

mctpd 實作 OpenBMC 定義的 MCTP 端點介面：

| OpenBMC 介面                        | mctpd 支援  |
| ----------------------------------- | ----------- |
| `xyz.openbmc_project.MCTP.Endpoint` | ✅ 完整支援 |
| `xyz.openbmc_project.Common.UUID`   | ✅ 完整支援 |

這使得其他 OpenBMC 服務（如 pldmd）可以無縫使用 mctpd 發現的端點。

---

## 相關文件

- [InterfaceAPI](InterfaceAPI.md) - Interface1 / BusOwner1
- [NetworkAPI](NetworkAPI.md) - Network1
- [EndpointAPI](EndpointAPI.md) - Endpoint / Endpoint1
- [BridgeAPI](BridgeAPI.md) - Bridge1
- [MctpdDaemon](MctpdDaemon.md) - mctpd 守護程式

---

[← 返回首頁](Home.md)
