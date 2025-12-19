# 設定指南

## 簡介

dbus-sensors 的感測器配置透過 Entity-Manager 的 JSON 配置檔進行。本文件說明配置結構和常用參數。

---

## 配置流程

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│ JSON 配置檔     │────▶│ Entity-Manager  │────▶│ dbus-sensors    │
│ (Exposes 記錄)  │     │ (發布配置)       │     │ (建立感測器)    │
└─────────────────┘     └─────────────────┘     └─────────────────┘
```

---

## 基本 JSON 結構

```json
{
    "Exposes": [
        {
            "Name": "Sensor Name",
            "Type": "SensorType",
            ...
        }
    ],
    "Name": "Configuration Name",
    "Probe": "Probe Expression",
    "Type": "Board"
}
```

---

## Exposes 記錄

每個 Exposes 元素定義一個感測器配置。

### 必填欄位

| 欄位 | 類型 | 說明 |
|------|------|------|
| `Name` | string | 感測器名稱（用於 D-Bus 路徑） |
| `Type` | string | 感測器類型（決定由哪個守護程式處理） |

### 常用選填欄位

| 欄位 | 類型 | 說明 |
|------|------|------|
| `Bus` | number/string | I2C 匯流排編號 |
| `Address` | string | I2C 位址（如 "0x48"） |
| `Index` | number | 通道索引 |
| `PowerState` | string | 電源狀態條件 |
| `Thresholds` | array | 閾值配置陣列 |

---

## Type 類型對應

| Type | 守護程式 | 資料來源 |
|------|----------|----------|
| `ADC` | ADCSensor | hwmon (IIO) |
| `TMP75`, `TMP441`, ... | HwmonTempSensor | hwmon |
| `pmbus`, `INA219`, ... | PSUSensor | hwmon (PMBus) |
| `AspeedFan`, `I2CFan` | FanSensor | hwmon |
| `XeonCPU` | CPUSensor | PECI |
| `NVME1000` | NVMeSensor | SMBus |
| `ExternalSensor` | ExternalSensor | D-Bus |
| `IpmbSensor` | IpmbSensor | IPMB |
| `ChassisIntrusionSensor` | IntrusionSensor | GPIO/I2C |

---

## PowerState 電源狀態

控制感測器何時啟用：

| 值 | 說明 |
|----|------|
| `"Always"` | 始終啟用 |
| `"On"` | 主機開機時啟用 |
| `"ChassisOn"` | 底板電源開啟時啟用 |
| `"BiosPost"` | BIOS POST 期間啟用 |

**範例：**

```json
{
    "Name": "CPU0 Temp",
    "Type": "XeonCPU",
    "PowerState": "On"
}
```

---

## 範本變數

Entity-Manager 支援範本變數，根據 Probe 匹配的裝置資訊填入：

| 變數 | 說明 | 來源 |
|------|------|------|
| `$bus` | I2C 匯流排編號 | FruDevice.BUS |
| `$address` | I2C 位址 | FruDevice.ADDRESS |
| `$name` | FRU 名稱 | FruDevice 物件路徑 |

**範例：**

```json
{
    "Exposes": [
        {
            "Address": "0x4C",
            "Bus": "$bus",
            "Name": "$bus Card Temp",
            "Type": "TMP441"
        }
    ],
    "Probe": "xyz.openbmc_project.FruDevice({'PRODUCT_PRODUCT_NAME': 'MyCard'})"
}
```

若在 bus 5 發現匹配的 FRU，則：
- `Bus` = 5
- `Name` = "5 Card Temp"

---

## I2C 匯流排與位址

### 位址格式

位址通常以十六進位字串表示：

```json
{
    "Address": "0x48",
    "Bus": 1
}
```

### 匯流排編號

可以是數字或字串：

```json
{
    "Bus": 1
}
```

或使用範本變數：

```json
{
    "Bus": "$bus"
}
```

---

## 完整配置範例

### 主機板溫度感測器

```json
{
    "Exposes": [
        {
            "Address": "0x48",
            "Bus": 1,
            "Name": "Inlet Temp",
            "PowerState": "Always",
            "Type": "TMP75",
            "Thresholds": [
                {
                    "Direction": "greater than",
                    "Name": "upper critical",
                    "Severity": 1,
                    "Value": 40
                },
                {
                    "Direction": "greater than",
                    "Name": "upper warning",
                    "Severity": 0,
                    "Value": 35
                }
            ]
        },
        {
            "Address": "0x49",
            "Bus": 1,
            "Name": "Outlet Temp",
            "PowerState": "Always",
            "Type": "TMP75"
        }
    ],
    "Name": "Baseboard Temps",
    "Probe": "TRUE",
    "Type": "Board"
}
```

### 動態插卡配置

```json
{
    "Exposes": [
        {
            "Address": "0x50",
            "Bus": "$bus",
            "Name": "$bus_EEPROM",
            "Type": "EEPROM_24C02"
        },
        {
            "Address": "0x4C",
            "Bus": "$bus",
            "Name": "$bus_Temp",
            "Name1": "$bus_Temp_Remote",
            "PowerState": "On",
            "Type": "TMP441"
        }
    ],
    "Name": "$bus Expansion Card",
    "Probe": "xyz.openbmc_project.FruDevice({'PRODUCT_MANUFACTURER': 'Acme'})",
    "Type": "Board"
}
```

---

## 配置檔位置

Entity-Manager 配置檔通常位於：

```
/usr/share/entity-manager/configurations/
```

檔案格式為 JSON，副檔名為 `.json`。

---

## 驗證配置

使用 Entity-Manager 的 schema 驗證：

```bash
# 驗證 JSON 語法
python -m json.tool < config.json

# 驗證 schema（如果有 jsonschema）
jsonschema -i config.json legacy.json
```

---

## 除錯技巧

### 檢查 Entity-Manager 載入的配置

```bash
busctl tree xyz.openbmc_project.EntityManager
```

### 確認配置介面

```bash
busctl introspect xyz.openbmc_project.EntityManager \
    /xyz/openbmc_project/inventory/system/board/Baseboard_Temps/Inlet_Temp
```

### 查看日誌

```bash
journalctl -u xyz.openbmc_project.EntityManager.service
```

---

## 相關文件

- [閾值設定](ThresholdConfiguration.md)
- [Entity-Manager 整合](EntityManagerIntegration.md)
- [Entity-Manager 設定指南](../entity-manager/ConfigurationGuide.md)
