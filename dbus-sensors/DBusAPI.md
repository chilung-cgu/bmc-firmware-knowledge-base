# D-Bus API 參考

## 簡介

dbus-sensors 透過 D-Bus 發布感測器數值，消費者（如 Redfish、IPMI、PID 控制）可透過這些介面讀取感測器資訊。

---

## 物件路徑結構

所有感測器遵循以下路徑結構：

```
/xyz/openbmc_project/sensors/<type>/<sensor_name>
```

| 類型 | 路徑 | 範例 |
|------|------|------|
| 溫度 | `/sensors/temperature/` | `/sensors/temperature/CPU0` |
| 電壓 | `/sensors/voltage/` | `/sensors/voltage/P12V` |
| 電流 | `/sensors/current/` | `/sensors/current/PSU1_IIN` |
| 功率 | `/sensors/power/` | `/sensors/power/Total_Power` |
| 風扇轉速 | `/sensors/fan_tach/` | `/sensors/fan_tach/Fan1` |
| 能量 | `/sensors/energy/` | `/sensors/energy/Total_Energy` |
| 使用率 | `/sensors/utilization/` | `/sensors/utilization/CPU_Util` |
| 氣流 | `/sensors/airflow/` | `/sensors/airflow/Exhaust_CFM` |
| 壓力 | `/sensors/pressure/` | `/sensors/pressure/Ambient` |
| 高度 | `/sensors/altitude/` | `/sensors/altitude/Altitude` |
| 頻率 | `/sensors/frequency/` | `/sensors/frequency/Clock` |
| 濕度 | `/sensors/humidity/` | `/sensors/humidity/Ambient_RH` |
| 液流 | `/sensors/liquidflow/` | `/sensors/liquidflow/Coolant` |

---

## D-Bus 介面

### xyz.openbmc_project.Sensor.Value

基本感測器數值介面。

**屬性：**

| 屬性 | 類型 | 說明 |
|------|------|------|
| `Value` | double | 目前感測器讀數 |
| `MaxValue` | double | 最大有效值 |
| `MinValue` | double | 最小有效值 |
| `Unit` | enum | 單位類型 |

**範例：**

```bash
$ busctl introspect xyz.openbmc_project.HwmonTempSensor \
    /xyz/openbmc_project/sensors/temperature/CPU0

xyz.openbmc_project.Sensor.Value         interface
.MaxValue                                property  d  127
.MinValue                                property  d  -128
.Value                                   property  d  42.5
```

---

### xyz.openbmc_project.Sensor.Threshold.Warning

警告閾值介面。

**屬性：**

| 屬性 | 類型 | 說明 |
|------|------|------|
| `WarningHigh` | double | 上限警告閾值（NaN 表示未設定） |
| `WarningLow` | double | 下限警告閾值 |
| `WarningAlarmHigh` | boolean | 是否觸發上限警告 |
| `WarningAlarmLow` | boolean | 是否觸發下限警告 |

**訊號：**

| 訊號 | 說明 |
|------|------|
| `WarningHighAlarmAsserted` | 上限警告觸發 |
| `WarningHighAlarmDeasserted` | 上限警告解除 |
| `WarningLowAlarmAsserted` | 下限警告觸發 |
| `WarningLowAlarmDeasserted` | 下限警告解除 |

---

### xyz.openbmc_project.Sensor.Threshold.Critical

嚴重閾值介面。

**屬性：**

| 屬性 | 類型 | 說明 |
|------|------|------|
| `CriticalHigh` | double | 上限嚴重閾值 |
| `CriticalLow` | double | 下限嚴重閾值 |
| `CriticalAlarmHigh` | boolean | 是否觸發上限嚴重警報 |
| `CriticalAlarmLow` | boolean | 是否觸發下限嚴重警報 |

**訊號：**

| 訊號 | 說明 |
|------|------|
| `CriticalHighAlarmAsserted` | 上限嚴重警報觸發 |
| `CriticalHighAlarmDeasserted` | 上限嚴重警報解除 |
| `CriticalLowAlarmAsserted` | 下限嚴重警報觸發 |
| `CriticalLowAlarmDeasserted` | 下限嚴重警報解除 |

