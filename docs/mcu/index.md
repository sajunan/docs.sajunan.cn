---
title: "单片机入门"
description: "CTF 单片机/硬件方向入门手册：电子基础、UART/I2C/SPI/SWD、Arduino/ESP32/STM32、串口调试、SPI Flash 固件提取与 Ghidra 分析。"
---

# 单片机入门

> **从一块开发板开始：先让它说话，再让它交出固件**

单片机（MCU）把 CPU、内存和外设塞进一块芯片。本文从电子基础、UART/I2C/SPI 讲到固件提取、SPI Flash、Ghidra 分析与常见硬件 CTF 题型，并给出从一块开发板到 flag 的完整演练。

## 第零章：单片机是什么

### 0.1 用一句话解释

单片机（MCU，Microcontroller Unit）就是“一块芯片里塞进了 CPU、内存、存储和外设”。它通常跑裸机程序或 RTOS，负责读传感器、控制电机、和别的设备通信。

和电脑的区别：

| | 单片机 | 电脑 |
|--|--------|------|
| 系统 | 裸机/RTOS | Windows/Linux |
| 存储 | 片上 Flash，几十 KB 到几 MB | 硬盘，几百 GB |
| 内存 | 几 KB 到几百 KB | 几 GB 到几十 GB |
| 调试 | SWD/JTAG/UART | 系统调试器 |
| 安全题 | 读固件、抓总线、绕过保护 | 内存破坏、权限提升 |

### 0.2 CTF 硬件题长什么样

- 给你一块开发板（或它的照片、PCB 文件），让你找串口日志里的 flag。
- 给你一段逻辑分析仪抓的波形，让你解码 SPI/I2C/UART。
- 给你一个固件 `.bin`，让你提取文件系统、找硬编码密码。
- 给你一个遥控器/门禁卡，让你分析无线信号。
- 给你一个带读保护的芯片，让你想办法 dump。
- 给你一个 Arduino 项目，让你逆它的小游戏。

### 0.3 新手先学什么

```text
电子基础（电压、电平、串口）
   ↓
一块 Arduino / ESP32 / STM32
   ↓
点灯、串口打印
   ↓
UART / I2C / SPI
   ↓
逻辑分析仪抓包
   ↓
固件提取与 binwalk
   ↓
Ghidra 看 ARM 代码
```

> 💡 不要一上来就买几千块的设备。一块几十块的 ESP32、一个几十块的逻辑分析仪、一个 CH341A，就能做完大多数入门题。

### 0.4 安全与授权

硬件题可能涉及真实设备。**只操作题目提供的设备、自己购买的开发板、明确授权的硬件**。不要拆别人的设备，不要读别人设备的固件。

---

## 第一章：电子基础与安全

### 1.1 三个基本量

- **电压 V**：电势差，单位伏特（V）。
- **电流 I**：电荷流动，单位安培（A）。
- **电阻 R**：阻碍电流，单位欧姆（Ω）。

欧姆定律：

```text
V = I × R
```

### 1.2 常见电平

| 电平 | 典型值 | 常见设备 |
|------|--------|----------|
| 5V | 4.5–5.5V | Arduino Uno、老式 51 |
| 3.3V | 3.0–3.6V | STM32、ESP32、树莓派 Pico |
| 1.8V | 1.7–1.9V | 部分 Flash、SoC 内核 |

> ⚠️ **3.3V 引脚不要接 5V**。轻则不工作，重则烧芯片。接线前先查引脚定义。

### 1.3 上拉与下拉

- **上拉电阻**：把引脚默认拉到高电平。
- **下拉电阻**：把引脚默认拉到低电平。
- 常见值：4.7kΩ、10kΩ。

UART 的 TX 空闲时通常是高电平，这就是为什么可以用万用表找 TX。

### 1.4 必备工具

| 工具 | 用途 | 价格区间 |
|------|------|----------|
| 万用表 | 测电压、通断、找 TX | 几十元 |
| 逻辑分析仪 | 抓 UART/SPI/I2C 波形 | 几十元（8 通道） |
| 面包板 + 杜邦线 | 搭电路 | 十几元 |
| 电烙铁 | 焊接排针/飞线 | 几十元 |
| 吸锡带/助焊剂 | 拆焊 | 十几元 |
| CH341A | 读写 SPI Flash | 几十元 |
| ST-Link V2 | STM32 下载/调试 | 几十元 |
| USB-TTL | 串口通信 | 十几元 |
| 防静电手环 | 防 ESD | 十几元 |

