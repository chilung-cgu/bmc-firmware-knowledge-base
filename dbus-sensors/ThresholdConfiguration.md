# 閾值設定

## 簡介

dbus-sensors 支援為感測器設定警告（Warning）和嚴重（Critical）閾值，當數值超過閾值時會觸發警報並發送 D-Bus 訊號。

---

## 閾值類型

| 類型 | Severity | 說明 |
|------|----------|------|
| 警告（Warning） | 0 | 提前警示，需要注意 |
| 嚴重（Critical） | 1 | 需要立即處理 |

---

## 閾值方向

| Direction | 說明 | 使用場景 |
|-----------|------|----------|
| `"greater than"` | 數值超過閾值時觸發 | 溫度過高、電壓過高 |
| `"less than"` | 數值低於閾值時觸發 | 電壓過低、風扇轉速過低 |

---

## JSON 配置格式

```json
{
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

---

## 閾值欄位說明

| 欄位 | 必填 | 類型 | 說明 |
|------|------|------|------|
| `Direction` | ✅ | string | 觸發方向 |
| `Name` | ✅ | string | 閾值名稱（唯一識別） |
| `Severity` | ✅ | number | 嚴重程度（0=警告, 1=嚴重） |
| `Value` | ✅ | number | 閾值數值 |
| `Hysteresis` | ❌ | number | 遲滯值 |
| `Label` | ❌ | string | PSU 感測器的標籤名稱 |

---

## Name 慣例

慣用的 Name 值：

| Name | Direction | Severity | D-Bus 對應 |
|------|-----------|----------|------------|
| `"upper critical"` | greater than | 1 | CriticalHigh |
| `"upper warning"` | greater than | 0 | WarningHigh |
| `"lower warning"` | less than | 0 | WarningLow |
| `"lower critical"` | less than | 1 | CriticalLow |

---

## Hysteresis 遲滯

遲滯用於避免數值在閾值附近反覆觸發/解除警報：

```json
{
    "Direction": "greater than",
    "Hysteresis": 2,
    "Name": "upper warning",
    "Severity": 0,
    "Value": 85
}
```

行為說明：
- 當溫度 > 85°C 時，觸發警告
- 溫度需降至 < 83°C（85 - 2）才解除警告

---

## D-Bus 閾值對應

JSON 配置會轉換為 D-Bus 閾值屬性：

| JSON | D-Bus 屬性 |
|------|------------|
| Severity=0, Direction="greater than" | WarningHigh |
| Severity=0, Direction="less than" | WarningLow |
| Severity=1, Direction="greater than" | CriticalHigh |
| Severity=1, Direction="less than" | CriticalLow |

---

## 完整配置範例

### 溫度感測器

```json
{
    "Address": "0x48",
    "Bus": 1,
    "Name": "CPU Temp",
    "Type": "TMP75",
    "Thresholds": [
        {
            "Direction": "greater than",
            "Hysteresis": 2,
            "Name": "upper critical",
            "Severity": 1,
            "Value": 95
        },
        {
            "Direction": "greater than",
            "Hysteresis": 2,
            "Name": "upper warning",
            "Severity": 0,
            "Value": 85
        }
    ]
}
```

### 電壓感測器

```json
{
    "Index": 0,
    "Name": "P12V",
    "ScaleFactor": 11.0,
    "Type": "ADC",
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
}
```

### 風扇轉速

```json
{
    "Name": "Fan1",
    "Type": "AspeedFan",
    "Connector": {
        "Name": "System Fan 1",
        "Pwm": 0,
        "Tachs": [0]
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
}
```

---

## PSU Label 閾值

對於 PSUSensor，使用 `Label` 指定閾值適用的通道：

```json
{
    "Name": "PSU1",
    "Type": "pmbus",
    "Thresholds": [
        {
            "Direction": "greater than",
            "Label": "temp1",
            "Name": "upper critical",
            "Severity": 1,
            "Value": 85
        },
        {
            "Direction": "greater than",
            "Label": "pout1",
            "Name": "upper warning",
            "Severity": 0,
            "Value": 700
        }
    ]
}
```

---

## D-Bus 閾值查詢

```bash
# 查看閾值設定
busctl introspect xyz.openbmc_project.HwmonTempSensor \
    /xyz/openbmc_project/sensors/temperature/CPU_Temp

# 輸出範例
xyz.openbmc_project.Sensor.Threshold.Warning  interface
.WarningHigh                                  property  d  85
.WarningLow                                   property  d  nan
.WarningAlarmHigh                             property  b  false
.WarningAlarmLow                              property  b  false

xyz.openbmc_project.Sensor.Threshold.Critical interface
.CriticalHigh                                 property  d  95
.CriticalLow                                  property  d  nan
.CriticalAlarmHigh                            property  b  false
.CriticalAlarmLow                             property  b  false
```

---

## 閾值警報訊號

當警報狀態變更時，會發送 D-Bus 訊號：

| 訊號 | 說明 |
|------|------|
| `WarningHighAlarmAsserted` | 上限警告觸發 |
| `WarningHighAlarmDeasserted` | 上限警告解除 |
| `CriticalHighAlarmAsserted` | 上限嚴重觸發 |
| `CriticalLowAlarmAsserted` | 下限嚴重觸發 |

---

## 相關文件

- [設定指南](ConfigurationGuide.md)
- [D-Bus API](DBusAPI.md)
- [感測器類型總覽](SensorTypes.md)
