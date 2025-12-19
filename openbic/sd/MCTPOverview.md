# MCTPOverview（MCTP 傳輸協議）

本文說明 OpenBIC 實作的 Management Component Transport Protocol (MCTP) 傳輸協議。

---

## MCTP 概述

**MCTP** (Management Component Transport Protocol) 是 DMTF 定義的標準傳輸協議，用於管理元件之間的通訊。

### 規範文件

| 規範 | 說明 |
|------|------|
| DSP0236 | MCTP Base Specification |
| DSP0237 | MCTP SMBus/I2C Transport Binding |
| DSP0238 | MCTP PCIe VDM Transport Binding |
| DSP0239 | MCTP USB Transport Binding |

### MCTP 特性

- **傳輸無關**：支援多種底層傳輸（SMBus、I3C、PCIe）
- **訊息分段**：大訊息自動分段與重組
- **路由**：支援端點定址與橋接
- **多協議**：承載 PLDM、SPDM 等上層協議

---

## 架構

```
┌─────────────────────────────────────────────────────────────────┐
│                      Upper Layer Protocols                      │
│         ┌──────────┐  ┌──────────┐  ┌──────────┐              │
│         │   PLDM   │  │   SPDM   │  │   MCTP   │              │
│         │          │  │          │  │   Ctrl   │              │
│         └────┬─────┘  └────┬─────┘  └────┬─────┘              │
├──────────────┼─────────────┼─────────────┼──────────────────────┤
│              │             │             │                      │
│              └─────────────┴─────────────┘                      │
│                          │                                       │
│              ┌───────────▼───────────┐                          │
│              │      MCTP Core        │                          │
│              │  ┌─────────────────┐  │                          │
│              │  │ Message Assembly│  │                          │
│              │  │ Routing         │  │                          │
│              │  │ Endpoint Mgmt   │  │                          │
│              │  └─────────────────┘  │                          │
│              └───────────┬───────────┘                          │
│                          │                                       │
├──────────────────────────┼──────────────────────────────────────┤
│                          │                                       │
│        ┌─────────────────┼─────────────────┐                    │
│        │                 │                 │                    │
│    ┌───▼────┐       ┌────▼───┐       ┌────▼───┐                │
│    │ SMBus  │       │  I3C   │       │ PCIe   │                │
│    │Binding │       │Binding │       │Binding │                │
│    └────────┘       └────────┘       └────────┘                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## MCTP 訊息格式

### 傳輸層標頭

```
┌───────────────────────────────────────────────────────────────┐
│ Byte 0  │ Byte 1  │ Byte 2  │ Byte 3  │ Byte 4... │
├─────────┼─────────┼─────────┼─────────┼───────────┤
│ HDR Ver │  Dest   │  Src    │ Msg Tag │  Payload  │
│ + Rsvd  │  EID    │  EID    │ + Flags │           │
└───────────────────────────────────────────────────────────────┘
```

| 欄位 | 位元 | 說明 |
|------|------|------|
| Header Version | 4 | 固定為 0x01 |
| Destination EID | 8 | 目標端點 ID |
| Source EID | 8 | 來源端點 ID |
| SOM | 1 | Start of Message |
| EOM | 1 | End of Message |
| Pkt Seq | 2 | 封包序號 |
| TO | 1 | Tag Owner |
| Msg Tag | 3 | 訊息標籤 |

### 訊息類型

```c
// 訊息類型定義
#define MCTP_MSG_TYPE_CTRL      0x00  // MCTP 控制訊息
#define MCTP_MSG_TYPE_PLDM      0x01  // PLDM
#define MCTP_MSG_TYPE_NCSI      0x02  // NC-SI
#define MCTP_MSG_TYPE_ETHERNET  0x03  // Ethernet
#define MCTP_MSG_TYPE_NVME      0x04  // NVMe-MI
#define MCTP_MSG_TYPE_SPDM      0x05  // SPDM
#define MCTP_MSG_TYPE_VENDOR    0x7E  // Vendor Defined
#define MCTP_MSG_TYPE_VENDOR2   0x7F  // Vendor Defined
```

---

## OpenBIC MCTP 實作

### 核心 API

```c
// mctp.h

