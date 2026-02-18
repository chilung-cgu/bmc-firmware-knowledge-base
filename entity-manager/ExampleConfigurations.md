# 設定範例

## 概述

本文件提供多種實際場景的 Entity-Manager 設定範例，從簡單的溫度感測器到複雜的多元件配置。

---

## 溫度感測器範例

### 簡單 TMP75 感測器

```json
{
  "Name": "My Baseboard",
  "Type": "Board",
  "Probe": "xyz.openbmc_project.FruDevice({'PRODUCT_PRODUCT_NAME': 'MyBoard'})",
  "Exposes": [
    {
      "Name": "Inlet Temperature",
      "Type": "TMP75",
      "Bus": "$bus",
      "Address": "0x48",
      "Thresholds": [
        {
          "Direction": "greater than",
          "Name": "upper critical",
          "Severity": 1,
          "Value": 55
        },
        {
          "Direction": "greater than",
          "Name": "upper warning",
          "Severity": 0,
          "Value": 45
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
  ]
}
```

### 雙通道 TMP441 感測器

```json
{
  "Name": "$bus GPU Card",
  "Type": "Board",
  "Probe": "xyz.openbmc_project.FruDevice({'PRODUCT_PRODUCT_NAME': 'GPU Riser'})",
  "Exposes": [
    {
      "Name": "$bus GPU Local",
      "Name1": "$bus GPU Remote",
      "Type": "TMP441",
      "Bus": "$bus",
      "Address": "0x4c",
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
          "Value": 85
        }
      ]
    }
  ]
}
```

---

## PCIe 擴充卡範例

### 完整 PCIe 卡配置

此範例展示如何配置一張帶有 EEPROM 和溫度感測器的 PCIe 卡：

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

### 說明

1. **Probe**：當 FruDevice 服務發現 `PRODUCT_PRODUCT_NAME` 為 "Super Great" 的 FRU 時觸發
2. **EEPROM**：使用 `$bus` 和 `$address` 範本變數，自動從 FRU 取得位址
3. **TMP441**：雙通道溫度感測器，`Name` 對應本地溫度，`Name1` 對應遠端溫度

---

## 電源供應器範例

### 基本 PSU 配置

```json
{
  "Name": "Delta PSU $index",
  "Type": "PowerSupply",
  "Probe": "xyz.openbmc_project.FruDevice({'PRODUCT_MANUFACTURER': 'Delta', 'PRODUCT_PRODUCT_NAME': 'DPS-1200'})",
  "Exposes": [
    {
      "Name": "PSU$index FRU",
      "Type": "EEPROM_24C02",
      "Bus": "$bus",
      "Address": "$address"
    },
    {
      "Name": "PSU$index Input Voltage",
      "Type": "PMBus",
      "Bus": "$bus",
      "Address": "0x58",
      "SensorType": "Voltage",
      "PowerState": "On"
    },
    {
      "Name": "PSU$index Output Power",
      "Type": "PMBus",
      "Bus": "$bus",
      "Address": "0x58",
      "SensorType": "Power",
      "PowerState": "On"
    },
    {
      "Name": "PSU$index Temperature",
      "Type": "PMBus",
      "Bus": "$bus",
      "Address": "0x58",
      "SensorType": "Temperature",
      "PowerState": "On",
      "Thresholds": [
        {
          "Direction": "greater than",
          "Name": "upper critical",
          "Severity": 1,
          "Value": 85
        },
        {
          "Direction": "greater than",
          "Name": "upper warning",
          "Severity": 0,
          "Value": 75
        }
      ]
    }
  ]
}
```

---

## 主機板範例

### 完整主機板配置