### 1.5 安全规则

1. **共地**：两个设备通信，GND 必须接在一起。
2. **先断电再接线**：接好再上电，避免短路。
3. **检查电压**：3.3V 设备不要接 5V。
4. **限流**：不确定时串一个电阻，或用带限流的电源。
5. **防静电**：摸一下接地金属，或戴防静电手环。
6. **不要短路**：万用表测通断，确认电源和地没有短在一起。

---

## 第二章：常见开发板与芯片

### 2.1 Arduino Uno

| 项目 | 参数 |
|------|------|
| 芯片 | ATmega328P |
| 架构 | AVR 8 位 |
| 主频 | 16 MHz |
| Flash | 32 KB |
| SRAM | 2 KB |
| 电平 | 5V |
| 烧录 | USB / ICSP / avrdude |

适合：入门点灯、串口、传感器。

### 2.2 STM32F103C8T6（Blue Pill）

| 项目 | 参数 |
|------|------|
| 内核 | ARM Cortex-M3 |
| 主频 | 72 MHz |
| Flash | 64 KB（部分芯片实际 128 KB） |
| SRAM | 20 KB |
| 电平 | 3.3V |
| 调试 | SWD |
| 烧录 | ST-Link、串口 ISP |

适合：学 ARM、学 SWD、学读保护。

### 2.3 ESP32

| 项目 | 参数 |
|------|------|
| 内核 | Xtensa LX6 双核（经典款） |
| 主频 | 240 MHz |
| Flash | 外置 SPI Flash，常见 4 MB |
| 无线 | WiFi + 蓝牙 |
| 电平 | 3.3V |
| 烧录 | USB / esptool |
| 调试 | JTAG（部分引脚复用） |

适合：WiFi/BLE 题、固件提取、IoT。

### 2.4 Raspberry Pi Pico

| 项目 | 参数 |
|------|------|
| 芯片 | RP2040 |
| 内核 | 双核 Cortex-M0+ |
| 主频 | 133 MHz |
| SRAM | 264 KB |
| Flash | 2 MB（板载） |
| 电平 | 3.3V |
| 烧录 | USB（拖拽 UF2） |

适合：MicroPython、入门硬件。

### 2.5 其他常见芯片

| 芯片 | 特点 |
|------|------|
| 51 单片机 | 老、简单、5V |
| nRF52 | BLE，Cortex-M4 |
| ATtiny | 小体积 AVR |
| ESP8266 | 便宜 WiFi |
| STM32F4 | Cortex-M4，性能强 |
| GD32 | STM32 兼容 |

---

## 第三章：通信总线

### 3.1 UART（串口）

最常用、最容易上手的总线。

| 引脚 | 作用 |
|------|------|
| TX | 发送 |
| RX | 接收 |
| GND | 地 |
| VCC | 电源（可选） |

接线：**A 的 TX 接 B 的 RX，A 的 RX 接 B 的 TX，GND 共地**。

参数：波特率、数据位、校验位、停止位。最常见 `115200 8N1` 或 `9600 8N1`。

```text
起始位(0) + 8 个数据位 + 校验位(可选) + 停止位(1)
```

> 💡 串口乱码，90% 是波特率不对；剩下 10% 是 TX/RX 接反或没共地。

### 3.2 I2C

| 引脚 | 作用 |
|------|------|
| SCL | 时钟 |
| SDA | 数据 |

- 两根线，支持多设备。
- 每个设备有 7 位地址。
- 需要上拉电阻（很多模块自带）。

### 3.3 SPI

| 引脚 | 作用 |
|------|------|
| CS | 片选 |
| CLK | 时钟 |
| MOSI | 主出从入 |
| MISO | 主入从出 |

- 速度快，常用于 Flash、屏幕、传感器。
- 每个从设备一根 CS。

### 3.4 SWD

ARM Cortex-M 常用调试接口。

| 引脚 | 作用 |
|------|------|
| SWDIO | 数据 |
| SWCLK | 时钟 |
| GND | 地 |
| VCC | 3.3V（参考电平） |
| RESET | 复位（可选） |