// 初始化 MCTP 實例
mctp *mctp_init(void);

// 設定介質配置
uint8_t mctp_set_medium_configure(mctp *mctp_inst, 
                                   MCTP_MEDIUM_TYPE type,
                                   mctp_medium_conf conf);

// 註冊端點解析函數
void mctp_reg_endpoint_resolve_func(mctp *mctp_inst, 
                                     mctp_fn_endpoint_resolve fn);

// 註冊訊息接收函數
void mctp_reg_msg_rx_func(mctp *mctp_inst, mctp_fn_msg_rx fn);

// 啟動 MCTP 服務
uint8_t mctp_start(mctp *mctp_inst);
```

### 介質類型

```c
typedef enum {
    MCTP_MEDIUM_TYPE_UNKNOWN = 0,
    MCTP_MEDIUM_TYPE_SMBUS,
    MCTP_MEDIUM_TYPE_TARGET_I3C,
    MCTP_MEDIUM_TYPE_CONTROLLER_I3C,
} MCTP_MEDIUM_TYPE;
```

### 配置結構

```c
typedef union {
    struct {
        uint8_t bus;
        uint8_t addr;
    } smbus_conf;
    
    struct {
        uint8_t bus;
        uint8_t addr;
    } i3c_conf;
} mctp_medium_conf;

typedef struct {
    mctp_medium_conf conf;
    MCTP_MEDIUM_TYPE medium_type;
    mctp *mctp_inst;
} mctp_port;
```

---

## yv4-sd MCTP 配置

### MCTP Port 配置

```c
// plat_mctp.c
static mctp_port plat_mctp_port[] = {
    // BMC via I2C (IPMB)
    { .conf.smbus_conf.addr = I2C_ADDR_BIC,
      .conf.smbus_conf.bus = I2C_BUS_BMC,
      .medium_type = MCTP_MEDIUM_TYPE_SMBUS },
    
    // BMC via I3C (Target 模式)
    { .conf.i3c_conf.addr = I3C_STATIC_ADDR_BMC,
      .conf.i3c_conf.bus = I3C_BUS_BMC,
      .medium_type = MCTP_MEDIUM_TYPE_TARGET_I3C },
    
    // FF BIC via I3C Hub (Controller 模式)
    { .conf.i3c_conf.addr = I3C_STATIC_ADDR_FF_BIC,
      .conf.i3c_conf.bus = I3C_BUS_HUB,
      .medium_type = MCTP_MEDIUM_TYPE_CONTROLLER_I3C },
    
    // WF BIC via I3C Hub (Controller 模式)
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
    { MCTP_EID_FF_CXL, I3C_BUS_HUB, I3C_STATIC_ADDR_FF_BIC, .set_endpoint = false },
    { MCTP_EID_WF_CXL1, I3C_BUS_HUB, I3C_STATIC_ADDR_WF_BIC, .set_endpoint = false },
    { MCTP_EID_WF_CXL2, I3C_BUS_HUB, I3C_STATIC_ADDR_WF_BIC, .set_endpoint = false },
};
```

---

## Endpoint ID (EID)

### EID 配置表

| 平台 | 裝置 | EID |
|------|------|-----|
| All | BIC (保留) | 0x00 |
| All | BMC (預設) | 0x08 |
| Cascade Creek | BIC | 0x0A |
| Crafter Lake | BIC | 0x20 |
| Y35 Baseboard | BIC | 0x21 |
| Great Lake | BIC | 0x22 |
| Halfdome | BIC | 0x23 |

### 動態 EID 分配

yv4-sd 根據 Slot 位置動態分配 EID：

```c
// plat_mctp.c
void plat_set_eid_by_slot()
{
    uint8_t slot_eid = get_slot_eid();
    plat_eid = slot_eid;
}

