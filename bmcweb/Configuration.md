# 編譯與配置選項

本文件說明 bmcweb 的編譯配置和 Meson 選項。

---

## 📋 目錄

1. [建置系統](#建置系統)
2. [編譯步驟](#編譯步驟)
3. [Meson 選項](#meson-選項)
4. [BitBake 配置](#bitbake-配置)

---

## 建置系統

bmcweb 使用 **Meson** 建置系統：

- **Meson** - 建置定義
- **Ninja** - 實際編譯

### 依賴項

| 依賴 | 說明 |
|------|------|
| Boost | Asio, Beast, URL, JSON |
| OpenSSL | TLS 支援 |
| sdbusplus | D-Bus 整合 |
| nlohmann/json | JSON 處理 |
| libpam | 認證 |

---

## 編譯步驟

### 基本編譯

```bash
# 配置
meson setup builddir

# 編譯
ninja -C builddir

# 安裝
ninja -C builddir install
```

### 使用特定選項

```bash
meson setup builddir \
    -Dredfish-aggregation=enabled \
    -Drest=enabled \
    -Dkvm=enabled
```

### 重新配置

```bash
meson setup builddir --reconfigure \
    -Doption=value
```

---

## Meson 選項

### 功能開關

| 選項 | 說明 | 預設 |
|------|------|------|
| `redfish` | Redfish 支援 | enabled |
| `rest` | D-Bus REST API | disabled |
| `kvm` | KVM 功能 | disabled |
| `vm-websocket` | Virtual Media WebSocket | disabled |
| `host-serial-socket` | Serial Console | disabled |

### 認證選項

| 選項 | 說明 | 預設 |
|------|------|------|
| `basic-auth` | Basic 認證 | enabled |
| `session-auth` | Session 認證 | enabled |
| `xtoken-auth` | X-Auth-Token | enabled |
| `cookie-auth` | Cookie 認證 | enabled |
| `mutual-tls-auth` | mTLS 認證 | enabled |

### Redfish 選項

| 選項 | 說明 | 預設 |
|------|------|------|
| `redfish-aggregation` | Redfish 聚合 | disabled |
| `redfish-new-powersubsystem` | 新版 Power API | disabled |
| `redfish-allow-deprecated-power-thermal` | 舊版 Power/Thermal | enabled |
| `redfish-dbus-log` | D-Bus 日誌整合 | disabled |

### 開發選項

| 選項 | 說明 | 預設 |
|------|------|------|
| `insecure-tftp-update` | 不安全的 TFTP 更新 | disabled |
| `insecure-disable-ssl` | 停用 SSL | disabled |
| `insecure-disable-auth` | 停用認證 | disabled |

> [!CAUTION]
> `insecure-*` 選項僅供開發使用，絕對不要在生產環境啟用！

### 雜項選項

| 選項 | 說明 | 預設 |
|------|------|------|
| `https_port` | HTTPS 端口 | 443 |
| `http_port` | HTTP 端口 | 80 |
| `tests` | 編譯測試 | enabled |

---

## BitBake 配置

### local.conf 配置

在 OpenBMC 建置中，透過 `local.conf` 配置 bmcweb：

```bash
# 啟用功能
EXTRA_OEMESON:pn-bmcweb:append = " \
    -Dredfish-aggregation='enabled' \
    -Drest='enabled' \
    -Dkvm='enabled' \
"

# 停用功能
EXTRA_OEMESON:pn-bmcweb:append = " \
    -Dbasic-auth='disabled' \
"
```

### 機器配置

也可在機器配置檔案中設定：

```bash
# meta-xxx/conf/machine/xxx.conf
EXTRA_OEMESON:pn-bmcweb:append = "-Dkvm='enabled'"
```

### 查詢可用選項

```bash
# 在 bmcweb 目錄中
meson configure builddir
```

---

## 配置範例

### 最小配置

只有 Redfish，無其他功能：

```bash
meson setup builddir \
    -Drest=disabled \
    -Dkvm=disabled \
    -Dvm-websocket=disabled \
    -Dhost-serial-socket=disabled
```

### 完整功能

```bash
meson setup builddir \
    -Dredfish=enabled \
    -Drest=enabled \
    -Dkvm=enabled \
    -Dvm-websocket=enabled \
    -Dhost-serial-socket=enabled \
    -Dredfish-aggregation=enabled
```

### 開發環境

```bash
meson setup builddir \
    -Dtests=enabled \
    -Drest=enabled \
    --buildtype=debug
```

---

## 子專案管理

bmcweb 使用 Meson wrap 管理依賴：

```
subprojects/
├── boost.wrap
├── googletest.wrap
├── nlohmann_json.wrap
└── ...
```

### 更新子專案

```bash
meson subprojects update
```

### 使用系統庫

若系統已安裝依賴，Meson 會優先使用系統庫。

---

## 相關文件

- [CodeOrganization](CodeOrganization.md) - 原始碼結構
- [Testing](Testing.md) - 測試方法
- [Architecture](Architecture.md) - 架構概述

---

*最後更新：2025-12-19*
