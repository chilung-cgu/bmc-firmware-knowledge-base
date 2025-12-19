# PowerManagement（電源管理）

本文說明 OpenBIC yv4-sd 的電源狀態監控與管理機制。

---

## 電源狀態概述

SD BIC 監控多種電源狀態信號：

| 信號 | GPIO | 說明 |
|------|------|------|
| PWRGD_CPU_LVC3 | GPIO A3 | CPU 電源正常 |
| FM_CPU_BIC_SLP_S3_N | GPIO A1 | S3 睡眠狀態 |
| FM_CPU_BIC_SLP_S5_N | GPIO A0 | S5 睡眠狀態 |
| RST_RSMRST_BMC_N | GPIO A2 | BMC RSMRST |
| PWRGD_HSC_SLOT_BIC | GPIO L2 | HSC 電源正常 |

---

## DC 電源狀態

### 狀態查詢

```c
// power_status.c
static bool dc_status = false;

void set_DC_status(uint8_t gpio_num)
{
    dc_status = (gpio_get(gpio_num) == GPIO_HIGH);
}

bool get_DC_status()
{
    return dc_status;
}
```

### DC 延遲狀態

```c
// power_status.c
static bool dc_on_delayed_status = false;

// 在 DC 開啟 5 秒後設定
#define DC_ON_5_SECOND 5

K_WORK_DELAYABLE_DEFINE(set_DC_on_5s_work, set_DC_on_delayed_status);

void set_DC_on_delayed_status(struct k_work *work)
{
    dc_on_delayed_status = get_DC_status();
}
```

---

## POST 狀態

### 狀態管理

```c
// power_status.c
static bool post_status = false;

void set_post_status(uint8_t gpio_num)
{
    // FM_BIOS_POST_CMPLT_BIC_N 是低電位有效
    post_status = (gpio_get(gpio_num) == GPIO_LOW);
}

bool get_post_status()
{
    return post_status;
}

void reset_post_status()
{
    post_status = false;
}
```

### POST 超時

```c
// plat_isr.c
#define POST_TIMEOUT_SECONDS 1200  // 20 分鐘

K_TIMER_DEFINE(power_on_timer, post_timeout_handler, NULL);

void post_timeout_handler(struct k_timer *timer_id)
{
    LOG_WRN("BIOS POST timeout!");
    k_work_submit(&add_post_timeout_sel_work);
}

// 啟動 POST 超時計時器
if (get_DC_status() == true) {
    k_timer_start(&power_on_timer, 
                  K_SECONDS(POST_TIMEOUT_SECONDS), 
                  K_NO_WAIT);
}
```

---

## 電源事件處理

### DC ON/OFF 中斷

```c
// plat_isr.c
void ISR_DC_ON()
{
    // 更新 DC 狀態
    set_DC_status(PWRGD_CPU_LVC3);

    if (get_DC_status() == true) {
        // DC 電源開啟
        LOG_INF("DC power ON");
        
        // 延遲 5 秒後設定狀態
        k_work_schedule_for_queue(
            &plat_work_q, 
            &set_DC_on_5s_work, 
            K_SECONDS(DC_ON_5_SECOND));
        
        // 重新初始化 I3C
        k_work_submit(&reinit_i3c_work);
        
        // 啟動 POST 超時計時器
        k_timer_start(&power_on_timer, 
                      K_SECONDS(POST_TIMEOUT_SECONDS), 
                      K_NO_WAIT);
        
    } else {
        // DC 電源關閉
        LOG_INF("DC power OFF");
        
        // 停止 POST 超時計時器
        k_timer_stop(&power_on_timer);
        
        // 重置 POST 狀態
        reset_post_status();
        
        // 清除 dimm prsnt valid flag
        clear_valid_flag();
    }
}
```

### SLP_S3/S5 中斷

```c
// plat_isr.c
void ISR_SLP3()
{
    // 監控 S3 睡眠狀態
    uint8_t status = gpio_get(FM_CPU_BIC_SLP_S3_N);
    
    if (status == GPIO_LOW) {
        // 進入 S3 睡眠
        LOG_INF("Enter S3 sleep");
    } else {
        // 離開 S3 睡眠
        LOG_INF("Exit S3 sleep");
    }
}
```

### POST Complete 中斷

```c
void ISR_POST_COMPLETE()
{
    set_post_status(FM_BIOS_POST_CMPLT_BIC_N);

    if (get_post_status()) {
        // POST 完成
        LOG_INF("BIOS POST complete");
        
        // 停止 POST 超時計時器
        k_timer_stop(&power_on_timer);
        
        // 執行 POST 完成後初始化
        apml_recovery();
        set_tsi_threshold();
        disable_mailbox_completion_alert();
        enable_alert_signal();
        read_cpuid();
    }
}
```

---

## BMC Ready 狀態

### 狀態同步

