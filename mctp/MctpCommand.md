# mctp 命令行工具 (mctp Command)

`mctp` 是一個輕量級的命令行工具，用於查詢和管理 Linux 核心 MCTP 堆疊的狀態。其設計類似於 `iproute2` 套件的 `ip` 命令。

---

## 概述

```bash
$ mctp help
mctp link
mctp link show [ifname]
mctp link set <ifname> [up|down] [mtu <mtu>] [network <net>] [bus-owner <physaddr>]

mctp address
mctp address show [IFNAME]
mctp address add <eid> dev <IFNAME>
mctp address del <eid> dev <IFNAME>

mctp route
mctp route show [net <network>]
mctp route add <eid>[-<eid>] via <dev> [mtu <mtu>]
mctp route add <eid>[-<eid>] gw <eid> [net <net>] [mtu <mtu>]
mctp route del <eid>[-<eid>] via <dev>
mctp route del <eid>[-<eid>] gw <eid> [net <net>]

mctp neigh
mctp neigh show [dev <network>]
mctp neigh add <eid> dev <device> lladdr <physaddr>
mctp neigh del <eid> dev <device>
```

---

## mctp link

管理 MCTP 網路介面。

### mctp link show

顯示 MCTP 介面狀態：

```bash
$ mctp link show
mctpi2c1: net 1, addr 0x10, mtu 254, up
  bus-owner: 0x10
mctpi2c2: net 2, addr 0x11, mtu 254, down
```

**輸出欄位說明**：

| 欄位 | 說明 |
|------|------|
| `net` | MCTP 網路 ID |
| `addr` | 實體地址（如 I2C slave address） |
| `mtu` | 最大傳輸單元 |
| `up/down` | 介面狀態 |
| `bus-owner` | Bus owner 的實體地址 |

### mctp link set

設定 MCTP 介面屬性：

```bash
# 啟用/停用介面
mctp link set mctpi2c1 up
mctp link set mctpi2c1 down

# 設定 MTU（I2C 範圍：68-254）
mctp link set mctpi2c1 mtu 254

# 設定網路 ID
mctp link set mctpi2c1 network 1

# 設定 bus-owner 實體地址
mctp link set mctpi2c1 bus-owner 0x10
```

**參數說明**：

| 參數 | 說明 |
|------|------|
| `up/down` | 啟用或停用介面 |
| `mtu <value>` | 設定 MTU 值 |
| `network <id>` | 設定 MCTP 網路 ID |
| `bus-owner <addr>` | 設定 bus owner 實體地址 |

---

## mctp address

管理本地 EID 地址。

### mctp address show

顯示本地分配的 EID：

```bash
$ mctp address show
dev mctpi2c1: eid 8
dev mctpi2c2: eid 8
```

```bash
# 僅顯示特定介面
$ mctp address show mctpi2c1
dev mctpi2c1: eid 8
```

### mctp address add

新增本地 EID：

```bash
# 在 mctpi2c1 上分配 EID 8
mctp address add 8 dev mctpi2c1
```

> [!NOTE]
> 通常 mctpd 會自動管理本地 EID 分配。手動設定主要用於測試或特殊配置。

### mctp address del

刪除本地 EID：

```bash
mctp address del 8 dev mctpi2c1
```

---

## mctp route

管理 MCTP 路由表。

### mctp route show

顯示路由表：

```bash
$ mctp route show
eid 8: dev mctpi2c1 mtu 0 (local)
eid 10: dev mctpi2c1 mtu 128
eid 50-60: dev mctpi2c1 mtu 0
eid 100: dev mctpi2c1 via 12 mtu 0
```

```bash
# 顯示特定網路的路由
$ mctp route show net 1
```

**路由類型**：

| 類型 | 說明 |
|------|------|
| `local` | 本地 EID |
| `dev` | 直接路由（透過介面） |
| `via/gw` | 閘道路由（透過橋接器） |

### mctp route add

新增路由：

```bash
# 直接路由（單一 EID）
mctp route add 10 via mctpi2c1

# 直接路由並設定 MTU
mctp route add 10 via mctpi2c1 mtu 128

# 範圍路由（EID 50 到 60）
mctp route add 50-60 via mctpi2c1

# 閘道路由（透過 EID 12 的橋接器）
mctp route add 100 gw 12

# 閘道路由指定網路
mctp route add 100 gw 12 net 1 mtu 128
```

**參數說明**：

| 參數 | 說明 |
|------|------|
| `<eid>` | 單一目標 EID |
| `<eid>-<eid>` | EID 範圍 |
| `via <dev>` | 直接路由的輸出介面 |
| `gw <eid>` | 閘道（下一跳）EID |
| `net <id>` | 網路 ID（用於閘道路由） |
| `mtu <value>` | 路由 MTU（0 = 使用介面 MTU） |

### mctp route del

