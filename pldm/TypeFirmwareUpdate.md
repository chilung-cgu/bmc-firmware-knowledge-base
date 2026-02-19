# PLDM Type 5: Firmware Update

Firmware Update Type 提供標準化的韌體更新流程，定義於 DSP0267。

---

## 概述

| 欄位          | 值                                 |
| ------------- | ---------------------------------- |
| **Type Code** | 0x05                               |
| **規範**      | DSP0267 — PLDM for Firmware Update |
| **功能**      | 韌體查詢、傳輸、驗證、套用、啟用   |

---

## 角色與術語

| 角色                     | 說明                                           |
| ------------------------ | ---------------------------------------------- |
| **Update Agent (UA)**    | 發起更新的一方（BMC 扮演此角色）               |
| **Firmware Device (FD)** | 被更新的裝置（GPU、NIC、FPGA 等）              |
| **FDP**                  | Firmware Device Package — 標準化的韌體打包格式 |
| **Component**            | FD 中的一個可獨立更新的韌體元件                |
| **Descriptor**           | 用於識別 FD 的唯一描述符（如 PCI VID/PID）     |

---

## 完整命令列表

### 探索與查詢

| Command                    | Code | 方向  | 說明                                 |
| -------------------------- | ---- | ----- | ------------------------------------ |
| QueryDeviceIdentifiers     | 0x01 | UA→FD | 查詢裝置描述符（Vendor/Device ID）   |
| GetFirmwareParameters      | 0x02 | UA→FD | 取得韌體參數（元件表、capabilities） |
| QueryDownstreamDevices     | 0x03 | UA→FD | 查詢下游裝置                         |
| QueryDownstreamIdentifiers | 0x04 | UA→FD | 查詢下游裝置描述符                   |

### 更新流程

| Command                 | Code     | 方向      | 說明                      |
| ----------------------- | -------- | --------- | ------------------------- |
| RequestUpdate           | 0x10     | UA→FD     | 請求進入更新模式          |
| PassComponentTable      | 0x13     | UA→FD     | 傳遞要更新的元件列表      |
| UpdateComponent         | 0x14     | UA→FD     | 開始更新特定元件          |
| **RequestFirmwareData** | **0x15** | **FD→UA** | FD 主動向 UA 請求韌體資料 |
| **TransferComplete**    | **0x16** | **FD→UA** | FD 通知傳輸完成           |
| **VerifyComplete**      | **0x17** | **FD→UA** | FD 通知驗證完成           |
| **ApplyComplete**       | **0x18** | **FD→UA** | FD 通知套用完成           |
| ActivateFirmware        | 0x1A     | UA→FD     | 啟用新韌體                |
| GetStatus               | 0x1B     | UA→FD     | 查詢更新狀態              |
| CancelUpdateComponent   | 0x1C     | UA→FD     | 取消當前元件更新          |
| CancelUpdate            | 0x1D     | UA→FD     | 取消整個更新              |

> **面試重點**：注意 `RequestFirmwareData`、`TransferComplete`、`VerifyComplete`、`ApplyComplete` 是 **FD 主動發給 UA** 的命令（FD 作為 Requester）。這是 PLDM FW Update 的獨特設計——FD 控制資料傳輸節奏。

---

## FD 狀態機（DSP0267 Figure 3）

```mermaid
stateDiagram-v2
    [*] --> IDLE

    IDLE --> LEARN_COMPONENTS : RequestUpdate (Success)
    LEARN_COMPONENTS --> READY_XFER : PassComponentTable (Success)

    READY_XFER --> DOWNLOAD : UpdateComponent (Success)

    DOWNLOAD --> DOWNLOAD : RequestFirmwareData (進行中)
    DOWNLOAD --> VERIFY : TransferComplete (Success)

    VERIFY --> APPLY : VerifyComplete (Success)

    APPLY --> READY_XFER : ApplyComplete (還有更多 Component)
    APPLY --> ACTIVATE : ApplyComplete (最後一個 Component)

    ACTIVATE --> IDLE : ActivateFirmware / 自動重啟

    IDLE --> IDLE : CancelUpdate
    LEARN_COMPONENTS --> IDLE : CancelUpdate
    READY_XFER --> IDLE : CancelUpdate
    DOWNLOAD --> IDLE : CancelUpdateComponent / CancelUpdate
```

> **逐步說明（狀態機）：**
>
> 這張圖展示韌體裝置（FD）在更新過程中的狀態轉換：
>
> 1. **IDLE**：初始狀態，等待更新。
> 2. **LEARN_COMPONENTS**：收到 `RequestUpdate` 後，裝置開始「學習」即將更新的元件資訊。
> 3. **READY_XFER**：元件確認完畢，準備接收韌體資料。
> 4. **DOWNLOAD**：裝置透過 `RequestFirmwareData` 反覆請求 Update Agent 傳送韌體資料。這裡的特殊之處是：**是裝置主動拉資料，不是 BMC 主動推送**。
> 5. **VERIFY**：傳輸完成後，裝置驗證韌體的完整性（如 CRC 檢查）。
> 6. **APPLY**：驗證通過後，裝置將韌體寫入 Flash。如果還有更多元件，回到 READY_XFER；否則進入 ACTIVATE。
> 7. **ACTIVATE**：啟用新韌體，可能需要重啟裝置。
> 8. **CancelUpdate**：大部分狀態都可以取消更新，回到 IDLE。
>
> **白話總結**：就像手機更新——下載（DOWNLOAD）→ 驗證（VERIFY）→ 安裝（APPLY）→ 重啟（ACTIVATE）。

