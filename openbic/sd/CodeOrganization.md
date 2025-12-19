# CodeOrganization（程式碼組織）

本文說明 OpenBIC 專案的目錄結構、平台特定程式碼組織，以及建置系統配置。

---

## 目錄結構總覽

```
openbic/
├── .clang-format           # 程式碼格式化配置
├── .github/                # GitHub Actions 工作流程
├── README.md               # 專案說明
├── west.yml                # West manifest 配置
├── common/                 # 通用程式碼
│   ├── dev/               # 裝置驅動程式
│   ├── hal/               # 硬體抽象層
│   ├── include/           # 共用標頭檔
│   ├── lib/               # 工具函式庫
│   ├── service/           # 服務層
│   └── shell/             # Shell 命令
├── docs/                   # 文件與 Doxygen 配置
├── fix_patch/              # 修補檔案
├── meta-facebook/          # 平台特定程式碼
│   ├── yv4-sd/            # Sentinel Dome ← 目標平台
│   ├── yv4-ff/            # Firefly
│   ├── yv4-wf/            # Waimea Falls
│   └── ...                # 其他平台
└── scripts/                # 建置與工具腳本
```

---

## common 目錄

`common/` 包含所有平台共用的程式碼：

### common/dev（裝置驅動）

```
common/dev/
├── include/               # 驅動標頭檔
│   ├── tmp75.h
│   ├── isl69259.h
│   ├── ina233.h
│   └── ...
├── tmp75.c                # TMP75 溫度感測器
├── isl69259.c             # ISL69259 VR
├── ina233.c               # INA233 電流/功率監控
├── nvme.c                 # NVMe 溫度感測器
├── pmic.c                 # PMIC 驅動
├── eeprom.c               # EEPROM 存取
├── mp2971.c               # MPS VR 驅動
├── xdpe12284c.c           # Infineon VR
└── ...                    # 其他驅動（約 90+ 個）
```

**驅動程式結構範例：**

```c
// tmp75.c
#include "sensor.h"
#include "hal_i2c.h"

uint8_t tmp75_read(sensor_cfg *cfg, int *reading)
{
    uint8_t retry = 5;
    I2C_MSG msg = { 0 };

    msg.bus = cfg->port;
    msg.target_addr = cfg->target_addr;
    msg.tx_len = 1;
    msg.rx_len = 2;
    msg.data[0] = 0x00;  // Temperature register

    if (i2c_master_read(&msg, retry) != 0) {
        return SENSOR_FAIL_TO_ACCESS;
    }

    // 處理讀取數據
    *reading = (msg.data[0] << 8) | msg.data[1];
    return SENSOR_READ_SUCCESS;
}
```

### common/hal（硬體抽象層）

```
common/hal/
├── hal_gpio.c/h           # GPIO 操作
├── hal_i2c.c/h            # I2C master
├── hal_i2c_target.c/h     # I2C slave
├── hal_i3c.c/h            # I3C 操作
├── hal_jtag.c/h           # JTAG 介面
├── hal_peci.c/h           # PECI 介面
├── hal_vw_gpio.c/h        # Virtual Wire GPIO
└── hal_wdt.c/h            # 看門狗
```

### common/lib（工具函式庫）

```
common/lib/
├── libutil.c/h            # 通用工具函式
├── power_status.c/h       # 電源狀態管理
├── timer.c/h              # 計時器
├── util_pmbus.c/h         # PMBus 工具
├── util_spi.c/h           # SPI 工具
├── util_sys.c/h           # 系統工具
└── util_worker.c/h        # 工作佇列
```

### common/service（服務層）

```
common/service/
├── main.c                 # 主程式進入點
├── mctp/                  # MCTP 傳輸協議
│   ├── mctp.c/h
│   ├── mctp_ctrl.c/h
│   ├── mctp_i3c.c
│   ├── mctp_smbus.c
│   └── README.md
├── pldm/                  # PLDM 平台管理
│   ├── pldm.c/h
│   ├── pldm_base.c/h
│   ├── pldm_monitor.c/h
│   ├── pldm_firmware_update.c/h
│   ├── pldm_oem.c/h
│   └── pldm_smbios.c/h
├── ipmi/                  # IPMI 命令處理
│   ├── ipmi.c
│   ├── oem_1s_handler.c
│   ├── oem_handler.c
│   ├── sensor_handler.c
│   ├── storage_handler.c
│   └── include/
├── ipmb/                  # IPMB 通訊
│   ├── ipmb.c/h
├── sensor/                # 感測器框架
│   ├── sensor.c/h
│   ├── pldm_sensor.c/h
│   └── sdr.c/h
├── host/                  # Host 介面
├── apml/                  # AMD APML
├── cci/                   # CCI 介面
├── ncsi/                  # NC-SI 網路
├── modbus/                # Modbus
└── usb/                   # USB 裝置
```