接线：

```text
ST-Link     目标板
SWDIO  →    SWDIO
SWCLK  →    SWCLK
GND    →    GND
3.3V   →    3.3V
```

### 3.5 JTAG

更完整的调试接口：

```text
TCK、TMS、TDI、TDO、TRST、GND、VCC
```

SWD 是 JTAG 的精简版，引脚更少。

### 3.6 其他

| 总线 | 用途 |
|------|------|
| CAN | 汽车、工业 |
| USB | 外设、键盘、存储 |
| 1-Wire | 温度传感器、iButton |
| PWM | 电机、LED 调光 |
| ADC | 模拟采样 |

---

## 第四章：开发环境与烧录

### 4.1 Arduino

1. 安装 Arduino IDE。
2. 插上板子，选择开发板和端口。
3. 写代码：

```cpp
void setup() {
  Serial.begin(115200);
  pinMode(LED_BUILTIN, OUTPUT);
}

void loop() {
  digitalWrite(LED_BUILTIN, HIGH);
  delay(500);
  digitalWrite(LED_BUILTIN, LOW);
  delay(500);
  Serial.println("hello");
}
```

4. 点击上传。

命令行烧录：

```bash
avrdude -c arduino -p m328p -P COM3 -b 115200 -U flash:w:sketch.hex:i
```

### 4.2 ESP32

安装 ESP-IDF 或 Arduino-ESP32。

```bash
# 看芯片信息
esptool.py -p COM3 chip_id
esptool.py -p COM3 flash_id

# 读 Flash（0 地址，4MB）
esptool.py -p COM3 -b 460800 read_flash 0 0x400000 dump.bin

# 写 Flash
esptool.py -p COM3 write_flash 0x0 firmware.bin

# 擦除
esptool.py -p COM3 erase_flash
```

> 💡 ESP32 读 Flash 前要进下载模式：按住 BOOT，按一下 EN，再松开 BOOT。

### 4.3 STM32

**ST-Link + OpenOCD：**

```bash
openocd -f interface/stlink.cfg -f target/stm32f1x.cfg
```

**ST-Link Utility / STM32CubeProgrammer**：图形化读写、看选项字节。

**pyOCD：**

```bash
pyocd list
pyocd flash -t stm32f103c8 firmware.hex
```

### 4.4 树莓派 Pico

- 按住 BOOTSEL，插 USB。
- 出现 `RPI-RP2` U 盘。
- 拖入 `.uf2` 文件。

### 4.5 逻辑分析仪

推荐 PulseView（sigrok）。

1. 接通道到目标信号和 GND。
2. 采样率至少是信号频率的 4-10 倍。
3. 添加协议解码器：UART、SPI、I2C。
4. 设置波特率/通道。
5. 看解码结果。

```bash
# sigrok-cli 示例
sigrok-cli -d fx2lafw --channels D0,D1 --config samplerate=1m \
  --time 2s -P uart:baudrate=115200:rx=D0 -A uart=rx_data
```

---

## 第五章：硬件题完整流程

### 5.1 一张总图

```text
拿到设备
   ↓
看板子：芯片型号、丝印、测试点、排针
   ↓
上电：测电压、看指示灯、摸温度
   ↓
找调试口：UART / SWD / JTAG
   ↓
抓串口日志 / 读 Flash
   ↓
固件分析：binwalk / strings / Ghidra
   ↓
找 flag / 密码 / 逻辑
   ↓
复现或写报告
```

### 5.2 看板子

- 芯片丝印：型号、厂商、批次。
- 测试点：`TX`、`RX`、`GND`、`3V3`、`SWDIO`、`SWCLK`。
- 排针：是否引出调试口。
- 按键：复位、BOOT。
- 接口：USB、JTAG、SWD、串口。
- Flash 芯片：SOIC8 封装，可能是外部 SPI Flash。

### 5.3 上电前检查

1. 万用表测电源和地是否短路。
2. 确认供电电压。
3. 确认极性。
4. 限流上电。

### 5.4 上电后观察

- 指示灯是否亮。
- 电流是否异常。
- 是否有串口输出。
- 芯片是否发热（发热可能有短路）。

### 5.5 记录

