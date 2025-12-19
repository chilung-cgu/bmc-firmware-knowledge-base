# YV4SDPlatform（Yosemite V4 SD 平台）

本文說明 Yosemite V4 Sentinel Dome (SD) BIC 平台的硬體拓撲與系統特性。

---

## 平台概述

**Yosemite V4** 是 Meta/Facebook 資料中心的多主機伺服器平台。**Sentinel Dome (SD)** BIC 是主要的 Bridge IC，負責管理單一計算模組。

### 平台代號

| 代號 | 全名 | 說明 |
|------|------|------|
| **yv4** | Yosemite V4 | 平台名稱 |
| **sd** | Sentinel Dome | 主 BIC |
| **ff** | Firefly | 擴展 BIC 1 |
| **wf** | Waimea Falls | 擴展 BIC 2 |

### 版本資訊

```c
// plat_version.h
#define PLATFORM_NAME "Yosemite V4"
#define PROJECT_NAME "Sentinel Dome"
#define PROJECT_STAGE MP

#define BOARD_ID 0x01
#define DEVICE_ID 0x00

#define BIC_FW_platform_0 0x73  // 's'
#define BIC_FW_platform_1 0x64  // 'd'
```

---

## 硬體拓撲

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Yosemite V4 System                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                         Blade Slot 1~8                              │   │
│  │  ┌───────────────────────────────────────────────────────────────┐  │   │
│  │  │                    Compute Module                             │  │   │
│  │  │                                                               │  │   │
│  │  │  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐       │  │   │
│  │  │  │  Host CPU   │◄──►│   SD BIC    │◄──►│   Sensors   │       │  │   │
│  │  │  │  (AMD/Intel)│    │  (AST1030)  │    │  VR, Temp   │       │  │   │
│  │  │  └─────────────┘    └──────┬──────┘    └─────────────┘       │  │   │
│  │  │                            │                                  │  │   │
│  │  │         ┌──────────────────┼──────────────────┐              │  │   │
│  │  │         │                  │                  │              │  │   │
│  │  │    ┌────▼────┐        ┌────▼────┐        ┌────▼────┐        │  │   │
│  │  │    │  FF BIC │        │  CPLD   │        │  WF BIC │        │  │   │
│  │  │    │         │        │   IO    │        │         │        │  │   │
│  │  │    └────┬────┘        └─────────┘        └────┬────┘        │  │   │
│  │  │         │                                      │             │  │   │
│  │  │    ┌────▼────┐                           ┌────▼────┐        │  │   │
│  │  │    │   CXL   │                           │   CXL   │        │  │   │
│  │  │    │ Device  │                           │ Device  │        │  │   │
│  │  │    └─────────┘                           └─────────┘        │  │   │
│  │  │                                                               │  │   │
│  │  └───────────────────────────────────────────────────────────────┘  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                      │                                      │
│                              I2C / I3C / MCTP                               │
│                                      │                                      │
│                              ┌───────▼───────┐                              │
│                              │      BMC      │                              │
│                              │   (OpenBMC)   │                              │
│                              └───────────────┘                              │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## SD BIC 角色

### 主要職責

| 職責 | 說明 |
|------|------|
| **感測器監控** | 讀取溫度、電壓、電流 |
| **電源管理** | 監控電源狀態、順序控制 |
| **Host 通訊** | KCS、eSPI 介面 |
| **BMC 橋接** | MCTP/PLDM、IPMB |
| **擴展管理** | 與 FF/WF BIC 通訊 |
| **韌體更新** | BIOS、CPLD 更新 |

### 通訊關係

```
                            ┌─────────┐
                            │   BMC   │
                            └────┬────┘
                                 │
                    ┌────────────┼────────────┐
                    │  MCTP/PLDM │  I2C/IPMB  │
                    │  over I3C  │            │
                    │            │            │
                ┌───▼────────────▼───┐        │
                │      SD BIC        │        │
                │                    │        │
                │  ┌─────┐  ┌─────┐ │        │
                │  │MCTP │  │IPMI │ │        │
                │  └──┬──┘  └──┬──┘ │        │
                └─────┼────────┼────┘        │
                      │        │             │
          ┌───────────┼────────┼─────────────┤
          │           │        │             │
     ┌────▼────┐ ┌────▼────┐   │        ┌────▼────┐
     │ FF BIC  │ │  Host   │   │        │ Sensors │
     │         │ │  CPU    │   │        │ VR/Temp │
     └────┬────┘ └─────────┘   │        └─────────┘
          │                    │
     ┌────▼────┐          ┌────▼────┐
     │   CXL   │          │  CPLD   │
     │ Device  │          │         │
     └─────────┘          └─────────┘
```

---

## 硬體介面

### I2C 匯流排配置

| Bus | 用途 | 裝置 |
|-----|------|------|
| I2C Bus 0 | 感測器 | TMP75, TMP431 |
| I2C Bus 2 | BMC | IPMB 通訊 |
| I2C Bus 4 | VR | CPU VR, DIMM VR |
| I2C Bus 5 | CPLD | IO 擴展器 |

### I3C 匯流排配置

| Bus | 用途 | 裝置 |
|-----|------|------|
| I3C Bus 0 | BMC + HUB | BMC (Target), I3C HUB |

### GPIO 分組

