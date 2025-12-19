# Enumerations - 列舉型別

本文件說明 phosphor-dbus-interfaces 中列舉（Enumeration）的定義與使用。

---

## 📋 概述

sdbusplus 將列舉視為一等型別，提供以下功能：

| 功能 | 說明 |
|------|------|
| 強型別 | 編譯時期型別檢查 |
| 自動轉換 | D-Bus 字串與 C++ enum 自動轉換 |
| 字串表示 | D-Bus 上以完整限定字串傳輸 |
| 錯誤處理 | 無效值自動產生錯誤 |

---

## 📝 在 YAML 中定義列舉

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

### 完整範例

```yaml
# State/Host.interface.yaml
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
                Host firmware is quiesced
```

---

## 🔄 D-Bus 表示

列舉值在 D-Bus 上以完整限定字串傳輸：

| C++ 列舉值 | D-Bus 字串 |
|-----------|------------|
| `HostState::Off` | `"xyz.openbmc_project.State.Host.HostState.Off"` |
| `HostState::Running` | `"xyz.openbmc_project.State.Host.HostState.Running"` |
| `Unit::DegreesC` | `"xyz.openbmc_project.Sensor.Value.Unit.DegreesC"` |

### 格式

```
{介面名稱}.{列舉名稱}.{值名稱}
```

---

## 💻 C++ 使用

### 生成的程式碼

sdbus++ 會生成 C++ enum class：

```cpp
namespace sdbusplus::xyz::openbmc_project::State::server
{

class Host
{
public:
    enum class HostState
    {
        Off,
        Running,
        TransitioningToOff,
        TransitioningToRunning,
        Standby,
        Quiesced,
        DiagnosticMode,
    };
    
    // 字串轉換函式
    static std::string convertHostStateToString(HostState value);
    static HostState convertHostStateFromString(const std::string& value);
};

} // namespace
```

### 使用方式

```cpp
#include <xyz/openbmc_project/State/Host/server.hpp>

using Host = sdbusplus::xyz::openbmc_project::State::server::Host;

// 設定屬性
Host::HostState state = Host::HostState::Running;
server.currentHostState(state);

// 讀取屬性
Host::HostState currentState = server.currentHostState();

// 轉換為字串
std::string stateStr = Host::convertHostStateToString(currentState);

// 從字串轉換
Host::HostState fromStr = Host::convertHostStateFromString(
    "xyz.openbmc_project.State.Host.HostState.Running");
```

---

## 🔧 在屬性中使用列舉

### 引用同一介面的列舉

使用 `enum[self.EnumName]` 語法：

```yaml
properties:
    - name: CurrentHostState
      type: enum[self.HostState]
      description: >
          A read-only property describing the current state.

enumerations:
    - name: HostState
      values:
          - name: "Off"
          - name: "Running"
```

### 引用其他介面的列舉

使用完整介面路徑：

```yaml
properties:
    - name: Severity
      type: enum[xyz.openbmc_project.Logging.Entry.Level]
      description: >
          The severity of the error event entry.
```

### 列舉集合

使用 `set[enum[...]]` 定義列舉集合：

```yaml
properties:
    - name: AllowedTransitions
      type: set[enum[self.Transition]]
      flags:
          - const
      description: >
          The set of allowed transitions.
```

---

## ⚠️ 錯誤處理

當提供無效的列舉字串時，sdbusplus 會拋出錯誤：

| 錯誤 | 說明 |
|------|------|
| `xyz.openbmc_project.sdbusplus.Error.InvalidEnumString` | 無效的列舉字串值 |

### 使用範例

```bash
# 正確的列舉值
busctl set-property xyz.openbmc_project.State.Host \
    /xyz/openbmc_project/state/host0 \
    xyz.openbmc_project.State.Host RequestedHostTransition s \
    "xyz.openbmc_project.State.Host.Transition.On"

# 錯誤的列舉值會產生錯誤
busctl set-property xyz.openbmc_project.State.Host \
    /xyz/openbmc_project/state/host0 \
    xyz.openbmc_project.State.Host RequestedHostTransition s \
    "InvalidValue"
# 錯誤：xyz.openbmc_project.sdbusplus.Error.InvalidEnumString
```

---

## 📊 常見列舉類型

### State 相關

| 介面 | 列舉 | 說明 |
|------|------|------|
| `State.Host` | `HostState` | 主機狀態 |
| `State.Host` | `Transition` | 狀態轉換 |
| `State.Host` | `RestartCause` | 重啟原因 |
| `State.BMC` | `BMCState` | BMC 狀態 |
| `State.Chassis` | `PowerState` | 電源狀態 |

### Sensor 相關

| 介面 | 列舉 | 說明 |
|------|------|------|
| `Sensor.Value` | `Unit` | 感測器單位 |

### Logging 相關

| 介面 | 列舉 | 說明 |
|------|------|------|
| `Logging.Entry` | `Level` | 日誌嚴重等級 |
| `Logging.Entry` | `Notify` | 通知狀態 |

### Software 相關

| 介面 | 列舉 | 說明 |
|------|------|------|
| `Software.Version` | `VersionPurpose` | 軟體用途 |
| `Software.Activation` | `Activations` | 啟用狀態 |

---

## 🎯 OEM/可擴展列舉

對於需要 OEM 擴展的情況，可以使用字串屬性但指定值應為特定格式的列舉：

```yaml
properties:
    - name: Type
      type: string
      description: >
          The type of this dump. Values should be sdbusplus-enumerations
          formatted strings from either xyz.openbmc_project.Dump.Create.DumpType
          or an OEM-specific enum.
```

範例：
- `xyz.openbmc_project.Dump.Create.DumpType.System`
- `com.ibm.Dump.Create.DumpType.Hostboot`

---

## ✍️ 命名規範

### 列舉名稱

- 使用 PascalCase
- 名稱應明確描述列舉的用途
- 例如：`HostState`, `Transition`, `Unit`

### 值名稱

- 使用 PascalCase
- 保留文字如 `"Off"`, `"On"` 需要引號
- 例如：`"Off"`, `Running`, `TransitioningToOff`

### 關鍵字衝突

使用引號避免 YAML 關鍵字衝突：

```yaml
values:
    - name: "Off"    # 需要引號
    - name: "On"     # 需要引號
    - name: Running  # 不需要引號
```

---

## 🔍 延伸閱讀

- [YAMLFormat](YAMLFormat.md) - YAML 語法詳解
- [CodeGeneration](CodeGeneration.md) - 程式碼生成細節
- [StateInterfaces](StateInterfaces.md) - 狀態介面列舉範例
- [SensorInterfaces](SensorInterfaces.md) - 感測器單位列舉

---

*最後更新：2025-12-19*
