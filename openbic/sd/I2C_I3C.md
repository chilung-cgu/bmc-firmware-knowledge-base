# I2C_I3C（I2C/I3C 介面）

本文說明 OpenBIC 的 I2C 與 I3C 介面使用，包括 HAL 層 API、Master/Target 模式，以及 I3C HUB 配置。

---

## I2C 介面

### 概述

AST1030 提供 16 個 I2C 控制器，支援 Master 和 Slave (Target) 模式。

### HAL 層 API

```c
// hal_i2c.h

// I2C 訊息結構
typedef struct _I2C_MSG_ {
    uint8_t bus;            // I2C bus 編號
    uint8_t target_addr;    // 目標地址 (7-bit)
    uint8_t tx_len;         // 發送長度
    uint8_t rx_len;         // 接收長度
    uint8_t data[I2C_BUFF_SIZE];  // 資料緩衝區
} I2C_MSG;

// 主要函數
int i2c_master_read(I2C_MSG *msg, uint8_t retry);
int i2c_master_write(I2C_MSG *msg, uint8_t retry);
void util_init_I2C(void);
```

### I2C 讀取範例

```c
int read_sensor_data(uint8_t bus, uint8_t addr, uint8_t reg, uint8_t *data)
{
    int retry = 5;
    I2C_MSG msg = { 0 };

    msg.bus = bus;
    msg.target_addr = addr;
    msg.tx_len = 1;
    msg.rx_len = 2;
    msg.data[0] = reg;

    if (i2c_master_read(&msg, retry) != 0) {
        LOG_ERR("I2C read failed");
        return -1;
    }

    data[0] = msg.data[0];
    data[1] = msg.data[1];
    return 0;
}
```

### I2C 寫入範例

```c
int write_cpld_register(uint8_t reg, uint8_t value)
{
    int retry = 3;
    I2C_MSG msg = { 0 };

    msg.bus = CPLD_IO_I2C_BUS;
    msg.target_addr = CPLD_IO_I2C_ADDR;
    msg.tx_len = 2;
    msg.data[0] = reg;
    msg.data[1] = value;

    if (i2c_master_write(&msg, retry) != 0) {
        LOG_ERR("Failed to write CPLD register");
        return -1;
    }

    return 0;
}
```

### I2C Target 模式

OpenBIC 作為 I2C Slave 接收 IPMB 訊息：

```c
// plat_i2c_target.c

// Target 配置表
const struct _i2c_target_config I2C_TARGET_CONFIG_TABLE[] = {
    { IPMB_config[0].bus, IPMB_config[0].self_addr },
    // ...
};

// 啟用 Target
for (int index = 0; index < MAX_TARGET_NUM; index++) {
    if (I2C_TARGET_ENABLE_TABLE[index])
        i2c_target_control(
            index, 
            (struct _i2c_target_config *)&I2C_TARGET_CONFIG_TABLE[index],
            1);
}
```

---

## I3C 介面

### 概述

I3C 是 MIPI 聯盟定義的新一代介面，相容 I2C 並提供更高速度與進階功能。

| 特性 | I2C | I3C |
|------|-----|-----|
| 速度 | 最高 1 MHz | 最高 12.5 MHz |
| 地址分配 | 靜態 | 動態 (DAA) |
| 中斷 | 需外部線路 | IBI (帶內中斷) |
| Hot-Join | 不支援 | 支援 |

### HAL 層 API

```c
// hal_i3c.h

// I3C 訊息結構
typedef struct _I3C_MSG_ {
    uint8_t bus;
    uint8_t target_addr;
    uint8_t tx_len;
    uint8_t rx_len;
    uint8_t data[I3C_MAX_DATA_SIZE];
} I3C_MSG;

// 主要函數
int i3c_attach(I3C_MSG *msg);
int i3c_transfer(I3C_MSG *msg);
int i3c_brocast_ccc(I3C_MSG *msg, uint8_t ccc_id, uint8_t addr);
int i3c_set_pid(I3C_MSG *msg, uint16_t pid);
void util_init_i3c(void);
```

### CCC (Common Command Codes)

| CCC | 值 | 說明 |
|-----|-----|------|
| RSTDAA | 0x06 | 重置 DAA |
| SETAASA | 0x29 | 設定靜態地址為動態地址 |
| SETDASA | 0x87 | 設定動態地址 |
| GETPID | 0x8D | 取得 PID |
| GETBCR | 0x8E | 取得 BCR |
| GETDCR | 0x8F | 取得 DCR |

### I3C 初始化

```c
// plat_init.c
void pal_pre_init()
{
    I3C_MSG i3c_msg = { 0 };
    i3c_msg.bus = I3C_BUS_HUB;
    i3c_msg.target_addr = I3C_ADDR_HUB;

    // 重置 DAA (執行兩次確保完全重置)
    for (int i = 0; i < 2; i++) {
        int ret = i3c_brocast_ccc(&i3c_msg, I3C_CCC_RSTDAA, I3C_BROADCAST_ADDR);
        if (ret != 0) {
            printf("Error to reset daa. count = %d\n", i);
        }
    }

    // 設定靜態地址為動態地址
    int ret = i3c_brocast_ccc(&i3c_msg, I3C_CCC_SETAASA, I3C_BROADCAST_ADDR);
    if (ret != 0) {
        printf("Error to set daa\n");
    }

    // 附加裝置
    i3c_attach(&i3c_msg);
}
```

---

## I3C HUB

### 支援的 I3C HUB

yv4-sd 支援兩種 I3C HUB：

