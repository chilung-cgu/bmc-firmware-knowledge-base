# Session 管理

本文件說明 bmcweb 的 Session 管理機制。

---

## 📋 目錄

1. [概述](#概述)
2. [Session 生命週期](#session-生命週期)
3. [Session API](#session-api)
4. [持久化](#持久化)

---

## 概述

bmcweb 支援 Redfish 標準的 Session 管理：

- **Session 建立** - 透過 SessionService
- **Token 認證** - X-Auth-Token Header
- **Session 逾時** - 可配置的逾時時間
- **持久化** - 重啟後恢復 Session

---

## Session 生命週期

```
┌─────────────────────────────────────────────────────────────────┐
│                     Session 生命週期                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. 登入          2. 使用中           3. 逾時/登出                │
│                                                                  │
│  ┌─────────┐     ┌─────────┐         ┌─────────┐                │
│  │  POST   │ ──▶ │ Active  │ ◀─────▶ │ Expired │                │
│  │ /login  │     │ Session │ ──────▶ │ Session │                │
│  └─────────┘     └─────────┘         └────┬────┘                │
│       │               │                   │                      │
│       ▼               ▼                   ▼                      │
│  建立 Token      請求時更新            自動清理                   │
│  儲存 Session    最後活動時間          或刪除 Session              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Session API

### 建立 Session

```bash
curl -k -H "Content-Type: application/json" \
    -X POST https://${bmc}/redfish/v1/SessionService/Sessions \
    -d '{"UserName": "root", "Password": "0penBmc"}' \
    -D headers.txt
```

**回應：**
```json
{
    "@odata.id": "/redfish/v1/SessionService/Sessions/1",
    "@odata.type": "#Session.v1_5_0.Session",
    "Id": "1",
    "Name": "User Session",
    "UserName": "root"
}
```

**從 headers.txt 取得 Token：**
```bash
X-Auth-Token: <session-token>
Location: /redfish/v1/SessionService/Sessions/1
```

### 列出 Sessions

```bash
curl -k -H "X-Auth-Token: $token" \
    https://${bmc}/redfish/v1/SessionService/Sessions
```

### 查詢特定 Session

```bash
curl -k -H "X-Auth-Token: $token" \
    https://${bmc}/redfish/v1/SessionService/Sessions/1
```

### 刪除 Session (登出)

```bash
curl -k -H "X-Auth-Token: $token" \
    -X DELETE https://${bmc}/redfish/v1/SessionService/Sessions/1
```

---

## Session 配置

### SessionService 設定

```bash
curl -k -H "X-Auth-Token: $token" \
    https://${bmc}/redfish/v1/SessionService
```

**回應：**
```json
{
    "@odata.id": "/redfish/v1/SessionService",
    "@odata.type": "#SessionService.v1_1_8.SessionService",
    "Id": "SessionService",
    "Name": "Session Service",
    "ServiceEnabled": true,
    "SessionTimeout": 1800,
    "Sessions": {
        "@odata.id": "/redfish/v1/SessionService/Sessions"
    }
}
```

### 修改 Session 逾時

```bash
curl -k -H "X-Auth-Token: $token" \
    -H "Content-Type: application/json" \
    -X PATCH https://${bmc}/redfish/v1/SessionService \
    -d '{"SessionTimeout": 3600}'
```

**逾時範圍：**
- 最小：30 秒
- 最大：86400 秒（24 小時）
- 預設：1800 秒（30 分鐘）

---

## Cookie 認證

除了 X-Auth-Token，bmcweb 也支援 Cookie 認證：

### 使用 Login 端點

```bash
# 登入並儲存 Cookie
curl -k -c cookies.txt \
    -H "Content-Type: application/json" \
    -X POST https://${bmc}/login \
    -d '{"username": "root", "password": "0penBmc"}'

# 使用 Cookie
curl -k -b cookies.txt https://${bmc}/redfish/v1/Systems
```

### Cookie 屬性

| 屬性 | 值 |
|------|-----|
| Name | SESSION |
| Secure | True |
| HttpOnly | True |
| SameSite | Strict |

---

## 持久化

### 儲存位置

Session 資料儲存於：

```
/var/lib/bmcweb/sessions.json
```

### 持久化行為

- BMC 重啟後 Session 仍有效
- 逾期 Session 在啟動時清理
- Token 保持不變

### 持久化資料結構

```json
{
    "sessions": [
        {
            "id": "1",
            "username": "root",
            "token": "<session-token>",
            "created": "2025-12-19T05:00:00+00:00",
            "lastActivity": "2025-12-19T05:30:00+00:00"
        }
    ]
}
```

---

## Session 限制

### 最大 Session 數

bmcweb 限制同時存在的 Session 數量：

- 預設最大值取決於編譯配置
- 達到限制時新登入會失敗

### Session 清理

- **自動清理** - 定期檢查並移除逾期 Session
- **手動登出** - DELETE Session
- **服務重啟** - 逾期 Session 被清理

---

## 最佳實踐

> [!TIP]
> **建議：**
> - 使用完畢後主動登出
> - 設定適當的 Session 逾時
> - 避免長期保持 Session

> [!IMPORTANT]
> **安全考量：**
> - Token 應妥善保管
> - 不要在 URL 中傳遞 Token
> - 使用 HTTPS

---

## 相關文件

- [Authentication](Authentication.md) - 認證機制
- [Authorization](Authorization.md) - 授權機制
- [RedfishAccountService](RedfishAccountService.md) - 帳戶服務

---

*最後更新：2025-12-19*
