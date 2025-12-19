# GPIO（GPIO 管理）

本文說明 OpenBIC 的 GPIO 管理機制，包括 HAL 層 API、平台 GPIO 定義，以及中斷處理。

---

## GPIO 概述

OpenBIC 使用 Zephyr GPIO 驅動與自訂 HAL 層管理 GPIO，提供：

- GPIO 初始化與配置
- 讀取/寫入操作
- 中斷回調註冊
- 防彈跳 (Debounce) 支援

---

## HAL 層 API

### 主要函數

| 函數 | 說明 |
|------|------|
| `gpio_init()` | GPIO 子系統初始化 |
| `gpio_cfg()` | 配置單一 GPIO |
| `gpio_get()` | 讀取 GPIO 狀態 |
| `gpio_set()` | 設定 GPIO 輸出 |
| `gpio_interrupt_conf()` | 配置中斷 |

### gpio_cfg 結構

```c
// hal_gpio.h
typedef struct _GPIO_CFG_ {
    uint8_t chip;           // GPIO 控制器
    uint8_t number;         // GPIO 編號
    uint8_t is_init;        // 是否初始化
    uint8_t direction;      // 輸入/輸出
    uint8_t status;         // 初始狀態
    uint8_t int_type;       // 中斷類型
    void (*int_cb)(void);   // 中斷回調函數
} GPIO_CFG;
```

### 方向與中斷類型

```c
// 方向
#define GPIO_INPUT      0
#define GPIO_OUTPUT     1

// 中斷類型
#define GPIO_INT_DISABLE      0
#define GPIO_INT_EDGE_RISING  1
#define GPIO_INT_EDGE_FALLING 2
#define GPIO_INT_EDGE_BOTH    3
#define GPIO_INT_LEVEL_LOW    4
#define GPIO_INT_LEVEL_HIGH   5
```

---

## 平台 GPIO 定義

### GPIO 命名宏

yv4-sd 平台在 `plat_gpio.h` 中定義所有 GPIO：

```c
// plat_gpio.h

// GPIO Group A (GPIO 0-7)
#define name_gpioA \
    gpio_name_to_num(FM_CPU_BIC_SLP_S5_N) \
    gpio_name_to_num(FM_CPU_BIC_SLP_S3_N) \
    gpio_name_to_num(RST_RSMRST_BMC_N) \
    gpio_name_to_num(PWRGD_CPU_LVC3) \
    gpio_name_to_num(CPU_SMERR_BIC_N) \
    gpio_name_to_num(BMC_READY) \
    gpio_name_to_num(RST_CPU_RESET_BIC_N) \
    gpio_name_to_num(RST_USB_HUB_R_N)

// GPIO Group B (GPIO 8-15)
#define name_gpioB \
    gpio_name_to_num(AUTH_PRSNT_BIC_N) \
    gpio_name_to_num(BIC_CPU_NMI_N) \
    gpio_name_to_num(FM_SMI_ACTIVE_N) \
    gpio_name_to_num(IRQ_BIC_CPU_SMI_N) \
    gpio_name_to_num(FM_CPU_BIC_THERMTRIP_N) \
    gpio_name_to_num(APML_CPU_ALERT_BIC_N) \
    gpio_name_to_num(PRSNT_CPU_R_N) \
    gpio_name_to_num(SYS_PWRBTN_BIC_N)

// ... 更多 GPIO 群組定義
```

### GPIO 列舉

```c
// 產生 GPIO 編號列舉
#define gpio_name_to_num(x) x,
enum _GPIO_NUMS_ {
    name_gpioA name_gpioB name_gpioC name_gpioD name_gpioE name_gpioF name_gpioG name_gpioH
    name_gpioI name_gpioJ name_gpioK name_gpioL name_gpioM name_gpioN name_gpioO
    name_gpioP name_gpioQ name_gpioR name_gpioS name_gpioT name_gpioU
};
#undef gpio_name_to_num
```

### 常用 GPIO 說明

| GPIO 名稱 | 方向 | 說明 |
|-----------|------|------|
| `FM_CPU_BIC_SLP_S5_N` | 輸入 | CPU S5 睡眠狀態 |
| `FM_CPU_BIC_SLP_S3_N` | 輸入 | CPU S3 睡眠狀態 |
| `PWRGD_CPU_LVC3` | 輸入 | CPU 電源正常 |
| `RST_PLTRST_BIC_N` | 輸入 | 平台重置信號 |
| `FM_BIOS_POST_CMPLT_BIC_N` | 輸入 | BIOS POST 完成 |
| `BMC_READY` | 輸入 | BMC 就緒信號 |
| `BIC_READY_R` | 輸出 | BIC 就緒信號 |
| `BIC_CPU_NMI_N` | 輸出 | 發送 NMI 至 CPU |

---

## GPIO 配置表

### plat_gpio.c 配置

```c
// plat_gpio.c
GPIO_CFG plat_gpio_cfg[] = {
    // chip, number, is_init, direction, status, int_type, int_cb
    
    // FM_CPU_BIC_SLP_S5_N - 輸入，上升沿中斷
    { CHIP_GPIO, FM_CPU_BIC_SLP_S5_N, ENABLE, GPIO_INPUT, 
      GPIO_LOW, GPIO_INT_EDGE_BOTH, ISR_DC_ON },
    
    // PWRGD_CPU_LVC3 - 輸入，邊緣中斷監控電源狀態
    { CHIP_GPIO, PWRGD_CPU_LVC3, ENABLE, GPIO_INPUT, 
      GPIO_LOW, GPIO_INT_EDGE_BOTH, ISR_DC_ON },
    
    // FM_BIOS_POST_CMPLT_BIC_N - POST 完成中斷
    { CHIP_GPIO, FM_BIOS_POST_CMPLT_BIC_N, ENABLE, GPIO_INPUT, 
      GPIO_HIGH, GPIO_INT_EDGE_FALLING, ISR_POST_COMPLETE },
    
    // RST_PLTRST_BIC_N - 平台重置
    { CHIP_GPIO, RST_PLTRST_BIC_N, ENABLE, GPIO_INPUT, 
      GPIO_HIGH, GPIO_INT_EDGE_BOTH, ISR_PLTRST },
    
    // BIC_READY_R - 輸出，指示 BIC 就緒
    { CHIP_GPIO, BIC_READY_R, ENABLE, GPIO_OUTPUT, 
      GPIO_LOW, GPIO_INT_DISABLE, NULL },
    
    // ... 更多配置
};
```

