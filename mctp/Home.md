# CodeConstruct/mctp Technical Wiki 🔗

歡迎來到 **CodeConstruct/mctp** 技術文件！本 Wiki 提供 MCTP（Management Component Transport Protocol）使用者空間工具的完整技術參考。

---

## 📖 專案簡介

[CodeConstruct/mctp](https://github.com/CodeConstruct/mctp) 是一套用於管理 Linux 核心 MCTP 堆疊的使用者空間工具，主要用於 OpenBMC 平台。專案包含兩個核心元件：

| 元件      | 說明                                                                       |
| --------- | -------------------------------------------------------------------------- |
| **mctp**  | 命令行工具，用於查詢和管理核心 MCTP 堆疊狀態，類似 `iproute2` 的 `ip` 命令 |
| **mctpd** | 守護程式，實作 MCTP 控制協議，作為 bus-owner 分配 EID 給遠端端點           |

### 主要功能

- 🔧 管理 MCTP 網路介面（link、address、route、neigh）
- 🆔 動態分配端點識別碼（EID）
- 🌉 支援 MCTP 橋接器模式
- 📡 透過 D-Bus 暴露端點資訊
- 🔄 相容 OpenBMC 標準介面規範

---

## 📖 建議閱讀路徑

### 🚀 新手入門

1. [QuickStart](QuickStart.md) - 快速開始使用
2. [MCTPOverview](MCTPOverview.md) - 了解 MCTP 協議基礎
3. [Architecture](Architecture.md) - 認識系統架構

### 🔧 開發者路線

1. [Architecture](Architecture.md) - 系統設計
2. [KernelStack](KernelStack.md) - 核心 API 與 Socket
3. [MctpCommand](MctpCommand.md) - 命令行工具
4. [MctpdDaemon](MctpdDaemon.md) - 守護程式配置
5. [SourceCodeWalkthrough](SourceCodeWalkthrough.md) - mctpd 完整呼叫鏈走讀
6. [DBusOverview](DBusOverview.md) → [InterfaceAPI](InterfaceAPI.md) → [EndpointAPI](EndpointAPI.md) - D-Bus API

### 🌉 進階功能

1. [EndpointDiscovery](EndpointDiscovery.md) - 端點發現機制
2. [BridgeMode](BridgeMode.md) - 橋接器與 EID 池
3. [OpenBMCIntegration](OpenBMCIntegration.md) - 整合其他 OpenBMC 服務

### 🔍 問題排查

1. [Troubleshooting](Troubleshooting.md) - 診斷問題
2. [TestTools](TestTools.md) - 除錯工具

---

## 📚 文件導覽

### 核心概念

| 文件                            | 說明                          |
| ------------------------------- | ----------------------------- |
| [Architecture](Architecture.md) | 系統架構與設計                |
| [MCTPOverview](MCTPOverview.md) | MCTP 協議概述（DSP0236 標準） |
| [KernelStack](KernelStack.md)   | Linux 核心 MCTP 堆疊          |

### 工具參考

| 文件                          | 說明                      |
| ----------------------------- | ------------------------- |
| [MctpCommand](MctpCommand.md) | `mctp` 命令行工具完整參考 |
| [MctpdDaemon](MctpdDaemon.md) | `mctpd` 守護程式詳解      |
| [TestTools](TestTools.md)     | 測試與除錯工具            |

### D-Bus API

| 文件                            | 說明                                    |
| ------------------------------- | --------------------------------------- |
| [DBusOverview](DBusOverview.md) | D-Bus 介面總覽與物件樹                  |
| [InterfaceAPI](InterfaceAPI.md) | MCTP 介面 API（Interface1 / BusOwner1） |
| [NetworkAPI](NetworkAPI.md)     | 網路 API（Network1）                    |
| [EndpointAPI](EndpointAPI.md)   | 端點 API（Endpoint / Endpoint1）        |
| [BridgeAPI](BridgeAPI.md)       | 橋接器 API（Bridge1）                   |

### 配置與部署

| 文件                                        | 說明                    |
| ------------------------------------------- | ----------------------- |
| [Configuration](Configuration.md)           | `mctpd.conf` 配置指南   |
| [SystemdIntegration](SystemdIntegration.md) | Systemd 服務整合        |
| [OpenBMCIntegration](OpenBMCIntegration.md) | 與 OpenBMC 其他元件整合 |

### 使用指南

| 文件                                      | 說明              |
| ----------------------------------------- | ----------------- |
| [QuickStart](QuickStart.md)               | 快速入門          |
| [EndpointDiscovery](EndpointDiscovery.md) | 端點發現流程      |
| [BridgeMode](BridgeMode.md)               | 橋接模式與 EID 池 |

### 進階主題

| 文件                                              | 說明                        |
| ------------------------------------------------- | --------------------------- |
| [Troubleshooting](Troubleshooting.md)             | 故障排除                    |
| [VersionHistory](VersionHistory.md)               | 版本歷史                    |
| [SourceCodeWalkthrough](SourceCodeWalkthrough.md) | mctpd main() 完整呼叫鏈走讀 |

---

## 🚀 快速開始

### 編譯安裝

```bash
# 配置和編譯
meson setup obj
ninja -C obj

# 安裝（可選 DESTDIR）
meson install -C obj
```

### 基本使用

```bash
# 查看 MCTP 介面
mctp link show

# 啟動 mctpd（作為 bus-owner）
mctpd

# 透過 D-Bus 設定端點
busctl call au.com.codeconstruct.MCTP1 \
    /au/com/codeconstruct/mctp1/interfaces/mctpi2c1 \
    au.com.codeconstruct.MCTP.BusOwner1 \
    SetupEndpoint ay 1 0x1d
```

---

## 📋 專案資訊

| 項目       | 內容                                            |
| ---------- | ----------------------------------------------- |
| **版本**   | v2.4（2025-10-28）                              |
| **授權**   | GPL-2.0                                         |
| **維護者** | [Code Construct](https://codeconstruct.com.au/) |
| **原始碼** | [GitHub](https://github.com/CodeConstruct/mctp) |
| **相依性** | libsystemd (≥247)、meson (≥0.59.0)              |

---

## 🔗 相關連結

- [DMTF MCTP 規範（DSP0236）](https://www.dmtf.org/standards/mctp)
- [Linux Kernel MCTP 文件](https://docs.kernel.org/networking/mctp.html)
- [OpenBMC MCTP 設計文件](https://github.com/openbmc/docs/blob/master/designs/mctp/mctp-kernel.md)
- [phosphor-dbus-interfaces MCTP 介面](https://github.com/openbmc/phosphor-dbus-interfaces/tree/master/yaml/xyz/openbmc_project/MCTP)

---

_最後更新：2025-12-19_
