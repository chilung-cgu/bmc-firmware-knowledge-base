# 故障排除 (Troubleshooting)

本文提供 mctpd 和 MCTP 堆疊的常見問題診斷和解決方案。

---

## 診斷工具

### 檢查系統狀態

```bash
#!/bin/bash
# mctp-diag.sh - MCTP 診斷腳本

echo "=== MCTP Diagnostic ==="
echo ""

echo "1. Kernel MCTP support:"
if [ -e /proc/config.gz ]; then
    zcat /proc/config.gz | grep MCTP
else
    cat /boot/config-$(uname -r) 2>/dev/null | grep MCTP || echo "Cannot check kernel config"
fi
echo ""

echo "2. MCTP modules:"
lsmod | grep mctp || echo "No MCTP modules loaded"
echo ""

echo "3. MCTP interfaces:"
mctp link show 2>/dev/null || echo "mctp command not available"
echo ""

echo "4. mctpd status:"
pgrep -x mctpd > /dev/null && echo "Running" || echo "Not running"
echo ""

echo "5. D-Bus service:"
busctl list 2>/dev/null | grep au.com.codeconstruct.MCTP1 || echo "Not available"
echo ""

echo "6. Endpoints:"
busctl tree au.com.codeconstruct.MCTP1 2>/dev/null | grep endpoints || echo "No endpoints"
```

---

## 常見問題

### mctpd 無法啟動

#### 症狀
```
Failed to start mctpd.service
```

#### 診斷步驟

```bash
# 查看詳細錯誤
journalctl -u mctpd -n 50

# 手動執行測試
sudo /usr/sbin/mctpd -v
```

#### 常見原因和解決方案

| 原因 | 錯誤訊息 | 解決方案 |
|------|----------|----------|
| libsystemd 版本不足 | symbol lookup error | 升級 libsystemd ≥247 |
| 配置檔案錯誤 | TOML parse error | 檢查 /etc/mctpd.conf 語法 |
| D-Bus 衝突 | Failed to acquire bus name | 停止其他 mctpd 實例 |
| 權限問題 | Permission denied | 以 root 執行 |

---

### MCTP 介面不存在

#### 症狀
```
$ mctp link show
(no output)
```

#### 診斷步驟

```bash
# 檢查核心模組
lsmod | grep mctp

# 載入模組
sudo modprobe mctp
sudo modprobe mctp-i2c

# 檢查 dmesg
dmesg | grep -i mctp
```

#### 常見原因和解決方案

| 原因 | 解決方案 |
|------|----------|
| 核心不支援 MCTP | 升級到 5.15+ 或重新編譯核心 |
| 模組未載入 | `modprobe mctp mctp-i2c` |
| I2C 設備未配置 | 檢查 Device Tree |

---

### 無法發現端點

#### 症狀
```
$ busctl call ... SetupEndpoint ...
Error: ... timeout
```

#### 診斷步驟

```bash
# 1. 確認介面已啟用
mctp link show

# 2. 確認本機 EID 已分配
mctp address show

# 3. 檢查 I2C 連接
i2cdetect -y <bus-number>

# 4. 查看 mctpd 日誌
journalctl -u mctpd -f
```

#### 常見原因和解決方案

| 原因 | 解決方案 |
|------|----------|
| 介面未啟用 | `mctp link set <if> up` |
| 硬體未連接 | 檢查實體連線 |
| I2C 地址錯誤 | 使用 `i2cdetect` 確認 |
| 設備不支援 MCTP | 確認設備規格 |

---

### D-Bus 連線問題

#### 症狀
```
$ busctl call au.com.codeconstruct.MCTP1 ...
Failed to connect to bus
```

#### 診斷步驟

```bash
# 確認 mctpd 運行中
pgrep -x mctpd

# 確認服務已註冊
busctl list | grep MCTP

# 檢查 D-Bus daemon
systemctl status dbus
```

#### 解決方案

```bash
# 重啟 mctpd
sudo systemctl restart mctpd

# 或手動啟動
sudo mctpd -v
```

---

### 端點連接降級

#### 症狀
```
Connectivity: "Degraded"
```

#### 診斷步驟

```bash
# 查看端點狀態
busctl get-property au.com.codeconstruct.MCTP1 \
    /au/com/codeconstruct/mctp1/networks/1/endpoints/10 \
    au.com.codeconstruct.MCTP.Endpoint1 Connectivity

# 查看日誌
journalctl -u mctpd | grep -i "degraded\|recovery"
```

#### 常見原因和解決方案

