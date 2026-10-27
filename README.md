

***

# 解决 YD-ESP32-S3 (N16R8) 因 XMC Flash 芯片 ID 不匹配导致 MicroPython 无法启动的问题

## 📌 背景信息
*   **开发板型号**：YD-ESP32-S3 (ESP32-S3-WROOM-1-N16R8 模块，16MB Flash + 8MB Octal PSRAM)
*   **初始状态**：出厂自带测试程序，板载 RGB 灯正常闪烁。
*   **目标**：刷入 MicroPython 固件进行 Python 开发。

## 🐛 问题现象
使用官方 `Flash Download Tool` 清除 Flash 并烧录 MicroPython 固件（无论是商家提供的旧版 `v1.19.1`，还是官方最新版 `v1.29.0`），地址均正确设置为 `0x0`。烧录软件显示成功，但重启后开发板无限重启，串口输出以下报错：

```text
E (31) bootloader_flash: XMC flash startup fail
E (31) boot.esp32s3: failed when running XMC startup flow, reboot!
```

连接 Thonny 会提示 `PermissionError(13, '拒绝访问。')`。

## 🔍 问题排查与定位
使用 `esptool` 读取芯片信息：
```bash
python -m esptool --port COM4 flash-id
```

输出关键信息：
```text
WARNING: XMC flash chip boot-up failure detected! Running XMC25QHxxC startup flow.
WARNING: Could not confirm XMC flash chip boot-up fix. The flash chip may not work.
Flash Memory Information:
Manufacturer: 20
Device: 7018
Detected flash size: 16MB
```

**根因分析**：
板载的 XMC Flash 芯片型号为 **XM25QH128AHIG**，其 Manufacturer ID (MFID) 为 `0x70`。而 ESP-IDF 底层的 `bootloader_flash.c` 中的 `is_xmc_chip_strict()` 函数存在硬编码白名单，仅允许 MFID 为 `0x40`、`0x41`、`0x50` 的芯片通过校验。因为 `0x70` 不在名单中，bootloader 判定芯片非法，直接触发无限重启。

*(注：该问题在 ESP-IDF GitHub 的 Issue #18947 和 #10999 中已有记录)*

## 🛠️ 解决方案与踩坑记录

### 阶段一：Windows 下的尝试（失败）
1.  **更换固件版本**：从商家旧版到 MicroPython 官网最新版 v1.29.0 (Support for Octal-SPIRAM)，报错不变。
2.  **尝试用 pip 安装 mpy-cross**：在 ESP-IDF 命令行中，pip 被锁定到本地离线源，无法安装。
3.  **尝试用 CMake/idf.py 在 Windows 编译 MicroPython**：
    *   报错 `Failed to resolve component 'newlib'`（ESP-IDF 5.5.5 组件重命名兼容性问题）。
    *   降级到 **ESP-IDF v5.5.1** 后，CMake 配置通过。
    *   编译过程卡在 `qstr.i.last`，报错 `命令行过长`（Windows CMD 8191 字符限制）。

### 阶段二：Ubuntu 虚拟机的尝试（成功）
**环境**：VMware 虚拟机 + Ubuntu 22.04 LTS

**步骤 1：解决虚拟机磁盘空间不足**
ESP-IDF 完整编译环境需要较大空间。若空间不足会报 `[Errno 28] No space left on device`。
使用 `lvextend` 和 `resize2fs` 扩展了 Ubuntu 虚拟机磁盘至 40GB+。

**步骤 2：安装 ESP-IDF 依赖与工具链**
```bash
sudo apt update
sudo apt install -y git wget flex bison gperf python3 python3-pip python3-venv cmake ninja-build ccache libffi-dev libssl-dev dfu-util libusb-1.0-0
```
下载 ESP-IDF v5.5.1 并使用国内源安装（避免网络超时）：
```bash
git clone -b v5.5.1 --recursive https://github.com/espressif/esp-idf.git
cd esp-idf
export IDF_GITHUB_ASSETS="dl.espressif.cn/github_assets"
./install.sh esp32s3
. ./export.sh
```

**步骤 3：修改 Bootloader 白名单源码（核心）**
打开文件：`nano components/bootloader_support/bootloader_flash/src/bootloader_flash.c`
找到 `is_xmc_chip_strict` 函数，将：
```c
} else if (mfid == 0x41) {
```
修改为：
```c
} else if (mfid == 0x41 || mfid == 0x70) {
```
保存退出。

**步骤 4：编译 MicroPython 固件**
```bash
cd ~
git clone --recursive https://github.com/micropython/micropython.git
cd micropython
pip3 install mpy-cross
cd ports/esp32
idf.py -D MICROPY_BOARD=ESP32_GENERIC_S3 -D MICROPY_BOARD_VARIANT=SPIRAM_OCT build
```
编译成功，生成三个关键文件：
*   `build/bootloader/bootloader.bin` (包含修改后的白名单)
*   `build/partition_table/partition-table.bin`
*   `build/micropython.bin`

**步骤 5：传输文件至 Windows 并烧录**

将编译生成的三个关键文件从 Ubuntu 传输至 Windows，并按照方案一重命名以便于区分：

*   `bootloader/bootloader.bin 重命名为 YD-ESP32-S3-XMC-Fix-bootloader.bin

*   `partition_table/partition-table.bin 重命名为 YD-ESP32-S3-partition-table.bin

*   `micropython.bin 重命名为 YD-ESP32-S3-MicroPython-v1.29.0.bin

使用 Flash Download Tool 进行烧录，请严格对应以下地址：

*   `YD-ESP32-S3-XMC-Fix-bootloader.bin -> 填 0x0

*   `YD-ESP32-S3-partition-table.bin -> 填 0x8000

*   `YD-ESP32-S3-MicroPython-v1.29.0.bin -> 填 0x10000

## 🎉 最终结果
烧录完成后，拔插 USB 或按下 RST 复位键。开发板成功启动，不再报 `XMC flash startup fail`。打开 Thonny，成功连接并出现 `>>>` 提示符，板载 RGB 灯（GPIO48）成功点亮！
固件已提供下载，适用XM25QH128AHIG或其他Manufacturer ID (MFID) 为 `0x70`的芯片

## 💡 总结与建议
*   对于使用较新批次 XMC Flash 芯片（MFID `0x70`）的 ESP32-S3 开发板，目前官方标准固件均无法直接启动。
*   在 Windows 下编译 MicroPython 极其痛苦（长路径限制、环境冲突），**强烈建议使用 Linux 环境**。
*   购买此类开发板时，建议先用 `esptool flash_id` 确认 Flash 芯片型号，若遇到兼容性问题可直接要求商家更换或提供适配固件。
