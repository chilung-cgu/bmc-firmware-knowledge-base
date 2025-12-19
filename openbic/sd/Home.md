# OpenBIC yv4-sd (Sentinel Dome) Technical Wiki 📚

歡迎來到 OpenBIC Sentinel Dome (yv4-sd) Bridge IC 的技術文件 Wiki。

---

## 🎯 快速導覽

| 文件 | 說明 |
|------|------|
| [Architecture](Architecture.md) | 系統架構總覽 |
| [QuickStart](QuickStart.md) | 開發環境設置與編譯指南 |
| [CodeOrganization](CodeOrganization.md) | 程式碼目錄結構 |

---

## 📋 專案概述

### 什麼是 OpenBIC？

**OpenBIC** 是 Meta/Facebook 開發的開源韌體框架，用於建構 Bridge IC (BIC) 的完整韌體映像。BIC 是部署在資料中心的獨立裝置，能夠使用單一 BMC 裝置監控多主機系統。

### Sentinel Dome (SD) BIC

在 **Yosemite V4** 平台中，SD BIC 扮演 Host 與 BMC 之間的**中間橋樑**角色：

```
┌─────────────────────────────────────────────────────────────────┐
│                      Yosemite V4 架構                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│    ┌──────────┐      ┌──────────┐      ┌──────────┐            │
│    │   Host   │      │  SD BIC  │      │   BMC    │            │
│    │  (CPU)   │◄────►│ (Bridge) │◄────►│          │            │
│    └──────────┘      └──────────┘      └──────────┘            │
│         │                  │                                    │
│         │            ┌─────┴─────┐                              │
│         │            │           │                              │
│         ▼            ▼           ▼                              │
│    ┌──────────┐ ┌──────────┐ ┌──────────┐                      │
│    │ FF BIC   │ │ WF BIC   │ │ Sensors  │                      │
│    │          │ │          │ │ VR, Temp │                      │
│    └──────────┘ └──────────┘ └──────────┘                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 關鍵特性

| 特性 | 說明 |
|------|------|
| **硬體晶片** | ASPEED AST1030 |
| **RTOS** | Zephyr OS |
| **通訊協議** | MCTP、PLDM、IPMI/IPMB |
| **介面** | I2C、I3C、GPIO、SPI、KCS、eSPI |

---

## 📂 文件分類

### 硬體與驅動

| 文件 | 說明 |
|------|------|
| [AST1030Overview](AST1030Overview.md) | ASPEED AST1030 晶片介紹 |
| [GPIO](GPIO.md) | GPIO 管理與中斷處理 |
| [I2C_I3C](I2C_I3C.md) | I2C/I3C 介面使用 |
| [SensorFramework](SensorFramework.md) | 感測器框架設計 |
| [DeviceDrivers](DeviceDrivers.md) | 裝置驅動程式 |

### 通訊協議

| 文件 | 說明 |
|------|------|
| [MCTPOverview](MCTPOverview.md) | MCTP 傳輸協議 |
| [PLDMOverview](PLDMOverview.md) | PLDM 平台管理協議 |
| [IPMIOverview](IPMIOverview.md) | IPMI 命令處理 |
| [IPMB](IPMB.md) | IPMB 智慧平台管理匯流排 |
| [KCS](KCS.md) | KCS 介面 |

### 平台特定

| 文件 | 說明 |
|------|------|
| [YV4SDPlatform](YV4SDPlatform.md) | Yosemite V4 SD 平台概述 |
| [PlatformInit](PlatformInit.md) | 平台初始化流程 |
| [SlotDetection](SlotDetection.md) | Slot 偵測機制 |
| [PowerManagement](PowerManagement.md) | 電源管理 |
| [FirmwareUpdate](FirmwareUpdate.md) | 韌體更新 |

### 進階主題

| 文件 | 說明 |
|------|------|
| [PLDMSensor](PLDMSensor.md) | PLDM 感測器 |
| [PLDMMonitor](PLDMMonitor.md) | PLDM 監控 |
| [InterruptHandling](InterruptHandling.md) | 中斷服務處理 |
| [OEMCommands](OEMCommands.md) | OEM 命令實作 |
| [Troubleshooting](Troubleshooting.md) | 故障排除 |
| [APIReference](APIReference.md) | API 參考 |

---

## 🔗 相關資源

### 官方資源

- [OpenBIC GitHub](https://github.com/facebook/OpenBIC)
- [OpenBIC 官方文件](https://facebook.github.io/OpenBIC/)
- [Zephyr Project](https://www.zephyrproject.org/)
- [ASPEED Technology](https://www.aspeedtech.com/)

### DMTF 規範

- [MCTP Base Specification (DSP0236)](https://www.dmtf.org/sites/default/files/standards/documents/DSP0236_1.3.0.pdf)
- [MCTP SMBus/I2C Transport Binding (DSP0237)](https://www.dmtf.org/sites/default/files/standards/documents/DSP0237_1.2.0.pdf)
- [PLDM Base Specification (DSP0240)](https://www.dmtf.org/sites/default/files/standards/documents/DSP0240_1.1.0.pdf)
- [PLDM for Platform Monitoring (DSP0248)](https://www.dmtf.org/sites/default/files/standards/documents/DSP0248_1.2.0.pdf)

### OpenBMC 相關

- [OpenBMC GitHub](https://github.com/openbmc)
- [OpenBMC pldm](https://github.com/openbmc/pldm)
- [OpenBMC mctp](https://github.com/CodeConstruct/mctp)

---

## 📖 建議閱讀順序

如果您剛開始接觸 OpenBIC，建議按照以下順序閱讀：

1. **[Architecture](Architecture.md)** - 了解整體系統架構
2. **[QuickStart](QuickStart.md)** - 設置開發環境
3. **[CodeOrganization](CodeOrganization.md)** - 熟悉程式碼結構
4. **[YV4SDPlatform](YV4SDPlatform.md)** - 了解 SD 平台特性
5. **[MCTPOverview](MCTPOverview.md)** & **[PLDMOverview](PLDMOverview.md)** - 核心通訊協議

### 🗺️ 學習路線圖

```
┌─────────────────────────────────────────────────────────────────┐
│                    OpenBIC 學習路線                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Level 1: 基礎入門                                               │
│  ┌─────────────────────┐    ┌─────────────────────┐             │
│  │ Architecture        │───►│ QuickStart          │             │
│  │ (系統架構)           │    │ (開發環境)           │             │
│  └─────────────────────┘    └─────────┬───────────┘             │
│                                       │                          │
│  Level 2: 硬體層                       ▼                          │
│  ┌─────────────────────┐    ┌─────────────────────┐             │
│  │ AST1030Overview     │───►│ GPIO / I2C_I3C      │             │
│  │ (晶片介紹)           │    │ (介面操作)           │             │
│  └─────────────────────┘    └─────────┬───────────┘             │
│                                       │                          │
│  Level 3: 感測器與服務                 ▼                          │
│  ┌─────────────────────┐    ┌─────────────────────┐             │
│  │ SensorFramework     │───►│ DeviceDrivers       │             │
│  │ (感測器框架)         │    │ (裝置驅動)           │             │
│  └─────────────────────┘    └─────────┬───────────┘             │
│                                       │                          │
│  Level 4: 通訊協議                     ▼                          │
│  ┌─────────────────────┐    ┌─────────────────────┐             │
│  │ MCTPOverview        │───►│ PLDMOverview        │             │
│  │ (傳輸層)             │    │ (平台管理協議)       │             │
│  └─────────────────────┘    └─────────┬───────────┘             │
│                                       │                          │
│  Level 5: 平台整合                     ▼                          │
│  ┌─────────────────────┐    ┌─────────────────────┐             │
│  │ PlatformInit        │───►│ InterruptHandling   │             │
│  │ (初始化流程)         │    │ (中斷處理)           │             │
│  └─────────────────────┘    └─────────────────────┘             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 🎯 針對不同角色的建議

| 角色 | 建議路徑 |
|------|----------|
| **BIC 韌體開發者** | Architecture → QuickStart → CodeOrganization → SensorFramework → 全部 |
| **協議開發者** | MCTPOverview → PLDMOverview → IPMIOverview → OEMCommands |
| **平台整合者** | YV4SDPlatform → PlatformInit → SlotDetection → PowerManagement |
| **問題排查者** | Troubleshooting → APIReference → InterruptHandling → GPIO |
| **硬體工程師** | AST1030Overview → GPIO → I2C_I3C → SensorFramework |

---

## 🏷️ 版本資訊

| 項目 | 內容 |
|------|------|
| **平台名稱** | Yosemite V4 |
| **專案名稱** | Sentinel Dome |
| **專案階段** | MP |
| **Board ID** | 0x01 |
| **韌體標識** | sd |

---

*最後更新：2025-12-19*
