# 認證機制

本文件說明 bmcweb 支援的認證機制。

---

## 📋 目錄

1. [概述](#概述)
2. [認證方式](#認證方式)
3. [PAM 整合](#pam-整合)
4. [配置選項](#配置選項)

---

## 概述

bmcweb 支援多種認證協議，所有使用者名稱/密碼認證均透過 **PAM (Pluggable Authentication Modules)** 處理。

```
┌─────────────────────────────────────────────────────────────────┐
│                        認證流程                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  HTTP Request                                                    │
│       │                                                          │
│       ▼                                                          │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │                  認證中介軟體                                 ││
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐            ││
│  │  │  Basic  │ │ Session │ │  mTLS   │ │ XToken  │            ││
│  │  │  Auth   │ │  Auth   │ │  Auth   │ │  Auth   │            ││
│  │  └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘            ││
│  │       │           │           │           │                  ││
│  │       ▼           ▼           ▼           ▼                  ││
│  │  ┌────────────────────────────────────────────────────────┐ ││
│  │  │                      PAM                                │ ││
│  │  └────────────────────────────────────────────────────────┘ ││
│  └─────────────────────────────────────────────────────────────┘│
│                             │                                    │
│                      認證成功/失敗                                │
│                             ▼                                    │
│                    繼續請求處理或返回 401                         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 認證方式

### 1. Basic Authentication (RFC 7617)

HTTP 基本認證，使用者名稱和密碼以 Base64 編碼。

**請求範例：**
```bash
curl -k -u root:0penBmc https://${bmc}/redfish/v1/Systems
```

**HTTP Header：**
```http
Authorization: Basic cm9vdDowcGVuQm1j
```

> [!WARNING]
> Basic Auth 密碼為 Base64 編碼（非加密），必須搭配 HTTPS 使用。

### 2. Session Authentication

透過登入端點建立 Session，使用 Cookie 維持狀態。

**建立 Session：**
```bash
# Redfish Session
curl -k -H "Content-Type: application/json" \
    -X POST https://${bmc}/redfish/v1/SessionService/Sessions \
    -d '{"UserName": "root", "Password": "0penBmc"}' \
    -D headers.txt

# 從 headers.txt 取得 X-Auth-Token
export token=$(grep "X-Auth-Token" headers.txt | awk '{print $2}' | tr -d '\r')

# 使用 Token
curl -k -H "X-Auth-Token: $token" https://${bmc}/redfish/v1/Systems
```

**替代方式 - Login 端點：**
```bash
curl -k -c cookies.txt -H "Content-Type: application/json" \
    -X POST https://${bmc}/login \
    -d '{"username": "root", "password": "0penBmc"}'

# 使用 Cookie
curl -k -b cookies.txt https://${bmc}/redfish/v1/Systems
```

### 3. X-Auth-Token

Redfish 標準的 Token 認證 (DSP0266)。

**使用方式：**
```bash
curl -k -H "X-Auth-Token: $token" https://${bmc}/redfish/v1/Systems
```

### 4. Mutual TLS (mTLS)

使用客戶端憑證進行認證。

**配置客戶端憑證：**

1. 上傳信任的 CA 憑證
```bash
curl -k -H "X-Auth-Token: $token" \
    -H "Content-Type: application/json" \
    -X POST https://${bmc}/redfish/v1/Managers/bmc/Truststore/Certificates \
    -d '{
        "CertificateType": "PEM",
        "CertificateString": "-----BEGIN CERTIFICATE-----\n...\n-----END CERTIFICATE-----"
    }'
```

2. 使用客戶端憑證連線
```bash
curl -k --cert client.pem --key client.key https://${bmc}/redfish/v1/Systems
```

**憑證-使用者映射：**

mTLS 憑證的 CN (Common Name) 對應到 BMC 使用者名稱。

---

## PAM 整合

所有使用者名稱/密碼認證透過 PAM 處理：

### PAM 服務檔案

bmcweb 使用 `/etc/pam.d/bmcweb` 配置：

```
# /etc/pam.d/bmcweb
auth    required    pam_unix.so
account required    pam_unix.so
```

### 支援的 PAM 模組

| 模組 | 用途 |
|------|------|
| `pam_unix` | 本地帳戶 |
| `pam_ldap` | LDAP 認證 |
| `pam_radius` | RADIUS 認證 |

---

## 配置選項

### 編譯時選項

| 選項 | 說明 | 預設 |
|------|------|------|
| `basic-auth` | 啟用 Basic Auth | enabled |
| `session-auth` | 啟用 Session Auth | enabled |
| `xtoken-auth` | 啟用 XToken Auth | enabled |
| `mutual-tls-auth` | 啟用 mTLS Auth | enabled |
| `cookie-auth` | 啟用 Cookie Auth | enabled |

**Meson 配置：**
```bash
meson setup builddir \
    -Dbasic-auth=enabled \
    -Dsession-auth=enabled \
    -Dmutual-tls-auth=enabled
```

### 執行時配置

可透過 Redfish API 調整認證設定：

```bash
# 查詢認證配置
curl -k -H "X-Auth-Token: $token" \
    https://${bmc}/redfish/v1/AccountService

# 啟用/停用特定認證方式
curl -k -H "X-Auth-Token: $token" \
    -H "Content-Type: application/json" \
    -X PATCH https://${bmc}/redfish/v1/AccountService \
    -d '{"Oem": {"OpenBMC": {"AuthMethods": {"BasicAuth": true}}}}'
```

---

## 認證優先順序

bmcweb 按以下順序嘗試認證：

1. **mTLS** - 如果提供客戶端憑證
2. **X-Auth-Token** - 檢查 Header
3. **Cookie/Session** - 檢查 Session Cookie
4. **Basic Auth** - 檢查 Authorization Header

第一個成功的認證方式將被使用。

---

## 安全建議

> [!IMPORTANT]
> **生產環境建議：**
> - 使用 HTTPS（強制）
> - 優先使用 Session 或 mTLS
> - 設定適當的 Session 逾時
> - 啟用帳戶鎖定策略

> [!CAUTION]
> 避免在不安全的網路使用 Basic Auth，即使有 HTTPS 保護，Token 也有被攔截的風險。

---

## 相關文件

- [Authorization](Authorization.md) - 授權機制
- [SessionManagement](SessionManagement.md) - Session 管理
- [RedfishAccountService](RedfishAccountService.md) - 帳戶服務

---

*最後更新：2025-12-19*
