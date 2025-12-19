# bmcweb 技術 Wiki

歡迎來到 OpenBMC **bmcweb** 技術文件！bmcweb 是 OpenBMC 的核心 Web 伺服器，提供 Redfish REST API、D-Bus REST API、WebSocket 介面以及 KVM 功能。

---

## 📖 專案概述

| 項目 | 說明 |
|------|------|
| **Repository** | [openbmc/bmcweb](https://github.com/openbmc/bmcweb) |
| **語言** | C++ (C++20) |
| **建置系統** | Meson |
| **授權** | Apache 2.0 |
| **主要功能** | Redfish API, D-Bus REST, KVM, Serial Console |

bmcweb 是一個「全功能」嵌入式 Web 伺服器，整合了 BMC 管理所需的各種介面：

- **Redfish REST API** - 符合 DMTF DSP0266 標準的伺服器管理介面
- **D-Bus REST API** - 直接存取 OpenBMC D-Bus 物件的低階介面
- **WebSocket 介面** - 包含 D-Bus 事件、Serial Console、KVM
- **多重認證** - Basic、Session、mTLS、XToken 認證機制

---

## 🗂️ 文件導覽

### 核心概念
| 文件 | 說明 |
|------|------|
| [Architecture](Architecture.md) | 系統架構與設計 |
| [CodeOrganization](CodeOrganization.md) | 原始碼結構 |
| [Configuration](Configuration.md) | 編譯與配置選項 |

### Redfish API
| 文件 | 說明 |
|------|------|
| [RedfishOverview](RedfishOverview.md) | Redfish 協議概述 |
| [RedfishSchema](RedfishSchema.md) | 支援的 Schema 清單 |
| [RedfishAccountService](RedfishAccountService.md) | 帳戶服務 API |
| [RedfishChassisService](RedfishChassisService.md) | Chassis 服務 API |
| [RedfishManagerService](RedfishManagerService.md) | Manager 服務 API |
| [RedfishSystemService](RedfishSystemService.md) | System 服務 API |
| [RedfishEventService](RedfishEventService.md) | 事件訂閱服務 |
| [RedfishUpdateService](RedfishUpdateService.md) | 韌體更新服務 |
| [RedfishTelemetryService](RedfishTelemetryService.md) | 遙測服務 |
| [RedfishAggregation](RedfishAggregation.md) | Redfish 聚合功能 |

### HTTP/WebSocket 介面
| 文件 | 說明 |
|------|------|
| [HTTPServer](HTTPServer.md) | HTTP 伺服器實作 |
| [WebSocketDBus](WebSocketDBus.md) | D-Bus 事件 WebSocket |
| [WebSocketSerial](WebSocketSerial.md) | Serial Console WebSocket |
| [WebSocketKVM](WebSocketKVM.md) | KVM WebSocket |
| [DBusRESTAPI](DBusRESTAPI.md) | D-Bus REST API |

### 認證與授權
| 文件 | 說明 |
|------|------|
| [Authentication](Authentication.md) | 認證機制總覽 |
| [Authorization](Authorization.md) | 授權與權限控制 |
| [SessionManagement](SessionManagement.md) | Session 管理 |

### 使用指南
| 文件 | 說明 |
|------|------|
| [QuickStart](QuickStart.md) | 快速入門指南 |
| [Testing](Testing.md) | 測試方法 |
| [Troubleshooting](Troubleshooting.md) | 故障排除 |

---

## 🎯 建議閱讀路線

根據不同的學習目標，建議依照以下順序閱讀：

### 🔰 初學者路線
適合第一次接觸 bmcweb 的讀者：

```
1. Architecture.md     → 了解整體架構
2. RedfishOverview.md  → 理解 Redfish 標準
3. QuickStart.md       → 動手操作 API
4. Authentication.md   → 學習認證機制
```

### 🔧 開發者路線
適合需要擴展或修改 bmcweb 的開發者：

```
1. Architecture.md     → 了解設計理念
2. CodeOrganization.md → 熟悉程式碼結構
3. Configuration.md    → 了解編譯選項
4. Testing.md          → 掌握測試流程
5. RedfishSchema.md    → 學習如何新增 Schema
```

### 🔌 API 使用者路線
適合需要透過 Redfish 管理 BMC 的使用者：

```
1. QuickStart.md           → 快速入門
2. RedfishSystemService.md → 電源控制
3. RedfishChassisService.md → 感測器讀取
4. RedfishEventService.md  → 事件訂閱
5. Troubleshooting.md      → 問題排除
```

### 🌐 進階功能路線
適合需要使用進階功能的讀者：

```
1. RedfishAggregation.md   → 多 BMC 聚合
2. WebSocketKVM.md         → 遠端桌面
3. WebSocketSerial.md      → 串列主控台
4. RedfishTelemetryService.md → 遙測監控
```

---

## 🚀 快速開始

### 建立連線

```bash
# 設定 BMC IP
export bmc=192.168.1.100

# 取得認證 token
export token=$(curl -k -H "Content-Type: application/json" \
  -X POST https://${bmc}/login \
  -d '{"username":"root","password":"0penBmc"}' \
  | grep token | awk '{print $2;}' | tr -d '"')

# 查詢 Redfish Service Root
curl -k -H "X-Auth-Token: $token" https://${bmc}/redfish/v1
```

### 常用操作

```bash
# 查詢系統狀態
curl -k -H "X-Auth-Token: $token" https://${bmc}/redfish/v1/Systems/system

# 電源控制 - 開機
curl -k -H "X-Auth-Token: $token" -H "Content-Type: application/json" \
  -X POST https://${bmc}/redfish/v1/Systems/system/Actions/ComputerSystem.Reset \
  -d '{"ResetType": "On"}'

# 查詢感測器
curl -k -H "X-Auth-Token: $token" https://${bmc}/redfish/v1/Chassis/chassis/Sensors
```

---

## 🔗 相關連結

- [OpenBMC bmcweb GitHub](https://github.com/openbmc/bmcweb)
- [DMTF Redfish 規格](https://www.dmtf.org/standards/redfish)
- [OpenBMC 官方文件](https://github.com/openbmc/docs)
- [Redfish Cheatsheet](https://github.com/openbmc/docs/blob/master/REDFISH-cheatsheet.md)

---

## 📊 bmcweb 在 OpenBMC 中的角色

```
┌─────────────────────────────────────────────────────────────────┐
│                        外部客戶端                                │
│    (Redfish Client, WebUI, CLI Tools, Management Software)      │
└────────────────────────────┬────────────────────────────────────┘
                             │ HTTPS/WSS
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                         bmcweb                                   │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌────────────┐ │
│  │  Redfish    │ │ D-Bus REST  │ │  WebSocket  │ │    KVM     │ │
│  │    API      │ │    API      │ │   Handler   │ │  Handler   │ │
│  └──────┬──────┘ └──────┬──────┘ └──────┬──────┘ └─────┬──────┘ │
│         │               │               │              │        │
│  ┌──────┴───────────────┴───────────────┴──────────────┴──────┐ │
│  │                    D-Bus Interface                          │ │
│  └─────────────────────────────────────────────────────────────┘ │
└────────────────────────────┬────────────────────────────────────┘
                             │ D-Bus
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                    OpenBMC Services                              │
│  phosphor-state-manager, dbus-sensors, phosphor-logging, etc.   │
└─────────────────────────────────────────────────────────────────┘
```

---

*最後更新：2025-12-19*
