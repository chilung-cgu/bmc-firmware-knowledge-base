# Sensor Interfaces - 感測器介面

本文件說明 `xyz.openbmc_project.Sensor` 命名空間下的感測器相關介面。

---

## 📋 概述

感測器介面用於在 D-Bus 上公開硬體感測器讀數，包括溫度、電壓、電流、風扇轉速等各類感測資料。這些介面主要由 [dbus-sensors](../dbus-sensors/Home.md) 專案實作。

### 介面總覽

| 介面 | 說明 |
|------|------|
| `xyz.openbmc_project.Sensor.Value` | 感測器讀數值 |
| `xyz.openbmc_project.Sensor.Threshold.Warning` | 警告閾值 |
| `xyz.openbmc_project.Sensor.Threshold.Critical` | 嚴重閾值 |
| `xyz.openbmc_project.Sensor.Threshold.PerformanceLoss` | 效能損失閾值 |
| `xyz.openbmc_project.Sensor.Threshold.SoftShutdown` | 軟關機閾值 |
| `xyz.openbmc_project.Sensor.Threshold.HardShutdown` | 硬關機閾值 |
| `xyz.openbmc_project.Sensor.Accuracy` | 感測器精度 |
| `xyz.openbmc_project.Sensor.ValueMutability` | 值可變性 |

---

## 📍 物件路徑結構

感測器物件必須位於 `/xyz/openbmc_project/sensors` 路徑下的正確層級：

```
/xyz/openbmc_project/sensors/
├── airflow/          # 氣流感測器 (CFM)
├── altitude/         # 高度感測器 (公尺)
├── current/          # 電流感測器 (安培)
├── energy/           # 能量感測器 (焦耳)
├── fan_tach/         # 風扇轉速感測器 (RPM)
├── frequency/        # 頻率感測器 (赫茲)
├── humidity/         # 濕度感測器 (% RH)
├── liquidflow/       # 液體流量感測器 (LPM)
├── power/            # 功率感測器 (瓦特)
├── pressure/         # 壓力感測器 (帕斯卡)
├── temperature/      # 溫度感測器 (攝氏)
├── utilization/      # 使用率感測器 (%)
├── valve/            # 閥門感測器 (%)
└── voltage/          # 電壓感測器 (伏特)
```

### 路徑範例

| 感測器 | D-Bus 路徑 |
|--------|------------|
| CPU 核心溫度 | `/xyz/openbmc_project/sensors/temperature/CPU_Core` |
| 系統風扇 0 | `/xyz/openbmc_project/sensors/fan_tach/Fan0` |
| 12V 電源軌 | `/xyz/openbmc_project/sensors/voltage/P12V` |

---

## 📊 xyz.openbmc_project.Sensor.Value

這是所有感測器的核心介面，提供感測器讀數。

### 屬性

| 屬性 | 型別 | 說明 |
|------|------|------|
| `Value` | `double` | 目前感測器讀數 |
| `MaxValue` | `double` | 最大支援讀數（預設：infinity） |
| `MinValue` | `double` | 最小支援讀數（預設：-infinity） |
| `Unit` | `enum[Unit]` | 讀數單位 |

### Unit 列舉

| 值 | 說明 | 對應路徑 |
|----|------|----------|
| `Amperes` | 安培（電流） | `current/` |
| `CFM` | 立方英尺/分鐘（氣流） | `airflow/` |
| `DegreesC` | 攝氏度（溫度） | `temperature/` |
| `Hertz` | 赫茲（頻率） | `frequency/` |
| `Joules` | 焦耳（能量） | `energy/` |
| `LPM` | 公升/分鐘（液體流量） | `liquidflow/` |
| `Meters` | 公尺（高度） | `altitude/` |
| `Percent` | 百分比（使用率/閥門） | `utilization/`, `valve/` |
| `PercentRH` | 相對濕度百分比 | `humidity/` |
| `Pascals` | 帕斯卡（壓力） | `pressure/` |
| `RPMS` | 每分鐘轉數（風扇） | `fan_tach/` |
| `Volts` | 伏特（電壓） | `voltage/` |
| `Watts` | 瓦特（功率） | `power/` |

### 關聯

| 關聯名稱 | 反向名稱 | 說明 |
|----------|----------|------|
| `inventory` | `sensors` | 連結到相關的硬體清單項目 |
| `monitoring` | `monitored_by` | 連結到被監控的 BMC 項目 |

### 使用範例

```bash
# 讀取溫度感測器值
busctl get-property xyz.openbmc_project.HwmonTempSensor \
    /xyz/openbmc_project/sensors/temperature/CPU_Core \
    xyz.openbmc_project.Sensor.Value Value

# 輸出：d 45.5
```

---

## ⚠️ xyz.openbmc_project.Sensor.Threshold.Warning

警告等級閾值介面，當感測器值超出範圍時觸發警告。

### 屬性

| 屬性 | 型別 | 說明 |
|------|------|------|
| `WarningHigh` | `double` | 警告上限（NaN 表示無此閾值） |
| `WarningLow` | `double` | 警告下限（NaN 表示無此閾值） |
| `WarningAlarmHigh` | `boolean` | 是否超出上限 |
| `WarningAlarmLow` | `boolean` | 是否低於下限 |