---

## 中斷處理

### 中斷服務程式

中斷處理函數定義在 `plat_isr.c`：

```c
// plat_isr.c

void ISR_DC_ON()
{
    set_DC_status(PWRGD_CPU_LVC3);
    
    if (get_DC_status() == true) {
        // DC 電源開啟
        k_work_schedule_for_queue(
            &plat_work_q, &set_DC_on_5s_work, 
            K_SECONDS(DC_ON_5_SECOND));
        k_work_submit(&reinit_i3c_work);
        k_timer_start(&power_on_timer, 
            K_SECONDS(POST_TIMEOUT_SECONDS), K_NO_WAIT);
    } else {
        // DC 電源關閉
        k_timer_stop(&power_on_timer);
        reset_post_status();
    }
}

void ISR_POST_COMPLETE()
{
    set_post_status(FM_BIOS_POST_CMPLT_BIC_N);
    
    if (get_post_status()) {
        // POST 完成
        k_timer_stop(&power_on_timer);
        apml_recovery();
        set_tsi_threshold();
    }
}

void ISR_BMC_READY()
{
    sync_bmc_ready_pin();
}
```

### 事件工作項

使用 Zephyr 工作佇列處理耗時操作：

```c
// 定義工作項
K_WORK_DELAYABLE_DEFINE(set_DC_on_5s_work, set_DC_on_delayed_status);
K_WORK_DEFINE(reinit_i3c_work, reinit_i3c_hub);

// 事件資訊結構
typedef struct {
    bool is_init;
    uint8_t gpio_num;
    uint8_t event_type;
    uint8_t assert_type;
    struct k_work_delayable add_sel_work;
} add_sel_info;

// 事件工作項陣列
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
    // ...
};
```

---

## GPIO 使用範例

### 讀取 GPIO

```c
// 讀取單一 GPIO
uint8_t status = gpio_get(PWRGD_CPU_LVC3);
if (status == GPIO_HIGH) {
    // CPU 電源正常
}

// 讀取多個 GPIO 組成狀態
uint8_t rtm_0 = gpio_get(RTM_TYPE_0);
uint8_t rtm_1 = gpio_get(RTM_TYPE_1);
// retimer type = (rtm_1 << 1) | rtm_0
```

### 設定 GPIO

```c
// 設定 GPIO 輸出高電位
gpio_set(BIC_READY_R, GPIO_HIGH);

// 設定 GPIO 輸出低電位
gpio_set(RST_USB_HUB_R_N, GPIO_LOW);
k_msleep(10);
gpio_set(RST_USB_HUB_R_N, GPIO_HIGH);
```

### 動態配置中斷

```c
// 配置 GPIO 中斷
gpio_interrupt_conf(FM_DBP_PRESENT_N, GPIO_INT_EDGE_BOTH);

// 禁用中斷
gpio_interrupt_conf(FM_DBP_PRESENT_N, GPIO_INT_DISABLE);
```

---

## Debounce 配置

透過 SCU 配置 GPIO 防彈跳：

```c
// plat_init.c
SCU_CFG scu_cfg[] = {
    // 啟用 GPIOD4 Debounce Timer#2 for throttle
    { 0x7e780040, 0x10000000 },
    // 設定 Debounce Timer#2 為 2500us
    { 0x7e780054, 0x0001e848 },
};

void pal_pre_init()
{
    // 配置 GPIO 使用 debounce
    gpio_pin_configure(gpio_dev, 26,
        GPIO_INPUT | GPIO_INT_DEBOUNCE);
}
```

---

## Shell 命令

透過 Zephyr Shell 操作 GPIO：

```bash
# 列出所有 GPIO
uart:~$ gpio
Usage: gpio <sub_command>

# 讀取 GPIO
uart:~$ gpio get GPIO0_A_D 0
Value: 1

# 設定 GPIO
uart:~$ gpio set GPIO0_A_D 23 1

# 配置 GPIO
uart:~$ gpio conf GPIO0_A_D 0 in
```

---

## EVT 階段特殊處理

不同硬體版本可能有 GPIO 差異：

```c
// plat_gpio.h
#define EVT_EXAMAX_RESERVED_1 18
#define EVT_CPU_TYPE_1 52
#define EVT_RTM_IOEXP_INT_N 53
// ...

// plat_class.c
void init_retimer_type()
{
    uint8_t board_rev = 0;
    get_board_rev(&board_rev);

    if (board_rev <= BOARD_REV_EVT) {
        retimer_type = RETIMER_TYPE_ASTERALABS;
    } else {
        rtm_0 = gpio_get(RTM_TYPE_0);
        rtm_1 = gpio_get(RTM_TYPE_1);
        // ...
    }
}
```

---

## 相關文件

- [AST1030Overview](AST1030Overview.md) - 晶片概述
- [InterruptHandling](InterruptHandling.md) - 中斷處理詳解
- [PowerManagement](PowerManagement.md) - 電源狀態管理

---

*返回 [Home](Home.md)*
