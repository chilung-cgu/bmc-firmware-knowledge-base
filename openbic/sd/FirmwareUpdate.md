# FirmwareUpdate（韌體更新）

本文說明 OpenBIC 的韌體更新機制，包括 PLDM Firmware Update 與 IPMI 更新方式。

---

## 更新方式概述

OpenBIC 支援多種韌體更新方式：

| 方式 | 協議 | 說明 |
|------|------|------|
| **PLDM FW Update** | PLDM Type 5 | DMTF 標準韌體更新 |
| **IPMI OEM** | OEM_1S_FW_UPDATE | 傳統 IPMI 更新 |

---

## PLDM Firmware Update

### 概述

PLDM Firmware Update (Type 5) 是 DMTF 定義的標準韌體更新協議。

### 狀態機

```
┌──────────────┐
│    IDLE      │
└──────┬───────┘
       │ Request Update
       ▼
┌──────────────┐
│LEARN COMPS   │◄────────────────────┐
└──────┬───────┘                     │
       │ Pass Component Table        │
       ▼                             │
┌──────────────┐                     │
│  READY XFER  │                     │
└──────┬───────┘                     │
       │ Update Component            │
       ▼                             │
┌──────────────┐                     │
│  DOWNLOAD    │                     │ Cancel
└──────┬───────┘                     │
       │ Transfer Complete           │
       ▼                             │
┌──────────────┐                     │
│   VERIFY     │                     │
└──────┬───────┘                     │
       │ Verify Complete             │
       ▼                             │
┌──────────────┐                     │
│    APPLY     ├────────────────────►┘
└──────┬───────┘
       │ Activate Firmware
       ▼
┌──────────────┐
│  ACTIVATE    │
└──────────────┘
```

### 支援的命令

| 命令 | 代碼 | 說明 |
|------|------|------|
| QueryDeviceIdentifiers | 0x01 | 查詢裝置識別碼 |
| GetFirmwareParameters | 0x02 | 取得韌體參數 |
| RequestUpdate | 0x10 | 請求更新 |
| PassComponentTable | 0x13 | 傳送元件表 |
| UpdateComponent | 0x14 | 更新元件 |
| TransferComplete | 0x20 | 傳輸完成 |
| VerifyComplete | 0x21 | 驗證完成 |
| ApplyComplete | 0x22 | 套用完成 |
| ActivateFirmware | 0x1A | 啟動韌體 |
| CancelUpdate | 0x1D | 取消更新 |

### 元件定義

```c
// plat_pldm_fw_update.c
enum pldm_comp_identifier {
    COMP_BIC      = 1,
    COMP_BIOS     = 2,
    COMP_CPLD     = 3,
    COMP_VR_CPU0  = 4,
    COMP_VR_CPU1  = 5,
    COMP_RETIMER  = 6,
};
```

---

## PLDM FW Update 實作

### 裝置識別碼查詢

```c
// plat_pldm_device_identifier.c
pldm_fw_update_info_t *pldm_get_fw_update_info(void)
{
    static pldm_fw_update_info_t fw_update_info = {
        .uuid = { ... },
        .firmware_device_package_data = NULL,
        .firmware_device_package_data_length = 0,
    };
    
    return &fw_update_info;
}

// 裝置描述符
pldm_descriptor_t plat_fw_desc[] = {
    { .title_string = "Yosemite V4 SD BIC",
      .type = PLDM_FWUP_DESCRIPTOR_VENDOR_DEFINED,
      // ...
    },
};
```

### 韌體參數

```c
// plat_pldm_fw_update.c
pldm_fw_component_info_t plat_comp_info[] = {
    {
        .comp_classification = COMP_CLASSIFICATION_SOFTWARE,
        .comp_identifier = COMP_BIC,
        .comp_version = { BIC_FW_YEAR_MSB, BIC_FW_YEAR_LSB, BIC_FW_WEEK, BIC_FW_VER },
        .comp_activation_methods = PLDM_COMP_ACTIVATION_SYSTEM_REBOOT,
    },
    {
        .comp_classification = COMP_CLASSIFICATION_FIRMWARE,
        .comp_identifier = COMP_BIOS,
        .comp_activation_methods = PLDM_COMP_ACTIVATION_SYSTEM_REBOOT,
    },
    // ... 更多元件
};
```

### 更新處理

```c
// pldm_firmware_update.c
uint8_t pldm_fw_update_component(pldm_msg *msg)
{
    struct pldm_update_component_req *req = 
        (struct pldm_update_component_req *)msg->buf;
    
    uint16_t comp_id = req->component_identifier;
    uint32_t comp_size = req->component_image_size;
    
    // 準備接收韌體資料
    switch (comp_id) {
    case COMP_BIC:
        // 初始化 BIC Flash 更新
        init_bic_update(comp_size);
        break;
    case COMP_BIOS:
        // 初始化 BIOS SPI Flash 更新
        init_bios_update(comp_size);
        break;
    // ...
    }
    
    return PLDM_SUCCESS;
}
```

---

## IPMI Firmware Update

### OEM_1S_FW_UPDATE

