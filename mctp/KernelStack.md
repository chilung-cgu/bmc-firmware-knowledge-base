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
struct sockaddr_mctp {
    unsigned short    smctp_family;   // AF_MCTP
    __be16           smctp_reserved;  // 保留
    unsigned int     smctp_network;   // 網路 ID
    struct mctp_addr smctp_addr;      // EID
    __u8             smctp_type;      // 訊息類型
    __u8             smctp_tag;       // 訊息標籤
};

struct mctp_addr {
    __u8 s_addr;                      // 8-bit EID
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

## 核心資料結構

### MCTP 設備

```c
// net/mctp/device.c
struct mctp_dev {
    struct net_device *dev;        // 底層網路裝置
    unsigned int net;              // MCTP 網路 ID
    
    struct mctp_neigh *neigh;      // 鄰居快取
    
    /* 本地 EID 列表 */
    struct list_head local_eids;
};
```

### MCTP 路由

```c
// net/mctp/route.c
struct mctp_route {
    mctp_eid_t min, max;           // EID 範圍
    unsigned int mtu;              // 路由 MTU
    struct mctp_dev *dev;          // 輸出裝置
    mctp_eid_t gw;                 // 閘道 EID（如果有）
    unsigned int type;             // 路由類型
};
```

### MCTP 鄰居

```c
// net/mctp/neigh.c
struct mctp_neigh {
    mctp_eid_t eid;                // 端點 EID
    struct mctp_dev *dev;          // 裝置
    u8 hwaddr[MAX_ADDR_LEN];       // 實體地址
    size_t hwaddr_len;             // 地址長度
};
```

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

| 版本 | 新增功能 |
|------|----------|
| 5.15 | 初始 MCTP 支援（AF_MCTP socket、基礎路由） |
| 5.16 | mctp-i2c 驅動程式 |
| 5.17 | 改進的錯誤處理 |
| 5.18 | 多網路支援改進 |
| 6.0+ | 持續改進和錯誤修復 |

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
