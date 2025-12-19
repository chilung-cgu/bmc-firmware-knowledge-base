# AST1030Overview（ASPEED AST1030 晶片概述）

本文說明 OpenBIC 使用的 ASPEED AST1030 微控制器晶片的主要特性和周邊介面。

---

## 晶片概述

**ASPEED AST1030** 是專為伺服器管理設計的嵌入式微控制器，專門用於 Bridge IC (BIC) 應用。

### 主要規格

| 特性 | 規格 |
|------|------|
| **處理器核心** | ARM Cortex-M4F |
| **時脈頻率** | 200 MHz |
| **Flash** | 1MB 內建 SPI Flash |
| **SRAM** | 768KB |
| **封裝** | 128-pin LQFP |
| **工作溫度** | -40°C ~ 85°C |

### 架構圖

```
┌─────────────────────────────────────────────────────────────────┐
│                       ASPEED AST1030                            │
├─────────────────────────────────────────────────────────────────┤
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                  ARM Cortex-M4F @ 200MHz                   │ │
│  │                  ┌───────────┐ ┌───────────┐               │ │
│  │                  │   FPU     │ │   MPU     │               │ │
│  │                  └───────────┘ └───────────┘               │ │
│  └────────────────────────────────────────────────────────────┘ │
│                              │                                   │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │   1MB SPI   │  │  768KB SRAM │  │    ROM      │             │
│  │   Flash     │  │             │  │  (Boot)     │             │
│  └─────────────┘  └─────────────┘  └─────────────┘             │
│                              │                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                      AHB Bus Matrix                       │   │
│  └──────────────────────────────────────────────────────────┘   │
│         │           │           │           │                    │
│  ┌──────┴───┐ ┌─────┴────┐ ┌────┴─────┐ ┌───┴────┐             │
│  │ I2C x16  │ │ I3C x6   │ │ GPIO     │ │ SPI    │             │
│  └──────────┘ └──────────┘ └──────────┘ └────────┘             │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌────────┐             │
│  │ ADC x16  │ │ PWM/Tach │ │ eSPI     │ │ JTAG   │             │
│  └──────────┘ └──────────┘ └──────────┘ └────────┘             │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌────────┐             │
│  │ UART x13 │ │ USB 2.0  │ │ PECI     │ │ WDT    │             │
│  └──────────┘ └──────────┘ └──────────┘ └────────┘             │
└─────────────────────────────────────────────────────────────────┘
```

---

## 主要周邊介面

### I2C 介面

| 項目 | 規格 |
|------|------|
| **控制器數量** | 16 個 |
| **模式** | Master / Slave (Target) |
| **速度** | Standard (100kHz), Fast (400kHz), Fast-plus (1MHz) |
| **Buffer** | 每通道 32 bytes |

**OpenBIC 使用的 I2C Bus：**

| Bus | 用途 |
|-----|------|
| I2C Bus 0 | 感測器讀取 |
| I2C Bus 2 | BMC 通訊 (IPMB) |
| I2C Bus 4 | VR 控制 |
| I2C Bus 5 | CPLD 通訊 |

### I3C 介面

| 項目 | 規格 |
|------|------|
| **控制器數量** | 6 個 |
| **模式** | Controller / Target |
| **速度** | 最高 12.5 MHz |
| **特性** | DAA, IBI, Hot-Join |

**I3C 特性：**

- **DAA (Dynamic Address Assignment)**：動態地址分配
- **IBI (In-Band Interrupt)**：帶內中斷
- **CCC (Common Command Codes)**：通用命令碼

```c
// I3C 初始化範例
I3C_MSG i3c_msg = { 0 };
i3c_msg.bus = I3C_BUS_HUB;
i3c_msg.target_addr = I3C_ADDR_HUB;

// 重置 DAA
i3c_brocast_ccc(&i3c_msg, I3C_CCC_RSTDAA, I3C_BROADCAST_ADDR);

// 設定靜態地址為動態地址
i3c_brocast_ccc(&i3c_msg, I3C_CCC_SETAASA, I3C_BROADCAST_ADDR);
```

### GPIO

| 項目 | 規格 |
|------|------|
| **GPIO 數量** | 最多 168 個 |
| **群組** | A ~ U (每組 8 個) |
| **中斷** | 支援邊緣/位準中斷 |
| **Debounce** | 硬體防彈跳 |

**GPIO 命名規則：**

