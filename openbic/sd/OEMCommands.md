# OEMCommands（OEM 命令實作）

本文說明 OpenBIC 的 OEM IPMI 與 PLDM 命令實作。

---

## OEM 命令概述

OpenBIC 支援多種 OEM 命令擴展：

| 協議 | NetFn/Type | 說明 |
|------|------------|------|
| **IPMI OEM** | 0x30/0x31 | 通用 OEM |
| **IPMI OEM_1S** | 0x38/0x39 | Meta/Facebook OEM |
| **PLDM OEM** | 0x3F | PLDM OEM 類型 |

---

## IPMI OEM_1S 命令

### 命令列表

| 命令 | 代碼 | 說明 |
|------|------|------|
| MSG_OUT | 0x01 | 訊息橋接 |
| GET_GPIO | 0x02 | 取得 GPIO 狀態 |
| SET_GPIO | 0x03 | 設定 GPIO |
| FW_UPDATE | 0x09 | 韌體更新 |
| GET_BIC_FW_INFO | 0x0A | 取得 BIC 韌體資訊 |
| GET_FW_VERSION | 0x0B | 取得韌體版本 |
| GET_POST_CODE | 0x12 | 取得 POST Code |
| SENSOR_POLL_EN | 0x14 | 感測器輪詢控制 |
| ACCURACY_SENSOR_READING | 0x1E | 精確感測器讀取 |
| GET_SET_GPIO | 0x41 | GPIO 讀寫 |
| GET_SET_M2 | 0x42 | M.2 操作 |
| CONTROL_SENSOR_POLLING | 0x53 | 感測器輪詢設定 |
| GET_BIC_STATUS | 0x58 | 取得 BIC 狀態 |
| RESET_BIC | 0x5A | 重啟 BIC |

---

## OEM_1S 命令實作

### MSG_OUT（訊息橋接）

```c
// oem_1s_handler.c
__weak OEM_1S_MSG_OUT(ipmi_msg *msg)
{
    CHECK_NULL_ARG(msg);
    
    if (msg->data_len < 2) {
        msg->completion_code = CC_INVALID_LENGTH;
        return;
    }
    
    uint8_t target_if = msg->data[0];
    uint8_t target_addr = msg->data[1];
    
    // 建立橋接訊息
    ipmi_msg bridge_msg = { 0 };
    bridge_msg.netfn = msg->data[2] >> 2;
    bridge_msg.cmd = msg->data[3];
    bridge_msg.data_len = msg->data_len - 4;
    memcpy(bridge_msg.data, &msg->data[4], bridge_msg.data_len);
    
    bridge_msg.InF_target = target_if;
    bridge_msg.target_addr = target_addr;
    
    // 發送到目標
    ipmb_send_request(&bridge_msg, target_if);
    
    msg->completion_code = CC_SUCCESS;
}
```

### GET_GPIO

```c
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
    
    memset(msg->data, 0, gpio_data_size);
    
    for (int i = 0; i < gpio_cnt; i++) {
        if (gpio_get(i) == GPIO_HIGH) {
            msg->data[i / 8] |= (1 << (i % 8));
        }
    }

    msg->data_len = gpio_data_size;
    msg->completion_code = CC_SUCCESS;
}
```

### GET_SET_GPIO

```c
__weak OEM_1S_GET_SET_GPIO(ipmi_msg *msg)
{
    CHECK_NULL_ARG(msg);

    if (msg->data_len < 2) {
        msg->completion_code = CC_INVALID_LENGTH;
        return;
    }

    uint8_t gpio_num = msg->data[0];
    uint8_t action = msg->data[1];  // 0=GET, 1=SET
    
    switch (action) {
    case 0:  // GET
        msg->data[0] = gpio_get(gpio_num);
        msg->data_len = 1;
        break;
        
    case 1:  // SET
        if (msg->data_len < 3) {
            msg->completion_code = CC_INVALID_LENGTH;
            return;
        }
        gpio_set(gpio_num, msg->data[2]);
        msg->data_len = 0;
        break;
        
    default:
        msg->completion_code = CC_INVALID_DATA_FIELD;
        return;
    }

    msg->completion_code = CC_SUCCESS;
}
```

