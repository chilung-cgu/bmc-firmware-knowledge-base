# 相容軟體

## 概述

Entity-Manager 作為 OpenBMC 的配置中心，與多個其他專案整合。這些軟體讀取 Entity-Manager 發布的配置或為其提供偵測資料。

---

## 整合架構

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          應用層 (Consumers)                              │
│                                                                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │
│  │   bmcweb    │  │ intel-ipmi  │  │ phosphor-   │  │   其他       │    │
│  │  (Redfish)  │  │    -oem     │  │ pid-control │  │   應用       │    │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘    │
│         │                │                │                │            │
└─────────┼────────────────┼────────────────┼────────────────┼────────────┘
          │                │                │                │
          ▼                ▼                ▼                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                            D-Bus                                         │
│         xyz.openbmc_project.Configuration.*                              │
│         xyz.openbmc_project.Sensor.*                                     │
│         xyz.openbmc_project.Inventory.*                                  │
└─────────────────────────────────────────────────────────────────────────┘
          ▲                ▲                ▲
          │                │                │
┌─────────┼────────────────┼────────────────┼─────────────────────────────┐
│         │                │                │                              │
│  ┌──────┴──────┐  ┌──────┴──────┐  ┌──────┴──────┐                      │
│  │   Entity-   │  │ dbus-       │  │ Detection   │                      │
│  │   Manager   │  │ sensors     │  │ Daemons     │                      │
│  └─────────────┘  └─────────────┘  └─────────────┘                      │
│                                                                          │
│                          核心層 (Providers)                               │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 偵測守護程式

### fru-device

**功能**：掃描 I2C 匯流排，發現並解析 IPMI FRU EEPROM

