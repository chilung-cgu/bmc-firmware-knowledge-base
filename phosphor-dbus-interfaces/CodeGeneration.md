# Code Generation - 程式碼生成

本文件說明如何使用 sdbusplus 的 `sdbus++` 工具從 YAML 介面定義生成 C++ 程式碼。

---

## 📋 概述

`sdbus++` 是 sdbusplus 專案提供的程式碼生成工具，可從 YAML 介面定義自動生成：

| 產出物 | 說明 |
|--------|------|
| Server Headers | D-Bus 服務端標頭檔 |
| Server Source | D-Bus 服務端實作檔 |
| Client Headers | D-Bus 客戶端標頭檔 |
| Error Headers | 錯誤例外標頭檔 |
| Error Source | 錯誤例外實作檔 |
| Markdown | 介面文件 |

---

## 🛠️ 安裝 sdbus++

### 從 sdbusplus 專案安裝

```bash
# 複製 sdbusplus 專案
git clone https://github.com/openbmc/sdbusplus.git
cd sdbusplus

# 安裝 Python 工具
cd tools
pip install .
```

### 依賴套件

| 套件 | 說明 |
|------|------|
| Python 3 | 執行環境 |
| mako | Python 模板引擎 |
| inflection | 命名轉換工具 |

---

## 📁 檔案路徑對應

YAML 檔案路徑直接對應到介面名稱：

| 介面名稱 | YAML 路徑 |
|----------|-----------|
| `xyz.openbmc_project.Sensor.Value` | `yaml/xyz/openbmc_project/Sensor/Value.interface.yaml` |
| `xyz.openbmc_project.State.Host` | `yaml/xyz/openbmc_project/State/Host.interface.yaml` |
| `org.freedesktop.DBus.ObjectManager` | `yaml/org/freedesktop/DBus/ObjectManager.interface.yaml` |

---

## 🔧 sdbus++ 命令

### 基本語法

```bash
sdbus++ <type> <command> <interface>
```

### 類型（type）

| 類型 | 說明 |
|------|------|
| `interface` | 處理介面定義 (.interface.yaml) |
| `error` | 處理錯誤定義 (.errors.yaml) |

### 介面命令（interface commands）

| 命令 | 說明 | 輸出檔案 |
|------|------|----------|
| `server-header` | 服務端標頭檔 | `server.hpp` |
| `server-cpp` | 服務端實作檔 | `server.cpp` |
| `client-header` | 客戶端標頭檔 | `client.hpp` |
| `markdown` | Markdown 文件 | `*.md` |

### 錯誤命令（error commands）

| 命令 | 說明 | 輸出檔案 |
|------|------|----------|
| `exception-header` | 例外標頭檔 | `error.hpp` |
| `exception-cpp` | 例外實作檔 | `error.cpp` |
| `markdown` | Markdown 文件 | `*.md` |

---

## 💻 使用範例

### 生成服務端程式碼

```bash
# 生成 Sensor.Value 的服務端標頭
sdbus++ interface server-header xyz.openbmc_project.Sensor.Value > \
    xyz/openbmc_project/Sensor/Value/server.hpp

# 生成服務端實作檔
sdbus++ interface server-cpp xyz.openbmc_project.Sensor.Value > \
    xyz/openbmc_project/Sensor/Value/server.cpp
```

### 生成錯誤例外

```bash
# 生成 Common 錯誤的例外標頭
sdbus++ error exception-header xyz.openbmc_project.Common > \
    xyz/openbmc_project/Common/error.hpp

# 生成例外實作檔
sdbus++ error exception-cpp xyz.openbmc_project.Common > \
    xyz/openbmc_project/Common/error.cpp
```

### 生成文件

```bash
# 生成介面文件
sdbus++ interface markdown xyz.openbmc_project.Sensor.Value > \
    xyz/openbmc_project/Sensor/Value.md

# 附加錯誤文件
sdbus++ error markdown xyz.openbmc_project.Sensor >> \
    xyz/openbmc_project/Sensor/Value.md
```

---

## 🏗️ Meson 建置整合

phosphor-dbus-interfaces 使用 Meson 建置系統自動執行程式碼生成。

### 建置專案

```bash
# 設定建置目錄
meson builddir

# 編譯（會自動生成程式碼）
ninja -C builddir

# 安裝
ninja -C builddir install
```

