# KCS（KCS 介面）

本文說明 OpenBIC 的 KCS (Keyboard Controller Style) 介面，用於 Host 與 BIC 之間的通訊。

---

## KCS 概述

**KCS** 是 IPMI 規範定義的系統介面，透過 LPC 或 eSPI 連接 Host 與 BMC/BIC。

### 應用場景

| 應用 | 說明 |
|------|------|
| POST Code | BIOS POST 狀態碼 |
| IPMI 命令 | Host 軟體 IPMI |
| BIOS 通訊 | 韌體更新、配置 |

---

## KCS 暫存器

### 暫存器映射

| 暫存器 | 說明 |
|--------|------|
| Data In | Host 寫入資料 |
| Data Out | Host 讀取資料 |
| Command | Host 寫入命令 |
| Status | 狀態暫存器 |

### 狀態位元

| 位元 | 名稱 | 說明 |
|------|------|------|
| 7 | S1 | 狀態位 1 |
| 6 | S0 | 狀態位 0 |
| 5 | OEM2 | OEM 使用 |
| 4 | OEM1 | OEM 使用 |
| 3 | C/D | Command/Data |
| 2 | IBF | Input Buffer Full |
| 1 | Reserved | - |
| 0 | OBF | Output Buffer Full |

---

## OpenBIC KCS 實作

### 配置

```c
// plat_kcs.h
#define KCS_CHANNEL 3  // 使用的 KCS 通道

// prj.conf
CONFIG_IPMI=y
CONFIG_IPMI_KCS_ASPEED=y
```

### 初始化

```c
// plat_kcs.c
void kcs_init(void)
{
    // 取得 KCS 裝置
    const struct device *kcs_dev = device_get_binding("kcs3");
    if (!kcs_dev) {
        LOG_ERR("KCS device not found");
        return;
    }
    
    // 註冊回調
    ipmi_kcs_register_callback(kcs_dev, kcs_callback);
    
    LOG_INF("KCS initialized on channel %d", KCS_CHANNEL);
}
```

---

## POST Code

### PCC (Platform Communication Channel)

OpenBIC 透過 PCC 機制接收 BIOS POST Code：

```c
// pcc.c
void pcc_init(void)
{
    const struct device *pcc_dev = device_get_binding("PCC");
    if (!pcc_dev) {
        LOG_ERR("PCC device not found");
        return;
    }
    
    // 啟用 POST Code 擷取
    pcc_enable(pcc_dev);
    
    // 註冊回調
    pcc_register_callback(pcc_dev, pcc_rx_callback);
}
```

### POST Code 緩衝區

```c
// snoop.c
#define SNOOP_MAX_LEN 240

static uint8_t post_code_buf[SNOOP_MAX_LEN];
static uint16_t post_code_idx = 0;

void snoop_callback(const struct device *dev, uint8_t code)
{
    // 儲存 POST Code
    post_code_buf[post_code_idx % SNOOP_MAX_LEN] = code;
    post_code_idx++;
}

// 取得 POST Code
uint8_t get_post_code(uint8_t *buf, uint16_t *len)
{
    *len = post_code_idx > SNOOP_MAX_LEN ? SNOOP_MAX_LEN : post_code_idx;
    memcpy(buf, post_code_buf, *len);
    return 0;
}
```

### 4-Byte POST Code

```c
// plat_pcc.c
#define FOUR_BYTE_POST_CODE_PAGE_SIZE 60

struct four_byte_post_code {
    uint8_t code[4];
    uint32_t timestamp;
};

static struct four_byte_post_code post_code_4byte_buf[FOUR_BYTE_POST_CODE_PAGE_SIZE];
static uint16_t post_code_4byte_idx = 0;

void pcc_rx_callback(const struct device *dev, uint8_t *data, uint16_t len)
{
    if (len == 4) {
        memcpy(&post_code_4byte_buf[post_code_4byte_idx % FOUR_BYTE_POST_CODE_PAGE_SIZE].code,
               data, 4);
        post_code_4byte_buf[post_code_4byte_idx % FOUR_BYTE_POST_CODE_PAGE_SIZE].timestamp = 
            k_uptime_get_32();
        post_code_4byte_idx++;
    }
}
```

---

## IPMI over KCS

### KCS 訊息處理

```c
// kcs.c
void kcs_rx_callback(const struct device *dev, uint8_t *data, uint16_t len)
{
    ipmi_msg_cfg msg_cfg = { 0 };
    
    // 解析 IPMI 訊息
    msg_cfg.buffer.netfn = data[0] >> 2;
    msg_cfg.buffer.lun = data[0] & 0x03;
    msg_cfg.buffer.cmd = data[1];
    msg_cfg.buffer.data_len = len - 2;
    memcpy(msg_cfg.buffer.data, &data[2], msg_cfg.buffer.data_len);
    
    // 設定來源介面
    msg_cfg.buffer.InF_source = HOST_KCS;
    
    // 通知 IPMI 處理器
    notify_ipmi_client(&msg_cfg);
}
```

