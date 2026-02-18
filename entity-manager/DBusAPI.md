# D-Bus API 參考

## 概述

Entity-Manager 透過 D-Bus 發布硬體配置資訊，供其他 OpenBMC 應用程式（如 bmcweb、IPMI handler、dbus-sensors）使用。本文件詳細說明 Entity-Manager 的 D-Bus API 規格。

---

## 服務資訊

| 項目           | 值                                   |
| -------------- | ------------------------------------ |
| **服務名稱**   | `xyz.openbmc_project.EntityManager`  |
| **匯流排類型** | System Bus                           |
| **根路徑**     | `/xyz/openbmc_project/EntityManager` |

---

## 物件路徑

### 路徑階層結構

```
/xyz/openbmc_project/
├── EntityManager                                    # 服務根節點
└── inventory/
    └── system/
        └── {EntityType}/                            # 實體類型 (board, chassis 等)
            └── {EntityName}/                        # 實體名稱
                └── {ConfigurationName}              # 配置項目名稱
```

### Entity 物件路徑

**格式**：

```
/xyz/openbmc_project/inventory/system/{entityType}/{EntityName}
```

**範例**：

```
/xyz/openbmc_project/inventory/system/board/Intel_Front_Panel
/xyz/openbmc_project/inventory/system/board/18_Great_Card
```

> 📝 **路徑規則**：`entityType` 使用 Entity JSON 中 `Type` 值的小寫形式（如 `Board` → `board`，`Chassis` → `chassis`）。`EntityName` 中的空格會替換為底線。

### Configuration 物件路徑

**格式**：

```
/xyz/openbmc_project/Inventory/Item/{EntityType}/{EntityName}/{ConfigurationName}
```

**範例**：

```
/xyz/openbmc_project/Inventory/Item/Board/Intel_Front_Panel/Front_Panel_Temp
/xyz/openbmc_project/inventory/system/board/18_Great_Card/18_great_local
```

Configuration 是當 Entity 被新增到系統配置時所公開的元件，例如在偵測到前面板時公開的 TMP75 感測器。

---

## D-Bus 介面

### xyz.openbmc_project.{InventoryType}

庫存項目類型介面，與 Redfish 類型密切對應。

**支援的類型**：

| 類型                                             | 說明          |
| ------------------------------------------------ | ------------- |
| `xyz.openbmc_project.Inventory.Item.Bmc`         | BMC 元件      |
| `xyz.openbmc_project.Inventory.Item.Board`       | 主機板/電路板 |
| `xyz.openbmc_project.Inventory.Item.Chassis`     | 機箱          |
| `xyz.openbmc_project.Inventory.Item.Cpu`         | 處理器        |
| `xyz.openbmc_project.Inventory.Item.Dimm`        | 記憶體模組    |
| `xyz.openbmc_project.Inventory.Item.Fan`         | 風扇          |
| `xyz.openbmc_project.Inventory.Item.PowerSupply` | 電源供應器    |

**屬性**：

| 屬性           | 類型           | 說明                   |
| -------------- | -------------- | ---------------------- |
| `num_children` | `unsigned int` | 此 Entity 下的配置數量 |
| `name`         | `string`       | 庫存項目名稱           |

### xyz.openbmc_project.Configuration

配置介面，描述 Entity 擁有的元件。

**屬性**：

屬性包含 JSON 中公開的所有非物件（簡單類型）值。

**範例**：

```
路徑: /xyz/openbmc_project/Inventory/Item/Board/Intel_Front_Panel/Front_Panel_Temp
介面: xyz.openbmc_project.Configuration
    string name = "front panel temp"
    string address = "0x4D"
    string "bus" = "1"
```

### xyz.openbmc_project.Configuration.{Type}

特定類型的配置介面，`{Type}` 對應 JSON 配置中的 `Type` 欄位。

**範例**：

```
介面: xyz.openbmc_project.Configuration.TMP441
    uint64 Address = 76
    uint64 Bus = 18
    string Name = "18 great local"
    string Name1 = "18 great ext"
    string Type = "TMP441"
```

### xyz.openbmc_project.Device.{Object}.{Index}

用於描述配置中的字典成員，允許建立更複雜的類型。陣列物件使用從零開始的索引。

**屬性**：

字典的所有成員。

**範例**：

