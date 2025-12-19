# 外部感測器

## 簡介

ExternalSensor 是一種特殊的虛擬感測器，其數值由外部來源透過 D-Bus 推送，而非從本地硬體讀取。這使得來自主機（Host）端或其他外部系統的感測器數值能夠整合到 OpenBMC 的感測器框架中。

---

## 守護程式資訊

| 項目 | 值 |
|------|-----|
| 執行檔 | `externalsensor` |
| D-Bus 服務 | `xyz.openbmc_project.ExternalSensor` |
| 配置類型 | ExternalSensor |
| 資料來源 | D-Bus（外部推送） |

---

## 設計動機

1. **彈性的物理量**：不受限於特定硬體，可測量任何 OpenBMC 支援的物理量
2. **外部資料整合**：接受 IPMI、Redfish 或其他來源的感測器數值
3. **逾時偵測**：自動偵測過期的感測器數值並標記為 NaN

---

## Entity-Manager 配置

### 基本配置

```json
{
    "Exposes": [
        {
            "Name": "HostDevTemp",
            "Type": "ExternalSensor",
            "Units": "DegreesC",
            "MinValue": -16.0,
            "MaxValue": 111.5
        }
    ],
    "Name": "External Sensors",
    "Probe": "TRUE",
    "Type": "Board"
}
```

### 配置欄位說明

| 欄位 | 必填 | 類型 | 說明 |
|------|------|------|------|
| `Name` | ✅ | string | 感測器名稱 |
| `Type` | ✅ | string | 必須為 `"ExternalSensor"` |
| `Units` | ✅ | string | 單位類型 |
| `MinValue` | ✅ | number | 最小有效值 |
| `MaxValue` | ✅ | number | 最大有效值 |
| `Timeout` | ❌ | number | 逾時秒數 |
| `Thresholds` | ❌ | array | 閾值配置 |
| `PowerState` | ❌ | string | 電源狀態條件 |

---

## Units 單位類型

ExternalSensor 支援所有標準單位類型：

| Units 值 | D-Bus 路徑 | 說明 |
|----------|------------|------|
| `"DegreesC"` | `/sensors/temperature/` | 攝氏溫度 |
| `"Volts"` | `/sensors/voltage/` | 電壓 |
| `"Amperes"` | `/sensors/current/` | 電流 |
| `"Watts"` | `/sensors/power/` | 功率 |
| `"RPMS"` | `/sensors/fan_tach/` | 轉速 |
| `"Joules"` | `/sensors/energy/` | 能量 |
| `"Percent"` | `/sensors/utilization/` | 百分比 |
| `"Pascals"` | `/sensors/pressure/` | 壓力 |
| `"CFM"` | `/sensors/airflow/` | 氣流 |

---

## Timeout 逾時機制

`Timeout` 欄位定義感測器數值的有效期限（秒）：

```json
{
    "Name": "HostTemp",
    "Type": "ExternalSensor",
    "Units": "DegreesC",
    "MinValue": -40,
    "MaxValue": 125,
    "Timeout": 4.0
}
```

行為說明：
- 若超過 `Timeout` 秒沒有收到更新，感測器值設為 **NaN**
- NaN 表示數值不可用，消費者可據此判斷感測器狀態
- 若未設定 `Timeout`，感測器值永不過期

> [!IMPORTANT]
> 使用 NaN 而非特殊數值（如 -1 或 9999）可明確區分「無數值」與「有效數值」。

---

## 完整配置範例

### 主機端感測器