| 項目 | 說明 |
|-----|------|
| **儲存庫** | [openbmc/entity-manager](https://github.com/openbmc/entity-manager) (內含) |
| **D-Bus 服務** | `xyz.openbmc_project.FruDevice` |
| **發布介面** | `xyz.openbmc_project.FruDevice` |

**用途**：

- 提供 Probe 匹配所需的 FRU 資料
- 提供 `$bus`、`$address` 範本變數值

### peci-pcie

**功能**：透過 PECI 介面從 CPU 讀取 PCIe 裝置清單

| 項目 | 說明 |
|-----|------|
| **儲存庫** | [openbmc/peci-pcie](https://github.com/openbmc/peci-pcie) |
| **D-Bus 服務** | `xyz.openbmc_project.PCIe` |
| **發布介面** | `xyz.openbmc_project.Inventory.Item.PCIeDevice` |

**用途**：

- 偵測 CPU 插槽中的 PCIe 裝置
- 提供 GPU、NIC 等擴充卡資訊

### smbios-mdr

**功能**：解析 x86 SMBIOS 表格

| 項目 | 說明 |
|-----|------|
| **儲存庫** | [openbmc/smbios-mdr](https://github.com/openbmc/smbios-mdr) |
| **D-Bus 服務** | `xyz.openbmc_project.Smbios.MDR_V2` |
| **發布介面** | 多種庫存介面 |

**用途**：

- 從 BIOS 取得系統資訊
- 提供 CPU、記憶體、主機板資訊

---

## 反應器 (Reactor) 守護程式

### dbus-sensors

**功能**：讀取各種感測器數值並發布到 D-Bus

| 項目 | 說明 |
|-----|------|
| **儲存庫** | [openbmc/dbus-sensors](https://github.com/openbmc/dbus-sensors) |
| **D-Bus 服務** | 多個（見下表） |
| **監聽介面** | `xyz.openbmc_project.Configuration.*` |

**包含的守護程式**：

| 守護程式 | 功能 |
|---------|------|
| `hwmontempsensor` | hwmon 溫度感測器 |
| `adcsensor` | ADC 電壓感測器 |
| `fansensor` | 風扇轉速 |
| `psusensor` | 電源供應器感測器 |
| `cpusensor` | CPU 溫度 (PECI) |
| `nvmesensor` | NVMe SSD 溫度 |
| `exitairtempsensor` | 出風口溫度計算 |
| `intrusionsensor` | 機箱入侵偵測 |

詳見 [dbus-sensors 整合](DbusSensorsIntegration.md)。

### phosphor-pid-control

**功能**：根據感測器數值執行 PID 風扇控制

| 項目 | 說明 |
|-----|------|
| **儲存庫** | [openbmc/phosphor-pid-control](https://github.com/openbmc/phosphor-pid-control) |
| **別名** | swampd |
| **監聽介面** | `xyz.openbmc_project.Configuration.Pid*` |

**用途**：

- 讀取 Entity-Manager 的 PID 配置
- 根據溫度自動調整風扇速度

---

## 應用層軟體

### bmcweb

**功能**：提供 Redfish REST API

| 項目 | 說明 |
|-----|------|
| **儲存庫** | [openbmc/bmcweb](https://github.com/openbmc/bmcweb) |
| **協定** | Redfish (DMTF 標準) |

**與 Entity-Manager 的整合**：

- 讀取庫存資訊，呈現於 Redfish 端點
- 使用關聯性建立正確的父子關係
- 提供感測器數值查詢

**Redfish 端點範例**：

```
GET /redfish/v1/Chassis
GET /redfish/v1/Chassis/{ChassisId}
GET /redfish/v1/Chassis/{ChassisId}/Thermal
GET /redfish/v1/Chassis/{ChassisId}/Power
GET /redfish/v1/Chassis/{ChassisId}/Sensors/{SensorId}
```

### intel-ipmi-oem

**功能**：提供 Intel OEM IPMI 命令

| 項目 | 說明 |
|-----|------|
| **儲存庫** | [openbmc/intel-ipmi-oem](https://github.com/openbmc/intel-ipmi-oem) |
| **協定** | IPMI |

**與 Entity-Manager 的整合**：

| IPMI 功能 | 資料來源 |
|----------|---------|
| SDR (Sensor Data Records) | Entity-Manager 感測器配置 |
| FRU 資訊 | FruDevice 發布的 FRU 資料 |
| Storage 命令 | 庫存資訊 |

---

## phosphor-dbus-interfaces

**功能**：定義 OpenBMC D-Bus 介面規範

| 項目 | 說明 |
|-----|------|
| **儲存庫** | [openbmc/phosphor-dbus-interfaces](https://github.com/openbmc/phosphor-dbus-interfaces) |
| **格式** | YAML |

**相關介面**：

```
xyz.openbmc_project.Configuration.*    # 配置介面
xyz.openbmc_project.Sensor.*           # 感測器介面
xyz.openbmc_project.Inventory.*        # 庫存介面
xyz.openbmc_project.FruDevice          # FRU 裝置介面
```

---

## 整合驗證

### 驗證 Redfish 存取

```bash
# 透過 curl 存取 Redfish
curl -k -u root:0penBmc https://localhost/redfish/v1/Chassis

# 查看特定機箱的感測器
curl -k -u root:0penBmc https://localhost/redfish/v1/Chassis/My_Board/Thermal
```

### 驗證 IPMI 存取

```bash
# 查看 SDR
ipmitool sdr list

# 查看 FRU
ipmitool fru print

# 讀取特定感測器
ipmitool sensor get "Inlet Temp"
```

### 驗證 D-Bus 感測器

```bash
# 列出所有感測器
busctl tree xyz.openbmc_project.HwmonTempSensor

# 讀取感測器值
busctl get-property xyz.openbmc_project.HwmonTempSensor \
    /xyz/openbmc_project/sensors/temperature/Inlet_Temp \
    xyz.openbmc_project.Sensor.Value \
    Value
```

---

## 資料流示例

### 溫度感測器到 Redfish

```
1. fru-device 掃描 I2C，發現 FRU
   └─▶ D-Bus: xyz.openbmc_project.FruDevice
   
2. Entity-Manager Probe 匹配
   └─▶ D-Bus: xyz.openbmc_project.Configuration.TMP75
   
3. hwmontempsensor 建立感測器
   └─▶ D-Bus: xyz.openbmc_project.Sensor.Value
   
4. bmcweb 讀取感測器
   └─▶ Redfish: /redfish/v1/Chassis/.../Thermal/Temperatures
```

### 配置到 IPMI SDR

```
1. Entity-Manager 發布配置
   └─▶ D-Bus: xyz.openbmc_project.Configuration.*
   
2. dbus-sensors 建立感測器
   └─▶ D-Bus: xyz.openbmc_project.Sensor.*
   
3. intel-ipmi-oem 讀取感測器
   └─▶ IPMI: SDR Get Device SDR
```

---

## 相容性矩陣

| 功能 | Entity-Manager | dbus-sensors | bmcweb | IPMI |
|-----|----------------|--------------|--------|------|
| 溫度感測器 | ✅ 配置 | ✅ 讀取 | ✅ Redfish | ✅ SDR |
| 電壓感測器 | ✅ 配置 | ✅ 讀取 | ✅ Redfish | ✅ SDR |
| 風扇感測器 | ✅ 配置 | ✅ 讀取 | ✅ Redfish | ✅ SDR |
| FRU 資訊 | ✅ 關聯 | - | ✅ Inventory | ✅ FRU |
| 機箱關聯 | ✅ 關聯 | - | ✅ Chassis | - |

---

## 故障排除整合問題

### Redfish 無法顯示感測器

1. 確認 Entity-Manager 配置已載入
2. 確認 dbus-sensors 已建立感測器
3. 檢查 bmcweb 日誌

### IPMI SDR 為空

1. 確認感測器存在於 D-Bus
2. 檢查 intel-ipmi-oem 是否啟用
3. 驗證感測器類型是否支援

---

## 下一步

- 了解 [架構概述](Architecture.md) 理解整體設計
- 查看 [dbus-sensors 整合](DbusSensorsIntegration.md) 了解感測器細節
- 閱讀 [故障排除](Troubleshooting.md) 解決整合問題

---

> 📖 **參考**：
> - [bmcweb 儲存庫](https://github.com/openbmc/bmcweb)
> - [dbus-sensors 儲存庫](https://github.com/openbmc/dbus-sensors)
> - [intel-ipmi-oem 儲存庫](https://github.com/openbmc/intel-ipmi-oem)
