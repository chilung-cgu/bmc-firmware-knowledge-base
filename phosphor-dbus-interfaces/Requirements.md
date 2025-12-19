# Requirements - 介面設計規範

本文件說明 phosphor-dbus-interfaces 的介面設計規範與最佳實踐。

---

## 📋 概述

本文件基於官方 [requirements.md](https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/requirements.md)，提供定義和修改 D-Bus 介面時應遵循的規範。

---

## 🎯 一般規範

### 優先使用 `size` 和 `uint64`/`int64`

不要過度優化屬性的型別大小，除非有明確的硬體或協定規格要求。

| 使用情境 | 建議型別 |
|----------|----------|
| 可數實體（數量） | `size` |
| 一般數值 | `uint64` 或 `int64` |
| 硬體規格明確要求 | 使用規格定義的型別 |

```yaml
# ✅ 推薦
properties:
    - name: Count
      type: size
      description: Number of items

    - name: Timestamp
      type: uint64
      description: Time in milliseconds

# ❌ 避免（除非有特定理由）
properties:
    - name: Count
      type: uint8  # 不必要的優化
```

### 避免使用任意字串

任意字串**只能**用於人類可讀的顯示，**不應**被程式碼解析。

```yaml
# ✅ 正確用法
properties:
    - name: PrettyName
      type: string
      description: >
          Human readable name. This field can be shown in user interfaces
          but should not be used for any programmatic interrogation.

# ❌ 錯誤用法
properties:
    - name: Status
      type: string  # 應使用列舉
      description: Current status
```

### 使用列舉取代字串或魔術值

sdbusplus 內建列舉支援，提供型別安全和自動轉換。

```yaml
# ✅ 使用列舉
properties:
    - name: Status
      type: enum[self.Status]
      description: Current status

enumerations:
    - name: Status
      values:
          - name: Active
          - name: Inactive
          - name: Error

# ❌ 避免使用魔術值
properties:
    - name: Status
      type: string  # 或 int
```

---

## 🔗 關聯規範

### 關聯類型

| 類型 | 用途 | 命名方式 |
|------|------|----------|
| Peer Association | 連結同一實體的不同表示 | 使用階層名稱 |
| Directed Association | 表示主從關係 | 使用動詞分詞 |

### Peer Association 命名

使用路徑階層名稱：

```yaml
# 連結 inventory 和 control 下的同一實體
associations:
    - name: inventory
      reverse_name: control
```

### Directed Association 命名

使用動詞分詞時態，**不要**在名稱中編碼類型：

| 方向 | 時態 | 範例 |
|------|------|------|
| 正向 | 現在分詞 (-ing) | `containing`, `powering`, `cooling` |
| 反向 | 過去分詞 (-ed/by) | `contained_by`, `powered_by`, `cooled_by` |

```yaml
# ✅ 正確命名
associations:
    - name: powering      # 正向：供電中
      reverse_name: powered_by  # 反向：被供電

# ❌ 錯誤命名
associations:
    - name: powered_processor  # 不應編碼類型
      reverse_name: power_supply
```

### 文法驗證

關聯命名應能組成正確的句子：

- 「The {primary} is {primary_association} the {secondary}」
- 「The {secondary} is {secondary_association} the {primary}」

範例：
- ✅ "The power_supply is **powering** the processor"
- ✅ "The processor is **powered_by** the power_supply"

### 雙向文件記錄

如果介面 A 定義了到介面 B 的關聯，介面 B 必須記錄反向關聯：

```yaml
# Sensor/Value.interface.yaml
associations:
    - name: inventory
      reverse_name: sensors
      required_endpoint_interfaces:
          - xyz.openbmc_project.Inventory.Item

# Inventory/Item.interface.yaml
associations:
    - name: sensors
      reverse_name: inventory
      required_endpoint_interfaces:
          - xyz.openbmc_project.Sensor.Value
```

---

## 📝 屬性規範

### 提供預設值

新增屬性時應提供 `default` 值，以維持向後相容：

```yaml
properties:
    - name: NewProperty
      type: uint32
      default: 0
      description: Newly added property
```