| 原因 | 解決方案 |
|------|----------|
| 端點無回應 | 檢查設備電源和連線 |
| 逾時設定過短 | 增加 message_timeout_ms |
| 網路擁塞 | 減少並發請求 |

---

### EID 分配衝突

#### 症狀
```
Error: EID already assigned
```

#### 診斷步驟

```bash
# 查看已分配的 EID
busctl tree au.com.codeconstruct.MCTP1 | grep endpoints

# 查看路由表
mctp route show

# 查看鄰居表
mctp neigh show
```

#### 解決方案

```bash
# 移除衝突的端點
busctl call au.com.codeconstruct.MCTP1 \
    /au/com/codeconstruct/mctp1/networks/1/endpoints/<eid> \
    au.com.codeconstruct.MCTP.Endpoint1 \
    Remove

# 或使用 AssignEndpointStatic 指定不同 EID
```

---

### 橋接器無法分配 EID 池

#### 症狀
- 橋接器設定成功但無 Bridge1 介面
- 下游端點無法發現

#### 診斷步驟

```bash
# 查看動態 EID 範圍使用情況
mctp route show

# 查看日誌
journalctl -u mctpd | grep -i "pool\|allocate"
```

#### 常見原因和解決方案

| 原因 | 解決方案 |
|------|----------|
| EID 範圍不足 | 擴大 dynamic_eid_range |
| 無連續區塊 | 移除不需要的端點 |
| max_pool_size 太小 | 增加 max_pool_size |

---

## 核心除錯

### 啟用動態除錯

```bash
# 啟用 MCTP 核心除錯
echo 'module mctp +p' > /sys/kernel/debug/dynamic_debug/control
echo 'module mctp_i2c +p' > /sys/kernel/debug/dynamic_debug/control

# 查看日誌
dmesg -w | grep mctp
```

### 檢查 Netlink 訊息

```bash
# 使用 strace 追蹤 mctp 命令
strace -e trace=sendto,recvfrom mctp link show
```

---

## mctpd 除錯

### 詳細日誌

```bash
# 停止 systemd 服務
sudo systemctl stop mctpd

# 手動執行詳細模式
sudo mctpd -v
```

### D-Bus 監控

```bash
# 監控所有 mctpd 訊號和方法呼叫
dbus-monitor --system "sender='au.com.codeconstruct.MCTP1'"
```

---

## 效能問題

### 端點發現緩慢

#### 診斷

```bash
# 測量發現時間
time busctl call au.com.codeconstruct.MCTP1 ... SetupEndpoint ...
```

#### 解決方案

| 問題 | 解決方案 |
|------|----------|
| 逾時過長 | 減少 message_timeout_ms（但注意慢速設備） |
| 串行發現 | 並行發現多個端點 |
| 網路延遲 | 檢查硬體傳輸速度 |

### 高 CPU 使用

```bash
# 監控 mctpd CPU
top -p $(pgrep mctpd)

# 檢查是否有恢復迴圈
journalctl -u mctpd | grep -c recovery
```

---

## 錯誤代碼參考

### MCTP 控制協議完成碼

| 代碼 | 名稱 | 說明 |
|------|------|------|
| 0x00 | Success | 成功 |
| 0x01 | Error | 一般錯誤 |
| 0x02 | Error Invalid Data | 無效資料 |
| 0x03 | Error Invalid Length | 無效長度 |
| 0x04 | Error Not Ready | 未就緒 |
| 0x05 | Error Unsupported Cmd | 不支援的命令 |

### D-Bus 錯誤

| 錯誤 | 說明 |
|------|------|
| org.freedesktop.DBus.Error.ServiceUnknown | mctpd 未運行 |
| org.freedesktop.DBus.Error.Failed | 通用錯誤，查看訊息 |
| org.freedesktop.DBus.Error.InvalidArgs | 參數錯誤 |

---

## 報告問題

提交問題時請包含：

1. **系統資訊**
   ```bash
   uname -a
   cat /etc/os-release
   ```

2. **MCTP 配置**
   ```bash
   mctp link show
   mctp address show
   cat /etc/mctpd.conf
   ```

3. **日誌**
   ```bash
   journalctl -u mctpd --since "10 minutes ago"
   dmesg | grep -i mctp
   ```

4. **重現步驟**

---

## 相關文件

- [QuickStart](QuickStart.md) - 快速入門
- [Configuration](Configuration.md) - 配置指南
- [TestTools](TestTools.md) - 測試工具

---

[← 返回首頁](Home.md)