拍照、画接线图、记录芯片型号、记录串口输出。后面写 WP 全靠这些。

---

## 第六章：找 UART 与抓串口

### 6.1 用万用表找 TX

1. 万用表直流电压档，黑表笔接 GND。
2. 红表笔依次点测试点。
3. TX 空闲时通常是高电平（3.3V 或 5V）。
4. 上电瞬间，TX 会有波动。
5. GND 与电源地导通。

### 6.2 用逻辑分析仪确认

1. 接一个通道到疑似 TX，GND 共地。
2. 采样率 1MHz 以上。
3. 添加 UART 解码器，试 9600、115200。
4. 看是否有可读字符。

### 6.3 常见波特率

```text
9600
19200
38400
57600
115200
230400
460800
921600
```

### 6.4 接线

```text
USB-TTL     目标板
RX     ←    TX
TX     →    RX
GND    →    GND
```

> ⚠️ 只接 TX/RX/GND，不要接 VCC，除非你确定需要给目标供电。两边都供电可能冲突。

### 6.5 打开串口

Windows：PuTTY、MobaXterm、SecureCRT。

Linux：

```bash
screen /dev/ttyUSB0 115200
# 或
minicom -D /dev/ttyUSB0 -b 115200
```

### 6.6 看启动日志

很多设备启动时会打印：

```text
U-Boot 2020.04
DRAM: 128 MiB
...
Hit any key to stop autoboot:
```

看到 U-Boot，可以尝试中断启动、读环境变量、读 Flash。

### 6.7 串口里的 shell

有些设备直接给你 root shell：

```text
login: root
Password:
```

试试空密码、`root`、`admin`、`123456`，或者题目提示。

---

## 第七章：SWD / JTAG 调试

### 7.1 找 SWD 引脚

- 板子上标 `SWDIO`、`SWCLK`、`GND`、`3V3`。
- 四针排针常见顺序：`3V3 SWDIO SWCLK GND`。
- 用万用表确认 GND 和 3.3V。

### 7.2 ST-Link 接线

```text
ST-Link     目标板
SWDIO  →    SWDIO
SWCLK  →    SWCLK
GND    →    GND
3.3V   →    3.3V
```

### 7.3 OpenOCD 连接

```bash
openocd -f interface/stlink.cfg -f target/stm32f1x.cfg
```

看到：

```text
Info : stm32f1x.cpu: hardware has 6 breakpoints, 4 watchpoints
```

说明连上了。

### 7.4 读 Flash

用 OpenOCD：

```text
openocd -f interface/stlink.cfg -f target/stm32f1x.cfg \
  -c "init; reset halt; flash read_bank 0 firmware.bin 0 0x10000; shutdown"
```

或用 STM32CubeProgrammer 图形化读取。

### 7.5 读保护

如果读到全 `0x00`、全 `0xFF`，或者报 “read protected”，可能是 RDP（Readout Protection）开启。

- RDP Level 0：无保护。
- RDP Level 1：调试口可连，但不能读 Flash。
- RDP Level 2：永久保护，调试口禁用。

> ⚠️ 解除 RDP Level 1 通常会触发全片擦除，固件就没了。CTF 里遇到读保护，通常要换思路（找串口、找漏洞、找未保护区域），而不是硬解。

### 7.6 JTAG

JTAG 引脚更多，但功能更全。用 OpenOCD 的 JTAG 配置，或 J-Link。

常见：

```bash
openocd -f interface/jlink.cfg -f target/stm32f4x.cfg
```

---

## 第八章：SPI Flash 与固件提取

### 8.1 认识 SPI Flash

常见封装：SOIC8、SOP16、WSON8。

SOIC8 引脚（从芯片丝印正放，左下角为 1 脚逆时针）：

```text
1  CS    8  VCC
2  DO    7  HOLD
3  WP    6  CLK
4  GND   5  DI
```

### 8.2 用 CH341A 读取

1. 断电。
2. 用 SOIC8 夹子夹住 Flash（注意 1 脚方向）。
3. 连接 CH341A 到电脑。
4. 用工具读取：

```bash
# Linux
sudo flashrom -p ch341a_spi -r dump1.bin

# 或
sudo flashrom -p ch341a_spi -c "W25Q32" -r dump1.bin
```

5. 至少读 3 次，比较 SHA256：

