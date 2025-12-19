# 故障排除

本文件提供 bmcweb 常見問題的診斷和解決方案。

---

## 📋 目錄

1. [連線問題](#連線問題)
2. [認證問題](#認證問題)
3. [Redfish API 問題](#redfish-api-問題)
4. [WebSocket 問題](#websocket-問題)
5. [日誌分析](#日誌分析)

---

## 連線問題

### 無法連線到 BMC

**症狀：** `curl: (7) Failed to connect`

**診斷步驟：**

1. 確認網路連通性
```bash
ping ${bmc}
```

2. 確認 HTTPS 端口開啟
```bash
nmap -p 443 ${bmc}
```

3. 檢查 bmcweb 服務狀態
```bash
# 在 BMC 上
systemctl status bmcweb
```

**解決方案：**
- 確認 BMC IP 正確
- 重啟 bmcweb 服務：`systemctl restart bmcweb`
- 檢查防火牆設定

### SSL 憑證錯誤

**症狀：** `curl: (60) SSL certificate problem`

**解決方案：**

```bash
# 暫時忽略憑證驗證（僅供測試）
curl -k https://${bmc}/redfish/v1

# 正確做法：取得並信任 BMC 憑證
```

---

## 認證問題

### 401 Unauthorized

**症狀：** 所有請求返回 401

**診斷步驟：**

1. 確認認證資訊正確
```bash
curl -k -u root:0penBmc https://${bmc}/redfish/v1/Systems
```

2. 確認 Token 有效
```bash
echo $token
```

3. 測試重新登入
```bash
export token=$(curl -k -s -H "Content-Type: application/json" \
    -X POST https://${bmc}/login \
    -d '{"username":"root","password":"0penBmc"}' \
    | jq -r '.token')
```

**常見原因：**
- Token 過期
- 密碼錯誤
- 帳戶被鎖定

### 403 Forbidden

**症狀：** 認證成功但操作被拒絕

**診斷步驟：**

1. 確認使用者角色
```bash
curl -k -H "X-Auth-Token: $token" \
    https://${bmc}/redfish/v1/AccountService/Accounts/root
```

2. 檢查操作所需權限

**解決方案：**
- 使用有足夠權限的帳戶
- 確認操作對應的權限需求

### Session 過期

**症狀：** 使用一段時間後突然 401

**解決方案：**

```bash
# 查詢 Session 逾時設定
curl -k -H "X-Auth-Token: $token" \
    https://${bmc}/redfish/v1/SessionService

# 增加逾時時間
curl -k -H "X-Auth-Token: $token" \
    -H "Content-Type: application/json" \
    -X PATCH https://${bmc}/redfish/v1/SessionService \
    -d '{"SessionTimeout": 3600}'
```

---

## Redfish API 問題

### 404 Not Found

**症狀：** 資源不存在

**診斷步驟：**

1. 確認 URI 正確
```bash
# 從 Service Root 開始導航
curl -k -H "X-Auth-Token: $token" https://${bmc}/redfish/v1
```

2. 確認資源存在
```bash
# 查詢集合
curl -k -H "X-Auth-Token: $token" https://${bmc}/redfish/v1/Systems
```

**常見原因：**
- URI 拼寫錯誤
- 資源 ID 不正確
- 功能未啟用

### 500 Internal Server Error

**症狀：** 伺服器內部錯誤

**診斷步驟：**

1. 查看 bmcweb 日誌
```bash
journalctl -u bmcweb -f
```

2. 檢查 D-Bus 服務狀態
```bash
busctl list | grep openbmc
```

**常見原因：**
- D-Bus 服務故障
- 內部異常未處理

### 資料不完整

**症狀：** 預期的屬性缺失

**診斷步驟：**

1. 檢查對應的 D-Bus 物件
```bash
busctl introspect xyz.openbmc_project.State.Host \
    /xyz/openbmc_project/state/host0
```

2. 確認相關服務正在運行

---

## WebSocket 問題

### 無法建立 WebSocket 連線

**症狀：** WebSocket 連線失敗

**診斷步驟：**

1. 確認功能已啟用
```bash
# 檢查 rest 選項是否啟用
# Serial Console / KVM 是否編譯
```

2. 確認認證
```bash
websocat -k --header "X-Auth-Token:$token" wss://${bmc}/console0
```

**常見原因：**
- 功能未編譯
- 認證失敗
- 端點不正確

### KVM 黑畫面

**診斷步驟：**

1. 確認主機已開機
2. 檢查 video 設備
```bash
ls -la /dev/video0
```

3. 查看 bmcweb 日誌
```bash
journalctl -u bmcweb | grep -i kvm
```

---

## 日誌分析

### 查看 bmcweb 日誌

```bash
# 即時查看
journalctl -u bmcweb -f

# 最近的日誌
journalctl -u bmcweb -n 100

# 特定時間範圍
journalctl -u bmcweb --since "2025-12-19 05:00:00"
```

### 常見日誌訊息

| 訊息 | 說明 |
|------|------|
| `Starting HTTPS server` | 伺服器啟動 |
| `Authentication failed` | 認證失敗 |
| `D-Bus call failed` | D-Bus 通訊失敗 |
| `SSL handshake failed` | TLS 握手失敗 |

### 增加日誌詳細度

編譯時啟用除錯：

```bash
meson setup builddir --buildtype=debug
```

---

## 常見錯誤代碼

| HTTP 狀態碼 | 含義 | 常見原因 |
|-------------|------|----------|
| 400 | Bad Request | 請求格式錯誤 |
| 401 | Unauthorized | 認證失敗 |
| 403 | Forbidden | 權限不足 |
| 404 | Not Found | 資源不存在 |
| 405 | Method Not Allowed | HTTP 方法錯誤 |
| 500 | Internal Server Error | 伺服器內部錯誤 |
| 502 | Bad Gateway | 聚合時衛星 BMC 不可達 |
| 503 | Service Unavailable | 服務暫時不可用 |

---

## 實用除錯命令

### D-Bus 檢查

```bash
# 列出所有服務
busctl list

# 查看物件樹
busctl tree xyz.openbmc_project.State.Host

# 讀取屬性
busctl get-property xyz.openbmc_project.State.Host \
    /xyz/openbmc_project/state/host0 \
    xyz.openbmc_project.State.Host CurrentHostState
```

### 網路診斷

```bash
# 檢查監聽端口
netstat -tlnp | grep bmcweb

# 測試 HTTPS
openssl s_client -connect ${bmc}:443
```

---

## 相關文件

- [Architecture](Architecture.md) - 架構概述
- [Authentication](Authentication.md) - 認證機制
- [Testing](Testing.md) - 測試方法

---

*最後更新：2025-12-19*
