# 配置指南 (Configuration)

本文說明 mctpd 的配置檔案格式和選項。

---

## 配置檔案

mctpd 使用 TOML 格式的配置檔案。

| 項目 | 值 |
|------|-----|
| **預設路徑** | `/etc/mctpd.conf` |
| **格式** | TOML |
| **命令行覆蓋** | `-c FILE` 或 `--config FILE` |

---

## 完整配置範例

```toml
# /etc/mctpd.conf
# mctpd 配置檔案

# 運作模式：bus-owner 或 endpoint
mode = "bus-owner"

# MCTP 協議配置
[mctp]
# 訊息逾時（毫秒）
message_timeout_ms = 250

# 端點 UUID（可選，預設使用系統 UUID）
# uuid = "21f0f554-7f7c-4211-9ca1-6d0f000ea9e7"

# Bus-owner 模式配置
[bus-owner]
# 動態 EID 分配範圍（包含）
dynamic_eid_range = [8, 254]

# 橋接器最大 EID 池大小
max_pool_size = 15
```

---

## 全域設定

### mode

mctpd 的運作模式。

| 項目 | 值 |
|------|-----|
| **型別** | string |
| **可選值** | `"bus-owner"`, `"endpoint"` |
| **預設值** | `"bus-owner"` |

**說明**：

| 模式 | 說明 |
|------|------|
| `bus-owner` | 作為匯流排擁有者，負責分配 EID |
| `endpoint` | 作為普通端點，接受外部 bus owner 分配的 EID |

**範例**：

```toml
mode = "bus-owner"
```

```toml
mode = "endpoint"
```

---

## [mctp] 區段

MCTP 協議相關配置，適用於所有模式。

### message_timeout_ms

MCTP 控制協議訊息的逾時時間。

| 項目 | 值 |
|------|-----|
| **型別** | integer（毫秒） |
| **預設值** | 250 |
| **建議範圍** | 30-1000 |

**說明**：
- 設定過短可能導致慢速設備無法回應
- 設定過長會降低 mctpd 效能（同步等待）
- 影響 SetupEndpoint 等操作的回應時間

**範例**：

```toml
[mctp]
message_timeout_ms = 250
```

```toml
# 適用於較慢的設備
[mctp]
message_timeout_ms = 500
```

### uuid

本機端點的 UUID。

| 項目 | 值 |
|------|-----|
| **型別** | string（RFC 4122 格式） |
| **預設值** | 系統 UUID |

**說明**：
- 通常不需要設定，mctpd 會自動取得系統 UUID
- 用於回應 Get Endpoint UUID 控制命令
- 必須是有效的 RFC 4122 UUID 格式

**範例**：

```toml
[mctp]
uuid = "21f0f554-7f7c-4211-9ca1-6d0f000ea9e7"
```

> [!NOTE]
> 如果未指定，mctpd 會使用 `sd_id128_get_machine_app_specific()` 產生基於系統的 UUID。

---

## [bus-owner] 區段

Bus-owner 模式特定配置。只在 `mode = "bus-owner"` 時有效。

### dynamic_eid_range

動態 EID 分配範圍。

| 項目 | 值 |
|------|-----|
| **型別** | array of 2 integers [min, max] |
| **預設值** | [8, 254] |
| **有效範圍** | 8-254（MCTP 可分配 EID 範圍） |

**說明**：
- mctpd 從此範圍內分配 EID 給新發現的端點
- 本機 EID 和靜態分配的 EID 可以超出此範圍
- 範圍包含端點（min 和 max 都可分配）

**範例**：

```toml
[bus-owner]
# 預設使用完整範圍
dynamic_eid_range = [8, 254]
```

```toml
[bus-owner]
# 限制範圍以保留部分 EID
dynamic_eid_range = [20, 100]
```

```toml
[bus-owner]
# 僅使用較高的 EID
dynamic_eid_range = [128, 254]
```

### max_pool_size

橋接器可請求的最大 EID 池大小。

| 項目 | 值 |
|------|-----|
| **型別** | integer |
| **預設值** | 15 |

**說明**：
- 限制單一橋接器可獲得的 EID 數量
- 如果橋接器請求超過此值，會被截斷
- 防止單一橋接器耗盡 EID 池

**範例**：

```toml
[bus-owner]
max_pool_size = 15
```

```toml
[bus-owner]
# 允許較大的橋接器池
max_pool_size = 50
```

---

## 配置範例場景

### 標準 Bus-Owner 配置

```toml
# 標準 OpenBMC bus-owner 配置
mode = "bus-owner"

[mctp]
message_timeout_ms = 250

[bus-owner]
dynamic_eid_range = [8, 254]
max_pool_size = 15
```

### Endpoint 模式配置

```toml
# 作為普通端點（非 bus owner）
mode = "endpoint"

[mctp]
message_timeout_ms = 30
```

### 限制 EID 範圍

```toml
# 保留 EID 8-19 給靜態分配
mode = "bus-owner"

[mctp]
message_timeout_ms = 250

[bus-owner]
dynamic_eid_range = [20, 200]
max_pool_size = 15
```

### 大型網路配置

```toml
# 支援多個橋接器的大型網路
mode = "bus-owner"

[mctp]
message_timeout_ms = 500

[bus-owner]
dynamic_eid_range = [8, 254]
max_pool_size = 30
```

---

## 配置檔案位置

### 預設路徑

mctpd 預設讀取：
```
/etc/mctpd.conf
```

### 編譯時配置

預設路徑在編譯時透過 meson 選項設定：

```c
// config.h (generated)
#define MCTPD_CONF_FILE_DEFAULT "/usr/local/etc/mctpd.conf"
```

實際路徑取決於 `--prefix` 和 `--sysconfdir`：

```bash
# 預設安裝
meson setup obj --prefix=/usr --sysconfdir=/etc
# 配置路徑：/etc/mctpd.conf

# 本機開發安裝
meson setup obj --prefix=/usr/local
# 配置路徑：/usr/local/etc/mctpd.conf
```

### 命令行覆蓋

```bash
# 使用指定配置檔案
mctpd -c /path/to/custom.conf
mctpd --config /path/to/custom.conf
```

---

## 驗證配置

### 語法檢查

配置檔案必須是有效的 TOML 格式：

```bash
# 使用 Python 檢查 TOML 語法
python3 -c "import tomllib; tomllib.load(open('/etc/mctpd.conf', 'rb'))"
```

### 啟動驗證

mctpd 在啟動時會進行配置驗證：

```bash
# 查看啟動日誌
journalctl -u mctpd | head -20
```

**常見錯誤**：

| 錯誤 | 說明 | 解決方案 |
|------|------|----------|
| TOML parse error | TOML 語法錯誤 | 檢查格式 |
| Invalid mode | mode 值無效 | 使用 bus-owner 或 endpoint |
| Invalid EID range | EID 範圍無效 | 確保 8 ≤ min ≤ max ≤ 254 |

---

## 配置熱更新

mctpd **不支援**配置熱更新。修改配置後需要重啟服務：

```bash
sudo systemctl restart mctpd
```

---

## 相關文件

- [MctpdDaemon](MctpdDaemon.md) - mctpd 守護程式
- [SystemdIntegration](SystemdIntegration.md) - Systemd 整合
- [QuickStart](QuickStart.md) - 快速入門

---

[← 返回首頁](Home.md)
