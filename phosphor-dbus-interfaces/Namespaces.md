# Namespaces - 命名空間結構

本文件說明 phosphor-dbus-interfaces 的命名空間組織結構。

---

## 📋 概述

phosphor-dbus-interfaces 使用階層式命名空間來組織 D-Bus 介面。命名空間反映在 YAML 檔案的目錄結構和最終的 D-Bus 介面名稱中。

### 命名空間對應

| 命名空間 | 目錄路徑 | 說明 |
|----------|----------|------|
| `xyz.openbmc_project` | `yaml/xyz/openbmc_project/` | OpenBMC 標準介面（預設啟用） |
| `org.freedesktop` | `yaml/org/freedesktop/` | 標準 D-Bus 介面（預設啟用） |
| `com.ibm` | `yaml/com/ibm/` | IBM 專用介面（需手動啟用） |
| `org.open_power` | `yaml/org/open_power/` | Open Power 專用介面（需手動啟用） |

---

## 🌐 xyz.openbmc_project

這是 OpenBMC 的主要命名空間，包含所有標準化的 D-Bus 介面。

### 目錄結構

```
yaml/xyz/openbmc_project/
├── Association/           # 關聯相關介面
├── Attestation/          # 認證介面
├── BIOSConfig/           # BIOS 設定介面
├── Certs/                # 憑證管理介面
├── Channel/              # 通道介面
├── Chassis/              # 機箱介面
├── Collection/           # 集合介面
├── Common/               # 通用定義
├── Condition/            # 條件介面
├── Configuration/        # 設定介面
├── Console/              # 主控台介面
├── Control/              # 控制介面
├── Debug/                # 除錯介面
├── Dump/                 # 傾印介面
├── HardwareIsolation/    # 硬體隔離介面
├── Inventory/            # 硬體清單介面
├── Ipmi/                 # IPMI 介面
├── Led/                  # LED 介面
├── Logging/              # 日誌介面
├── MCTP/                 # MCTP 協定介面
├── Memory/               # 記憶體介面
├── Metric/               # 度量介面
├── Network/              # 網路介面
├── Nvme/                 # NVMe 介面
├── Object/               # 物件基礎介面
├── PFR/                  # 平台韌體韌性介面
├── PLDM/                 # PLDM 協定介面
├── Sensor/               # 感測器介面
├── Smbios/               # SMBIOS 介面
├── Software/             # 軟體管理介面
├── State/                # 狀態管理介面
├── Telemetry/            # 遙測介面
├── Time/                 # 時間介面
├── User/                 # 使用者管理介面
├── VirtualMedia/         # 虛擬媒體介面
├── Association.interface.yaml
├── ObjectMapper.interface.yaml
└── ...
```

### 主要介面類別

#### Sensor - 感測器

| 介面 | 說明 |
|------|------|
| `xyz.openbmc_project.Sensor.Value` | 感測器讀數 |
| `xyz.openbmc_project.Sensor.Threshold.Warning` | 警告閾值 |
| `xyz.openbmc_project.Sensor.Threshold.Critical` | 嚴重閾值 |
| `xyz.openbmc_project.Sensor.Accuracy` | 感測器精度 |

詳見 [SensorInterfaces](SensorInterfaces.md)。

#### State - 狀態管理

| 介面 | 說明 |
|------|------|
| `xyz.openbmc_project.State.BMC` | BMC 狀態 |
| `xyz.openbmc_project.State.Host` | 主機狀態 |
| `xyz.openbmc_project.State.Chassis` | 機箱電源狀態 |
| `xyz.openbmc_project.State.Watchdog` | 看門狗計時器 |

詳見 [StateInterfaces](StateInterfaces.md)。

#### Inventory - 硬體清單

| 介面 | 說明 |
|------|------|
| `xyz.openbmc_project.Inventory.Item` | 基礎清單項目 |
| `xyz.openbmc_project.Inventory.Item.Cpu` | CPU 資訊 |
| `xyz.openbmc_project.Inventory.Item.Dimm` | 記憶體模組資訊 |
| `xyz.openbmc_project.Inventory.Item.Board` | 主機板資訊 |

詳見 [InventoryInterfaces](InventoryInterfaces.md)。

#### Control - 控制

| 介面 | 說明 |
|------|------|
| `xyz.openbmc_project.Control.Power.Cap` | 電源上限控制 |
| `xyz.openbmc_project.Control.FanSpeed` | 風扇速度控制 |
| `xyz.openbmc_project.Control.Mode` | 模式控制 |

詳見 [ControlInterfaces](ControlInterfaces.md)。

#### Logging - 日誌

| 介面 | 說明 |
|------|------|
| `xyz.openbmc_project.Logging.Entry` | 日誌條目 |
| `xyz.openbmc_project.Logging.Create` | 建立日誌 |

