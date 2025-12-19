# phosphor-dbus-interfaces 技術 Wiki

歡迎來到 OpenBMC **phosphor-dbus-interfaces** 技術文件。本 Wiki 提供關於 D-Bus 介面定義的完整技術參考。

---

## 📖 專案簡介

**phosphor-dbus-interfaces** 是 OpenBMC 生態系統的核心專案，負責定義所有標準 D-Bus 介面的 YAML 描述檔。這些定義透過 [sdbusplus](https://github.com/openbmc/sdbusplus) 的 `sdbus++` 工具自動生成 C++ 綁定程式碼。

### 核心功能

| 功能 | 說明 |
|------|------|
| 介面定義 | 使用 YAML 格式定義 D-Bus 介面契約 |
| 程式碼生成 | 自動生成 C++ headers 和 bindings |
| 標準化通訊 | 為 OpenBMC 各服務提供統一的 IPC 契約 |
| 硬體抽象 | 提供與硬體無關的通用介面 |

---

## 🗂️ 文件導覽

### 入門與架構

| 文件 | 說明 |
|------|------|
| [Architecture](Architecture.md) | 系統架構與設計原則 |
| [Namespaces](Namespaces.md) | 命名空間結構總覽 |

### YAML 介面格式

| 文件 | 說明 |
|------|------|
| [YAMLFormat](YAMLFormat.md) | YAML 介面定義格式詳解 |
| [CodeGeneration](CodeGeneration.md) | sdbus++ 程式碼生成 |

### 介面類別

| 文件 | 說明 |
|------|------|
| [SensorInterfaces](SensorInterfaces.md) | 感測器介面 (`xyz.openbmc_project.Sensor.*`) |
| [StateInterfaces](StateInterfaces.md) | 狀態管理介面 (`xyz.openbmc_project.State.*`) |
| [InventoryInterfaces](InventoryInterfaces.md) | 硬體清單介面 (`xyz.openbmc_project.Inventory.*`) |
| [ControlInterfaces](ControlInterfaces.md) | 控制介面 (`xyz.openbmc_project.Control.*`) |
| [LoggingInterfaces](LoggingInterfaces.md) | 日誌介面 (`xyz.openbmc_project.Logging.*`) |
| [NetworkInterfaces](NetworkInterfaces.md) | 網路介面 (`xyz.openbmc_project.Network.*`) |
| [SoftwareInterfaces](SoftwareInterfaces.md) | 軟體更新介面 (`xyz.openbmc_project.Software.*`) |
| [ObjectMapperInterface](ObjectMapperInterface.md) | 物件映射器介面 |

### 進階主題

| 文件 | 說明 |
|------|------|
| [Associations](Associations.md) | 關聯機制詳解 |
| [Enumerations](Enumerations.md) | 列舉型別使用 |
| [Errors](Errors.md) | 錯誤處理機制 |
| [Requirements](Requirements.md) | 介面設計規範 |
| [BuildConfig](BuildConfig.md) | 建置與設定 |

### 參考與除錯

| 文件 | 說明 |
|------|------|
| [CommonDBusTypes](CommonDBusTypes.md) | 常用 D-Bus 型別參考 |
| [Troubleshooting](Troubleshooting.md) | 故障排除指南 |

---

## 🔗 相關專案

| 專案 | 說明 | 連結 |
|------|------|------|
| sdbusplus | C++ D-Bus 函式庫與綁定生成工具 | [GitHub](https://github.com/openbmc/sdbusplus) |
| entity-manager | 執行時期硬體配置管理 | [Wiki](../entity-manager/Home.md) |
| dbus-sensors | 感測器讀取守護程式 | [Wiki](../dbus-sensors/Home.md) |
| phosphor-state-manager | 電源狀態管理 | [GitHub](https://github.com/openbmc/phosphor-state-manager) |
| bmcweb | Redfish REST API 實作 | [GitHub](https://github.com/openbmc/bmcweb) |

---

## 📚 官方資源

- [phosphor-dbus-interfaces GitHub](https://github.com/openbmc/phosphor-dbus-interfaces)
- [OpenBMC 官方文件](https://github.com/openbmc/docs)
- [D-Bus 規格](https://dbus.freedesktop.org/doc/dbus-specification.html)

---

*最後更新：2025-12-19*
