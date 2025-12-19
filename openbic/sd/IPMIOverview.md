# IPMIOverview（IPMI 命令處理）

本文說明 OpenBIC 的 IPMI 命令處理架構，包括 NetFn/Cmd 處理流程與 OEM 命令擴展。

---

## IPMI 概述

**IPMI** (Intelligent Platform Management Interface) 是標準的伺服器管理介面。OpenBIC 實作 IPMI 命令處理，與 BMC 和 Host 通訊。

### 傳輸介面

| 介面 | 說明 |
|------|------|
| **IPMB** | I2C 介面，與 BMC 通訊 |
| **KCS** | Keyboard Controller Style，與 Host 通訊 |
| **PLDM** | PLDM OEM 橋接 |

---

## IPMI 訊息結構

### ipmi_msg 結構

```c
// ipmi.h
typedef struct _ipmi_msg {
    uint8_t netfn;          // Network Function
    uint8_t cmd;            // Command
    uint8_t completion_code;// 完成代碼
    uint16_t data_len;      // 資料長度
    uint8_t *data;          // 資料指標
    uint8_t seq_source;     // 序列來源
    uint8_t InF_source;     // 介面來源
    uint8_t InF_target;     // 目標介面
    // ...
} ipmi_msg;

typedef struct _ipmi_msg_cfg {
    ipmi_msg buffer;
    // 額外配置...
} ipmi_msg_cfg;
```

### Network Function

| NetFn | 說明 |
|-------|------|
| 0x00/0x01 | Chassis |
| 0x04/0x05 | Sensor/Event |
| 0x06/0x07 | App |
| 0x0A/0x0B | Storage |
| 0x2C/0x2D | Bridge |
| 0x30/0x31 | Transport |
| 0x38/0x39 | OEM (1S) |

---

## 命令處理流程

```
┌──────────────────────────────────────────────────────────────┐
│                      IPMI Handler                            │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────┐    ┌──────────┐    ┌──────────────────────┐  │
│  │  IPMB    │    │   KCS    │    │   PLDM OEM Bridge    │  │
│  │   RX     │    │    RX    │    │         RX           │  │
│  └────┬─────┘    └────┬─────┘    └──────────┬───────────┘  │
│       │               │                      │              │
│       └───────────────┴──────────────────────┘              │
│                          │                                   │
│               ┌──────────▼──────────┐                       │
│               │     ipmi_msgq       │                       │
│               │   (Message Queue)   │                       │
│               └──────────┬──────────┘                       │
│                          │                                   │
│               ┌──────────▼──────────┐                       │
│               │   IPMI_handler      │                       │
│               │      Thread         │                       │
│               └──────────┬──────────┘                       │
│                          │                                   │
│               ┌──────────▼──────────┐                       │
│               │  ipmi_cmd_handle    │                       │
│               └──────────┬──────────┘                       │
│                          │                                   │
│     ┌────────────────────┼────────────────────┐             │
│     │          NetFn     │        Router      │             │
│     ▼                    ▼                    ▼             │
│  ┌──────────┐   ┌──────────────┐   ┌──────────────┐        │
│  │ Chassis  │   │   Sensor     │   │     OEM      │        │
│  │ Handler  │   │   Handler    │   │   Handler    │        │
│  └──────────┘   └──────────────┘   └──────────────┘        │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 命令處理器

### 主處理函數

```c
// ipmi.c
void ipmi_cmd_handle(void *parameters, void *arvg0, void *arvg1)
{
    ipmi_msg_cfg *msg_cfg = (ipmi_msg_cfg *)parameters;
    ipmi_msg *msg = &msg_cfg->buffer;
    
    // 根據 NetFn 分發
    switch (msg->netfn) {
    case NETFN_CHASSIS_REQ:
        CHASSIS_handler(msg);
        break;
    case NETFN_SENSOR_REQ:
        SENSOR_handler(msg);
        break;
    case NETFN_APP_REQ:
        APP_handler(msg);
        break;
    case NETFN_STORAGE_REQ:
        STORAGE_handler(msg);
        break;
    case NETFN_OEM_1S_REQ:
        OEM_1S_handler(msg);
        break;
    case NETFN_OEM_REQ:
        OEM_handler(msg);
        break;
    default:
        msg->completion_code = CC_INVALID_CMD;
        break;
    }
    
    // 發送回應
    if (msg->InF_source != SELF) {
        ipmb_send_response(msg, ...);
    }
}
```

### 處理器分發表

```c
// oem_1s_handler.c
typedef void (*oem_1s_handler_func)(ipmi_msg *msg);

