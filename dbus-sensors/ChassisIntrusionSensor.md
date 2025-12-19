# 機箱入侵感測器

## 簡介

ChassisIntrusionSensor 守護程式偵測機箱開啟狀態，用於安全監控目的。支援 GPIO 和 I2C 兩種偵測方式。

---

## 守護程式資訊

| 項目 | 值 |
|------|-----|
| 執行檔 | `intrusionsensor` |
| D-Bus 服務 | `xyz.openbmc_project.IntrusionSensor` |
| 配置類型 | ChassisIntrusionSensor |
| 資料來源 | GPIO 或 I2C |

---

## Entity-Manager 配置

### GPIO 類型配置

```json
{
    "Exposes": [
        {
            "Class": "Gpio",
            "Name": "Chassis Intrusion",
            "Type": "ChassisIntrusionSensor"
        }
    ],
    "Name": "Intrusion Sensor",
    "Probe": "TRUE",
    "Type": "Board"
}
```

### I2C 類型配置

```json
{
    "Exposes": [
        {
            "Address": "0x77",
            "Bus": 3,
            "Class": "i2c",
            "Name": "Chassis Intrusion",
            "Type": "ChassisIntrusionSensor"
        }
    ],
    "Name": "Intrusion Sensor",
    "Probe": "TRUE",
    "Type": "Board"
}
```

---

## 配置欄位說明

| 欄位 | 必填 | 類型 | 說明 |
|------|------|------|------|
| `Name` | ✅ | string | 感測器名稱 |
| `Type` | ✅ | string | 必須為 `"ChassisIntrusionSensor"` |
| `Class` | ✅ | string | 偵測類型（`Gpio` 或 `i2c`） |
| `Address` | ⚠️ | string | I2C 位址（Class 為 i2c 時必填） |
| `Bus` | ⚠️ | number | I2C 匯流排（Class 為 i2c 時必填） |
| `GpioPolarity` | ❌ | string | GPIO 有效電位 |
| `Rearm` | ❌ | string | 重置模式 |

---

## Class 選項

| Class | 說明 | 額外必填欄位 |
|-------|------|--------------|
| `Gpio` | 使用 GPIO 輸入偵測 | 無 |
| `i2c` | 使用 I2C 裝置偵測 | `Address`、`Bus` |

---

## GpioPolarity 設定

定義 GPIO 的有效電位：

```json
{
    "Class": "Gpio",
    "GpioPolarity": "Low",
    "Name": "Chassis Intrusion",
    "Type": "ChassisIntrusionSensor"
}
```

| 值 | 說明 |
|----|------|
| `"Low"` | 低電位表示入侵（預設） |
| `"High"` | 高電位表示入侵 |

---

## Rearm 重置模式

定義入侵警報的重置方式：

```json
{
    "Class": "Gpio",
    "Name": "Chassis Intrusion",
    "Type": "ChassisIntrusionSensor",
    "Rearm": "Manual"
}
```

| 值 | 說明 |
|----|------|
| `"Automatic"` | 機箱關閉後自動重置 |
| `"Manual"` | 需要手動重置警報 |

---

## 完整配置範例

### GPIO 入侵偵測（手動重置）

```json
{
    "Exposes": [
        {
            "Class": "Gpio",
            "GpioPolarity": "Low",
            "Name": "Chassis Intrusion",
            "Rearm": "Manual",
            "Type": "ChassisIntrusionSensor"
        }
    ],
    "Name": "Chassis Security",
    "Probe": "TRUE",
    "Type": "Board"
}
```

### I2C 入侵偵測

通常使用具有入侵偵測功能的 I2C 晶片（如 PCF8574 IO 擴展器）：

```json
{
    "Exposes": [
        {
            "Address": "0x38",
            "Bus": 2,
            "Class": "i2c",
            "Name": "Chassis Intrusion",
            "Rearm": "Automatic",
            "Type": "ChassisIntrusionSensor"
        }
    ],
    "Name": "I2C Intrusion Sensor",
    "Probe": "TRUE",
    "Type": "Board"
}
```

---

## D-Bus 輸出

```bash
$ busctl tree xyz.openbmc_project.IntrusionSensor
└─/xyz
  └─/xyz/openbmc_project
    └─/xyz/openbmc_project/Intrusion
      └─/xyz/openbmc_project/Intrusion/Chassis_Intrusion

$ busctl introspect xyz.openbmc_project.IntrusionSensor \
    /xyz/openbmc_project/Intrusion/Chassis_Intrusion

xyz.openbmc_project.Chassis.Intrusion   interface
.Status                                 property  s  "Normal"
```

---

## 入侵狀態

`Status` 屬性的可能值：

| 值 | 說明 |
|----|------|
| `"Normal"` | 機箱關閉，正常狀態 |
| `"HardwareIntrusion"` | 偵測到機箱開啟 |

---

## 設備樹 GPIO 配置

對於 GPIO 類型，需要在設備樹中定義對應的 GPIO：

```dts
&gpio {
    chassis-intrusion-hog {
        gpio-hog;
        gpios = <ASPEED_GPIO(E, 6) GPIO_ACTIVE_LOW>;
        input;
        line-name = "chassis-intrusion";
    };
};
```

---

## 手動重置警報

對於 `Rearm: Manual` 配置，可透過 D-Bus 重置警報：

```bash
# 讀取目前狀態
busctl get-property xyz.openbmc_project.IntrusionSensor \
    /xyz/openbmc_project/Intrusion/Chassis_Intrusion \
    xyz.openbmc_project.Chassis.Intrusion Status

# 手動重置（如果支援）
busctl call xyz.openbmc_project.IntrusionSensor \
    /xyz/openbmc_project/Intrusion/Chassis_Intrusion \
    xyz.openbmc_project.Chassis.Intrusion Rearm
```

---

## 除錯指令

```bash
# 檢查 GPIO 狀態
gpioinfo | grep intrusion

# 讀取 D-Bus 狀態
busctl get-property xyz.openbmc_project.IntrusionSensor \
    /xyz/openbmc_project/Intrusion/Chassis_Intrusion \
    xyz.openbmc_project.Chassis.Intrusion Status

# 查看日誌
journalctl -u xyz.openbmc_project.intrusionsensor.service
```

---

## 與 Redfish 整合

入侵狀態可透過 Redfish Chassis 端點存取：

```
GET /redfish/v1/Chassis/chassis/
{
    "PhysicalSecurity": {
        "IntrusionSensor": "Normal" | "HardwareIntrusion"
    }
}
```

---

## 相關文件

- [感測器類型總覽](SensorTypes.md)
- [設定指南](ConfigurationGuide.md)
- [bmcweb Redfish](https://github.com/openbmc/bmcweb)