```c
// plat_gpio.h
#define name_gpioA \
    gpio_name_to_num(FM_CPU_BIC_SLP_S5_N) \
    gpio_name_to_num(FM_CPU_BIC_SLP_S3_N) \
    gpio_name_to_num(RST_RSMRST_BMC_N) \
    gpio_name_to_num(PWRGD_CPU_LVC3) \
    // ...
```

### ADC

| 項目 | 規格 |
|------|------|
| **通道數量** | 16 個 |
| **解析度** | 10-bit |
| **參考電壓** | 2.5V 或 1.2V |
| **取樣率** | 可配置 |

**ADC 用於 Slot 偵測：**

```c
// 讀取 ADC 電壓
float voltage;
get_adc_voltage(ADC_CHANNEL_13, &voltage);

// 參考電壓選擇
switch (reg_value & (BIT(7) | BIT(6))) {
case REF_VOL_2_5V:
    reference_voltage = 2.5;
    break;
case REF_VOL_1_2V:
    reference_voltage = 1.2;
    break;
}
```

### eSPI

| 項目 | 規格 |
|------|------|
| **模式** | Slave |
| **通道** | Peripheral, Virtual Wire, OOB, Flash |
| **速度** | 最高 66 MHz |

**Virtual Wire 處理：**

- SLP_S3/S5 狀態
- Host Power 狀態
- Platform Reset

### USB

| 項目 | 規格 |
|------|------|
| **版本** | USB 2.0 |
| **角色** | Device |
| **類別** | CDC ACM (串口) |

```ini
# prj.conf
CONFIG_USB=y
CONFIG_USB_DEVICE_STACK=y
CONFIG_USB_CDC_ACM=y
CONFIG_USB_CDC_ACM_RINGBUF_SIZE=576
```

### PECI

| 項目 | 規格 |
|------|------|
| **版本** | PECI 4.0 |
| **用途** | Intel CPU 溫度/狀態讀取 |

> [!NOTE]
> yv4-sd 平台使用 AMD CPU，因此 PECI 未啟用 (`CONFIG_PECI=n`)

### KCS (Keyboard Controller Style)

| 項目 | 規格 |
|------|------|
| **通道數量** | 4 個 |
| **用途** | Host BIOS 通訊 |
| **功能** | POST Code, IPMI |

---

## 記憶體映射

### Flash 空間配置

```
┌─────────────────────────────────────────┐ 0x00000000
│            Bootloader                    │
│            (可選)                        │
├─────────────────────────────────────────┤ 0x00010000
│                                          │
│            Application                   │
│            (OpenBIC FW)                  │
│                                          │
├─────────────────────────────────────────┤ 0x000F0000
│            Configuration                 │
│            (FRU, etc.)                   │
└─────────────────────────────────────────┘ 0x00100000 (1MB)
```

### SRAM 空間配置

```
┌─────────────────────────────────────────┐ 0x20000000
│            Stack                         │
├─────────────────────────────────────────┤
│            Heap                          │
├─────────────────────────────────────────┤
│            BSS                           │
├─────────────────────────────────────────┤
│            Data                          │
├─────────────────────────────────────────┤
│            Thread Stacks                 │
└─────────────────────────────────────────┘ 0x200C0000 (768KB)
```

---

## SCU (System Control Unit)

SCU 負責系統層級配置，包括時脈、多功能 pin 設定等：

```c
// plat_init.c
SCU_CFG scu_cfg[] = {
    // register    value
    { 0x7e6e2610, 0x04020000 },  // 多功能 pin 配置
    { 0x7e6e2618, 0x00c30000 },
    { 0x7e780040, 0x10000000 },  // GPIO Debounce Timer#2
    { 0x7e780054, 0x0001e848 },  // Debounce Timer = 2500us
};

void pal_pre_init()
{
    scu_init(scu_cfg, sizeof(scu_cfg) / sizeof(SCU_CFG));
    // ...
}
```

---

## 時脈架構

```
        ┌───────────┐
        │   PLL     │
        └─────┬─────┘
              │
    ┌─────────┼─────────┐
    │         │         │
┌───▼───┐ ┌───▼───┐ ┌───▼───┐
│ HCLK  │ │ PCLK  │ │ ECLK  │
│ 200M  │ │ 50M   │ │ 24M   │
└───┬───┘ └───┬───┘ └───┬───┘
    │         │         │
  ARM M4   Periph    eSPI/USB
```

---

## 相關文件

- [GPIO](GPIO.md) - GPIO 詳細說明
- [I2C_I3C](I2C_I3C.md) - I2C/I3C 介面
- [Architecture](Architecture.md) - 系統架構

---

*返回 [Home](Home.md)*
