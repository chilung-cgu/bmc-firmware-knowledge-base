# OpenBMC 整合 (OpenBMC Integration)

本文說明 mctpd 如何與 OpenBMC 其他元件整合。

---

## 概述

mctpd 是 OpenBMC 生態系統中的 MCTP 基礎設施元件，提供：

1. MCTP 端點發現和管理
2. D-Bus 介面供其他服務使用
3. 與 OpenBMC 標準介面的相容性

```
┌─────────────────────────────────────────────────────────────────┐
│                      OpenBMC 架構                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    應用層服務                            │   │
│  │                                                          │   │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌────────────┐  │   │
│  │  │ pldmd   │  │ nvmed   │  │ spdmd   │  │ mctp-ctrl  │  │   │
│  │  │ (PLDM)  │  │(NVMe-MI)│  │ (SPDM)  │  │ (control)  │  │   │
│  │  └────┬────┘  └────┬────┘  └────┬────┘  └─────┬──────┘  │   │
│  │       │            │            │             │          │   │
│  └───────┼────────────┼────────────┼─────────────┼──────────┘   │
│          │            │            │             │              │
│    xyz.openbmc_project.MCTP.Endpoint             │              │
│          │            │            │             │              │
│  ┌───────▼────────────▼────────────▼─────────────▼──────────┐   │
│  │                        mctpd                              │   │
│  │                                                           │   │
│  │  D-Bus: au.com.codeconstruct.MCTP1                        │   │
│  │                                                           │   │
│  │  實作：                                                   │   │
│  │  • xyz.openbmc_project.MCTP.Endpoint                      │   │
│  │  • xyz.openbmc_project.Common.UUID                        │   │
│  │  • au.com.codeconstruct.MCTP.*                            │   │
│  │                                                           │   │
│  └───────────────────────────┬───────────────────────────────┘   │
│                              │                                   │
│  ┌───────────────────────────▼───────────────────────────────┐   │
│  │                    Linux Kernel                            │   │
│  │                    MCTP Stack                              │   │
│  └───────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## D-Bus 介面相容性

### OpenBMC 標準介面

mctpd 實作以下 OpenBMC 標準介面：

| 介面                                | 規範位置                 | 說明          |
| ----------------------------------- | ------------------------ | ------------- |
| `xyz.openbmc_project.MCTP.Endpoint` | phosphor-dbus-interfaces | MCTP 端點資訊 |
| `xyz.openbmc_project.Common.UUID`   | phosphor-dbus-interfaces | 端點 UUID     |

**介面定義來源**：

```
https://github.com/openbmc/phosphor-dbus-interfaces/tree/master/yaml/xyz/openbmc_project/MCTP
```

### 使用標準介面的優點

- 其他 OpenBMC 服務可以無縫使用 mctpd 發現的端點
- 符合 OpenBMC 的 D-Bus 物件命名慣例
- 可與 phosphor-dbus-interfaces 產生的綁定程式碼相容

---

## 與 pldmd 整合

pldmd（PLDM daemon）是主要的 mctpd 消費者之一。

### 端點發現

```python
# pldmd 可以這樣發現 PLDM 端點

import dbus

bus = dbus.SystemBus()

# 取得所有 MCTP 端點
om = dbus.Interface(
    bus.get_object('au.com.codeconstruct.MCTP1',
                   '/au/com/codeconstruct/mctp1'),
    'org.freedesktop.DBus.ObjectManager'
)

for path, interfaces in om.GetManagedObjects().items():
    if 'xyz.openbmc_project.MCTP.Endpoint' not in interfaces:
        continue

    endpoint = interfaces['xyz.openbmc_project.MCTP.Endpoint']
    types = endpoint['SupportedMessageTypes']

    # 檢查是否支援 PLDM (type 1)
    if 1 in types:
        eid = endpoint['EID']
        net = endpoint['NetworkId']
        print(f"PLDM endpoint: EID {eid} on network {net}")
```

### MCTP 通訊

pldmd 使用 AF_MCTP socket 與發現的端點通訊：

```c
// 建立 MCTP socket
int sock = socket(AF_MCTP, SOCK_DGRAM, 0);

// 設定目標地址（使用 mctpd 發現的 EID 和 NetworkId）
struct sockaddr_mctp addr = {
    .smctp_family = AF_MCTP,
    .smctp_network = network_id,
    .smctp_addr.s_addr = eid,
    .smctp_type = MCTP_MSG_TYPE_PLDM,  // 0x01
    .smctp_tag = MCTP_TAG_OWNER,
};

// 發送 PLDM 訊息
sendto(sock, pldm_msg, len, 0, (struct sockaddr *)&addr, sizeof(addr));
```

---

## 與 Entity Manager 整合

Entity Manager 可以透過 D-Bus 監控 MCTP 端點：

### 監聽端點新增

```python
# 監聽新端點
bus.add_signal_receiver(
    on_interfaces_added,
    signal_name='InterfacesAdded',
    dbus_interface='org.freedesktop.DBus.ObjectManager',
    bus_name='au.com.codeconstruct.MCTP1',
    path='/au/com/codeconstruct/mctp1'
)

def on_interfaces_added(path, interfaces):
    if 'xyz.openbmc_project.MCTP.Endpoint' in interfaces:
        # 新端點已發現
        eid = interfaces['xyz.openbmc_project.MCTP.Endpoint']['EID']
        print(f"New endpoint discovered: EID {eid}")
