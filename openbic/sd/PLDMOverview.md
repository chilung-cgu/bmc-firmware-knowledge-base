# PLDMOverview（PLDM 平台管理協議）

本文說明 OpenBIC 實作的 Platform Level Data Model (PLDM) 平台管理協議。

---

## PLDM 概述

**PLDM** (Platform Level Data Model) 是 DMTF 定義的標準協議，用於平台監控與控制。

### 規範文件

| 規範 | 說明 |
|------|------|
| DSP0240 | PLDM Base Specification |
| DSP0248 | PLDM for Platform Monitoring and Control |
| DSP0267 | PLDM for Firmware Update |
| DSP0245 | PLDM for FRU Data |
| DSP0246 | PLDM for SMBIOS |

### PLDM 特性

- **結構化資料**：定義標準資料格式
- **多功能**：感測器、Effecter、事件、韌體更新
- **可擴展**：OEM 類型支援
- **高效**：二進制協議，低開銷

---

## PLDM 類型

```c
// pldm.h
typedef enum {
    PLDM_TYPE_BASE      = 0x00,  // 基礎命令
    PLDM_TYPE_SMBIOS    = 0x01,  // SMBIOS
    PLDM_TYPE_PLAT_MON  = 0x02,  // 平台監控與控制
    PLDM_TYPE_BIOS_CTRL = 0x03,  // BIOS 控制
    PLDM_TYPE_FRU       = 0x04,  // FRU 資料
    PLDM_TYPE_FW_UPDATE = 0x05,  // 韌體更新
    PLDM_TYPE_RDE       = 0x06,  // Redfish Device Enablement
    PLDM_TYPE_OEM       = 0x3F,  // OEM 定義
} PLDM_TYPE;
```

---

## PLDM 訊息格式

### 訊息標頭

```
┌───────────────────────────────────────────────────────────────┐
│   Byte 0    │   Byte 1    │   Byte 2    │   Byte 3...  │
├─────────────┼─────────────┼─────────────┼──────────────┤
│ Instance ID │   Header    │   Command   │   Payload    │
│ + Rq/D/Rsvd │   Ver/Type  │    Code     │              │
└───────────────────────────────────────────────────────────────┘
```

| 欄位 | 位元 | 說明 |
|------|------|------|
| Rq | 1 | 請求/回應 |
| D | 1 | Datagram |
| Instance ID | 5 | 實例 ID |
| Header Ver | 2 | 標頭版本 |
| PLDM Type | 6 | PLDM 類型 |
| Command | 8 | 命令代碼 |

### 訊息結構

```c
// pldm.h
typedef struct __attribute__((packed)) {
    uint8_t inst_id : 5;
    uint8_t rsvd : 1;
    uint8_t d : 1;
    uint8_t rq : 1;
    uint8_t pldm_type : 6;
    uint8_t ver : 2;
    uint8_t cmd;
} pldm_hdr;

typedef struct {
    pldm_hdr hdr;
    uint8_t *buf;
    uint16_t len;
    mctp_ext_params ext_params;
    // 回調函數
    void (*recv_resp_cb_fn)(void *, uint8_t *, uint16_t);
    void (*timeout_cb_fn)(void *);
    void *timeout_cb_fn_args;
} pldm_msg;
```

---

## OpenBIC PLDM 實作

### 支援的 PLDM 命令

#### Base (Type 0x00)

| 命令 | 代碼 | 說明 |
|------|------|------|
| GetTID | 0x02 | 取得 Terminus ID |
| GetPLDMVersion | 0x03 | 取得版本 |
| GetPLDMTypes | 0x04 | 取得支援類型 |
| GetPLDMCommands | 0x05 | 取得支援命令 |

#### Platform Monitoring (Type 0x02)

| 命令 | 代碼 | 說明 |
|------|------|------|
| GetSensorReading | 0x11 | 讀取感測器 |
| SetNumericEffecterValue | 0x31 | 設定數值 Effecter |
| GetNumericEffecterValue | 0x32 | 取得數值 Effecter |
| SetStateEffecterStates | 0x39 | 設定狀態 Effecter |
| GetPDR | 0x51 | 取得 PDR |

#### Firmware Update (Type 0x05)