void set_routing_table_eid()
{
    // 跳過 BMC，從 Slot EID 開始分配
    for (uint8_t i = 2, j = 1; i < ARRAY_SIZE(plat_mctp_route_tbl); i++, j++) {
        mctp_route_entry *p = plat_mctp_route_tbl + i;
        p->endpoint = plat_eid + j;
    }
}
```

---

## MCTP 初始化

```c
// plat_mctp.c
void plat_mctp_init(void)
{
    plat_set_eid_by_slot();
    set_routing_table_eid();
    
    for (uint8_t i = 0; i < ARRAY_SIZE(plat_mctp_port); i++) {
        mctp_port *p = plat_mctp_port + i;
        
        // 初始化 MCTP 實例
        p->mctp_inst = mctp_init();
        if (!p->mctp_inst) {
            LOG_ERR("mctp_init failed!!");
            continue;
        }

        // 設定介質配置
        uint8_t rc = mctp_set_medium_configure(p->mctp_inst, 
                                                p->medium_type, 
                                                p->conf);
        if (rc != MCTP_SUCCESS) {
            LOG_ERR("mctp set medium configure failed");
        }

        // 註冊端點解析函數
        mctp_reg_endpoint_resolve_func(p->mctp_inst, get_mctp_route_info);

        // 註冊訊息接收函數
        mctp_reg_msg_rx_func(p->mctp_inst, mctp_msg_recv);

        // 啟動服務
        mctp_start(p->mctp_inst);
    }
    
    // 延遲後設定下游裝置 EID
    k_timer_start(&send_cmd_timer, K_MSEC(3000), K_NO_WAIT);
}
```

---

## MCTP 訊息處理

### 訊息接收回調

```c
static uint8_t mctp_msg_recv(void *mctp_p, uint8_t *buf, uint32_t len, 
                              mctp_ext_params ext_params)
{
    CHECK_NULL_ARG_WITH_RETURN(mctp_p, MCTP_ERROR);
    CHECK_NULL_ARG_WITH_RETURN(buf, MCTP_ERROR);
    
    // 解析訊息類型
    uint8_t msg_type = (buf[0] & MCTP_MSG_TYPE_MASK) >> MCTP_MSG_TYPE_SHIFT;

    switch (msg_type) {
    case MCTP_MSG_TYPE_CTRL:
        LOG_DBG("type: mctp_ctrl");
        mctp_ctrl_cmd_handler(mctp_p, buf, len, ext_params);
        break;

    case MCTP_MSG_TYPE_PLDM:
        LOG_DBG("type: mctp_pldm");
        mctp_pldm_cmd_handler(mctp_p, buf, len, ext_params);
        break;

    default:
        LOG_WRN("Cannot find message receive function!!");
        return MCTP_ERROR;
    }

    return MCTP_SUCCESS;
}
```

---

## MCTP Control 命令

### 支援的命令

| 命令 | 代碼 | 說明 |
|------|------|------|
| Set Endpoint ID | 0x01 | 設定 EID |
| Get Endpoint ID | 0x02 | 取得 EID |
| Get Endpoint UUID | 0x03 | 取得 UUID |
| Get MCTP Version | 0x04 | 取得版本 |
| Get Message Type Support | 0x05 | 取得支援類型 |

### Set Endpoint ID

```c
static void set_dev_endpoint(void)
{
    for (uint8_t i = 0; i < ARRAY_SIZE(plat_mctp_route_tbl); i++) {
        mctp_route_entry *p = plat_mctp_route_tbl + i;
        if (!p->set_endpoint)
            continue;

        struct _set_eid_req req = { 0 };
        req.op = SET_EID_REQ_OP_SET_EID;
        req.eid = p->endpoint;

        mctp_ctrl_msg msg;
        memset(&msg, 0, sizeof(msg));
        msg.ext_params.type = plat_mctp_port[j].medium_type;
        msg.ext_params.i3c_ext_params.addr = p->addr;

        msg.hdr.cmd = MCTP_CTRL_CMD_SET_ENDPOINT_ID;
        msg.hdr.rq = 1;

        msg.cmd_data = (uint8_t *)&req;
        msg.cmd_data_len = sizeof(req);

        mctp_ctrl_send_msg(find_mctp_by_addr(p->addr), &msg);
    }
}
```

---

## 相關文件

- [PLDMOverview](PLDMOverview.md) - PLDM 協議
- [I2C_I3C](I2C_I3C.md) - 傳輸介面
- [Architecture](Architecture.md) - 系統架構

---

*返回 [Home](Home.md)*