詳見 [LoggingInterfaces](LoggingInterfaces.md)。

---

## 🔧 org.freedesktop

標準 D-Bus 介面，來自 freedesktop.org 規格。

### 常用介面

| 介面 | 說明 |
|------|------|
| `org.freedesktop.DBus.ObjectManager` | 物件管理器 |
| `org.freedesktop.DBus.Properties` | 屬性存取 |
| `org.freedesktop.DBus.Introspectable` | 介面內省 |

---

## 🏢 com.ibm

IBM 專用介面，需要手動啟用：

```bash
meson builddir -Ddata_com_ibm=true
```

### 主要介面

| 介面 | 說明 |
|------|------|
| `com.ibm.ipzvpd.*` | VPD 資料介面 |
| `com.ibm.Control.*` | IBM 專用控制介面 |

---

## ⚡ org.open_power

Open Power 專用介面，需要手動啟用：

```bash
meson builddir -Ddata_org_open_power=true
```

### 主要介面

| 介面 | 說明 |
|------|------|
| `org.open_power.OCC.*` | OCC 控制介面 |
| `org.open_power.Proc.*` | 處理器介面 |

---

## 📍 D-Bus 物件路徑

介面通常實作在特定的 D-Bus 路徑下：

### 常用路徑

| 路徑 | 說明 |
|------|------|
| `/xyz/openbmc_project/sensors` | 感測器物件 |
| `/xyz/openbmc_project/state` | 狀態物件 |
| `/xyz/openbmc_project/inventory` | 清單物件 |
| `/xyz/openbmc_project/control` | 控制物件 |
| `/xyz/openbmc_project/logging` | 日誌物件 |
| `/xyz/openbmc_project/network` | 網路物件 |
| `/xyz/openbmc_project/software` | 軟體物件 |
| `/xyz/openbmc_project/ObjectMapper` | 物件映射器 |

### 路徑階層範例

```
/xyz/openbmc_project/
├── sensors/
│   ├── temperature/
│   │   ├── CPU_Core_Temp
│   │   └── Ambient_Temp
│   ├── voltage/
│   │   ├── P12V
│   │   └── P3V3
│   └── fan_tach/
│       ├── Fan0
│       └── Fan1
├── state/
│   ├── bmc0
│   ├── host0
│   └── chassis0
├── inventory/
│   └── system/
│       ├── chassis/
│       │   └── motherboard/
│       │       ├── cpu0
│       │       ├── dimm0
│       │       └── fan0
│       └── ...
└── logging/
    └── entry/
        ├── 1
        ├── 2
        └── ...
```

---

## 🔗 服務名稱

每個 D-Bus 服務使用唯一的 bus name：

### 常見服務名稱

| 服務名稱 | 專案 | 說明 |
|----------|------|------|
| `xyz.openbmc_project.EntityManager` | entity-manager | 硬體配置管理 |
| `xyz.openbmc_project.HwmonTempSensor` | dbus-sensors | 溫度感測器 |
| `xyz.openbmc_project.FanSensor` | dbus-sensors | 風扇感測器 |
| `xyz.openbmc_project.State.BMC` | phosphor-state-manager | BMC 狀態 |
| `xyz.openbmc_project.State.Host` | phosphor-state-manager | 主機狀態 |
| `xyz.openbmc_project.ObjectMapper` | phosphor-objmgr | 物件映射器 |
| `xyz.openbmc_project.Logging` | phosphor-logging | 日誌服務 |
| `xyz.openbmc_project.Network` | phosphor-networkd | 網路服務 |

---

## 🔍 查詢介面

### 使用 busctl 列出介面

```bash
# 列出所有 xyz.openbmc_project 開頭的服務
busctl --list | grep xyz.openbmc_project

# 列出特定服務的物件樹
busctl tree xyz.openbmc_project.ObjectMapper

# 查看物件實作的介面
busctl introspect xyz.openbmc_project.ObjectMapper /xyz/openbmc_project/ObjectMapper
```

### 使用 ObjectMapper 查詢

```bash
# 查詢實作特定介面的所有物件
busctl call xyz.openbmc_project.ObjectMapper \
    /xyz/openbmc_project/ObjectMapper \
    xyz.openbmc_project.ObjectMapper \
    GetSubTree sias "/" 0 1 "xyz.openbmc_project.Sensor.Value"
```

---

## 🔍 延伸閱讀

- [SensorInterfaces](SensorInterfaces.md) - 感測器介面詳解
- [StateInterfaces](StateInterfaces.md) - 狀態介面詳解
- [InventoryInterfaces](InventoryInterfaces.md) - 清單介面詳解
- [ObjectMapperInterface](ObjectMapperInterface.md) - 物件映射器使用

---

*最後更新：2025-12-19*
