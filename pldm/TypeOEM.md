# PLDM Type 63: OEM

OEM Type 提供廠商自訂的 PLDM 命令與功能擴充機制。

---

## 概述

| 欄位          | 值           |
| ------------- | ------------ |
| **Type Code** | 0x3F (63)    |
| **規範**      | 廠商自訂     |
| **功能**      | 廠商特定功能 |

---

## OEM 機制

PLDM 提供兩種 OEM 擴充方式：

1. **Type 63**: 完全自訂的 OEM Type
2. **OEM 命令**: 在既有 Type 中使用保留的命令範圍

### 命令範圍

| 範圍         | 說明                      |
| ------------ | ------------------------- |
| 0x00-0x3F    | 標準命令                  |
| 0xF0-0xFE    | OEM 命令 (在既有 Type 中) |
| Type 63 全部 | OEM Type 專用             |

---

## OpenBMC OEM 結構

### 目錄組織

```
pldm/oem/
├── ampere/                # Ampere OEM
│   └── oem_ampere.hpp     # Ampere 初始化
├── ibm/                   # IBM OEM
│   ├── libpldmresponder/  # OEM Handler
│   │   ├── oem_ibm_handler.cpp/hpp
│   │   ├── file_io.cpp/hpp
│   │   └── ...
│   ├── configurations/    # OEM 配置
│   │   └── bios/          # BIOS 屬性
│   └── pldmtool/          # OEM pldmtool 命令
│       └── oem_ibm_cmd.cpp
├── meta/                  # Meta OEM
│   ├── oem_meta.cpp/hpp   # Meta OEM Handler
│   └── utils.cpp/hpp      # Meta 工具函式
└── nvidia/                # NVIDIA OEM
    └── oem_nvidia.hpp     # NVIDIA 初始化
```

---

## 新增 OEM 支援

### 步驟 1: 建立目錄

```bash
mkdir -p oem/<vendor>/libpldmresponder
mkdir -p oem/<vendor>/configurations
mkdir -p oem/<vendor>/pldmtool
```

### 步驟 2: 實作 Handler

```cpp
// oem/<vendor>/libpldmresponder/oem_<vendor>_handler.hpp
#pragma once
#include "libpldmresponder/oem_handler.hpp"

namespace pldm::responder::oem_<vendor> {

class Handler : public oem_platform::Handler {
public:
    // 實作 OEM 命令處理
    Response handleOemCommand(Request request, size_t payloadLength);
};

} // namespace
```

### 步驟 3: 新增 Meson 選項

```meson
# meson.options
option('oem-<vendor>', type: 'feature', value: 'disabled',
       description: 'Enable OEM <vendor> support')
```

### 步驟 4: 更新建置

```meson
# meson.build
if get_option('oem-<vendor>').allowed()
    add_project_arguments('-DOEM_<VENDOR>', language: 'cpp')
    subdir('oem/<vendor>')
endif
```

---

## OEM Handler 介面

```cpp
// libpldmresponder/oem_handler.hpp
namespace pldm::responder::oem_platform {

class Handler {
public:
    virtual ~Handler() = default;

    // 處理 OEM 狀態 Effecter
    virtual int setOemEffecter(
        uint16_t effecterId,
        uint8_t compositeEffecterCount,
        const std::vector<set_effecter_state_field>& stateField,
        uint16_t effecterIntf);

    // 處理 OEM 事件
    virtual int processOemEvent(uint8_t eventType,
                                const uint8_t* eventData,
                                size_t eventDataLen);

    // 取得 OEM PDR
    virtual void buildOemPDR(pdr_utils::Repo& repo);
};

} // namespace
```

---

## 各廠商 OEM 實作對比

> ⚠️ **簡化說明**：以下表格僅包含 upstream source code 中實際存在的 OEM。

| 功能                 | IBM          | Ampere | Meta | NVIDIA |
| -------------------- | ------------ | ------ | ---- | ------ |
| OEM Platform Handler | ✅ 完整      | ✅     | ✅   | ✅     |
| OEM FRU Handler      | ✅           | -      | -    | -      |
| OEM BIOS Handler     | ✅           | -      | -    | -      |
| OEM Utils Handler    | ✅           | -      | ✅   | -      |
| File I/O (Type 63)   | ✅           | -      | -    | -      |
| pldmtool 子命令      | ✅ `oem-ibm` | -      | -    | -      |
| OEM PDR              | ✅           | ✅     | -    | -      |
| Lamp Test            | ✅           | -      | -    | -      |

---

## pldmtool OEM 命令

```bash
# 使用 OEM-IBM 命令 (如果啟用)
$ pldmtool oem-ibm GetFileTable

# 查看 OEM 命令幫助
$ pldmtool oem-ibm -h
```

---

## 建置 OEM 支援

```bash
# 啟用 IBM OEM
meson setup build -Doem-ibm=enabled

# 啟用 Ampere OEM
meson setup build -Doem-ampere=enabled

# 啟用 Meta OEM
meson setup build -Doem-meta=enabled

# 啟用 NVIDIA OEM
meson setup build -Doem-nvidia=enabled

# 停用所有 OEM
meson setup build -Doem-ibm=disabled -Doem-ampere=disabled -Doem-meta=disabled -Doem-nvidia=disabled

# 建置
meson compile -C build
```

---

## 相關文件

- [OEMExtension](OEMExtension.md) - OEM 擴充開發指南
- [CodeOrganization](CodeOrganization.md) - 程式碼組織

---

_返回 [Home](Home.md)_