```bash
sha256sum dump1.bin dump2.bin dump3.bin
```

三个一样才说明读取稳定。

> 💡 夹子接触不良、目标板还有电、1 脚方向反了，都会导致 dump 不一致或全 FF。

### 8.3 用 ESP32 读 Flash

```bash
esptool.py -p COM3 -b 460800 read_flash 0 0x400000 esp32.bin
```

### 8.4 用 Bootloader / U-Boot 读

有些设备 U-Boot 支持：

```text
md.b 0x80000000 0x100
sf read 0x80000000 0x0 0x100000
tftpboot
```

或者用 `dd` 从 `/dev/mtd` 读。

### 8.5 读出来之后

```bash
file firmware.bin
binwalk firmware.bin
binwalk -e firmware.bin
strings -a firmware.bin | head
```

---

## 第九章：固件分析

### 9.1 先看文件类型

```bash
file firmware.bin
binwalk firmware.bin
```

可能看到：

```text
DECIMAL       HEXADECIMAL     DESCRIPTION
0             0x0             uImage header
64            0x40            LZMA compressed data
...
```

### 9.2 提取

```bash
binwalk -e firmware.bin
```

常见文件系统：

| 类型 | 工具 |
|------|------|
| SquashFS | `unsquashfs` |
| JFFS2 | `jefferson` |
| UBIFS | `ubi_reader` |
| CramFS | `cramfsck` |
| ext2/3/4 | `mount` / `debugfs` |
| CPIO | `cpio -id` |
| TAR | `tar xf` |

### 9.3 找字符串

```bash
strings -a firmware.bin | grep -i "flag"
strings -a firmware.bin | grep -i "password"
strings -a firmware.bin | grep -i "admin"
strings -a firmware.bin | grep -E "https?://"
```

### 9.4 Ghidra 分析 ARM

1. 新建项目，导入 `.bin`。
2. 选择处理器：`ARM Cortex-M`，Little Endian。
3. 设置基址：STM32 通常是 `0x08000000`。
4. 让 Ghidra 自动分析。
5. 找向量表：第一个字是初始 SP，第二个字是 Reset Handler。
6. 找 `main`、`printf`、`strcmp`。

### 9.5 找硬编码密码

- `strings` 搜 `password`、`passwd`、`secret`、`key`。
- 在 Ghidra 里对这些字符串按 `X` 看引用。
- 看比较逻辑。

### 9.6 找 flag

- 字符串里直接有。
- 被异或/Base64 编码。
- 在文件系统里的某个文件。
- 在 Flash 的某个偏移。
- 在 EEPROM。
- 在启动日志。

### 9.7 熵分析

```bash
binwalk -E firmware.bin
```

熵高的区域可能是压缩/加密；熵低的地方可能是明文配置。

---

## 第十章：常见 CTF 题型

### 10.1 串口日志藏 flag

设备启动时打印 flag。接 UART，抓日志即可。

### 10.2 逻辑分析仪解码

给一段 SPI/I2C/UART 波形，解码出数据。

- 先看有几条线。
- 猜协议。
- 试不同波特率/时钟极性。
- 解码结果可能是 ASCII、hex、图片。

### 10.3 固件里的硬编码密码

```bash
strings firmware.bin | grep -i pass
```

### 10.4 EEPROM

小容量存储，常存配置/密码。用 CH341A 或 Arduino 读。

### 10.5 RFID / NFC

- 读卡器读 UID、扇区数据。
- M1 卡有默认密钥 `FFFFFFFFFFFF`。
- 工具：Proxmark3、ACR122U、Flipper Zero。

### 10.6 红外 / 无线

- 红外：逻辑分析仪抓 38kHz 载波，解码 NEC 协议。
- 315/433MHz：用 SDR（RTL-SDR）或逻辑分析仪抓。
- 工具：Universal Radio Hacker（URH）。

### 10.7 BadUSB

- 模拟键盘输入。
- 分析 HID 报告。
- 工具： Rubber Ducky、Digispark。

### 10.8 侧信道（了解）

- 功耗分析：通过功耗曲线推断密钥。
- 时序攻击：通过执行时间推断。
- 故障注入：电压/时钟毛刺跳过校验。

新手先知道概念，遇到再查。

