# IPMB（IPMB 通訊）

本文說明 OpenBIC 的 IPMB (Intelligent Platform Management Bus) 通訊機制。

---

## IPMB 概述

**IPMB** 是基於 I2C 的智慧平台管理匯流排，用於 BMC 與 BIC 之間的通訊。

### 特性

- **I2C 基礎**：使用 I2C 作為物理層
- **雙向通訊**：支援請求/回應模式
- **多節點**：支援多個 IPMB 節點
- **可靠傳輸**：包含校驗與重試機制

---

## IPMB 訊息格式

### 請求訊息

```
┌────────┬────────┬────────┬────────┬────────┬────────┬────────┬────────┐
│ rsSA   │ netFn  │ Hdr    │ rqSA   │ rqSeq  │ Cmd    │ Data   │ Chksum │
│        │ /rsLUN │ Chksum │        │ /rqLUN │        │        │        │
└────────┴────────┴────────┴────────┴────────┴────────┴────────┴────────┘
```

| 欄位 | 說明 |
|------|------|
| rsSA | 回應者 Slave Address |
| netFn/rsLUN | Network Function + 回應者 LUN |
| Hdr Chksum | 標頭校驗和 |
| rqSA | 請求者 Slave Address |
| rqSeq/rqLUN | 序列號 + 請求者 LUN |
| Cmd | 命令代碼 |
| Data | 資料內容 |
| Chksum | 資料校驗和 |

### 回應訊息

```
┌────────┬────────┬────────┬────────┬────────┬────────┬────────┬────────┬────────┐
│ rqSA   │ netFn  │ Hdr    │ rsSA   │ rqSeq  │ Cmd    │  CC    │ Data   │ Chksum │
│        │ /rqLUN │ Chksum │        │ /rsLUN │        │        │        │        │
└────────┴────────┴────────┴────────┴────────┴────────┴────────┴────────┴────────┘
```

---

## OpenBIC IPMB 實作

### 配置結構

```c
// ipmb.h
typedef struct {
    uint8_t bus;            // I2C bus
    uint8_t self_addr;      // 自身地址
    uint8_t target_addr;    // 目標地址
    bool enable;            // 啟用狀態
} IPMB_config;

// 平台配置
IPMB_config IPMB_config_table[] = {
    // BMC IPMB
    { .bus = I2C_BUS_BMC, 
      .self_addr = IPMB_ADDR_BIC, 
      .target_addr = IPMB_ADDR_BMC,
      .enable = true },
};
```

### 介面定義

```c
// plat_ipmb.h
#define I2C_BUS_BMC         2
#define IPMB_ADDR_BIC       0x20  // BIC I2C 地址
#define IPMB_ADDR_BMC       0x10  // BMC I2C 地址

enum {
    BMC_IPMB = 0,
    MAX_IPMB_IDX
};
```

---

## TX/RX 任務

### 發送任務

```c
// ipmb.c
void IPMB_TXTask(void *pvParameters, void *arvg0, void *arvg1)
{
    uint8_t index = (uintptr_t)pvParameters;
    ipmi_msg_cfg current_msg;
    
    while (1) {
        // 從發送佇列取得訊息
        if (k_msgq_get(&ipmb_txqueue[index], &current_msg, K_FOREVER) == 0) {
            
            // 編碼 IPMB 訊息
            uint8_t buffer[IPMB_MAX_LEN];
            ipmb_encode(buffer, &current_msg.buffer);
            
            // 透過 I2C 發送
            i2c_master_write_retry(
                dev_ipmb[IPMB_config_table[index].bus],
                buffer,
                current_msg.buffer.data_len + IPMB_HEADER_LEN,
                IPMB_config_table[index].target_addr,
                IPMB_RETRY_COUNT);
            
            // 如果是請求，記錄等待回應
            if (current_msg.buffer.netfn & 0x01 == 0) {
                insert_req_ipmi_msg(&req_list[index], 
                                    &current_msg.buffer, 
                                    index);
            }
        }
    }
}
```

### 接收任務

