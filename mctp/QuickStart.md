# 快速入門 (Quick Start)

本文提供 CodeConstruct/mctp 的快速入門指南。

---

## 前置需求

### 系統需求

- Linux 核心 5.15+ (支援 MCTP)
- libsystemd ≥ 247
- meson ≥ 0.59.0
- ninja

### 核心配置

確認核心已啟用 MCTP 支援：

```bash
# 檢查核心配置
zcat /proc/config.gz | grep MCTP

# 需要以下配置
# CONFIG_MCTP=y
# CONFIG_MCTP_TRANSPORT_I2C=y  (如果使用 I2C)
```

---

## 編譯安裝

### 取得原始碼

```bash
git clone https://github.com/CodeConstruct/mctp.git
cd mctp
```

### 編譯

```bash
# 配置
meson setup obj

# 編譯
ninja -C obj

# （可選）運行測試
ninja -C obj test
```

### 安裝

```bash
# 安裝到預設位置 (/usr/local)
sudo meson install -C obj

# 或指定 DESTDIR
sudo DESTDIR=/path/to/root meson install -C obj
```

### 最小化編譯（無測試）

```bash
# 跳過測試相依性
meson setup obj -Dtests=false
ninja -C obj
```

---

## 基本配置

### 1. 建立配置檔案

```bash
sudo mkdir -p /etc
sudo cp conf/mctpd.conf /etc/mctpd.conf
```

編輯 `/etc/mctpd.conf`：

```toml
mode = "bus-owner"

[mctp]
message_timeout_ms = 250

[bus-owner]
dynamic_eid_range = [8, 254]
max_pool_size = 15
```

### 2. 設定本地 MCTP 堆疊

```bash
# 啟用介面
sudo mctp link set mctpi2c1 up

# 設定網路 ID
sudo mctp link set mctpi2c1 network 1

# 設定 bus-owner 地址
sudo mctp link set mctpi2c1 bus-owner 0x10

# 分配本機 EID
sudo mctp address add 8 dev mctpi2c1
```

### 3. 啟動 mctpd

```bash
# 前景執行（除錯用）
sudo mctpd -v

# 或透過 systemd
sudo systemctl start mctpd
```

---

## 發現端點

### 使用 D-Bus 發現端點

```bash
# 發現 I2C 地址 0x1d 的端點
busctl call au.com.codeconstruct.MCTP1 \
    /au/com/codeconstruct/mctp1/interfaces/mctpi2c1 \
    au.com.codeconstruct.MCTP.BusOwner1 \
    SetupEndpoint ay 1 0x1d
```

**成功輸出**：
```
yisb 10 1 "/au/com/codeconstruct/mctp1/networks/1/endpoints/10" true
```

- EID: 10
- 網路 ID: 1
- 物件路徑: /au/com/codeconstruct/mctp1/networks/1/endpoints/10
- 新分配: true

### 查看已發現的端點

```bash
# 列出物件樹
busctl tree au.com.codeconstruct.MCTP1

# 查看端點屬性
busctl introspect au.com.codeconstruct.MCTP1 \
    /au/com/codeconstruct/mctp1/networks/1/endpoints/10
```

---

## 快速命令參考

### mctp 命令

```bash
# 查看介面
mctp link show

# 查看地址
mctp address show

# 查看路由
mctp route show

# 查看鄰居
mctp neigh show
```

### busctl 命令

```bash
# 列出 mctpd 物件
busctl tree au.com.codeconstruct.MCTP1

# 讀取端點 EID
busctl get-property au.com.codeconstruct.MCTP1 \
    /au/com/codeconstruct/mctp1/networks/1/endpoints/10 \
    xyz.openbmc_project.MCTP.Endpoint EID

# 讀取端點 UUID
busctl get-property au.com.codeconstruct.MCTP1 \
    /au/com/codeconstruct/mctp1/networks/1/endpoints/10 \
    xyz.openbmc_project.Common.UUID UUID
```

---

## 完整範例：設定並發現端點

```bash
#!/bin/bash
# setup-mctp.sh - MCTP 快速設定腳本

set -e

# 配置
INTERFACE="mctpi2c1"
NETWORK=1
LOCAL_EID=8
BUS_OWNER_ADDR="0x10"

echo "=== 設定 MCTP 堆疊 ==="

# 1. 等待介面存在
while [ ! -e /sys/class/net/$INTERFACE ]; do
    echo "Waiting for $INTERFACE..."
    sleep 0.5
done

# 2. 啟用介面
echo "Enabling $INTERFACE..."
mctp link set $INTERFACE up
mctp link set $INTERFACE network $NETWORK
mctp link set $INTERFACE bus-owner $BUS_OWNER_ADDR

# 3. 分配本機 EID
echo "Assigning local EID $LOCAL_EID..."
mctp address add $LOCAL_EID dev $INTERFACE 2>/dev/null || true

# 4. 啟動 mctpd（如果未運行）
if ! pgrep -x mctpd > /dev/null; then
    echo "Starting mctpd..."
    mctpd &
    sleep 1
fi

# 5. 顯示狀態
echo ""
echo "=== MCTP Status ==="
mctp link show
echo ""
mctp address show
echo ""

echo "=== Setup Complete ==="
echo ""
echo "To discover an endpoint:"
echo "  busctl call au.com.codeconstruct.MCTP1 \\"
echo "      /au/com/codeconstruct/mctp1/interfaces/$INTERFACE \\"
echo "      au.com.codeconstruct.MCTP.BusOwner1 \\"
echo "      SetupEndpoint ay 1 <I2C_ADDRESS>"
```

---

## 驗證設定

### 檢查 mctpd 狀態

```bash
# 確認 mctpd 正在運行
pgrep -x mctpd && echo "mctpd is running"

# 確認 D-Bus 服務可用
busctl list | grep MCTP && echo "D-Bus service available"
```

### 檢查 MCTP 堆疊

```bash
# 確認介面已啟用
mctp link show | grep "up"

# 確認本機 EID 已分配
mctp address show
```

### 測試端點通訊

```bash
# 發現端點後，檢查路由和鄰居
mctp route show
mctp neigh show
```

---

## 常見問題

### mctpd 無法啟動

```bash
# 檢查 libsystemd 版本
pkg-config --modversion libsystemd

# 檢查配置檔案
cat /etc/mctpd.conf
```

### 無法發現端點

```bash
# 確認介面已啟用
mctp link show

# 確認硬體連接
i2cdetect -y 6  # 適用於 I2C

# 查看 mctpd 日誌
journalctl -u mctpd -f
```

### D-Bus 連線問題

```bash
# 確認服務可用
busctl list | grep au.com.codeconstruct

# 測試方法呼叫
busctl call au.com.codeconstruct.MCTP1 /au/com/codeconstruct/mctp1 \
    org.freedesktop.DBus.ObjectManager GetManagedObjects
```

---

## 下一步

- [Architecture](Architecture.md) - 了解系統架構
- [MctpdDaemon](MctpdDaemon.md) - 深入了解 mctpd
- [Configuration](Configuration.md) - 詳細配置選項
- [EndpointDiscovery](EndpointDiscovery.md) - 端點發現流程

---

[← 返回首頁](Home.md)