```
路徑: /xyz/openbmc_project/Inventory/Item/Board/Intel_Front_Panel/Front_Panel_Temp
介面: xyz.openbmc_project.Device.threshold.0
    string direction = "greater than"
    int value = 55
```

---

## 實際操作範例

### 使用 busctl 查看樹狀結構

```bash
# 查看 Entity-Manager 的物件樹
busctl tree --no-pager xyz.openbmc_project.EntityManager
```

**輸出範例**：

```
`-/xyz
  `-/xyz/openbmc_project
    |-/xyz/openbmc_project/EntityManager
    `-/xyz/openbmc_project/inventory
      `-/xyz/openbmc_project/inventory/system
        `-/xyz/openbmc_project/inventory/system/board
          |-/xyz/openbmc_project/inventory/system/board/18_Great_Card
          | |-/xyz/openbmc_project/inventory/system/board/18_Great_Card/18_great_local
          |-/xyz/openbmc_project/inventory/system/board/19_Great_Card
          | |-/xyz/openbmc_project/inventory/system/board/19_Great_Card/19_great_local
```

### 查看物件詳細資訊

```bash
# 查看特定物件的介面和屬性
busctl introspect --no-pager xyz.openbmc_project.EntityManager \
    /xyz/openbmc_project/inventory/system/board/18_Great_Card/18_great_local
```

**輸出範例**：

```
NAME                                     TYPE      SIGNATURE RESULT/VALUE    FLAGS
org.freedesktop.DBus.Introspectable      interface -         -               -
.Introspect                              method    -         s               -
org.freedesktop.DBus.Peer                interface -         -               -
.GetMachineId                            method    -         s               -
.Ping                                    method    -         -               -
org.freedesktop.DBus.Properties          interface -         -               -
.Get                                     method    ss        v               -
.GetAll                                  method    s         a{sv}           -
.Set                                     method    ssv       -               -
.PropertiesChanged                       signal    sa{sv}as  -               -
xyz.openbmc_project.Configuration.TMP441 interface -         -               -
.Address                                 property  t         76              emits-change
.Bus                                     property  t         18              emits-change
.Name                                    property  s         "18 great local" emits-change
.Name1                                   property  s         "18 great ext"  emits-change
.Type                                    property  s         "TMP441"        emits-change
```

### 讀取特定屬性

```bash
# 讀取單一屬性
busctl get-property xyz.openbmc_project.EntityManager \
    /xyz/openbmc_project/inventory/system/board/18_Great_Card/18_great_local \
    xyz.openbmc_project.Configuration.TMP441 \
    Name

# 輸出: s "18 great local"
```

### 使用 GetManagedObjects

```bash
# 取得服務下所有物件（用於全系統掃描）
busctl call xyz.openbmc_project.EntityManager \
    /xyz/openbmc_project/EntityManager \
    org.freedesktop.DBus.ObjectManager \
    GetManagedObjects
```

---

## FruDevice D-Bus API

fru-device 守護程式也會在 D-Bus 上發布 FRU 資訊，這些資訊被 Entity-Manager 用於 Probe 匹配。

### 服務資訊

| 項目         | 值                               |
| ------------ | -------------------------------- |
| **服務名稱** | `xyz.openbmc_project.FruDevice`  |
| **根路徑**   | `/xyz/openbmc_project/FruDevice` |

### 物件路徑範例

```
/xyz/openbmc_project/FruDevice/Super_Great
/xyz/openbmc_project/FruDevice/Super_Great_0
```

路徑基於 FRU 中最顯著的名稱欄位，重複的裝置會加上數字後綴。

### 介面與屬性

```bash
busctl introspect --no-pager xyz.openbmc_project.FruDevice \
    /xyz/openbmc_project/FruDevice/Super_Great
