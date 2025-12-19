# EEPROM 偵測模式

## 概述

fru-device 守護程式需要判斷 EEPROM 使用 8 位元還是 16 位元位址。這對於正確讀取 FRU 資料至關重要。本文件介紹兩種偵測模式及其適用場景。

---

## 背景知識

### EEPROM 位址類型

| 類型 | 位址大小 | 最大容量 | 常見型號 |
|-----|---------|---------|---------|
| 8 位元 | 1 位元組 | 256 bytes (2Kbit) | 24C02, 24C04 |
| 16 位元 | 2 位元組 | 64KB+ (512Kbit+) | 24C64, 24C256 |

### 問題

- 無法單純從 I2C 位址判斷 EEPROM 類型
- 使用錯誤的位址大小會導致讀取失敗或錯誤資料
- 需要透過探測行為來判斷

---

## MODE-1（傳統方式）

### 演算法

MODE-1 使用 1 位元組寫入操作和 8 次 1 位元組讀取來判斷：

```
操作流程：
1. 發送 1 位元組寫入（設定位址為 0）
2. 執行 8 次 1 位元組讀取
3. 比較讀取的 8 個位元組
```

### 判斷邏輯

```
如果 8 個位元組全部相同:
    → 判定為 8 位元位址裝置
否則:
    → 判定為 16 位元位址裝置
```

### 原理

- **8 位元裝置**：位址指標在每次讀取後自動遞增，但只從同一位置讀取
- **16 位元裝置**：預期會從不同位置讀取，資料應不同

### I2C 操作序列

```text
| Start | SlaveAddr + W | 0x00 | Stop |
| Start | SlaveAddr + R | data0 | Stop |
| Start | SlaveAddr + R | data1 | Stop |
| Start | SlaveAddr + R | data2 | Stop |
| Start | SlaveAddr + R | data3 | Stop |
| Start | SlaveAddr + R | data4 | Stop |
| Start | SlaveAddr + R | data5 | Stop |
| Start | SlaveAddr + R | data6 | Stop |
| Start | SlaveAddr + R | data7 | Stop |
```

### 問題

#### 問題 1：相同資料誤判

如果 16 位元 EEPROM 在位址 0-7 恰好存有相同資料，會被錯誤判定為 8 位元裝置。

#### 問題 2：廠商行為差異

某些 EEPROM（如 ONSEMI 產品）在收到 1 位元組位址時的行為與預期不同：

- 這些裝置即使是 16 位元裝置，也會從同一位置返回資料
- 導致被錯誤識別為 8 位元裝置

---

## MODE-2（改進方式）

### 演算法

MODE-2 改用 2 位元組寫入操作，但**禁止 STOP 條件**：

```
操作流程：
1. 發送 2 位元組位址 (0x00, 0x00)，不發送 STOP
2. 立即發送重新啟動 + 讀取
3. 重複 8 次，每次遞增第二個位址位元組
```

### I2C 操作序列

```text
| Start | SlaveAddr + W | 0x00 | 0x00 | NO STOP | Start | SlaveAddr + R | data | Stop |
| Start | SlaveAddr + W | 0x00 | 0x01 | NO STOP | Start | SlaveAddr + R | data | Stop |
| Start | SlaveAddr + W | 0x00 | 0x02 | NO STOP | Start | SlaveAddr + R | data | Stop |
| Start | SlaveAddr + W | 0x00 | 0x03 | NO STOP | Start | SlaveAddr + R | data | Stop |
| Start | SlaveAddr + W | 0x00 | 0x04 | NO STOP | Start | SlaveAddr + R | data | Stop |
| Start | SlaveAddr + W | 0x00 | 0x05 | NO STOP | Start | SlaveAddr + R | data | Stop |
| Start | SlaveAddr + W | 0x00 | 0x06 | NO STOP | Start | SlaveAddr + R | data | Stop |
| Start | SlaveAddr + W | 0x00 | 0x07 | NO STOP | Start | SlaveAddr + R | data | Stop |
```

### 判斷邏輯

```
如果 8 個位元組全部相同:
    → 判定為 8 位元位址裝置
否則:
    → 判定為 16 位元位址裝置
```

### 原理

#### 8 位元裝置

