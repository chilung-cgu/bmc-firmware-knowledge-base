# Troubleshooting - 故障排除

本文件提供使用 phosphor-dbus-interfaces 時常見問題的解決方案。

---

## 🛠️ 建置問題

### 找不到 sdbus++ 工具

**症狀**：
```
Program 'sdbus++' not found or not executable
```

**解決方案**：
```bash
# 安裝 sdbusplus 工具
git clone https://github.com/openbmc/sdbusplus.git
cd sdbusplus/tools
pip install .

# 確認安裝
which sdbus++
sdbus++ --help
```

### 缺少 Python 模組

**症狀**：
```
ModuleNotFoundError: No module named 'mako'
```

**解決方案**：
```bash
pip install mako inflection pyyaml
```

### YAML 語法錯誤

**症狀**：
```
yaml.scanner.ScannerError: while scanning a simple key
```

**解決方案**：

1. 檢查縮排是否一致（使用空格，不要混用 Tab）
2. 檢查特殊字元是否需要引號
3. 驗證 YAML 語法：

```bash
python3 -c "import yaml; yaml.safe_load(open('file.interface.yaml'))"
```

### Meson 設定失敗

**症狀**：
```
ERROR: Dependency "sdbusplus" not found
```

**解決方案**：
```bash
# 安裝 sdbusplus 函式庫
cd sdbusplus
meson builddir && ninja -C builddir
sudo ninja -C builddir install

# 更新 pkg-config 路徑
export PKG_CONFIG_PATH=/usr/local/lib/pkgconfig:$PKG_CONFIG_PATH
```

---

## 🔍 執行時期問題

### 介面不存在

**症狀**：
```
Error: org.freedesktop.DBus.Error.UnknownInterface
Interface "xyz.openbmc_project.Sensor.Value" not found
```

**可能原因**：
1. 服務未運行
2. 物件路徑錯誤
3. 服務未實作該介面

**解決方案**：

```bash
# 1. 確認服務運行中
systemctl status xyz.openbmc_project.HwmonTempSensor.service

# 2. 列出所有相關服務
busctl --list | grep openbmc

# 3. 使用 ObjectMapper 查詢正確的服務和路徑
busctl call xyz.openbmc_project.ObjectMapper \
    /xyz/openbmc_project/ObjectMapper \
    xyz.openbmc_project.ObjectMapper \
    GetSubTree sias "/" 0 1 "xyz.openbmc_project.Sensor.Value"
```

### 屬性不存在

**症狀**：
```
Error: org.freedesktop.DBus.Error.UnknownProperty
Property "Value" not found
```

**解決方案**：

```bash
# 列出物件的所有介面和屬性
busctl introspect <SERVICE> <PATH>

# 範例
busctl introspect xyz.openbmc_project.HwmonTempSensor \
    /xyz/openbmc_project/sensors/temperature/CPU_Core
```

### 權限被拒

**症狀**：
```
Error: org.freedesktop.DBus.Error.AccessDenied
```

**解決方案**：

1. 檢查 D-Bus 策略設定
2. 確認使用者權限
3. 某些屬性可能需要特權存取

```bash
# 以 root 執行
sudo busctl set-property ...
```

---

## 📊 列舉錯誤

### 無效的列舉字串

**症狀**：
```
Error: xyz.openbmc_project.sdbusplus.Error.InvalidEnumString
```

**原因**：提供的字串不符合列舉格式

**解決方案**：

使用完整限定的列舉字串：

```bash
# ❌ 錯誤
busctl set-property ... s "On"

# ✅ 正確
busctl set-property ... s "xyz.openbmc_project.State.Host.Transition.On"
```

### 列舉值未知

**解決方案**：

查詢介面文件或 YAML 定義以獲取有效值：

```bash
# 查看 YAML 定義中的列舉值
cat yaml/xyz/openbmc_project/State/Host.interface.yaml
```

---

## 🔗 關聯問題

### 找不到關聯端點

**症狀**：
關聯查詢回傳空結果

**解決方案**：

1. 確認關聯已正確建立
2. 檢查關聯路徑

```bash
# 查看物件的關聯定義
busctl get-property <SERVICE> <PATH> \
    xyz.openbmc_project.Association.Definitions Associations

# 查看關聯端點
busctl get-property xyz.openbmc_project.ObjectMapper \
    <PATH>/sensors \
    xyz.openbmc_project.Association endpoints
```

