# Linux 核心 MCTP 堆疊 (Kernel Stack)

本文說明 Linux 核心中的 MCTP 協議堆疊實作，包括 socket API、傳輸驅動程式和配置方式。

---

## 概述

Linux 核心自 **5.15** 版本開始支援 MCTP 協議。核心實作提供：

- **AF_MCTP** socket 家族
- MCTP 路由和鄰居管理
- 訊息分片和重組
- 多種傳輸驅動程式（I2C、Serial、PCIe）

```
┌─────────────────────────────────────────────────────────────────┐
│                        使用者空間                               │
│                                                                 │
│    ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐    │
│    │   mctp CLI   │  │    mctpd     │  │   Application    │    │
│    └──────┬───────┘  └──────┬───────┘  └────────┬─────────┘    │
│           │                 │                   │               │
│           │ Netlink         │ AF_MCTP           │ AF_MCTP       │
│           ▼                 ▼                   ▼               │
├─────────────────────────────────────────────────────────────────┤
│                        核心空間                                  │
│                                                                 │
│    ┌────────────────────────────────────────────────────────┐  │
│    │                    net/mctp/                            │  │
│    │                                                         │  │
│    │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐ │  │
│    │  │ af_mctp.c   │  │  route.c    │  │  neigh.c        │ │  │
│    │  │ (Socket)    │  │  (Routing)  │  │  (Neighbour)    │ │  │
│    │  └─────────────┘  └─────────────┘  └─────────────────┘ │  │
│    │                                                         │  │
│    │  ┌─────────────────────────────────────────────────┐   │  │
│    │  │                 device.c                         │   │  │
│    │  │           (MCTP Network Device)                  │   │  │
│    │  └─────────────────────────────────────────────────┘   │  │
│    │                                                         │  │
│    └────────────────────────────────────────────────────────┘  │
│                               │                                 │
│    ┌────────────────────────────────────────────────────────┐  │
│    │              drivers/net/mctp/                          │  │
│    │                                                         │  │
│    │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐ │  │
│    │  │ mctp-i2c    │  │ mctp-serial │  │ mctp-pcie-vdm   │ │  │
│    │  └─────────────┘  └─────────────┘  └─────────────────┘ │  │
│    │                                                         │  │
│    └────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Socket API

### 建立 MCTP Socket

```c
#include <sys/socket.h>
#include <linux/mctp.h>

// 建立 MCTP datagram socket
int sock = socket(AF_MCTP, SOCK_DGRAM, 0);
```

### sockaddr_mctp 結構

```c
// include/uapi/linux/mctp.h (Upstream Kernel)
typedef __u8 mctp_eid_t;

struct mctp_addr {
    mctp_eid_t s_addr;                     // 8-bit EID
};

struct sockaddr_mctp {
    __kernel_sa_family_t smctp_family;     // AF_MCTP
    __u16                __smctp_pad0;     // 填充（padding），不使用
    unsigned int         smctp_network;    // 網路 ID
    struct mctp_addr     smctp_addr;       // EID
    __u8                 smctp_type;       // 訊息類型
    __u8                 smctp_tag;        // 訊息標籤
    __u8                 __smctp_pad1;     // 填充（padding），不使用
};
```

### 傳送訊息

```c
struct sockaddr_mctp addr = {
    .smctp_family = AF_MCTP,
    .smctp_network = 1,                    // 網路 ID
    .smctp_addr.s_addr = 10,               // 目標 EID
    .smctp_type = MCTP_MSG_TYPE_PLDM,      // 訊息類型（如 PLDM = 0x01）
    .smctp_tag = MCTP_TAG_OWNER,           // 請求核心分配標籤
};

char msg[] = { /* PLDM message data */ };

sendto(sock, msg, sizeof(msg), 0,
       (struct sockaddr *)&addr, sizeof(addr));
```

### 接收訊息

```c
char buf[256];
struct sockaddr_mctp from;
socklen_t fromlen = sizeof(from);

ssize_t len = recvfrom(sock, buf, sizeof(buf), 0,
                       (struct sockaddr *)&from, &fromlen);

