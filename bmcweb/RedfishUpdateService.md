# Redfish UpdateService API

本文件說明 bmcweb 的更新服務 API，用於韌體更新。

---

## 📋 目錄

1. [服務概述](#服務概述)
2. [韌體清單](#韌體清單)
3. [更新方法](#更新方法)
4. [更新流程](#更新流程)
5. [任務追蹤](#任務追蹤)

---

## 服務概述

UpdateService 提供韌體更新功能：

- **韌體清單** - 列出已安裝的韌體
- **HTTP Push** - 直接上傳韌體檔案
- **Multipart Upload** - 多部分表單上傳
- **SimpleUpdate** - TFTP/HTTP 遠端更新

---

## 查詢 UpdateService

```bash
curl -k -H "X-Auth-Token: $token" \
    https://${bmc}/redfish/v1/UpdateService
```

**回應範例：**
```json
{
    "@odata.id": "/redfish/v1/UpdateService",
    "@odata.type": "#UpdateService.v1_11_0.UpdateService",
    "Id": "UpdateService",
    "Name": "Update Service",
    "ServiceEnabled": true,
    "HttpPushUri": "/redfish/v1/UpdateService/update",
    "MultipartHttpPushUri": "/redfish/v1/UpdateService/update",
    "FirmwareInventory": {
        "@odata.id": "/redfish/v1/UpdateService/FirmwareInventory"
    },
    "Actions": {
        "#UpdateService.SimpleUpdate": {
            "target": "/redfish/v1/UpdateService/Actions/UpdateService.SimpleUpdate",
            "TransferProtocol@Redfish.AllowableValues": [
                "TFTP", "HTTP", "HTTPS"
            ]
        }
    }
}
```

---

## 韌體清單

### 列出韌體

```bash
curl -k -H "X-Auth-Token: $token" \
    https://${bmc}/redfish/v1/UpdateService/FirmwareInventory
```

### 查詢韌體詳情

```bash
curl -k -H "X-Auth-Token: $token" \
    https://${bmc}/redfish/v1/UpdateService/FirmwareInventory/bmc_active
```

**回應範例：**
```json
{
    "@odata.id": "/redfish/v1/UpdateService/FirmwareInventory/bmc_active",
    "@odata.type": "#SoftwareInventory.v1_4_0.SoftwareInventory",
    "Id": "bmc_active",
    "Name": "BMC Firmware",
    "Status": {
        "State": "Enabled",
        "Health": "OK"
    },
    "Updateable": true,
    "Version": "2.12.0",
    "SoftwareId": "openbmc-phosphor-image"
}
```

### 典型韌體項目

| ID | 說明 |
|-----|------|
| `bmc_active` | 當前運行的 BMC 韌體 |
| `bmc_backup` | 備份的 BMC 韌體 |
| `bios_active` | 當前的 BIOS 韌體 |

---

## 更新方法

### 方法 1：HTTP Push

直接 POST 韌體檔案：

```bash
# 取得 HttpPushUri
uri=$(curl -k -H "X-Auth-Token: $token" \
    https://${bmc}/redfish/v1/UpdateService | jq -r '.HttpPushUri')

# 上傳韌體
curl -k -H "X-Auth-Token: $token" \
    -H "Content-Type: application/octet-stream" \
    -X POST -T firmware.tar \
    https://${bmc}${uri}
```

### 方法 2：Multipart Upload

使用多部分表單，可指定額外參數：

```bash
# 取得 MultipartHttpPushUri
uri=$(curl -k -H "X-Auth-Token: $token" \
    https://${bmc}/redfish/v1/UpdateService | jq -r '.MultipartHttpPushUri')

# 上傳並指定立即套用
curl -k -H "X-Auth-Token: $token" \
    -H "Content-Type: multipart/form-data" \
    -X POST \
    -F 'UpdateParameters={"Targets":["/redfish/v1/Managers/bmc"],"@Redfish.OperationApplyTime":"Immediate"};type=application/json' \
    -F "UpdateFile=@firmware.tar;type=application/octet-stream" \
    https://${bmc}${uri}
```

### 方法 3：SimpleUpdate (TFTP)

從 TFTP 伺服器下載韌體：

```bash
curl -k -H "X-Auth-Token: $token" \
    -H "Content-Type: application/json" \
    -X POST https://${bmc}/redfish/v1/UpdateService/Actions/UpdateService.SimpleUpdate \
    -d '{
        "TransferProtocol": "TFTP",
        "ImageURI": "192.168.1.10/firmware.tar"
    }'
```

### 方法 4：SimpleUpdate (HTTP/HTTPS)

從 HTTP 伺服器下載韌體：

```bash
curl -k -H "X-Auth-Token: $token" \
    -H "Content-Type: application/json" \
    -X POST https://${bmc}/redfish/v1/UpdateService/Actions/UpdateService.SimpleUpdate \
    -d '{
        "ImageURI": "https://server.example.com/firmware.tar"
    }'
```

---

## 更新流程

### ApplyTime 選項

| 選項 | 說明 |
|------|------|
| `Immediate` | 立即套用 |
| `OnReset` | 下次重啟時套用 |

### 完整更新流程

```
1. 上傳韌體
   ────────────▶ 驗證韌體映像
                     │
2. 韌體驗證完成  ◀────┘
   ────────────▶ 寫入韌體到快閃記憶體
                     │
3. 寫入完成      ◀────┘
   ────────────▶ (ApplyTime=Immediate) 重啟 BMC
                  (ApplyTime=OnReset)  等待手動重啟
                     │
4. 重啟後新韌體生效 ◀─┘
```

---

## 任務追蹤

韌體更新會建立非同步任務，可透過 TaskService 追蹤進度：

### 查詢任務

```bash
curl -k -H "X-Auth-Token: $token" \
    https://${bmc}/redfish/v1/TaskService/Tasks
```

### 任務狀態

```json
{
    "@odata.id": "/redfish/v1/TaskService/Tasks/0",
    "@odata.type": "#Task.v1_6_0.Task",
    "Id": "0",
    "Name": "Task 0",
    "TaskState": "Running",
    "TaskStatus": "OK",
    "PercentComplete": 50,
    "Messages": [
        {
            "Message": "Firmware update in progress"
        }
    ]
}
```

### TaskState 說明

| TaskState | 說明 |
|-----------|------|
| `New` | 新建立 |
| `Starting` | 開始中 |
| `Running` | 執行中 |
| `Completed` | 已完成 |
| `Exception` | 發生錯誤 |

---

## 更新範例

### 完整 BMC 韌體更新

```bash
#!/bin/bash
export bmc=192.168.1.100
export token=$(curl -k -s -H "Content-Type: application/json" \
    -X POST https://${bmc}/login \
    -d '{"username":"root","password":"0penBmc"}' \
    | jq -r '.token')

# 上傳韌體
echo "Uploading firmware..."
response=$(curl -k -s -H "X-Auth-Token: $token" \
    -H "Content-Type: application/octet-stream" \
    -X POST -T obmc-phosphor-image.tar \
    https://${bmc}/redfish/v1/UpdateService/update)

echo "Response: $response"

# 追蹤任務
task_uri=$(echo $response | jq -r '."@odata.id"')
while true; do
    status=$(curl -k -s -H "X-Auth-Token: $token" \
        https://${bmc}${task_uri} | jq -r '.TaskState')
    echo "Task Status: $status"
    
    if [ "$status" == "Completed" ]; then
        echo "Update completed successfully!"
        break
    elif [ "$status" == "Exception" ]; then
        echo "Update failed!"
        break
    fi
    
    sleep 5
done
```

---

## 注意事項

> [!WARNING]
> 韌體更新期間請勿中斷電源或網路連線。

> [!IMPORTANT]
> 更新前請確認:
> - 韌體版本相容性
> - 有足夠的儲存空間
> - 已備份重要配置

---

## 相關文件

- [RedfishOverview](RedfishOverview.md) - Redfish 概述
- [RedfishManagerService](RedfishManagerService.md) - Manager 服務
- [Troubleshooting](Troubleshooting.md) - 故障排除

---

*最後更新：2025-12-19*
