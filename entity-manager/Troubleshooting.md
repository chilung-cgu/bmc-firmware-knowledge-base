# 故障排除

## 概述

本文件提供 Entity-Manager 及其相關元件的故障排除指南，包括常見問題、診斷方法和解決方案。

---

## 診斷工具

### 基本指令

| 指令 | 用途 |
|-----|------|
| `busctl tree <service>` | 查看服務的物件樹 |
| `busctl introspect <service> <path>` | 查看物件詳情 |
| `busctl get-property <service> <path> <iface> <prop>` | 讀取屬性 |
| `busctl monitor <service>` | 監聽服務信號 |
| `systemctl status <service>` | 檢查服務狀態 |
| `journalctl -u <service>` | 查看服務日誌 |

### Entity-Manager 相關服務

| 服務名稱 | systemd 服務 |
|---------|-------------|
| Entity-Manager | `xyz.openbmc_project.EntityManager.service` |
| FruDevice | `xyz.openbmc_project.FruDevice.service` |
| hwmontempsensor | `xyz.openbmc_project.hwmontempsensor.service` |
| ADCSensor | `xyz.openbmc_project.adcsensor.service` |
| FanSensor | `xyz.openbmc_project.fansensor.service` |

---

## 常見問題

### 1. Entity-Manager 未載入配置

#### 症狀

- D-Bus 上沒有預期的配置物件
- `busctl tree xyz.openbmc_project.EntityManager` 顯示為空

#### 診斷步驟

1. **檢查服務狀態**：

   ```bash
   systemctl status xyz.openbmc_project.EntityManager.service
   ```

2. **檢查日誌**：

   ```bash
   journalctl -u xyz.openbmc_project.EntityManager.service --no-pager
   ```

3. **確認配置檔存在**：

   ```bash
   ls /usr/share/entity-manager/configurations/
   ```

4. **驗證 JSON 語法**：

   ```bash
   python3 -m json.tool /usr/share/entity-manager/configurations/my_board.json
   ```

#### 常見原因與解決方案

| 原因 | 解決方案 |
|-----|---------|
| JSON 語法錯誤 | 修正 JSON 格式 |
| 配置檔不在正確位置 | 移動到 `/usr/share/entity-manager/configurations/` |
| Probe 不匹配 | 檢查 FruDevice 資料是否符合 Probe |
| 服務未啟動 | `systemctl start xyz.openbmc_project.EntityManager.service` |

---

### 2. Probe 不匹配

#### 症狀

- 配置檔存在但未載入
- 沒有錯誤訊息

#### 診斷步驟

1. **檢查 FruDevice 物件**：

   ```bash
   busctl tree xyz.openbmc_project.FruDevice
   ```

2. **查看 FRU 屬性**：

   ```bash
   busctl introspect xyz.openbmc_project.FruDevice \
       /xyz/openbmc_project/FruDevice/{DeviceName}
   ```

3. **比較 Probe 條件**：

   ```bash
   # 讀取 PRODUCT_PRODUCT_NAME
   busctl get-property xyz.openbmc_project.FruDevice \
       /xyz/openbmc_project/FruDevice/{DeviceName} \
       xyz.openbmc_project.FruDevice \
       PRODUCT_PRODUCT_NAME
   ```

4. **確認值完全匹配**（包括大小寫、空格）

#### 常見原因

- 屬性值不完全匹配（大小寫、空格、特殊字元）
- Probe 語法錯誤
- FRU 資料尚未發布

---

### 3. FruDevice 未發現裝置

#### 症狀

- `busctl tree xyz.openbmc_project.FruDevice` 沒有預期裝置

#### 診斷步驟

1. **檢查 I2C 匯流排**：

   ```bash
   ls /dev/i2c-*
   i2cdetect -y <bus_number>
   ```

2. **確認 EEPROM 可存取**：

   ```bash
   i2cget -y <bus> <address>
   ```

3. **檢查 FruDevice 服務**：

   ```bash
   systemctl status xyz.openbmc_project.FruDevice.service
   journalctl -u xyz.openbmc_project.FruDevice.service
   ```

4. **手動讀取 FRU**（如有 eeprom 裝置）：

   ```bash
   hexdump -C /sys/bus/i2c/devices/<bus>-00<addr>/eeprom | head
   ```

#### 常見原因

| 原因 | 解決方案 |
|-----|---------|
| I2C 控制器未啟用 | 檢查裝置樹 |
| EEPROM 位址錯誤 | 確認正確的 I2C 位址 |
| FRU 格式無效 | 檢查 FRU 資料格式 |
| 位址大小誤判 | 參見 [EEPROM 偵測](EEPROMDetection.md) |

---

### 4. 感測器未出現

#### 症狀

- Entity-Manager 配置已載入
- 但感測器未在 D-Bus 上出現

#### 診斷步驟

1. **確認配置已發布**：

   ```bash
   busctl tree xyz.openbmc_project.EntityManager
   busctl introspect xyz.openbmc_project.EntityManager \
       /xyz/openbmc_project/inventory/system/board/{Board}/{Sensor}
   ```

2. **檢查感測器服務**：

   ```bash
   systemctl status xyz.openbmc_project.hwmontempsensor.service
   journalctl -u xyz.openbmc_project.hwmontempsensor.service
   ```

3. **確認 hwmon 裝置存在**：

   ```bash
   ls /sys/class/hwmon/
   cat /sys/class/hwmon/hwmon*/name
   ```

4. **直接讀取感測器**：

   ```bash
   cat /sys/class/hwmon/hwmon*/temp1_input
   ```

#### 常見原因

