# CPU 感測器

## 簡介

CPUSensor 守護程式透過 Intel PECI（Platform Environment Control Interface）介面讀取 CPU 溫度和其他處理器相關的感測器數值。

---

## 守護程式資訊

| 項目 | 值 |
|------|-----|
| 執行檔 | `cpusensor` |
| D-Bus 服務 | `xyz.openbmc_project.CPUSensor` |
| 配置類型 | XeonCPU |
| 資料來源 | PECI |

---

## PECI 簡介

PECI 是 Intel 開發的單線介面，用於：
- 讀取 CPU 核心溫度
- 讀取 DIMM 溫度
- 存取 CPU 內部暫存器
- CPU 電源管理

---

## Entity-Manager 配置

### 基本配置

```json
{
    "Exposes": [
        {
            "Address": "0x30",
            "Bus": 0,
            "CpuID": 0,
            "Name": "CPU0",
            "Type": "XeonCPU"
        }
    ],
    "Name": "CPU Configuration",
    "Probe": "TRUE",
    "Type": "Board"
}
```

### 配置欄位說明

| 欄位 | 必填 | 類型 | 說明 |
|------|------|------|------|
| `Name` | ✅ | string | CPU 名稱 |
| `Type` | ✅ | string | 必須為 `"XeonCPU"` |
| `Address` | ✅ | string | PECI 位址（通常 0x30-0x37） |
| `Bus` | ❌ | number | PECI 匯流排 |
| `CpuID` | ✅ | number | CPU 識別碼（0, 1, 2...） |
| `DtsCritOffset` | ❌ | number | DTS 嚴重閾值偏移量 |
| `PresenceGpio` | ❌ | number/array | CPU 存在偵測 GPIO |
| `PowerState` | ❌ | string | 電源狀態條件 |
| `Thresholds` | ❌ | array | 閾值配置 |

---

## CPU 存在偵測

使用 GPIO 偵測 CPU 是否安裝：

```json
{
    "Address": "0x30",
    "CpuID": 0,
    "Name": "CPU0",
    "Type": "XeonCPU",
    "PresenceGpio": [10]
}
```

也可以指定多個 GPIO：

```json
{
    "PresenceGpio": [10, 11]
}
```

---

## DtsCritOffset 配置

DTS（Digital Thermal Sensor）提供 CPU 核心溫度。`DtsCritOffset` 用於計算嚴重閾值：

```
嚴重閾值 = Tjmax - DtsCritOffset
```

```json
{
    "Address": "0x30",
    "CpuID": 0,
    "Name": "CPU0",
    "Type": "XeonCPU",
    "DtsCritOffset": 5
}
```

若 Tjmax = 100°C 且 DtsCritOffset = 5，則嚴重閾值為 95°C。

---

## 完整配置範例

### 雙路 CPU 系統

```json
{
    "Exposes": [
        {
            "Address": "0x30",
            "Bus": 0,
            "CpuID": 0,
            "Name": "CPU0",
            "Type": "XeonCPU",
            "DtsCritOffset": 5,
            "PresenceGpio": [10],
            "PowerState": "On",
            "Thresholds": [
                {
                    "Direction": "greater than",
                    "Name": "upper warning",
                    "Severity": 0,
                    "Value": 85
                }
            ]
        },
        {
            "Address": "0x31",
            "Bus": 0,
            "CpuID": 1,
            "Name": "CPU1",
            "Type": "XeonCPU",
            "DtsCritOffset": 5,
            "PresenceGpio": [11],
            "PowerState": "On"
        }
    ],
    "Name": "CPU Board",
    "Probe": "TRUE",
    "Type": "Board"
}
```

---

## 自動發現的感測器

CPUSensor 會自動發現並建立以下感測器：

| 感測器類型 | D-Bus 路徑 | 說明 |
|------------|------------|------|
| CPU Package 溫度 | `/sensors/temperature/CPU0` | 封裝溫度 |
| Core 溫度 | `/sensors/temperature/CPU0_Core0` | 各核心溫度 |
| DIMM 溫度 | `/sensors/temperature/DIMM0` | 記憶體溫度 |
| CPU 功耗 | `/sensors/power/CPU0_Power` | CPU 功率消耗 |

---

## D-Bus 輸出

```bash
$ busctl tree xyz.openbmc_project.CPUSensor
└─/xyz
  └─/xyz/openbmc_project
    └─/xyz/openbmc_project/sensors
      ├─/xyz/openbmc_project/sensors/temperature
      │ ├─/xyz/openbmc_project/sensors/temperature/CPU0
      │ ├─/xyz/openbmc_project/sensors/temperature/CPU0_Core0
      │ ├─/xyz/openbmc_project/sensors/temperature/CPU0_Core1
      │ ├─/xyz/openbmc_project/sensors/temperature/DIMM_A0
      │ └─/xyz/openbmc_project/sensors/temperature/DIMM_B0
      └─/xyz/openbmc_project/sensors/power
        └─/xyz/openbmc_project/sensors/power/CPU0_Power
```

---

## PECI 位址分配

標準 Intel 伺服器 PECI 位址：

| CPU | PECI 位址 |
|-----|-----------|
| CPU0 | 0x30 |
| CPU1 | 0x31 |
| CPU2 | 0x32 |
| CPU3 | 0x33 |

---

## 設備樹要求

需要在設備樹中啟用 PECI 控制器：

```dts
&peci0 {
    status = "okay";
    gpios = <&gpio0 ASPEED_GPIO(F, 6) 0>;
};
```

---

## 除錯指令

```bash
# 檢查 PECI 裝置
ls /sys/bus/peci/devices/

# 讀取 CPU 溫度（需要 peci-tools）
peci_cmds -a 0x30 RdPkgConfig 0 2

# 檢查 D-Bus 感測器
busctl introspect xyz.openbmc_project.CPUSensor \
    /xyz/openbmc_project/sensors/temperature/CPU0

# 查看 CPU 存在 GPIO
gpioget gpiochip0 10
```

---

## 常見問題

### CPU 感測器未出現

1. 確認 CPU 已安裝且系統開機
2. 檢查 `PowerState` 是否為 `"On"`
3. 驗證 `PresenceGpio` 狀態
4. 確認 PECI 位址正確

### 溫度讀數異常

1. 檢查 PECI 線路連接
2. 確認設備樹中 PECI GPIO 配置正確
3. 嘗試使用 peci_cmds 工具直接讀取

---

## 相關文件

- [感測器類型總覽](SensorTypes.md)
- [設定指南](ConfigurationGuide.md)
- [peci-pcie](https://github.com/openbmc/peci-pcie) - PCIe 裝置偵測