### ObjectMapper 查詢失敗

**症狀**：
```
Error: xyz.openbmc_project.Common.Error.ResourceNotFound
```

**可能原因**：
1. 指定的路徑不存在
2. 沒有物件實作指定的介面
3. ObjectMapper 尚未索引該物件

**解決方案**：

```bash
# 確認 ObjectMapper 服務運行中
systemctl status xyz.openbmc_project.ObjectMapper.service

# 列出 ObjectMapper 的完整快取
busctl call xyz.openbmc_project.ObjectMapper \
    /xyz/openbmc_project/ObjectMapper \
    xyz.openbmc_project.ObjectMapper \
    GetSubTree sias "/" 0 0
```

---

## 💻 開發問題

### 新介面未出現

**症狀**：新增的 YAML 檔案未被建置

**解決方案**：

```bash
# 重新生成 Meson 設定
cd gen
./regenerate-meson

# 重新設定建置
cd ..
meson builddir --wipe
ninja -C builddir
```

### 生成的程式碼有編譯錯誤

**可能原因**：
1. YAML 語法錯誤
2. 引用了不存在的型別或介面
3. sdbusplus 版本不匹配

**解決方案**：

```bash
# 手動執行 sdbus++ 檢查輸出
sdbus++ interface server-header xyz.openbmc_project.MyInterface

# 更新 sdbusplus 到最新版本
cd sdbusplus
git pull
meson builddir --wipe && ninja -C builddir
sudo ninja -C builddir install
```

---

## 🐛 除錯工具

### busctl

```bash
# 列出所有服務
busctl --list

# 查看服務的物件樹
busctl tree <SERVICE>

# 查看物件的介面和屬性
busctl introspect <SERVICE> <PATH>

# 讀取屬性
busctl get-property <SERVICE> <PATH> <INTERFACE> <PROPERTY>

# 設定屬性
busctl set-property <SERVICE> <PATH> <INTERFACE> <PROPERTY> <TYPE> <VALUE>

# 呼叫方法
busctl call <SERVICE> <PATH> <INTERFACE> <METHOD> [SIGNATURE] [ARGS...]

# 監控訊號
busctl monitor <SERVICE>
```

### dbus-monitor

```bash
# 監控所有 D-Bus 訊息
sudo dbus-monitor --system

# 過濾特定介面
sudo dbus-monitor --system "type='signal',interface='org.freedesktop.DBus.Properties'"
```

### journalctl

```bash
# 查看服務日誌
journalctl -u xyz.openbmc_project.ObjectMapper.service

# 即時追蹤
journalctl -f -u <SERVICE>
```

---

## ❓ FAQ

### Q: 如何知道哪個服務提供特定介面？

**A**: 使用 ObjectMapper 的 `GetObject` 方法：

```bash
busctl call xyz.openbmc_project.ObjectMapper \
    /xyz/openbmc_project/ObjectMapper \
    xyz.openbmc_project.ObjectMapper \
    GetObject sas "<PATH>" 0
```

### Q: 如何列出所有感測器？

**A**:

```bash
busctl call xyz.openbmc_project.ObjectMapper \
    /xyz/openbmc_project/ObjectMapper \
    xyz.openbmc_project.ObjectMapper \
    GetSubTreePaths sias "/xyz/openbmc_project/sensors" 0 1 \
    "xyz.openbmc_project.Sensor.Value"
```

### Q: 屬性變更時如何收到通知？

**A**: 監聽 `PropertiesChanged` 訊號：

```bash
busctl monitor --match="type='signal',interface='org.freedesktop.DBus.Properties',member='PropertiesChanged'"
```

### Q: 如何檢查版本相容性？

**A**: 檢查 phosphor-dbus-interfaces 和 sdbusplus 的版本：

```bash
# 在 meson builddir 檢查
meson configure builddir | grep version
```

---

## 🔍 延伸資源

- [OpenBMC Discord](https://discord.gg/69Km47zH98) - 社群支援
- [OpenBMC GitHub Issues](https://github.com/openbmc/phosphor-dbus-interfaces/issues) - 問題回報
- [D-Bus 除錯教學](https://dbus.freedesktop.org/doc/debugging.html) - 官方除錯指南

---

*最後更新：2025-12-19*
