# 端點 API (Endpoint API)

本文說明 mctpd 的 MCTP 端點 D-Bus 介面，包含 OpenBMC 標準介面和 CodeConstruct 擴展介面。

---

## 物件路徑

```
/au/com/codeconstruct/mctp1/networks/<net>/endpoints/<eid>
```

**範例**：
- `/au/com/codeconstruct/mctp1/networks/1/endpoints/8`（本機）
- `/au/com/codeconstruct/mctp1/networks/1/endpoints/10`（遠端）

---

## 介面列表

每個端點物件包含以下介面：

| 介面 | 來源 | 說明 |
|------|------|------|
| `xyz.openbmc_project.MCTP.Endpoint` | OpenBMC | MCTP 端點標準資訊 |
| `xyz.openbmc_project.Common.UUID` | OpenBMC | 端點 UUID |
| `au.com.codeconstruct.MCTP.Endpoint1` | CodeConstruct | 端點控制方法 |

---

## xyz.openbmc_project.MCTP.Endpoint

OpenBMC 定義的 MCTP 端點標準介面，提供端點的基本資訊。

### 介面定義

```
NAME                                   TYPE      SIGNATURE RESULT/VALUE FLAGS
xyz.openbmc_project.MCTP.Endpoint      interface -         -            -
.EID                                   property  y         10           const
.NetworkId                             property  u         1            const
.SupportedMessageTypes                 property  ay        2 0 1        const
```

### 屬性

#### EID

| 項目 | 值 |
|------|-----|
| **型別** | `y` (byte) |
| **存取** | 唯讀 |
| **訊號** | const |

端點識別碼（Endpoint ID）。

**讀取範例**：

```bash
$ busctl get-property au.com.codeconstruct.MCTP1 \
    /au/com/codeconstruct/mctp1/networks/1/endpoints/10 \
    xyz.openbmc_project.MCTP.Endpoint EID
y 10
```

#### NetworkId

| 項目 | 值 |
|------|-----|
| **型別** | `u` (uint32) |
| **存取** | 唯讀 |
| **訊號** | const |

端點所屬的 MCTP 網路 ID。

**讀取範例**：

```bash
$ busctl get-property au.com.codeconstruct.MCTP1 \
    /au/com/codeconstruct/mctp1/networks/1/endpoints/10 \
    xyz.openbmc_project.MCTP.Endpoint NetworkId
u 1
```

#### SupportedMessageTypes

| 項目 | 值 |
|------|-----|
| **型別** | `ay` (array of bytes) |
| **存取** | 唯讀 |
| **訊號** | const |

端點支援的 MCTP 訊息類型列表。

**常見訊息類型**：

| 值 | 類型 |
|----|------|
| 0x00 | MCTP Control |
| 0x01 | PLDM |
| 0x02 | NC-SI |
| 0x03 | Ethernet |
| 0x04 | NVMe-MI |
| 0x05 | SPDM |

**讀取範例**：

```bash
$ busctl get-property au.com.codeconstruct.MCTP1 \
    /au/com/codeconstruct/mctp1/networks/1/endpoints/10 \
    xyz.openbmc_project.MCTP.Endpoint SupportedMessageTypes
ay 2 0 1
```

**輸出說明**：
- `ay` - byte array
- `2` - 陣列長度
- `0 1` - 支援 Control (0) 和 PLDM (1)

---

## xyz.openbmc_project.Common.UUID

OpenBMC 定義的 UUID 介面。

### 介面定義

```
NAME                                   TYPE      SIGNATURE RESULT/VALUE FLAGS
xyz.openbmc_project.Common.UUID        interface -         -            -
.UUID                                  property  s         "..."        emits-change
```

### 屬性

#### UUID

| 項目 | 值 |
|------|-----|
| **型別** | `s` (string) |
| **存取** | 唯讀 |
| **訊號** | PropertyChanged |

端點的 UUID，RFC 4122 格式。

**讀取範例**：