| 命令 | 代碼 | 說明 |
|------|------|------|
| QueryDeviceIdentifiers | 0x01 | 查詢裝置識別碼 |
| GetFirmwareParameters | 0x02 | 取得韌體參數 |
| RequestUpdate | 0x10 | 請求更新 |
| PassComponentTable | 0x13 | 傳送元件表 |
| UpdateComponent | 0x14 | 更新元件 |

#### OEM (Type 0x3F)

| 命令 | 代碼 | 說明 |
|------|------|------|
| IPMI Bridge | 0x01 | IPMI 橋接 |
| Write File IO | 0x02 | 寫入檔案 |
| Read File IO | 0x03 | 讀取檔案 |
| Echo | 0x80 | 回聲測試 |

---

## PLDM 命令處理

### 命令處理器架構

```c
// pldm.c
uint8_t mctp_pldm_cmd_handler(void *mctp_p, uint8_t *buf, uint32_t len, 
                               mctp_ext_params ext_params)
{
    pldm_hdr *hdr = (pldm_hdr *)buf;
    
    // 處理回應訊息
    if (hdr->rq == PLDM_RESPONSE) {
        return pldm_resp_msg_process(mctp_p, buf, len, ext_params);
    }
    
    // 查找處理器
    pldm_cmd_handler handler = pldm_handler_query(hdr->pldm_type, hdr->cmd);
    
    if (handler) {
        // 執行命令處理
        pldm_msg msg = { ... };
        handler(&msg);
        
        // 發送回應
        mctp_pldm_send_msg(mctp_p, &msg);
    }
    
    return PLDM_SUCCESS;
}
```

### 處理器查詢

```c
struct _pldm_handler_query_entry {
    PLDM_TYPE type;
    uint8_t (*handler_query)(uint8_t, void **);
};

static struct _pldm_handler_query_entry pldm_handler_query_tbl[] = {
    { PLDM_TYPE_BASE, pldm_base_handler_query },
    { PLDM_TYPE_PLAT_MON, pldm_monitor_handler_query },
    { PLDM_TYPE_FW_UPDATE, pldm_fw_update_handler_query },
    { PLDM_TYPE_OEM, pldm_oem_handler_query },
};
```

---

## yv4-sd PLDM 配置

### TID 設定

```c
// plat_pldm.c
uint8_t plat_pldm_get_tid()
{
    // 使用 EID 作為 TID
    return plat_get_eid();
}
```

### OEM 命令實作

```c
// plat_pldm.c

// 取得 HTTP Boot 屬性
uint8_t plat_pldm_get_http_boot_attr(uint8_t length, uint8_t *httpBootattr)
{
    pldm_msg pmsg = { 0 };
    pmsg.ext_params.ep = MCTP_EID_BMC;
    pmsg.hdr.rq = PLDM_REQUEST;
    pmsg.hdr.pldm_type = PLDM_TYPE_OEM;
    pmsg.hdr.cmd = PLDM_OEM_READ_FILE_IO;

    struct pldm_oem_read_file_io_attr_req *ptr = malloc(...);
    ptr->cmd_code = HTTP_BOOT;
    ptr->read_option = READ_FILE_ATTR;
    
    pmsg.buf = (uint8_t *)ptr;
    pmsg.len = sizeof(...);

    uint16_t resp_len = BMC_PLDM_DATA_MAXIMUM;
    resp_len = mctp_pldm_read(find_mctp_by_bus(bmc_bus), &pmsg, rbuf, resp_len);
    
    // 處理回應...
    return PLDM_SUCCESS;
}

// 設定 Boot Order
uint8_t plat_pldm_set_boot_order(uint8_t *bootOrderData)
{
    pldm_msg msg = { 0 };
    msg.hdr.rq = PLDM_REQUEST;
    msg.hdr.pldm_type = PLDM_TYPE_OEM;
    msg.hdr.cmd = PLDM_OEM_WRITE_FILE_IO;

    struct pldm_oem_write_file_io_req *ptr = malloc(...);
    ptr->cmd_code = BOOT_ORDER;
    ptr->data_length = BOOT_ORDER_LENGTH;
    memcpy(ptr->messages, bootOrderData, BOOT_ORDER_LENGTH);

    // 發送請求並等待回應
    mctp_pldm_read(find_mctp_by_bus(bmc_bus), &msg, rbuf, resp_len);
    
    return PLDM_SUCCESS;
}
```

---

## PLDM 訊息收發

### 發送請求

