# OpenBMC dbus-sensors 技術文件

## 歡迎

歡迎來到 OpenBMC dbus-sensors 技術文件。本文件提供完整的技術指南，涵蓋架構設計、感測器類型、API 參考、設定指南以及故障排除。

## 專案概述

**dbus-sensors** 是 OpenBMC 中負責讀取感測器數值的應用程式集合，提供 `xyz.openbmc_project.Sensor` 系列介面。它從 hwmon、D-Bus 或直接驅動程式存取讀取感測器數值。

### 🚀 主要功能

| 功能 | 說明 |
|------|------|
| **執行時期可重新配置** | 透過 D-Bus（Entity-Manager 或類似工具）動態配置 |
| **隔離式架構** | 每種感測器類型獨立於自己的守護程式，單一感測器的錯誤不會影響其他感測器 |
| **非同步單執行緒** | 使用 sdbusplus/asio 綁定 |
| **多資料來源** | 支援 hwmon、D-Bus、直接驅動程式存取 |

### 支援的感測器類型

- 🌡️ 溫度感測器（TMP75、TMP441、LM75A 等）
- ⚡ 電壓/電流/功率感測器（INA219、ADM1275 等）
- 🌀 風扇轉速感測器（Aspeed、Nuvoton、I2C）
- 💻 CPU 溫度感測器（Intel PECI）
- 💾 NVMe 溫度感測器
- 📡 外部感測器（IPMI/Redfish 推送）
- 🔌 ADC 感測器
- 🚨 機箱入侵感測器

---

## 快速開始

### 系統架構流程

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  Entity-Manager │────▶│  dbus-sensors   │────▶│  D-Bus          │
│  (配置發布)      │     │  (讀取數值)      │     │  (發布數值)      │
└─────────────────┘     └─────────────────┘     └─────────────────┘
        │                       │                       │
        ▼                       ▼                       ▼
    發布 Configuration      讀取 hwmon              提供給 Redfish
    介面到 D-Bus            或直接驅動程式           IPMI、PID 控制
```

### 資料流向

1. **Entity-Manager** 發布 `xyz.openbmc_project.Configuration.<Type>` 介面
2. **dbus-sensors** 守護程式監聽對應的配置類型
3. **dbus-sensors** 從 hwmon 或其他資料來源讀取數值
4. **dbus-sensors** 將數值發布到 `xyz.openbmc_project.Sensor.Value` 介面
5. **消費者**（Redfish、IPMI、PID 控制）讀取感測器數值

---

## 文件目錄

### 🏗️ 核心文件

| 文件 | 說明 |
|------|------|
| [架構概述](Architecture.md) | 系統設計、元件互動、Reactor 模式 |
| [感測器類型總覽](SensorTypes.md) | 所有支援的感測器類型與單位 |

### 🔧 感測器類型文件

| 文件 | 說明 |
|------|------|
| [ADC 感測器](ADCSensor.md) | 類比數位轉換器感測器 |
| [Hwmon 溫度感測器](HwmonTempSensor.md) | I2C 溫度感測器晶片 |
| [PSU 感測器](PSUSensor.md) | 電源供應器感測器（PMBus） |
| [風扇感測器](FanSensor.md) | 風扇轉速與 PWM 控制 |
| [CPU 感測器](CPUSensor.md) | Intel PECI CPU 溫度 |
| [NVMe 感測器](NVMeSensor.md) | NVMe SSD 溫度 |
| [外部感測器](ExternalSensor.md) | 外部來源推送的虛擬感測器 |
| [IPMB 感測器](IpmbSensor.md) | IPMB 協定感測器 |
| [機箱入侵感測器](ChassisIntrusionSensor.md) | 機箱開啟偵測 |

### 📡 API 參考

| 文件 | 說明 |
|------|------|
| [D-Bus API](DBusAPI.md) | 物件路徑、介面、屬性、訊號 |
| [設定指南](ConfigurationGuide.md) | JSON 結構、必填欄位、範本變數 |
| [閾值設定](ThresholdConfiguration.md) | 警告/嚴重閾值配置 |

### 🔌 整合文件

| 文件 | 說明 |
|------|------|
| [Entity-Manager 整合](EntityManagerIntegration.md) | Reactor 模式與配置發現 |
| [故障排除](Troubleshooting.md) | 常見問題、除錯指令、日誌分析 |

---

## 相關連結

### 官方資源

- 🔗 [OpenBMC dbus-sensors GitHub](https://github.com/openbmc/dbus-sensors)
- 📚 [phosphor-dbus-interfaces](https://github.com/openbmc/phosphor-dbus-interfaces)
- 🔧 [Entity-Manager 儲存庫](https://github.com/openbmc/entity-manager)

### 消費者應用程式

- [bmcweb Redfish](https://github.com/openbmc/bmcweb) - Redfish API 感測器端點
- [phosphor-pid-control](https://github.com/openbmc/phosphor-pid-control) - PID 風扇控制
- [phosphor-host-ipmid](https://github.com/openbmc/phosphor-host-ipmid) - IPMI SDR

### 相關文件

- [Entity-Manager 技術文件](../entity-manager/Home.md) - 硬體配置管理

---

## 版本資訊

- **文件版本**：1.0.0
- **最後更新**：2025-12-19
- **適用版本**：OpenBMC master 分支

---

> 💡 **提示**：建議從 [架構概述](Architecture.md) 開始閱讀，以全面了解 dbus-sensors 的設計理念和運作方式。