// from.smctp_addr.s_addr 包含來源 EID
// from.smctp_network 包含網路 ID
// from.smctp_tag 包含訊息標籤
```

### 訊息標籤管理

MCTP 使用 3-bit 標籤（0-7）來配對請求和回應：

```c
// 發送請求時，設定 MCTP_TAG_OWNER 讓核心分配標籤
addr.smctp_tag = MCTP_TAG_OWNER;

// 回應時，使用接收到的標籤（不設定 MCTP_TAG_OWNER）
addr.smctp_tag = received_tag;  // 不含 MCTP_TAG_OWNER bit
```

---

## Netlink 配置

MCTP 堆疊透過 Netlink 介面進行配置。`mctp` 命令行工具封裝了這些介面。

### 介面管理

```bash
# 查看所有 MCTP 介面
mctp link show

# 設定介面
mctp link set mctpi2c1 up
mctp link set mctpi2c1 mtu 254
mctp link set mctpi2c1 network 1
mctp link set mctpi2c1 bus-owner 0x10
```

### EID 管理

```bash
# 查看本地 EID
mctp address show

# 新增本地 EID
mctp address add 8 dev mctpi2c1

# 刪除本地 EID
mctp address del 8 dev mctpi2c1
```

### 路由管理

```bash
# 查看路由
mctp route show

# 新增直接路由（透過介面）
mctp route add 10 via mctpi2c1

# 新增範圍路由
mctp route add 50-60 via mctpi2c1

# 新增閘道路由（透過橋接器）
mctp route add 100 gw 12 net 1

# 設定路由 MTU
mctp route add 10 via mctpi2c1 mtu 128
```

### 鄰居管理

```bash
# 查看鄰居表
mctp neigh show

# 新增鄰居（EID 到實體地址映射）
mctp neigh add 10 dev mctpi2c1 lladdr 0x1d

# 刪除鄰居
mctp neigh del 10 dev mctpi2c1
```

---

## 傳輸驅動程式

### mctp-i2c 驅動程式

最常用的 MCTP 傳輸驅動程式，用於 I2C/SMBus 通訊：

```
┌─────────────────────────────────────────────────────────────────┐
│                    mctp-i2c Driver                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  核心模組：mctp-i2c.ko                                          │
│  裝置樹綁定：compatible = "mctp-i2c-controller"                  │
│  介面名稱：mctpi2c<N>                                           │
│                                                                 │
│  MTU 範圍：                                                     │
│  • 最小 MTU：68 bytes                                           │
│  • 最大 MTU：254 bytes（I2C 限制）                              │
│                                                                 │
│  硬體地址：7-bit I2C slave address                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Device Tree 範例**：

```dts
&i2c6 {
    status = "okay";

    mctp@10 {
        compatible = "mctp-i2c-controller";
        reg = <(0x10 | I2C_OWN_SLAVE_ADDRESS)>;
    };
};
```

**Kernel Config**：

```
CONFIG_MCTP=y
CONFIG_MCTP_TRANSPORT_I2C=y
```

### mctp-serial 驅動程式

用於串列埠通訊：

```
CONFIG_MCTP_SERIAL=y
```

---

## 核心配置選項

```
# 核心 MCTP 支援
CONFIG_MCTP=y

# MCTP 傳輸驅動程式
CONFIG_MCTP_TRANSPORT_I2C=y        # I2C/SMBus 傳輸
CONFIG_MCTP_SERIAL=y               # Serial 傳輸

# 相關子系統
CONFIG_NET=y                       # 網路支援
CONFIG_I2C=y                       # I2C 子系統
CONFIG_I2C_SLAVE=y                 # I2C slave 模式
```

---

## 核心資料結構與設計與範例解析

這幾個資料結構的設計，主要對應網路通訊中 **「我（主機）」**、**「路（通道）」** 與 **「對方（端點）」** 之間的關係。

