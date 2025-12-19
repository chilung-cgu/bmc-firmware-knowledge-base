# Redfish System Service API

本文件說明 bmcweb 的 System 服務 API，用於管理主機系統。

---

## 📋 目錄

1. [服務概述](#服務概述)
2. [System 資源](#system-資源)
3. [電源控制](#電源控制)
4. [開機設定](#開機設定)
5. [硬體資訊](#硬體資訊)
6. [日誌服務](#日誌服務)

---

## 服務概述

System 服務提供主機系統管理功能：

- **電源控制** - 開機、關機、重啟
- **開機設定** - Boot Source、UEFI/Legacy
- **硬體資訊** - CPU、記憶體、PCIe 裝置
- **日誌存取** - SEL、Event Log

---

## System 資源

### 列出所有系統

```bash
curl -k -H "X-Auth-Token: $token" \
    https://${bmc}/redfish/v1/Systems
```

### 查詢系統詳情

```bash
curl -k -H "X-Auth-Token: $token" \
    https://${bmc}/redfish/v1/Systems/system
```

**回應範例：**
```json
{
    "@odata.id": "/redfish/v1/Systems/system",
    "@odata.type": "#ComputerSystem.v1_20_0.ComputerSystem",
    "Id": "system",
    "Name": "system",
    "SystemType": "Physical",
    "PowerState": "On",
    "Status": {
        "State": "Enabled",
        "Health": "OK"
    },
    "Boot": {
        "BootSourceOverrideEnabled": "Disabled",
        "BootSourceOverrideTarget": "None",
        "BootSourceOverrideMode": "UEFI"
    },
    "ProcessorSummary": {
        "Count": 2,
        "Model": "Intel Xeon"
    },
    "MemorySummary": {
        "TotalSystemMemoryGiB": 128
    },
    "Processors": {
        "@odata.id": "/redfish/v1/Systems/system/Processors"
    },
    "Memory": {
        "@odata.id": "/redfish/v1/Systems/system/Memory"
    },
    "Storage": {
        "@odata.id": "/redfish/v1/Systems/system/Storage"
    },
    "LogServices": {
        "@odata.id": "/redfish/v1/Systems/system/LogServices"
    },
    "Actions": {
        "#ComputerSystem.Reset": {
            "target": "/redfish/v1/Systems/system/Actions/ComputerSystem.Reset",
            "ResetType@Redfish.AllowableValues": [
                "On", "ForceOff", "GracefulShutdown", 
                "GracefulRestart", "ForceRestart", "ForceOn"
            ]
        }
    }
}
```

---

## 電源控制

### 電源狀態

| PowerState | 說明 |
|------------|------|
| `On` | 已開機 |
| `Off` | 已關機 |
| `PoweringOn` | 正在開機 |
| `PoweringOff` | 正在關機 |

### 電源操作

#### 開機
```bash
curl -k -H "X-Auth-Token: $token" \
    -H "Content-Type: application/json" \
    -X POST https://${bmc}/redfish/v1/Systems/system/Actions/ComputerSystem.Reset \
    -d '{"ResetType": "On"}'
```

#### 軟關機
```bash
curl -k -H "X-Auth-Token: $token" \
    -H "Content-Type: application/json" \
    -X POST https://${bmc}/redfish/v1/Systems/system/Actions/ComputerSystem.Reset \
    -d '{"ResetType": "GracefulShutdown"}'
```

#### 強制關機
```bash
curl -k -H "X-Auth-Token: $token" \
    -H "Content-Type: application/json" \
    -X POST https://${bmc}/redfish/v1/Systems/system/Actions/ComputerSystem.Reset \
    -d '{"ResetType": "ForceOff"}'
```

#### 重啟
```bash
curl -k -H "X-Auth-Token: $token" \
    -H "Content-Type: application/json" \
    -X POST https://${bmc}/redfish/v1/Systems/system/Actions/ComputerSystem.Reset \
    -d '{"ResetType": "GracefulRestart"}'
```

### ResetType 說明

| ResetType | 說明 |
|-----------|------|
| `On` | 開機 |
| `ForceOff` | 強制關機（相當於拔電源） |
| `GracefulShutdown` | 軟關機（ACPI shutdown） |
| `GracefulRestart` | 軟重啟 |
| `ForceRestart` | 強制重啟 |
| `ForceOn` | 強制開機 |

---

## 開機設定

### 查詢開機設定

```bash
curl -k -H "X-Auth-Token: $token" \
    https://${bmc}/redfish/v1/Systems/system | jq '.Boot'
```

### 設定開機來源

#### 一次性從 PXE 開機
```bash
curl -k -H "X-Auth-Token: $token" \
    -H "Content-Type: application/json" \
    -X PATCH https://${bmc}/redfish/v1/Systems/system \
    -d '{
        "Boot": {
            "BootSourceOverrideEnabled": "Once",
            "BootSourceOverrideTarget": "Pxe"
        }
    }'
```

#### 進入 BIOS 設定
```bash
curl -k -H "X-Auth-Token: $token" \
    -H "Content-Type: application/json" \
    -X PATCH https://${bmc}/redfish/v1/Systems/system \
    -d '{
        "Boot": {
            "BootSourceOverrideEnabled": "Continuous",
            "BootSourceOverrideTarget": "BiosSetup"
        }
    }'
```

#### 切換 UEFI/Legacy 模式
```bash
curl -k -H "X-Auth-Token: $token" \
    -H "Content-Type: application/json" \
    -X PATCH https://${bmc}/redfish/v1/Systems/system \
    -d '{
        "Boot": {
            "BootSourceOverrideMode": "UEFI"
        }
    }'
```

#### 清除開機覆蓋
```bash
curl -k -H "X-Auth-Token: $token" \
    -H "Content-Type: application/json" \
    -X PATCH https://${bmc}/redfish/v1/Systems/system \
    -d '{
        "Boot": {
            "BootSourceOverrideEnabled": "Disabled",
            "BootSourceOverrideTarget": "None"
        }
    }'
```

### Boot 參數說明

| 參數 | 值 | 說明 |
|------|-----|------|
| `BootSourceOverrideEnabled` | `Disabled` | 停用覆蓋 |
| | `Once` | 僅下次開機 |
| | `Continuous` | 持續覆蓋 |
| `BootSourceOverrideTarget` | `None` | 無 |
| | `Pxe` | 網路開機 |
| | `Hdd` | 硬碟 |
| | `Cd` | 光碟 |
| | `BiosSetup` | BIOS 設定 |
| `BootSourceOverrideMode` | `Legacy` | 傳統 BIOS |
| | `UEFI` | UEFI |

---

## 硬體資訊

### 處理器

```bash
# 列出處理器
curl -k -H "X-Auth-Token: $token" \
    https://${bmc}/redfish/v1/Systems/system/Processors

# 查詢特定處理器
curl -k -H "X-Auth-Token: $token" \
    https://${bmc}/redfish/v1/Systems/system/Processors/cpu0
```

### 記憶體

```bash
# 列出記憶體
curl -k -H "X-Auth-Token: $token" \
    https://${bmc}/redfish/v1/Systems/system/Memory

# 查詢特定 DIMM
curl -k -H "X-Auth-Token: $token" \
    https://${bmc}/redfish/v1/Systems/system/Memory/dimm0

# 記憶體指標
curl -k -H "X-Auth-Token: $token" \
    https://${bmc}/redfish/v1/Systems/system/Memory/dimm0/MemoryMetrics
```

### PCIe 裝置

```bash
# 列出 PCIe 裝置
curl -k -H "X-Auth-Token: $token" \
    https://${bmc}/redfish/v1/Systems/system/PCIeDevices
```

### 儲存

```bash
# 列出儲存控制器
curl -k -H "X-Auth-Token: $token" \
    https://${bmc}/redfish/v1/Systems/system/Storage

# 列出磁碟
curl -k -H "X-Auth-Token: $token" \
    https://${bmc}/redfish/v1/Systems/system/Storage/storage0/Drives
```

---

## 日誌服務

### 列出日誌服務

```bash
curl -k -H "X-Auth-Token: $token" \
    https://${bmc}/redfish/v1/Systems/system/LogServices
```

### Event Log

```bash
# 查詢事件日誌
curl -k -H "X-Auth-Token: $token" \
    https://${bmc}/redfish/v1/Systems/system/LogServices/EventLog/Entries

# 清除事件日誌
curl -k -H "X-Auth-Token: $token" \
    -H "Content-Type: application/json" \
    -X POST https://${bmc}/redfish/v1/Systems/system/LogServices/EventLog/Actions/LogService.ClearLog
```

### SEL (System Event Log)

```bash
# 查詢 SEL
curl -k -H "X-Auth-Token: $token" \
    https://${bmc}/redfish/v1/Systems/system/LogServices/SEL/Entries
```

---

## BIOS 配置

### 查詢 BIOS 屬性

```bash
curl -k -H "X-Auth-Token: $token" \
    https://${bmc}/redfish/v1/Systems/system/Bios
```

### 修改 BIOS 設定

```bash
curl -k -H "X-Auth-Token: $token" \
    -H "Content-Type: application/json" \
    -X PATCH https://${bmc}/redfish/v1/Systems/system/Bios/Settings \
    -d '{
        "Attributes": {
            "HyperThreading": "Enabled"
        }
    }'
```

---

## D-Bus 對應

| Redfish 資源 | D-Bus 服務 |
|--------------|------------|
| PowerState | `xyz.openbmc_project.State.Host` |
| Reset Actions | `xyz.openbmc_project.State.Host` |
| Boot 設定 | `xyz.openbmc_project.Control.Boot.*` |
| Processors | `xyz.openbmc_project.Inventory.*` |
| Memory | `xyz.openbmc_project.Inventory.*` |
| LogServices | `xyz.openbmc_project.Logging` |

---

## 相關文件

- [RedfishChassisService](RedfishChassisService.md) - Chassis 服務
- [RedfishManagerService](RedfishManagerService.md) - Manager 服務
- [QuickStart](QuickStart.md) - 快速入門

---

*最後更新：2025-12-19*
