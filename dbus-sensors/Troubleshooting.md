# 故障排除

## 簡介

本文件提供 dbus-sensors 常見問題的診斷和解決方法。

---

## 診斷工具

### busctl

D-Bus 互動工具：

```bash
# 列出所有感測器服務
busctl list | grep -i sensor

# 顯示感測器樹狀結構
busctl tree xyz.openbmc_project.HwmonTempSensor

# 內省感測器物件
busctl introspect xyz.openbmc_project.HwmonTempSensor \
    /xyz/openbmc_project/sensors/temperature/CPU_Temp

# 讀取感測器數值
busctl get-property xyz.openbmc_project.HwmonTempSensor \
    /xyz/openbmc_project/sensors/temperature/CPU_Temp \
    xyz.openbmc_project.Sensor.Value Value
```

### journalctl

檢視服務日誌：

```bash
# HwmonTempSensor 日誌
journalctl -u xyz.openbmc_project.hwmontempsensor.service

# ADCSensor 日誌
journalctl -u xyz.openbmc_project.adcsensor.service

# 即時監控
journalctl -f -u xyz.openbmc_project.hwmontempsensor.service
```

### i2cdetect

掃描 I2C 匯流排：

```bash
# 掃描 bus 1
i2cdetect -y 1

# 預期輸出
     0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
00:          -- -- -- -- -- -- -- -- -- -- -- -- -- 
10: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
30: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
40: -- -- -- -- -- -- -- -- 48 49 -- -- -- -- -- -- 
50: 50 -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
```

### hwmon 檔案

檢查 hwmon 子系統：

```bash
# 列出 hwmon 裝置
ls /sys/class/hwmon/

# 檢查裝置名稱
cat /sys/class/hwmon/hwmon5/name

# 讀取溫度（毫度）
cat /sys/class/hwmon/hwmon5/temp1_input
```

---

## 常見問題

### 感測器未出現在 D-Bus

**症狀：** `busctl tree` 沒有顯示預期的感測器

**診斷步驟：**

1. **檢查 Entity-Manager 配置**
   ```bash
   busctl tree xyz.openbmc_project.EntityManager
   ```
   確認 Configuration 介面已發布

2. **檢查感測器服務狀態**
   ```bash
   systemctl status xyz.openbmc_project.hwmontempsensor.service
   ```

3. **檢視服務日誌**
   ```bash
   journalctl -u xyz.openbmc_project.hwmontempsensor.service
   ```

4. **驗證 I2C 裝置**
   ```bash
   i2cdetect -y <bus>
   ```

**可能原因：**

| 原因 | 解決方法 |
|------|----------|
| JSON 配置語法錯誤 | 使用 `python -m json.tool` 驗證 |
| Probe 不匹配 | 檢查 FruDevice 是否存在 |
| PowerState 條件不滿足 | 確認系統電源狀態 |
| I2C 裝置不存在 | 檢查硬體連接 |

---

### 感測器數值為 NaN

**症狀：** `Value` 屬性顯示 `nan`

**診斷步驟：**

1. **檢查 hwmon 檔案**
   ```bash
   cat /sys/class/hwmon/hwmonX/temp1_input
   ```
   若讀取失敗，可能是硬體問題

2. **檢查 PowerState**
   ```bash
   busctl introspect xyz.openbmc_project.EntityManager \
       /.../CPU_Temp | grep PowerState
   ```

3. **檢查 OperationalStatus**
   ```bash
   busctl get-property xyz.openbmc_project.HwmonTempSensor \
       /.../CPU_Temp \
       xyz.openbmc_project.State.Decorator.OperationalStatus Functional
   ```

**可能原因：**

| 原因 | 解決方法 |
|------|----------|
| 電源狀態不符 | 開機後等待 |
| hwmon 讀取失敗 | 檢查 I2C 通訊 |
| ExternalSensor 逾時 | 確認外部來源正在推送數值 |
| 驅動程式問題 | 檢查核心日誌 |

---

### 閾值警報未觸發

**症狀：** 數值超過閾值但 `*AlarmHigh/Low` 仍為 false

**診斷步驟：**

1. **確認閾值已設定**
   ```bash
   busctl get-property xyz.openbmc_project.HwmonTempSensor \
       /.../Temp xyz.openbmc_project.Sensor.Threshold.Warning WarningHigh
   ```
   若為 `nan` 表示未設定

2. **檢查 JSON 配置中的 Thresholds**

3. **確認 Direction 和 Severity 設定正確**

---

### I2C 裝置未綁定

**症狀：** hwmon 裝置不存在

**診斷步驟：**

1. **檢查 I2C 裝置存在**
   ```bash
   ls /sys/bus/i2c/devices/1-0048/
   ```

2. **手動綁定裝置**
   ```bash
   echo "tmp75 0x48" > /sys/bus/i2c/devices/i2c-1/new_device
   ```

3. **檢查核心日誌**
   ```bash
   dmesg | grep -i tmp75
   ```

---

### 風扇感測器無讀數

**症狀：** 風扇轉速為 0 或 NaN

**診斷步驟：**

1. **檢查風扇存在偵測**
   ```bash
   gpioget gpiochip0 FAN1_PRESENT
   ```

2. **確認 hwmon 風扇檔案**
   ```bash
   cat /sys/class/hwmon/hwmonX/fan1_input
   ```

3. **檢查 PowerState**

---

## 日誌等級調整

增加詳細日誌輸出：

```bash
# 編輯 service 檔案（暫時）
systemctl edit xyz.openbmc_project.hwmontempsensor.service

# 添加環境變數
[Service]
Environment="SENSOR_VERBOSE=1"
```

---

## 服務重啟

```bash
# 重啟特定感測器服務
systemctl restart xyz.openbmc_project.hwmontempsensor.service

# 重啟 Entity-Manager
systemctl restart xyz.openbmc_project.EntityManager.service

# 重啟所有感測器服務
systemctl restart xyz.openbmc_project.adcsensor.service
systemctl restart xyz.openbmc_project.psusensor.service
systemctl restart xyz.openbmc_project.fansensor.service
```

---

## 常用除錯指令彙整

```bash
# 1. 檢查所有感測器服務
busctl list | grep -i sensor

# 2. 列出 EntityManager 配置
busctl tree xyz.openbmc_project.EntityManager

# 3. 掃描 I2C
i2cdetect -y 1

# 4. 列出 hwmon 裝置
ls /sys/class/hwmon/ | xargs -I{} sh -c 'echo -n "{}: "; cat /sys/class/hwmon/{}/name'

# 5. 讀取感測器數值
busctl call xyz.openbmc_project.HwmonTempSensor \
    /xyz/openbmc_project/sensors \
    org.freedesktop.DBus.ObjectManager GetManagedObjects

# 6. 監控數值變更
busctl monitor xyz.openbmc_project.HwmonTempSensor

# 7. 檢查 GPIO
gpioinfo
```

---

## 相關文件

- [架構概述](Architecture.md)
- [設定指南](ConfigurationGuide.md)
- [D-Bus API](DBusAPI.md)
- [Entity-Manager 整合](EntityManagerIntegration.md)