```c
// oem_1s_handler.c
__weak OEM_1S_FW_UPDATE(ipmi_msg *msg)
{
    CHECK_NULL_ARG(msg);
    
    if (msg->data_len < 7) {
        msg->completion_code = CC_INVALID_LENGTH;
        return;
    }
    
    uint8_t target = msg->data[0];
    uint32_t offset = (msg->data[1] << 24) | (msg->data[2] << 16) |
                      (msg->data[3] << 8) | msg->data[4];
    uint16_t length = (msg->data[5] << 8) | msg->data[6];
    uint8_t *data = &msg->data[7];
    
    switch (target) {
    case UPDATE_TARGET_BIC:
        if (offset + length > BIC_UPDATE_MAX_OFFSET) {
            msg->completion_code = CC_INVALID_DATA_FIELD;
            return;
        }
        do_bic_update(offset, length, data);
        break;
        
    case UPDATE_TARGET_BIOS:
        if (offset + length > BIOS_UPDATE_MAX_OFFSET) {
            msg->completion_code = CC_INVALID_DATA_FIELD;
            return;
        }
        do_bios_update(offset, length, data);
        break;
        
    default:
        msg->completion_code = CC_INVALID_DATA_FIELD;
        return;
    }

    msg->completion_code = CC_SUCCESS;
    msg->data_len = 0;
}
```

### 更新目標

```c
// oem_1s_handler.h
enum {
    UPDATE_TARGET_BIC   = 0,
    UPDATE_TARGET_BIOS  = 1,
    UPDATE_TARGET_CPLD  = 2,
    UPDATE_TARGET_VR    = 3,
};
```

---

## Flash 操作

### BIC 韌體寫入

```c
// util_spi.c
int do_bic_update(uint32_t offset, uint16_t length, uint8_t *data)
{
    const struct device *flash_dev = device_get_binding("spi_flash");
    
    // 擦除 sector (如需)
    if ((offset % FLASH_SECTOR_SIZE) == 0) {
        flash_erase(flash_dev, offset, FLASH_SECTOR_SIZE);
    }
    
    // 寫入資料
    int ret = flash_write(flash_dev, offset, data, length);
    if (ret != 0) {
        LOG_ERR("Flash write failed at offset 0x%x", offset);
        return ret;
    }
    
    return 0;
}
```

### BIOS SPI Flash

```c
// bios_update.c
int do_bios_update(uint32_t offset, uint16_t length, uint8_t *data)
{
    // 透過 SPI 控制器存取 BIOS SPI Flash
    // 需要先將控制權從 Host 切換到 BIC
    
    // 切換 SPI Mux
    gpio_set(BIC_JTAG_SEL_R, GPIO_HIGH);
    
    // 執行 SPI 寫入
    int ret = spi_flash_write(BIOS_SPI_BUS, offset, data, length);
    
    // 切換回 Host
    gpio_set(BIC_JTAG_SEL_R, GPIO_LOW);
    
    return ret;
}
```

---

## 韌體驗證

### SHA256 計算

```c
#ifdef CONFIG_CRYPTO_ASPEED
__weak OEM_1S_GET_FW_SHA256(ipmi_msg *msg)
{
    uint8_t target = msg->data[0];
    uint32_t offset = (msg->data[1] << 24) | ...;
    uint32_t length = (msg->data[5] << 24) | ...;
    
    uint8_t hash[32];
    
    switch (target) {
    case UPDATE_TARGET_BIC:
        calculate_sha256(BIC_FLASH_BASE + offset, length, hash);
        break;
    // ...
    }
    
    memcpy(msg->data, hash, 32);
    msg->data_len = 32;
    msg->completion_code = CC_SUCCESS;
}
#endif
```

---

## 版本查詢

### 取得韌體版本

```c
__weak OEM_1S_GET_FW_VERSION(ipmi_msg *msg)
{
    uint8_t target = msg->data[0];
    
    switch (target) {
    case FW_BIC:
        msg->data[0] = BIC_FW_YEAR_MSB;
        msg->data[1] = BIC_FW_YEAR_LSB;
        msg->data[2] = BIC_FW_WEEK;
        msg->data[3] = BIC_FW_VER;
        msg->data_len = 4;
        break;
        
    case FW_CPLD:
        get_cpld_version(&msg->data[0]);
        msg->data_len = 4;
        break;
        
    case FW_VR:
        get_vr_version(&msg->data[0]);
        msg->data_len = 8;
        break;
    }
    
    msg->completion_code = CC_SUCCESS;
}
```

---

## 更新後啟動

### 系統重啟

```c
void activate_firmware(uint8_t component)
{
    switch (component) {
    case COMP_BIC:
        // BIC 重啟
        LOG_INF("Rebooting BIC...");
        k_msleep(100);
        sys_reboot(SYS_REBOOT_COLD);
        break;
        
    case COMP_BIOS:
        // 通知 BMC 重啟 Host
        inform_bmc_to_power_cycle();
        break;
    }
}
```

---

## 相關文件

- [PLDMOverview](PLDMOverview.md) - PLDM 協議
- [IPMIOverview](IPMIOverview.md) - IPMI 命令
- [YV4SDPlatform](YV4SDPlatform.md) - 平台概述

---

*返回 [Home](Home.md)*
