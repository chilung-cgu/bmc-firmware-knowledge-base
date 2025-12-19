# 測試方法

本文件說明 bmcweb 的測試方法和開發測試環境配置。

---

## 📋 目錄

1. [測試方法論](#測試方法論)
2. [開發環境設置](#開發環境設置)
3. [測試類型](#測試類型)
4. [Redfish Validator](#redfish-validator)

---

## 測試方法論

提交 bmcweb 變更前，開發者應執行相關測試：

- **功能測試** - 驗證變更行為正確
- **路徑覆蓋測試** - 測試所有受影響的程式碼路徑
- **Redfish 合規測試** - Redfish Validator

---

## 開發環境設置

### 使用 SDK 和 QEMU

1. **設置開發環境**

參考 [OpenBMC 開發環境文件](https://github.com/openbmc/docs/blob/master/development/dev-environment.md)

2. **Clone bmcweb**

```bash
git clone ssh://your-gerrit/openbmc/bmcweb/
```

3. **編譯**

```bash
meson setup builddir
ninja -C builddir
```

4. **縮小二進位檔案**

```bash
arm-openbmc-linux-gnueabi-strip bmcweb
```

> [!NOTE]
> 移除除錯符號可大幅減少傳輸時間。

---

## 部署到 QEMU

### 傳輸可執行檔

```bash
scp -P 2222 bmcweb root@127.0.0.1:/tmp/
```

### 停止服務

```bash
systemctl stop bmcweb
```

### 替換可執行檔

```bash
# 若檔案系統唯讀，需先掛載為可寫
mkdir -p /var/persist/usr
mkdir -p /var/persist/work/usr
mount -t overlay -o lowerdir=/usr,upperdir=/var/persist/usr,workdir=/var/persist/work/usr overlay /usr

# 替換
rm /usr/bin/bmcweb
ln -sf /tmp/bmcweb /usr/bin/bmcweb
```

### 測試

```bash
# 取得 Token
curl -c cjar -b cjar -k \
    -H "Content-Type: application/json" \
    -X POST https://127.0.0.1:2443/login \
    -d '{"data": ["root", "0penBmc"]}'

# 測試 API
curl -c cjar -b cjar -k \
    https://127.0.0.1:2443/redfish/v1/Systems/system
```

---

## 測試類型

### 1. Robot Framework 測試

```bash
# 執行 QEMU Robot 測試
run-qemu-robot-test.sh
```

這是 CI 自動執行的測試，確保通過可減少 PR 被回退的風險。

### 2. WebSocket 測試

啟用 `rest` 選項後，可測試 WebSocket：

```bash
python3 scripts/websocket_test.py --host 1.2.3.4:443 --ssl
```

### 3. 單元測試

編譯時啟用測試：

```bash
meson setup builddir -Dtests=enabled
ninja -C builddir test
```

---

## Redfish Validator

### 安裝

```bash
pip3 install redfish_service_validator
```

### 執行

```bash
rf_service_validator \
    --auth Session \
    -i https://bmc-ip:443 \
    -u root \
    -p 0penBmc
```

### 要求

| 要求 | 說明 |
|------|------|
| **無 ERROR** | Validator 不得報告 ERROR |
| **無 WARNING** | Validator 不應報告新的 WARNING |

### 報告

Validator 會生成 HTML 報告，包含：

- 已檢查的資源清單
- 各資源的合規狀態
- 錯誤和警告詳情

---

## 測試建議

### 核心變更

若修改 HTTP 核心程式碼：

- 測試 HTTP GET
- 測試 HTTP POST
- 測試 WebSocket

### Redfish 變更

若修改 Redfish 資源：

- 測試 GET 操作
- 測試 PATCH/POST/DELETE（如適用）
- 執行 Redfish Validator

### 認證變更

若修改認證相關程式碼：

- 測試各種認證方式
- 測試無效認證情況
- 測試權限邊界

---

## 提交訊息

測試結果應記錄在提交訊息中：

```
Subject: 修正 XXX 問題

詳細說明...

Tested:
- 執行 robot test on QEMU
- Redfish Validator 通過
- curl 手動測試 GET/POST

Signed-off-by: Name <email>
```

---

## 相關文件

- [Configuration](Configuration.md) - 編譯配置
- [CodeOrganization](CodeOrganization.md) - 原始碼結構
- [Troubleshooting](Troubleshooting.md) - 故障排除

---

*最後更新：2025-12-19*
