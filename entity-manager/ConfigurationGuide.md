# 設定指南

## 概述

Entity-Manager 使用 JSON 格式的設定檔來定義硬體配置。這些設定檔位於 entity-manager 儲存庫的 `./configurations` 目錄中，每個支援的裝置型號都有一個專屬的設定檔。

---

## 設定檔位置

### 標準位置

Entity-Manager 的 `main.cpp` 中定義了兩個配置檔路徑：

```cpp
// main.cpp L13-14
const std::vector<std::filesystem::path> configurationDirectories = {
    PACKAGE_DIR "configurations", SYSCONF_DIR "configurations"};
```

實際路徑通常為：

```
/usr/share/entity-manager/configurations/    # PACKAGE_DIRฌ編譯時安裝的配置
/etc/entity-manager/configurations/          # SYSCONF_DIR，用於執行時期的自訂配置
```

```
entity-manager/configurations/
```

### 持久化檔案

```
/var/configuration/system.json
```

Entity-Manager 會將目前的系統配置寫入此檔案以便持久化。

---

## 設定檔結構

### 基本結構

```json
{
  "Name": "裝置名稱",
  "Type": "Entity 類型",
  "Probe": "探測規則",
  "Exposes": [
    // 公開的功能列表
  ]
}
```

### 完整範例

```json
{
  "Name": "My Baseboard",
  "Type": "Board",
  "Probe": "xyz.openbmc_project.FruDevice({'PRODUCT_PRODUCT_NAME': 'MyBoard'})",
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
          "Value": 100
        }
      ]
    },
    {
      "Name": "Board FRU",
      "Type": "EEPROM_24C02",
      "Bus": "$bus",
      "Address": "$address"
    }
  ]
}
```

---

## 必填欄位

### Entity 層級

| 欄位    | 類型         | 說明                                        |
| ------- | ------------ | ------------------------------------------- |
| `Name`  | string       | Entity 的顯示名稱，可使用範本變數           |
| `Type`  | string       | Entity 類型（見下方支援類型）               |
| `Probe` | string/array | 偵測規則（見 [Probe 語法](ProbeSyntax.md)） |

### Exposes 層級

| 欄位   | 類型   | 說明                                |
| ------ | ------ | ----------------------------------- |
| `Name` | string | 配置項目名稱                        |
| `Type` | string | 功能類型（決定由哪個 Reactor 處理） |

---

## 支援的 Entity 類型

| Type          | 說明                   | D-Bus 路徑前綴           | Topology 特殊語義                 |
| ------------- | ---------------------- | ------------------------ | --------------------------------- |
| `Board`       | 電路板（主機板、子卡） | `.../board/{Name}`       | `contained_by`, `powered_by` 目標 |
| `Chassis`     | 機箱                   | `.../chassis/{Name}`     | `containing`, `powering` 發起者   |
| `PowerSupply` | 電源供應器             | `.../powersupply/{Name}` | `powering` 發起者                 |

> ⚠️ **簡化說明**：`Type` 欄位可以是任意字串（如 `CPU`、`Fan` 等），但上表三種是在 `topology.cpp` 的 `AssocName` 定義中有明確語義的。其他 Type 值會以小寫形式作為物件路徑的子目錄。

---

## 範本變數

### 可用變數

當 Probe 匹配成功後，以下變數會被自動替換：

| 變數          | 來源                            | 說明                         |
| ------------- | ------------------------------- | ---------------------------- |
| `$bus`        | 匹配物件的 `BUS` 屬性           | I2C 匯流排編號               |
| `$address`    | 匹配物件的 `ADDRESS` 屬性       | I2C 裝置位址（7-bit）        |
| `$index`      | `PerformScan::run()` 中的計數器 | 多重匹配時的自動遞增索引     |
| `${PROPERTY}` | 匹配物件的任意 D-Bus 屬性       | 如 `${PRODUCT_MANUFACTURER}` |

### 使用範例

```json
{
  "Name": "$bus Great Card",
  "Type": "Board",
  "Probe": "xyz.openbmc_project.FruDevice({'PRODUCT_PRODUCT_NAME': 'Super Great'})",
  "Exposes": [
    {
      "Name": "$bus-$address sensor",
      "Type": "TMP75",
      "Bus": "$bus",
      "Address": "0x48"
    }
  ]
}
```

### 替換結果

假設在匯流排 18、位址 0x50 發現匹配的 FRU：

```json
{
  "Name": "18 Great Card",
  "Exposes": [
    {
      "Name": "18-80 sensor",
      "Bus": 18,
      "Address": "0x48"
    }
  ]
}
```

---

## Exposes 類型詳解

### 溫度感測器

#### TMP75 / TMP175 / TMP275

```json
{
    "Name": "Inlet Temperature",
    "Type": "TMP75",
    "Bus": "$bus",
    "Address": "0x48",
    "PowerState": "On",
    "Thresholds": [...]
}
```

#### TMP421 / TMP441 / TMP461

多通道溫度感測器：

```json
{
  "Name": "Local Temp",
  "Name1": "Remote Temp",
  "Type": "TMP441",
  "Bus": "$bus",
  "Address": "0x4c"
}
```

