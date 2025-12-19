# QuickStart（快速入門）

本文說明如何設置 OpenBIC 開發環境、編譯 yv4-sd 韌體，以及進行基本除錯。

---

## 系統需求

### 作業系統

- Ubuntu 18.04 LTS 或更新版本（64-bit）
- bash shell

### 必要工具版本

| 工具 | 最低版本 |
|------|----------|
| CMake | 3.20.0 |
| Python | 3.6 |
| Device Tree Compiler | 1.4.6 |

---

## 環境設置

### 1. 安裝依賴套件

```bash
sudo apt install --no-install-recommends git cmake ninja-build gperf \
    ccache dfu-util device-tree-compiler wget \
    python3-dev python3-pip python3-setuptools python3-tk python3-wheel xz-utils file \
    make gcc gcc-multilib g++-multilib libsdl2-dev
```

### 2. 安裝 West 工具

[West](https://docs.zephyrproject.org/latest/develop/west/index.html) 是 Zephyr 的元工具，用於管理多儲存庫專案：

```bash
pip3 install --user -U west
echo 'export PATH=~/.local/bin:"$PATH"' >> ~/.bashrc
source ~/.bashrc
```

### 3. 取得 OpenBIC 原始碼

```bash
# 初始化 West workspace
west init -m https://github.com/facebook/OpenBIC zephyrproject

# 進入專案目錄
cd zephyrproject

# 更新所有子模組
west update
```

> [!NOTE]
> `west init` 會從 OpenBIC 的 `west.yml` 取得所有依賴的 manifest，包括 Zephyr kernel 和相關模組。

### 4. 安裝 Zephyr SDK

```bash
cd ~

# 下載 SDK
wget https://github.com/zephyrproject-rtos/sdk-ng/releases/download/v0.12.4/zephyr-sdk-0.12.4-x86_64-linux-setup.run

# 設定執行權限
chmod +x zephyr-sdk-0.12.4-x86_64-linux-setup.run

# 安裝到 ~/zephyr-sdk-0.12.4
./zephyr-sdk-0.12.4-x86_64-linux-setup.run -- -d ~/zephyr-sdk-0.12.4
```

### 5. 設定環境變數

```bash
# 設定 Zephyr base 路徑
export ZEPHYR_BASE=~/zephyrproject/zephyr

# 設定 SDK 路徑
export ZEPHYR_SDK_INSTALL_DIR=~/zephyr-sdk-0.12.4
```

建議將上述設定加入 `~/.bashrc`：

```bash
echo 'export ZEPHYR_BASE=~/zephyrproject/zephyr' >> ~/.bashrc
echo 'export ZEPHYR_SDK_INSTALL_DIR=~/zephyr-sdk-0.12.4' >> ~/.bashrc
source ~/.bashrc
```

---

## 編譯韌體

### 編譯 yv4-sd

```bash
cd ~/zephyrproject/openbic

# 觸發 CMakeLists.txt 更新（確保 build 系統識別變更）
touch meta-facebook/yv4-sd/CMakeLists.txt

# 使用 West 編譯
west build -p auto -b ast1030_evb meta-facebook/yv4-sd/
```

**參數說明：**

| 參數 | 說明 |
|------|------|
| `-p auto` | 自動決定是否進行乾淨建置 |
| `-b ast1030_evb` | 指定目標板為 AST1030 EVB |
| `meta-facebook/yv4-sd/` | 指定平台目錄 |

### 編譯輸出

編譯完成後，韌體映像位於：

```
build/zephyr/
├── zephyr.bin      # 二進制映像
├── zephyr.elf      # ELF 格式（含除錯資訊）
├── zephyr.hex      # Intel HEX 格式
└── zephyr.map      # 記憶體映射
```

### 清除並重新編譯

```bash
# 完全清除 build 目錄
west build -t pristine

# 或直接刪除
rm -rf build

# 重新編譯
west build -b ast1030_evb meta-facebook/yv4-sd/
```

---

## 燒錄韌體

### 使用 JTAG/SWD

OpenBIC 支援透過 ASPEED 的 OpenOCD 進行燒錄：

```bash
# 使用 West flash 命令
west flash
```

### 使用 UART 更新

透過 IPMI 命令進行韌體更新（需要 BMC 支援）：

```bash
# 從 BMC 執行
bic-util <slot> --update bic <firmware_path>
```

---

## 除錯

### Shell 命令

OpenBIC 整合 Zephyr Shell，提供除錯命令列介面。透過 UART 連接後：

```
uart:~$ help
Available commands:
  gpio      : GPIO commands
  i2c       : I2C commands
  i3c       : I3C commands
  sensor    : Sensor commands
  ...
```

**常用 Shell 命令：**

| 命令 | 說明 |
|------|------|
| `gpio get <pin>` | 讀取 GPIO 狀態 |
| `gpio set <pin> <0\|1>` | 設定 GPIO |
| `i2c scan <bus>` | 掃描 I2C 設備 |
| `i3c attach <bus>` | I3C 裝置附加 |
| `sensor list` | 列出所有感測器 |

### 日誌系統

OpenBIC 使用 Zephyr LOG 系統：

```c
#include <logging/log.h>
LOG_MODULE_REGISTER(module_name, LOG_LEVEL_DBG);

LOG_INF("Info message");
LOG_DBG("Debug message");
LOG_WRN("Warning message");
LOG_ERR("Error message");
```

**日誌等級設定（prj.conf）：**

```
CONFIG_LOG=y
CONFIG_LOG_BACKEND_UART=y
CONFIG_LOG_BUFFER_SIZE=5120
```

### GDB 除錯

透過 OpenOCD 使用 GDB：

```bash
# 啟動 OpenOCD
openocd -f board/aspeed_ast1030.cfg

# 在另一終端啟動 GDB
arm-zephyr-eabi-gdb build/zephyr/zephyr.elf

# GDB 命令
(gdb) target remote :3333
(gdb) monitor reset halt
(gdb) load
(gdb) continue
```

---

## 專案結構預覽

```
zephyrproject/
├── .west/                 # West 配置
├── zephyr/                # Zephyr kernel
├── modules/               # Zephyr 模組
├── bootloader/            # Bootloader
└── openbic/               # OpenBIC 專案
    ├── common/            # 通用程式碼
    │   ├── dev/          # 裝置驅動
    │   ├── hal/          # 硬體抽象層
    │   ├── lib/          # 工具函式庫
    │   ├── service/      # 服務（MCTP, PLDM, IPMI...）
    │   └── shell/        # Shell 命令
    ├── meta-facebook/     # 平台特定
    │   ├── yv4-sd/       # Sentinel Dome ← 我們的目標
    │   ├── yv4-ff/       # Firefly
    │   ├── yv4-wf/       # Waimea Falls
    │   └── ...
    └── docs/              # 文件
```

---

## 常見問題

### Q: West update 失敗？

```bash
# 清除快取並重試
rm -rf ~/.cache/west
west update
```

### Q: 編譯時找不到標頭檔？

確認 `ZEPHYR_BASE` 環境變數已正確設定：

```bash
echo $ZEPHYR_BASE
# 應顯示：/path/to/zephyrproject/zephyr
```

### Q: 如何查看支援的平台列表？

```bash
ls meta-facebook/
```

### Q: 如何修改韌體版本？

編輯 `meta-facebook/yv4-sd/src/platform/plat_version.h`：

```c
#define BIC_FW_YEAR_MSB 0x20
#define BIC_FW_YEAR_LSB 0x25
#define BIC_FW_WEEK 0x47
#define BIC_FW_VER 0x01
```

---

## 下一步

- [Architecture](Architecture.md) - 了解系統架構
- [CodeOrganization](CodeOrganization.md) - 深入程式碼組織
- [YV4SDPlatform](YV4SDPlatform.md) - 平台特定說明

---

*返回 [Home](Home.md)*
