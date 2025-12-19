# 風扇感測器

## 簡介

FanSensor 守護程式處理風扇轉速讀取和 PWM 控制相關的感測器。支援多種風扇控制器類型。

---

## 守護程式資訊

| 項目 | 值 |
|------|-----|
| 執行檔 | `fansensor` |
| D-Bus 服務 | `xyz.openbmc_project.FanSensor` |
| 資料來源 | hwmon |

---

## 支援的風扇類型

| 類型 | 平台 | 說明 |
|------|------|------|
| AspeedFan | Aspeed AST2500/2600 | 內建 PWM 和轉速計 |
| NuvotonFan | Nuvoton | Nuvoton 風扇控制器 |
| I2CFan | I2C | I2C 介面風扇控制晶片 |
| HPEFan | HPE 平台 | HPE 特定風扇控制 |

---

## Entity-Manager 配置

### AspeedFan 配置

```json
{
    "Exposes": [
        {
            "Connector": {
                "Name": "System Fan 1",
                "Pwm": 0,
                "Tachs": [0, 1]
            },
            "Index": 0,
            "Name": "Fan1",
            "Type": "AspeedFan",
            "PowerState": "On"
        }
    ],
    "Name": "Baseboard Fans",
    "Probe": "TRUE",
    "Type": "Board"
}
```

### 配置欄位說明

| 欄位 | 必填 | 類型 | 說明 |
|------|------|------|------|
| `Name` | ✅ | string | 風扇名稱 |
| `Type` | ✅ | string | 風扇類型（AspeedFan、I2CFan 等） |
| `Index` | ❌ | number | 風扇索引 |
| `Connector` | ❌ | object | 連接器配置 |
| `Presence` | ❌ | object | 存在偵測配置 |
| `PowerState` | ❌ | string | 電源狀態條件 |
| `Thresholds` | ❌ | array | 閾值配置 |
| `MaxReading` | ❌ | number | 最大轉速值 |

---

## Connector 配置

Connector 物件定義 PWM 和轉速計的對應關係：

```json
{
    "Connector": {
        "Name": "System Fan 1",
        "Pwm": 0,
        "Tachs": [0, 1]
    }
}
```

| 欄位 | 說明 |
|------|------|
| `Name` | 連接器顯示名稱 |
| `Pwm` | PWM 通道索引 |
| `Tachs` | 轉速計通道索引陣列（雙轉子風扇有兩個） |

---

## 風扇存在偵測

使用 GPIO 偵測風扇是否插入：

```json
{
    "Presence": {
        "PinName": "FAN1_PRESENT",
        "Polarity": "Low",
        "MonitorType": "Polling"
    }
}
```

| 欄位 | 說明 |
|------|------|
| `PinName` | GPIO 名稱 |
| `Polarity` | 有效電位（High/Low） |
| `MonitorType` | 監控方式（Polling/Event） |

---

## I2CFan 配置

用於 I2C 介面的風扇控制晶片：

```json
{
    "Address": "0x2C",
    "Bus": 1,
    "Index": 0,
    "Name": "Chassis Fan",
    "Type": "I2CFan",
    "Connector": {
        "Name": "Chassis Fan 1",
        "Pwm": 0,
        "Tachs": [0]
    }
}
```

---

## NuvotonFan 配置

用於 Nuvoton BMC 的內建風扇控制器：

```json
{
    "Connector": {
        "Name": "System Fan 1",
        "Pwm": 0,
        "Tachs": [0]
    },
    "Index": 0,
    "Name": "SysFan1",
    "Type": "NuvotonFan",
    "MaxReading": 10000
}
```

> [!NOTE]
> NuvotonFan 類型必須指定 `Connector` 欄位。

---

## 完整配置範例

### 多風扇系統