### GET_FW_VERSION

```c
__weak OEM_1S_GET_FW_VERSION(ipmi_msg *msg)
{
    CHECK_NULL_ARG(msg);

    if (msg->data_len != 1) {
        msg->completion_code = CC_INVALID_LENGTH;
        return;
    }

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
        I2C_MSG i2c_msg = { 0 };
        i2c_msg.bus = CPLD_IO_I2C_BUS;
        i2c_msg.target_addr = CPLD_IO_I2C_ADDR;
        i2c_msg.tx_len = 1;
        i2c_msg.rx_len = 4;
        i2c_msg.data[0] = CPLD_REG_VERSION;
        
        if (i2c_master_read(&i2c_msg, 3) == 0) {
            memcpy(msg->data, i2c_msg.data, 4);
            msg->data_len = 4;
        } else {
            msg->completion_code = CC_UNSPECIFIED_ERROR;
            return;
        }
        break;

    case FW_VR:
        get_vr_version(&msg->data[0]);
        msg->data_len = 8;
        break;

    default:
        msg->completion_code = CC_INVALID_DATA_FIELD;
        return;
    }

    msg->completion_code = CC_SUCCESS;
}
```

### ACCURACY_SENSOR_READING

```c
__weak OEM_1S_ACCURACY_SENSOR_READING(ipmi_msg *msg)
{
    CHECK_NULL_ARG(msg);

    if (msg->data_len != 1) {
        msg->completion_code = CC_INVALID_LENGTH;
        return;
    }

    uint8_t sensor_num = msg->data[0];
    
    // 查找感測器
    sensor_cfg *cfg = NULL;
    for (int i = 0; i < sensor_config_count; i++) {
        if (sensor_config[i].num == sensor_num) {
            cfg = &sensor_config[i];
            break;
        }
    }

    if (!cfg) {
        msg->completion_code = CC_SENSOR_NOT_PRESENT;
        return;
    }

    // 讀取感測器 (4-byte 精度)
    int32_t reading = cfg->cache;
    uint8_t status = cfg->cache_status;

    msg->data[0] = status;
    memcpy(&msg->data[1], &reading, 4);
    msg->data_len = _4BYTE_ACCURACY_SENSOR_READING_RES_LEN;
    msg->completion_code = CC_SUCCESS;
}
```

---

## 平台特定 OEM 命令

### plat_ipmi.c

```c
// meta-facebook/yv4-sd/src/ipmi/plat_ipmi.c

void OEM_1S_GET_BOARD_ID(ipmi_msg *msg)
{
    CHECK_NULL_ARG(msg);

    msg->data[0] = BOARD_ID;
    msg->data[1] = get_slot_id();
    msg->data_len = 2;
    msg->completion_code = CC_SUCCESS;
}

void OEM_1S_GET_CARD_TYPE(ipmi_msg *msg)
{
    CHECK_NULL_ARG(msg);

    uint8_t card_type = gpio_get(CARD_TYPE_EXP);
    msg->data[0] = card_type;
    msg->data_len = 1;
    msg->completion_code = CC_SUCCESS;
}

void OEM_1S_GET_RETIMER_TYPE(ipmi_msg *msg)
{
    CHECK_NULL_ARG(msg);

    msg->data[0] = get_retimer_type();
    msg->data_len = 1;
    msg->completion_code = CC_SUCCESS;
}

void OEM_1S_GET_VR_TYPE(ipmi_msg *msg)
{
    CHECK_NULL_ARG(msg);

    msg->data[0] = get_vr_type();
    msg->data_len = 1;
    msg->completion_code = CC_SUCCESS;
}
```

---

## PLDM OEM 命令

### 命令列表

| 命令 | 代碼 | 說明 |
|------|------|------|
| IPMI_BRIDGE | 0x01 | IPMI 橋接 |
| WRITE_FILE_IO | 0x02 | 寫入檔案 |
| READ_FILE_IO | 0x03 | 讀取檔案 |
| ECHO | 0x80 | 回聲測試 |

### IPMI Bridge

