# Troubleshooting（故障排除）

本文說明 OpenBIC yv4-sd 的常見問題排除與除錯技巧。

---

## 常見問題

### 編譯錯誤

#### Q: 找不到 Zephyr kernel

```
CMake Error: ZEPHYR_BASE not set
```

**解決方案：**
```bash
export ZEPHYR_BASE=~/zephyrproject/zephyr
```

#### Q: West update 失敗

```
ERROR: Failed to update manifest
```

**解決方案：**
```bash
rm -rf ~/.cache/west
west update
```

#### Q: SDK 版本不匹配

**解決方案：**
```bash
# 確認 SDK 版本
ls ~/zephyr-sdk-*

# 設定正確版本
export ZEPHYR_SDK_INSTALL_DIR=~/zephyr-sdk-0.12.4
```

---

### 運行時問題

#### Q: BIC 無法與 BMC 通訊

**檢查步驟：**

1. **確認 I2C 連線**
```bash
uart:~$ i2c scan I2C_2
```

2. **確認 BMC Ready**
```bash
uart:~$ gpio get GPIO0_A_D 5
# BMC_READY 應為 1
```

3. **檢查 MCTP 配置**
```c
// 確認 EID 設定正確
LOG_INF("EID: 0x%02x", plat_eid);
```

#### Q: 感測器讀取失敗

**檢查步驟：**

1. **確認電源狀態**
```c
if (!get_DC_status()) {
    LOG_WRN("DC power is off");
}
```

2. **確認 I2C 地址**
```bash
uart:~$ i2c scan I2C_0
uart:~$ i2c scan I2C_4
```

3. **檢查感測器配置**
```c
// plat_sensor_table.c
sensor_cfg *cfg = find_sensor_by_num(sensor_num);
if (!cfg) {
    LOG_ERR("Sensor not found");
}
```

#### Q: I3C HUB 初始化失敗

**檢查步驟：**

1. **確認 I3C 電源**
2. **重試 DAA**
```c
for (int i = 0; i < 3; i++) {
    ret = i3c_brocast_ccc(&i3c_msg, I3C_CCC_RSTDAA, I3C_BROADCAST_ADDR);
    if (ret == 0) break;
    k_msleep(10);
}
```

3. **檢查 HUB 類型**
```c
uint16_t hub_type = get_i3c_hub_type();
LOG_INF("I3C HUB type: 0x%04x", hub_type);
```

---

## 日誌系統

### 設定日誌等級

```ini
# prj.conf
CONFIG_LOG=y
CONFIG_LOG_BACKEND_UART=y
CONFIG_LOG_BUFFER_SIZE=5120
CONFIG_LOG_DEFAULT_LEVEL=3  # INF

# 模組特定等級
CONFIG_SENSOR_LOG_LEVEL_DBG=y
CONFIG_I2C_LOG_LEVEL_DBG=y
```

### 使用日誌

```c
#include <logging/log.h>
LOG_MODULE_REGISTER(my_module, LOG_LEVEL_DBG);

LOG_DBG("Debug message");
LOG_INF("Info message");
LOG_WRN("Warning message");
LOG_ERR("Error message: %d", error_code);
```

### 日誌等級

| 等級 | 值 | 說明 |
|------|-----|------|
| OFF | 0 | 關閉 |
| ERR | 1 | 錯誤 |
| WRN | 2 | 警告 |
| INF | 3 | 資訊 |
| DBG | 4 | 除錯 |

---

## Shell 命令

### 啟用 Shell

```ini
# prj.conf
CONFIG_SHELL=y
CONFIG_GPIO_SHELL=y
CONFIG_I3C_SHELL=y
CONFIG_SENSOR_SHELL=y
```

### 常用命令