| 型號 | 製造商 | Device Info |
|------|--------|-------------|
| RG3MXXB12 | Renesas | 0x0105 |
| P3H284X | NXP | 0x0C01 |

### HUB 類型偵測

```c
// plat_class.c
static uint16_t i3c_hub_type = I3C_HUB_TYPE_UNKNOWN;

void init_i3c_hub_type(void)
{
    if (rg3mxxb12_get_device_info_i3c(I3C_BUS_HUB, &i3c_hub_type) &&
        (i3c_hub_type == RG3M87B12_DEVICE_INFO)) {
        LOG_INF("I3C hub type: rg3mxxb12");
    } else if (p3h284x_get_device_info_i3c(I3C_BUS_HUB, &i3c_hub_type) &&
               (i3c_hub_type == P3H2840_DEVICE_INFO)) {
        LOG_INF("I3C hub type: p3h284x");
    } else {
        LOG_ERR("I3C hub get device type fail");
    }
}
```

### HUB 初始化

```c
// plat_init.c
init_i3c_hub_type();
i3c_hub_type = get_i3c_hub_type();

// 根據 HUB 類型初始化
if (i3c_hub_type == P3H2840_DEVICE_INFO) {
    if (!p3h284x_i3c_mode_only_init(&i3c_msg, p3h284x_cmd_initial, 
                                     P3H284X_CMD_INITIAL_SIZE)) {
        printk("failed to initialize 1ou p3h284x\n");
    }
} else {
    if (!rg3mxxb12_i3c_mode_only_init(&i3c_msg, rg3mxxb12_cmd_initial, 
                                       RG3MXXB12_CMD_INITIAL_SIZE)) {
        printk("failed to initialize 1ou rg3mxxb12\n");
    }
}
```

---

## MCTP over I3C

### 配置

```c
// plat_mctp.c
static mctp_port plat_mctp_port[] = {
    // I2C to BMC
    { .conf.smbus_conf.addr = I2C_ADDR_BIC,
      .conf.smbus_conf.bus = I2C_BUS_BMC,
      .medium_type = MCTP_MEDIUM_TYPE_SMBUS },
    
    // I3C Target (BMC 連接)
    { .conf.i3c_conf.addr = I3C_STATIC_ADDR_BMC,
      .conf.i3c_conf.bus = I3C_BUS_BMC,
      .medium_type = MCTP_MEDIUM_TYPE_TARGET_I3C },
    
    // I3C Controller (FF BIC)
    { .conf.i3c_conf.addr = I3C_STATIC_ADDR_FF_BIC,
      .conf.i3c_conf.bus = I3C_BUS_HUB,
      .medium_type = MCTP_MEDIUM_TYPE_CONTROLLER_I3C },
    
    // I3C Controller (WF BIC)
    { .conf.i3c_conf.addr = I3C_STATIC_ADDR_WF_BIC,
      .conf.i3c_conf.bus = I3C_BUS_HUB,
      .medium_type = MCTP_MEDIUM_TYPE_CONTROLLER_I3C },
};
```

### 路由表

```c
mctp_route_entry plat_mctp_route_tbl[] = {
    { MCTP_EID_BMC, I2C_BUS_BMC, I2C_ADDR_BMC, .set_endpoint = false },
    { MCTP_EID_BMC, I3C_BUS_BMC, I3C_STATIC_ADDR_BMC, .set_endpoint = false },
    { MCTP_EID_FF_BIC, I3C_BUS_HUB, I3C_STATIC_ADDR_FF_BIC, .set_endpoint = true },
    { MCTP_EID_WF_BIC, I3C_BUS_HUB, I3C_STATIC_ADDR_WF_BIC, .set_endpoint = true },
    // ...
};
```

---

## Bus 配置

### yv4-sd I2C Bus 配置

| Bus | 用途 | 說明 |
|-----|------|------|
| I2C_BUS0 | 感測器 | 溫度監控 IC |
| I2C_BUS2 | BMC | IPMB 通訊 |
| I2C_BUS4 | VR | 電壓調節器 |
| I2C_BUS5 | CPLD | IO 擴展 |

### plat_i2c.h 定義

```c
// plat_i2c.h
#define I2C_BUS_BMC         2
#define I2C_BUS_SENSOR      0
#define I2C_BUS_VR          4
#define CPLD_IO_I2C_BUS     5
#define CPLD_IO_I2C_ADDR    0x21

#define I3C_BUS_BMC         0
#define I3C_BUS_HUB         0
#define I3C_ADDR_HUB        0x70
#define I3C_STATIC_ADDR_BMC 0x40
```

---

## I2C Mux

OpenBIC 支援 I2C Mux 操作：

```c
// common_i2c_mux.c
int i2c_mux_select(uint8_t bus, uint8_t addr, uint8_t channel)
{
    I2C_MSG msg = { 0 };
    msg.bus = bus;
    msg.target_addr = addr;
    msg.tx_len = 1;
    msg.data[0] = (1 << channel);
    
    return i2c_master_write(&msg, 3);
}

// TCA9548 支援
#include "i2c-mux-tca9548.c"
```

---

## Shell 命令

```bash
# I2C 掃描
uart:~$ i2c scan I2C_0

# I3C 附加裝置
uart:~$ i3c attach I3C_0

# I3C CCC 命令
uart:~$ i3c ccc I3C_0 ...
```

---

## 相關文件

- [AST1030Overview](AST1030Overview.md) - 晶片概述
- [MCTPOverview](MCTPOverview.md) - MCTP 傳輸協議
- [SensorFramework](SensorFramework.md) - 感測器讀取

---

*返回 [Home](Home.md)*