```c
// pldm.c
uint8_t mctp_pldm_send_msg(void *mctp_p, pldm_msg *msg)
{
    mctp *mctp_inst = (mctp *)mctp_p;
    
    // 註冊 Instance ID
    uint8_t inst_id;
    register_instid(mctp_p, &inst_id);
    msg->hdr.inst_id = inst_id;
    
    // 組裝訊息
    uint16_t len = sizeof(pldm_hdr) + msg->len + 1;  // +1 for msg type
    uint8_t *buf = malloc(len);
    
    buf[0] = MCTP_MSG_TYPE_PLDM;
    memcpy(buf + 1, &msg->hdr, sizeof(pldm_hdr));
    memcpy(buf + 1 + sizeof(pldm_hdr), msg->buf, msg->len);
    
    // 透過 MCTP 發送
    mctp_send_msg(mctp_inst, buf, len, msg->ext_params);
    
    free(buf);
    return PLDM_SUCCESS;
}
```

### 同步讀取

```c
// pldm.c
uint16_t mctp_pldm_read(void *mctp_p, pldm_msg *msg, uint8_t *rbuf, uint16_t rbuf_len)
{
    // 發送請求
    mctp_pldm_send_msg(mctp_p, msg);
    
    // 等待回應
    k_event_wait(&resp_event, PLDM_READ_EVENT_SUCCESS, ...);
    
    // 複製回應資料
    memcpy(rbuf, resp_buf, resp_len);
    
    return resp_len;
}
```

---

## PDR (Platform Descriptor Record)

### PDR 概述

PDR 描述平台資源（感測器、Effecter 等）的結構化記錄。

### 常見 PDR 類型

| 類型 | 代碼 | 說明 |
|------|------|------|
| Terminus Locator | 1 | Terminus 位置 |
| Numeric Sensor | 2 | 數值感測器 |
| State Sensor | 4 | 狀態感測器 |
| Numeric Effecter | 9 | 數值 Effecter |
| State Effecter | 11 | 狀態 Effecter |
| Entity Association | 15 | 實體關聯 |
| FRU Record Set | 20 | FRU 記錄集 |

---

## IPMI over PLDM

### 橋接 IPMI 訊息

```c
// pldm.c
int pldm_send_ipmi_response(uint8_t interface, ipmi_msg *msg)
{
    pldm_msg pmsg = { 0 };
    pmsg.hdr.pldm_type = PLDM_TYPE_OEM;
    pmsg.hdr.cmd = PLDM_OEM_IPMI_BRIDGE;
    
    struct pldm_ipmi_req req = { 0 };
    set_iana(req.iana, sizeof(req.iana));
    req.netfn_lun = msg->netfn << 2;
    req.ipmi_cmd = msg->cmd;
    memcpy(req.data, msg->data, msg->data_len);
    
    pmsg.buf = (uint8_t *)&req;
    pmsg.len = sizeof(req);
    
    mctp_pldm_send_msg(mctp_inst, &pmsg);
    
    return 0;
}
```

### SEL 記錄

```c
// plat_mctp.c
bool mctp_add_sel_to_ipmi(common_addsel_msg_t *sel_msg)
{
    pldm_msg msg = { 0 };
    struct mctp_to_ipmi_sel_req req = { 0 };
    
    msg.hdr.pldm_type = PLDM_TYPE_OEM;
    msg.hdr.cmd = PLDM_OEM_IPMI_BRIDGE;
    
    req.header.netfn_lun = (NETFN_STORAGE_REQ << 2);
    req.header.ipmi_cmd = CMD_STORAGE_ADD_SEL;
    req.req_data.event.record_type = 0x02;  // System Event
    
    memcpy(&req.req_data.event.sensor_type, &sel_msg->sensor_type, ...);
    
    msg.buf = (uint8_t *)&req;
    msg.len = sizeof(req);
    
    mctp_pldm_read(find_mctp_by_bus(bmc_bus), &msg, rbuf, resp_len);
    
    return true;
}
```

---

## 相關文件

- [MCTPOverview](MCTPOverview.md) - MCTP 傳輸協議
- [PLDMSensor](PLDMSensor.md) - PLDM 感測器
- [PLDMMonitor](PLDMMonitor.md) - PLDM 監控
- [FirmwareUpdate](FirmwareUpdate.md) - 韌體更新

---

*返回 [Home](Home.md)*