```

---

## 訊息類型註冊

應用程式可以向 mctpd 註冊其支援的訊息類型：

### 架構

```
┌─────────────────────────────────────────────────────────────────┐
│                    訊息類型註冊                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────┐                                                │
│  │   pldmd     │ ──RegisterTypeSupport(1, versions)──┐          │
│  └─────────────┘                                     │          │
│                                                      ▼          │
│  ┌─────────────┐                           ┌─────────────────┐  │
│  │   nvmed     │ ──RegisterTypeSupport(4)──▶│     mctpd       │  │
│  └─────────────┘                           │                 │  │
│                                            │  訊息類型表：   │  │
│  ┌─────────────┐                           │  • 0: Control   │  │
│  │   spdmd     │ ──RegisterTypeSupport(5)──▶│  • 1: PLDM     │  │
│  └─────────────┘                           │  • 4: NVMe-MI  │  │
│                                            │  • 5: SPDM     │  │
│                                            └─────────────────┘  │
│                                                      │          │
│                                                      ▼          │
│                                            Get Message Type     │
│                                            Support 回應         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 註冊 API

```bash
# 註冊 PLDM 支援
busctl call au.com.codeconstruct.MCTP1 /au/com/codeconstruct/mctp1 \
    au.com.codeconstruct.MCTP1 \
    RegisterTypeSupport yau 1 2 0xF1F0F000 0xF1F1F000
```

### 註冊生命週期

- 註冊在 D-Bus 連線斷開時自動取消
- 服務重啟會重新註冊
- mctpd 會更新其 Get Message Type Support 回應

---

## 與 phosphor-dbus-interfaces 的關係

### MCTP.Endpoint.yaml

```yaml
# phosphor-dbus-interfaces/yaml/xyz/openbmc_project/MCTP/Endpoint.yaml
description: >
  This interface represents an MCTP endpoint.

properties:
  - name: NetworkId
    type: uint32
    description: >
      The network id of this endpoint.

  - name: EID
    type: byte
    description: >
      The Endpoint ID of this endpoint.

  - name: SupportedMessageTypes
    type: array[byte]
    description: >
      The MCTP message types supported by this endpoint.
```

### mctpd 實作

mctpd 完全實作此介面：

```c
// 在 mctpd.c 中（簡化說明，實際 getter 均使用同一函式 bus_endpoint_get_prop）
static const sd_bus_vtable bus_endpoint_obmc_vtable[] = {
    SD_BUS_VTABLE_START(0),
    SD_BUS_PROPERTY("NetworkId", "u", bus_endpoint_get_prop, 0,
                    SD_BUS_VTABLE_PROPERTY_CONST),
    SD_BUS_PROPERTY("EID", "y", bus_endpoint_get_prop, 0,
                    SD_BUS_VTABLE_PROPERTY_CONST),
    SD_BUS_PROPERTY("SupportedMessageTypes", "ay", bus_endpoint_get_prop, 0,
                    SD_BUS_VTABLE_PROPERTY_CONST),
    SD_BUS_VTABLE_END,
};
```

---

## Yocto/OpenBMC 整合

### Recipe 範例

```bitbake
# meta-phosphor/recipes-protocols/mctp/mctp_git.bb

SUMMARY = "MCTP userspace tools"
DESCRIPTION = "MCTP daemon and utilities for OpenBMC"
HOMEPAGE = "https://github.com/CodeConstruct/mctp"
LICENSE = "GPL-2.0-only"

DEPENDS = "systemd"
RDEPENDS:${PN} = "libsystemd"

SRC_URI = "git://github.com/CodeConstruct/mctp.git;branch=main;protocol=https"
SRCREV = "..."

S = "${WORKDIR}/git"

inherit meson systemd

SYSTEMD_SERVICE:${PN} = "mctpd.service"
SYSTEMD_AUTO_ENABLE = "enable"

do_install:append() {
    install -d ${D}${sysconfdir}
    install -m 0644 ${S}/conf/mctpd.conf ${D}${sysconfdir}/
}
```

### 配置覆蓋

不同平台可能需要不同配置：

```bitbake
# 平台特定 bbappend
# meta-platform/recipes-protocols/mctp/mctp_%.bbappend

FILESEXTRAPATHS:prepend := "${THISDIR}/files:"

SRC_URI += "file://mctpd.conf"

do_install:append() {
    install -m 0644 ${WORKDIR}/mctpd.conf ${D}${sysconfdir}/
}
```

---

## 除錯與監控

### 監控端點變化

```bash
# 監控 mctpd D-Bus 訊號
busctl monitor au.com.codeconstruct.MCTP1
```

### 列出所有端點

```bash
#!/bin/bash
# 列出所有 MCTP 端點及其支援的訊息類型

busctl call au.com.codeconstruct.MCTP1 /au/com/codeconstruct/mctp1 \
    org.freedesktop.DBus.ObjectManager GetManagedObjects | \
    python3 -c "
import sys
import re

# 簡單解析 busctl 輸出
# 實際應用中應使用 D-Bus 綁定
for line in sys.stdin:
    if 'endpoints' in line:
        print(line.strip())
"
```

---

## 相關連結

- [phosphor-dbus-interfaces MCTP](https://github.com/openbmc/phosphor-dbus-interfaces/tree/master/yaml/xyz/openbmc_project/MCTP)
- [OpenBMC MCTP 設計文件](https://github.com/openbmc/docs/blob/master/designs/mctp/mctp-kernel.md)
- [pldm](https://github.com/openbmc/pldm) - PLDM 守護程式

---

## 相關文件

- [DBusOverview](DBusOverview.md) - D-Bus 介面總覽
- [EndpointAPI](EndpointAPI.md) - 端點 API
- [SystemdIntegration](SystemdIntegration.md) - Systemd 整合

---

[← 返回首頁](Home.md)
