# Configuration 配置指南

本文件詳述 PLDM 專案的所有組態選項，包含 Meson 建置選項、JSON 設定檔、以及編譯時常數。

---

## Meson 建置選項（`meson.options`）

### 功能開關

| 選項                        | 類型    | 預設     | 說明                     |
| --------------------------- | ------- | -------- | ------------------------ |
| `tests`                     | feature | enabled  | 建置測試                 |
| `utilities`                 | feature | enabled  | 建置除錯工具             |
| `libpldmresponder`          | feature | enabled  | 啟用 responder 函式庫    |
| `systemd`                   | feature | enabled  | Systemd 整合             |
| `softoff`                   | feature | enabled  | 建置軟關機應用           |
| `system-specific-bios-json` | feature | disabled | 支援不同系統的 BIOS 屬性 |
| `fw-update-pkg-inotify`     | feature | disabled | inotify 韌體包監視       |

### Transport 配置

| 選項                       | 類型  | 值                       | 說明              |
| -------------------------- | ----- | ------------------------ | ----------------- |
| `transport-implementation` | combo | `af-mctp` / `mctp-demux` | MCTP 傳輸實作選擇 |

> **`af-mctp`**：使用 Linux kernel AF_MCTP socket，現代首選。
> **`mctp-demux`**：透過 mctp-demux daemon 中轉，已逐漸淘汰。

### Terminus 配置

| 選項              | 類型 | 預設 | 範圍    | 說明                    |
| ----------------- | ---- | ---- | ------- | ----------------------- |
| `terminus-id`     | int  | 1    | 0-255   | BMC 的 PLDM Terminus ID |
| `terminus-handle` | int  | 1    | 0-65535 | BMC 的 Terminus Handle  |

### Timing 參數（DSP0240 Table 6）

| 選項                              | 類型 | 預設     | 範圍     | 說明                       |
| --------------------------------- | ---- | -------- | -------- | -------------------------- |
| `dbus-timeout-value`              | int  | **5**    | 3-10     | D-Bus 呼叫超時（秒）       |
| `heartbeat-timeout-seconds`       | int  | **120**  | —        | Host 心跳超時（秒）        |
| `number-of-request-retries`       | int  | **2**    | 2-30     | 請求重試次數               |
| `instance-id-expiration-interval` | int  | **5**    | 5-6      | Instance ID 過期間隔（秒） |
| `response-time-out`               | int  | **2000** | 300-4800 | 單次等待回應超時（毫秒）   |

> **DSP0240 規範依據**：Instance ID 在 6 秒後過期（Table 6）。`dbus-timeout-value` 設為 5 秒以確保 D-Bus 呼叫不會在 Instance ID 過期後仍在等待。`response-time-out` 設為 2 秒，搭配 2 次重試，總等待時間 = 2 × (2+1) = 6 秒 ≤ Instance ID 過期時間。

### Flight Recorder

| 選項                         | 類型 | 預設  | 範圍 | 說明             |
| ---------------------------- | ---- | ----- | ---- | ---------------- |
| `flightrecorder-max-entries` | int  | **0** | 0-30 | 記錄條數，0=停用 |

### Firmware Update 配置

| 選項                     | 類型 | 預設     | 範圍          | 說明                             |
| ------------------------ | ---- | -------- | ------------- | -------------------------------- |
| `maximum-transfer-size`  | int  | **4096** | 16-4294967295 | FD 每次請求的最大資料量（bytes） |
| `update-timeout-seconds` | int  | **60**   | 60-90         | FW 更新無命令超時（秒）          |

### SoftOff 配置

| 選項                      | 類型 | 預設     | 說明                           |
| ------------------------- | ---- | -------- | ------------------------------ |
| `softoff-timeout-seconds` | int  | **7200** | 等待 Host 優雅關機的超時（秒） |

### Sensor Polling 配置

| 選項                             | 類型 | 預設    | 範圍         | 說明                         |
| -------------------------------- | ---- | ------- | ------------ | ---------------------------- |
| `default-sensor-update-interval` | int  | **999** | 1-4294967295 | 預設 Sensor 輪詢間隔（ms）   |
| `sensor-polling-time`            | int  | **249** | 1-10000      | 通用 Sensor 輪詢計時器（ms） |

### OEM 選項

| 選項                  | 類型    | 預設    | 說明                                |
| --------------------- | ------- | ------- | ----------------------------------- |
| `oem-ibm`             | feature | enabled | IBM OEM 支援                        |
| `oem-ampere`          | feature | enabled | Ampere OEM 支援                     |
| `oem-meta`            | feature | enabled | Meta OEM 支援 (下游/特定分支擴充)   |
| `oem-nvidia`          | feature | enabled | NVIDIA OEM 支援 (下游/特定分支擴充) |
| `oem-ibm-dma-maxsize` | int     | 8384512 | IBM DMA 最大傳輸（4096-16773120）   |

---

## 編譯範例

```bash
# 基本編譯（af-mctp transport）
meson setup build \
    -Dtransport-implementation=af-mctp \
    -Dterminus-id=1

# 啟用 IBM OEM + Flight Recorder
meson setup build \
    -Dtransport-implementation=af-mctp \
    -Doem-ibm=enabled \
    -Dflightrecorder-max-entries=30

# 僅 IBM OEM（停用其他 OEM）
meson setup build \
    -Doem-ibm=enabled \
    -Doem-ampere=disabled

# 建置
ninja -C build

# 執行測試
ninja -C build test
```

---

## JSON 配置檔

### PDR JSON（`configurations/`）

Platform Handler 使用 JSON 定義 PDR，通常位於 OEM 特定目錄。

### FRU JSON（`fru_master.json`）

定義 D-Bus 物件路徑到 FRU 記錄欄位的映射。

### BIOS JSON

定義 BIOS 屬性（Enum、Integer、String）及其預設值。

### SoftOff JSON

軟關機超時配置。

---

## 相關文件

- [Pldmd](Pldmd.md) - 各選項在 pldmd 中的對應
- [Requester](Requester.md) - Timing 參數的實際使用
- [FirmwareUpdate](FirmwareUpdate.md) - FW Update 配置參數

---

_返回 [Home](Home.md)_
