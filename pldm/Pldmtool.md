# pldmtool 命令列工具

pldmtool 是 PLDM 的診斷和測試工具，作為 PLDM Requester 向 pldmd 或遠端端點發送 PLDM 命令。

---

## 概述

| 項目 | 說明 |
|------|------|
| **執行檔** | `/usr/bin/pldmtool` |
| **位置** | `pldmtool/` |
| **語言** | C++，使用 CLI11 框架 |
| **傳輸** | 透過 `PldmTransport` 與 pldmd 通訊 |
| **預設 EID** | 8（可用 `-m` 指定） |

---

## 子命令總覽

```bash
$ pldmtool -h
PLDM requester tool for OpenBMC
Usage: pldmtool [OPTIONS] SUBCOMMAND

Options:
  -h,--help     Print this help message and exit

Subcommands:
  raw           send a raw request and print response
  base          base type command
  bios          bios type command
  platform      platform type command
  fru           FRU type command
  oem-ibm       oem type command
```

---

## Base 命令（Type 0）

```bash
$ pldmtool base -h
Subcommands:
  GetPLDMTypes      get pldm supported types
  GetPLDMVersion    get version of a certain type
  GetTID            get Terminus ID (TID)
  GetPLDMCommands   get supported commands of pldm type
```

### 使用範例

```bash
# 取得支援的 PLDM Types
$ pldmtool base GetPLDMTypes
[
    { "PLDM Type": "base",     "PLDM Type Code": 0 },
    { "PLDM Type": "platform", "PLDM Type Code": 2 },
    { "PLDM Type": "bios",     "PLDM Type Code": 3 },
    { "PLDM Type": "fru",      "PLDM Type Code": 4 }
]

# 取得 Terminus ID
$ pldmtool base GetTID
{ "TID": 1 }

# 取得 Base Type 支援的命令
$ pldmtool base GetPLDMCommands -t 0
```

---

## Platform 命令（Type 2）

```bash
$ pldmtool platform -h
Subcommands:
  GetPDR                    get platform descriptor records
  SetStateEffecterStates    set effecter states
  SetNumericEffecterValue   set numeric effecter value
  GetStateSensorReadings    get state sensor readings
  GetNumericEffecterValue   get numeric effecter value
  GetSensorReading          get sensor reading
```

### PDR 操作（最常用）

```bash
# 取得第一筆 PDR
$ pldmtool platform GetPDR -d 0

# 遍歷所有 PDR
$ pldmtool platform GetPDR -d 0
# 然後用 nextRecordHandle 繼續...

# 指定遠端端點
$ pldmtool platform GetPDR -d 0 -m 8
```

### Effecter/Sensor 操作

```bash
# 設定 State Effecter
$ pldmtool platform SetStateEffecterStates -i <effecterId> -c 1 -d <requestSet,effecterState>

# 取得 State Sensor 讀數
$ pldmtool platform GetStateSensorReadings -i <sensorId>

# 取得 Numeric Sensor 讀數
$ pldmtool platform GetSensorReading -i <sensorId>
```

---

## BIOS 命令（Type 3）

```bash
$ pldmtool bios -h
Subcommands:
  GetDateTime               get date time
  SetDateTime               set date time
  GetBIOSTable              get bios table
  GetBIOSAttributeCurrentValueByHandle  get attribute value
  SetBIOSAttributeCurrentValue          set attribute value
```

### 使用範例

```bash
# 取得日期時間
$ pldmtool bios GetDateTime

# 取得 BIOS 屬性表
$ pldmtool bios GetBIOSTable -t 0    # StringTable
$ pldmtool bios GetBIOSTable -t 1    # AttributeTable
$ pldmtool bios GetBIOSTable -t 2    # AttributeValueTable
```

---

## FRU 命令（Type 4）

```bash
$ pldmtool fru -h
Subcommands:
  GetFRURecordTableMetadata  get FRU record table metadata
  GetFRURecordTable          get FRU record table
  GetFRURecordByOption       get FRU record by option
```

### 使用範例

```bash
# 取得 FRU 表格元資料
$ pldmtool fru GetFRURecordTableMetadata

# 取得完整 FRU 表格
$ pldmtool fru GetFRURecordTable
```

---

## Raw 命令（任意 PLDM 命令）

當命令未被 pldmtool 實作時，可直接發送原始 PLDM 訊息：

```bash
# 格式
pldmtool raw --data 0x80 <pldmType> <cmdType> <payloadReq>

# 範例：手動發送 GetTID (Type=0x00, Cmd=0x02)
$ pldmtool raw -d 0x80 0x00 0x02
Request Message:  08 01 80 00 02
Response Message: 08 01 00 00 02 00 01

# 指定遠端端點
$ pldmtool raw -d 0x80 0x00 0x04 0x00 0x00 -m 0x08
```

### 請求格式

```
0x80 <pldmType> <cmdType> [payloadReq...]
 │       │          │          └─ 依命令定義的 payload
 │       │          └─ PLDM 命令代碼
 │       └─ PLDM Type (0=Base, 2=Platform, 3=BIOS, 4=FRU)
 └─ Request bit (固定 0x80)
```

### 回應格式

```
<instanceId> <hdrVersion> <pldmType> <cmdType> <completionCode> [payloadResp...]
```

---

## 通用選項

| 選項 | 說明 |
|------|------|
| `-m, --mctp_eid <EID>` | 指定 MCTP 端點 ID（預設 8） |
| `-v, --verbose` | 啟用詳細輸出 |
| `-h, --help` | 顯示說明 |

---

## 錯誤處理

錯誤訊息包含 rc（return code）和 cc（completion code）：

```bash
$ pldmtool platform GetPDR -d 17
Response Message Error: rc=0, cc=130
```

Completion code 定義在 libpldm 的各 type header 中（如 `platform.h`）。

---

## 原始碼結構

| 檔案 | 說明 |
|------|------|
| `pldmtool.cpp` | 主程式入口，解析子命令 |
| `pldm_cmd_helper.cpp/hpp` | 通用命令輔助函式 |
| `pldm_base_cmd.cpp/hpp` | Base Type 命令實作 |
| `pldm_platform_cmd.cpp/hpp` | Platform Type 命令實作 |
| `pldm_bios_cmd.cpp/hpp` | BIOS Type 命令實作 |
| `pldm_fru_cmd.cpp/hpp` | FRU Type 命令實作 |
| `pldm_fw_update_cmd.cpp/hpp` | FW Update Type 命令實作 |
| `oem/ibm/` | IBM OEM 特有命令 |

---

## 相關文件

- [Pldmd](Pldmd.md) - pldmd 守護程式
- [Troubleshooting](Troubleshooting.md) - 故障排除（常用 pldmtool 診斷）

---

*返回 [Home](Home.md)*
