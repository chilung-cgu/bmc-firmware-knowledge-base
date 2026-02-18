# 核心概念

## 概述

Entity-Manager 的運作基於三個核心概念：**Entity（實體）**、**Exposes（公開記錄）** 和 **Probe（探測規則）**。理解這三個概念是掌握 Entity-Manager 設定的關鍵。

---

## Entity（實體）

### 定義

**Entity** 是指可以從 OpenBMC 系統中新增或移除的、實體獨立且可偵測的伺服器元件。一個 Entity 可能包含多個子元件，但作為整體被視為一個實體。

### 為什麼使用 "Entity" 這個術語？

傳統術語如 _Component_、_Field Replaceable Unit (FRU)_ 或 _Assembly_ 在產業中已有特定定義，而 Entity 的範圍更為廣泛，因此採用這個新術語以避免混淆。

### Entity 範例

| Entity 類型   | 範例                       | Topology 意義                                      |
| ------------- | -------------------------- | -------------------------------------------------- |
| `Board`       | 伺服器主機板、前面板、背板 | 可作為 `contained_by`/`powered_by` 的目標          |
| `Chassis`     | 伺服器機箱、刀鋒機箱       | 可作為 `containing`/`powering` 的發起者            |
| `PowerSupply` | 冗餘 PSU                   | 可作為 `powering` 的發起者和 `contained_by` 的目標 |

> ⚠️ **簡化說明**：上表列出的是在 `topology.cpp` 的 `AssocName` 定義中有特殊語義的三種 Type。實際上，`Type` 欄位可以是任意字串值，Entity-Manager 不會對未知的 Type 做限制，它只影響物件路徑生成（如 `board/`, `chassis/`）和 Topology 建模。

### JSON 表示

```json
{
  "Name": "Intel Front Panel",
  "Type": "Board",
  "Probe": "xyz.openbmc_project.FruDevice({'PRODUCT_PRODUCT_NAME': 'Front Panel'})",
  "Exposes": [
    // ... Exposes 記錄
  ]
}
```

### Entity 屬性說明

| 屬性          | 類型         | 必填 | 說明                                                                                              | Source                      |
| ------------- | ------------ | ---- | ------------------------------------------------------------------------------------------------- | --------------------------- |
| `Name`        | string       | ✅   | Entity 的人類可讀名稱，用於生成 D-Bus 物件路徑                                                    | CONFIG_FORMAT.md            |
| `Type`        | string       | ✅   | Entity 類型（`Board`, `Chassis`, `PowerSupply` 等）                                               | CONFIG_FORMAT.md            |
| `Probe`       | string/array | ✅   | 偵測規則（詳見 [Probe 語法](ProbeSyntax.md)）                                                     | CONFIG_FORMAT.md            |
| `Exposes`     | array        | ❌   | 該 Entity 公開的功能記錄                                                                          | CONFIG_FORMAT.md            |
| `Status`      | string       | ❌   | 設為 `"disabled"` 可預設停用此配置                                                                | CONFIG_FORMAT.md            |
| `Bind*`       | string       | ❌   | 繫結機制（如 `BindConnector`），將此記錄與另一個 Entity 的 Exposes 合併。值為目標 Exposes 的 Name | `perform_scan.cpp` L296-340 |
| `DisableNode` | string       | ❌   | 設定另一個 Entity 的狀態為 disabled。值為目標 Entity 的 Name                                      | `perform_scan.cpp` L311     |

> 💡 **Bind\* 機制**：`Bind` 前綴後面接的字串（如 `BindConnector` 中的 `Connector`）是繫結的類型。在 source code 中，`applyBindExposeAction()` 函數會找到目標 Exposes 記錄，並將兩者的屬性合併。這讓你可以將風扇的物理連接器和風扇本身分開定義在不同配置檔中。

---

## Exposes（公開記錄）

### 定義

**Exposes** 是 Entity 的特定功能或特性。一個 Entity 通常會有多個 Exposes 記錄，描述該元件支援的各種功能。

### Exposes 類型範例

| 類型           | 說明                  | 使用者             |
| -------------- | --------------------- | ------------------ |
| `TMP75`        | I2C 溫度感測器        | hwmontempsensor    |
| `TMP441`       | 雙通道溫度感測器      | hwmontempsensor    |
| `ADC`          | 類比數位轉換器感測器  | adcsensor          |
| `AspeedFan`    | Aspeed BMC 風扇 Tacho | fansensor          |
| `EEPROM_24C02` | 2Kbit I2C EEPROM      | fru-device         |
| `Port`         | 關聯性端口            | Entity-Manager     |
| `Pid`          | PID 控制參數          | swampd/pid-control |