struct oem_1s_cmd_handler_entry {
    uint8_t cmd;
    oem_1s_handler_func handler;
};

static struct oem_1s_cmd_handler_entry oem_1s_handler_tbl[] = {
    { CMD_OEM_1S_MSG_OUT, OEM_1S_MSG_OUT },
    { CMD_OEM_1S_GET_GPIO, OEM_1S_GET_GPIO },
    { CMD_OEM_1S_FW_UPDATE, OEM_1S_FW_UPDATE },
    { CMD_OEM_1S_GET_FW_VERSION, OEM_1S_GET_FW_VERSION },
    { CMD_OEM_1S_GET_POST_CODE, OEM_1S_GET_POST_CODE },
    { CMD_OEM_1S_SENSOR_POLL_EN, OEM_1S_SENSOR_POLL_EN },
    { CMD_OEM_1S_ACCURACY_SENSOR_READING, OEM_1S_ACCURACY_SENSOR_READING },
    // ...
};
```

---

## 常用 OEM 命令

### OEM_1S_GET_GPIO

```c
// oem_1s_handler.c
__weak OEM_1S_GET_GPIO(ipmi_msg *msg)
{
    CHECK_NULL_ARG(msg);

    if (msg->data_len != 0) {
        msg->completion_code = CC_INVALID_LENGTH;
        return;
    }

    // 取得所有 GPIO 狀態
    uint8_t gpio_cnt = gpio_get_count();
    uint8_t gpio_data_size = (gpio_cnt + 7) / 8;
    
    for (int i = 0; i < gpio_cnt; i++) {
        if (gpio_get(i) == GPIO_HIGH) {
            msg->data[i / 8] |= (1 << (i % 8));
        }
    }

    msg->data_len = gpio_data_size;
    msg->completion_code = CC_SUCCESS;
}
```

### OEM_1S_FW_UPDATE

```c
__weak OEM_1S_FW_UPDATE(ipmi_msg *msg)
{
    CHECK_NULL_ARG(msg);

    uint8_t target = msg->data[0];
    uint32_t offset = (msg->data[1] << 24) | (msg->data[2] << 16) |
                      (msg->data[3] << 8) | msg->data[4];
    uint16_t length = (msg->data[5] << 8) | msg->data[6];
    uint8_t *data = &msg->data[7];

    switch (target) {
    case UPDATE_TARGET_BIC:
        // BIC 韌體更新
        do_bic_update(offset, length, data);
        break;
    case UPDATE_TARGET_BIOS:
        // BIOS 更新
        do_bios_update(offset, length, data);
        break;
    default:
        msg->completion_code = CC_INVALID_DATA_FIELD;
        return;
    }

    msg->completion_code = CC_SUCCESS;
    msg->data_len = 0;
}
```

### OEM_1S_GET_FW_VERSION

```c
__weak OEM_1S_GET_FW_VERSION(ipmi_msg *msg)
{
    CHECK_NULL_ARG(msg);

    uint8_t target = msg->data[0];

    switch (target) {
    case FW_BIC:
        msg->data[0] = BIC_FW_YEAR_MSB;
        msg->data[1] = BIC_FW_YEAR_LSB;
        msg->data[2] = BIC_FW_WEEK;
        msg->data[3] = BIC_FW_VER;
        msg->data_len = 4;
        break;
    case FW_CPLD:
        // 讀取 CPLD 版本
        get_cpld_version(&msg->data[0]);
        msg->data_len = 4;
        break;
    // ... 其他目標
    default:
        msg->completion_code = CC_INVALID_DATA_FIELD;
        return;
    }

    msg->completion_code = CC_SUCCESS;
}
```

---

## 平台 IPMI 命令

### plat_ipmi.c

```c
// meta-facebook/yv4-sd/src/ipmi/plat_ipmi.c

