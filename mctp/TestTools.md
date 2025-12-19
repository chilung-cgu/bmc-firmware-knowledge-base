# 測試與除錯工具 (Test Tools)

CodeConstruct/mctp 專案包含多個測試和除錯工具，用於驗證 MCTP 功能和效能測試。

---

## 工具列表

| 工具 | 用途 | 安裝 |
|------|------|------|
| `mctp-req` | 發送 MCTP 請求 | 不預設安裝 |
| `mctp-echo` | MCTP echo 伺服器 | 不預設安裝 |
| `mctp-bench` | 效能測試工具 | 不預設安裝 |
| `mctp-client` | MCTP 客戶端工具 | 預設安裝 |

---

## mctp-client

簡單的 MCTP 客戶端工具，用於基本的 MCTP 通訊測試。

### 用途

- 發送簡單的 MCTP 訊息
- 測試 MCTP 連通性
- 除錯 MCTP 通訊問題

### 建構

```bash
# mctp-client 是預設建構並安裝的
meson setup obj
ninja -C obj
meson install -C obj
```

---

## mctp-req

發送 MCTP 請求訊息並接收回應。

### 用途

- 測試特定端點的回應
- 驗證控制協議訊息
- 基本功能測試

### 建構

```bash
# mctp-req 會建構但不預設安裝
ninja -C obj mctp-req
```

### 位置

```
obj/mctp-req
```

---

## mctp-echo

MCTP echo 伺服器，接收訊息並回傳相同內容。

### 用途

- 提供測試對端
- 驗證雙向通訊
- 延遲測試

### 建構

```bash
# mctp-echo 會建構但不預設安裝
ninja -C obj mctp-echo
```

### 運行

```bash
# 在一個終端啟動 echo 伺服器
./obj/mctp-echo

# 在另一個終端發送請求測試
./obj/mctp-req <eid>
```

---

## mctp-bench

MCTP 效能測試工具，用於測量訊息吞吐量和延遲。

### 用途

- 效能基準測試
- 延遲測量
- 吞吐量分析

### 建構

```bash
# mctp-bench 會建構但不預設安裝
ninja -C obj mctp-bench
```

### 功能特性

- 支援「request receive」模式
- 每 2 秒報告一次統計
- 使用 vendor-defined 訊息類型

### 使用模式

#### 發送模式

```bash
# 發送請求到指定 EID
./obj/mctp-bench send <eid>
```

#### 接收模式

```bash
# 接收模式（等待請求）
./obj/mctp-bench recv

# 發送命令啟動接收端的基準測試
./obj/mctp-bench recv eid <remote-eid>
```

---

## 測試框架

### pytest 測試套件

專案包含完整的 Python 測試套件：

```
tests/
├── test_mctp.py           # mctp 命令行工具測試
├── test_mctpd.py          # mctpd 基本測試
├── test_mctpd_endpoint.py # mctpd 端點測試
├── mctpd/                 # mctpd 測試框架
│   ├── __init__.py        # 可獨立運行的 mock 環境
│   └── ...
├── pytest.ini             # pytest 配置
└── requirements.txt       # Python 相依性
```

### 運行測試

```bash
# 透過 meson
meson setup obj
ninja -C obj test

# 或直接使用 pytest
cd obj
pytest ../tests
```

### 無 D-Bus session 運行

```bash
dbus-run-session env DBUS_STARTER_BUS_TYPE=user pytest ../tests
```

### 設定測試環境

```bash
# 建立 Python 虛擬環境
python3 -m venv venv
venv/bin/pip install -r tests/requirements.txt

# 使用虛擬環境運行測試
PATH=$PWD/venv/bin/:$PATH meson setup obj
ninja -C obj test
```

---

## mock 測試環境

mctpd 測試框架提供了一個 mock 環境，可以模擬 MCTP 設備行為：

### 獨立運行 mock 環境

```bash
cd obj
python3 ../tests/mctpd/__init__.py
```

### mock 環境功能

- 模擬 MCTP 控制協議回應
- 透過 Unix domain socket 封裝 Netlink 和 MCTP 訊息
- 支援自訂設備行為測試