```bash
$ busctl get-property au.com.codeconstruct.MCTP1 \
    /au/com/codeconstruct/mctp1/networks/1/endpoints/10 \
    xyz.openbmc_project.Common.UUID UUID
s "12345678-1234-1234-1234-123456789abc"
```

---

## au.com.codeconstruct.MCTP.Endpoint1

CodeConstruct 定義的端點控制介面。

### 介面定義

```
NAME                                   TYPE      SIGNATURE RESULT/VALUE FLAGS
au.com.codeconstruct.MCTP.Endpoint1    interface -         -            -
.Remove                                method    -         -            -
.SetMTU                                method    u         -            -
.Connectivity                          property  s         "Available"  emits-change
```

### 方法

#### SetMTU

設定此端點路由的 MTU（Maximum Transmission Unit）。

| 項目 | 值 |
|------|-----|
| **輸入** | `u` - MTU 值 (uint32) |
| **輸出** | 無 |

**MTU 範圍**（依傳輸而異）：
- I2C：68-254 bytes

**使用範例**：

```bash
$ busctl call au.com.codeconstruct.MCTP1 \
    /au/com/codeconstruct/mctp1/networks/1/endpoints/10 \
    au.com.codeconstruct.MCTP.Endpoint1 \
    SetMTU u 128
```

**注意**：
- MTU 設定為 0 會使用介面預設 MTU
- MTU 必須在傳輸允許的範圍內

#### Remove

移除此端點。會刪除對應的路由和鄰居條目。

| 項目 | 值 |
|------|-----|
| **輸入** | 無 |
| **輸出** | 無 |

**使用範例**：

```bash
$ busctl call au.com.codeconstruct.MCTP1 \
    /au/com/codeconstruct/mctp1/networks/1/endpoints/10 \
    au.com.codeconstruct.MCTP.Endpoint1 \
    Remove
```

**效果**：
1. 從 mctpd 移除端點記錄
2. 刪除核心路由條目
3. 刪除核心鄰居條目
4. 發出 InterfacesRemoved D-Bus 訊號

### 屬性

#### Connectivity

| 項目 | 值 |
|------|-----|
| **型別** | `s` (string) |
| **存取** | 唯讀（預設）/ 可寫（開發模式） |
| **訊號** | PropertyChanged |

端點的連接狀態。

**可能的值**：

| 值 | 說明 |
|----|------|
| `"Available"` | 端點正常可用 |
| `"Degraded"` | 端點連接降級（無回應） |

**讀取範例**：

```bash
$ busctl get-property au.com.codeconstruct.MCTP1 \
    /au/com/codeconstruct/mctp1/networks/1/endpoints/10 \
    au.com.codeconstruct.MCTP.Endpoint1 Connectivity
s "Available"
```

**連接狀態轉換**：

```
┌───────────────┐                    ┌───────────────┐
│   Available   │ ── 無回應 ──────▶ │   Degraded    │
│               │                    │               │
│  正常通訊     │ ◀── 恢復回應 ────  │  定期輪詢     │
└───────────────┘                    └───────────────┘
```

> [!NOTE]
> 在開發模式下（`unsafe-writable-connectivity=true`），Connectivity 屬性可寫入，用於測試。

---

## 完整內省範例

```bash
$ busctl introspect au.com.codeconstruct.MCTP1 \
    /au/com/codeconstruct/mctp1/networks/1/endpoints/10
NAME                                   TYPE      SIGNATURE RESULT/VALUE             FLAGS
au.com.codeconstruct.MCTP.Endpoint1    interface -         -                        -
.Remove                                method    -         -                        -
.SetMTU                                method    u         -                        -
.Connectivity                          property  s         "Available"              emits-change
org.freedesktop.DBus.Introspectable    interface -         -                        -
.Introspect                            method    -         s                        -
org.freedesktop.DBus.Peer              interface -         -                        -
.GetMachineId                          method    -         s                        -
.Ping                                  method    -         -                        -
org.freedesktop.DBus.Properties        interface -         -                        -
.Get                                   method    ss        v                        -
.GetAll                                method    s         a{sv}                    -
.Set                                   method    ssv       -                        -
.PropertiesChanged                     signal    sa{sv}as  -                        -
xyz.openbmc_project.Common.UUID        interface -         -                        -
.UUID                                  property  s         "12345678-1234-..."      emits-change
xyz.openbmc_project.MCTP.Endpoint      interface -         -                        -
.EID                                   property  y         10                       const
.NetworkId                             property  u         1                        const
.SupportedMessageTypes                 property  ay        2 0 1                    const
```

