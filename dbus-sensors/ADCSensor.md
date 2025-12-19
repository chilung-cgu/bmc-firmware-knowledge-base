# ADC 感測器

## 簡介

ADC（Analog to Digital Converter）感測器用於讀取類比訊號並轉換為數位值。在 OpenBMC 中，ADC 感測器透過 Linux 核心的 IIO（Industrial I/O）子系統讀取。

---

## 守護程式資訊

| 項目 | 值 |
|------|-----|
| 執行檔 | `adcsensor` |
| D-Bus 服務 | `xyz.openbmc_project.ADCSensor` |
| 配置類型 | `ADC` |
| 資料來源 | hwmon (IIO) |

---

## 設備樹配置

以 Aspeed AST2600 為例，需要在設備樹中啟用 ADC：

```dts
iio-hwmon {
    compatible = "iio-hwmon";
    io-channels = <&adc0 0>, <&adc0 1>, <&adc0 2>;
};

&adc0 {
    status = "okay";
    vref = <2500>;  // 參考電壓 2.5V
};

&adc1 {
    status = "okay";
};
```

> [!NOTE]
> Aspeed SoC 通常有 `adc0` 和 `adc1` 兩組 ADC 控制器，預設為停用狀態。

---

## hwmon 對應

設備樹配置完成後，系統啟動時會在 `/sys/class/hwmon/` 建立對應的 hwmon 裝置：

```bash
# 範例：ADC hwmon 路徑
/sys/class/hwmon/hwmon5/
├── in0_input    # 通道 0 (毫伏)
├── in1_input    # 通道 1
├── in2_input    # 通道 2
└── name         # iio_hwmon
```

ADCSensor 守護程式會根據 Entity-Manager 配置中的 `Index` 欄位，對應到 `inX_input` 檔案。

---

## Entity-Manager 配置

### 基本配置

```json
{
    "Exposes": [
        {
            "Index": 0,
            "Name": "P12V",
            "Type": "ADC",
            "ScaleFactor": 1.0,
            "PowerState": "Always"
        }
    ],
    "Name": "Baseboard",
    "Probe": "TRUE",
    "Type": "Board"
}
```

### 配置欄位說明

| 欄位 | 必填 | 類型 | 說明 |
|------|------|------|------|
| `Name` | ✅ | string | 感測器名稱 |
| `Type` | ✅ | string | 必須為 `"ADC"` |
| `Index` | ✅ | number | ADC 通道索引（對應 inX_input） |
| `ScaleFactor` | ❌ | number | 縮放因子，預設 1.0 |
| `PowerState` | ❌ | string | 電源狀態條件 |
| `Thresholds` | ❌ | array | 閾值配置 |
| `BridgeGpio` | ❌ | array | 橋接 GPIO 配置 |
| `PollRate` | ❌ | number | 輪詢頻率（秒） |
| `MinValue` | ❌ | number | 最小值 |
| `MaxValue` | ❌ | number | 最大值 |

---

## 電壓分壓計算

對於電壓感測，通常需要使用分壓電阻。設 R1 為上臂電阻，R2 為下臂電阻：

```
Vin ──┬── R1 ──┬── ADC 輸入
      │        │
      └── R2 ──┴── GND
```

計算公式：
```
實際電壓 = ADC讀數 × (R1 + R2) / R2
```

使用 `ScaleFactor` 設定此縮放比例。

### 範例：12V 電壓監測

假設使用 10kΩ / 1kΩ 分壓電阻：

```json
{
    "Index": 0,
    "Name": "P12V",
    "Type": "ADC",
    "ScaleFactor": 11.0
}
```

---

## 完整配置範例

```json
{
    "Exposes": [
        {
            "Index": 0,
            "Name": "P12V_MAIN",
            "Type": "ADC",
            "ScaleFactor": 11.0,
            "PowerState": "Always",
            "Thresholds": [
                {
                    "Direction": "greater than",
                    "Name": "upper critical",
                    "Severity": 1,
                    "Value": 13.2
                },
                {
                    "Direction": "greater than",
                    "Name": "upper warning",
                    "Severity": 0,
                    "Value": 12.6
                },
                {
                    "Direction": "less than",
                    "Name": "lower warning",
                    "Severity": 0,
                    "Value": 11.4
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
            "Index": 1,
            "Name": "P3V3_STBY",
            "Type": "ADC",
            "ScaleFactor": 2.0,
            "PowerState": "Always"
        },
        {
            "Index": 2,
            "Name": "P5V_MAIN",
            "Type": "ADC",
            "ScaleFactor": 3.0,
            "PowerState": "On"
        }
    ],
    "Name": "Baseboard Voltages",
    "Probe": "TRUE",
    "Type": "Board"
}
```

---

## BridgeGpio 配置

某些 ADC 通道可能需要透過 GPIO 切換才能讀取：

```json
{
    "Index": 5,
    "Name": "DIMM_VPP",
    "Type": "ADC",
    "BridgeGpio": [
        {
            "Name": "ADC_MUX_SEL",
            "Polarity": "High",
            "SetupTime": 10
        }
    ]
}
```

---

## D-Bus 輸出

配置完成後，ADCSensor 會建立以下 D-Bus 物件：

```bash
$ busctl tree xyz.openbmc_project.ADCSensor
└─/xyz
  └─/xyz/openbmc_project
    └─/xyz/openbmc_project/sensors
      └─/xyz/openbmc_project/sensors/voltage
        ├─/xyz/openbmc_project/sensors/voltage/P12V_MAIN
        ├─/xyz/openbmc_project/sensors/voltage/P3V3_STBY
        └─/xyz/openbmc_project/sensors/voltage/P5V_MAIN
```

---

## 除錯指令

```bash
# 檢查 hwmon 路徑
ls /sys/class/hwmon/

# 讀取 ADC 原始值（毫伏）
cat /sys/class/hwmon/hwmonX/in0_input

# 檢查 D-Bus 感測器
busctl introspect xyz.openbmc_project.ADCSensor \
    /xyz/openbmc_project/sensors/voltage/P12V_MAIN

# 檢查 Entity-Manager 配置
busctl tree xyz.openbmc_project.EntityManager | grep ADC
```

---

## 相關文件

- [感測器類型總覽](SensorTypes.md)
- [設定指南](ConfigurationGuide.md)
- [閾值設定](ThresholdConfiguration.md)
- [Entity-Manager 整合](EntityManagerIntegration.md)