刪除路由：

```bash
# 刪除直接路由
mctp route del 10 via mctpi2c1

# 刪除範圍路由
mctp route del 50-60 via mctpi2c1

# 刪除閘道路由
mctp route del 100 gw 12
mctp route del 100 gw 12 net 1
```

---

## mctp neigh

管理鄰居表（ARP for MCTP）。

### mctp neigh show

顯示鄰居表：

```bash
$ mctp neigh show
eid 10: dev mctpi2c1 lladdr 0x1d
eid 11: dev mctpi2c1 lladdr 0x1e
eid 12: dev mctpi2c1 lladdr 0x1f
```

```bash
# 顯示特定介面的鄰居
$ mctp neigh show dev mctpi2c1
```

**輸出欄位說明**：

| 欄位 | 說明 |
|------|------|
| `eid` | 端點識別碼 |
| `dev` | 網路介面 |
| `lladdr` | 鏈路層地址（I2C slave address） |

### mctp neigh add

新增鄰居條目：

```bash
# EID 10 對應 I2C 地址 0x1d
mctp neigh add 10 dev mctpi2c1 lladdr 0x1d
```

> [!NOTE]
> 通常 mctpd 會在執行 SetupEndpoint 時自動建立鄰居條目。手動設定主要用於靜態配置。

### mctp neigh del

刪除鄰居條目：

```bash
mctp neigh del 10 dev mctpi2c1
```

---

## 實用範例

### 完整設定流程

```bash
#!/bin/bash
# 設定 MCTP 介面並新增端點

# 1. 啟用介面
mctp link set mctpi2c1 up

# 2. 設定網路 ID
mctp link set mctpi2c1 network 1

# 3. 分配本地 EID
mctp address add 8 dev mctpi2c1

# 4. 設定 bus-owner（如果是 bus owner）
mctp link set mctpi2c1 bus-owner 0x10

# 5. 新增遠端端點的鄰居和路由
mctp neigh add 10 dev mctpi2c1 lladdr 0x1d
mctp route add 10 via mctpi2c1

# 查看設定結果
echo "=== Links ==="
mctp link show

echo "=== Addresses ==="
mctp address show

echo "=== Routes ==="
mctp route show

echo "=== Neighbours ==="
mctp neigh show
```

### 檢查網路狀態

```bash
#!/bin/bash
# 快速檢查 MCTP 網路狀態

echo "=== MCTP Network Status ==="
echo ""

echo "Links:"
mctp link show

echo ""
echo "Local EIDs:"
mctp address show

echo ""
echo "Routes:"
mctp route show

echo ""
echo "Neighbours:"
mctp neigh show
```

### 設定橋接路由

```bash
#!/bin/bash
# 設定經由橋接器（EID 12）到達下游端點（EID 50-60）的路由

# 首先設定橋接器
mctp neigh add 12 dev mctpi2c1 lladdr 0x1f
mctp route add 12 via mctpi2c1

# 設定下游端點的閘道路由
mctp route add 50-60 gw 12 net 1
```

---

## 輸出格式

### 詳細模式

目前 mctp 工具沒有 verbose 選項，輸出格式是固定的。

### 常見輸出範例

**mctp link show**：
```
mctpi2c1: net 1, addr 0x10, mtu 254, up
  bus-owner: 0x10
```

**mctp address show**：
```
dev mctpi2c1: eid 8
```

**mctp route show**：
```
eid 8: dev mctpi2c1 mtu 0 (local)
eid 10: dev mctpi2c1 mtu 128
eid 50-60: dev mctpi2c1 mtu 0
```

**mctp neigh show**：
```
eid 10: dev mctpi2c1 lladdr 0x1d
```

---

## 錯誤處理

### 常見錯誤訊息

| 錯誤 | 原因 | 解決方案 |
|------|------|----------|
| `RTNETLINK answers: No such device` | 介面不存在 | 確認介面名稱正確 |
| `RTNETLINK answers: File exists` | 條目已存在 | 先刪除再新增 |
| `RTNETLINK answers: Invalid argument` | 參數無效 | 檢查 EID 範圍（8-254） |
| `RTNETLINK answers: Permission denied` | 權限不足 | 使用 sudo 執行 |

---

## 與 ip 命令對比

| 功能 | mctp | ip |
|------|------|-----|
| 介面管理 | `mctp link` | `ip link` |
| 地址管理 | `mctp address` | `ip address` |
| 路由管理 | `mctp route` | `ip route` |
| 鄰居管理 | `mctp neigh` | `ip neigh` |

---

## 相關文件

- [MctpdDaemon](MctpdDaemon.md) - mctpd 守護程式
- [KernelStack](KernelStack.md) - Linux 核心 MCTP 堆疊
- [QuickStart](QuickStart.md) - 快速入門

---

[← 返回首頁](Home.md)