```json
{
    "Exposes": [
        {
            "Connector": {
                "Name": "System Fan 1",
                "Pwm": 0,
                "Tachs": [0, 1]
            },
            "Index": 0,
            "Name": "Fan1",
            "Type": "AspeedFan",
            "PowerState": "On",
            "Presence": {
                "PinName": "FAN1_PRESENT",
                "Polarity": "Low"
            },
            "Thresholds": [
                {
                    "Direction": "less than",
                    "Name": "lower critical",
                    "Severity": 1,
                    "Value": 500
                },
                {
                    "Direction": "less than",
                    "Name": "lower warning",
                    "Severity": 0,
                    "Value": 1000
                }
            ]
        },
        {
            "Connector": {
                "Name": "System Fan 2",
                "Pwm": 1,
                "Tachs": [2, 3]
            },
            "Index": 1,
            "Name": "Fan2",
            "Type": "AspeedFan",
            "PowerState": "On"
        },
        {
            "Connector": {
                "Name": "System Fan 3",
                "Pwm": 2,
                "Tachs": [4, 5]
            },
            "Index": 2,
            "Name": "Fan3",
            "Type": "AspeedFan",
            "PowerState": "On"
        }
    ],
    "Name": "System Fans",
    "Probe": "TRUE",
    "Type": "Board"
}
```

---

## FanRedundancy 配置

用於設定風扇冗餘策略：

```json
{
    "AllowedFailures": 1,
    "Name": "Fan Redundancy",
    "Type": "FanRedundancy"
}
```

---

## hwmon 對應

Aspeed 風扇控制器的 hwmon 結構：

```
/sys/class/hwmon/hwmonX/
├── fan1_input    # 風扇 1 轉速 (RPM)
├── fan2_input    # 風扇 2 轉速
├── pwm1          # PWM 1 佔空比 (0-255)
├── pwm2          # PWM 2 佔空比
└── name          # aspeed_pwm_tacho
```

---

## D-Bus 輸出

```bash
$ busctl tree xyz.openbmc_project.FanSensor
└─/xyz
  └─/xyz/openbmc_project
    ├─/xyz/openbmc_project/sensors
    │ └─/xyz/openbmc_project/sensors/fan_tach
    │   ├─/xyz/openbmc_project/sensors/fan_tach/Fan1
    │   ├─/xyz/openbmc_project/sensors/fan_tach/Fan1_1
    │   └─/xyz/openbmc_project/sensors/fan_tach/Fan2
    └─/xyz/openbmc_project/control
      └─/xyz/openbmc_project/control/fanpwm
        ├─/xyz/openbmc_project/control/fanpwm/Pwm_0
        └─/xyz/openbmc_project/control/fanpwm/Pwm_1
```

---

## PWM 控制

PWM 控制物件提供風扇速度調整：

```bash
# 讀取 PWM 值
busctl get-property xyz.openbmc_project.FanSensor \
    /xyz/openbmc_project/control/fanpwm/Pwm_0 \
    xyz.openbmc_project.Control.FanPwm Target

# 設定 PWM 值（需要適當權限）
busctl set-property xyz.openbmc_project.FanSensor \
    /xyz/openbmc_project/control/fanpwm/Pwm_0 \
    xyz.openbmc_project.Control.FanPwm Target t 200
```

---

## 除錯指令

```bash
# 檢查風扇 hwmon
ls /sys/class/hwmon/*/fan*_input

# 讀取風扇轉速
cat /sys/class/hwmon/hwmonX/fan1_input

# 讀取 PWM 值
cat /sys/class/hwmon/hwmonX/pwm1

# 檢查 D-Bus 感測器
busctl introspect xyz.openbmc_project.FanSensor \
    /xyz/openbmc_project/sensors/fan_tach/Fan1

# 檢查風扇存在狀態
gpioget gpiochip0 FAN1_PRESENT
```

---

## 相關文件

- [感測器類型總覽](SensorTypes.md)
- [設定指南](ConfigurationGuide.md)
- [phosphor-pid-control](https://github.com/openbmc/phosphor-pid-control) - PID 風扇控制
