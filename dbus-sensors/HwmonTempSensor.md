# Hwmon 溫度感測器

## 簡介

HwmonTempSensor 守護程式處理透過 Linux hwmon 子系統讀取的 I2C 溫度感測器晶片。這是最常用的溫度感測器類型。

---

## 守護程式資訊

| 項目 | 值 |
|------|-----|
| 執行檔 | `hwmontempsensor` |
| D-Bus 服務 | `xyz.openbmc_project.HwmonTempSensor` |
| 資料來源 | hwmon |

---

## 支援的晶片類型

### 單通道感測器

| 類型 | 晶片說明 | I2C 位址範例 |
|------|----------|--------------|
| TMP75 | TI 單通道溫度感測器 | 0x48-0x4F |
| TMP100 | TI 精密溫度感測器 | 0x48-0x4B |
| TMP1075 | TMP75 升級版 | 0x48-0x4F |
| LM75A | NXP 標準溫度感測器 | 0x48-0x4F |
| MAX31725 | Maxim 高精度 | 0x48-0x4F |
| JC42 | JEDEC DIMM 溫度 | 0x18-0x1F |

### 多通道感測器

| 類型 | 通道數 | 說明 |
|------|--------|------|
| TMP441 | 2 | 本地 + 1 遠端二極體 |
| TMP461 | 2 | TMP441 升級版 |
| TMP421 | 2 | 2 遠端二極體 |
| TMP432 | 3 | 本地 + 2 遠端 |
| TMP464 | 5 | 本地 + 4 遠端 |
| TMP468 | 9 | 本地 + 8 遠端 |
| EMC1403 | 3 | 3 通道溫度感測器 |
| EMC1412 | 2 | 雙通道 |
| LM95234 | 5 | 多通道遠端二極體 |
| MAX6581 | 8 | 8 通道溫度監控 |
| NCT7802 | 5 | 硬體監控 IC |

### 特殊感測器

| 類型 | 說明 |
|------|------|
| BME280 | 溫度 + 濕度 + 氣壓 |
| DPS310 | 數位壓力感測器 |
| HDC1080 | 溫度 + 濕度 |
| SI7020 | 溫度 + 濕度 |
| SBTSI | AMD CPU 側頻溫度介面 |
| MCP9600 | 熱電偶放大器 |

---

## Entity-Manager 配置

### 基本單通道配置

```json
{
    "Exposes": [
        {
            "Address": "0x48",
            "Bus": 1,
            "Name": "Ambient Temp",
            "Type": "TMP75"
        }
    ],
    "Name": "Temperature Board",
    "Probe": "TRUE",
    "Type": "Board"
}
```

### 多通道配置

對於多通道感測器，使用 `Name`、`Name1`、`Name2` 等欄位命名各通道：

```json
{
    "Address": "0x4C",
    "Bus": 2,
    "Name": "CPU0 Local",
    "Name1": "CPU0 Remote",
    "Type": "TMP441"
}
```

hwmon 對應：
- `Name` → `temp1_input`（本地溫度）
- `Name1` → `temp2_input`（遠端溫度 1）

---

## 配置欄位說明

| 欄位 | 必填 | 類型 | 說明 |
|------|------|------|------|
| `Name` | ✅ | string | 主感測器名稱（temp1） |
| `Type` | ✅ | string | 晶片類型（TMP75、TMP441 等） |
| `Bus` | ✅ | number/string | I2C 匯流排編號 |
| `Address` | ✅ | string | I2C 位址（如 "0x48"） |
| `Name1`~`NameN` | ❌ | string | 額外通道名稱 |
| `PowerState` | ❌ | string | 電源狀態條件 |
| `Thresholds` | ❌ | array | 閾值配置 |
| `Labels` | ❌ | array | 明確指定要讀取的 hwmon 標籤 |
| `PollRate` | ❌ | number | 輪詢頻率（秒） |

---

## Labels 配置

某些晶片使用非標準的 hwmon 標籤，可使用 `Labels` 明確指定：