### JSON 範例：溫度感測器

```json
{
  "Exposes": [
    {
      "Name": "CPU Temperature",
      "Type": "TMP75",
      "Bus": "$bus",
      "Address": "0x48",
      "Thresholds": [
        {
          "Direction": "greater than",
          "Name": "upper critical",
          "Severity": 1,
          "Value": 95
        },
        {
          "Direction": "greater than",
          "Name": "upper warning",
          "Severity": 0,
          "Value": 85
        }
      ]
    }
  ]
}
```

### JSON 範例：雙通道溫度感測器

```json
{
  "Exposes": [
    {
      "Name": "Local Temp",
      "Name1": "Remote Temp",
      "Type": "TMP441",
      "Bus": "$bus",
      "Address": "0x4c"
    }
  ]
}
```

> 💡 **說明**：`TMP441` 是雙通道感測器，`Name` 對應 temp1_input，`Name1` 對應 temp2_input。

### Exposes 通用屬性

| 屬性      | 類型       | 必填   | 說明                              |
| --------- | ---------- | ------ | --------------------------------- |
| `Name`    | string     | ✅     | 功能的人類可讀名稱                |
| `Type`    | string     | ✅     | 功能類型（決定哪個 Reactor 處理） |
| `Bus`     | string/int | 視類型 | I2C 匯流排編號                    |
| `Address` | string/int | 視類型 | I2C 位址                          |

### 門檻值設定

溫度感測器通常需要設定門檻值：

```json
{
  "Thresholds": [
    {
      "Direction": "greater than",
      "Name": "upper critical",
      "Severity": 1,
      "Value": 100
    },
    {
      "Direction": "greater than",
      "Name": "upper warning",
      "Severity": 0,
      "Value": 90
    },
    {
      "Direction": "less than",
      "Name": "lower warning",
      "Severity": 0,
      "Value": 5
    },
    {
      "Direction": "less than",
      "Name": "lower critical",
      "Severity": 1,
      "Value": 0
    }
  ]
}
```

#### 門檻值屬性

| 屬性        | 值                                                                              | 說明     |
| ----------- | ------------------------------------------------------------------------------- | -------- |
| `Direction` | `"greater than"` / `"less than"`                                                | 比較方向 |
| `Name`      | `"upper critical"` / `"upper warning"` / `"lower warning"` / `"lower critical"` | 門檻名稱 |
| `Severity`  | `0` (警告) / `1` (嚴重)                                                         | 嚴重程度 |
| `Value`     | 數值                                                                            | 門檻值   |

---

## Probe（探測規則）

### 定義

**Probe** 是一組用於偵測特定 Entity 的規則，通常以 D-Bus 介面定義的形式表達。當 Probe 規則匹配成功時，對應的 Entity 配置就會被載入。

### Probe 語法格式

```
介面名稱({'屬性名稱': '期望值', ...})
```

### 基本範例

```json
{
  "Probe": "xyz.openbmc_project.FruDevice({'PRODUCT_PRODUCT_NAME': 'Super Great'})"
}
```

這個 Probe 會匹配任何在 D-Bus 上有 `FruDevice` 介面，且 `PRODUCT_PRODUCT_NAME` 屬性值為 `"Super Great"` 的物件。

### 多條件 Probe（AND 運算）

在單一 Probe 陳述中指定多個屬性條件時，它們之間是 **AND** 關係：

```json
{
  "Probe": "xyz.openbmc_project.FruDevice({'BOARD_PRODUCT_NAME': 'Management Board', 'PRODUCT_PRODUCT_NAME': 'Yosemite V4'})"
}
```

這會匹配**同時**符合兩個條件的物件。

### 多重 Probe（OR 運算）

使用陣列可以表達 **OR** 邏輯：

```json
{
  "Probe": [
    "xyz.openbmc_project.FruDevice({'PRODUCT_PRODUCT_NAME': 'Super Great'})",
    "xyz.openbmc_project.FruDevice({'PRODUCT_PRODUCT_NAME': 'Ultra Great'})"
  ]
}
```

> ⚠️ **注意**：OR 運算存在已知的解析問題（GitHub Issue #24）。若屬性值中包含 "OR" 或 "AND" 字串，`perform_probe.cpp` 中的字串分割邏輯可能會誤判為邏輯運算子，建議避免在屬性值中使用這些保留字。

### 範本變數替換

當 Probe 匹配成功後，Entity-Manager 會從匹配的 D-Bus 物件中提取屬性值，並替換配置中的範本變數：