為了便於理解，我們使用一個具體情境來對照：**「BMC (EID 8) 要去讀取一張連在 I2C Bus 6 上的溫度感測器 (EID 50, I2C Addr 0x4A)」**。

### 1. `struct mctp_dev` ——「網卡（通道）」

這是 **「物理通道的負責人」**。在上述情境中，它代表 **"I2C Bus 6 控制器"**。

- **功能**：管理物理層的狀態（UP/DOWN）、MTU 大小、以及該通道上的網路 ID。
- **情境對應**：BMC 可能有多條 I2C bus，每一條都需要一個 `mctp_dev` 來管理。

```c
// include/net/mctpdevice.h (Upstream Kernel)
struct mctp_dev {
    struct net_device     *dev;         // 底層網路裝置 (例如 "mctpi2c6")

    refcount_t            refs;         // 引用計數

    unsigned int          net;          // MCTP 網路 ID
    enum mctp_phys_binding binding;    // 物理綁定類型 (I2C/PCIe/Serial 等)

    const struct mctp_netdev_ops *ops; // 裝置操作函數指標

    /* 本地 EID 管理：addrs 是一個動態陣列，
     * 儲存分配給此裝置的所有本地 EID */
    u8                    *addrs;       // 本地 EID 陣列
    size_t                num_addrs;    // 本地 EID 數量
    spinlock_t            addrs_lock;   // 保護 addrs 的自旋鎖

    struct rcu_head       rcu;          // RCU 保護
};
```

> **⚠️ 注意**：`mctp_dev` 中**沒有**直接指向 `mctp_neigh` 的指標。鄰居 (`mctp_neigh`) 是被串在 per-net namespace 的全域鄰居列表中（`struct netns_mctp.neighbours`），並透過 `mctp_neigh.dev` 欄位反向關聯到對應的 `mctp_dev`。

### 2. `struct mctp_neigh` ——「鄰居（通訊錄）」

這是 **「住在這條路上的特定住戶」**。在情境中，它代表那個 **"溫度感測器"**。

- **功能**：MCTP EID 是邏輯地址 (EID 50)，但要在 I2C 傳輸，必須知道對方的 **實體地址 (HW Address)**，即 I2C Address (0x4A)。這個結構就是 **「對照表 (ARP Table)」**。
- **實作細節**：使用 Linux 標準的 `struct list_head` 串接在 per-net namespace 的全域鄰居列表 (`struct netns_mctp.neighbours`) 中。

```c
// include/net/mctp.h (Upstream Kernel)
enum mctp_neigh_source {
    MCTP_NEIGH_STATIC,       // 靜態設定 (由使用者手動新增)
    MCTP_NEIGH_DISCOVER,     // 動態發現
};

struct mctp_neigh {
    struct mctp_dev          *dev;          // 此鄰居所在的裝置
    mctp_eid_t               eid;          // 對方的 EID
    enum mctp_neigh_source   source;       // 來源

    unsigned char            ha[MAX_ADDR_LEN]; // 實體地址 (如 I2C Addr 0x4A)

    struct list_head         list;         // 鏈結串列節點
    struct rcu_head          rcu;          // RCU 保護
};
```

### 3. `struct mctp_route` ——「導航規則」

這是 **「指路牌」**。當 App 說「我要寄信給 EID 50」時，Kernel 透過它知道該走哪條路。

- **功能**：路由表，決定封包該交給哪個 `mctp_dev` 送出。
- **實作細節**：同樣使用 `struct list_head` 串接在全域路由表 (`net->mctp.routes`) 中。

```c
// include/net/mctp.h (Upstream Kernel)
struct mctp_route {
    mctp_eid_t              min, max;      // EID 範圍

    unsigned char           type;          // 路由類型

    unsigned int            mtu;           // 路徑 MTU

    enum {
        MCTP_ROUTE_DIRECT,                 // 直接連接
        MCTP_ROUTE_GATEWAY,                // 透過閘道
    } dst_type;
    union {
        struct mctp_dev     *dev;          // 出口裝置 (Direct)
        struct mctp_fq_addr gateway;       // 閘道地址 (Gateway)
    };

    int (*output)(struct mctp_dst *dst,    // 輸出函數指標
                  struct sk_buff *skb);

    struct list_head        list;          // 鏈結串列節點
    refcount_t              refs;          // 引用計數
    struct rcu_head         rcu;           // RCU 保護
};
```

