# InterruptHandling（中斷服務處理）

本文說明 OpenBIC yv4-sd 的中斷服務處理機制，包括 ISR 設計與工作佇列。

---

## 中斷處理架構

```
┌─────────────────────────────────────────────────────────────────┐
│                     Hardware Events                             │
│  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐                  │
│  │ GPIO │ │ I2C  │ │ I3C  │ │ eSPI │ │ ADC  │                  │
│  └───┬──┘ └───┬──┘ └───┬──┘ └───┬──┘ └───┬──┘                  │
│      │        │        │        │        │                      │
│      └────────┴────────┴────────┴────────┘                      │
│                          │                                       │
│              ┌───────────▼───────────┐                          │
│              │    ISR (Interrupt     │                          │
│              │    Service Routine)   │                          │
│              └───────────┬───────────┘                          │
│                          │                                       │
│           ┌──────────────┼──────────────┐                       │
│           │   Quick Op   │  Deferred Op │                       │
│           ▼              ▼              ▼                       │
│     ┌──────────┐  ┌──────────┐  ┌──────────┐                   │
│     │ set FLAG │  │ k_work   │  │ k_timer  │                   │
│     │ read GPIO│  │ submit   │  │ start    │                   │
│     └──────────┘  └────┬─────┘  └────┬─────┘                   │
│                        │             │                          │
│              ┌─────────▼─────────────▼─────────┐               │
│              │      Work Queue / Timer         │               │
│              │   (Deferred Processing)         │               │
│              └────────────┬────────────────────┘               │
│                           │                                     │
│              ┌────────────▼────────────────────┐               │
│              │  Worker Functions               │               │
│              │  - SEL Recording                │               │
│              │  - State Changes                │               │
│              │  - Communication                │               │
│              └─────────────────────────────────┘               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## GPIO ISR

### 常見中斷事件

| GPIO | 事件 | ISR 函數 |
|------|------|----------|
| PWRGD_CPU_LVC3 | DC 電源變化 | ISR_DC_ON |
| FM_BIOS_POST_CMPLT_BIC_N | POST 完成 | ISR_POST_COMPLETE |
| BMC_READY | BMC 就緒 | ISR_BMC_READY |
| FM_CPU_BIC_SLP_S3_N | S3 睡眠狀態 | ISR_SLP3 |
| FM_DBP_PRESENT_N | DBP 存在 | ISR_DBP_PRSNT |
| RST_PLTRST_BIC_N | 平台重置 | ISR_PLTRST |
| CPU_SMERR_BIC_N | CPU SMERR | ISR_SMERR |
| FM_CPU_BIC_THERMTRIP_N | 溫度過高 | ISR_THERMTRIP |

### GPIO 中斷配置

```c
// plat_gpio.c
GPIO_CFG plat_gpio_cfg[] = {
    // PWRGD_CPU_LVC3 - 雙邊緣觸發
    { CHIP_GPIO, PWRGD_CPU_LVC3, ENABLE, GPIO_INPUT, 
      GPIO_LOW, GPIO_INT_EDGE_BOTH, ISR_DC_ON },
    
    // FM_BIOS_POST_CMPLT_BIC_N - 下降緣觸發
    { CHIP_GPIO, FM_BIOS_POST_CMPLT_BIC_N, ENABLE, GPIO_INPUT, 
      GPIO_HIGH, GPIO_INT_EDGE_FALLING, ISR_POST_COMPLETE },
    
    // FM_CPU_BIC_THERMTRIP_N - 下降緣觸發
    { CHIP_GPIO, FM_CPU_BIC_THERMTRIP_N, ENABLE, GPIO_INPUT, 
      GPIO_HIGH, GPIO_INT_EDGE_FALLING, ISR_THERMTRIP },
};
```

---

## ISR 實作

### ISR_DC_ON

```c
// plat_isr.c
void ISR_DC_ON()
{
    set_DC_status(PWRGD_CPU_LVC3);

    if (get_DC_status() == true) {
        // DC 電源開啟
        k_work_schedule_for_queue(&plat_work_q, 
                                   &set_DC_on_5s_work, 
                                   K_SECONDS(DC_ON_5_SECOND));
        k_work_submit(&reinit_i3c_work);
        k_timer_start(&power_on_timer, 
                      K_SECONDS(POST_TIMEOUT_SECONDS), 
                      K_NO_WAIT);
    } else {
        // DC 電源關閉
        k_timer_stop(&power_on_timer);
        reset_post_status();
    }
}
```

### ISR_POST_COMPLETE

```c
void ISR_POST_COMPLETE()
{
    set_post_status(FM_BIOS_POST_CMPLT_BIC_N);

    if (get_post_status()) {
        k_timer_stop(&power_on_timer);
        apml_recovery();
        set_tsi_threshold();
        disable_mailbox_completion_alert();
        enable_alert_signal();
        read_cpuid();
    }
}
```

### ISR_THERMTRIP

```c
void ISR_THERMTRIP()
{
    if (gpio_get(FM_CPU_BIC_THERMTRIP_N) == GPIO_LOW) {
        // 發送 SEL 事件
        k_work_submit(&add_thermtrip_sel_work);
    }
}
```

---

## 工作佇列

### 工作佇列定義

```c
// plat_isr.c
#define PLAT_WORK_Q_STACK_SIZE 2048
#define PLAT_WORK_Q_PRIORITY K_PRIO_PREEMPT(2)