### 10.9 小游戏/密码锁

- 固件里找验证逻辑。
- 用 Ghidra 还原。
- 或直接改 Flash 里的分数/密码。

---

## 第十一章：保护机制与绕过

### 11.1 读保护（RDP）

STM32：

| 级别 | 行为 |
|------|------|
| Level 0 | 无保护 |
| Level 1 | 调试口可连，Flash 不可读；解除会全擦 |
| Level 2 | 永久保护，调试口禁用 |

### 11.2 Secure Boot

芯片启动时校验固件签名。修改固件会导致启动失败。

CTF 里常见：

- 找到未校验的分区。
- 找到调试后门。
- 找签名校验的漏洞。

### 11.3 Flash 加密

ESP32 等芯片支持 Flash 加密。读出来的数据是密文。

CTF 里通常会给解密后的固件，或让你找密钥。

### 11.4 调试口禁用

- 熔断 eFuse。
- 关闭 JTAG。
- 改选项字节。

### 11.5 绕过思路

1. 找串口/其他未保护接口。
2. 找 Bootloader 漏洞。
3. 找固件升级签名漏洞。
4. 找硬件测试点。
5. 侧信道/故障注入（进阶）。

> 💡 CTF 里遇到保护，先判断“题目是不是想让你硬碰硬”。大多数题会留一个入口，比如串口、U-Boot、未保护分区。

---

## 第十二章：仿真与调试

### 12.1 Renode

适合仿真 Cortex-M 系统：

```bash
renode
```

```text
# 使用 Renode 自带的 STM32 平台文件（不同版本文件名可能不同）
machine add @platforms/cpus/stm32f4_discovery.repl
sysbus LoadELF firmware.elf
start
```

### 12.2 QEMU

QEMU 对 MCU 支持有限，但可以仿真部分 ARM 系统：

```bash
qemu-system-arm -M microbit -kernel firmware.hex
```

### 12.3 Unicorn

只仿真 CPU 指令，不仿真外设，适合跑加密函数：

```python
from unicorn import Uc, UC_ARCH_ARM, UC_MODE_THUMB
from unicorn.arm_const import UC_ARM_REG_R0

mu = Uc(UC_ARCH_ARM, UC_MODE_THUMB)
mu.mem_map(0x08000000, 0x10000)
mu.mem_write(0x08000000, code)
mu.reg_write(UC_ARM_REG_R0, 0x1234)
mu.emu_start(0x08000001, 0x08000000 + len(code))
print(hex(mu.reg_read(UC_ARM_REG_R0)))
```

### 12.4 GDB + OpenOCD

```bash
openocd -f interface/stlink.cfg -f target/stm32f1x.cfg
arm-none-eabi-gdb firmware.elf
(gdb) target remote :3333
(gdb) load
(gdb) break main
(gdb) continue
```

### 12.5 逻辑分析仪 + 调试器结合

- 逻辑分析仪看总线时序。
- 调试器看寄存器和内存。
- 两边对照，定位问题最快。

---

## 第十三章：完整演练：从一块板到 flag

### 13.1 场景

你拿到一块 STM32 开发板，题目说“flag 在设备里”。板子上有 4 个测试点、一个 USB 口、一个 4 针排针。

### 13.2 第一步：看板子

- 芯片丝印：`STM32F103C8T6`。
- 4 针排针旁有丝印：`3V3 SWDIO SWCLK GND`。
- 测试点旁有丝印：`TX RX GND`。
- 有一个 SOIC8 Flash 芯片。

### 13.3 第二步：上电

- 万用表测 3.3V 正常。
- 指示灯亮。
- 电流正常。

### 13.4 第三步：接串口

```text
USB-TTL RX  ←  板子 TX
USB-TTL TX  →  板子 RX
USB-TTL GND →  板子 GND
```

打开串口 115200：

```text
[boot] STM32 CTF demo
[init] flash ok
[init] uart ok
[flag] flag{uart_is_the_front_door}
```

拿到 flag。

### 13.5 第四步：如果串口没有

用 ST-Link 接 SWD：

```text
ST-Link SWDIO  → SWDIO
ST-Link SWCLK  → SWCLK
ST-Link GND    → GND
ST-Link 3.3V   → 3.3V
```

OpenOCD 读取：