---

---

### 資料流動的動態過程

#### 1. 封包傳送流程 (Tx) - `sendto(EID=50)`

當 App 呼叫 `sendto(EID=50)` 時，核心內部的運作流程如下：

1.  **查路由 (`mctp_route`)**：
    - Kernel: "有人要寄給 EID 50，該走哪？"
    - 查表：在 `net->mctp.routes` 串列中遍歷，找到吻合的 `mctp_route`。
    - 決策：交給 **I2C Bus 6 (`mctp_dev`)**。

2.  **查鄰居 (`mctp_neigh`)**：
    - Driver (I2C Bus 6): "我要送給 EID 50，但 I2C 暫存器要填哪個 Slave Address？"
    - 查表：透過 `mctp_neigh_lookup()` 函數（遍歷 `netns_mctp.neighbours` 串列）找到 EID=50 的 `mctp_neigh`。
    - 結果：找到 `mctp_neigh { eid=50, ha=0x4A }`。
    - 決策：硬體傳輸目標設為 **0x4A**。

3.  **實際發送 (`mctp_dev` -> Hardware)**：
    - Kernel 操作 I2C 控制器硬體，發出訊號：`[Start] [Addr: 0x4A] [Data...] [Stop]`。

#### 2. 封包接收流程 (Rx)

當感測器回傳資料時，流程是反過來的：

1.  **硬體接收**：I2C Controller 收到數據。
2.  **建立鄰居關聯**：Driver 透過來源 I2C Address (0x4A) 查找或建立 `mctp_neigh`。
3.  **上層傳遞**：封包透過 `mctp_dev` 往上層核心協議堆疊送。
4.  **Socket 接收**：App 透過 `recvfrom` 收到資料。

---

### 資料結構關聯說明

`mctp_dev`、`mctp_neigh`、`mctp_route` 三者的關聯如下：

- **`mctp_neigh` → `mctp_dev`**：每個 `mctp_neigh` 透過 `dev` 欄位指向它所屬的 `mctp_dev`。
- **`mctp_route` → `mctp_dev`**：Direct 路由透過 `dev` 欄位指向出口裝置。
- **`mctp_dev` → `mctp_neigh`**：`mctp_dev` **沒有**直接指標指向 `mctp_neigh`。鄰居列表存放在 per-net namespace 的 `netns_mctp.neighbours`，需要查詢時透過 `mctp_neigh_lookup()` 遍歷全域列表並比對 `mctp_neigh.dev` 欄位。
- **`mctp_route` 和 `mctp_neigh` 的列表**：都使用 `struct list_head` 串接，分別存放在 `netns_mctp.routes` 和 `netns_mctp.neighbours` 中。

> 關於 `struct list_head` 的詳細說明，請參考 [Linux Kernel Linked List](../linux-kernel-syntax/LinkedList.md)。

### 記憶體佈局示意圖

```
                        struct netns_mctp (per-net namespace)
                    ┌─────────────────────────────────────────────┐
                    │  routes (list_head)                          │
                    │   └─ mctp_route ─── mctp_route ─── ...      │
                    │                                              │
                    │  neighbours (list_head)                      │
                    │   └─ mctp_neigh ── mctp_neigh ── ...        │
                    └─────────────────────────────────────────────┘
                               │                │
                               │ (.dev)         │ (.dev)
                               ▼                ▼
                    ┌──────────────────┐  ┌──────────────────┐
                    │ mctp_dev (Bus 6) │  │ mctp_dev (Bus 7) │
                    │  .net = 1        │  │  .net = 2        │
                    │  .addrs = [8]    │  │  .addrs = [9]    │
                    └──────────────────┘  └──────────────────┘
```

**總結：**

