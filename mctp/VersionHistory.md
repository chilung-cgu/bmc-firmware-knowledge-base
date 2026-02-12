# 版本歷史 (Version History)

本文記錄 CodeConstruct/mctp 專案的版本變更。

---

## 版本概覽

| 版本 | 發布日期 | 重要變更 |
|------|----------|----------|
| v2.4 | 2025-10-28 | VDM 類型支援 |
| v2.3 | 2025-10-22 | Endpoint 模式增強 |
| v2.2 | 2025-07-28 | Network1 介面、mctp-bench |
| v2.1 | 2024-12-16 | IID 檢查、MTU 改進 |
| v2.0 | 2024-09-19 | D-Bus 介面重構、配置檔案 |

---

## Unreleased（v2.4 之後）

### 新增

- **Bridged endpoint polling**：Bus-owner 可定期輪詢橋接器下游端點，確認端點可達性
  - 新增 `endpoint_poll_ms` 配置項（`[bus-owner]` 區段），有效範圍 2500-10000ms
  - 設為 0 時禁用（預設）
  - 符合 DSP0236 第 8.17.6 節的 EID 回收機制

- **Endpoint recovery 機制**：端點失聯後的自動恢復流程
  - 新增 `docs/endpoint-recovery.md` 文件

### 變更

- `conf/mctpd.conf` 新增 `endpoint_poll_ms = 0` 預設值

---

## v2.4 (2025-10-28)

### 新增

- **VDM 類型支援**：mctpd 現在實作 Get Vendor Defined Message Support 控制協議命令
- **VDM 註冊 API**：應用程式可透過 D-Bus 註冊 vendor-defined 訊息類型

### 修復

- 修復 mctp-bench 在 musl libc 上的編譯問題

---

## v2.3 (2025-10-22)

### 新增

- **mctp-bench 增強**：支援 "request receive" 模式
  ```bash
  mctp-bench recv eid <remote-eid>
  ```
- **Bus-owner 配置區段**：新增 `[bus-owner]` 配置區段
- **mctpd.conf 文件**：新增配置選項文件
- **動態 EID 範圍配置**：`dynamic_eid_range` 選項
- **Endpoint 模式增強**：
  - 處理 Set Endpoint ID 訊息
  - 支援 MCTP 橋接器的 EID 池請求
  - 處理發現流程（Prepare/Endpoint Discovery）
- **訊息類型支援**：`RegisterTypeSupport` D-Bus 方法
- **Network1.LearnEndpoint**：發布前驗證端點存在

### 修復

- 修復介面刪除時端點可能殘留的問題
- 修復 linux/mctp.h 相容性問題
- 修復傾印 netlink 事件時的陣列溢出

---

## v2.2 (2025-07-28)

### 新增

- **mctp-bench**：新的效能測試工具
  ```bash
  mctp-bench send <eid>
  mctp-bench recv
  ```
- **au.com.codeconstruct.MCTP.Network1 介面**：
  - `LocalEIDs` 屬性
  - `LearnEndpoint` 方法
- **獨立測試環境**：fake mctp 環境可獨立運行
- **NetworkId 屬性**：在 Interface1 上新增
- **閘道路由支援**：`mctp route add <eid> gw <gateway>`
- **範圍路由**：`mctp route add <min>-<max> via <dev>`
- **mctp 工具測試**：新增覆蓋測試
- **介面重命名處理**：自動更新 D-Bus 物件

### 變更

- 測試現在啟用 Address Sanitizer
- mctp neigh 硬體地址格式改進
- mctp-bench 改為每 2 秒報告

### 移除

- mctpd 測試模式 (`-N`)：改用 Python wrapper
- mctp-bench/req/echo 改用 vendor-defined 訊息類型

### 修復

- 修復 peer 指標在 realloc 後失效的問題
- 修復 Netlink 介面刪除處理
- 修復作為回應者時的 IID 處理
- 正確處理僅含 CC 的錯誤回應
- 確保錯誤回應資料初始化

---

## v2.1 (2024-12-16)

### 變更

- **IID 檢查強制執行**：防止延遲或無效回應的異常行為
- **初始路由 MTU**：設為介面最小 MTU（改善相容性）
- **硬體地址格式**：改善非 1-byte 地址的顯示

### 修復

- 修復 musl 編譯問題（AF_MCTP 定義）
- 修復標頭檔包含問題
- 修復 peer 訊息類型資料設定錯誤
- Interface 物件前綴統一為 au.com.codeconstruct

---

## v2.0 (2024-09-19)

### 重大變更

> [!IMPORTANT]
> v2.0 對 D-Bus 介面進行了重大重構。

#### D-Bus 介面變更

- **NetworkID 型別**：從 `i` (int32) 改為 `u` (uint32)
- **介面前綴標準化**：使用 `au.com.codeconstruct` 和 `xyz.openbmc_project`
- **物件路徑重構**：
  ```
  舊：/xyz/openbmc_project/mctp/...
  新：/au/com/codeconstruct/mctp1/...
  ```
- **Bus-owner 方法位置**：移至 Interface 物件
- **方法簽名變更**：不再需要介面名稱作為第一個參數

### 新增

- **端點恢復**：支援失聯端點的自動恢復
- **nil UUID 恢復**：開發用選項
- **可寫 Connectivity**：開發用選項
- **AssignEndpointStatic**：靜態 EID 分配
- **新測試框架**：控制協議測試
- **配置檔案**：預設 /etc/mctpd.conf
- **Interface D-Bus 物件**：新增 `/mctp1/interfaces/<name>`

### 修復

- 修復 EID 衝突處理
- 修復控制 socket 讀取錯誤偵測

---

## 版本相容性

### API 相容性

| 版本 | 與前版相容 | 說明 |
|------|------------|------|
| v2.4 | ✅ 是 | 僅新增功能 |
| v2.3 | ✅ 是 | 僅新增功能 |
| v2.2 | ✅ 是 | 僅新增功能 |
| v2.1 | ✅ 是 | 僅修復和改進 |
| v2.0 | ❌ 否 | D-Bus 介面重構 |

### 升級指南

#### 從 v1.x 升級到 v2.x

1. **更新 D-Bus 客戶端**：
   - 服務名稱：`au.com.codeconstruct.MCTP1`
   - 根路徑：`/au/com/codeconstruct/mctp1`

2. **更新方法呼叫**：
   - SetupEndpoint 等方法移至 Interface 物件
   - 不再需要介面名稱參數

3. **更新屬性型別**：
   - NetworkID 從 int32 改為 uint32

#### 從 v2.x 升級到 v2.y

通常無需變更，新增功能向後相容。

---

## CHANGELOG 格式

本專案遵循 [Keep a Changelog](https://keepachangelog.com/en/1.0.0/) 格式。

---

## 參考來源

完整變更記錄請參閱：
- [GitHub CHANGELOG.md](https://github.com/CodeConstruct/mctp/blob/main/CHANGELOG.md)
- [GitHub Releases](https://github.com/CodeConstruct/mctp/releases)

---

## 相關文件

- [Home](Home.md) - 首頁
- [QuickStart](QuickStart.md) - 快速入門
- [DBusOverview](DBusOverview.md) - D-Bus 介面

---

[← 返回首頁](Home.md)