---

## busctl 除錯

使用 `busctl` 命令與 mctpd D-Bus 服務互動：

### 列出物件樹

```bash
busctl tree au.com.codeconstruct.MCTP1
```

### 內省介面

```bash
# 查看介面物件
busctl introspect au.com.codeconstruct.MCTP1 \
    /au/com/codeconstruct/mctp1/interfaces/mctpi2c1

# 查看端點物件
busctl introspect au.com.codeconstruct.MCTP1 \
    /au/com/codeconstruct/mctp1/networks/1/endpoints/10
```

### 呼叫方法

```bash
# SetupEndpoint
busctl call au.com.codeconstruct.MCTP1 \
    /au/com/codeconstruct/mctp1/interfaces/mctpi2c1 \
    au.com.codeconstruct.MCTP.BusOwner1 \
    SetupEndpoint ay 1 0x1d

# Remove endpoint
busctl call au.com.codeconstruct.MCTP1 \
    /au/com/codeconstruct/mctp1/networks/1/endpoints/10 \
    au.com.codeconstruct.MCTP.Endpoint1 \
    Remove
```

### 監聽訊號

```bash
# 監聽所有 mctpd 訊號
busctl monitor au.com.codeconstruct.MCTP1
```

---

## 核心除錯

### 動態除錯

```bash
# 啟用 MCTP 核心除錯
echo 'module mctp +p' > /sys/kernel/debug/dynamic_debug/control
echo 'module mctp_i2c +p' > /sys/kernel/debug/dynamic_debug/control

# 查看核心日誌
dmesg -w | grep -i mctp
```

### 檢查網路狀態

```bash
# MCTP 介面
ip link show type mctp

# MCTP 統計（如果 procfs 支援）
cat /proc/net/mctp
```

### I2C 除錯

```bash
# 掃描 I2C 匯流排
i2cdetect -y 6

# 使用 i2c-tools 測試
i2cget -y 6 0x1d 0x00
```

---

## 常見測試場景

### 測試 MCTP 連通性

```bash
#!/bin/bash
# 基本連通性測試

# 1. 確認介面存在
mctp link show

# 2. 設定端點
busctl call au.com.codeconstruct.MCTP1 \
    /au/com/codeconstruct/mctp1/interfaces/mctpi2c1 \
    au.com.codeconstruct.MCTP.BusOwner1 \
    SetupEndpoint ay 1 0x1d

# 3. 確認端點建立
busctl tree au.com.codeconstruct.MCTP1

# 4. 查看端點屬性
busctl introspect au.com.codeconstruct.MCTP1 \
    /au/com/codeconstruct/mctp1/networks/1/endpoints/10
```

### 效能測試

```bash
#!/bin/bash
# MCTP 效能測試

# 啟動 echo 伺服器（在遠端端點或使用 mctp-echo）
./obj/mctp-echo &
ECHO_PID=$!

# 運行效能測試
./obj/mctp-bench send <eid>

# 清理
kill $ECHO_PID
```

### 壓力測試

```bash
#!/bin/bash
# 多端點壓力測試

for addr in 0x1d 0x1e 0x1f 0x20; do
    busctl call au.com.codeconstruct.MCTP1 \
        /au/com/codeconstruct/mctp1/interfaces/mctpi2c1 \
        au.com.codeconstruct.MCTP.BusOwner1 \
        SetupEndpoint ay 1 $addr &
done

wait
echo "All endpoints configured"
```

---

## 測試配置

### Address Sanitizer

測試預設啟用 Address Sanitizer：

```bash
# Meson 會自動配置 -fsanitize=address
meson setup obj
ninja -C obj test
```

### TAP 輸出

測試使用 TAP（Test Anything Protocol）輸出：

```bash
# pytest 會自動產生 TAP 格式輸出（給 meson test）
ninja -C obj test
```

---

## 相關文件

- [MctpdDaemon](MctpdDaemon.md) - mctpd 守護程式
- [DBusOverview](DBusOverview.md) - D-Bus 介面總覽
- [Troubleshooting](Troubleshooting.md) - 故障排除

---

[← 返回首頁](Home.md)