---

## meta-facebook/yv4-sd（平台特定）

```
meta-facebook/yv4-sd/
├── CMakeLists.txt         # CMake 建置配置
├── prj.conf               # Zephyr 專案配置
├── boards/                # 板級配置
│   └── ast1030_evb.overlay
└── src/
    ├── ipmi/              # 平台 IPMI
    │   ├── include/
    │   │   └── plat_ipmi.h
    │   └── plat_ipmi.c    # OEM IPMI 命令
    ├── lib/               # 平台工具
    └── platform/          # 平台核心
        ├── plat_apml.c/h
        ├── plat_class.c/h      # 平台分類（Slot偵測）
        ├── plat_def.h          # 平台定義
        ├── plat_dimm.c/h       # DIMM 管理
        ├── plat_fru.h          # FRU 配置
        ├── plat_gpio.c/h       # GPIO 配置
        ├── plat_guid.h         # GUID 配置
        ├── plat_hook.c/h       # 感測器 Hook
        ├── plat_i2c.h          # I2C 配置
        ├── plat_i2c_target.c/h
        ├── plat_i3c.c/h        # I3C 配置
        ├── plat_init.c         # 初始化
        ├── plat_ipmb.h         # IPMB 配置
        ├── plat_isr.c/h        # 中斷服務
        ├── plat_kcs.c/h        # KCS 配置
        ├── plat_mctp.c/h       # MCTP 配置
        ├── plat_pcc.c/h        # PCC 配置
        ├── plat_pldm.c/h       # PLDM 配置
        ├── plat_pldm_device_identifier.c/h
        ├── plat_pldm_fw_update.c/h
        ├── plat_pldm_monitor.c/h
        ├── plat_pldm_sensor.c/h  # PLDM 感測器表
        ├── plat_pmic.c/h       # PMIC 處理
        ├── plat_sdr_table.c/h  # SDR 表
        ├── plat_sensor_table.c/h
        └── plat_version.h      # 版本資訊
```

---

## CMakeLists.txt 說明

```cmake
# meta-facebook/yv4-sd/CMakeLists.txt

cmake_minimum_required(VERSION 3.13.1)

find_package(Zephyr REQUIRED HINTS $ENV{ZEPHYR_BASE})
project(yv4-sd)

set(common_path ../../common)

# 收集來源檔案
FILE(GLOB app_sources 
    src/ipmi/*.c 
    src/platform/*.c 
    src/lib/*.c
)

FILE(GLOB common_sources 
    ${common_path}/service/*.c 
    ${common_path}/service/*/*.c 
    ${common_path}/hal/*.c 
    ${common_path}/dev/*.c 
    ${common_path}/shell/*.c 
    ${common_path}/shell/commands/*.c
)

# 添加來源
target_sources(app PRIVATE ${app_sources})
target_sources(app PRIVATE ${common_sources})

# Common Library
target_sources(app PRIVATE ${common_path}/lib/libutil.c)
target_sources(app PRIVATE ${common_path}/lib/power_status.c)
target_sources(app PRIVATE ${common_path}/lib/timer.c)
target_sources(app PRIVATE ${common_path}/lib/util_pmbus.c)
target_sources(app PRIVATE ${common_path}/lib/util_spi.c)
target_sources(app PRIVATE ${common_path}/lib/util_sys.c)
target_sources(app PRIVATE ${common_path}/lib/util_worker.c)

# Include 路徑
target_include_directories(app PRIVATE ${common_path})
target_include_directories(app PRIVATE ${common_path}/include)
target_include_directories(app PRIVATE ${common_path}/dev/include)
target_include_directories(app PRIVATE ${common_path}/hal)
target_include_directories(app PRIVATE ${common_path}/lib)
target_include_directories(app PRIVATE ${common_path}/service/host)
target_include_directories(app PRIVATE ${common_path}/service/ipmb)
target_include_directories(app PRIVATE ${common_path}/service/ipmi/include)
# ... 更多 include 路徑

# 平台特定 Include
target_include_directories(app PRIVATE src/ipmi/include)
target_include_directories(app PRIVATE src/lib)
target_include_directories(app PRIVATE src/platform)

# 編譯選項：將警告視為錯誤
target_compile_options(app PRIVATE -Werror)
```

