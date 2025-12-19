# APIReference（API 參考）

本文列出 OpenBIC 常用 API、資料結構與錯誤碼定義。

---

## GPIO API

### 函數

| 函數 | 說明 |
|------|------|
| `gpio_init(GPIO_CFG *cfg)` | 初始化 GPIO |
| `gpio_get(uint8_t pin)` | 讀取 GPIO 狀態 |
| `gpio_set(uint8_t pin, uint8_t val)` | 設定 GPIO 輸出 |
| `gpio_interrupt_conf(uint8_t pin, uint8_t int_type)` | 配置中斷 |

### 常數

```c
// 方向
#define GPIO_INPUT      0
#define GPIO_OUTPUT     1

// 狀態
#define GPIO_LOW        0
#define GPIO_HIGH       1

// 中斷類型
#define GPIO_INT_DISABLE      0
#define GPIO_INT_EDGE_RISING  1
#define GPIO_INT_EDGE_FALLING 2
#define GPIO_INT_EDGE_BOTH    3
#define GPIO_INT_LEVEL_LOW    4
#define GPIO_INT_LEVEL_HIGH   5
```

---

## I2C API

### 函數

| 函數 | 說明 |
|------|------|
| `i2c_master_read(I2C_MSG *msg, uint8_t retry)` | I2C 讀取 |
| `i2c_master_write(I2C_MSG *msg, uint8_t retry)` | I2C 寫入 |
| `util_init_I2C()` | 初始化 I2C |
| `i2c_target_control(int idx, cfg, enable)` | I2C Target 控制 |

### 結構

```c
typedef struct _I2C_MSG_ {
    uint8_t bus;
    uint8_t target_addr;
    uint8_t tx_len;
    uint8_t rx_len;
    uint8_t data[I2C_BUFF_SIZE];
} I2C_MSG;
```

---

## I3C API

### 函數

| 函數 | 說明 |
|------|------|
| `i3c_attach(I3C_MSG *msg)` | 附加 I3C 裝置 |
| `i3c_transfer(I3C_MSG *msg)` | I3C 傳輸 |
| `i3c_brocast_ccc(msg, ccc_id, addr)` | 廣播 CCC 命令 |
| `util_init_i3c()` | 初始化 I3C |

### CCC 命令

```c
#define I3C_CCC_RSTDAA   0x06  // 重置 DAA
#define I3C_CCC_SETAASA  0x29  // 設定靜態地址為動態地址
#define I3C_CCC_SETDASA  0x87  // 設定動態地址
#define I3C_CCC_GETPID   0x8D  // 取得 PID
```

---

## Sensor API

### 函數

| 函數 | 說明 |
|------|------|
| `sensor_init()` | 初始化感測器框架 |
| `sensor_read(sensor_cfg *cfg, int *reading)` | 讀取感測器 |
| `find_sensor_by_num(uint8_t num)` | 查找感測器 |
| `control_sensor_polling(num, enable)` | 控制輪詢 |

### 結構

```c
typedef struct _sensor_cfg_ {
    uint8_t num;
    uint8_t type;
    uint8_t port;
    uint8_t target_addr;
    uint8_t offset;
    bool (*access_checker)(uint8_t);
    uint8_t arg0;
    uint8_t arg1;
    int cache;
    uint8_t cache_status;
    int (*pre_sensor_read_hook)(struct _sensor_cfg_ *, void *);
    int (*post_sensor_read_hook)(struct _sensor_cfg_ *, void *, int *);
} sensor_cfg;
```

### 感測器狀態

```c
#define SENSOR_READ_SUCCESS       0x00
#define SENSOR_NOT_ACCESSIBLE     0x01
#define SENSOR_INIT_STATUS        0x02
#define SENSOR_NOT_FOUND          0x03
#define SENSOR_FAIL_TO_ACCESS     0x04
#define SENSOR_UNSPECIFIED_ERROR  0x05
#define SENSOR_POLLING_DISABLE    0x06
```

---

## MCTP API

### 函數

| 函數 | 說明 |
|------|------|
| `mctp_init()` | 初始化 MCTP |
| `mctp_set_medium_configure(inst, type, conf)` | 設定介質 |
| `mctp_reg_endpoint_resolve_func(inst, fn)` | 註冊端點解析 |
| `mctp_reg_msg_rx_func(inst, fn)` | 註冊接收回調 |
| `mctp_start(inst)` | 啟動 MCTP |
| `mctp_send_msg(inst, buf, len, ext_params)` | 發送訊息 |

### 介質類型

```c
typedef enum {
    MCTP_MEDIUM_TYPE_UNKNOWN = 0,
    MCTP_MEDIUM_TYPE_SMBUS,
    MCTP_MEDIUM_TYPE_TARGET_I3C,
    MCTP_MEDIUM_TYPE_CONTROLLER_I3C,
} MCTP_MEDIUM_TYPE;
```

---

## PLDM API

### 函數

| 函數 | 說明 |
|------|------|
| `mctp_pldm_send_msg(mctp_p, pldm_msg)` | 發送 PLDM 訊息 |
| `mctp_pldm_read(mctp_p, msg, rbuf, rbuf_len)` | 同步讀取 |
| `pldm_handler_query(type, cmd)` | 查詢命令處理器 |

### 結構

```c
typedef struct {
    pldm_hdr hdr;
    uint8_t *buf;
    uint16_t len;
    mctp_ext_params ext_params;
    void (*recv_resp_cb_fn)(void *, uint8_t *, uint16_t);
    void (*timeout_cb_fn)(void *);
} pldm_msg;

typedef struct __attribute__((packed)) {
    uint8_t inst_id : 5;
    uint8_t rsvd : 1;
    uint8_t d : 1;
    uint8_t rq : 1;
    uint8_t pldm_type : 6;
    uint8_t ver : 2;
    uint8_t cmd;
} pldm_hdr;
```