| 原因 | 解決方案 |
|-----|---------|
| Type 不被感測器服務識別 | 確認 Type 名稱正確 |
| 驅動未載入 | 檢查 kernel 日誌 |
| 電源狀態條件不滿足 | 檢查 PowerState 設定 |
| 匯流排/位址錯誤 | 驗證 Bus 和 Address |

---

### 5. 高 CPU/IO 使用率

#### 症狀

- Entity-Manager 服務消耗大量 CPU 或 IO
- 系統響應緩慢

#### 診斷步驟

1. **確認是 Entity-Manager 造成**：

   ```bash
   top -p $(pgrep entity-manager)
   
   # 暫時停止服務
   systemctl stop xyz.openbmc_project.EntityManager.service
   # 觀察資源使用是否下降
   ```

2. **檢查 D-Bus 活動**：

   ```bash
   busctl monitor xyz.openbmc_project.EntityManager
   ```

#### 常見原因

- 持續的 D-Bus 介面變更觸發重新配置
- 配置檔案過多或過於複雜
- 迴圈 Probe 匹配

#### 解決方案

- 減少不必要的配置檔
- 避免在緊密迴圈中查詢 D-Bus
- 檢查是否有配置造成無限重新掃描

---

### 6. Entity-Manager 卡住

#### 症狀

- 服務運行但不響應
- D-Bus 呼叫逾時

#### 原因

連續在迴圈中呼叫 `busctl` 可能導致 Entity-Manager 卡住，因為每次 D-Bus 介面變更會觸發重新配置。

#### 解決方案

```bash
# 重新啟動服務
systemctl restart xyz.openbmc_project.EntityManager.service

# 避免在腳本中頻繁查詢 D-Bus
```

---

### 7. OR Probe 解析問題

#### 症狀

- 使用 OR 的 Probe 無法正確匹配
- 包含 "OR" 字串的屬性值造成問題

#### 原因

GitHub Issue #24：解析器可能將屬性值中的 "OR" 誤解為邏輯運算子。

#### 解決方案

- 避免在屬性值中使用 "OR"、"AND" 等保留字
- 如需 OR 邏輯，使用陣列語法：

  ```json
  {
      "Probe": [
          "xyz.openbmc_project.FruDevice({'PRODUCT': 'TypeA'})",
          "xyz.openbmc_project.FruDevice({'PRODUCT': 'TypeB'})"
      ]
  }
  ```

---

## 除錯日誌

### 啟用除錯輸出

較新版本的 Entity-Manager 支援除錯日誌。查看最新提交了解可用的除錯選項。

### 常用日誌指令

```bash
# Entity-Manager 日誌
journalctl -u xyz.openbmc_project.EntityManager.service -f

# FruDevice 日誌
journalctl -u xyz.openbmc_project.FruDevice.service -f

# 所有相關服務
journalctl -u 'xyz.openbmc_project.*' -f

# 特定時間範圍
journalctl -u xyz.openbmc_project.EntityManager.service --since "10 minutes ago"
```

---

## D-Bus 偵錯技巧

### 監聽特定物件變更

```bash
# 監聽 Entity-Manager 所有事件
busctl monitor xyz.openbmc_project.EntityManager

# 監聽特定服務的 InterfacesAdded
dbus-monitor --system "type='signal',member='InterfacesAdded'"
```

### 取得完整物件資訊

```bash
# 取得服務下所有物件
busctl call xyz.openbmc_project.EntityManager \
    / org.freedesktop.DBus.ObjectManager \
    GetManagedObjects
```

### 觸發重新掃描

```bash
# 重新啟動 FruDevice（觸發重新掃描）
systemctl restart xyz.openbmc_project.FruDevice.service

# 重新啟動 Entity-Manager
systemctl restart xyz.openbmc_project.EntityManager.service
```

---

## 驗證配置

### JSON 語法驗證

```bash
# 使用 Python
python3 -m json.tool myconfig.json > /dev/null && echo "Valid"

# 使用 jq
jq . myconfig.json > /dev/null && echo "Valid"
```

### 批量驗證所有配置

```bash
cd /usr/share/entity-manager/configurations/
for f in *.json; do
    python3 -m json.tool "$f" > /dev/null 2>&1 || echo "Invalid: $f"
done
```

---

## 快速參考

### 常見問題速查表

| 問題 | 檢查項目 | 指令 |
|-----|---------|------|
| 無配置 | 服務狀態、JSON 語法 | `systemctl status`, `python -m json.tool` |
| Probe 不匹配 | FRU 屬性值 | `busctl get-property` |
| 無 FRU | I2C 裝置 | `i2cdetect -y <bus>` |
| 無感測器 | 配置發布、hwmon | `busctl tree`, `ls /sys/class/hwmon/` |
| 高 CPU | 服務狀態 | `top`, `systemctl stop` |

---

## 取得幫助

### 社群資源

- [OpenBMC Discord](https://discord.gg/69Km47zH98)
- [OpenBMC 郵件列表](https://lists.ozlabs.org/listinfo/openbmc)
- [GitHub Issues](https://github.com/openbmc/entity-manager/issues)

### 提交問題報告

提交 Issue 時請包含：

1. OpenBMC 版本
2. 相關配置檔案
3. 完整日誌輸出
4. D-Bus 物件狀態
5. 重現步驟

---

## 下一步

- 了解 [架構概述](Architecture.md) 理解系統運作
- 查看 [設定指南](ConfigurationGuide.md) 正確配置
- 閱讀 [相容軟體](CompatibleSoftware.md) 了解整合驗證

---

> 📖 **參考**：[Entity-Manager GitHub Issues](https://github.com/openbmc/entity-manager/issues)
