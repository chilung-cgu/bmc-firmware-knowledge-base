# OpenBMC Entity-Manager 技術文件

## 歡迎

歡迎來到 OpenBMC Entity-Manager 技術文件。本文件提供完整的技術指南，涵蓋架構設計、API 參考、設定指南以及故障排除。

## 專案概述

**Entity-Manager** 是 OpenBMC 中負責管理實體系統元件的核心服務，它將硬體元件映射到 BMC（基板管理控制器）中的軟體資源。其主要設計目標包括：

- 🚀 **最小化移植時間**：減少將 OpenBMC 移植到新系統所需的時間和除錯工作
- 🔧 **減少程式碼差異**：降低不同平台間的程式碼差異
- 📊 **長期可維護性**：確保系統在數百個平台和元件間的可維護性

## 快速開始

### 基本架構流程

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  Detection      │────▶│  Entity-Manager │────▶│  Reactor        │
│  Daemon         │     │                 │     │  Daemon         │
│  (fru-device)   │     │  (配置解析)      │     │  (dbus-sensors) │
└─────────────────┘     └─────────────────┘     └─────────────────┘
       │                        │                       │
       ▼                        ▼                       ▼
   掃描 I2C             解析 JSON 配置            建立感測器
   發現 FRU             發布到 D-Bus              讀取數值
```

### 典型使用流程

1. **偵測守護程式**（如 `fru-device`）掃描硬體並發布至 D-Bus
2. **Entity-Manager** 根據 Probe 規則匹配配置檔
3. **Entity-Manager** 將 Exposes 記錄發布到 D-Bus
4. **反應器守護程式**（如 `dbus-sensors`）讀取配置並建立感測器

---

## 📖 建議閱讀路徑

### 🚀 新手入門

1. [架構概述](Architecture.md) - 了解系統設計
2. [核心概念](CoreConcepts.md) - 學習 Entity、Exposes、Probe
3. [設定指南](ConfigurationGuide.md) - 開始撰寫配置

### 🔧 平台開發者

1. [架構概述](Architecture.md) - 了解整體架構
2. [Probe 語法](ProbeSyntax.md) - 掌握探測語法
3. [設定範例](ExampleConfigurations.md) - 參考實際範例
4. [FruDevice 守護程式](FruDevice.md) - I2C/FRU 整合
5. [dbus-sensors 整合](DbusSensorsIntegration.md) - 感測器配置

### 🔍 問題排查

1. [故障排除](Troubleshooting.md) - 診斷問題
2. [D-Bus API](DBusAPI.md) - 驗證物件與屬性

---

## 文件目錄

### 🏗️ 核心文件

| 文件                        | 說明                              |
| --------------------------- | --------------------------------- |
| [架構概述](Architecture.md) | 系統設計、元件互動、設計目標      |
| [核心概念](CoreConcepts.md) | Entity、Exposes、Probe 定義與範例 |

### 📡 API 參考

| 文件                    | 說明                       |
| ----------------------- | -------------------------- |
| [D-Bus API](DBusAPI.md) | 物件路徑、介面、屬性、方法 |

### ⚙️ 設定指南

| 文件                                 | 說明                                |
| ------------------------------------ | ----------------------------------- |
| [設定指南](ConfigurationGuide.md)    | JSON 結構、必填欄位、範本變數       |
| [Probe 語法](ProbeSyntax.md)         | 探測語法、AND/OR 運算、已知限制     |
| [設定範例](ExampleConfigurations.md) | 溫度感測器、PCIe 卡、電源供應器範例 |

### 🔌 元件文件

| 文件                                           | 說明                                   |
| ---------------------------------------------- | -------------------------------------- |
| [FruDevice 守護程式](FruDevice.md)             | I2C 掃描、IPMI FRU EEPROM 解析         |
| [dbus-sensors 整合](DbusSensorsIntegration.md) | hwmon 感測器守護程式整合               |
| [關聯性](Associations.md)                      | 實體拓撲建模、containing/powering 關聯 |
| [EEPROM 偵測](EEPROMDetection.md)              | 位址大小偵測模式 (MODE-1/MODE-2)       |

### 🧬 深度走讀

| 文件                                         | 說明                                                   |
| -------------------------------------------- | ------------------------------------------------------ |
| [Source Code 走讀](SourceCodeWalkthrough.md) | main() → PerformScan → doProbe → postToDbus 完整呼叫鏈 |
| [JSON Schema 驗證](Schemas.md)               | 24 個 schema 檔案、驗證機制、新增類型步驟              |

### 🛠️ 操作指南

| 文件                              | 說明                                 |
| --------------------------------- | ------------------------------------ |
| [故障排除](Troubleshooting.md)    | 常見問題、除錯指令、日誌分析         |
| [相容軟體](CompatibleSoftware.md) | bmcweb、intel-ipmi-oem、偵測守護程式 |

---

## 相關連結

### 官方資源

- 🔗 [OpenBMC Entity-Manager GitHub](https://github.com/openbmc/entity-manager)
- 📚 [OpenBMC 官方文件](https://github.com/openbmc/docs)
- 🔧 [dbus-sensors 儲存庫](https://github.com/openbmc/dbus-sensors)

### 相關專案

- [phosphor-dbus-interfaces](https://github.com/openbmc/phosphor-dbus-interfaces) - D-Bus 介面定義
- [bmcweb](https://github.com/openbmc/bmcweb) - Redfish API 實作
- [peci-pcie](https://github.com/openbmc/peci-pcie) - PCIe 裝置偵測
- [smbios-mdr](https://github.com/openbmc/smbios-mdr) - SMBIOS 表格解析

---

## 版本資訊

- **文件版本**：1.0.0
- **最後更新**：2025-12-19
- **適用版本**：OpenBMC master 分支

---

> 💡 **提示**：建議從 [架構概述](Architecture.md) 開始閱讀，以全面了解 Entity-Manager 的設計理念和運作方式。