### 使用適當的旗標

| 旗標 | 用途 |
|------|------|
| `const` | 初始化後不可變的屬性 |
| `deprecated` | 已棄用，將來會移除 |
| `emits_change` | 變更時發送訊號（預設行為） |
| `emits_invalidation` | 只發送無效化通知 |

```yaml
properties:
    - name: SerialNumber
      type: string
      flags:
          - const  # 序號不應變更
      description: Device serial number
```

### 避免寬泛的 variant 型別

如果可能的值類型有限，使用明確的列舉或結構：

```yaml
# ❌ 過於寬泛
properties:
    - name: Value
      type: variant[...]  # 避免無限制的 variant

# ✅ 更明確
properties:
    - name: Value
      type: double
```

---

## 🔧 方法規範

### 清楚定義錯誤

列出方法可能拋出的所有錯誤：

```yaml
methods:
    - name: SetValue
      description: Sets the value
      parameters:
          - name: value
            type: uint32
      errors:
          - xyz.openbmc_project.Common.Error.InvalidArgument
          - xyz.openbmc_project.Common.Error.NotAllowed
```

### 使用有意義的參數名稱

```yaml
# ✅ 清晰的參數名稱
parameters:
    - name: temperatureValue
      type: double
      description: Temperature in degrees Celsius

# ❌ 模糊的參數名稱
parameters:
    - name: val
      type: double
```

---

## 📊 列舉規範

### 值名稱使用 PascalCase

```yaml
# ✅ PascalCase
values:
    - name: ReadyToStart
    - name: InProgress
    - name: Completed

# ❌ 其他風格
values:
    - name: READY_TO_START  # 避免
    - name: ready_to_start  # 避免
```

### 保留文字需要引號

```yaml
values:
    - name: "Off"    # 需要引號
    - name: "On"     # 需要引號
    - name: Running  # 不需要引號
```

---

## 📁 命名空間規範

### 遵循現有結構

| 功能類別 | 命名空間 |
|----------|----------|
| 感測器相關 | `xyz.openbmc_project.Sensor.*` |
| 狀態管理 | `xyz.openbmc_project.State.*` |
| 硬體清單 | `xyz.openbmc_project.Inventory.*` |
| 控制功能 | `xyz.openbmc_project.Control.*` |
| 日誌相關 | `xyz.openbmc_project.Logging.*` |

### 避免創建重複介面

在定義新介面前，檢查是否已有類似功能的介面可重用或擴展。

---

## 🔄 版本相容性

### 避免破壞性變更

| 動作 | 允許 | 說明 |
|------|------|------|
| 新增屬性（有預設值） | ✅ | 向後相容 |
| 新增方法 | ✅ | 向後相容 |
| 新增列舉值 | ✅ | 向後相容 |
| 移除屬性 | ❌ | 破壞性 |
| 變更屬性型別 | ❌ | 破壞性 |
| 重新命名 | ❌ | 破壞性 |

### 棄用流程

1. 將舊介面標記為 `deprecated`
2. 新增替代介面
3. 在多個版本後移除舊介面

```yaml
properties:
    - name: OldProperty
      type: string
      flags:
          - deprecated  # 標記為棄用
      description: >
          DEPRECATED: Use NewProperty instead
```

---

## ✅ 審查檢查清單

在提交新介面前，確認：

- [ ] 使用適當的型別（優先 `size`、`uint64`）
- [ ] 使用列舉而非任意字串
- [ ] 新屬性有預設值
- [ ] 方法列出所有可能的錯誤
- [ ] 關聯遵循命名規範
- [ ] 關聯在兩端都有文件記錄
- [ ] 介面有清晰的描述
- [ ] 屬性和方法有描述
- [ ] 沒有破壞性變更

---

## 🔍 延伸閱讀

- [官方 requirements.md](https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/requirements.md)
- [YAMLFormat](YAMLFormat.md) - YAML 語法詳解
- [Associations](Associations.md) - 關聯機制詳解
- [Enumerations](Enumerations.md) - 列舉使用說明

---

*最後更新：2025-12-19*