| 變數              | 來源                            | 說明                                                     |
| ----------------- | ------------------------------- | -------------------------------------------------------- |
| `$bus`            | 匹配物件的 `BUS` 屬性           | I2C 匯流排編號                                           |
| `$address`        | 匹配物件的 `ADDRESS` 屬性       | I2C 位址（7-bit 格式）                                   |
| `$index`          | `PerformScan::run()` 中的計數器 | 當同一個 Probe 匹配到多個 D-Bus 物件時，每次匹配自動遞增 |
| `${PropertyName}` | 匹配物件的任意屬性              | 如 `${PRODUCT_MANUFACTURER}` 會被替換為實際製造商名稱    |

#### 範本變數使用範例

```json
{
  "Probe": "xyz.openbmc_project.FruDevice({'PRODUCT_PRODUCT_NAME': 'Super Great'})",
  "Exposes": [
    {
      "Name": "$bus Great Card Sensor",
      "Type": "TMP441",
      "Bus": "$bus",
      "Address": "0x4c"
    }
  ]
}
```

假設在匯流排 18 上偵測到匹配的 FRU：

```json
// 替換後的結果
{
  "Name": "18 Great Card Sensor",
  "Type": "TMP441",
  "Bus": 18,
  "Address": "0x4c"
}
```

---

## 概念關係圖

```
┌─────────────────────────────────────────────────────────────┐
│                         Entity                               │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │ Name: "Intel Front Panel"                               │ │
│  │ Type: "Board"                                           │ │
│  │                                                         │ │
│  │ ┌─────────────────────────────────────────────────────┐ │ │
│  │ │ Probe                                               │ │ │
│  │ │ 規則: FruDevice({'PRODUCT_NAME': 'Front Panel'})    │ │ │
│  │ │                                                     │ │ │
│  │ │ 匹配條件 ──▶ 決定何時載入此 Entity                   │ │ │
│  │ └─────────────────────────────────────────────────────┘ │ │
│  │                                                         │ │
│  │ ┌─────────────────────────────────────────────────────┐ │ │
│  │ │ Exposes                                             │ │ │
│  │ │ ┌───────────────────┐ ┌───────────────────┐        │ │ │
│  │ │ │ TMP75 Sensor     │ │ EEPROM            │        │ │ │
│  │ │ │ Name: "Panel Temp"│ │ Name: "FRU EEPROM"│        │ │ │
│  │ │ │ Bus: $bus        │ │ Bus: $bus         │        │ │ │
│  │ │ │ Address: 0x48    │ │ Address: $address │        │ │ │
│  │ │ └───────────────────┘ └───────────────────┘        │ │ │
│  │ │                                                     │ │ │
│  │ │ 功能描述 ──▶ 發布到 D-Bus 供 Reactor 使用            │ │ │
│  │ └─────────────────────────────────────────────────────┘ │ │
│  └─────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

---

## 完整配置範例

以下是一個完整的 Entity 配置檔案範例，包含所有核心概念：

```json
{
  "Name": "$bus Great Card",
  "Type": "Board",
  "Probe": "xyz.openbmc_project.FruDevice({'PRODUCT_PRODUCT_NAME': 'Super Great'})",
  "Exposes": [
    {
      "Name": "$bus great eeprom",
      "Type": "EEPROM_24C02",
      "Bus": "$bus",
      "Address": "$address"
    },
    {
      "Name": "$bus great local",
      "Name1": "$bus great ext",
      "Type": "TMP441",
      "Bus": "$bus",
      "Address": "0x4c"
    }
  ]
}
```

### 執行流程

1. **偵測**：`fru-device` 在匯流排 18 上發現 FRU，`PRODUCT_PRODUCT_NAME` 為 "Super Great"
2. **匹配**：Entity-Manager 的 Probe 規則匹配成功
3. **替換**：`$bus` 被替換為 `18`，`$address` 被替換為實際位址
4. **發布**：Entity-Manager 在 D-Bus 上發布：
   - `/xyz/openbmc_project/inventory/system/board/18_Great_Card`
   - `/xyz/openbmc_project/inventory/system/board/18_Great_Card/18_great_local`
5. **反應**：`hwmontempsensor` 偵測到 `TMP441` 類型，建立感測器

---

## 下一步

- 了解 [D-Bus API](DBusAPI.md) 中詳細的介面規格
- 查看 [Probe 語法](ProbeSyntax.md) 了解進階探測規則
- 閱讀 [設定範例](ExampleConfigurations.md) 取得更多實際案例

---

> 📖 **參考**：[官方文件 - My First Sensors](https://github.com/openbmc/entity-manager/blob/master/docs/my_first_sensors.md)