```c
// ipmb.c
void IPMB_RXTask(void *pvParameters, void *arvg0, void *arvg1)
{
    uint8_t index = (uintptr_t)pvParameters;
    ipmi_msg msg;
    uint8_t buffer[IPMB_MAX_LEN];
    
    while (1) {
        // 從接收佇列取得資料
        if (k_msgq_get(&ipmb_rxqueue[index], buffer, K_FOREVER) == 0) {
            
            // 驗證校驗和
            if (validate_checksum(buffer, buffer_len) != IPMB_ERROR_SUCCESS) {
                continue;
            }
            
            // 解碼 IPMB 訊息
            ipmb_decode(&msg, buffer, buffer_len);
            
            // 判斷是請求還是回應
            if (msg.netfn & 0x01) {
                // 回應訊息 - 匹配等待的請求
                if (find_req_ipmi_msg(&req_list[index], &msg, index)) {
                    // 處理回應
                }
            } else {
                // 請求訊息 - 通知 IPMI 處理器
                msg.InF_source = IPMB_inf_index_map[index];
                notify_ipmi_client(&msg_cfg);
            }
        }
    }
}
```

---

## 序列號管理

### 序列號分配

```c
// ipmb.c
static bool seq_table[MAX_IPMB_IDX][SEQ_NUM];

uint8_t get_free_seq(uint8_t index)
{
    k_mutex_lock(&mutex_id[index], K_FOREVER);
    
    for (int i = 0; i < SEQ_NUM; i++) {
        if (seq_table[index][i] == false) {
            seq_table[index][i] = true;
            k_mutex_unlock(&mutex_id[index]);
            return i;
        }
    }
    
    k_mutex_unlock(&mutex_id[index]);
    return SEQ_NUM;  // 無可用序列號
}

void register_seq(uint8_t index, uint8_t seq_num)
{
    seq_table[index][seq_num] = true;
}

void unregister_seq(uint8_t index, uint8_t seq_num)
{
    seq_table[index][seq_num] = false;
}
```

### 請求記錄

```c
// 記錄待回應的請求
void insert_req_ipmi_msg(ipmi_msg_cfg *pnode, ipmi_msg *msg, uint8_t index)
{
    k_mutex_lock(&mutex_id[index], K_FOREVER);
    
    // 在鏈結串列中插入新節點
    ipmi_msg_cfg *new_node = malloc(sizeof(ipmi_msg_cfg));
    memcpy(&new_node->buffer, msg, sizeof(ipmi_msg));
    new_node->timestamp = k_uptime_get();
    
    sys_slist_append(&req_list[index], &new_node->node);
    
    k_mutex_unlock(&mutex_id[index]);
}

// 尋找匹配的請求
bool find_req_ipmi_msg(ipmi_msg_cfg *pnode, ipmi_msg *msg, uint8_t index)
{
    k_mutex_lock(&mutex_id[index], K_FOREVER);
    
    ipmi_msg_cfg *node;
    SYS_SLIST_FOR_EACH_CONTAINER(&req_list[index], node, node) {
        if (node->buffer.seq == msg->seq &&
            node->buffer.cmd == msg->cmd) {
            // 找到匹配
            k_mutex_unlock(&mutex_id[index]);
            return true;
        }
    }
    
    k_mutex_unlock(&mutex_id[index]);
    return false;
}
```

---

## 發送/接收 API

### 發送請求

```c
// ipmb.c
ipmb_error ipmb_send_request(ipmi_msg *req, uint8_t index)
{
    if (index >= MAX_IPMB_IDX) {
        return IPMB_ERROR_INVALID_INDEX;
    }
    
    ipmi_msg_cfg msg_cfg;
    memcpy(&msg_cfg.buffer, req, sizeof(ipmi_msg));
    
    // 分配序列號
    msg_cfg.buffer.seq = get_free_seq(index);
    if (msg_cfg.buffer.seq == SEQ_NUM) {
        return IPMB_ERROR_NO_SEQ;
    }
    
    // 放入發送佇列
    if (k_msgq_put(&ipmb_txqueue[index], &msg_cfg, K_NO_WAIT) != 0) {
        unregister_seq(index, msg_cfg.buffer.seq);
        return IPMB_ERROR_FAILURE;
    }
    
    return IPMB_ERROR_SUCCESS;
}
```

### 發送回應

```c
ipmb_error ipmb_send_response(ipmi_msg *resp, uint8_t index)
{
    if (index >= MAX_IPMB_IDX) {
        return IPMB_ERROR_INVALID_INDEX;
    }
    
    ipmi_msg_cfg msg_cfg;
    memcpy(&msg_cfg.buffer, resp, sizeof(ipmi_msg));
    
    // 回應使用請求的序列號
    // netfn 增加 1 表示回應
    msg_cfg.buffer.netfn = resp->netfn | 0x01;
    
    if (k_msgq_put(&ipmb_txqueue[index], &msg_cfg, K_NO_WAIT) != 0) {
        return IPMB_ERROR_FAILURE;
    }
    
    return IPMB_ERROR_SUCCESS;
}
```

