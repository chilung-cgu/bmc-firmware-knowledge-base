# 快速入門指南

本文件提供 bmcweb Redfish API 的快速入門教學和常用命令範例。

---

## 📋 目錄

1. [準備工作](#準備工作)
2. [認證](#認證)
3. [常用操作](#常用操作)
4. [進階操作](#進階操作)

---

## 準備工作

### 設定環境變數

```bash
# 設定 BMC IP
export bmc=192.168.1.100
```

### 驗證連線

```bash
# 測試連線（無需認證）
curl -k https://${bmc}/redfish/v1
```

---

## 認證

### 方法 1：取得 Token

```bash
# 登入取得 Token
export token=$(curl -k -s -H "Content-Type: application/json" \
    -X POST https://${bmc}/login \
    -d '{"username":"root","password":"0penBmc"}' \
    | grep -o '"token":"[^"]*"' | cut -d'"' -f4)

# 驗證 Token
echo $token
```

### 方法 2：直接使用 Basic Auth

```bash
curl -k -u root:0penBmc https://${bmc}/redfish/v1/Systems
```

---

## 常用操作

### 查詢系統資訊

```bash
# Service Root
curl -k -H "X-Auth-Token: $token" https://${bmc}/redfish/v1

# 系統狀態
curl -k -H "X-Auth-Token: $token" https://${bmc}/redfish/v1/Systems/system

# BMC 資訊
curl -k -H "X-Auth-Token: $token" https://${bmc}/redfish/v1/Managers/bmc

# 機箱資訊
curl -k -H "X-Auth-Token: $token" https://${bmc}/redfish/v1/Chassis/chassis
```

### 電源控制

```bash
# 查詢電源狀態
curl -k -H "X-Auth-Token: $token" https://${bmc}/redfish/v1/Systems/system \
    | jq '.PowerState'

# 開機
curl -k -H "X-Auth-Token: $token" \
    -H "Content-Type: application/json" \
    -X POST https://${bmc}/redfish/v1/Systems/system/Actions/ComputerSystem.Reset \
    -d '{"ResetType": "On"}'

# 軟關機
curl -k -H "X-Auth-Token: $token" \
    -H "Content-Type: application/json" \
    -X POST https://${bmc}/redfish/v1/Systems/system/Actions/ComputerSystem.Reset \
    -d '{"ResetType": "GracefulShutdown"}'

# 強制關機
curl -k -H "X-Auth-Token: $token" \
    -H "Content-Type: application/json" \
    -X POST https://${bmc}/redfish/v1/Systems/system/Actions/ComputerSystem.Reset \
    -d '{"ResetType": "ForceOff"}'

# 重啟
curl -k -H "X-Auth-Token: $token" \
    -H "Content-Type: application/json" \
    -X POST https://${bmc}/redfish/v1/Systems/system/Actions/ComputerSystem.Reset \
    -d '{"ResetType": "GracefulRestart"}'
```

### 感測器讀取

```bash
# 列出所有感測器
curl -k -H "X-Auth-Token: $token" \
    https://${bmc}/redfish/v1/Chassis/chassis/Sensors

# 查詢特定感測器
curl -k -H "X-Auth-Token: $token" \
    https://${bmc}/redfish/v1/Chassis/chassis/Sensors/temperature_CPU0
```

### 日誌操作

```bash
# 查詢事件日誌
curl -k -H "X-Auth-Token: $token" \
    https://${bmc}/redfish/v1/Systems/system/LogServices/EventLog/Entries

# 清除日誌
curl -k -H "X-Auth-Token: $token" \
    -H "Content-Type: application/json" \
    -X POST https://${bmc}/redfish/v1/Systems/system/LogServices/EventLog/Actions/LogService.ClearLog
```

---

## 進階操作

### BMC 管理

```bash
# 重啟 BMC
curl -k -H "X-Auth-Token: $token" \
    -H "Content-Type: application/json" \
    -X POST https://${bmc}/redfish/v1/Managers/bmc/Actions/Manager.Reset \
    -d '{"ResetType": "GracefulRestart"}'

# 恢復原廠設定（謹慎！）
curl -k -H "X-Auth-Token: $token" \
    -H "Content-Type: application/json" \
    -X POST https://${bmc}/redfish/v1/Managers/bmc/Actions/Manager.ResetToDefaults \
    -d '{"ResetType": "ResetAll"}'
```

### 帳戶管理

```bash
# 列出帳戶
curl -k -H "X-Auth-Token: $token" \
    https://${bmc}/redfish/v1/AccountService/Accounts

# 修改密碼
curl -k -H "X-Auth-Token: $token" \
    -H "Content-Type: application/json" \
    -X PATCH https://${bmc}/redfish/v1/AccountService/Accounts/root \
    -d '{"Password": "NewPassword123!"}'
```

### 網路配置

```bash
# 查詢網路設定
curl -k -H "X-Auth-Token: $token" \
    https://${bmc}/redfish/v1/Managers/bmc/EthernetInterfaces/eth0

# 設定 NTP
curl -k -H "X-Auth-Token: $token" \
    -H "Content-Type: application/json" \
    -X PATCH https://${bmc}/redfish/v1/Managers/bmc/NetworkProtocol \
    -d '{"NTP": {"NTPServers": ["time.nist.gov"], "ProtocolEnabled": true}}'
```

### 韌體更新

```bash
# 取得更新 URI
uri=$(curl -k -H "X-Auth-Token: $token" \
    https://${bmc}/redfish/v1/UpdateService | jq -r '.HttpPushUri')

# 上傳韌體
curl -k -H "X-Auth-Token: $token" \
    -H "Content-Type: application/octet-stream" \
    -X POST -T firmware.tar \
    https://${bmc}${uri}
```

### 開機設定

```bash
# 設定從 PXE 開機（一次性）
curl -k -H "X-Auth-Token: $token" \
    -H "Content-Type: application/json" \
    -X PATCH https://${bmc}/redfish/v1/Systems/system \
    -d '{"Boot": {"BootSourceOverrideEnabled": "Once", "BootSourceOverrideTarget": "Pxe"}}'

# 進入 BIOS 設定
curl -k -H "X-Auth-Token: $token" \
    -H "Content-Type: application/json" \
    -X PATCH https://${bmc}/redfish/v1/Systems/system \
    -d '{"Boot": {"BootSourceOverrideEnabled": "Continuous", "BootSourceOverrideTarget": "BiosSetup"}}'
```

---

## 實用腳本

### 監控電源狀態

```bash
#!/bin/bash
while true; do
    state=$(curl -s -k -H "X-Auth-Token: $token" \
        https://${bmc}/redfish/v1/Systems/system | jq -r '.PowerState')
    echo "$(date): PowerState = $state"
    sleep 5
done
```

### 匯出感測器資料

```bash
curl -k -H "X-Auth-Token: $token" \
    https://${bmc}/redfish/v1/Chassis/chassis/Sensors \
    | jq '.Members[]."@odata.id"' \
    | while read uri; do
        curl -s -k -H "X-Auth-Token: $token" \
            https://${bmc}${uri//\"/} \
            | jq '{Name, Reading, ReadingUnits}'
    done
```

---

## 相關文件

- [RedfishOverview](RedfishOverview.md) - Redfish 概述
- [RedfishSystemService](RedfishSystemService.md) - 系統服務
- [Authentication](Authentication.md) - 認證機制
- [Troubleshooting](Troubleshooting.md) - 故障排除

---

*最後更新：2025-12-19*
