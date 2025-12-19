# 故障排除

本文件提供 PLDM 常見問題的診斷與解決方案。

---

## 診斷工具

### pldmtool

```bash
# 確認 PLDM 服務回應
pldmtool base GetTID

# 查詢支援的 Types
pldmtool base GetPLDMTypes

# 取得所有 PDR
pldmtool platform GetPDR -a
```

### journalctl

```bash
# 查看 pldmd 日誌
journalctl -u pldmd -f

# 詳細日誌
journalctl -u pldmd --no-pager | tail -100
```

---

## 常見問題

### 問題 1: pldmd 無法啟動

**症狀**: `systemctl status pldmd` 顯示 failed

**檢查步驟**:
1. 確認 MCTP 服務正常: `systemctl status mctpd`
2. 檢查日誌: `journalctl -u pldmd`
3. 確認配置檔案存在

### 問題 2: GetPDR 回傳空

**可能原因**:
- PDR JSON 配置錯誤
- Host 尚未 Ready
- Entity Manager 未發布 Inventory

**檢查步驟**:
1. 確認 JSON 語法正確
2. 檢查 Host 狀態
3. 確認 D-Bus 物件存在

### 問題 3: BIOS 屬性未顯示

**檢查步驟**:
1. 確認 BIOS JSON 路徑正確
2. 檢查系統類型是否匹配
3. 確認 bios-settings-mgr 執行中

```bash
busctl tree xyz.openbmc_project.BIOSConfigManager
```

---

## 啟用 Verbose 模式

```bash
echo 'PLDMD_ARGS="--verbose"' > /etc/default/pldmd
systemctl restart pldmd
journalctl -u pldmd -f
```

---

## 相關連結

- [pldmd 原始碼](https://github.com/openbmc/pldm)
- [OpenBMC Discord](https://discord.gg/69Km47zH98)

---

*返回 [Home](Home.md)*