```

**輸出範例**：

```
NAME                                TYPE      SIGNATURE RESULT/VALUE        FLAGS
xyz.openbmc_project.FruDevice       interface -         -                   -
.ADDRESS                            property  u         80                  emits-change
.BUS                                property  u         18                  emits-change
.Common_Format_Version              property  s         "1"                 emits-change
.PRODUCT_ASSET_TAG                  property  s         "--"                emits-change
.PRODUCT_FRU_VERSION_ID             property  s         "..."               emits-change
.PRODUCT_LANGUAGE_CODE              property  s         "0"                 emits-change
.PRODUCT_MANUFACTURER               property  s         "Awesome"           emits-change
.PRODUCT_PART_NUMBER                property  s         "12345"             emits-change
.PRODUCT_PRODUCT_NAME               property  s         "Super Great"       emits-change
.PRODUCT_SERIAL_NUMBER              property  s         "12312490840"       emits-change
.PRODUCT_VERSION                    property  s         "0A"                emits-change
```

### 常用 FRU 屬性

| 屬性                    | 說明           | 常用於 Probe       |
| ----------------------- | -------------- | ------------------ |
| `BUS`                   | I2C 匯流排編號 | ✅ 替換 `$bus`     |
| `ADDRESS`               | I2C 位址       | ✅ 替換 `$address` |
| `PRODUCT_PRODUCT_NAME`  | 產品名稱       | ✅                 |
| `PRODUCT_MANUFACTURER`  | 製造商         | ✅                 |
| `PRODUCT_PART_NUMBER`   | 料號           | ✅                 |
| `PRODUCT_SERIAL_NUMBER` | 序號           |                    |
| `BOARD_PRODUCT_NAME`    | 電路板名稱     | ✅                 |
| `BOARD_MANUFACTURER`    | 電路板製造商   | ✅                 |

---

## JSON 與 D-Bus 映射規則

### 基本映射

| JSON                | D-Bus                              |
| ------------------- | ---------------------------------- |
| `"Name": "value"`   | `property s "value"`               |
| `"Bus": 18`         | `property t 18`                    |
| `"Address": "0x4c"` | `property t 76` (十六進位轉十進位) |

### 巢狀物件限制

D-Bus API 要求配置物件最多只能有一層字典或字典陣列。不支援更深層的巢狀結構。

**合法**：

```json
{
  "exposes": [
    {
      "name": "front panel temp",
      "property": { "key": "value" }
    }
  ]
}
```

**不合法**：

```json
{
  "exposes": [
    {
      "name": "front panel temp",
      "property": {
        "key": {
          "key2": "value" // ❌ 超過一層
        }
      }
    }
  ]
}
```

### 特殊類型名稱處理

D-Bus 介面名稱只允許以字母開頭的點分隔字串。因此，某些 Type 名稱會被調整：

| JSON Type         | D-Bus 介面                                       |
| ----------------- | ------------------------------------------------ |
| `24C02` ❌        | 無法使用（數字開頭）                             |
| `EEPROM_24C02` ✅ | `xyz.openbmc_project.Configuration.EEPROM_24C02` |

---

## 信號

### PropertiesChanged

當配置屬性變更時發出。

```
介面: org.freedesktop.DBus.Properties
信號: PropertiesChanged
簽章: sa{sv}as

參數:
  1. 介面名稱 (string)
  2. 變更的屬性 (dict<string, variant>)
  3. 失效的屬性名稱列表 (array<string>)
```

### InterfacesAdded / InterfacesRemoved

當新增或移除物件時，由 ObjectManager 發出。

```
介面: org.freedesktop.DBus.ObjectManager
信號: InterfacesAdded
簽章: oa{sa{sv}}

參數:
  1. 物件路徑 (object_path)
  2. 介面和屬性 (dict<string, dict<string, variant>>)
```

---

## 常用指令速查表

| 操作         | 指令                                                                           |
| ------------ | ------------------------------------------------------------------------------ |
| 查看物件樹   | `busctl tree xyz.openbmc_project.EntityManager`                                |
| 查看物件詳情 | `busctl introspect xyz.openbmc_project.EntityManager {path}`                   |
| 讀取屬性     | `busctl get-property {service} {path} {interface} {property}`                  |
| 列出所有物件 | `busctl call {service} / org.freedesktop.DBus.ObjectManager GetManagedObjects` |
| 監聽信號     | `busctl monitor xyz.openbmc_project.EntityManager`                             |

---

## 下一步

- 了解 [設定指南](ConfigurationGuide.md) 學習如何編寫配置檔
- 查看 [Probe 語法](ProbeSyntax.md) 了解進階匹配規則
- 閱讀 [FruDevice](FruDevice.md) 了解 FRU 掃描機制

---

> 📖 **參考**：[官方文件 - Entity Manager D-Bus API](https://github.com/openbmc/entity-manager/blob/master/docs/entity_manager_dbus_api.md)
