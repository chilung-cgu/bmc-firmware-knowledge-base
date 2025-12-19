# Errors - 錯誤處理

本文件說明 phosphor-dbus-interfaces 中的錯誤定義與處理機制。

---

## 📋 概述

phosphor-dbus-interfaces 定義了標準化的 D-Bus 錯誤，讓服務可以拋出結構化的錯誤回應。錯誤透過 YAML 檔案定義，sdbus++ 自動生成 C++ 例外類別。

### 檔案類型

| 檔案類型 | 說明 |
|----------|------|
| `.errors.yaml` | 定義錯誤名稱和說明 |
| `.metadata.yaml` | 定義錯誤的附加元資料 |

---

## 📝 定義錯誤

### .errors.yaml 語法

```yaml
- name: ErrorName
  description: >
      錯誤說明
- name: AnotherError
  description: >
      另一個錯誤說明
```

### 範例：Common.errors.yaml

```yaml
# yaml/xyz/openbmc_project/Common.errors.yaml
- name: InternalFailure
  description: >
      The operation failed internally.
- name: InvalidArgument
  description: >
      An argument was invalid.
- name: NotAllowed
  description: >
      The caller is not allowed to perform this operation.
- name: ResourceNotFound
  description: >
      The resource was not found.
- name: Unavailable
  description: >
      The resource is temporarily unavailable.
```

---

## 📋 定義元資料

### .metadata.yaml 語法

```yaml
- name: ErrorName
  meta:
      - str: METADATA_NAME
        type: string
```

### 範例：Common.metadata.yaml

```yaml
# yaml/xyz/openbmc_project/Common.metadata.yaml
- name: InvalidArgument
  meta:
      - str: ARGUMENT
        type: string
      - str: VALUE
        type: string
- name: ResourceNotFound
  meta:
      - str: RESOURCE
        type: string
```

---

## 🔧 sdbus++ 生成

### 生成命令

```bash
# 生成例外標頭
sdbus++ error exception-header xyz.openbmc_project.Common > \
    xyz/openbmc_project/Common/error.hpp

# 生成例外實作
sdbus++ error exception-cpp xyz.openbmc_project.Common > \
    xyz/openbmc_project/Common/error.cpp
```

### 生成的 C++ 程式碼

```cpp
namespace sdbusplus::xyz::openbmc_project::Common::Error
{

struct InternalFailure
{
    static constexpr auto errName = 
        "xyz.openbmc_project.Common.Error.InternalFailure";
    static constexpr auto errDesc = 
        "The operation failed internally.";
    // 繼承自 sdbusplus::exception::exception
};

struct InvalidArgument
{
    static constexpr auto errName = 
        "xyz.openbmc_project.Common.Error.InvalidArgument";
    static constexpr auto errDesc = 
        "An argument was invalid.";
    // 元資料存取方法
    static constexpr auto ARGUMENT = "ARGUMENT";
    static constexpr auto VALUE = "VALUE";
};

} // namespace
```

---

## 💻 使用錯誤

### 在介面方法中宣告錯誤

```yaml
methods:
    - name: DoSomething
      description: >
          Performs an operation.
      errors:
          - xyz.openbmc_project.Common.Error.InvalidArgument
          - xyz.openbmc_project.Common.Error.InternalFailure
```

### 在 C++ 中拋出錯誤

```cpp
#include <xyz/openbmc_project/Common/error.hpp>

using namespace sdbusplus::xyz::openbmc_project::Common::Error;

void doSomething(const std::string& arg)
{
    if (arg.empty())
    {
        // 拋出錯誤
        throw InvalidArgument();
    }
    
    try
    {
        // 操作...
    }
    catch (const std::exception& e)
    {
        throw InternalFailure();
    }
}
```

### 附帶元資料的錯誤

```cpp
#include <phosphor-logging/elog.hpp>
#include <phosphor-logging/elog-errors.hpp>
#include <xyz/openbmc_project/Common/error.hpp>

using namespace phosphor::logging;
using InvalidArgument = 
    sdbusplus::xyz::openbmc_project::Common::Error::InvalidArgument;

void validate(const std::string& arg, int value)
{
    if (value < 0)
    {
        elog<InvalidArgument>(
            xyz::openbmc_project::Common::InvalidArgument::ARGUMENT("value"),
            xyz::openbmc_project::Common::InvalidArgument::VALUE(
                std::to_string(value).c_str()));
    }
}
```

