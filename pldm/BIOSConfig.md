# BIOS 配置

本文件說明 PLDM BIOS 配置的詳細設定方式。

---

## JSON 配置檔

BIOS 屬性透過 JSON 檔案定義：

```
oem/<vendor>/configurations/bios/
├── bios_attrs.json           # 通用配置
└── <system_type>/
    └── bios_attrs.json       # 系統特定配置
```

---

## 屬性類型

### Enumeration

```json
{
    "attribute_type": "enum",
    "attribute_name": "BootMode",
    "possible_values": ["Legacy", "UEFI"],
    "default_values": ["UEFI"],
    "help_text": "Boot mode selection",
    "display_name": "Boot Mode"
}
```

### Integer

```json
{
    "attribute_type": "integer",
    "attribute_name": "MemorySize",
    "lower_bound": 0,
    "upper_bound": 65535,
    "default_value": 4096,
    "display_name": "Memory Size (MB)"
}
```

### String

```json
{
    "attribute_type": "string",
    "attribute_name": "AssetTag",
    "string_type": "ASCII",
    "minimum_string_length": 0,
    "maximum_string_length": 64,
    "default_string": ""
}
```

---

## 系統特定配置

啟用系統特定 BIOS 配置：

```bash
meson setup build -Dsystem-specific-bios-json=enabled
```

PLDM 會從 Entity Manager 取得系統類型並載入對應配置。

---

## D-Bus 整合

BIOS 屬性透過 `xyz.openbmc_project.BIOSConfig.Manager` 發布：

| 屬性 | 說明 |
|------|------|
| `BaseBIOSTable` | BIOS 屬性表格 |
| `PendingAttributes` | 等待套用的變更 |

---

*返回 [Home](Home.md)*
