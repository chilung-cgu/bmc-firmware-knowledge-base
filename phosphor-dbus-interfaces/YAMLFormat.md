# YAML Format - YAML 介面定義格式

本文件詳細說明 phosphor-dbus-interfaces 使用的 YAML 介面定義格式。

---

## 📋 概述

phosphor-dbus-interfaces 使用 YAML 格式描述 D-Bus 介面。每個介面定義檔包含該介面的完整規格，包括描述、方法、屬性、訊號和列舉。

### 檔案命名規則

| 檔案類型 | 命名格式 | 說明 |
|----------|----------|------|
| 介面定義 | `InterfaceName.interface.yaml` | 主要介面描述 |
| 錯誤定義 | `InterfaceName.errors.yaml` | 錯誤/例外定義 |
| 元資料 | `InterfaceName.metadata.yaml` | 錯誤附加資訊 |
| 事件定義 | `InterfaceName.events.yaml` | 事件日誌定義 |

---

## 📄 介面定義結構

一個完整的 `.interface.yaml` 檔案可包含以下區段：

```yaml
description: >
    介面說明文字

properties:
    # 屬性列表

methods:
    # 方法列表

signals:
    # 訊號列表

enumerations:
    # 列舉定義

associations:
    # 關聯定義

paths:
    # D-Bus 路徑定義

service_names:
    # 服務名稱定義
```

---

## 📝 description - 介面描述

每個介面應以 `description` 開頭，說明介面的用途和使用規範：

```yaml
description: >
    Implement to provide sensor readings. Objects implementing Sensor.Value
    must be instantiated in the correct hierarchy within the sensors
    namespace. The following sensor hierarchies are recognized:
      airflow
      altitude
      current
      temperature
      voltage
```

> [!TIP]
> 使用 YAML 的 `>` 符號可將多行文字合併為單行，便於撰寫長描述。

---

## 🔧 properties - 屬性定義

屬性是介面公開的數據欄位，可被讀取或寫入。

### 基本語法

```yaml
properties:
    - name: PropertyName
      type: string
      description: >
          屬性說明
```

### 完整屬性選項

| 欄位 | 必填 | 說明 |
|------|------|------|
| `name` | ✅ | 屬性名稱（PascalCase） |
| `type` | ✅ | 資料型別 |
| `description` | 建議 | 屬性說明 |
| `default` | 否 | 預設值 |
| `flags` | 否 | 屬性旗標 |
| `errors` | 否 | 可能拋出的錯誤 |

### 屬性旗標（flags）

| 旗標 | 說明 |
|------|------|
| `const` | 屬性為常數，初始化後不可變更 |
| `deprecated` | 已棄用的屬性 |
| `hidden` | 不生成公開 getter/setter |
| `unprivileged` | 允許未授權存取 |
| `emits_change` | 變更時發送 PropertiesChanged 訊號（預設） |
| `emits_invalidation` | 變更時只發送無效化通知 |
| `explicit` | 不自動發送變更訊號 |

### 範例：感測器值屬性

```yaml
properties:
    - name: Value
      type: double
      description: >
          The sensor reading.

    - name: MaxValue
      type: double
      default: infinity
      description: >
          The Maximum supported sensor reading.

    - name: Unit
      type: enum[self.Unit]
      description: >
          The unit of the reading. Immutable once set for a sensor.

    - name: AllowedTransitions
      type: set[enum[self.Transition]]
      flags:
          - const
      description: >
          A const property describing the allowed transitions.
```

---

## 📞 methods - 方法定義

方法是介面提供的可呼叫操作。

### 基本語法

```yaml
methods:
    - name: MethodName
      description: >
          方法說明
      parameters:
          - name: paramName
            type: string
            description: 參數說明
      returns:
          - name: resultName
            type: string
            description: 回傳值說明
```

### 完整方法選項