---

## 編碼/解碼

### 編碼

```c
// ipmb.c
ipmb_error ipmb_encode(uint8_t *buffer, ipmi_msg *msg)
{
    buffer[0] = msg->dest_addr;
    buffer[1] = (msg->netfn << 2) | msg->dest_lun;
    buffer[2] = calculate_checksum(buffer, 2);
    buffer[3] = msg->src_addr;
    buffer[4] = (msg->seq << 2) | msg->src_lun;
    buffer[5] = msg->cmd;
    
    memcpy(&buffer[6], msg->data, msg->data_len);
    
    buffer[6 + msg->data_len] = calculate_checksum(&buffer[3], 
                                                    3 + msg->data_len);
    
    return IPMB_ERROR_SUCCESS;
}
```

### 解碼

```c
ipmb_error ipmb_decode(ipmi_msg *msg, uint8_t *buffer, uint8_t len)
{
    msg->dest_addr = buffer[0];
    msg->netfn = buffer[1] >> 2;
    msg->dest_lun = buffer[1] & 0x03;
    msg->src_addr = buffer[3];
    msg->seq = buffer[4] >> 2;
    msg->src_lun = buffer[4] & 0x03;
    msg->cmd = buffer[5];
    
    msg->data_len = len - IPMB_HEADER_LEN - 1;  // -1 for checksum
    memcpy(msg->data, &buffer[6], msg->data_len);
    
    return IPMB_ERROR_SUCCESS;
}
```

---

## 校驗和

```c
// ipmb.c
uint8_t calculate_checksum(uint8_t *buffer, uint8_t range)
{
    uint8_t checksum = 0;
    for (int i = 0; i < range; i++) {
        checksum += buffer[i];
    }
    return (~checksum) + 1;  // 二補數
}

ipmb_error validate_checksum(uint8_t *buffer, uint8_t buffer_len)
{
    // 驗證標頭校驗和
    if (calculate_checksum(buffer, 2) != buffer[2]) {
        return IPMB_ERROR_HDR_CHECKSUM;
    }
    
    // 驗證資料校驗和
    if (calculate_checksum(&buffer[3], buffer_len - 4) != 
        buffer[buffer_len - 1]) {
        return IPMB_ERROR_DATA_CHECKSUM;
    }
    
    return IPMB_ERROR_SUCCESS;
}
```

---

## 超時處理

```c
// ipmb.c
void IPMB_SeqTimeout_handler(void *arug0, void *arug1, void *arug2)
{
    while (1) {
        k_msleep(IPMB_TIMEOUT_CHECK_MS);
        
        for (int index = 0; index < MAX_IPMB_IDX; index++) {
            k_mutex_lock(&mutex_id[index], K_FOREVER);
            
            int64_t current_time = k_uptime_get();
            ipmi_msg_cfg *node, *tmp;
            
            SYS_SLIST_FOR_EACH_CONTAINER_SAFE(&req_list[index], 
                                               node, tmp, node) {
                if (current_time - node->timestamp > IPMB_TIMEOUT_MS) {
                    // 超時處理
                    unregister_seq(index, node->buffer.seq);
                    sys_slist_find_and_remove(&req_list[index], 
                                              &node->node);
                    free(node);
                }
            }
            
            k_mutex_unlock(&mutex_id[index]);
        }
    }
}
```

---

## 初始化

```c
// ipmb.c
void ipmb_init(void)
{
    channel_index_mapping();
    
    for (int i = 0; i < MAX_IPMB_IDX; i++) {
        if (!IPMB_config_table[i].enable) {
            continue;
        }
        
        // 初始化互斥鎖
        k_mutex_init(&mutex_id[i]);
        k_mutex_init(&mutex_send_req[i]);
        
        // 初始化訊息佇列
        k_msgq_init(&ipmb_txqueue[i], ...);
        k_msgq_init(&ipmb_rxqueue[i], ...);
        
        // 建立 TX/RX 線程
        create_ipmb_threads(i);
        
        // 註冊 I2C Target
        register_target_device();
    }
}
```

---

## 相關文件

- [IPMIOverview](IPMIOverview.md) - IPMI 命令處理
- [I2C_I3C](I2C_I3C.md) - I2C 介面
- [MCTPOverview](MCTPOverview.md) - MCTP（替代選項）

---

*返回 [Home](Home.md)*
