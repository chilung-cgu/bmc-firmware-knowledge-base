# Build & Configuration - 建置與設定

本文件說明如何建置 phosphor-dbus-interfaces 專案。

---

## 📋 概述

phosphor-dbus-interfaces 使用 [Meson](https://mesonbuild.com/) 建置系統，可生成 C++ 標頭檔和綁定程式碼供其他專案使用。

### 建置產出

| 產出物 | 說明 |
|--------|------|
| C++ Headers | 介面定義的標頭檔 |
| C++ Source | 介面實作程式碼 |
| pkg-config | 供其他專案引用的設定檔 |
| 文件 | Markdown 格式的介面文件 |

---

## 🔧 建置需求

### 系統需求

| 套件 | 版本 | 說明 |
|------|------|------|
| meson | >= 0.57.0 | 建置系統 |
| ninja | 任意 | 建置引擎 |
| Python 3 | >= 3.6 | 執行 sdbus++ |
| sdbusplus | 最新 | 程式碼生成工具和函式庫 |

### Python 依賴

```bash
pip install mako inflection pyyaml
```

---

## 🏗️ 標準建置流程

### 基本建置

```bash
# 複製專案
git clone https://github.com/openbmc/phosphor-dbus-interfaces.git
cd phosphor-dbus-interfaces

# 設定建置目錄
meson builddir

# 編譯
ninja -C builddir

# 安裝（選用）
sudo ninja -C builddir install
```

### 建置輸出

```
builddir/
├── xyz/
│   └── openbmc_project/
│       ├── Sensor/
│       │   ├── Value/
│       │   │   ├── server.hpp
│       │   │   └── server.cpp
│       │   └── Threshold/
│       │       └── ...
│       ├── State/
│       │   └── ...
│       └── ...
└── ...
```

---

## ⚙️ Meson 選項

### 可配置選項

| 選項 | 預設值 | 說明 |
|------|--------|------|
| `data_com_ibm` | `false` | 包含 IBM 專用介面 |
| `data_org_open_power` | `false` | 包含 Open Power 介面 |

### 啟用擴展命名空間

```bash
# 啟用 IBM 介面
meson builddir -Ddata_com_ibm=true

# 啟用 Open Power 介面
meson builddir -Ddata_org_open_power=true

# 同時啟用多個
meson builddir -Ddata_com_ibm=true -Ddata_org_open_power=true
```

### 重新設定

```bash
meson configure builddir -Ddata_com_ibm=true
ninja -C builddir
```

### 查看所有選項

```bash
meson configure builddir
```

---

## 📁 專案結構

```
phosphor-dbus-interfaces/
├── yaml/                    # YAML 介面定義
│   ├── xyz/
│   │   └── openbmc_project/
│   │       ├── Sensor/
│   │       ├── State/
│   │       └── ...
│   ├── org/
│   │   ├── freedesktop/
│   │   └── open_power/
│   └── com/
│       └── ibm/
├── gen/                     # 生成的 Meson 檔案
│   ├── meson.build
│   └── regenerate-meson    # 重新生成腳本
├── meson.build             # 主要建置設定
├── meson.options           # 建置選項
├── requirements.md         # 介面設計規範
└── README.md
```

---

## 🔄 新增或移除 YAML 檔案

當新增或移除 YAML 檔案時，需要重新生成 `gen/` 目錄下的 Meson 設定：

### 步驟

```bash
# 1. 新增或移除 YAML 檔案
vim yaml/xyz/openbmc_project/MyNew/Interface.interface.yaml

# 2. 重新生成 Meson 設定
cd gen
./regenerate-meson

# 3. 重新建置
cd ..
meson builddir --wipe  # 或者只是重新執行 ninja
ninja -C builddir
```

### regenerate-meson 腳本

這個 Python 腳本會：
1. 掃描 `yaml/` 目錄下所有 YAML 檔案
2. 生成對應的 `meson.build` 設定
3. 確保新介面被正確包含在建置中

---

## 🧪 開發環境設定

### OpenBMC 開發環境

如果您在 OpenBMC 開發環境中：

```bash
# 進入 OpenBMC 工作目錄
cd openbmc

# 透過 bitbake 建置
bitbake phosphor-dbus-interfaces
```

### 獨立開發

```bash
# 安裝 sdbusplus
git clone https://github.com/openbmc/sdbusplus.git
cd sdbusplus
meson builddir && ninja -C builddir && sudo ninja -C builddir install
cd tools && pip install .
cd ../..

# 設定環境變數
export PKG_CONFIG_PATH=/usr/local/lib/pkgconfig:$PKG_CONFIG_PATH

# 建置 phosphor-dbus-interfaces
cd phosphor-dbus-interfaces
meson builddir && ninja -C builddir
```

---

## 📦 使用生成的程式碼

### 在其他專案中

在 `meson.build` 中加入依賴：

```meson
phosphor_dbus_interfaces = dependency('phosphor-dbus-interfaces')

executable('my-app',
    'main.cpp',
    dependencies: [
        phosphor_dbus_interfaces,
        sdbusplus,
    ]
)
```

### 包含標頭檔

```cpp
// 感測器值介面
#include <xyz/openbmc_project/Sensor/Value/server.hpp>

// 主機狀態介面
#include <xyz/openbmc_project/State/Host/server.hpp>

// 錯誤定義
#include <xyz/openbmc_project/Common/error.hpp>
```

---

## 🐛 常見問題

### 找不到 sdbusplus

```
FAILED: xyz/openbmc_project/Sensor/Value/server.hpp
Program 'sdbus++' not found or not executable
```

**解決方案**：安裝 sdbusplus 並確保 sdbus++ 在 PATH 中

```bash
pip install mako inflection pyyaml
cd sdbusplus/tools && pip install .
```

### 缺少 Python 模組

```
ModuleNotFoundError: No module named 'mako'
```

**解決方案**：安裝 Python 依賴

```bash
pip install mako inflection pyyaml
```

### YAML 語法錯誤

```
yaml.scanner.ScannerError: while scanning a simple key
```

**解決方案**：檢查 YAML 檔案的縮排和語法

```bash
python3 -c "import yaml; yaml.safe_load(open('file.yaml'))"
```

---

## 📊 建置選項摘要

```bash
# 基本建置
meson builddir && ninja -C builddir

# 包含所有命名空間
meson builddir -Ddata_com_ibm=true -Ddata_org_open_power=true

# 清理重建
meson builddir --wipe

# 只重新編譯
ninja -C builddir

# 安裝
sudo ninja -C builddir install

# 解除安裝
sudo ninja -C builddir uninstall
```

---

## 🔍 延伸閱讀

- [CodeGeneration](CodeGeneration.md) - sdbus++ 程式碼生成詳解
- [YAMLFormat](YAMLFormat.md) - YAML 格式說明
- [sdbusplus](https://github.com/openbmc/sdbusplus) - 綁定生成工具

---

*最後更新：2025-12-19*