---

## prj.conf 說明

```ini
# meta-facebook/yv4-sd/prj.conf

# CMSIS RTOS 配置
CONFIG_CMSIS_RTOS_V2=y
CONFIG_NUM_PREEMPT_PRIORITIES=56
CONFIG_HEAP_MEM_POOL_SIZE=32768
CONFIG_CMSIS_V2_THREAD_MAX_COUNT=23

# ASPEED 驅動程式
CONFIG_GPIO=y
CONFIG_SPI=y
CONFIG_I2C=y
CONFIG_I2C_SLAVE=y
CONFIG_I2C_IPMB_SLAVE=y
CONFIG_I3C=y
CONFIG_I3C_SLAVE=y
CONFIG_ADC=y
CONFIG_JTAG=y
CONFIG_FLASH=y
CONFIG_ESPI=y
CONFIG_WATCHDOG=y

# IPMI
CONFIG_IPMI=y
CONFIG_IPMI_KCS_ASPEED=y

# USB
CONFIG_USB=y
CONFIG_USB_DEVICE_STACK=y
CONFIG_USB_CDC_ACM=y

# 日誌
CONFIG_LOG=y
CONFIG_LOG_BACKEND_UART=y
CONFIG_LOG_BUFFER_SIZE=5120

# Shell
CONFIG_SHELL=y
CONFIG_GPIO_SHELL=y
CONFIG_I3C_SHELL=y

# 韌體名稱
CONFIG_KERNEL_BIN_NAME="Y4BSD"
```

---

## 平台程式碼命名規則

### plat_* 命名

平台特定程式碼使用 `plat_` 前綴：

| 檔案 | 用途 |
|------|------|
| `plat_init.c` | 平台初始化 |
| `plat_gpio.c/h` | GPIO 定義與處理 |
| `plat_mctp.c/h` | MCTP 路由與配置 |
| `plat_pldm.c/h` | PLDM 平台功能 |
| `plat_isr.c/h` | 中斷服務程式 |
| `plat_class.c/h` | 平台分類/Slot 偵測 |
| `plat_sensor_table.c/h` | 感測器配置表 |

### pal_* 函數

Platform Abstraction Layer 函數使用 `pal_` 前綴，這些是平台需要實作的介面：

```c
// common/service/main.c 中定義的弱符號
__weak void pal_pre_init() { return; }
__weak void pal_post_init() { return; }
__weak void pal_device_init() { return; }
__weak void pal_set_sys_status() { return; }

// 在 plat_init.c 中實作
void pal_pre_init()
{
    gpio_init(NULL);
    scu_init(scu_cfg, sizeof(scu_cfg) / sizeof(SCU_CFG));
    init_platform_config();
    // ...
}
```

---

## 擴展平台程式碼

### 新增感測器

1. **新增驅動**（如需）：在 `common/dev/` 新增驅動程式
2. **修改感測器表**：編輯 `plat_sensor_table.c`

```c
sensor_cfg plat_sensor_config[] = {
    // { sensor_num, type, port, addr, access_checker, ... }
    { SENSOR_NUM_TEMP_CPU, sensor_dev_tmp75, I2C_BUS0, 0x48, ... },
    { SENSOR_NUM_VR_CPU0_VOUT, sensor_dev_isl69259, I2C_BUS4, 0x60, ... },
};
```

### 新增 OEM IPMI 命令

修改 `src/ipmi/plat_ipmi.c`：

```c
void OEM_CMD_HANDLER(ipmi_msg *msg)
{
    CHECK_NULL_ARG(msg);
    
    switch (msg->data[0]) {
    case OEM_CMD_1:
        // 處理命令
        msg->completion_code = CC_SUCCESS;
        break;
    default:
        msg->completion_code = CC_INVALID_CMD;
        break;
    }
}
```

### 新增 GPIO 中斷

修改 `plat_isr.c`：

```c
void ISR_MY_EVENT()
{
    // 處理中斷事件
    if (gpio_get(MY_GPIO_PIN) == GPIO_HIGH) {
        // 事件發生
        common_addsel_msg_t sel_msg = { ... };
        bool ret = mctp_add_sel_to_ipmi(&sel_msg);
    }
}
```

---

## 相關文件

- [Architecture](Architecture.md) - 系統架構
- [SensorFramework](SensorFramework.md) - 感測器框架
- [DeviceDrivers](DeviceDrivers.md) - 裝置驅動
- [PlatformInit](PlatformInit.md) - 初始化流程

---

*返回 [Home](Home.md)*