---

## 📊 常見錯誤類型

### xyz.openbmc_project.Common.Error

| 錯誤 | 說明 |
|------|------|
| `InternalFailure` | 內部操作失敗 |
| `InvalidArgument` | 無效的參數 |
| `NotAllowed` | 操作不被允許 |
| `ResourceNotFound` | 找不到資源 |
| `Unavailable` | 資源暫時不可用 |
| `WriteFailure` | 寫入失敗 |
| `ReadFailure` | 讀取失敗 |
| `Timeout` | 操作逾時 |

### xyz.openbmc_project.State.Host.Error

| 錯誤 | 說明 |
|------|------|
| `BMCNotReady` | BMC 尚未就緒 |

### xyz.openbmc_project.State.Chassis.Error

| 錯誤 | 說明 |
|------|------|
| `BMCNotReady` | BMC 尚未就緒 |
| `PowerOnFailure` | 開機失敗 |

### xyz.openbmc_project.Sensor.Device.Error

| 錯誤 | 說明 |
|------|------|
| `ReadFailure` | 感測器讀取失敗 |
| `WriteFailure` | 感測器寫入失敗 |

---

## 🔄 錯誤的 D-Bus 表示

錯誤在 D-Bus 上以錯誤回覆傳輸：

| 項目 | 範例 |
|------|------|
| 錯誤名稱 | `xyz.openbmc_project.Common.Error.InvalidArgument` |
| 錯誤訊息 | `An argument was invalid.` |

### 使用 busctl 測試

```bash
# 呼叫可能產生錯誤的方法
busctl call xyz.openbmc_project.State.Host \
    /xyz/openbmc_project/state/host0 \
    xyz.openbmc_project.State.Host GetNonExistentMethod

# 可能的錯誤回覆：
# Error: xyz.openbmc_project.Common.Error.NotAllowed
```

---

## 📁 錯誤檔案結構

```
yaml/xyz/openbmc_project/
├── Common.errors.yaml       # 通用錯誤
├── Common.metadata.yaml     # 通用錯誤元資料
├── Sensor/
│   ├── Device.errors.yaml   # 感測器裝置錯誤
│   ├── Device.metadata.yaml
│   ├── Threshold.errors.yaml
│   └── Threshold.metadata.yaml
├── State/
│   ├── Host.errors.yaml
│   ├── Host.metadata.yaml
│   ├── Chassis.errors.yaml
│   └── Chassis.metadata.yaml
└── ...
```

---

## 📊 與日誌的整合

錯誤通常與日誌系統整合，以便記錄問題：

```cpp
#include <phosphor-logging/log.hpp>
#include <phosphor-logging/elog.hpp>
#include <phosphor-logging/elog-errors.hpp>

using namespace phosphor::logging;
using InternalFailure = 
    sdbusplus::xyz::openbmc_project::Common::Error::InternalFailure;

void operation()
{
    try
    {
        // 可能失敗的操作
        riskyOperation();
    }
    catch (const std::exception& e)
    {
        log<level::ERR>("Operation failed",
                        entry("ERROR=%s", e.what()));
        // 記錄並拋出結構化錯誤
        elog<InternalFailure>();
    }
}
```

---

## ✍️ 最佳實踐

### 1. 選擇適當的錯誤類型

- 使用現有的通用錯誤（如 `InvalidArgument`）
- 只在必要時定義新的特定錯誤

### 2. 提供有用的元資料

- 包含足夠的上下文資訊以利除錯
- 使用描述性的元資料名稱

### 3. 在介面中宣告錯誤

- 在 YAML 的 `errors` 欄位列出所有可能的錯誤
- 這有助於 API 使用者了解需要處理的情況

### 4. 錯誤說明應清晰

- 說明何時會發生此錯誤
- 提供可能的解決方向

---

## 🔍 延伸閱讀

- [LoggingInterfaces](LoggingInterfaces.md) - 錯誤與日誌整合
- [YAMLFormat](YAMLFormat.md) - 在介面中宣告錯誤
- [phosphor-logging](https://github.com/openbmc/phosphor-logging) - 日誌服務

---

*最後更新：2025-12-19*
