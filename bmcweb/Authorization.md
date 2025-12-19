# 授權與權限控制

本文件說明 bmcweb 的授權機制和權限控制。

---

## 📋 目錄

1. [概述](#概述)
2. [Redfish 權限模型](#redfish-權限模型)
3. [路由級權限](#路由級權限)
4. [角色對應](#角色對應)

---

## 概述

bmcweb 使用 **Redfish PrivilegeRegistry** 進行授權：

- 每個路由定義所需權限
- 認證後使用者獲得角色
- 角色對應到權限集合
- 請求時比對權限

```
┌─────────────────────────────────────────────────────────────────┐
│                        授權流程                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  認證成功 → 取得使用者角色 → 角色對應權限 → 檢查路由權限 → 允許/拒絕  │
│                                                                  │
│  ┌─────────┐     ┌─────────┐     ┌─────────┐     ┌─────────┐    │
│  │  User   │ ──▶ │  Role   │ ──▶ │Privilege│ ──▶ │ Route   │    │
│  │  root   │     │  Admin  │     │ConfigMgr│     │ Check   │    │
│  └─────────┘     └─────────┘     │ConfigUsr│     └─────────┘    │
│                                  │ Login   │                     │
│                                  └─────────┘                     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Redfish 權限模型

### 權限類型

| 權限 | 說明 |
|------|------|
| `Login` | 登入權限（所有角色） |
| `ConfigureManager` | 配置 BMC 設定 |
| `ConfigureUsers` | 管理使用者帳戶 |
| `ConfigureSelf` | 修改自己的帳戶 |
| `ConfigureComponents` | 配置系統元件 |

### 預設角色

| 角色 | 權限 |
|------|------|
| **Administrator** | Login, ConfigureManager, ConfigureUsers, ConfigureSelf, ConfigureComponents |
| **Operator** | Login, ConfigureSelf, ConfigureComponents |
| **ReadOnly** | Login, ConfigureSelf |

---

## 路由級權限

### 權限定義

bmcweb 為每個路由定義所需權限：

```cpp
BMCWEB_ROUTE(app, "/redfish/v1/Systems/<str>/")
    .privileges(redfish::privileges::getComputerSystem)
    .methods(boost::beast::http::verb::get)(handleSystemGet);

BMCWEB_ROUTE(app, "/redfish/v1/Systems/<str>/")
    .privileges(redfish::privileges::patchComputerSystem)
    .methods(boost::beast::http::verb::patch)(handleSystemPatch);
```

### 權限定義檔案

權限定義於 `redfish-core/include/privileges.hpp`：

```cpp
namespace redfish::privileges {

// GET 操作 - 通常只需 Login
inline const std::array<Privileges, 1> getComputerSystem = {
    Privileges({"Login"})
};

// PATCH 操作 - 需要更高權限
inline const std::array<Privileges, 1> patchComputerSystem = {
    Privileges({"ConfigureComponents"})
};

// POST 操作 - 可能需要管理員權限
inline const std::array<Privileges, 1> postAccountService = {
    Privileges({"ConfigureUsers"})
};

}
```

### 非 Redfish 路由

非 Redfish 功能映射到最接近的 Redfish 權限：

| 功能 | 對應權限 |
|------|----------|
| Serial Console | ConfigureComponents |
| KVM | ConfigureComponents |
| D-Bus REST GET | Login |
| D-Bus REST PUT | ConfigureComponents |

---

## 角色對應

### 查詢角色

```bash
curl -k -H "X-Auth-Token: $token" \
    https://${bmc}/redfish/v1/AccountService/Roles/Administrator
```

**回應：**
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

### 角色層級

```
Administrator ─┬─ ConfigureManager
               ├─ ConfigureUsers  
               ├─ ConfigureComponents
               ├─ ConfigureSelf
               └─ Login

Operator ──────┬─ ConfigureComponents
               ├─ ConfigureSelf
               └─ Login

ReadOnly ──────┬─ ConfigureSelf
               └─ Login
```

---

## 權限檢查流程

### 請求處理

```
1. 請求進入
       │
2. 認證中介軟體
       │ 成功
       ▼
3. 取得使用者資訊
   - Username
   - Role
       │
       ▼
4. 路由匹配
       │
       ▼
5. 權限檢查
   route.privileges ⊆ user.privileges?
       │
    ┌──┴──┐
    │     │
   Yes    No
    │     │
    ▼     ▼
 繼續   返回 403
處理    Forbidden
```

### 權限不足回應

```json
{
    "error": {
        "@Message.ExtendedInfo": [
            {
                "Message": "There are insufficient privileges for the account.",
                "MessageId": "Base.1.13.0.InsufficientPrivilege"
            }
        ],
        "code": "Base.1.13.0.InsufficientPrivilege",
        "message": "There are insufficient privileges for the account."
    }
}
```

---

## 常見權限需求

### 讀取操作

| 資源 | 權限 |
|------|------|
| Service Root | 無（公開） |
| Systems, Chassis, Managers | Login |
| AccountService | Login |
| 特定帳戶 | Login（自己）或 ConfigureUsers |

### 修改操作

| 資源 | 權限 |
|------|------|
| 系統電源控制 | ConfigureComponents |
| 修改自己密碼 | ConfigureSelf |
| 修改其他帳戶 | ConfigureUsers |
| BMC 設定 | ConfigureManager |
| 韌體更新 | ConfigureComponents |

---

## 相關文件

- [Authentication](Authentication.md) - 認證機制
- [RedfishAccountService](RedfishAccountService.md) - 帳戶服務
- [SessionManagement](SessionManagement.md) - Session 管理

---

*最後更新：2025-12-19*