| 群組 | 用途 |
|------|------|
| GPIO A | 電源狀態 (SLP_S3/S5, PWRGD) |
| GPIO B | 系統狀態 (THERMTRIP, PROCHOT) |
| GPIO C | VR 警報 |
| GPIO D | Throttle 控制 |
| GPIO E~F | BIOS 狀態、其他 |
| GPIO L | Retimer/VR 類型偵測 |

---

## Slot 配置

### Slot 偵測

SD BIC 透過 ADC 電壓偵測所在 Slot 位置：

```c
// plat_class.c
struct _SLOT_EID_MAPPING_TABLE _slot_eid_mapping_table[] = {
    { 0, 0.15, LOWER, SLOT1_EID, SLOT1_PID },
    { 0.15, 0.45, RANGE, SLOT2_EID, SLOT2_PID },
    { 0.45, 0.75, RANGE, SLOT3_EID, SLOT3_PID },
    { 0.75, 1.05, RANGE, SLOT4_EID, SLOT4_PID },
    { 1.05, 1.35, RANGE, SLOT5_EID, SLOT5_PID },
    { 1.35, 1.625, RANGE, SLOT6_EID, SLOT6_PID },
    { 1.625, 1.875, RANGE, SLOT7_EID, SLOT7_PID },
    { 1.875, 2.5, HIGHER, SLOT8_EID, SLOT8_PID },
};
```

### ADC 通道

| 通道 | 用途 |
|------|------|
| ADC Channel 2 | P3V3_STBY 監控 |
| ADC Channel 12 | Blade Config 偵測 |
| ADC Channel 13 | Slot ID 偵測 |

---

## Blade 配置

### 配置類型

```c
// plat_class.h
enum {
    BLADE_CONFIG_UNKNOWN = 0,
    BLADE_CONFIG_T1C,     // Type 1 Compute
    BLADE_CONFIG_T1M,     // Type 1 Memory
};

bool get_blade_config(uint8_t *blade_config)
{
    float voltage = 0.0f;
    get_adc_voltage(ADC_CHANNEL_12, &voltage);

    if (voltage >= 0.0f && voltage <= 1.05f) {
        *blade_config = BLADE_CONFIG_T1C;
    } else if (voltage >= 1.2f && voltage <= 1.7f) {
        *blade_config = BLADE_CONFIG_T1M;
    } else {
        *blade_config = BLADE_CONFIG_UNKNOWN;
        return false;
    }
    return true;
}
```

---

## Board Revision

### 版本偵測

```c
// plat_class.c
bool get_board_rev(uint8_t *board_rev)
{
    I2C_MSG msg = { 0 };
    msg.bus = CPLD_IO_I2C_BUS;
    msg.target_addr = CPLD_IO_I2C_ADDR;
    msg.tx_len = 1;
    msg.rx_len = 1;
    msg.data[0] = CPLD_REG_BOARD_REVISION_ID;
    
    if (i2c_master_read(&msg, 5) != 0) {
        return false;
    }

    *board_rev = msg.data[0] & 0x07;
    return true;
}
```

### 版本定義

```c
enum {
    BOARD_REV_POC = 0,
    BOARD_REV_EVT = 1,
    BOARD_REV_DVT = 2,
    BOARD_REV_PVT = 3,
    BOARD_REV_MP = 4,
};
```

---

## Retimer 類型

### 類型偵測

```c
// plat_class.c
enum {
    RETIMER_TYPE_ASTERALABS = 0,
    RETIMER_TYPE_NO_RETIMER = 1,
    RETIMER_TYPE_KANDOU = 2,
    RETIMER_TYPE_BROADCOM = 3,
};

void init_retimer_type()
{
    uint8_t board_rev = 0;
    get_board_rev(&board_rev);

    if (board_rev <= BOARD_REV_EVT) {
        retimer_type = RETIMER_TYPE_ASTERALABS;
    } else {
        uint8_t rtm_0 = gpio_get(RTM_TYPE_0);
        uint8_t rtm_1 = gpio_get(RTM_TYPE_1);
        
        // [1:0]: 00=Asteralabs, 01=No retimer, 10=Kandou, 11=Broadcom
        retimer_type = (rtm_1 << 1) | rtm_0;
    }
}
```

---

## 與其他 BIC 互動

### FF BIC

- 透過 I3C HUB 連接
- 管理 CXL 擴展卡
- MCTP Endpoint ID 動態分配

### WF BIC

- 透過 I3C HUB 連接
- 管理另一組 CXL 擴展
- 支援雙 CXL 裝置 (CXL1, CXL2)

### EID 分配

```c
// plat_mctp.c
void set_routing_table_eid()
{
    // SD BIC EID 基於 Slot 位置
    // FF BIC = SD EID + 1
    // WF BIC = SD EID + 2
    // FF CXL = SD EID + 3
    // WF CXL1 = SD EID + 4
    // WF CXL2 = SD EID + 5
    
    for (uint8_t i = 2, j = 1; i < ARRAY_SIZE(plat_mctp_route_tbl); i++, j++) {
        plat_mctp_route_tbl[i].endpoint = plat_eid + j;
    }
}
```

---

## 相關文件

- [Architecture](Architecture.md) - 系統架構
- [SlotDetection](SlotDetection.md) - Slot 偵測詳解
- [PlatformInit](PlatformInit.md) - 初始化流程
- [MCTPOverview](MCTPOverview.md) - MCTP 配置

---

*返回 [Home](Home.md)*