| 屬性    | 對應        | 說明               |
| ------- | ----------- | ------------------ |
| `Name`  | temp1_input | 本地溫度           |
| `Name1` | temp2_input | 遠端溫度 1         |
| `Name2` | temp3_input | 遠端溫度 2（若有） |

### ADC 感測器

```json
{
  "Name": "P12V Voltage",
  "Type": "ADC",
  "Index": 0,
  "ScaleFactor": 1.0,
  "PowerState": "On",
  "Thresholds": [
    {
      "Direction": "greater than",
      "Name": "upper critical",
      "Severity": 1,
      "Value": 13.2
    },
    {
      "Direction": "less than",
      "Name": "lower critical",
      "Severity": 1,
      "Value": 10.8
    }
  ]
}
```

### 風扇感測器

```json
{
  "Name": "System Fan 1",
  "Type": "AspeedFan",
  "Index": 0,
  "Connector": {
    "Name": "Fan Connector 1",
    "Pwm": 0,
    "Tachs": [0]
  }
}
```

### EEPROM

```json
{
  "Name": "Board FRU EEPROM",
  "Type": "EEPROM_24C02",
  "Bus": "$bus",
  "Address": "$address"
}
```

### Port（關聯性端口）

```json
{
  "Name": "ContainingPort",
  "Type": "Port",
  "PortType": "contained_by"
}
```

詳見 [關聯性](Associations.md)。

---

## 門檻值設定

### 標準門檻值結構

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
      "Value": 10
    },
    {
      "Direction": "less than",
      "Name": "lower critical",
      "Severity": 1,
      "Value": 5
    }
  ]
}
```

### 門檻值屬性說明

| 屬性        | 可用值                                                                       | 說明     |
| ----------- | ---------------------------------------------------------------------------- | -------- |
| `Direction` | `"greater than"`, `"less than"`                                              | 比較方向 |
| `Name`      | `"upper critical"`, `"upper warning"`, `"lower warning"`, `"lower critical"` | 門檻名稱 |
| `Severity`  | `0` (警告), `1` (嚴重)                                                       | 嚴重程度 |
| `Value`     | 數值                                                                         | 門檻數值 |

---

## PowerState 屬性

控制感測器在不同電源狀態下的行為：

| 值           | 說明                   |
| ------------ | ---------------------- |
| `"On"`       | 僅在主機電源開啟時輪詢 |
| `"BiosPost"` | 在 BIOS POST 期間輪詢  |
| `"Always"`   | 始終輪詢（預設）       |

```json
{
    "Name": "CPU Temperature",
    "Type": "TMP75",
    "PowerState": "On",
    ...
}
```

---

## 模組化設定

### 建議的檔案分割方式

建議將配置資訊分割到多個檔案，而非將所有內容放在單一檔案中：

```
configurations/
├── my_baseboard.json      # 主機板配置
├── my_chassis.json        # 機箱配置
├── my_psu_model_a.json    # PSU 型號 A
└── my_psu_model_b.json    # PSU 型號 B
```

### 優點

- 每個硬體模組獨立管理
- 易於維護和除錯
- 支援相同 PSU 用於不同主機板

---

## JSON 語法注意事項

### 支援 C 風格註解

Entity-Manager 的 JSON 解析器支援 C 風格註解：

```json
{
  "Name": "My Board",
  // 這是單行註解
  "Type": "Board",
  /*
   * 這是多行註解
   * 可以用於說明
   */
  "Probe": "..."
}
```

### 字串中的特殊字元

使用跳脫字元處理特殊字元：

```json
{
  "Name": "Board \"Special\" Name",
  "Description": "Line1\nLine2"
}
```

### 數值格式

```json
{
  "Address": "0x48", // 十六進位字串
  "Address": 72, // 十進位整數
  "Bus": 1, // 整數
  "ScaleFactor": 1.5 // 浮點數
}
```

---

## 驗證配置

### JSON Schema 驗證

Entity-Manager 提供 JSON Schema 進行驗證：

```bash
# 在建置時驗證
meson test -C build
```

### 手動驗證

使用 Python 進行 JSON 語法檢查：

```bash
python3 -m json.tool my_board.json
```

---

## 配置檔命名慣例

### 建議格式

```
{製造商}_{型號}[_{變體}].json
```

### 範例

```
intel_s2600wf.json
supermicro_x11spl.json
dell_r740_baseboard.json
generic_psu_1200w.json
```

---

## 最佳實踐

### ✅ 建議

1. **每個裝置一個檔案**：便於維護和支援確認
2. **使用有意義的名稱**：便於識別
3. **設定完整的門檻值**：包含上下限警告和嚴重門檻
4. **使用範本變數**：支援多個實例
5. **加入註解**：說明特殊設定

### ❌ 避免

1. **過度巢狀**：D-Bus 只支援一層字典
2. **硬編碼匯流排號**：使用 `$bus` 變數
3. **重複配置**：使用模組化設計
4. **跳過驗證**：提交前先驗證 JSON

---

## 下一步

- 了解 [Probe 語法](ProbeSyntax.md) 的進階用法
- 查看 [設定範例](ExampleConfigurations.md) 取得更多實際案例
- 閱讀 [故障排除](Troubleshooting.md) 解決常見問題

---

> 📖 **參考**：[官方配置文件](https://github.com/openbmc/entity-manager/tree/master/configurations)