K_THREAD_STACK_DEFINE(plat_work_q_stack, PLAT_WORK_Q_STACK_SIZE);
struct k_work_q plat_work_q;

void init_plat_worker(int priority)
{
    k_work_queue_start(&plat_work_q, 
                        plat_work_q_stack,
                        PLAT_WORK_Q_STACK_SIZE, 
                        priority,
                        NULL);
}
```

### 工作項定義

```c
// 即時工作項
K_WORK_DEFINE(reinit_i3c_work, reinit_i3c_hub);
K_WORK_DEFINE(add_thermtrip_sel_work, add_thermtrip_sel_handler);

// 延遲工作項
K_WORK_DELAYABLE_DEFINE(set_DC_on_5s_work, set_DC_on_delayed_status);
```

### 工作項處理函數

```c
void reinit_i3c_hub(struct k_work *work)
{
    I3C_MSG i3c_msg = { 0 };
    i3c_msg.bus = I3C_BUS_HUB;
    i3c_msg.target_addr = I3C_ADDR_HUB;

    // 重新初始化 I3C HUB
    uint16_t i3c_hub_type = get_i3c_hub_type();
    if (i3c_hub_type == P3H2840_DEVICE_INFO) {
        p3h284x_i3c_mode_only_init(&i3c_msg, 
                                    p3h284x_cmd_initial, 
                                    P3H284X_CMD_INITIAL_SIZE);
    } else {
        rg3mxxb12_i3c_mode_only_init(&i3c_msg, 
                                      rg3mxxb12_cmd_initial, 
                                      RG3MXXB12_CMD_INITIAL_SIZE);
    }
}
```

---

## 事件工作項陣列

### 結構定義

```c
// plat_isr.c
typedef struct {
    bool is_init;
    uint8_t gpio_num;
    uint8_t event_type;
    uint8_t assert_type;
    struct k_work_delayable add_sel_work;
} add_sel_info;
```

### 事件表

```c
add_sel_info event_work_items[] = {
    {
        .is_init = false,
        .gpio_num = RST_PLTRST_BIC_N,
        .event_type = PLTRST_ASSERT,
        .assert_type = EVENT_ASSERTED,
    },
    {
        .is_init = false,
        .gpio_num = FM_CPU_BIC_THERMTRIP_N,
        .event_type = SOC_THERMAL_TRIP,
        .assert_type = EVENT_ASSERTED,
    },
    {
        .is_init = false,
        .gpio_num = CPU_SMERR_BIC_N,
        .event_type = SMERR,
        .assert_type = EVENT_ASSERTED,
    },
};
```

### 初始化

```c
void init_event_work(void)
{
    for (int i = 0; i < ARRAY_SIZE(event_work_items); i++) {
        if (!event_work_items[i].is_init) {
            k_work_init_delayable(&event_work_items[i].add_sel_work,
                                   add_sel_work_handler);
            event_work_items[i].is_init = true;
        }
    }
}
```

---

## VR 警報處理

### VR 事件工作項

```c
typedef struct {
    bool is_init;
    uint8_t gpio_num;
    uint16_t vr_addr;
    struct k_work_delayable add_vr_sel_work;
} vr_event_work;