```json
{
    "Exposes": [
        {
            "Name": "Host GPU0 Temp",
            "Type": "ExternalSensor",
            "Units": "DegreesC",
            "MinValue": 0,
            "MaxValue": 105,
            "Timeout": 5.0,
            "Thresholds": [
                {
                    "Direction": "greater than",
                    "Name": "upper critical",
                    "Severity": 1,
                    "Value": 95
                },
                {
                    "Direction": "greater than",
                    "Name": "upper warning",
                    "Severity": 0,
                    "Value": 85
                }
            ]
        },
        {
            "Name": "Host GPU1 Temp",
            "Type": "ExternalSensor",
            "Units": "DegreesC",
            "MinValue": 0,
            "MaxValue": 105,
            "Timeout": 5.0
        },
        {
            "Name": "Host Memory Usage",
            "Type": "ExternalSensor",
            "Units": "Percent",
            "MinValue": 0,
            "MaxValue": 100
        }
    ],
    "Name": "Host Sensors",
    "Probe": "TRUE",
    "Type": "Board"
}
```

---

## D-Bus 數值更新

外部來源透過 D-Bus 設定感測器數值：

```bash
# 更新感測器數值
busctl set-property xyz.openbmc_project.ExternalSensor \
    /xyz/openbmc_project/sensors/temperature/Host_GPU0_Temp \
    xyz.openbmc_project.Sensor.Value Value d 72.5
```

### IPMI 整合

IPMI `Set Sensor Reading` 命令可用於更新 ExternalSensor：

1. 主機透過 IPMI 發送感測器數值
2. phosphor-host-ipmid 接收命令
3. 透過 D-Bus 更新 ExternalSensor 數值

---

## D-Bus 輸出

```bash
$ busctl tree xyz.openbmc_project.ExternalSensor
└─/xyz
  └─/xyz/openbmc_project
    └─/xyz/openbmc_project/sensors
      ├─/xyz/openbmc_project/sensors/temperature
      │ ├─/xyz/openbmc_project/sensors/temperature/Host_GPU0_Temp
      │ └─/xyz/openbmc_project/sensors/temperature/Host_GPU1_Temp
      └─/xyz/openbmc_project/sensors/utilization
        └─/xyz/openbmc_project/sensors/utilization/Host_Memory_Usage

$ busctl introspect xyz.openbmc_project.ExternalSensor \
    /xyz/openbmc_project/sensors/temperature/Host_GPU0_Temp

xyz.openbmc_project.Sensor.Value         interface
.MaxValue                                property  d  105
.MinValue                                property  d  0
.Value                                   property  d  72.5     writable
```

> [!NOTE]
> ExternalSensor 的 `Value` 屬性是**可寫入**的，這與其他感測器不同。

---

## 除錯指令

```bash
# 檢查感測器數值
busctl get-property xyz.openbmc_project.ExternalSensor \
    /xyz/openbmc_project/sensors/temperature/Host_GPU0_Temp \
    xyz.openbmc_project.Sensor.Value Value

# 手動設定測試數值
busctl set-property xyz.openbmc_project.ExternalSensor \
    /xyz/openbmc_project/sensors/temperature/Host_GPU0_Temp \
    xyz.openbmc_project.Sensor.Value Value d 55.5

# 檢查操作狀態（判斷是否逾時）
busctl get-property xyz.openbmc_project.ExternalSensor \
    /xyz/openbmc_project/sensors/temperature/Host_GPU0_Temp \
    xyz.openbmc_project.State.Decorator.OperationalStatus Functional
```

---

## NaN 處理

當感測器逾時或無數值時，`Value` 屬性為 NaN：

```bash
$ busctl get-property ... Value
d nan
```

消費者應用程式（Redfish、IPMI）應正確處理 NaN：
- Redfish：回報感測器狀態為「Unavailable」
- IPMI：回報感測器讀取失敗

---

## 使用場景

| 場景 | 說明 |
|------|------|
| GPU 溫度 | 主機端 GPU 驅動程式報告的溫度 |
| 儲存健康 | 主機端儲存子系統狀態 |
| 應用程式指標 | 主機應用程式報告的效能指標 |
| 遠端感測器 | 其他 BMC 或控制器報告的數值 |

---

## 相關文件

- [感測器類型總覽](SensorTypes.md)
- [設定指南](ConfigurationGuide.md)
- [D-Bus API](DBusAPI.md)
- [ExternalSensor 設計文件](https://github.com/openbmc/docs/blob/master/designs/external-sensor.md)