```bash
openocd -f interface/stlink.cfg -f target/stm32f1x.cfg \
  -c "init; reset halt; flash read_bank 0 firmware.bin 0 0x10000; shutdown"
```

### 13.6 第五步：分析固件

```bash
strings -a firmware.bin | grep -i flag
```

如果没找到：

```bash
binwalk -e firmware.bin
```

解出文件系统，找配置/脚本。

### 13.7 第六步：Ghidra

- 导入 `firmware.bin`。
- 处理器 `ARM Cortex-M`。
- 基址 `0x08000000`。
- 找字符串引用。
- 还原校验逻辑。

### 13.8 第七步：写报告

记录接线、命令、固件哈希、分析过程、flag。

---

## 第十四章：工具链与学习路线

### 14.1 工具速查

| 场景 | 工具 |
|------|------|
| 看芯片 | 丝印、Datasheet、万用表 |
| 串口 | USB-TTL、PuTTY、screen、minicom |
| 逻辑分析 | PulseView、sigrok、Saleae |
| SPI Flash | CH341A、flashrom |
| ESP32 | esptool |
| STM32 | ST-Link、OpenOCD、STM32CubeProgrammer |
| AVR | avrdude |
| 固件提取 | binwalk、foremost、dd |
| 文件系统 | unsquashfs、jefferson、ubi_reader |
| 反编译 | Ghidra、IDA |
| 仿真 | Renode、QEMU、Unicorn |
| 无线 | URH、RTL-SDR、Proxmark3 |

### 14.2 学习路线

```text
电子基础 + 万用表
   ↓
Arduino 点灯 + 串口
   ↓
逻辑分析仪 + UART
   ↓
ESP32 / STM32 + SWD
   ↓
SPI Flash + 固件提取
   ↓
binwalk + Ghidra
   ↓
综合硬件题
```

### 14.3 推荐练习

- **Arduino 官方示例**：点灯、串口、I2C。
- **ESP32 官方示例**：WiFi、BLE、Flash。
- **CTF Wiki 硬件/杂项**：基础题。
- **Hack The Box Hardware**：进阶。
- **picoCTF**：部分硬件/嵌入式题。
- **Flipper Zero 文档**：RFID/红外/无线。
- **各芯片 Datasheet**：最权威。

### 14.4 买什么

入门套装：

```text
ESP32 开发板 × 2
Arduino Uno × 1
USB-TTL × 1
逻辑分析仪（8 通道）× 1
CH341A + SOIC8 夹子 × 1
ST-Link V2 × 1
面包板 + 杜邦线
万用表
```

---

## 附录：速查表与参考资料

### A.1 串口接线

```text
设备 A TX → 设备 B RX
设备 A RX → 设备 B TX
GND      → GND
```

### A.2 常用命令

```bash
# ESP32
esptool.py -p COM3 chip_id
esptool.py -p COM3 read_flash 0 0x400000 dump.bin

# SPI Flash
sudo flashrom -p ch341a_spi -r dump.bin

# STM32
openocd -f interface/stlink.cfg -f target/stm32f1x.cfg

# 固件
binwalk -e firmware.bin
strings -a firmware.bin | grep -i flag

# 逻辑分析仪
sigrok-cli -d fx2lafw --channels D0 --config samplerate=1m -P uart:baudrate=115200:rx=D0
```

### A.3 常见波特率

```text
9600 / 19200 / 38400 / 57600 / 115200 / 230400 / 460800 / 921600
```

### A.4 参考资料

- Arduino 文档：<https://docs.arduino.cc/>
- ESP32 文档：<https://docs.espressif.com/projects/esp-idf/>
- STM32 文档：<https://www.st.com/>
- OpenOCD：<https://openocd.org/>
- flashrom：<https://www.flashrom.org/>
- binwalk：<https://github.com/ReFirmLabs/binwalk>
- PulseView/sigrok：<https://sigrok.org/>
- Ghidra：<https://ghidra-sre.org/>
- Renode：<https://renode.io/>
- CTF Wiki 硬件：<https://ctf-wiki.org/>

---

> 📌 **最后一句：** 硬件题的第一步永远不是“读 Flash”，而是“先让板子说话”。串口日志、启动信息、指示灯，往往比拆芯片更快给你答案。
