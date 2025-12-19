# Common D-Bus Types - 常用 D-Bus 型別

本文件說明 phosphor-dbus-interfaces 中使用的 D-Bus 資料型別。

---

## 📋 概述

D-Bus 定義了一組基本資料型別，sdbusplus 將這些型別映射到 C++ 型別。YAML 介面定義使用描述性的型別名稱。

---

## 🔢 基本型別

### 布林與數值型別

| YAML 型別 | D-Bus 簽章 | C++ 型別 | 說明 |
|-----------|------------|----------|------|
| `boolean` | `b` | `bool` | 布林值 |
| `byte` | `y` | `uint8_t` | 無號 8 位元 |
| `int16` | `n` | `int16_t` | 有號 16 位元 |
| `uint16` | `q` | `uint16_t` | 無號 16 位元 |
| `int32` | `i` | `int32_t` | 有號 32 位元 |
| `uint32` | `u` | `uint32_t` | 無號 32 位元 |
| `int64` | `x` | `int64_t` | 有號 64 位元 |
| `uint64` | `t` | `uint64_t` | 無號 64 位元 |
| `double` | `d` | `double` | 雙精度浮點數 |

### 字串型別

| YAML 型別 | D-Bus 簽章 | C++ 型別 | 說明 |
|-----------|------------|----------|------|
| `string` | `s` | `std::string` | UTF-8 字串 |
| `object_path` | `o` | `sdbusplus::message::object_path` | D-Bus 物件路徑 |
| `signature` | `g` | `sdbusplus::message::signature` | D-Bus 型別簽章 |
| `unix_fd` | `h` | `sdbusplus::message::unix_fd` | Unix 檔案描述符 |

### sdbusplus 特殊型別

| YAML 型別 | C++ 型別 | 說明 |
|-----------|----------|------|
| `size` | `size_t` | 架構相關的大小型別 |

---

## 📦 複合型別

### 陣列

```yaml
type: array[T]
```

| 範例 | D-Bus 簽章 | C++ 型別 |
|------|------------|----------|
| `array[string]` | `as` | `std::vector<std::string>` |
| `array[uint32]` | `au` | `std::vector<uint32_t>` |
| `array[object_path]` | `ao` | `std::vector<sdbusplus::message::object_path>` |

### 集合

```yaml
type: set[T]
```

與 `array` 類似，但元素唯一且有序：

| 範例 | C++ 型別 |
|------|----------|
| `set[string]` | `std::set<std::string>` |
| `set[enum[...]]` | `std::set<EnumType>` |

### 字典

```yaml
type: dict[K, V]
```

| 範例 | D-Bus 簽章 | C++ 型別 |
|------|------------|----------|
| `dict[string, string]` | `a{ss}` | `std::map<std::string, std::string>` |
| `dict[string, variant]` | `a{sv}` | `std::map<std::string, sdbusplus::message::variant<...>>` |
| `dict[string, array[string]]` | `a{sas}` | `std::map<std::string, std::vector<std::string>>` |

### 結構

```yaml
type: struct[T1, T2, ...]
```

| 範例 | D-Bus 簽章 | C++ 型別 |
|------|------------|----------|
| `struct[string, uint32]` | `(su)` | `std::tuple<std::string, uint32_t>` |
| `struct[string, string, object_path]` | `(sso)` | `std::tuple<std::string, std::string, sdbusplus::message::object_path>` |

### Variant

```yaml
type: variant[T1, T2, ...]
```

| 範例 | C++ 型別 |
|------|----------|
| `variant[string, uint32, boolean]` | `std::variant<std::string, uint32_t, bool>` |

---

## 📊 列舉型別

### 引用列舉

```yaml
# 引用同一介面的列舉
type: enum[self.EnumName]

# 引用其他介面的列舉
type: enum[xyz.openbmc_project.Other.Interface.EnumName]
```

### D-Bus 表示

列舉在 D-Bus 上以完整限定字串傳輸：

```
xyz.openbmc_project.State.Host.HostState.Running
```

---

## 🎯 型別選擇指南

### 數值型別選擇

| 使用情境 | 建議型別 | 說明 |
|----------|----------|------|
| 計數/數量 | `size` | 架構相關，通常夠大 |
| 時間戳 | `uint64` | 毫秒或微秒 |
| 一般整數 | `int64` / `uint64` | 避免過度優化 |
| 感測器讀數 | `double` | 支援小數和特殊值 |
| 硬體規格定義 | 按規格選擇 | 如 IPMI、Redfish |

### 字串 vs 列舉

| 使用情境 | 建議型別 |
|----------|----------|
| 固定值集合 | `enum[...]` |
| 人類可讀資訊 | `string` |
| 可擴展但有規範的值 | `string`（文件說明格式） |

### 容器型別選擇

| 使用情境 | 建議型別 |
|----------|----------|
| 有序列表 | `array[T]` |
| 唯一元素集合 | `set[T]` |
| 鍵值對應 | `dict[K, V]` |
| 固定結構資料 | `struct[...]` |
| 多型態值 | `variant[...]` |

---

## 💡 特殊值

### 數值特殊值

| YAML 關鍵字 | 說明 |
|-------------|------|
| `infinity` | 正無限大 |
| `-infinity` | 負無限大 |
| `NaN` | 非數字（用於表示無此值） |
| `maxint` | 該型別的最大值 |

### 使用範例

```yaml
properties:
    - name: MaxValue
      type: double
      default: infinity  # 預設無上限
      
    - name: WarningHigh
      type: double
      default: NaN  # 預設無此閾值
```

---

## 📝 常見型別模式

### 屬性字典

用於傳遞動態屬性集合：

```yaml
type: dict[string, variant[string, uint64, boolean, double]]
```

### 物件/服務映射

ObjectMapper 回傳格式：

```yaml
# GetObject 回傳
type: dict[string, array[string]]  # 服務 -> 介面列表

# GetSubTree 回傳
type: dict[string, dict[string, array[string]]]  # 路徑 -> (服務 -> 介面列表)
```

### 關聯定義

```yaml
# 關聯三元組：(正向名稱, 反向名稱, 端點路徑)
type: array[struct[string, string, string]]
```

### 日誌附加資料

```yaml
type: dict[string, string]  # 鍵值對
```

---

## ⚠️ 注意事項

### 巢狀複合型別

複雜的巢狀型別應謹慎使用：

```yaml
# 可接受但複雜
type: dict[string, array[struct[string, uint32]]]

# 考慮分解為多個屬性
properties:
    - name: Keys
      type: array[string]
    - name: Values
      type: array[struct[string, uint32]]
```

### Variant 型別

- 使用時應明確列出所有可能的型別
- 避免過於寬泛的 variant（會增加使用複雜度）
- 考慮是否能用列舉或多屬性替代

---

## 🔍 延伸閱讀

- [YAMLFormat](YAMLFormat.md) - 在 YAML 中使用型別
- [Enumerations](Enumerations.md) - 列舉型別詳解
- [D-Bus 規格](https://dbus.freedesktop.org/doc/dbus-specification.html#type-system) - 官方型別系統文件

---

*最後更新：2025-12-19*