| 欄位 | 必填 | 說明 |
|------|------|------|
| `name` | ✅ | 方法名稱（PascalCase） |
| `description` | 建議 | 方法說明 |
| `parameters` | 否 | 輸入參數列表 |
| `returns` | 否 | 回傳值列表 |
| `flags` | 否 | 方法旗標 |
| `errors` | 否 | 可能拋出的錯誤 |

### 方法旗標（flags）

| 旗標 | 說明 |
|------|------|
| `deprecated` | 已棄用的方法 |
| `hidden` | 不生成公開方法 |
| `unprivileged` | 允許未授權呼叫 |
| `no_reply` | 不等待回覆 |

### 範例：ObjectMapper 方法

```yaml
methods:
    - name: GetObject
      description: >
          Obtain a dictionary of service -> implemented interface(s) for the
          given path.
      parameters:
          - name: path
            type: string
            description: >
                The object path for which the result should be fetched.
          - name: interfaces
            type: array[string]
            description: >
                An array of result set constraining interfaces.
      returns:
          - name: services
            type: dict[string,array[string]]
            description: >
                A dictionary of service -> implemented interface(s).
      errors:
          - xyz.openbmc_project.Common.Error.ResourceNotFound
```

---

## 📡 signals - 訊號定義

訊號是介面可發出的非同步通知。

### 基本語法

```yaml
signals:
    - name: SignalName
      description: >
          訊號說明
      properties:
          - name: propertyName
            type: double
            description: 訊號攜帶的屬性
```

### 範例：閾值警報訊號

```yaml
signals:
    - name: WarningHighAlarmAsserted
      description: >
          The high threshold alarm asserted.
      properties:
          - name: SensorValue
            type: double
            description: >
                The sensor value that triggered the alarm change.

    - name: WarningHighAlarmDeasserted
      description: >
          The high threshold alarm deasserted.
      properties:
          - name: SensorValue
            type: double
            description: >
                The sensor value that triggered the alarm change.
```

---

## 📊 enumerations - 列舉定義

列舉定義了一組具名的常數值。

### 基本語法

```yaml
enumerations:
    - name: EnumName
      description: >
          列舉說明
      values:
          - name: Value1
            description: 值說明
          - name: Value2
            description: 值說明
```

### 範例：主機狀態列舉

```yaml
enumerations:
    - name: HostState
      description: >
          The current state of the host firmware
      values:
          - name: "Off"
            description: >
                Host firmware is not running
          - name: "Running"
            description: >
                Host firmware is running
          - name: "TransitioningToOff"
            description: >
                Host firmware is transitioning to an Off state
          - name: "Quiesced"
            description: >
                Host firmware is quiesced. The host firmware is enabled but
                either unresponsive or only processing a restricted set of
                commands.
```

### 引用列舉

在屬性或參數中引用列舉：

```yaml
properties:
    - name: CurrentHostState
      type: enum[self.HostState]  # 引用同檔案的列舉
      description: >
          A read-only property describing the current state.

    - name: Severity
      type: enum[xyz.openbmc_project.Logging.Entry.Level]  # 引用其他介面的列舉
      description: >
          The severity of the error event entry.
```

---

## 🔗 associations - 關聯定義

關聯定義物件之間的關係。

### 基本語法

```yaml
associations:
    - name: association_name
      description: >
          關聯說明
      reverse_name: reverse_association_name
      required_endpoint_interfaces:
          - xyz.openbmc_project.Some.Interface
```

### 範例：感測器與清單關聯

```yaml
associations:
    - name: inventory
      description: >
          Sensors may implement an 'inventory' to 'sensors' association with
          the inventory item related to it.
      reverse_name: sensors
      required_endpoint_interfaces:
          - xyz.openbmc_project.Inventory.Item

    - name: monitoring
      description: >
          Sensors may monitor the BMC's resource utilization and implement an
          association to the Bmc item.
      reverse_name: monitored_by
      required_endpoint_interfaces:
          - xyz.openbmc_project.Inventory.Item.Bmc
```

