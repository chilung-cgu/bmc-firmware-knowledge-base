# pldmtool 命令列工具

pldmtool 是用於測試和診斷 PLDM 功能的命令列工具。

---

## 概述

| 項目 | 說明 |
|------|------|
| **執行檔** | `/usr/bin/pldmtool` |
| **語言** | C++ |
| **輸出** | JSON 格式 |

---

## 基本用法

```bash
$ pldmtool -h
PLDM requester tool for OpenBMC
Usage: pldmtool [OPTIONS] SUBCOMMAND

Options:
  -h,--help                   顯示說明

Subcommands:
  raw                         發送原始請求
  base                        Base type 命令
  bios                        BIOS type 命令
  platform                    Platform type 命令
  fru                         FRU type 命令
  fw_update                   FW Update 命令
  oem-ibm                     IBM OEM 命令 (如啟用)
```

---

## Base 命令

```bash
# 查詢支援的 PLDM Types
$ pldmtool base GetPLDMTypes

# 取得 TID
$ pldmtool base GetTID

# 查詢版本
$ pldmtool base GetPLDMVersion -t platform
```

---

## Platform 命令

```bash
# 取得所有 PDR
$ pldmtool platform GetPDR -a

# 讀取 Sensor
$ pldmtool platform GetSensorReading -i 1

# 設定 Effecter
$ pldmtool platform SetStateEffecterStates -i 1 -c 1 -s 2
```

---

## BIOS 命令

```bash
# 取得 Attribute Value Table
$ pldmtool bios GetBIOSTable -t 2

# 設定屬性
$ pldmtool bios SetBIOSAttributeCurrentValue -a BootMode -d Enum -v UEFI
```

---

## FRU 命令

```bash
# 取得 FRU 元資料
$ pldmtool fru GetFRURecordTableMetadata

# 取得 FRU 表格
$ pldmtool fru GetFRURecordTable
```

---

## Raw 命令

發送原始 PLDM 請求：

```bash
# 格式: pldmtool raw -d <data>
$ pldmtool raw -d 0x80 0x00 0x04
Request Message:
08 01 80 00 04
Response Message:
08 01 00 00 04 00 1d 00 00 00 00 00 00 80
```

---

## 指定 EID

```bash
# 發送到特定 MCTP Endpoint
$ pldmtool base GetPLDMTypes -m 20
```

---

## 原始碼

| 檔案 | 說明 |
|------|------|
| `pldmtool/pldmtool.cpp` | 主程式 |
| `pldmtool/pldm_base_cmd.cpp` | Base 命令 |
| `pldmtool/pldm_platform_cmd.cpp` | Platform 命令 |
| `pldmtool/pldm_bios_cmd.cpp` | BIOS 命令 |
| `pldmtool/pldm_fru_cmd.cpp` | FRU 命令 |

---

*返回 [Home](Home.md)*