### PLDM 類型

```c
#define PLDM_TYPE_BASE       0x00
#define PLDM_TYPE_SMBIOS     0x01
#define PLDM_TYPE_PLAT_MON   0x02
#define PLDM_TYPE_BIOS_CTRL  0x03
#define PLDM_TYPE_FRU        0x04
#define PLDM_TYPE_FW_UPDATE  0x05
#define PLDM_TYPE_OEM        0x3F
```

---

## IPMI API

### 函數

| 函數 | 說明 |
|------|------|
| `ipmi_init()` | 初始化 IPMI |
| `notify_ipmi_client(ipmi_msg_cfg *)` | 通知 IPMI 處理器 |
| `ipmb_send_request(msg, index)` | 發送 IPMB 請求 |
| `ipmb_send_response(msg, index)` | 發送 IPMB 回應 |

### 結構

```c
typedef struct _ipmi_msg {
    uint8_t netfn;
    uint8_t cmd;
    uint8_t completion_code;
    uint16_t data_len;
    uint8_t *data;
    uint8_t seq_source;
    uint8_t InF_source;
    uint8_t InF_target;
} ipmi_msg;
```

### 完成代碼

```c
#define CC_SUCCESS              0x00
#define CC_NODE_BUSY            0xC0
#define CC_INVALID_CMD          0xC1
#define CC_INVALID_LUN          0xC2
#define CC_TIMEOUT              0xC3
#define CC_OUT_OF_SPACE         0xC4
#define CC_INVALID_DATA_FIELD   0xCC
#define CC_SENSOR_NOT_PRESENT   0xCB
#define CC_INVALID_LENGTH       0xC7
#define CC_UNSPECIFIED_ERROR    0xFF
```

---

## Power Status API

### 函數

| 函數 | 說明 |
|------|------|
| `set_DC_status(gpio_num)` | 設定 DC 狀態 |
| `get_DC_status()` | 取得 DC 狀態 |
| `get_DC_on_delayed_status()` | 取得延遲狀態 |
| `set_post_status(gpio_num)` | 設定 POST 狀態 |
| `get_post_status()` | 取得 POST 狀態 |
| `reset_post_status()` | 重置 POST 狀態 |

---

## Platform API

### 函數

| 函數 | 說明 |
|------|------|
| `pal_pre_init()` | 平台預初始化 |
| `pal_post_init()` | 平台後初始化 |
| `pal_device_init()` | 裝置初始化 |
| `pal_set_sys_status()` | 設定系統狀態 |
| `get_slot_eid()` | 取得 Slot EID |
| `get_slot_id()` | 取得 Slot ID |
| `get_board_rev(uint8_t *rev)` | 取得板本 |
| `get_retimer_type()` | 取得 Retimer 類型 |
| `get_vr_type()` | 取得 VR 類型 |

---

## 工作佇列 API

### Zephyr API

```c
// 定義工作佇列
K_THREAD_STACK_DEFINE(my_work_q_stack, 1024);
struct k_work_q my_work_q;

// 啟動工作佇列
k_work_queue_start(&my_work_q, my_work_q_stack, 1024, K_PRIO_PREEMPT(2), NULL);

// 定義工作項
K_WORK_DEFINE(my_work, my_work_handler);
K_WORK_DELAYABLE_DEFINE(my_delayed_work, my_delayed_handler);

// 提交工作
k_work_submit(&my_work);
k_work_submit_to_queue(&my_work_q, &my_work);
k_work_schedule(&my_delayed_work, K_MSEC(100));
k_work_schedule_for_queue(&my_work_q, &my_delayed_work, K_SECONDS(5));
```

---

## 定時器 API

### Zephyr API

```c
// 定義定時器
K_TIMER_DEFINE(my_timer, expiry_fn, stop_fn);

// 啟動定時器
k_timer_start(&my_timer, K_SECONDS(10), K_NO_WAIT);  // 一次性
k_timer_start(&my_timer, K_MSEC(100), K_MSEC(100));  // 週期性

// 停止定時器
k_timer_stop(&my_timer);
```

---

## 錯誤碼參考

### MCTP 錯誤

```c
#define MCTP_SUCCESS        0x00
#define MCTP_ERROR          0x01
#define MCTP_ERROR_INVALID  0x02
```

### PLDM 錯誤

```c
#define PLDM_SUCCESS                    0x00
#define PLDM_ERROR                      0x01
#define PLDM_ERROR_INVALID_DATA         0x02
#define PLDM_ERROR_INVALID_LENGTH       0x03
#define PLDM_ERROR_NOT_READY            0x04
#define PLDM_ERROR_UNSUPPORTED_CMD      0x05
#define PLDM_ERROR_INVALID_SENSOR_ID    0x20
#define PLDM_ERROR_INVALID_EFFECTER_ID  0x21
```

### IPMB 錯誤

```c
#define IPMB_ERROR_SUCCESS       0
#define IPMB_ERROR_HDR_CHECKSUM  1
#define IPMB_ERROR_DATA_CHECKSUM 2
#define IPMB_ERROR_FAILURE       3
#define IPMB_ERROR_INVALID_INDEX 4
#define IPMB_ERROR_NO_SEQ        5
```

---

## 相關文件

- [Architecture](Architecture.md) - 系統架構
- [SensorFramework](SensorFramework.md) - 感測器框架
- [MCTPOverview](MCTPOverview.md) - MCTP 協議
- [Troubleshooting](Troubleshooting.md) - 故障排除

---

*返回 [Home](Home.md)*