詳細說明請參考 [Associations](Associations.md)。

---

## 📍 paths - 路徑定義

定義介面應實作的 D-Bus 物件路徑。

### 基本語法

```yaml
paths:
    - namespace: /xyz/openbmc_project/sensors
      segments:
          - name: Temperature
            value: temperature
          - name: Voltage
            value: voltage
```

### 範例：感測器路徑

```yaml
paths:
    - namespace: /xyz/openbmc_project/sensors
      segments:
          - name: Airflow
            value: airflow
          - name: Current
            value: current
          - name: FanTach
            value: fan_tach
          - name: Temperature
            value: temperature
          - name: Voltage
            value: voltage
```

---

## 🏷️ service_names - 服務名稱

定義可實作此介面的服務名稱。

```yaml
service_names:
    - default: xyz.openbmc_project.Sensor
    - xyz.openbmc_project.HwmonTempSensor
    - xyz.openbmc_project.ADCSensor
```

---

## 📊 資料型別參考

### 基本型別

| YAML 型別 | D-Bus 型別 | C++ 型別 |
|-----------|------------|----------|
| `boolean` | `b` | `bool` |
| `byte` | `y` | `uint8_t` |
| `int16` | `n` | `int16_t` |
| `uint16` | `q` | `uint16_t` |
| `int32` | `i` | `int32_t` |
| `uint32` | `u` | `uint32_t` |
| `int64` | `x` | `int64_t` |
| `uint64` | `t` | `uint64_t` |
| `double` | `d` | `double` |
| `string` | `s` | `std::string` |
| `object_path` | `o` | `sdbusplus::message::object_path` |
| `signature` | `g` | `sdbusplus::message::signature` |
| `unix_fd` | `h` | `sdbusplus::message::unix_fd` |
| `size` | 依架構 | `size_t` |

### 複合型別

| YAML 型別 | D-Bus 型別 | C++ 型別 |
|-----------|------------|----------|
| `array[T]` | `aT` | `std::vector<T>` |
| `set[T]` | `aT` | `std::set<T>` |
| `dict[K,V]` | `a{KV}` | `std::map<K,V>` |
| `struct[T1,T2,...]` | `(T1T2...)` | `std::tuple<T1,T2,...>` |
| `variant[T1,T2,...]` | `v` | `std::variant<T1,T2,...>` |
| `enum[E]` | `s` | `E` (C++ enum class) |

詳細說明請參考 [CommonDBusTypes](CommonDBusTypes.md)。

---

## 🔍 完整範例

以下是一個完整的介面定義範例：

```yaml
description: >
    Implement to provide power cap control and monitoring.

properties:
    - name: PowerCap
      type: uint32
      description: >
          Power cap value in Watts.

    - name: PowerCapEnable
      type: boolean
      description: >
          Power cap enable. Set to true to enable the PowerCap.

    - name: ExceptionAction
      type: enum[self.ExceptionActions]
      default: NoAction
      description: >
          Exception Actions, taken if the Power Limit is exceeded.

enumerations:
    - name: ExceptionActions
      description: >
          Exception actions for power limit violations.
      values:
          - name: NoAction
            description: No action is called.
          - name: HardPowerOff
            description: Hard Power Off system and log event.
          - name: LogEventOnly
            description: Log Event Only.

associations:
    - name: power_controlling
      description: >
          Association to the chassis being power controlled.
      reverse_name: power_controlled_by
      required_endpoint_interfaces:
          - xyz.openbmc_project.Inventory.Item.Chassis
```

---

## 🔍 延伸閱讀

- [程式碼生成](CodeGeneration.md) - 如何使用 sdbus++ 生成程式碼
- [列舉型別](Enumerations.md) - 列舉的詳細使用說明
- [關聯機制](Associations.md) - 關聯的進階用法
- [介面設計規範](Requirements.md) - YAML 撰寫最佳實踐

---

*最後更新：2025-12-19*