// 平台特定命令處理
void OEM_1S_GET_BOARD_ID(ipmi_msg *msg)
{
    msg->data[0] = BOARD_ID;
    msg->data[1] = get_slot_id();
    msg->data_len = 2;
    msg->completion_code = CC_SUCCESS;
}

void OEM_1S_GET_CARD_TYPE(ipmi_msg *msg)
{
    uint8_t card_type = gpio_get(CARD_TYPE_EXP);
    msg->data[0] = card_type;
    msg->data_len = 1;
    msg->completion_code = CC_SUCCESS;
}

void OEM_1S_GET_RETIMER_TYPE(ipmi_msg *msg)
{
    msg->data[0] = get_retimer_type();
    msg->data_len = 1;
    msg->completion_code = CC_SUCCESS;
}
```

---

## 完成代碼

```c
// ipmi.h
#define CC_SUCCESS              0x00
#define CC_NODE_BUSY            0xC0
#define CC_INVALID_CMD          0xC1
#define CC_INVALID_LUN          0xC2
#define CC_TIMEOUT              0xC3
#define CC_OUT_OF_SPACE         0xC4
#define CC_INVALID_DATA_FIELD   0xCC
#define CC_SENSOR_NOT_PRESENT   0xCB
#define CC_INVALID_LENGTH       0xC7
#define CC_NOT_SUPP_IN_CUR_STATE 0xD5
#define CC_UNSPECIFIED_ERROR    0xFF
```

---

## SEL 事件

### 新增 SEL

```c
// ipmi.c
bool common_add_sel_evt_record(common_addsel_msg_t *sel_msg)
{
    ipmi_msg msg = { 0 };
    
    msg.netfn = NETFN_STORAGE_REQ;
    msg.cmd = CMD_STORAGE_ADD_SEL;
    
    // 填充 SEL 記錄
    msg.data[0] = 0x02;  // System Event Record
    msg.data[2] = 0x04;  // Event Message Rev
    memcpy(&msg.data[3], &sel_msg->sensor_type, ...);
    
    msg.InF_target = BMC_IPMB;
    
    // 透過 IPMB 發送
    ipmb_send_request(&msg, BMC_IPMB_IDX);
    
    return true;
}
```

### SEL 事件結構

```c
typedef struct {
    uint8_t sensor_type;
    uint8_t sensor_num;
    uint8_t event_type;
    uint8_t event_data1;
    uint8_t event_data2;
    uint8_t event_data3;
} common_addsel_msg_t;
```

---

## 訊息佇列

### IPMI 訊息佇列

```c
// ipmi.c
K_MSGQ_DEFINE(ipmi_msgq, sizeof(ipmi_msg_cfg), IPMI_BUF_LEN, 4);

// 發送訊息到佇列
ipmb_error notify_ipmi_client(ipmi_msg_cfg *msg_cfg)
{
    if (k_msgq_put(&ipmi_msgq, msg_cfg, K_NO_WAIT) != 0) {
        return IPMB_ERROR_FAILURE;
    }
    return IPMB_ERROR_SUCCESS;
}

// 處理器線程
void IPMI_handler(void *arug0, void *arug1, void *arug2)
{
    ipmi_msg_cfg msg_cfg;
    
    while (1) {
        if (k_msgq_get(&ipmi_msgq, &msg_cfg, K_FOREVER) == 0) {
            ipmi_cmd_handle(&msg_cfg, NULL, NULL);
        }
    }
}
```

---

## 相關文件

- [IPMB](IPMB.md) - IPMB 通訊
- [KCS](KCS.md) - KCS 介面
- [OEMCommands](OEMCommands.md) - OEM 命令詳解
- [PLDMOverview](PLDMOverview.md) - PLDM 橋接

---

*返回 [Home](Home.md)*
