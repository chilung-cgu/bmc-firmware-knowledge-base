# OpenBMC Technical Wiki 📚

這個儲存庫用於存放 OpenBMC 各個核心專案的技術文件（Technical Wiki），採用繁體中文撰寫，遵循 Google Code Wiki 風格。

---

## 📋 專案進度

### ✅ 已完成的 Repository

| Repository | 目錄 | 文件數 | 完成日期 | 說明 |
|------------|------|--------|----------|------|
| [entity-manager](https://github.com/openbmc/entity-manager) | [entity-manager/](entity-manager/) | 13 | 2025-12-19 | 執行時期硬體配置管理 |

### 📝 計畫中的 Repository

| Repository | 優先順序 | 說明 |
|------------|---------|------|
| [dbus-sensors](https://github.com/openbmc/dbus-sensors) | 高 | 感測器讀取守護程式 |
| [bmcweb](https://github.com/openbmc/bmcweb) | 高 | Redfish REST API 實作 |
| [phosphor-state-manager](https://github.com/openbmc/phosphor-state-manager) | 中 | 電源狀態管理 |
| [phosphor-dbus-interfaces](https://github.com/openbmc/phosphor-dbus-interfaces) | 中 | D-Bus 介面定義 |
| [sdbusplus](https://github.com/openbmc/sdbusplus) | 中 | C++ D-Bus 函式庫 |
| [phosphor-pid-control](https://github.com/openbmc/phosphor-pid-control) | 低 | PID 風扇控制 |
| [peci-pcie](https://github.com/openbmc/peci-pcie) | 低 | PCIe 裝置偵測 |
| [smbios-mdr](https://github.com/openbmc/smbios-mdr) | 低 | SMBIOS 表格解析 |

---

## 📂 目錄結構

```
~/notes/
├── README.md              # 本文件 - 專案總覽與進度追蹤
├── entity-manager/        # Entity-Manager 技術文件
│   ├── Home.md           # 首頁與導覽
│   ├── Architecture.md   # 架構概述
│   ├── CoreConcepts.md   # 核心概念
│   ├── DBusAPI.md        # D-Bus API 參考
│   ├── ...               # 更多文件
│   └── Troubleshooting.md
└── {repository}/          # 未來新增的 repository
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

## 📖 相關連結

- [OpenBMC 官方 GitHub](https://github.com/openbmc)
- [OpenBMC 官方文件](https://github.com/openbmc/docs)
- [OpenBMC Discord](https://discord.gg/69Km47zH98)

---

*最後更新：2025-12-19*