---

### xyz.openbmc_project.State.Decorator.Availability

可用性狀態介面。

**屬性：**

| 屬性 | 類型 | 說明 |
|------|------|------|
| `Available` | boolean | 感測器是否可用 |

---

### xyz.openbmc_project.State.Decorator.OperationalStatus

操作狀態介面。

**屬性：**

| 屬性 | 類型 | 說明 |
|------|------|------|
| `Functional` | boolean | 感測器是否正常運作（false 表示故障） |

---

### xyz.openbmc_project.Association.Definitions

關聯定義介面，建立感測器與庫存項目的關聯。

**屬性：**

| 屬性 | 類型 | 說明 |
|------|------|------|
| `Associations` | array(sss) | 關聯三元組陣列 |

**關聯格式：**

```
(forward_type, reverse_type, endpoint_path)
```

**範例：**

```bash
$ busctl get-property xyz.openbmc_project.HwmonTempSensor \
    /xyz/openbmc_project/sensors/temperature/CPU0 \
    xyz.openbmc_project.Association.Definitions Associations

a(sss) 1 "chassis" "all_sensors" "/xyz/openbmc_project/inventory/system/board"
```

---

## 介面組合

典型的 dbus-sensors 物件支援的介面組合：

```
/xyz/openbmc_project/sensors/temperature/CPU0

├── xyz.openbmc_project.Sensor.Value
├── xyz.openbmc_project.Sensor.Threshold.Warning
├── xyz.openbmc_project.Sensor.Threshold.Critical
├── xyz.openbmc_project.State.Decorator.Availability
├── xyz.openbmc_project.State.Decorator.OperationalStatus
├── xyz.openbmc_project.Association.Definitions
├── org.freedesktop.DBus.Properties
├── org.freedesktop.DBus.Introspectable
└── org.freedesktop.DBus.Peer
```

---

## ObjectManager

每個 dbus-sensors 服務在 `/xyz/openbmc_project/sensors` 實作 `org.freedesktop.DBus.ObjectManager`：

```bash
# 取得所有感測器
busctl call xyz.openbmc_project.HwmonTempSensor \
    /xyz/openbmc_project/sensors \
    org.freedesktop.DBus.ObjectManager GetManagedObjects
```

---

## 數值更新訊號

當感測器數值變更時，發送 `PropertiesChanged` 訊號：

```bash
# 監聽數值變更
busctl monitor xyz.openbmc_project.HwmonTempSensor \
    --match "type='signal',interface='org.freedesktop.DBus.Properties'"
```

---

## NaN 值處理

dbus-sensors 使用 IEEE 754 NaN 表示無效或不可用的數值：

- **Value = NaN**：感測器讀取失敗或數值無效
- **WarningHigh = NaN**：未設定上限警告閾值
- **CriticalLow = NaN**：未設定下限嚴重閾值

消費者應檢查 NaN：

```python
import math

if math.isnan(value):
    print("Sensor value unavailable")
```

---

## D-Bus 查詢範例

### 讀取感測器數值

```bash
busctl get-property xyz.openbmc_project.HwmonTempSensor \
    /xyz/openbmc_project/sensors/temperature/CPU0 \
    xyz.openbmc_project.Sensor.Value Value
```

### 檢查閾值狀態

```bash
busctl get-property xyz.openbmc_project.HwmonTempSensor \
    /xyz/openbmc_project/sensors/temperature/CPU0 \
    xyz.openbmc_project.Sensor.Threshold.Warning WarningAlarmHigh
```

### 列出所有感測器

```bash
busctl tree xyz.openbmc_project.HwmonTempSensor
```

### 完整內省

```bash
busctl introspect xyz.openbmc_project.HwmonTempSensor \
    /xyz/openbmc_project/sensors/temperature/CPU0
```

---

## 相關文件

- [設定指南](ConfigurationGuide.md)
- [閾值設定](ThresholdConfiguration.md)
- [phosphor-dbus-interfaces](https://github.com/openbmc/phosphor-dbus-interfaces/tree/master/yaml/xyz/openbmc_project/Sensor)
