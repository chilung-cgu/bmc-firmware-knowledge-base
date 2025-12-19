# OpenBMC Technical Wiki 📚

這個儲存庫用於存放 OpenBMC 各個核心專案的技術文件（Technical Wiki），採用繁體中文撰寫，遵循 Google Code Wiki 風格。

---

## 📋 專案進度

### ✅ 已完成的 Repository

| Repository | 目錄 | 文件數 | 完成日期 | 說明 |
|------------|------|--------|----------|------|
| [entity-manager](https://github.com/openbmc/entity-manager) | [entity-manager/](entity-manager/) | 13 | 2025-12-19 | 執行時期硬體配置管理 |
| [dbus-sensors](https://github.com/openbmc/dbus-sensors) | [dbus-sensors/](dbus-sensors/) | 17 | 2025-12-19 | 感測器讀取守護程式 |
| [phosphor-dbus-interfaces](https://github.com/openbmc/phosphor-dbus-interfaces) | [phosphor-dbus-interfaces/](phosphor-dbus-interfaces/) | 20 | 2025-12-19 | D-Bus 介面定義 |
| [mctp](https://github.com/CodeConstruct/mctp) | [mctp/](mctp/) | 20 | 2025-12-19 | MCTP 使用者空間工具 |
| [pldm](https://github.com/openbmc/pldm) | [pldm/](pldm/) | 25 | 2025-12-19 | PLDM 平台管理協議 |
| [bmcweb](https://github.com/openbmc/bmcweb) | [bmcweb/](bmcweb/) | 25 | 2025-12-19 | Redfish REST API 實作 |


### 📝 計畫中的 Repository

> 💡 以下是建議學習的重要 Repository，依照理解 OpenBMC 架構的重要性排序。

#### 🔴 高優先（核心架構）

| Repository | 說明 | 理由 |
|------------|------|------|
| [sdbusplus](https://github.com/openbmc/sdbusplus) | C++ D-Bus 函式庫 | 理解 D-Bus bindings 生成機制，所有服務的基礎 |
| [phosphor-state-manager](https://github.com/openbmc/phosphor-state-manager) | 電源/系統狀態管理 | BMC、Host、Chassis 狀態機，核心控制流程 |
| [phosphor-logging](https://github.com/openbmc/phosphor-logging) | 事件日誌系統 | SEL、Redfish Event Log，除錯必備 |

#### 🟡 中優先（功能擴展）

| Repository | 說明 | 理由 |
|------------|------|------|
| [phosphor-host-ipmid](https://github.com/openbmc/phosphor-host-ipmid) | IPMI 守護程式 | 傳統 IPMI 介面，仍廣泛使用 |
| [phosphor-pid-control](https://github.com/openbmc/phosphor-pid-control) | PID 風扇控制 | 熱管理的核心元件 |
| [phosphor-hwmon](https://github.com/openbmc/phosphor-hwmon) | hwmon 感測器橋接 | dbus-sensors 的替代方案 |
| [phosphor-power](https://github.com/openbmc/phosphor-power) | 電源管理 | PSU 監控、電源順序控制 |

#### 🟢 低優先（特定功能）

| Repository | 說明 | 理由 |
|------------|------|------|
| [peci-pcie](https://github.com/openbmc/peci-pcie) | PCIe 裝置偵測 | Intel 平台 PECI 介面 |
| [smbios-mdr](https://github.com/openbmc/smbios-mdr) | SMBIOS 表格解析 | Host 硬體資訊讀取 |
| [phosphor-user-manager](https://github.com/openbmc/phosphor-user-manager) | 使用者管理 | 認證與授權 |
| [phosphor-certificate-manager](https://github.com/openbmc/phosphor-certificate-manager) | 憑證管理 | TLS/HTTPS 憑證 |


---

## 📂 目錄結構

```
~/notes/
├── README.md                      # 本文件 - 專案總覽與進度追蹤
├── entity-manager/                # Entity-Manager 技術文件 (13 篇)
│   ├── Home.md                   # 首頁與導覽
│   ├── Architecture.md           # 架構概述
│   └── ...                       # 更多文件
├── dbus-sensors/                  # dbus-sensors 技術文件 (17 篇)
│   ├── Home.md                   # 首頁與導覽
│   ├── Architecture.md           # 架構概述
│   ├── SensorTypes.md            # 感測器類型總覽
│   └── ...                       # 更多文件
├── phosphor-dbus-interfaces/      # phosphor-dbus-interfaces 技術文件 (20 篇)
│   ├── Home.md                   # 首頁與導覽
│   ├── Architecture.md           # 架構概述
│   ├── YAMLFormat.md             # YAML 介面定義格式
│   ├── Namespaces.md             # 命名空間結構
│   └── ...                       # 更多文件
├── mctp/                          # CodeConstruct/mctp 技術文件 (20 篇)
│   ├── Home.md                   # 首頁與導覽
│   ├── Architecture.md           # 架構概述
│   ├── MCTPOverview.md           # MCTP 協議概述
│   ├── MctpdDaemon.md            # mctpd 守護程式
│   ├── DBusOverview.md           # D-Bus 介面總覽
│   └── ...                       # 更多文件
├── pldm/                          # openbmc/pldm 技術文件 (25 篇)
│   ├── Home.md                   # 首頁與導覽
│   ├── Architecture.md           # 架構概述
│   ├── PLDMOverview.md           # PLDM 協議概述
│   ├── TypePlatform.md           # Platform M&C Type
│   ├── Pldmd.md                  # pldmd 守護程式
│   └── ...                       # 更多文件
├── bmcweb/                        # openbmc/bmcweb 技術文件 (25 篇)
│   ├── Home.md                   # 首頁與導覽
│   ├── Architecture.md           # 架構概述
│   ├── RedfishOverview.md        # Redfish 協議概述
│   ├── QuickStart.md             # 快速入門
│   └── ...                       # 更多文件
└── {repository}/                  # 未來新增的 repository
    └── ...

```

---

## 🎯 文件標準

每個 Repository Wiki 應包含：

1. **Home.md** - 首頁與導覽連結
2. **Architecture.md** - 系統架構與設計
3. **核心概念文件** - 解釋關鍵術語
4. **API 參考** - D-Bus 介面、方法、屬性
5. **設定指南** - 配置方式與範例
6. **故障排除** - 常見問題與解決方案

---

## 🚀 如何開始新的 Repository Wiki

1. 在 `~/notes/` 下建立新目錄：`{repository-name}/`
2. 研究該專案的官方文件和原始碼
3. 依照上述標準建立文件
4. 更新本 README 的進度表
5. 提交變更

---

## 🎓 OpenBMC 學習路徑建議

### 📚 基礎入門（建議順序）

想要理解 OpenBMC，建議按照以下順序學習：

| 順序 | Repository | 理由 |
|------|------------|------|
| 1️⃣ | [phosphor-dbus-interfaces](phosphor-dbus-interfaces/Home.md) | **D-Bus 契約是核心**。OpenBMC 所有服務透過 D-Bus 通訊，先理解介面定義才能理解其他元件 |
| 2️⃣ | [entity-manager](entity-manager/Home.md) | **硬體配置中心**。了解 OpenBMC 如何發現和描述硬體 |
| 3️⃣ | [dbus-sensors](dbus-sensors/Home.md) | **感測器讀取**。學習如何將硬體數值暴露為 D-Bus 物件 |
| 4️⃣ | [mctp](mctp/Home.md) | **傳輸層**。MCTP 是 PLDM 等協議的傳輸基礎 |
| 5️⃣ | [pldm](pldm/Home.md) | **平台管理協議**。學習 BMC 與 Host、外部裝置的標準化通訊 |

### 🗺️ 學習路線圖

```
┌─────────────────────────────────────────────────────────────────┐
│                    OpenBMC 學習路線                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Level 1: 基礎概念                                               │
│  ┌─────────────────────┐                                        │
│  │ phosphor-dbus-      │ ◄── 從這裡開始！                        │
│  │ interfaces          │     D-Bus 是所有通訊的基礎             │
│  └─────────┬───────────┘                                        │
│            │                                                    │
│  Level 2: 硬體管理                                               │
│  ┌─────────▼───────────┐    ┌─────────────────────┐             │
│  │ entity-manager      │────▶│ dbus-sensors        │             │
│  │ (配置管理)           │    │ (感測器讀取)         │             │
│  └─────────────────────┘    └─────────────────────┘             │
│                                                                 │
│  Level 3: 通訊協議                                               │
│  ┌─────────────────────┐    ┌─────────────────────┐             │
│  │ mctp                │────▶│ pldm                │             │
│  │ (傳輸層)             │    │ (平台管理協議)       │             │
│  └─────────────────────┘    └─────────────────────┘             │
│                                                                 │
│  Level 4: 進階主題（計畫中）                                      │
│  ┌─────────────────────┐    ┌─────────────────────┐             │
│  │ bmcweb              │    │ phosphor-state-     │             │
│  │ (Redfish API)       │    │ manager             │             │
│  └─────────────────────┘    └─────────────────────┘             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 🎯 針對不同角色的建議

| 角色 | 建議路徑 |
|------|----------|
| **平台整合者** | entity-manager → dbus-sensors → phosphor-dbus-interfaces |
| **協議開發者** | phosphor-dbus-interfaces → mctp → pldm |
| **BMC 韌體開發者** | 全部按順序 |
| **外部整合者** | mctp → pldm（專注在 Host-BMC 通訊） |

---

## 📖 相關連結

- [OpenBMC 官方 GitHub](https://github.com/openbmc)
- [OpenBMC 官方文件](https://github.com/openbmc/docs)
- [OpenBMC Discord](https://discord.gg/69Km47zH98)

---

*最後更新：2025-12-19*