- 裝置只解釋第一個資料位元組 (0x00) 作為位址
- 第二個資料位元組被視為要寫入的資料（但因無 STOP 不會寫入）
- 每次讀取都從位址 0x00 返回相同資料

#### 16 位元裝置

- 裝置解釋兩個位元組作為完整位址
- 第一次讀取位址 0x0000
- 第二次讀取位址 0x0001
- 每次讀取從不同位置返回資料

### 為什麼禁止 STOP 條件？

> ⚠️ **重要**：在第二個位元組後發送 STOP 條件會觸發 8 位元裝置的寫入週期，可能**損壞 EEPROM 內容**！

根據 I2C 規範，8 位元 EEPROM 在收到 STOP 時會開始內部寫入週期。MODE-2 使用重新啟動 (Repeated Start) 來避免這個問題。

---

## 比較

| 特性 | MODE-1 | MODE-2 |
|-----|--------|--------|
| 準確度 | 較低 | 較高 |
| 廠商相容性 | 受限 | 較廣 |
| 風險 | 誤判風險 | 較低 |
| 複雜度 | 簡單 | 較複雜 |
| I2C 操作 | 標準讀寫 | 需支援無 STOP 寫入 |

### MODE-2 的限制

MODE-2 仍有一個與 MODE-1 相同的邊緣情況：

- 如果 16 位元 EEPROM 在位址 0x0000-0x0007 存有相同資料
- 仍會被誤判為 8 位元裝置

但這種情況比 MODE-1 的誤判情況更少見。

---

## I2C 規範參考

### Note 1 (Section 3.1.10)

> "Combined formats can be used, for example, to control a serial memory. The internal memory location must be written during the first data byte. After the START condition and slave address is repeated, data can be transferred."

MODE-2 利用此特性，先寫入位址，再透過重新啟動讀取資料。

### Note 2 (Section 3.1.10)

> "All decisions on auto-increment or decrement of previously accessed memory locations, etc., are taken by the designer of the device."

這說明位址指標的自動遞增行為是製造商特定的，不是 I2C 標準規定的。這是造成 MODE-1 問題的根本原因。

---

## 實作建議

### 預設使用 MODE-2

對於新部署，建議使用 MODE-2 以獲得更好的相容性。

### 例外處理

如果已知 EEPROM 類型，可以：

1. 在裝置樹中明確指定
2. 使用配置選項覆蓋自動偵測

### 驗證偵測結果

```bash
# 檢查 fru-device 偵測到的裝置
busctl tree xyz.openbmc_project.FruDevice

# 查看特定裝置的 BUS 和 ADDRESS
busctl get-property xyz.openbmc_project.FruDevice \
    /xyz/openbmc_project/FruDevice/My_Device \
    xyz.openbmc_project.FruDevice \
    BUS ADDRESS
```

### 手動驗證 EEPROM 類型

```bash
# 嘗試以 8 位元方式讀取
i2cget -y <bus> <addr> 0x00

# 嘗試以 16 位元方式讀取（使用 i2ctransfer）
i2ctransfer -y <bus> w2@<addr> 0x00 0x00 r1
```

---

## 故障排除

### 症狀：FRU 解析失敗

**可能原因**：EEPROM 位址大小誤判

**診斷**：

```bash
# 查看日誌
journalctl -u xyz.openbmc_project.FruDevice.service | grep -i address
```

**解決方案**：

1. 確認 EEPROM 實際類型
2. 如有需要，調整偵測模式配置

### 症狀：某些廠商 EEPROM 無法識別

**可能原因**：廠商特定的 I2C 行為

**解決方案**：

1. 確認正在使用 MODE-2
2. 查閱 EEPROM 資料手冊
3. 必要時提交問題報告

---

## 下一步

- 了解 [FruDevice 守護程式](FruDevice.md) 的完整運作方式
- 查看 [故障排除](Troubleshooting.md) 解決 EEPROM 相關問題
- 閱讀 [設定指南](ConfigurationGuide.md) 了解 EEPROM 配置

---

> 📖 **參考**：
> - [I2C 規範 (NXP UM10204)](https://www.nxp.com/docs/en/user-guide/UM10204.pdf)
> - [Entity-Manager EEPROM 偵測模式文件](https://github.com/openbmc/entity-manager/blob/master/docs/address_size_detection_modes.md)