- `mctp_dev` = **硬體介面** (I2C Controller)
- `mctp_route` = **地圖** (Routing Table，用 `list_head` 串在 `netns_mctp.routes`)
- `mctp_neigh` = **電話簿** (ARP Table: EID → HW Addr，用 `list_head` 串在 `netns_mctp.neighbours`)

---

## sysfs 介面

```bash
# 查看 MCTP 網路裝置資訊
ls /sys/class/net/mctpi2c1/

# 相關檔案
/sys/class/net/mctpi2c1/address      # 實體地址
/sys/class/net/mctpi2c1/mtu          # MTU
/sys/class/net/mctpi2c1/operstate    # 運作狀態
```

---

## procfs 資訊

```bash
# MCTP 協議統計
cat /proc/net/mctp
```

---

## 核心版本歷史

| 版本 | 新增功能                                   |
| ---- | ------------------------------------------ |
| 5.15 | 初始 MCTP 支援（AF_MCTP socket、基礎路由） |
| 5.16 | mctp-i2c 驅動程式                          |
| 5.17 | 改進的錯誤處理                             |
| 5.18 | 多網路支援改進                             |
| 6.0+ | 持續改進和錯誤修復                         |

---

## 疑難排解

### 檢查核心支援

```bash
# 確認核心配置
zcat /proc/config.gz | grep MCTP

# 或查看模組
lsmod | grep mctp

# 載入模組
modprobe mctp
modprobe mctp-i2c
```

### 檢查介面

```bash
# 列出 MCTP 介面
ip link show type mctp

# 或使用 mctp 工具
mctp link show
```

### 除錯

```bash
# 啟用動態除錯
echo 'module mctp +p' > /sys/kernel/debug/dynamic_debug/control
echo 'module mctp_i2c +p' > /sys/kernel/debug/dynamic_debug/control

# 查看核心日誌
dmesg | grep -i mctp
```

---

## 範例程式

### 簡單 MCTP 客戶端

```c
#include <stdio.h>
#include <string.h>
#include <sys/socket.h>
#include <linux/mctp.h>

#define MCTP_TYPE_CONTROL 0x00

int main() {
    int sock;
    struct sockaddr_mctp addr;
    char request[] = {0x00, 0x80, 0x02};  // Get Endpoint ID
    char response[256];
    socklen_t addrlen;

    // 建立 socket
    sock = socket(AF_MCTP, SOCK_DGRAM, 0);
    if (sock < 0) {
        perror("socket");
        return 1;
    }

    // 設定目標地址
    memset(&addr, 0, sizeof(addr));
    addr.smctp_family = AF_MCTP;
    addr.smctp_network = 1;
    addr.smctp_addr.s_addr = 10;  // 目標 EID
    addr.smctp_type = MCTP_TYPE_CONTROL;
    addr.smctp_tag = MCTP_TAG_OWNER;

    // 發送請求
    if (sendto(sock, request, sizeof(request), 0,
               (struct sockaddr *)&addr, sizeof(addr)) < 0) {
        perror("sendto");
        return 1;
    }

    // 接收回應
    addrlen = sizeof(addr);
    ssize_t len = recvfrom(sock, response, sizeof(response), 0,
                           (struct sockaddr *)&addr, &addrlen);
    if (len < 0) {
        perror("recvfrom");
        return 1;
    }

    printf("Received %zd bytes from EID %d\n",
           len, addr.smctp_addr.s_addr);

    return 0;
}
```

---

## 相關文件

- [MCTPOverview](MCTPOverview.md) - MCTP 協議概述
- [MctpCommand](MctpCommand.md) - mctp 命令行工具
- [Architecture](Architecture.md) - 系統架構

---

## 外部連結

- [Linux Kernel MCTP 文件](https://docs.kernel.org/networking/mctp.html)
- [mctp(7) man page](https://man7.org/linux/man-pages/man7/mctp.7.html)
- [Code Construct MCTP 部落格](https://codeconstruct.com.au/docs/mctp-on-linux-introduction/)

---

[← 返回首頁](Home.md)