```c
// pldm_oem.c
uint8_t pldm_oem_ipmi_bridge(pldm_msg *msg)
{
    struct pldm_oem_ipmi_bridge_req *req = 
        (struct pldm_oem_ipmi_bridge_req *)msg->buf;
    
    // 驗證 IANA
    if (!check_iana(req->iana)) {
        return PLDM_OEM_INVALID_IANA;
    }
    
    // 解析 IPMI 訊息
    ipmi_msg ipmi = { 0 };
    ipmi.netfn = req->netfn_lun >> 2;
    ipmi.cmd = req->ipmi_cmd;
    ipmi.data_len = msg->len - sizeof(*req);
    memcpy(ipmi.data, req->data, ipmi.data_len);
    
    // 處理 IPMI 命令
    ipmi_cmd_handle(&ipmi, NULL, NULL);
    
    // 建立回應
    struct pldm_oem_ipmi_bridge_resp *resp = ...;
    resp->completion_code = ipmi.completion_code;
    memcpy(resp->data, ipmi.data, ipmi.data_len);
    
    return PLDM_SUCCESS;
}
```

### Read/Write File IO

```c
// plat_pldm.c

// 讀取 HTTP Boot 屬性
uint8_t plat_pldm_get_http_boot_attr(uint8_t length, uint8_t *httpBootattr)
{
    pldm_msg pmsg = { 0 };
    pmsg.ext_params.ep = MCTP_EID_BMC;
    pmsg.hdr.rq = PLDM_REQUEST;
    pmsg.hdr.pldm_type = PLDM_TYPE_OEM;
    pmsg.hdr.cmd = PLDM_OEM_READ_FILE_IO;

    struct pldm_oem_read_file_io_attr_req req = {
        .iana = { ... },
        .cmd_code = HTTP_BOOT,
        .read_option = READ_FILE_ATTR,
    };

    pmsg.buf = (uint8_t *)&req;
    pmsg.len = sizeof(req);

    uint16_t resp_len = mctp_pldm_read(
        find_mctp_by_bus(bmc_bus), 
        &pmsg, rbuf, BMC_PLDM_DATA_MAXIMUM);

    if (resp_len > 0) {
        memcpy(httpBootattr, rbuf, length);
        return PLDM_SUCCESS;
    }
    
    return PLDM_ERROR;
}

// 設定 Boot Order
uint8_t plat_pldm_set_boot_order(uint8_t *bootOrderData)
{
    pldm_msg msg = { 0 };
    msg.ext_params.ep = MCTP_EID_BMC;
    msg.hdr.rq = PLDM_REQUEST;
    msg.hdr.pldm_type = PLDM_TYPE_OEM;
    msg.hdr.cmd = PLDM_OEM_WRITE_FILE_IO;

    struct pldm_oem_write_file_io_req req = {
        .iana = { ... },
        .cmd_code = BOOT_ORDER,
        .data_length = BOOT_ORDER_LENGTH,
    };
    memcpy(req.messages, bootOrderData, BOOT_ORDER_LENGTH);

    msg.buf = (uint8_t *)&req;
    msg.len = sizeof(req);

    mctp_pldm_read(find_mctp_by_bus(bmc_bus), 
                    &msg, rbuf, resp_len);

    return PLDM_SUCCESS;
}
```

---

## 新增 OEM 命令

### 步驟

1. **定義命令代碼**
2. **實作處理函數**
3. **註冊到處理器表**

### 範例

```c
// 1. 定義命令代碼 (oem_1s_handler.h)
#define CMD_OEM_1S_MY_COMMAND 0xF0

// 2. 實作處理函數 (plat_ipmi.c)
void OEM_1S_MY_COMMAND(ipmi_msg *msg)
{
    CHECK_NULL_ARG(msg);
    
    // 處理邏輯
    // ...
    
    msg->completion_code = CC_SUCCESS;
}

// 3. 註冊 (若使用弱符號覆蓋，無需額外註冊)
```

---

## 相關文件

- [IPMIOverview](IPMIOverview.md) - IPMI 概述
- [PLDMOverview](PLDMOverview.md) - PLDM 概述
- [FirmwareUpdate](FirmwareUpdate.md) - 韌體更新命令

---

*返回 [Home](Home.md)*
