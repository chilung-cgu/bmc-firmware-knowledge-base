# PSU 感測器

## 簡介

PSUSensor 守護程式處理電源供應器相關的感測器，主要透過 PMBus 協定與裝置通訊。支援電壓、電流、功率和溫度監測。

---

## 守護程式資訊

| 項目 | 值 |
|------|-----|
| 執行檔 | `psusensor` |
| D-Bus 服務 | `xyz.openbmc_project.PSUSensor` |
| 資料來源 | hwmon (PMBus) |

---

## 支援的裝置類型

### 電流/電壓監控 IC

| 類型 | 說明 | 常用場景 |
|------|------|----------|
| INA219 | 雙向電流/功率監控 | 電源軌監測 |
| INA230 | 高精度電流監控 | 伺服器電源 |
| INA233 | PMBus 電流監控 | 機架電源 |
| INA238 | 85V 電力監控 | 高壓監測 |
| LTC2945 | 寬範圍電力監控 | 資料中心 |

### 熱插拔控制器

| 類型 | 說明 |
|------|------|
| ADM1272 | 熱插拔控制器 + 數位功率監控 |
| ADM1275 | 熱插拔控制器 |
| ADM1278 | 熱插拔控制器 + PMBus |
| ADM1281 | 熱插拔控制器 |
| LM25066 | 系統功率管理 |

### 多相電壓調節器

| 類型 | 說明 |
|------|------|
| TPS53679 | 雙相控制器 |
| XDPE12284 | Infineon 多相控制器 |
| RAA228228 | Renesas 電壓調節器 |
| ISL68220 | Intersil 多相調節器 |
| MP2971 | MPS 多相控制器 |

### 電源供應器

| 類型 | 說明 |
|------|------|
| pmbus | 通用 PMBus 電源 |
| cffps | IBM Common Form Factor PSU |
| DPS800 | Delta 800W 電源 |
| IPSPS1 | Intel 電源 |

---

## Entity-Manager 配置

### 基本配置

```json
{
    "Exposes": [
        {
            "Address": "0x40",
            "Bus": 5,
            "Name": "PSU1",
            "Type": "pmbus"
        }
    ],
    "Name": "Power Supply",
    "Probe": "TRUE",
    "Type": "Board"
}
```

### 配置欄位說明

| 欄位 | 必填 | 類型 | 說明 |
|------|------|------|------|
| `Name` | ✅ | string | 感測器基本名稱 |
| `Type` | ✅ | string | 裝置類型 |
| `Bus` | ❌ | number | I2C 匯流排（部分類型需要） |
| `Address` | ❌ | string | I2C 位址 |
| `Labels` | ❌ | array | 指定要讀取的 hwmon 標籤 |
| `CurrScaleFactor` | ❌ | number | 電流縮放因子 |
| `PowerScaleFactor` | ❌ | number | 功率縮放因子 |
| `InScaleFactor` | ❌ | number | 電壓縮放因子 |

---

## 動態命名

PSUSensor 支援使用 hwmon 標籤命名各個感測器通道：

| hwmon 檔案 | 配置欄位 | D-Bus 路徑 |
|------------|----------|------------|
| `vin` | `vin_Name` | `/sensors/voltage/<name>` |
| `vout1` | `vout1_Name` | `/sensors/voltage/<name>` |
| `iin` | `iin_Name` | `/sensors/current/<name>` |
| `iout1` | `iout1_Name` | `/sensors/current/<name>` |
| `pin` | `pin_Name` | `/sensors/power/<name>` |
| `pout1` | `pout1_Name` | `/sensors/power/<name>` |
| `temp1` | `temp1_Name` | `/sensors/temperature/<name>` |

### 範例：指定感測器名稱

```json
{
    "Address": "0x58",
    "Bus": 5,
    "Name": "PSU1",
    "Type": "ADM1275",
    "vin_Name": "PSU1 Input Voltage",
    "iin_Name": "PSU1 Input Current",
    "pin_Name": "PSU1 Input Power"
}
```

---

## 縮放配置

### 電壓/電流/功率縮放

某些裝置需要額外的縮放因子：

```json
{
    "Address": "0x40",
    "Bus": 3,
    "Name": "VR1",
    "Type": "RAA228228",
    "CurrScaleFactor": 1.5,
    "PowerScaleFactor": 1.5,
    "iout1_Scale": 2.0,
    "pout1_Scale": 2.0
}
```