```json
{
    "Address": "0x48",
    "Bus": 1,
    "Name": "CPU Temp",
    "Type": "NCT7802",
    "Labels": ["temp1", "temp3", "temp5"]
}
```

---

## 完整設定範例

### PCIe 插卡溫度感測器

```json
{
    "Exposes": [
        {
            "Address": "0x50",
            "Bus": "$bus",
            "Name": "$bus EEPROM",
            "Type": "EEPROM_24C02"
        },
        {
            "Address": "0x4C",
            "Bus": "$bus",
            "Name": "$bus Card Local",
            "Name1": "$bus Card Remote",
            "Type": "TMP441",
            "PowerState": "On",
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
    ],
    "Name": "$bus PCIe Card",
    "Probe": "xyz.openbmc_project.FruDevice({'PRODUCT_PRODUCT_NAME': 'Super Great'})",
    "Type": "Board"
}
```

### BME280 環境感測器

```json
{
    "Address": "0x76",
    "Bus": 3,
    "Name": "Ambient Temp",
    "NamePressure": "Ambient Pressure",
    "NameHumidity": "Ambient Humidity",
    "Type": "BME280"
}
```

---

## hwmon 裝置綁定

Entity-Manager 會自動將 I2C 裝置綁定到核心驅動程式：

```bash
# Entity-Manager 執行的操作
echo "tmp441 0x4c" > /sys/bus/i2c/devices/i2c-2/new_device
```

綁定後會建立 hwmon 裝置：

```
/sys/class/hwmon/hwmonX/
├── temp1_input    # 本地溫度（毫度）
├── temp1_max      # 最大閾值
├── temp1_crit     # 嚴重閾值
├── temp2_input    # 遠端溫度
└── name           # tmp441
```

---

## D-Bus 輸出

```bash
$ busctl tree xyz.openbmc_project.HwmonTempSensor
└─/xyz
  └─/xyz/openbmc_project
    └─/xyz/openbmc_project/sensors
      └─/xyz/openbmc_project/sensors/temperature
        ├─/xyz/openbmc_project/sensors/temperature/Ambient_Temp
        ├─/xyz/openbmc_project/sensors/temperature/CPU0_Local
        └─/xyz/openbmc_project/sensors/temperature/CPU0_Remote

$ busctl introspect xyz.openbmc_project.HwmonTempSensor \
    /xyz/openbmc_project/sensors/temperature/Ambient_Temp

xyz.openbmc_project.Sensor.Value    interface
.MaxValue                           property  d  127
.MinValue                           property  d  -128
.Value                              property  d  32.5
```

---

## 除錯指令

```bash
# 掃描 I2C 匯流排
i2cdetect -y 1

# 檢查 hwmon 裝置
ls /sys/class/hwmon/

# 讀取溫度（毫度，除以 1000 得到攝氏度）
cat /sys/class/hwmon/hwmonX/temp1_input

# 檢查驅動程式綁定
ls /sys/bus/i2c/devices/1-0048/

# 查詢 D-Bus 感測器值
busctl get-property xyz.openbmc_project.HwmonTempSensor \
    /xyz/openbmc_project/sensors/temperature/Ambient_Temp \
    xyz.openbmc_project.Sensor.Value Value
```

---

## 常見問題

### 感測器未出現

1. 檢查 I2C 裝置是否存在：`i2cdetect -y <bus>`
2. 檢查 Entity-Manager 配置是否正確載入
3. 確認驅動程式已綁定裝置

### 讀數為 NaN

1. 檢查感測器電源狀態是否滿足 `PowerState` 條件
2. 確認 hwmon 檔案可讀取
3. 檢查 I2C 通訊是否正常

---

## 相關文件

- [感測器類型總覽](SensorTypes.md)
- [設定指南](ConfigurationGuide.md)
- [Entity-Manager 整合](EntityManagerIntegration.md)
- [故障排除](Troubleshooting.md)
