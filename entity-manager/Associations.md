# 關聯性

## 概述

Entity-Manager 會在特定情況下自動建立實體間的**關聯性（Associations）**。這些關聯性是 OpenBMC 物理拓撲設計的一部分，用於描述硬體元件之間的物理和邏輯關係。

---

## 關聯性類型

### 概覽

| 關聯類型   | 正向         | 反向           | 用途                    | 允許的發起者 Type        | 允許的目標 Type                   |
| ---------- | ------------ | -------------- | ----------------------- | ------------------------ | --------------------------------- |
| containing | `containing` | `contained_by` | 容納關係（機箱→主機板） | `Chassis`                | `Board`, `Chassis`, `PowerSupply` |
| powering   | `powering`   | `powered_by`   | 供電關係（PSU→主機板）  | `Chassis`, `PowerSupply` | `Board`, `Chassis`, `PowerSupply` |
| probing    | `probing`    | `probed_by`    | 探測關係（自動建立）    | 任意                     | 任意                              |

> 📝 **Source**：上表的 `allowedOnBoardTypes` 和 `allowedOnBoardTypesReverse` 定義在 `topology.cpp` L3-27。

---

## containing 關聯

### 用途

描述父子容納關係，例如機箱包含主機板。

### 配置方式

**被容納者（子元件）**：

```json
{
  "Name": "My Baseboard",
  "Type": "Board",
  "Exposes": [
    {
      "Name": "ContainingPort",
      "Type": "Port",
      "PortType": "contained_by"
    }
  ]
}
```

**容納者（父元件）**：

```json
{
  "Name": "Server Chassis",
  "Type": "Chassis",
  "Exposes": [
    {
      "Name": "ContainingPort",
      "Type": "Port",
      "PortType": "containing"
    }
  ]
}
```

### 運作原理

1. Entity-Manager 載入兩個配置
2. 發現 "ContainingPort" 名稱相符
3. 自動建立雙向關聯：
   - Chassis → Board (containing)
   - Board → Chassis (contained_by)

### D-Bus 表示

```bash
# 在機箱物件上
busctl get-property xyz.openbmc_project.EntityManager \
    /xyz/openbmc_project/inventory/system/chassis/Server_Chassis \
    xyz.openbmc.Associations \
    Associations

# 輸出: a(sss) 1 "containing" "contained_by" "/xyz/openbmc_project/inventory/system/board/My_Baseboard"
```

---

## powering 關聯

### 用途

描述供電關係，例如 PSU 為主機板供電。

### 配置方式

**被供電者（主機板）**：

```json
{
  "Name": "My Baseboard",
  "Type": "Board",
  "Exposes": [
    {
      "Name": "GenericPowerPort",
      "Type": "Port",
      "PortType": "powered_by"
    }
  ]
}
```

**供電者（PSU）**：

```json
{
  "Name": "Generic PSU",
  "Type": "PowerSupply",
  "Exposes": [
    {
      "Name": "GenericPowerPort",
      "Type": "Port",
      "PortType": "powering"
    }
  ]
}
```

### 特點

- PSU 可以是通用的，適用於多種主機板
- 只要 Port 名稱相符，就會建立關聯

---

## probing 關聯

### 用途

將庫存項目與其被探測到的 D-Bus 路徑連結。這是**自動建立**的關聯。

### 運作原理

當 Probe 陳述匹配到特定 D-Bus 路徑時，Entity-Manager 會自動建立 probing 關聯。

**範例配置**：

```json
{
  "Name": "Yosemite 4 Management Board",
  "Type": "Board",
  "Probe": "xyz.openbmc_project.FruDevice({'BOARD_PRODUCT_NAME': 'Management Board wBMC', 'PRODUCT_PRODUCT_NAME': 'Yosemite V4'})"
}
```

**匹配過程**：

1. Entity-Manager 查詢 ObjectMapper
2. 找到符合條件的 FruDevice 路徑：`/xyz/openbmc_project/FruDevice/Management_Board_wBMC`
3. 載入配置
4. 建立 probing 關聯

### D-Bus 結果

```
inventory 物件: /xyz/openbmc_project/inventory/system/board/Yosemite_4_Management_Board

Associations:
- probing: /xyz/openbmc_project/FruDevice/Management_Board_wBMC
- probed_by: (反向關聯)
```

---

## Port 配置詳解

### Port 類型屬性

```json
{
  "Name": "PortName",
  "Type": "Port",
  "PortType": "PortDirection"
}
```

### PortType 值

| PortType       | 方向 | 說明           |
| -------------- | ---- | -------------- |
| `containing`   | 正向 | 包含其他元件   |
| `contained_by` | 反向 | 被其他元件包含 |
| `powering`     | 正向 | 為其他元件供電 |
| `powered_by`   | 反向 | 由其他元件供電 |

### 名稱匹配規則

- Port 的 `Name` 屬性用於匹配
- 當兩個配置有相同的 Port `Name` 且 PortType 互補時，建立關聯
- 如果找不到匹配的 Port，不會報錯，只是不建立關聯

### DownstreamPort 類型

> ⚠️ **簡化說明**：`DownstreamPort` 是 `topology.cpp` L125-143 中定義的另一種 Port 類型，它使用 `ConnectsToType` 屬性而非 `PortType` 來指定連接目標。這是一種更靈活的配置方式，但目前文件較少提及。另外，若有 `PowerPort` 屬性存在，還會額外建立 `powering` 關聯。

```json
{
  "Name": "DownstreamSlot",
  "Type": "DownstreamPort",
  "ConnectsToType": "UpstreamSlotType"
}
```

---

## 實際範例

### 完整系統拓撲

**機箱配置 (chassis.json)**：