### 通道特定縮放

| 配置欄位 | 說明 |
|----------|------|
| `iout1_Scale` | 輸出電流 1 縮放 |
| `curr1_Scale` | 電流 1 縮放 |
| `power1_Scale` | 功率 1 縮放 |

---

## 完整配置範例

### 電源供應器監測

```json
{
    "Exposes": [
        {
            "Address": "0x58",
            "Bus": 5,
            "Name": "PSU1",
            "Type": "pmbus",
            "Labels": ["vin", "vout1", "iin", "iout1", "pin", "pout1", "temp1", "temp2"],
            "vin_Name": "PSU1_VIN",
            "vout1_Name": "PSU1_VOUT",
            "iin_Name": "PSU1_IIN",
            "iout1_Name": "PSU1_IOUT",
            "pin_Name": "PSU1_PIN",
            "pout1_Name": "PSU1_POUT",
            "temp1_Name": "PSU1_Temp1",
            "temp2_Name": "PSU1_Temp2",
            "PowerState": "On",
            "Thresholds": [
                {
                    "Direction": "greater than",
                    "Label": "pout1",
                    "Name": "upper critical",
                    "Severity": 1,
                    "Value": 750
                }
            ]
        }
    ],
    "Name": "PSU1 Board",
    "Probe": "xyz.openbmc_project.FruDevice({'PRODUCT_PRODUCT_NAME': 'PSU1'})",
    "Type": "Board"
}
```

### 電壓調節器 (VR)

```json
{
    "Exposes": [
        {
            "Address": "0x60",
            "Bus": 3,
            "Name": "VR_VCCIN",
            "Type": "XDPE12284",
            "vout1_Name": "CPU0 VCCIN",
            "iout1_Name": "CPU0 VCCIN Current",
            "temp1_Name": "CPU0 VR Temp",
            "PowerState": "On"
        }
    ],
    "Name": "CPU0 VR",
    "Probe": "TRUE",
    "Type": "Board"
}
```

---

## 閾值配置

PSUSensor 的閾值可以針對特定 Label 設定：

```json
{
    "Thresholds": [
        {
            "Direction": "greater than",
            "Label": "iout1",
            "Name": "upper critical",
            "Severity": 1,
            "Value": 100
        },
        {
            "Direction": "greater than",
            "Label": "temp1",
            "Name": "upper critical",
            "Severity": 1,
            "Value": 85
        }
    ]
}
```

---

## D-Bus 輸出

```bash
$ busctl tree xyz.openbmc_project.PSUSensor
└─/xyz
  └─/xyz/openbmc_project
    └─/xyz/openbmc_project/sensors
      ├─/xyz/openbmc_project/sensors/voltage
      │ ├─/xyz/openbmc_project/sensors/voltage/PSU1_VIN
      │ └─/xyz/openbmc_project/sensors/voltage/PSU1_VOUT
      ├─/xyz/openbmc_project/sensors/current
      │ ├─/xyz/openbmc_project/sensors/current/PSU1_IIN
      │ └─/xyz/openbmc_project/sensors/current/PSU1_IOUT
      ├─/xyz/openbmc_project/sensors/power
      │ ├─/xyz/openbmc_project/sensors/power/PSU1_PIN
      │ └─/xyz/openbmc_project/sensors/power/PSU1_POUT
      └─/xyz/openbmc_project/sensors/temperature
        └─/xyz/openbmc_project/sensors/temperature/PSU1_Temp1
```

---

## 除錯指令

```bash
# 檢查 PMBus 裝置
ls /sys/class/hwmon/ | xargs -I{} cat /sys/class/hwmon/{}/name

# 列出電源相關的 hwmon 檔案
ls /sys/class/hwmon/hwmonX/

# 讀取輸入功率
cat /sys/class/hwmon/hwmonX/power1_input

# 查詢 D-Bus 功率感測器
busctl introspect xyz.openbmc_project.PSUSensor \
    /xyz/openbmc_project/sensors/power/PSU1_PIN
```

---

## 相關文件

- [感測器類型總覽](SensorTypes.md)
- [設定指南](ConfigurationGuide.md)
- [閾值設定](ThresholdConfiguration.md)