```json
{
  "Name": "Intel S2600WF Baseboard",
  "Type": "Board",
  "Probe": "xyz.openbmc_project.FruDevice({'PRODUCT_PART_NUMBER': 'S2600WF'})",
  "Exposes": [
    // FRU EEPROM
    {
      "Name": "Baseboard FRU",
      "Type": "EEPROM_24C64",
      "Bus": "$bus",
      "Address": "$address"
    },

    // 入風口溫度
    {
      "Name": "Inlet Temp",
      "Type": "TMP75",
      "Bus": 6,
      "Address": "0x48",
      "PowerState": "Always",
      "Thresholds": [
        {
          "Direction": "greater than",
          "Name": "upper critical",
          "Severity": 1,
          "Value": 45
        },
        {
          "Direction": "greater than",
          "Name": "upper warning",
          "Severity": 0,
          "Value": 40
        }
      ]
    },

    // 出風口溫度
    {
      "Name": "Outlet Temp",
      "Type": "TMP75",
      "Bus": 6,
      "Address": "0x49",
      "PowerState": "Always",
      "Thresholds": [
        {
          "Direction": "greater than",
          "Name": "upper critical",
          "Severity": 1,
          "Value": 70
        },
        {
          "Direction": "greater than",
          "Name": "upper warning",
          "Severity": 0,
          "Value": 60
        }
      ]
    },

    // ADC 電壓感測器
    {
      "Name": "P12V",
      "Type": "ADC",
      "Index": 0,
      "ScaleFactor": 6.8,
      "PowerState": "Always",
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
    },
    {
      "Name": "P3V3",
      "Type": "ADC",
      "Index": 1,
      "ScaleFactor": 2.0,
      "PowerState": "Always",
      "Thresholds": [
        {
          "Direction": "greater than",
          "Name": "upper critical",
          "Severity": 1,
          "Value": 3.63
        },
        {
          "Direction": "less than",
          "Name": "lower critical",
          "Severity": 1,
          "Value": 2.97
        }
      ]
    },

    // 關聯性端口
    {
      "Name": "ChassisPort",
      "Type": "Port",
      "PortType": "contained_by"
    }
  ]
}
```

---

## 機箱範例

### 基本機箱配置