```c
// plat_gpio.c
void sync_bmc_ready_pin()
{
    // 同步 BMC Ready 狀態到內部變數
    bmc_ready_status = (gpio_get(BMC_READY) == GPIO_HIGH);
    
    if (bmc_ready_status) {
        LOG_INF("BMC is ready");
        // 可以開始與 BMC 通訊
    }
}
```

### BMC Ready 中斷

```c
// plat_isr.c
void ISR_BMC_READY()
{
    sync_bmc_ready_pin();
    
    if (get_bmc_ready_status()) {
        // 觸發 MCTP/PLDM 通訊初始化
        k_work_submit(&init_mctp_comm_work);
    }
}
```

---

## 電源順序控制

### HSC 控制

```c
// 讀取 HSC 電源狀態
bool get_hsc_power_good()
{
    return (gpio_get(PWRGD_HSC_SLOT_BIC) == GPIO_HIGH);
}
```

### 電源控制 IPMI 命令

```c
// 透過 BMC 控制電源
void OEM_1S_INFORM_BMC_TO_CONTROL_POWER(ipmi_msg *msg)
{
    CHECK_NULL_ARG(msg);
    
    if (msg->data_len != 1) {
        msg->completion_code = CC_INVALID_LENGTH;
        return;
    }
    
    // 轉發電源控制請求至 BMC
    pldm_msg pmsg = { 0 };
    pmsg.ext_params.ep = MCTP_EID_BMC;
    pmsg.hdr.pldm_type = PLDM_TYPE_OEM;
    pmsg.hdr.cmd = PLDM_OEM_POWER_CONTROL;
    
    pmsg.buf = msg->data;
    pmsg.len = msg->data_len;
    
    mctp_pldm_send_msg(find_mctp_by_bus(bmc_bus), &pmsg);
    
    msg->completion_code = CC_SUCCESS;
}
```

---

## 感測器存取控制

### 電源狀態檢查器

```c
// 始終可存取
bool stby_access(uint8_t sensor_num)
{
    return true;
}

// DC 電源開啟時可存取
bool dc_access(uint8_t sensor_num)
{
    return get_DC_status();
}

// DC 電源開啟 5 秒後可存取
bool dc_on_5s_access(uint8_t sensor_num)
{
    return get_DC_on_delayed_status();
}

// POST 完成後可存取
bool post_access(uint8_t sensor_num)
{
    return (get_DC_status() && get_post_status());
}

// VR 無故障時可存取
bool vr_access(uint8_t sensor_num)
{
    return (get_DC_status() && !is_vr_fault());
}
```

### 感測器表配置

```c
sensor_cfg plat_sensor_config[] = {
    // Standby 電源感測器
    { SENSOR_NUM_TEMP_TMP75, sensor_dev_tmp75, ...,
      .access_checker = stby_access },
    
    // DC 電源感測器
    { SENSOR_NUM_VR_CPU_TEMP, sensor_dev_mp2971, ...,
      .access_checker = dc_access },
    
    // POST 完成後感測器
    { SENSOR_NUM_CPU_TEMP, sensor_dev_apml, ...,
      .access_checker = post_access },
};
```

---

## AC Lost 處理

```c
// plat_init.c
bool is_ac_lost()
{
    // 檢查是否發生 AC Lost
    // 通常透過 CPLD 或 HSC 暫存器判斷
    return (read_cpld_register(CPLD_AC_LOST_REG) != 0);
}

void pal_post_init()
{
    // ...
    
    if (is_ac_lost()) {
        LOG_INF("AC Lost detected, clearing VR faults");
        
        // 清除 VR 故障位元
        plat_pldm_sensor_clear_vr_fault(ADDR_VR_CPU0, I2C_BUS4, 2);
        plat_pldm_sensor_clear_vr_fault(ADDR_VR_CPU1, I2C_BUS4, 2);
        plat_pldm_sensor_clear_vr_fault(ADDR_VR_PVDD11, I2C_BUS4, 1);
    }
}
```

---

## THERMTRIP 處理

```c
// plat_isr.c
void ISR_THERMTRIP()
{
    // CPU 溫度過高觸發
    if (gpio_get(FM_CPU_BIC_THERMTRIP_N) == GPIO_LOW) {
        LOG_ERR("THERMTRIP asserted!");
        
        // 發送 SEL 事件
        common_addsel_msg_t sel_msg = { 0 };
        sel_msg.sensor_type = IPMI_SENSOR_TYPE_PROC;
        sel_msg.sensor_num = SENSOR_NUM_CPU_THERMTRIP;
        sel_msg.event_type = IPMI_EVENT_TYPE_SENSOR_SPECIFIC;
        sel_msg.event_data1 = 0x00;
        
        mctp_add_sel_to_ipmi(&sel_msg);
    }
}
```

---

## 相關文件

- [InterruptHandling](InterruptHandling.md) - 中斷處理
- [GPIO](GPIO.md) - GPIO 管理
- [SensorFramework](SensorFramework.md) - 感測器存取控制

---

*返回 [Home](Home.md)*
