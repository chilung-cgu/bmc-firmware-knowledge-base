# Redfish Manager Service API

本文件說明 bmcweb 的 Manager 服務 API，用於管理 BMC 本身。

---

## 📋 目錄

1. [服務概述](#服務概述)
2. [Manager 資源](#manager-資源)
3. [網路配置](#網路配置)
4. [日誌服務](#日誌服務)
5. [BMC 操作](#bmc-操作)

---

## 服務概述

Manager 服務提供 BMC 管理功能：

- **BMC 資訊** - 韌體版本、狀態
- **網路配置** - Ethernet、DNS、NTP
- **日誌服務** - BMC 日誌
- **重啟/重設** - BMC 操作

---

## Manager 資源

### 列出所有 Manager

```bash
curl -k -H "X-Auth-Token: $token" \
    https://${bmc}/redfish/v1/Managers
```

### 查詢 BMC 詳情

```bash
curl -k -H "X-Auth-Token: $token" \
    https://${bmc}/redfish/v1/Managers/bmc
```

**回應範例：**
```json
{
    "@odata.id": "/redfish/v1/Managers/bmc",
    "@odata.type": "#Manager.v1_14_0.Manager",
    "Id": "bmc",
    "Name": "OpenBMC Manager",
    "ManagerType": "BMC",
    "FirmwareVersion": "2.12.0",
    "Status": {
        "State": "Enabled",
        "Health": "OK"
    },
    "DateTime": "2025-12-19T05:00:00+00:00",
    "DateTimeLocalOffset": "+00:00",
    "EthernetInterfaces": {
        "@odata.id": "/redfish/v1/Managers/bmc/EthernetInterfaces"
    },
    "NetworkProtocol": {
        "@odata.id": "/redfish/v1/Managers/bmc/NetworkProtocol"
    },
    "LogServices": {
        "@odata.id": "/redfish/v1/Managers/bmc/LogServices"
    },
    "Actions": {
        "#Manager.Reset": {
            "target": "/redfish/v1/Managers/bmc/Actions/Manager.Reset",
            "ResetType@Redfish.AllowableValues": [
                "GracefulRestart",
                "ForceRestart"
            ]
        },
        "#Manager.ResetToDefaults": {
            "target": "/redfish/v1/Managers/bmc/Actions/Manager.ResetToDefaults"
        }
    }
}
```

---

## 網路配置

### 列出網路介面

```bash
curl -k -H "X-Auth-Token: $token" \
    https://${bmc}/redfish/v1/Managers/bmc/EthernetInterfaces
```

### 查詢網路介面詳情

```bash
curl -k -H "X-Auth-Token: $token" \
    https://${bmc}/redfish/v1/Managers/bmc/EthernetInterfaces/eth0
```

**回應範例：**
```json
{
    "@odata.id": "/redfish/v1/Managers/bmc/EthernetInterfaces/eth0",
    "@odata.type": "#EthernetInterface.v1_9_0.EthernetInterface",
    "Id": "eth0",
    "Name": "Manager Ethernet Interface",
    "MACAddress": "00:11:22:33:44:55",
    "IPv4Addresses": [
        {
            "Address": "192.168.1.100",
            "SubnetMask": "255.255.255.0",
            "Gateway": "192.168.1.1",
            "AddressOrigin": "Static"
        }
    ],
    "IPv6Addresses": [],
    "HostName": "openbmc",
    "FQDN": "openbmc.local",
    "NameServers": ["8.8.8.8", "8.8.4.4"]
}
```

### 修改 IP 配置

```bash
# 設定靜態 IP
curl -k -H "X-Auth-Token: $token" \
    -H "Content-Type: application/json" \
    -X PATCH https://${bmc}/redfish/v1/Managers/bmc/EthernetInterfaces/eth0 \
    -d '{
        "IPv4StaticAddresses": [
            {
                "Address": "192.168.1.200",
                "SubnetMask": "255.255.255.0",
                "Gateway": "192.168.1.1"
            }
        ]
    }'
```

### 網路協議配置

```bash
curl -k -H "X-Auth-Token: $token" \
    https://${bmc}/redfish/v1/Managers/bmc/NetworkProtocol
```

**回應範例：**
```json
{
    "@odata.id": "/redfish/v1/Managers/bmc/NetworkProtocol",
    "@odata.type": "#ManagerNetworkProtocol.v1_8_0.ManagerNetworkProtocol",
    "Id": "NetworkProtocol",
    "Name": "Manager Network Protocol",
    "HTTPS": {
        "ProtocolEnabled": true,
        "Port": 443,
        "Certificates": {
            "@odata.id": "/redfish/v1/Managers/bmc/NetworkProtocol/HTTPS/Certificates"
        }
    },
    "SSH": {
        "ProtocolEnabled": true,
        "Port": 22
    },
    "IPMI": {
        "ProtocolEnabled": true,
        "Port": 623
    },
    "NTP": {
        "ProtocolEnabled": true,
        "NTPServers": ["time.nist.gov"]
    }
}
```

### 配置 NTP

```bash
# 設定 NTP 伺服器
curl -k -H "X-Auth-Token: $token" \
    -H "Content-Type: application/json" \
    -X PATCH https://${bmc}/redfish/v1/Managers/bmc/NetworkProtocol \
    -d '{"NTP": {"NTPServers": ["time.nist.gov", "pool.ntp.org"]}}'

# 啟用 NTP
curl -k -H "X-Auth-Token: $token" \
    -H "Content-Type: application/json" \
    -X PATCH https://${bmc}/redfish/v1/Managers/bmc/NetworkProtocol \
    -d '{"NTP": {"ProtocolEnabled": true}}'
```

### 停用 IPMI

```bash
curl -k -H "X-Auth-Token: $token" \
    -H "Content-Type: application/json" \
    -X PATCH https://${bmc}/redfish/v1/Managers/bmc/NetworkProtocol \
    -d '{"IPMI": {"ProtocolEnabled": false}}'
```

---

## 日誌服務

### 列出日誌服務

```bash
curl -k -H "X-Auth-Token: $token" \
    https://${bmc}/redfish/v1/Managers/bmc/LogServices
```

### 查詢 BMC 日誌

```bash
curl -k -H "X-Auth-Token: $token" \
    https://${bmc}/redfish/v1/Managers/bmc/LogServices/RedfishLog/Entries
```

### Manager 診斷資料

```bash
curl -k -H "X-Auth-Token: $token" \
    https://${bmc}/redfish/v1/Managers/bmc/ManagerDiagnosticData
```

**回應範例：**
```json
{
    "@odata.id": "/redfish/v1/Managers/bmc/ManagerDiagnosticData",
    "@odata.type": "#ManagerDiagnosticData.v1_2_0.ManagerDiagnosticData",
    "FreeStorageSpaceKiB": 1048576,
    "MemoryStatistics": {
        "TotalBytes": 536870912,
        "UsedBytes": 268435456,
        "FreeBytes": 268435456
    },
    "ProcessorStatistics": {
        "KernelPercent": 5,
        "UserPercent": 10
    }
}
```

---

## BMC 操作

### 重啟 BMC

```bash
curl -k -H "X-Auth-Token: $token" \
    -H "Content-Type: application/json" \
    -X POST https://${bmc}/redfish/v1/Managers/bmc/Actions/Manager.Reset \
    -d '{"ResetType": "GracefulRestart"}'
```

### 恢復原廠設定

> [!CAUTION]
> 此操作將清除所有配置！

```bash
curl -k -H "X-Auth-Token: $token" \
    -H "Content-Type: application/json" \
    -X POST https://${bmc}/redfish/v1/Managers/bmc/Actions/Manager.ResetToDefaults \
    -d '{"ResetType": "ResetAll"}'
```

**ResetToDefaults 類型：**
| 類型 | 說明 |
|------|------|
| `ResetAll` | 完全重設所有設定 |
| `PreserveNetworkAndUsers` | 保留網路和使用者設定 |

---

## HTTPS 憑證管理

### 列出 HTTPS 憑證

```bash
curl -k -H "X-Auth-Token: $token" \
    https://${bmc}/redfish/v1/Managers/bmc/NetworkProtocol/HTTPS/Certificates
```

### 更換憑證

```bash
curl -k -H "X-Auth-Token: $token" \
    -H "Content-Type: application/json" \
    -X POST https://${bmc}/redfish/v1/CertificateService/Actions/CertificateService.ReplaceCertificate \
    -d '{
        "CertificateUri": "/redfish/v1/Managers/bmc/NetworkProtocol/HTTPS/Certificates/1",
        "CertificateType": "PEM",
        "CertificateString": "-----BEGIN CERTIFICATE-----\n...\n-----END CERTIFICATE-----"
    }'
```

---

## D-Bus 對應

| Redfish 資源 | D-Bus 服務 |
|--------------|------------|
| Manager 狀態 | `xyz.openbmc_project.State.BMC` |
| 網路配置 | `xyz.openbmc_project.Network` |
| 時間/NTP | `xyz.openbmc_project.Time.*` |
| 日誌 | `xyz.openbmc_project.Logging` |

---

## 相關文件

- [RedfishSystemService](RedfishSystemService.md) - 系統服務
- [Authentication](Authentication.md) - 認證機制
- [Configuration](Configuration.md) - 配置選項

---

*最後更新：2025-12-19*
