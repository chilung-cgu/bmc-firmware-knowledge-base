# 設定與建置

本文件說明 PLDM 專案的建置選項與設定方式。

---

## 建置需求

- Meson (>= 0.58)
- Ninja
- C++20 相容編譯器

---

## 基本建置

```bash
# 設定建置目錄
meson setup build

# 編譯
meson compile -C build

# 執行測試
meson test -C build
```

---

## Meson 選項

### 功能選項

| 選項 | 預設 | 說明 |
|------|------|------|
| `oem-ibm` | disabled | 啟用 IBM OEM 支援 |
| `system-specific-bios-json` | disabled | 系統特定 BIOS 配置 |
| `fw-update-pkg-inotify` | disabled | inotify 韌體更新監控 |
| `transport-implementation` | mctp-demux | MCTP 傳輸實作 |

### 範例

```bash
# 啟用 IBM OEM
meson setup build -Doem-ibm=enabled

# 啟用 inotify 韌體更新
meson setup build -Dfw-update-pkg-inotify=enabled

# 啟用系統特定 BIOS
meson setup build -Dsystem-specific-bios-json=enabled
```

---

## Verbose 模式

執行時啟用詳細輸出：

```bash
# 設定環境變數
echo 'PLDMD_ARGS="--verbose"' > /etc/default/pldmd
systemctl restart pldmd

# 停用
rm /etc/default/pldmd
systemctl restart pldmd
```

---

## 相關文件

- [Pldmd](Pldmd.md) - pldmd 守護程式

---

*返回 [Home](Home.md)*