---

## 程式範例

### 讀取端點資訊

```python
import dbus

bus = dbus.SystemBus()
proxy = bus.get_object(
    'au.com.codeconstruct.MCTP1',
    '/au/com/codeconstruct/mctp1/networks/1/endpoints/10'
)
props = dbus.Interface(proxy, 'org.freedesktop.DBus.Properties')

# 讀取 EID
eid = props.Get('xyz.openbmc_project.MCTP.Endpoint', 'EID')
print(f"EID: {eid}")

# 讀取 UUID
uuid = props.Get('xyz.openbmc_project.Common.UUID', 'UUID')
print(f"UUID: {uuid}")

# 讀取支援的訊息類型
types = props.Get('xyz.openbmc_project.MCTP.Endpoint', 'SupportedMessageTypes')
print(f"Supported Types: {list(types)}")

# 讀取連接狀態
conn = props.Get('au.com.codeconstruct.MCTP.Endpoint1', 'Connectivity')
print(f"Connectivity: {conn}")
```

### 設定 MTU 並監聽狀態變化

```python
import dbus
from dbus.mainloop.glib import DBusGMainLoop
from gi.repository import GLib

DBusGMainLoop(set_as_default=True)
bus = dbus.SystemBus()

def on_properties_changed(interface, changed, invalidated):
    if 'Connectivity' in changed:
        print(f"Connectivity changed to: {changed['Connectivity']}")

proxy = bus.get_object(
    'au.com.codeconstruct.MCTP1',
    '/au/com/codeconstruct/mctp1/networks/1/endpoints/10'
)

# 訂閱屬性變更
proxy.connect_to_signal(
    'PropertiesChanged',
    on_properties_changed,
    dbus_interface='org.freedesktop.DBus.Properties'
)

# 設定 MTU
endpoint = dbus.Interface(proxy, 'au.com.codeconstruct.MCTP.Endpoint1')
endpoint.SetMTU(dbus.UInt32(128))

# 運行主迴圈
loop = GLib.MainLoop()
loop.run()
```

---

## 與 OpenBMC 的整合

mctpd 的端點介面完全相容 OpenBMC 的 MCTP 端點規範，使其他服務可以無縫使用：

### pldmd 整合

```python
# pldmd 可以枚舉 mctpd 發現的端點
bus = dbus.SystemBus()
om = dbus.Interface(
    bus.get_object('au.com.codeconstruct.MCTP1',
                   '/au/com/codeconstruct/mctp1'),
    'org.freedesktop.DBus.ObjectManager'
)

# 取得所有端點
objects = om.GetManagedObjects()
for path, interfaces in objects.items():
    if 'xyz.openbmc_project.MCTP.Endpoint' in interfaces:
        endpoint = interfaces['xyz.openbmc_project.MCTP.Endpoint']
        eid = endpoint['EID']
        types = endpoint['SupportedMessageTypes']
        
        # 檢查是否支援 PLDM (type 1)
        if 1 in types:
            print(f"PLDM endpoint found: EID {eid}")
```

---

## 相關文件

- [DBusOverview](DBusOverview.md) - D-Bus 介面總覽
- [InterfaceAPI](InterfaceAPI.md) - Interface1 / BusOwner1
- [BridgeAPI](BridgeAPI.md) - Bridge1（橋接器端點）
- [OpenBMCIntegration](OpenBMCIntegration.md) - OpenBMC 整合

---

[← 返回首頁](Home.md)