```json
{
  "Name": "1U Server Chassis",
  "Type": "Chassis",
  "Probe": "xyz.openbmc_project.FruDevice({'PRODUCT_PRODUCT_NAME': '1U Chassis'})",
  "Exposes": [
    {
      "Name": "Chassis FRU",
      "Type": "EEPROM_24C02",
      "Bus": "$bus",
      "Address": "$address"
    },
    {
      "Name": "MainBoardPort",
      "Type": "Port",
      "PortType": "containing"
    },
    {
      "Name": "PowerPort",
      "Type": "Port",
      "PortType": "powered_by"
    }
  ]
}
```

**主機板配置 (baseboard.json)**：

```json
{
  "Name": "Server Baseboard",
  "Type": "Board",
  "Probe": "xyz.openbmc_project.FruDevice({'PRODUCT_PRODUCT_NAME': 'Server Board'})",
  "Exposes": [
    {
      "Name": "Board FRU",
      "Type": "EEPROM_24C02",
      "Bus": "$bus",
      "Address": "$address"
    },
    {
      "Name": "MainBoardPort",
      "Type": "Port",
      "PortType": "contained_by"
    },
    {
      "Name": "PowerPort",
      "Type": "Port",
      "PortType": "powered_by"
    }
  ]
}
```

**PSU 配置 (psu.json)**：

```json
{
  "Name": "Delta PSU",
  "Type": "PowerSupply",
  "Probe": "xyz.openbmc_project.FruDevice({'PRODUCT_MANUFACTURER': 'Delta'})",
  "Exposes": [
    {
      "Name": "PSU FRU",
      "Type": "EEPROM_24C02",
      "Bus": "$bus",
      "Address": "$address"
    },
    {
      "Name": "PowerPort",
      "Type": "Port",
      "PortType": "powering"
    }
  ]
}
```

### 結果拓撲

```
                    ┌─────────────────┐
                    │  Delta PSU      │
                    │  (PowerSupply)  │
                    └────────┬────────┘
                             │ powering / powered_by
                             ▼
┌────────────────────────────────────────────────────┐
│                  1U Server Chassis                  │
│                     (Chassis)                       │
│                                                     │
│    containing │                                     │
│               ▼                                     │
│    ┌─────────────────────┐                          │
│    │   Server Baseboard  │                          │
│    │      (Board)        │                          │
│    │   contained_by ▲    │                          │
│    └─────────────────────┘                          │
└────────────────────────────────────────────────────┘
```

---

## D-Bus 查詢關聯性

### 查看物件的關聯

```bash
busctl get-property xyz.openbmc_project.EntityManager \
    /xyz/openbmc_project/inventory/system/board/Server_Baseboard \
    xyz.openbmc.Associations \
    associations
```

### 輸出格式

```
a(sss) 2 \
    "contained_by" "containing" "/xyz/openbmc_project/inventory/system/chassis/1U_Server_Chassis" \
    "powered_by" "powering" "/xyz/openbmc_project/inventory/system/powersupply/Delta_PSU"
```

### 欄位說明

每個關聯是一個三元組 `(forward, reverse, path)`：

| 欄位    | 說明                       |
| ------- | -------------------------- |
| forward | 從目前物件到目標的關聯類型 |
| reverse | 從目標到目前物件的關聯類型 |
| path    | 目標物件的 D-Bus 路徑      |

---

## 物理拓撲設計

### 設計目標

關聯性機制是 OpenBMC [物理拓撲設計](https://github.com/openbmc/docs/blob/master/designs/physical-topology.md) 的一部分，目標包括：

1. **描述硬體層次**：機箱 → 主機板 → 子卡
2. **追蹤電源路徑**：PSU → 機箱 → 子元件
3. **支援 Redfish 呈現**：正確的父子關係

### 與 Redfish 的對應

| 關聯類型     | Redfish             | 用途           |
| ------------ | ------------------- | -------------- |
| containing   | Chassis.Contains    | 機箱包含的元件 |
| contained_by | Chassis.ContainedBy | 所在的機箱     |
| powering     | PoweredBy           | 電源來源       |

---

## 注意事項

### 可選的關聯

- Port 關聯是**可選的**
- 如果找不到匹配的 Port，不會報錯
- 這允許元件在不同系統中使用

### 多對多關聯

- 一個 PSU 可以為多個主機板供電
- 一個機箱可以包含多個主機板
- 只需確保 Port 名稱正確匹配

### 除錯技巧

```bash
# 查看所有 Entity-Manager 物件
busctl tree xyz.openbmc_project.EntityManager

# 查看特定物件的所有介面
busctl introspect xyz.openbmc_project.EntityManager \
    /xyz/openbmc_project/inventory/system/board/My_Board

# 檢查 Port 配置是否正確載入
busctl get-property xyz.openbmc_project.EntityManager \
    /xyz/openbmc_project/inventory/system/board/My_Board/MainBoardPort \
    xyz.openbmc_project.Configuration.Port \
    PortType
```

---

## 下一步

- 了解 [設定指南](ConfigurationGuide.md) 中 Port 配置的詳細說明
- 查看 [相容軟體](CompatibleSoftware.md) 了解 Redfish 如何使用關聯性
- 閱讀 [故障排除](Troubleshooting.md) 解決關聯性問題

---

> 📖 **參考**：
>
> - [Entity-Manager 關聯性文件](https://github.com/openbmc/entity-manager/blob/master/docs/associations.md)
> - [OpenBMC 物理拓撲設計](https://github.com/openbmc/docs/blob/master/designs/physical-topology.md)
> - [OpenBMC Object Mapper 關聯性](https://github.com/openbmc/docs/blob/master/architecture/object-mapper.md#associations)