---

## Firmware Device Package 格式

```
+─────────────────────────────────────+
│ Package Header Information          │
│   ├── PackageHeaderIdentifier (UUID)│
│   ├── PackageHeaderFormatRevision   │
│   ├── PackageHeaderSize             │
│   ├── ComponentBitmapBitLength      │
│   ├── PackageVersionString          │
│   └── PackageReleaseDateTime        │
+─────────────────────────────────────+
│ Firmware Device Identification Area │
│   ├── DeviceIDRecordCount           │
│   └── DeviceIDRecord[]              │
│       ├── RecordLength              │
│       ├── DescriptorCount           │
│       ├── ComponentBitmap           │
│       └── Descriptors[]             │
│           ├── Type (IANA/PCI/UUID)  │
│           └── Data                  │
+─────────────────────────────────────+
│ Component Image Information Area    │
│   ├── ComponentImageCount           │
│   └── ComponentImageInfo[]          │
│       ├── Classification            │
│       ├── Identifier                │
│       ├── ComparisonStamp           │
│       ├── ComponentOptions          │
│       ├── ImageSize                 │
│       └── VersionString             │
+─────────────────────────────────────+
│ Component Image Data                │
│   └── Binary firmware images...     │
+─────────────────────────────────────+
│ Package Header Checksum (CRC-32)    │
+─────────────────────────────────────+
```

---

## 完整更新序列

```mermaid
sequenceDiagram
    participant UA as Update Agent (BMC)
    participant FD as Firmware Device

    Note over UA,FD: ── 階段 1：探索 ──
    UA->>FD: QueryDeviceIdentifiers
    FD-->>UA: Descriptors (PCI VID/PID)
    UA->>FD: GetFirmwareParameters
    FD-->>UA: ActiveComponents, PendingComponents

    Note over UA,FD: ── 階段 2：協商 ──
    UA->>FD: RequestUpdate (maxTransferSize, numComponents)
    FD-->>UA: FirmwareDeviceMetaDataLength, GetPackageDataCmd

    UA->>FD: PassComponentTable (for each component)
    FD-->>UA: CompResponse, CompResponseCode

    Note over UA,FD: ── 階段 3：傳輸 ──  (每個 Component)
    UA->>FD: UpdateComponent (classification, version, size)
    FD-->>UA: CompCompatibilityResponse

    loop 資料傳輸
        FD->>UA: RequestFirmwareData (offset, length)
        UA-->>FD: Component Image Data
    end

    FD->>UA: TransferComplete (transferResult)
    UA-->>FD: OK

    Note over UA,FD: ── 階段 4：驗證與套用 ──
    FD->>UA: VerifyComplete (verifyResult)
    UA-->>FD: OK
    FD->>UA: ApplyComplete (applyResult)
    UA-->>FD: OK

    Note over UA,FD: ── 階段 5：啟用 ──
    UA->>FD: ActivateFirmware (selfContainedActivation)
    FD-->>UA: EstimatedTimeForActivation
```

> **逐步說明：**
>
> 這張圖展示一次完整韌體更新的所有步驟：
>
> **階段 1：探索**
>
> - UA 查詢裝置的識別資訊（PCI VID/PID）和當前韌體參數。
>
> **階段 2：協商**
>
> - UA 告訴裝置「我要更新」，設定傳輸參數。
> - 透過 `PassComponentTable` 告訴裝置將要更新哪些元件。
>
> **階段 3：傳輸**（每個 Component 重複）
>
> - UA 發送 `UpdateComponent` 開始更新某個元件。
> - **重要**：是**裝置主動拉取資料**（`RequestFirmwareData`），不是 UA 主動推送。裝置決定每次要哪些資料、要多少。
> - 傳輸完成後，裝置發 `TransferComplete` 通知。
>
> **階段 4：驗證與套用**
>
> - 裝置驗證韌體完整性，然後寫入 Flash。
>
> **階段 5：啟用**
>
> - UA 發送 `ActivateFirmware`，裝置回傳預計啟用時間。
>
> **白話總結**：就像線上購物——瀏覽商品（探索）→ 確認訂單（協商）→ 出貨（傳輸）→ 驗貨（驗證）→ 上架（啟用）。

---

## Descriptor 類型

| Type                    | Code   | 說明                |
| ----------------------- | ------ | ------------------- |
| PCI Vendor ID           | 0x0000 | PCI 供應商 ID       |
| IANA Enterprise ID      | 0x0001 | IANA 企業 ID        |
| UUID                    | 0x0002 | 通用唯一識別符      |
| PnP Vendor ID           | 0x0003 | PnP 供應商 ID       |
| ACPI Vendor ID          | 0x0004 | ACPI 供應商 ID      |
| PCI Device ID           | 0x0100 | PCI 設備 ID         |
| PCI Subsystem Vendor ID | 0x0101 | PCI 子系統供應商 ID |
| PCI Subsystem ID        | 0x0102 | PCI 子系統 ID       |
| PCI Revision ID         | 0x0103 | PCI 修訂 ID         |
| Vendor Defined          | 0xFFFF | 廠商自訂            |

---

## pldmtool 使用

```bash
# 查詢裝置識別
$ pldmtool fw_update QueryDeviceIdentifiers -m 20

# 查詢韌體參數
$ pldmtool fw_update GetFirmwareParameters -m 20
```

---

## 相關文件

- [FirmwareUpdate](FirmwareUpdate.md) - OpenBMC 實作詳解
- [Requester](Requester.md) - 請求管理

---

_返回 [Home](Home.md)_