```json
{
  "Name": "1U Chassis",
  "Type": "Chassis",
  "Probe": "xyz.openbmc_project.FruDevice({'PRODUCT_PRODUCT_NAME': '1U Server Chassis'})",
  "Exposes": [
    {
      "Name": "Chassis FRU",
      "Type": "EEPROM_24C02",
      "Bus": "$bus",
      "Address": "$address"
    },
    {
      "Name": "ChassisPort",
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

---

## 風扇配置範例

### Aspeed BMC 風扇

```json
{
  "Name": "System Fans",
  "Type": "Board",
  "Probe": "xyz.openbmc_project.FruDevice({'PRODUCT_PRODUCT_NAME': 'Baseboard'})",
  "Exposes": [
    {
      "Name": "Fan 1",
      "Type": "AspeedFan",
      "Connector": {
        "Name": "Fan Connector 1",
        "Pwm": 0,
        "Tachs": [0, 1]
      },
      "Thresholds": [
        {
          "Direction": "less than",
          "Name": "lower critical",
          "Severity": 1,
          "Value": 1000
        },
        {
          "Direction": "less than",
          "Name": "lower warning",
          "Severity": 0,
          "Value": 2000
        }
      ]
    },
    {
      "Name": "Fan 2",
      "Type": "AspeedFan",
      "Connector": {
        "Name": "Fan Connector 2",
        "Pwm": 1,
        "Tachs": [2, 3]
      },
      "Thresholds": [
        {
          "Direction": "less than",
          "Name": "lower critical",
          "Severity": 1,
          "Value": 1000
        },
        {
          "Direction": "less than",
          "Name": "lower warning",
          "Severity": 0,
          "Value": 2000
        }
      ]
    }
  ]
}
```

---

## 多型號支援範例

### 使用 OR Probe 支援多個型號

```json
{
  "Name": "$bus Riser Card",
  "Type": "Board",
  "Probe": [
    "xyz.openbmc_project.FruDevice({'PRODUCT_PRODUCT_NAME': 'Riser Card Rev A'})",
    "OR",
    "xyz.openbmc_project.FruDevice({'PRODUCT_PRODUCT_NAME': 'Riser Card Rev B'})",
    "OR",
    "xyz.openbmc_project.FruDevice({'PRODUCT_PRODUCT_NAME': 'Riser Card Rev C'})"
  ],
  "Exposes": [
    {
      "Name": "$bus Riser Temp",
      "Type": "TMP75",
      "Bus": "$bus",
      "Address": "0x48"
    }
  ]
}
```

> 💡 **建議**：使用明確的 `"OR"` 關鍵字分隔多個 Probe 條件，避免依賴陣列的隱含語義。詳見 [Probe 語法](ProbeSyntax.md) 中的說明。

---

## I2C Mux 後的裝置

### 透過 Mux 存取的感測器

當裝置在 I2C Mux 後時，`$bus` 會是 Mux 通道的匯流排編號：

```json
{
  "Name": "Slot $index Card",
  "Type": "Board",
  "Probe": "xyz.openbmc_project.FruDevice({'PRODUCT_PRODUCT_NAME': 'Expansion Card'})",
  "Exposes": [
    {
      "Name": "Slot $index Card Temp",
      "Type": "TMP75",
      "Bus": "$bus",
      "Address": "0x48"
    }
  ]
}
```

**裝置樹範例**：

```dts
&i2c1 {
    i2c-switch@71 {
        compatible = "nxp,pca9546";
        reg = <0x71>;

        i2c@0 {
            // 這裡的裝置會有新的匯流排編號
            // $bus 會對應此通道的編號
        };
    };
};
```

---

## 帶 PID 控制的配置

### 風扇 PID 控制

```json
{
  "Name": "Fan Control Zone 1",
  "Type": "Pid.Zone",
  "Probe": "xyz.openbmc_project.FruDevice({'PRODUCT_PRODUCT_NAME': 'Baseboard'})",
  "Exposes": [
    {
      "Name": "Zone 1",
      "Type": "Pid.Zone",
      "ZoneID": 1,
      "MinThermalOutput": 30,
      "FailsafePercent": 100
    },
    {
      "Name": "CPU Pid",
      "Type": "Pid",
      "Inputs": ["CPU Temperature"],
      "Outputs": ["Fan 1", "Fan 2"],
      "SetPoint": 70,
      "PCoefficient": 0.5,
      "ICoefficient": 0.1,
      "DCoefficient": 0.0,
      "OutLimitMax": 100,
      "OutLimitMin": 30,
      "SlewNeg": 10,
      "SlewPos": 10
    }
  ]
}
```

---

## 除錯用配置

### 使用 TRUE Probe 強制載入

僅用於開發和除錯：

```json
{
  "Name": "Debug Board",
  "Type": "Board",
  "Probe": "TRUE",
  "Exposes": [
    {
      "Name": "Debug Sensor",
      "Type": "TMP75",
      "Bus": 1,
      "Address": "0x48"
    }
  ]
}
```

> ⚠️ **警告**：不要在生產環境使用 `"Probe": "TRUE"`。

---

## 配置驗證

### 驗證 JSON 語法

```bash
# 使用 Python 驗證
python3 -m json.tool my_config.json > /dev/null && echo "Valid JSON"

# 使用 jq 驗證
jq . my_config.json > /dev/null && echo "Valid JSON"
```

### 驗證 D-Bus 發布結果

```bash
# 查看 Entity-Manager 物件樹
busctl tree xyz.openbmc_project.EntityManager

# 查看特定配置
busctl introspect xyz.openbmc_project.EntityManager \
    /xyz/openbmc_project/inventory/system/board/My_Board
```

---

## 下一步

- 了解 [FruDevice](FruDevice.md) 如何提供 Probe 所需的 FRU 資料
- 查看 [dbus-sensors 整合](DbusSensorsIntegration.md) 了解感測器如何使用配置
- 閱讀 [故障排除](Troubleshooting.md) 解決配置問題

---

> 📖 **更多範例**：[官方配置目錄](https://github.com/openbmc/entity-manager/tree/master/configurations)
