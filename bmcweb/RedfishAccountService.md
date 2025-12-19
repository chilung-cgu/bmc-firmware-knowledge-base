# Redfish AccountService API

本文件說明 bmcweb 的帳戶服務 API，用於管理使用者帳戶和角色。

---

## 📋 目錄

1. [服務概述](#服務概述)
2. [API 端點](#api-端點)
3. [使用範例](#使用範例)
4. [LDAP 整合](#ldap-整合)
5. [MFA 支援](#mfa-支援)

---

## 服務概述

AccountService 提供使用者帳戶管理功能：

- **帳戶管理** - 建立、修改、刪除使用者
- **角色管理** - 定義權限角色
- **密碼原則** - 密碼複雜度、過期設定
- **LDAP 整合** - 外部身分驗證
- **MFA 支援** - 多因素認證

---

## API 端點

### AccountService 配置

```
GET /redfish/v1/AccountService
```

**回應範例：**
```json
{
    "@odata.id": "/redfish/v1/AccountService",
    "@odata.type": "#AccountService.v1_10_0.AccountService",
    "Id": "AccountService",
    "Name": "Account Service",
    "Accounts": {
        "@odata.id": "/redfish/v1/AccountService/Accounts"
    },
    "Roles": {
        "@odata.id": "/redfish/v1/AccountService/Roles"
    },
    "LDAP": {
        "AccountProviderType": "LDAPService",
        "ServiceEnabled": false
    },
    "MinPasswordLength": 8,
    "MaxPasswordLength": 20
}
```

### 帳戶管理

#### 列出所有帳戶
```bash
curl -k -H "X-Auth-Token: $token" \
    https://${bmc}/redfish/v1/AccountService/Accounts
```

#### 查詢特定帳戶
```bash
curl -k -H "X-Auth-Token: $token" \
    https://${bmc}/redfish/v1/AccountService/Accounts/root
```

**回應範例：**
```json
{
    "@odata.id": "/redfish/v1/AccountService/Accounts/root",
    "@odata.type": "#ManagerAccount.v1_9_0.ManagerAccount",
    "Id": "root",
    "Name": "User Account",
    "UserName": "root",
    "RoleId": "Administrator",
    "Enabled": true,
    "Locked": false,
    "Links": {
        "Role": {
            "@odata.id": "/redfish/v1/AccountService/Roles/Administrator"
        }
    }
}
```

#### 建立新帳戶
```bash
curl -k -H "X-Auth-Token: $token" \
    -H "Content-Type: application/json" \
    -X POST https://${bmc}/redfish/v1/AccountService/Accounts \
    -d '{
        "UserName": "newuser",
        "Password": "SecurePass123!",
        "RoleId": "Operator"
    }'
```

#### 修改帳戶
```bash
# 修改密碼
curl -k -H "X-Auth-Token: $token" \
    -H "Content-Type: application/json" \
    -X PATCH https://${bmc}/redfish/v1/AccountService/Accounts/root \
    -d '{"Password": "NewSecurePass456!"}'

# 停用帳戶
curl -k -H "X-Auth-Token: $token" \
    -H "Content-Type: application/json" \
    -X PATCH https://${bmc}/redfish/v1/AccountService/Accounts/user1 \
    -d '{"Enabled": false}'
```

#### 刪除帳戶
```bash
curl -k -H "X-Auth-Token: $token" \
    -X DELETE https://${bmc}/redfish/v1/AccountService/Accounts/user1
```

### 角色管理

#### 列出所有角色
```bash
curl -k -H "X-Auth-Token: $token" \
    https://${bmc}/redfish/v1/AccountService/Roles
```

#### 預設角色

| 角色 | 說明 | 權限 |
|------|------|------|
| `Administrator` | 完整管理權限 | 所有操作 |
| `Operator` | 操作員權限 | 讀取 + 操作 |
| `ReadOnly` | 唯讀權限 | 僅讀取 |

**角色詳情範例：**
```json
{
    "@odata.id": "/redfish/v1/AccountService/Roles/Administrator",
    "@odata.type": "#Role.v1_3_0.Role",
    "Id": "Administrator",
    "Name": "Administrator Role",
    "IsPredefined": true,
    "AssignedPrivileges": [
        "Login",
        "ConfigureManager",
        "ConfigureUsers",
        "ConfigureSelf",
        "ConfigureComponents"
    ]
}
```

---

## 使用範例

### 完整帳戶管理流程

```bash
# 1. 建立操作員帳戶
curl -k -H "X-Auth-Token: $token" \
    -H "Content-Type: application/json" \
    -X POST https://${bmc}/redfish/v1/AccountService/Accounts \
    -d '{
        "UserName": "operator1",
        "Password": "OpPass123!",
        "RoleId": "Operator"
    }'

# 2. 驗證帳戶建立
curl -k -H "X-Auth-Token: $token" \
    https://${bmc}/redfish/v1/AccountService/Accounts/operator1

# 3. 變更角色為 ReadOnly
curl -k -H "X-Auth-Token: $token" \
    -H "Content-Type: application/json" \
    -X PATCH https://${bmc}/redfish/v1/AccountService/Accounts/operator1 \
    -d '{"RoleId": "ReadOnly"}'

# 4. 刪除帳戶
curl -k -H "X-Auth-Token: $token" \
    -X DELETE https://${bmc}/redfish/v1/AccountService/Accounts/operator1
```

---

## LDAP 整合

### 配置 LDAP

```bash
curl -k -H "X-Auth-Token: $token" \
    -H "Content-Type: application/json" \
    -X PATCH https://${bmc}/redfish/v1/AccountService \
    -d '{
        "LDAP": {
            "ServiceEnabled": true,
            "ServiceAddresses": ["ldap://ldap.example.com"],
            "Authentication": {
                "AuthenticationType": "UsernameAndPassword",
                "Username": "cn=admin,dc=example,dc=com",
                "Password": "ldappassword"
            },
            "LDAPService": {
                "SearchSettings": {
                    "BaseDistinguishedNames": ["dc=example,dc=com"],
                    "UsernameAttribute": "uid",
                    "GroupsAttribute": "memberOf"
                }
            }
        }
    }'
```

### LDAP 角色對應

```bash
curl -k -H "X-Auth-Token: $token" \
    -H "Content-Type: application/json" \
    -X PATCH https://${bmc}/redfish/v1/AccountService \
    -d '{
        "LDAP": {
            "RemoteRoleMapping": [
                {
                    "RemoteGroup": "cn=admins,ou=groups,dc=example,dc=com",
                    "LocalRole": "Administrator"
                },
                {
                    "RemoteGroup": "cn=operators,ou=groups,dc=example,dc=com",
                    "LocalRole": "Operator"
                }
            ]
        }
    }'
```

### LDAP 憑證

```bash
# 列出 LDAP 憑證
curl -k -H "X-Auth-Token: $token" \
    https://${bmc}/redfish/v1/AccountService/LDAP/Certificates
```

---

## MFA 支援

### 客戶端憑證認證

bmcweb 支援 Mutual TLS 作為第二因素：

```
/redfish/v1/AccountService/MultiFactorAuth/ClientCertificate/Certificates
```

### 配置 MFA

```bash
# 啟用客戶端憑證認證
curl -k -H "X-Auth-Token: $token" \
    -H "Content-Type: application/json" \
    -X PATCH https://${bmc}/redfish/v1/AccountService \
    -d '{
        "MultiFactorAuth": {
            "ClientCertificate": {
                "Enabled": true
            }
        }
    }'
```

---

## D-Bus 對應

AccountService 透過以下 D-Bus 服務實作：

| Redfish 操作 | D-Bus 服務 |
|--------------|------------|
| 帳戶管理 | `xyz.openbmc_project.User.Manager` |
| 角色查詢 | 內建於 bmcweb |
| LDAP 配置 | `xyz.openbmc_project.Ldap.Config` |

---

## 相關文件

- [Authentication](Authentication.md) - 認證機制
- [Authorization](Authorization.md) - 授權機制
- [SessionManagement](SessionManagement.md) - Session 管理

---

*最後更新：2025-12-19*
