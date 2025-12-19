# NVMe 感測器

## 簡介

NVMeSensor 守護程式透過 NVMe-MI（Management Interface）SMBus 協定讀取 NVMe SSD 的溫度和狀態資訊。

---

## 守護程式資訊

| 項目 | 值 |
|------|-----|
| 執行檔 | `nvmesensor` |
| D-Bus 服務 | `xyz.openbmc_project.NVMeSensor` |
| 配置類型 | NVME1000 |
| 資料來源 | SMBus (NVMe-MI) |

---

## NVMe-MI 簡介

NVMe Management Interface (NVMe-MI) 提供帶外（Out-of-Band）管理功能：
- 透過 SMBus/I2C 通訊
- 無需主機參與
- 讀取 SSD 溫度、健康狀態等

標準 NVMe-MI SMBus 位址為 **0x6A**（7 位元位址）。

---

## Entity-Manager 配置

### 基本配置

```json
{
    "Exposes": [
        {
            "Address": "0x6A",
            "Bus": 10,
            "Name": "NVMe SSD 0",
            "Type": "NVME1000"
        }
    ],
    "Name": "NVMe Storage",
    "Probe": "TRUE",
    "Type": "Board"
}
```

### 配置欄位說明

| 欄位 | 必填 | 類型 | 說明 |
|------|------|------|------|
| `Name` | ✅ | string | 感測器名稱 |
| `Type` | ✅ | string | 必須為 `"NVME1000"` |
| `Bus` | ✅ | number | I2C/SMBus 匯流排編號 |
| `Address` | ✅ | string | SMBus 位址（通常 "0x6A"） |
| `PowerState` | ❌ | string | 電源狀態條件 |
| `Thresholds` | ❌ | array | 閾值配置 |
| `PEC` | ❌ | string | SMBus PEC 設定 |

---

## SMBus PEC 設定

PEC（Packet Error Checking）提供額外的錯誤檢測：

```json
{
    "Address": "0x6A",
    "Bus": 10,
    "Name": "NVMe SSD",
    "Type": "NVME1000",
    "PEC": "Required"
}
```

| PEC 值 | 說明 |
|--------|------|
| `"Required"` | 必須使用 PEC |
| `"NotRequired"` | 不使用 PEC |

---

## 完整配置範例

### 多 NVMe SSD 系統

```json
{
    "Exposes": [
        {
            "Address": "0x6A",
            "Bus": 10,
            "Name": "NVMe0 Temp",
            "Type": "NVME1000",
            "PowerState": "On",
            "Thresholds": [
                {
                    "Direction": "greater than",
                    "Name": "upper critical",
                    "Severity": 1,
                    "Value": 83
                },
                {
                    "Direction": "greater than",
                    "Name": "upper warning",
                    "Severity": 0,
                    "Value": 70
                }
            ]
        },
        {
            "Address": "0x6A",
            "Bus": 11,
            "Name": "NVMe1 Temp",
            "Type": "NVME1000",
            "PowerState": "On"
        },
        {
            "Address": "0x6A",
            "Bus": 12,
            "Name": "NVMe2 Temp",
            "Type": "NVME1000",
            "PowerState": "On"
        }
    ],
    "Name": "NVMe Storage Board",
    "Probe": "TRUE",
    "Type": "Board"
}
```

---

## PCIe Slot 配置

結合 PCIe MUX 的 NVMe 配置：

```json
{
    "Exposes": [
        {
            "Address": "0x71",
            "Bus": 5,
            "ChannelNames": ["NVMe0", "NVMe1", "NVMe2", "NVMe3"],
            "Name": "NVMe Mux",
            "Type": "PCA9546Mux"
        },
        {
            "Address": "0x6A",
            "Bus": "$bus",
            "Name": "$bus NVMe Temp",
            "Type": "NVME1000",
            "PowerState": "On"
        }
    ],
    "Name": "NVMe Backplane",
    "Probe": "xyz.openbmc_project.FruDevice({'PRODUCT_PRODUCT_NAME': 'NVMe BP'})",
    "Type": "Board"
}
```

---

## D-Bus 輸出

```bash
$ busctl tree xyz.openbmc_project.NVMeSensor
└─/xyz
  └─/xyz/openbmc_project
    └─/xyz/openbmc_project/sensors
      └─/xyz/openbmc_project/sensors/temperature
        ├─/xyz/openbmc_project/sensors/temperature/NVMe0_Temp
        ├─/xyz/openbmc_project/sensors/temperature/NVMe1_Temp
        └─/xyz/openbmc_project/sensors/temperature/NVMe2_Temp

$ busctl introspect xyz.openbmc_project.NVMeSensor \
    /xyz/openbmc_project/sensors/temperature/NVMe0_Temp

xyz.openbmc_project.Sensor.Value    interface
.MaxValue                           property  d  127
.MinValue                           property  d  -60
.Value                              property  d  42
```

---

## NVMe-MI 溫度讀取

NVMe-MI 透過 SMBus 命令讀取溫度：

| 命令 | 說明 |
|------|------|
| NVMe-MI Read | 讀取 NVMe 子系統健康狀態 |
| 溫度欄位 | Composite Temperature（組合溫度） |

溫度單位為攝氏度，範圍通常為 -60°C 至 127°C。

---

## 除錯指令

```bash
# 檢測 I2C 位址
i2cdetect -y 10

# 檢查 D-Bus 感測器
busctl get-property xyz.openbmc_project.NVMeSensor \
    /xyz/openbmc_project/sensors/temperature/NVMe0_Temp \
    xyz.openbmc_project.Sensor.Value Value

# 查看 Journal 日誌
journalctl -u xyz.openbmc_project.nvmesensor.service
```

---

## 常見問題

### NVMe 無法偵測

1. 確認 SSD 支援 NVMe-MI SMBus
2. 檢查 I2C 匯流排連接
3. 驗證 SMBus 位址（標準為 0x6A）
4. 確認電源狀態

### 溫度讀數為 NaN

1. 檢查 `PowerState` 設定
2. 確認 SSD 已開機
3. 嘗試更換 PEC 設定

---

## 相關文件

- [感測器類型總覽](SensorTypes.md)
- [設定指南](ConfigurationGuide.md)
- [Entity-Manager 整合](EntityManagerIntegration.md)