### 訊號

| 訊號 | 說明 |
|------|------|
| `WarningHighAlarmAsserted` | 上限警報觸發 |
| `WarningHighAlarmDeasserted` | 上限警報解除 |
| `WarningLowAlarmAsserted` | 下限警報觸發 |
| `WarningLowAlarmDeasserted` | 下限警報解除 |

每個訊號攜帶一個 `SensorValue` 屬性（`double`），表示觸發警報變更的感測器值。

### 使用範例

```bash
# 讀取警告閾值
busctl get-property xyz.openbmc_project.HwmonTempSensor \
    /xyz/openbmc_project/sensors/temperature/CPU_Core \
    xyz.openbmc_project.Sensor.Threshold.Warning WarningHigh

# 檢查是否處於警告狀態
busctl get-property xyz.openbmc_project.HwmonTempSensor \
    /xyz/openbmc_project/sensors/temperature/CPU_Core \
    xyz.openbmc_project.Sensor.Threshold.Warning WarningAlarmHigh
```

---

## 🚨 xyz.openbmc_project.Sensor.Threshold.Critical

嚴重等級閾值介面，當感測器值超出嚴重範圍時觸發。

### 屬性

| 屬性 | 型別 | 說明 |
|------|------|------|
| `CriticalHigh` | `double` | 嚴重上限（NaN 表示無此閾值） |
| `CriticalLow` | `double` | 嚴重下限（NaN 表示無此閾值） |
| `CriticalAlarmHigh` | `boolean` | 是否超出上限 |
| `CriticalAlarmLow` | `boolean` | 是否低於下限 |

### 訊號

| 訊號 | 說明 |
|------|------|
| `CriticalHighAlarmAsserted` | 上限嚴重警報觸發 |
| `CriticalHighAlarmDeasserted` | 上限嚴重警報解除 |
| `CriticalLowAlarmAsserted` | 下限嚴重警報觸發 |
| `CriticalLowAlarmDeasserted` | 下限嚴重警報解除 |

---

## 🎯 xyz.openbmc_project.Sensor.Accuracy

感測器精度資訊介面。

### 屬性

| 屬性 | 型別 | 說明 |
|------|------|------|
| `Accuracy` | `double` | 精度值（百分比） |

---

## 🔄 xyz.openbmc_project.Sensor.ValueMutability

指示感測器值是否可被外部修改。

### 屬性

| 屬性 | 型別 | 說明 |
|------|------|------|
| `Mutable` | `boolean` | 是否可寫入 |

> [!NOTE]
> 當此介面存在於感測器物件上時，表示 `Sensor.Value` 的 `Value` 屬性可被修改。否則所有 `Sensor.Value` 屬性皆為唯讀。

---

## 📡 PropertiesChanged 訊號

所有感測器屬性變更時會自動發送 `org.freedesktop.DBus.Properties.PropertiesChanged` 訊號，供訂閱者監聽：

```python
# Python 範例：監聽感測器值變化
import dbus
from dbus.mainloop.glib import DBusGMainLoop
from gi.repository import GLib

DBusGMainLoop(set_as_default=True)
bus = dbus.SystemBus()

def on_properties_changed(interface, changed, invalidated):
    if 'Value' in changed:
        print(f"感測器值變更: {changed['Value']}")

bus.add_signal_receiver(
    on_properties_changed,
    signal_name='PropertiesChanged',
    dbus_interface='org.freedesktop.DBus.Properties',
    path='/xyz/openbmc_project/sensors/temperature/CPU_Core'
)

loop = GLib.MainLoop()
loop.run()
```

---

## 🔗 物件管理器需求

> [!IMPORTANT]
> 任何實作 `Sensor.Value` 介面的服務，必須在 `/xyz/openbmc_project/sensors` 路徑實作 `org.freedesktop.DBus.ObjectManager` 介面。

這允許客戶端透過單一呼叫取得所有感測器物件：

```bash
# 取得所有感測器物件
busctl call xyz.openbmc_project.HwmonTempSensor \
    /xyz/openbmc_project/sensors \
    org.freedesktop.DBus.ObjectManager \
    GetManagedObjects
```

---

## 📊 閾值層級比較

| 閾值等級 | 介面 | 嚴重程度 |
|----------|------|----------|
| Warning | `Threshold.Warning` | ⚠️ 警告 |
| Critical | `Threshold.Critical` | 🔴 嚴重 |
| PerformanceLoss | `Threshold.PerformanceLoss` | 📉 效能下降 |
| SoftShutdown | `Threshold.SoftShutdown` | 🔻 軟關機 |
| HardShutdown | `Threshold.HardShutdown` | ⛔ 硬關機 |

---

## 🔍 延伸閱讀

- [dbus-sensors Wiki](../dbus-sensors/Home.md) - 感測器服務實作
- [InventoryInterfaces](InventoryInterfaces.md) - 感測器與硬體清單關聯
- [Associations](Associations.md) - 關聯機制詳解

---

*最後更新：2025-12-19*