### 啟用擴展命名空間

預設只建置 `xyz/openbmc_project` 和 `org/freedesktop`。其他命名空間需手動啟用：

```bash
# 啟用 IBM 介面
meson builddir -Ddata_com_ibm=true

# 啟用 Open Power 介面
meson builddir -Ddata_org_open_power=true

# 同時啟用多個
meson builddir -Ddata_com_ibm=true -Ddata_org_open_power=true
```

### 重新生成 Meson 檔案

新增或移除 YAML 檔案後，需重新生成 Meson 設定：

```bash
cd gen
./regenerate-meson
```

---

## 📝 生成的程式碼結構

### Server Header 結構

生成的 `server.hpp` 包含：

```cpp
namespace sdbusplus::xyz::openbmc_project::Sensor::server
{

class Value
{
  public:
    // 介面名稱常數
    static constexpr auto interface = "xyz.openbmc_project.Sensor.Value";

    // 列舉定義
    enum class Unit
    {
        Amperes,
        DegreesC,
        Volts,
        Watts,
        // ...
    };

    // 屬性 getter（虛擬函式，需覆寫）
    virtual double value() const = 0;
    virtual double maxValue() const = 0;
    virtual Unit unit() const = 0;

    // 屬性 setter
    virtual double value(double value) = 0;

    // 內部實作
    // ...
};

} // namespace sdbusplus::xyz::openbmc_project::Sensor::server
```

### 實作介面

繼承生成的類別並實作虛擬函式：

```cpp
#include <xyz/openbmc_project/Sensor/Value/server.hpp>

namespace phosphor::sensor
{

using ValueInterface = sdbusplus::xyz::openbmc_project::Sensor::server::Value;

class TemperatureSensor : public ValueInterface
{
  public:
    // 實作屬性 getter
    double value() const override
    {
        return currentReading_;
    }

    double maxValue() const override
    {
        return 125.0;  // 最高 125°C
    }

    Unit unit() const override
    {
        return Unit::DegreesC;
    }

    // 實作屬性 setter
    double value(double value) override
    {
        currentReading_ = value;
        return currentReading_;
    }

  private:
    double currentReading_ = 0.0;
};

} // namespace phosphor::sensor
```

---

## 🔄 新增介面流程

### 步驟 1：建立 YAML 檔案

在適當的目錄下建立 `.interface.yaml` 檔案：

```bash
mkdir -p yaml/xyz/openbmc_project/MyFeature
vim yaml/xyz/openbmc_project/MyFeature/MyInterface.interface.yaml
```

### 步驟 2：編寫介面定義

```yaml
description: >
    My custom interface for feature X.

properties:
    - name: Status
      type: string
      description: Current status

methods:
    - name: DoSomething
      description: Perform an action
      parameters:
          - name: input
            type: string
      returns:
          - name: result
            type: boolean
```

### 步驟 3：重新生成 Meson 設定

```bash
cd gen
./regenerate-meson
```

### 步驟 4：建置並驗證

```bash
meson builddir
ninja -C builddir

# 檢查生成的標頭檔
ls builddir/xyz/openbmc_project/MyFeature/MyInterface/
```

---

## 🐛 除錯技巧

### 驗證 YAML 語法

```bash
# 使用 Python 解析測試
python3 -c "import yaml; yaml.safe_load(open('MyInterface.interface.yaml'))"
```

### 檢視生成內容

```bash
# 直接輸出到終端機檢視
sdbus++ interface server-header xyz.openbmc_project.MyFeature.MyInterface
```

### 常見錯誤

| 錯誤 | 原因 | 解決方式 |
|------|------|----------|
| `FileNotFoundError` | 找不到 YAML 檔案 | 確認路徑與介面名稱對應 |
| `yaml.scanner.ScannerError` | YAML 語法錯誤 | 檢查縮排和特殊字元 |
| `KeyError: 'name'` | 缺少必要欄位 | 檢查 properties/methods 格式 |

---

## 🔍 延伸閱讀

- [YAML 格式](YAMLFormat.md) - 完整的 YAML 語法說明
- [建置與設定](BuildConfig.md) - Meson 建置詳細設定
- [sdbusplus GitHub](https://github.com/openbmc/sdbusplus) - 官方專案

---

*最後更新：2025-12-19*