```bash
# GPIO 操作
uart:~$ gpio get GPIO0_A_D <pin>
uart:~$ gpio set GPIO0_A_D <pin> <0|1>
uart:~$ gpio conf GPIO0_A_D <pin> in|out

# I2C 掃描
uart:~$ i2c scan I2C_0
uart:~$ i2c scan I2C_2
uart:~$ i2c scan I2C_4

# I2C 讀寫
uart:~$ i2c read I2C_0 <addr> <reg> <len>
uart:~$ i2c write I2C_0 <addr> <reg> <data...>

# I3C 操作
uart:~$ i3c attach I3C_0
uart:~$ i3c info I3C_0

# 感測器
uart:~$ sensor list
uart:~$ sensor read <sensor_name>

# 系統資訊
uart:~$ kernel version
uart:~$ kernel threads
uart:~$ kernel stacks
```

---

## IPMI 診斷命令

### 從 BMC 執行

```bash
# 取得 BIC 狀態
ipmitool raw 0x38 0x58

# 取得 GPIO 狀態
ipmitool raw 0x38 0x02

# 取得韌體版本
ipmitool raw 0x38 0x0B 0x00  # BIC
ipmitool raw 0x38 0x0B 0x01  # CPLD

# 取得感測器讀數
ipmitool raw 0x38 0x1E <sensor_num>

# 取得 POST Code
ipmitool raw 0x38 0x12
```

---

## 硬體除錯

### JTAG/SWD

使用 OpenOCD 進行 JTAG 除錯：

```bash
# 啟動 OpenOCD
openocd -f board/aspeed_ast1030.cfg

# GDB 連線
arm-zephyr-eabi-gdb build/zephyr/zephyr.elf
(gdb) target remote :3333
(gdb) monitor reset halt
(gdb) load
(gdb) continue
```

### 記憶體檢查

```bash
# 檢查堆疊使用
uart:~$ kernel stacks

# 檢查記憶體
uart:~$ kernel heap
```

---

## 常見錯誤訊息

### I2C 錯誤

| 錯誤 | 可能原因 |
|------|----------|
| NACK | 地址錯誤或裝置離線 |
| Timeout | 匯流排卡住 |
| Bus Busy | 多主機衝突 |

**解決方案：**
```c
// 重試機制
int retry = 5;
while (retry--) {
    if (i2c_master_read(&msg, 1) == 0) {
        break;
    }
    k_msleep(10);
}
```

### MCTP 錯誤

| 錯誤 | 可能原因 |
|------|----------|
| EID not found | 路由表未配置 |
| Timeout | 目標無回應 |
| Instance ID exhausted | 並發請求過多 |

### PLDM 錯誤

| 完成碼 | 說明 |
|--------|------|
| 0x00 | 成功 |
| 0x01 | 錯誤 |
| 0x02 | 無效資料 |
| 0x03 | 無效長度 |
| 0x04 | 未就緒 |
| 0x05 | 不支援的命令 |

---

## 效能分析

### 計時測量

```c
#include <timing/timing.h>

timing_t start, end;
uint64_t cycles;

timing_init();
timing_start();

start = timing_counter_get();

// 待測量的程式碼
do_something();

end = timing_counter_get();
cycles = timing_cycles_get(&start, &end);

LOG_INF("Execution time: %llu cycles", cycles);
```

### 堆疊分析

```c
// 檢查線程堆疊使用
size_t unused = k_thread_stack_space_get(k_current_get());
LOG_INF("Stack unused: %zu bytes", unused);
```

---

## 日誌分析

### 常見日誌模式

```
# 正常啟動
Hello, welcome to Yosemite V4 Sentinel Dome 2024.47.01
I3C hub type: p3h284x
Detected Slot 3, EID: 0x20
BIOS POST complete

# 問題指示
Failed to read sensor 0x10
I2C read failed on bus 4, addr 0x60
MCTP timeout waiting for response
VR fault detected on CPU0
```

---

## 相關文件

- [QuickStart](QuickStart.md) - 開發環境
- [Architecture](Architecture.md) - 系統架構
- [APIReference](APIReference.md) - API 參考

---

*返回 [Home](Home.md)*