### KCS 回應發送

```c
void kcs_send_response(ipmi_msg *msg)
{
    const struct device *kcs_dev = device_get_binding("kcs3");
    
    uint8_t buf[IPMB_MAX_LEN];
    buf[0] = (msg->netfn << 2) | msg->lun;
    buf[1] = msg->cmd;
    buf[2] = msg->completion_code;
    memcpy(&buf[3], msg->data, msg->data_len);
    
    ipmi_kcs_write(kcs_dev, buf, msg->data_len + 3);
}
```

---

## OEM_1S_GET_POST_CODE

### 取得 POST Code

```c
// oem_1s_handler.c
__weak OEM_1S_GET_POST_CODE(ipmi_msg *msg)
{
    CHECK_NULL_ARG(msg);

    if (msg->data_len != 0) {
        msg->completion_code = CC_INVALID_LENGTH;
        return;
    }

    uint16_t len = 0;
    uint8_t status = get_post_code(msg->data, &len);
    
    if (status != 0) {
        msg->completion_code = CC_UNSPECIFIED_ERROR;
        return;
    }

    msg->data_len = len;
    msg->completion_code = CC_SUCCESS;
}
```

### 取得 4-Byte POST Code

```c
__weak OEM_1S_GET_4BYTE_POST_CODE(ipmi_msg *msg)
{
    CHECK_NULL_ARG(msg);

    if (msg->data_len != 1) {
        msg->completion_code = CC_INVALID_LENGTH;
        return;
    }

    uint8_t page = msg->data[0];
    uint16_t start_idx = page * FOUR_BYTE_POST_CODE_PAGE_SIZE;
    
    // 填充 POST Code 資料
    for (int i = 0; i < FOUR_BYTE_POST_CODE_PAGE_SIZE; i++) {
        if (start_idx + i >= post_code_4byte_idx) {
            break;
        }
        memcpy(&msg->data[i * 8], 
               &post_code_4byte_buf[(start_idx + i) % FOUR_BYTE_POST_CODE_PAGE_SIZE],
               8);
    }

    msg->completion_code = CC_SUCCESS;
}
```

---

## POST 完成偵測

### GPIO 中斷

```c
// plat_isr.c
void ISR_POST_COMPLETE()
{
    set_post_status(FM_BIOS_POST_CMPLT_BIC_N);
    
    if (get_post_status()) {
        // POST 完成
        LOG_INF("BIOS POST complete");
        
        // 停止 POST 超時計時器
        k_timer_stop(&power_on_timer);
        
        // 執行 POST 完成後操作
        apml_recovery();
        set_tsi_threshold();
        disable_mailbox_completion_alert();
        enable_alert_signal();
        read_cpuid();
    }
}
```

### POST 超時

```c
// plat_isr.c
#define POST_TIMEOUT_SECONDS 1200  // 20 分鐘

K_TIMER_DEFINE(power_on_timer, post_timeout_handler, NULL);

void post_timeout_handler(struct k_timer *timer_id)
{
    // POST 超時 - 發送 SEL
    LOG_WRN("BIOS POST timeout!");
    k_work_submit(&add_post_timeout_sel_work);
}

void add_post_timeout_sel_work_handler(struct k_work *work)
{
    common_addsel_msg_t sel_msg = { 0 };
    sel_msg.sensor_type = IPMI_SENSOR_TYPE_PROC;
    sel_msg.sensor_num = SENSOR_NUM_POST_TIMEOUT;
    sel_msg.event_type = IPMI_EVENT_TYPE_SENSOR_SPECIFIC;
    sel_msg.event_data1 = 0x00;
    
    mctp_add_sel_to_ipmi(&sel_msg);
}
```

---

## eSPI Virtual Wire

### VW 訊號處理

```c
// hal_vw_gpio.c
typedef struct {
    uint8_t vw_num;
    uint8_t bit_pos;
    void (*callback)(void);
} vw_gpio_cfg;

// SLP_S3/S5 透過 eSPI Virtual Wire 接收
void vw_slp_s3_handler(void)
{
    uint8_t status = vw_gpio_get(VW_SLP_S3);
    if (status == GPIO_LOW) {
        // SLP_S3 asserted - 進入 S3
        LOG_INF("SLP_S3 asserted");
    } else {
        // SLP_S3 de-asserted - 離開 S3
        LOG_INF("SLP_S3 de-asserted");
    }
}
```

---

## 相關文件

- [IPMIOverview](IPMIOverview.md) - IPMI 概述
- [PowerManagement](PowerManagement.md) - 電源管理
- [InterruptHandling](InterruptHandling.md) - 中斷處理

---

*返回 [Home](Home.md)*