vr_event_work vr_event_work_item[] = {
    {
        .is_init = false,
        .gpio_num = PVDDCR_CPU0_PMALERT_N,
        .vr_addr = ADDR_VR_CPU0,
    },
    {
        .is_init = false,
        .gpio_num = PVDDCR_CPU1_PMALERT_N,
        .vr_addr = ADDR_VR_CPU1,
    },
};

void init_vr_event_work(void)
{
    for (int i = 0; i < ARRAY_SIZE(vr_event_work_item); i++) {
        if (!vr_event_work_item[i].is_init) {
            k_work_init_delayable(&vr_event_work_item[i].add_vr_sel_work,
                                   add_vr_alert_sel_work_handler);
            vr_event_work_item[i].is_init = true;
        }
    }
}
```

---

## SEL 事件記錄

### 新增 SEL

```c
// plat_mctp.c
bool mctp_add_sel_to_ipmi(common_addsel_msg_t *sel_msg)
{
    CHECK_NULL_ARG_WITH_RETURN(sel_msg, false);

    pldm_msg msg = { 0 };
    struct mctp_to_ipmi_sel_req req = { 0 };

    msg.ext_params.ep = MCTP_EID_BMC;
    msg.hdr.pldm_type = PLDM_TYPE_OEM;
    msg.hdr.cmd = PLDM_OEM_IPMI_BRIDGE;

    req.header.netfn_lun = (NETFN_STORAGE_REQ << 2);
    req.header.ipmi_cmd = CMD_STORAGE_ADD_SEL;
    req.req_data.event.record_type = 0x02;  // System Event

    memcpy(&req.req_data.event.sensor_type, 
           &sel_msg->sensor_type,
           sizeof(common_addsel_msg_t));

    msg.buf = (uint8_t *)&req;
    msg.len = sizeof(req);

    mctp_pldm_read(find_mctp_by_bus(bmc_bus), 
                    &msg, rbuf, resp_len);

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

## Throttle 處理

### 工作佇列

```c
#define SYS_THROTTLE_WORKQ_STACK_SIZE 1024
K_THREAD_STACK_DEFINE(sys_throttle_workq_stack, 
                       SYS_THROTTLE_WORKQ_STACK_SIZE);
struct k_work_q sys_throttle_work_q;

void init_throttle_work_q(void)
{
    k_work_queue_start(&sys_throttle_work_q, 
                        sys_throttle_workq_stack,
                        SYS_THROTTLE_WORKQ_STACK_SIZE, 
                        K_PRIO_PREEMPT(2),
                        NULL);
}
```

### Throttle ISR

```c
void ISR_SYS_THROTTLE()
{
    if (gpio_get(FAST_PROCHOT_N) == GPIO_LOW) {
        k_work_submit_to_queue(&sys_throttle_work_q, 
                                &add_throttle_sel_work);
    }
}
```

---

## 定時器

### POST 超時定時器

```c
K_TIMER_DEFINE(power_on_timer, post_timeout_handler, NULL);

void post_timeout_handler(struct k_timer *timer_id)
{
    LOG_WRN("BIOS POST timeout!");
    k_work_submit(&add_post_timeout_sel_work);
}
```

### 使用方式

```c
// 啟動定時器
k_timer_start(&power_on_timer, 
              K_SECONDS(POST_TIMEOUT_SECONDS), 
              K_NO_WAIT);

// 停止定時器
k_timer_stop(&power_on_timer);
```

---

## 相關文件

- [GPIO](GPIO.md) - GPIO 管理
- [PowerManagement](PowerManagement.md) - 電源管理
- [PLDMMonitor](PLDMMonitor.md) - PLDM 事件

---

*返回 [Home](Home.md)*
