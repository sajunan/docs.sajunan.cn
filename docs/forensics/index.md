---
title: "取证入门"
description: "CTF 取证方向入门手册：磁盘镜像、文件系统、Windows 痕迹、内存、流量、浏览器/邮件、手机、云日志、时间线与反取证。"
---

# 取证入门

> **像侦探一样工作：先保护现场，再按时间线还原发生了什么**

取证是在不破坏证据的前提下，从磁盘、内存、流量和日志中还原事件。本文覆盖镜像获取、Windows 痕迹、内存取证、流量分析、手机与云取证，并给出从镜像到 flag 的完整演练。

## 第零章：取证到底是什么

### 0.1 用一句话解释

取证（Forensics）就是：**在尽量不破坏证据的前提下，从存储介质、内存、流量、日志里还原“谁在什么时候做了什么”**。

CTF 里的取证题通常给你：

- 一个磁盘镜像（`.img`、`.E01`、`.vmdk`）。
- 一个内存镜像（`.raw`、`.mem`、`.dmp`）。
- 一个流量包（`.pcap`、`.pcapng`）。
- 一个手机备份、浏览器数据、邮件文件。
- 一个被删除文件、被清空的日志、被改过时间戳的系统。

你要找出 flag、恶意程序、入侵痕迹、聊天记录、被删除的文件。

### 0.2 取证和 Misc 的区别

| 方向 | 重点 |
|------|------|
| Misc | 编码、隐写、杂项套路，偏“猜”和“剥” |
| 取证 | 有明确的证据来源和分析流程，偏“还原事实” |

Misc 可能一张图就结束；取证通常要你在几十万个文件、几百万条日志里找到那一条关键记录。

### 0.3 取证思维：先问五个问题

1. **证据从哪来？** 磁盘、内存、流量、日志、手机、云。
2. **什么时间发生的？** 时间戳、时区、时间线。
3. **谁做的？** 用户、进程、IP、账号。
4. **做了什么？** 执行、下载、删除、外传、登录。
5. **留下了什么？** 文件、注册表、日志、缓存、网络连接。

> 💡 新手最容易犯的错：一上来就 `grep flag`。取证题的关键往往不是 flag 字符串，而是“某个文件被谁在什么时候打开过”。先把证据链搭起来，flag 会自己出现。

### 0.4 授权与法律边界

取证涉及大量个人数据。**只分析 CTF 附件、自己设备的数据、明确授权的样本**。不要分析来路不明的他人磁盘、聊天记录、账号数据。

---

## 第一章：取证流程与证据规则

### 1.1 易失性顺序（Order of Volatility）

RFC 3227 给出了收集证据的顺序：**越容易消失的越先收集**。

```text
CPU 寄存器、缓存
   ↓
内存（RAM）
   ↓
交换分区、临时文件
   ↓
网络连接、进程状态
   ↓
磁盘
   ↓
远程日志
   ↓
归档备份、离线介质
```

> ⚠️ 机器一关机，内存里的进程、网络连接、解密密钥就没了。所以真实取证要先抓内存，再做磁盘镜像。CTF 里通常已经给你镜像，但你要知道这个顺序，写报告时用得上。

### 1.2 写保护

真实取证必须保证“分析过程不修改证据”：

- 硬件写保护器（Write Blocker）。
- 软件只读挂载：`mount -o ro`。
- 镜像文件计算哈希后，后续都基于副本分析。

### 1.3 哈希校验

拿到镜像第一件事：

```bash
sha256sum disk.img
md5sum disk.img
```

把哈希写进报告。之后每次复制、挂载、分析，都要能证明“我分析的是同一份数据”。

### 1.4 证据链（Chain of Custody）

记录：

- 谁在什么时间、从哪里拿到证据。
- 证据的编号、描述、哈希。
- 每一次转移、复制、分析。
- 谁接触过、做了什么。

CTF 不要求这么正式，但写 WP 时记录“我用了哪个镜像、算了什么哈希、跑了哪些命令”，会让过程更可信。

### 1.5 取证的四个阶段

```text
1. 识别（Identification）：证据在哪、是什么格式
2. 保全（Preservation）：镜像、哈希、写保护
3. 分析（Analysis）：文件系统、痕迹、内存、流量、时间线
4. 报告（Reporting）：结论、证据、复现步骤
```

---

## 第二章：环境与工具准备

### 2.1 推荐环境

- **虚拟机**：Kali Linux 或 Ubuntu，装取证工具。
- **Windows 虚拟机**：跑 Registry Explorer、KAPE、FTK Imager、Volatility。
- **足够大的硬盘**：镜像动辄几十 GB。
- **快照**：分析前打快照，工具把系统搞乱了可以回滚。

### 2.2 核心工具

| 工具 | 用途 | 平台 |
|------|------|------|
| Autopsy | 图形化取证平台 | Windows/Linux |
| Sleuth Kit（TSK） | 命令行文件系统分析 | Linux |
| FTK Imager | 镜像获取、挂载、导出 | Windows |
| dd / dcfldd | 原始镜像 | Linux |
| ewfacquire | 生成 E01 镜像 | Linux |
| Volatility 3 | 内存取证 | 跨平台 |
| Wireshark / tshark | 流量分析 | 跨平台 |
| KAPE | 批量收集 Windows 痕迹 | Windows |
| Eric Zimmerman Tools | 注册表、LNK、MFT、Prefetch 分析 | Windows |
| Registry Explorer | 注册表浏览器 | Windows |
| plaso / log2timeline | 时间线生成 | Linux |
| hashcat / john | 密码破解 | 跨平台 |
| exiftool | 元数据 | 跨平台 |
| photorec / testdisk | 文件恢复 | 跨平台 |
| HxD / 010 Editor | 十六进制 | Windows |
| CyberChef | 编码/解码 | 浏览器 |

### 2.3 安装示例（Ubuntu）

```bash
sudo apt update
sudo apt install -y sleuthkit autopsy volatility3 wireshark tshark \
    plaso-tools ewf-tools exiftool testdisk photorec hashcat

pip3 install volatility3
```

> 💡 Volatility 3 的命令是 `vol`，不是老版本的 `volatility`。网上很多教程还是 Volatility 2，注意区分。

---

## 第三章：磁盘镜像与文件系统

### 3.1 镜像格式

| 格式 | 说明 |
|------|------|
| `.img` / `.raw` / `.dd` | 原始字节流，最简单 |
| `.E01` | EnCase 格式，带元数据、压缩、分段 |
| `.vmdk` / `.vhd` | 虚拟机磁盘 |
| `.qcow2` | QEMU 磁盘 |
| `.001` / `.002` | 分卷镜像 |

### 3.2 获取镜像

```bash
# 原始镜像
sudo dd if=/dev/sdb of=disk.img bs=4M status=progress

# 带哈希
sudo dcfldd if=/dev/sdb of=disk.img bs=4M hash=sha256 hashlog=hash.txt

# E01
sudo ewfacquire /dev/sdb -t disk -f encase6
```

> ⚠️ `if=` 一定是证据盘，`of=` 一定是你的工作盘。写反了就把证据覆盖了。

### 3.3 看分区

```bash
fdisk -l disk.img
mmls disk.img
```

`mmls` 输出示例：

```text
DOS Partition Table
Offset Sector: 0
Units are in 512-byte sectors

      Slot      Start        End          Length       Description
      000:  Meta      0000000000   0000000000   0000000001   Primary Table (#0)
      001:  -------   0000000000   0000002047   0000002048   Unallocated
      002:  000:000   0000002048   0000204799   0000202752   NTFS / exFAT (0x07)
```

分区起始扇区（`Start`）乘以 512 就是字节偏移。

### 3.4 只读挂载

```bash
sudo mkdir -p /mnt/evidence
sudo mount -o ro,loop,offset=$((2048*512)) disk.img /mnt/evidence
```

`offset` 是分区起始字节。挂载后建议再确认：

```bash
mount | grep evidence
```

### 3.5 Sleuth Kit 常用命令

```bash
# 列出分区
mmls disk.img

# 列出文件（-r 递归，-o 分区偏移）
fls -r -o 2048 disk.img

# 提取 inode 对应的文件
icat -o 2048 disk.img 12345 > recovered.bin

# 看文件系统信息
fsstat -o 2048 disk.img

# 看未分配空间
blkls -o 2048 disk.img > unallocated.bin
```

### 3.6 文件系统要点

| 文件系统 | 关键结构 | 取证价值 |
|----------|----------|----------|
| NTFS | MFT、`$J`、`$LogFile`、ADS | 文件名、时间戳、删除记录、隐藏数据流 |
| FAT32 | FAT 表、目录项 | 删除文件、碎片 |
| exFAT | 目录项、簇位图 | 大容量存储 |
| ext4 | inode、journal | 删除文件、日志 |
| APFS | 对象映射、快照 | macOS/iOS |

### 3.7 NTFS 的四个时间戳

| 时间戳 | 含义 |
|--------|------|
| `$STANDARD_INFORMATION` 创建时间 | 文件创建 |
| `$STANDARD_INFORMATION` 修改时间 | 属性修改 |
| `$FILE_NAME` 创建时间 | 文件名创建 |
| `$FILE_NAME` 修改时间 | 文件名修改 |

> 💡 时间戳可以被篡改（timestomp）。如果 `$STANDARD_INFORMATION` 和 `$FILE_NAME` 时间不一致，或者时间早于系统安装时间，就要警惕。

### 3.8 删除文件恢复

- NTFS：删除后 MFT 记录可能还在，数据簇可能未被覆盖。
- `photorec` 按文件签名恢复。
- `testdisk` 修复分区表。
- Autopsy 里直接看“Deleted Files”。

```bash
photorec disk.img
```

### 3.9 隐藏数据

- **NTFS ADS（Alternate Data Streams）**：`file.txt:secret.txt`。
- **slack space**：文件末尾未使用的空间。
- **未分配空间**：删除文件残留。
- **分区间隙**：分区之间的空隙。

```bash
# 看 ADS（Windows）
dir /r

# 看 slack space（TSK）
blkls -s -o 2048 disk.img > slack.bin
```

---

## 第四章：Windows 痕迹分析

这一章是 Windows 取证的核心。每个痕迹都按“路径 → 内容 → 取证价值 → 工具”来讲。

### 4.1 注册表 Hive

| Hive | 路径 | 内容 |
|------|------|------|
| SAM | `C:\Windows\System32\config\SAM` | 本地用户、密码哈希 |
| SYSTEM | `C:\Windows\System32\config\SYSTEM` | 系统配置、服务、启动项、USB 历史 |
| SOFTWARE | `C:\Windows\System32\config\SOFTWARE` | 安装软件、系统设置 |
| SECURITY | `C:\Windows\System32\config\SECURITY` | 安全策略、LSA 机密 |
| NTUSER.DAT | `C:\Users\<user>\NTUSER.DAT` | 用户设置、最近文件、UserAssist |
| UsrClass.dat | `C:\Users\<user>\AppData\Local\Microsoft\Windows\UsrClass.dat` | Shellbags、文件关联 |

工具：Registry Explorer、`regripper`、`hivexsh`。

### 4.2 Prefetch

```text
C:\Windows\Prefetch\*.pf
```

记录程序执行过几次、最后执行时间、加载的 DLL。

工具：`PECmd`（Eric Zimmerman）。

```bash
PECmd.exe -d C:\Windows\Prefetch --csv out
```

> 💡 Prefetch 默认只在客户端版 Windows 开启，服务器版可能关闭。没有 Prefetch 不代表没执行过。

### 4.3 Amcache

```text
C:\Windows\AppCompat\Programs\Amcache.hve
```

记录安装/执行过的程序、路径、SHA1、首次看到时间。

工具：`AmcacheParser`。

### 4.4 ShimCache（AppCompatCache）

```text
SYSTEM\CurrentControlSet\Control\Session Manager\AppCompatCache
```

记录程序“存在过/执行过”的痕迹，即使 Prefetch 被删也可能保留。

工具：`AppCompatCacheParser`。

### 4.5 UserAssist

```text
NTUSER.DAT\Software\Microsoft\Windows\CurrentVersion\Explorer\UserAssist
```

记录用户通过资源管理器启动的程序、运行次数、最后运行时间（ROT13 编码）。

工具：Registry Explorer 自动解码。

### 4.6 LNK 与 Jump Lists

| 痕迹 | 路径 |
|------|------|
| 最近文件快捷方式 | `C:\Users\<user>\AppData\Roaming\Microsoft\Windows\Recent\*.lnk` |
| 任务栏跳转列表 | `...\Recent\AutomaticDestinations\*.automaticDestinations-ms` |
| 自定义跳转列表 | `...\Recent\CustomDestinations\*.customDestinations-ms` |

记录打开过的文件路径、卷序列号、时间。

工具：`LECmd`、`JLECmd`。

### 4.7 Shellbags

```text
UsrClass.dat\Local Settings\Software\Microsoft\Windows\Shell\Bags
```

记录用户浏览过的文件夹（即使文件夹已删除）。

工具：Registry Explorer、`ShellBagsExplorer`。

### 4.8 SRUM

```text
C:\Windows\System32\sru\SRUDB.dat
```

系统资源使用记录：哪个程序用了多少网络流量、CPU、内存，按小时统计。

工具：`SrumECmd`。

### 4.9 事件日志

```text
C:\Windows\System32\winevt\Logs\*.evtx
```

重点日志：

| 日志 | 事件 ID | 含义 |
|------|---------|------|
| Security | 4624 | 登录成功 |
| Security | 4625 | 登录失败 |
| Security | 4672 | 管理员登录 |
| Security | 4688 | 进程创建 |
| Security | 1102 | 审计日志被清空 |
| System | 7045 | 服务安装 |
| PowerShell | 4104 | 脚本块记录 |
| TaskScheduler | 106 | 计划任务注册 |
| Windows Defender | 1116/1117 | 恶意软件检测 |

工具：Event Viewer、`EvtxECmd`、`evtx_dump.py`。

### 4.10 其他常用痕迹

| 痕迹 | 路径/位置 | 价值 |
|------|-----------|------|
| 回收站 | `C:\$Recycle.Bin\<SID>\$I`、`$R` | 删除的文件 |
| 缩略图缓存 | `...\Explorer\thumbcache_*.db` | 看过的图片 |
| 最近文档 | `NTUSER.DAT\...\RecentDocs` | 最近打开的文件 |
| 运行对话框 | `NTUSER.DAT\...\RunMRU` | 执行过的命令 |
| USB 历史 | `SYSTEM\...\Enum\USBSTOR` | 插过哪些 U 盘 |
| 网络历史 | `NTUSER.DAT\...\Network` | 访问过的共享 |
| WMI 持久化 | `SOFTWARE\Microsoft\Wbem` | 恶意持久化 |
| 启动项 | `Run`、`RunOnce`、启动文件夹 | 自启动 |

### 4.11 用 KAPE 批量收集

KAPE（Kroll Artifact Parser and Extractor）可以按目标（Target）收集痕迹，再按模块（Module）解析。

```text
KAPE.exe --tsource C: --tdest D:\out --target !SANS_Triage
```

适合真实应急，也适合 CTF 里快速把常见痕迹打包。

---

## 第五章：内存取证

### 5.1 为什么内存重要

内存里有：

- 正在运行的进程、命令行。
- 网络连接。
- 解密后的文件、密钥。
- 注入的 shellcode。
- 已删除但还没被覆盖的数据。

### 5.2 Volatility 3 基本命令

```bash
vol -f memory.raw windows.info
vol -f memory.raw windows.pslist
vol -f memory.raw windows.psscan
vol -f memory.raw windows.cmdline
vol -f memory.raw windows.netscan
vol -f memory.raw windows.filescan
vol -f memory.raw windows.malfind
```

### 5.3 进程分析

```bash
vol -f memory.raw windows.pslist > pslist.txt
vol -f memory.raw windows.psscan > psscan.txt
```

对比两者：

- `pslist` 走进程链表。
- `psscan` 扫描内存池。
- `psscan` 有、`pslist` 没有 → 可能是被隐藏的进程。

看进程树和命令行：

```bash
vol -f memory.raw windows.cmdline
vol -f memory.raw windows.pstree
```

### 5.4 文件提取

```bash
vol -f memory.raw windows.filescan > files.txt
grep -i "flag" files.txt
grep -i "\.txt" files.txt
```

拿到虚拟地址后：

```bash
vol -f memory.raw windows.dumpfiles --virtaddr 0x7ff...
```

### 5.5 网络连接

```bash
vol -f memory.raw windows.netscan
```

找外联 IP、端口、进程名。结合 `cmdline` 判断是不是恶意程序。

### 5.6 恶意代码

```bash
vol -f memory.raw windows.malfind
```

`malfind` 找“可执行 + 可写 + 无文件映射”的内存页，常用于发现注入的 shellcode。

### 5.7 密码与哈希

```bash
vol -f memory.raw windows.hashdump
vol -f memory.raw windows.lsadump
```

> ⚠️ 这些命令涉及敏感凭据，只用于授权的取证环境。

### 5.8 Linux 内存

```bash
vol -f memory.raw linux.pslist
vol -f memory.raw linux.bash
vol -f memory.raw linux.files
```

### 5.9 内存镜像里的字符串

```bash
strings -a -n 8 memory.raw | grep -i "flag{"
strings -el memory.raw | grep -i "password"
```

简单粗暴，但经常有效。

---

## 第六章：网络流量取证

### 6.1 先看协议分布

Wireshark → Statistics → Protocol Hierarchy。

异常点：

- 大量 DNS 查询。
- 大量 ICMP。
- 非标准端口。
- 明文协议（FTP、Telnet、HTTP）。

### 6.2 常用过滤

```text
http
http.request.method == "POST"
http.response.code == 200
dns
ftp
telnet
usb.capdata
tcp.stream eq 0
ip.addr == 192.168.1.10
```

### 6.3 导出对象

Wireshark → 文件 → 导出对象 → HTTP / SMB / TFTP。

能直接导出图片、压缩包、可执行文件。

### 6.4 tshark 命令行

```bash
# HTTP 请求列表
tshark -r traffic.pcap -Y "http.request" -T fields -e http.host -e http.request.uri

# 跟踪 TCP 流
tshark -r traffic.pcap -q -z follow,tcp,ascii,0

# 提取 HTTP 文件数据
tshark -r traffic.pcap -Y "http.file_data" -T fields -e http.file_data

# USB HID
tshark -r usb.pcap -T fields -e usb.capdata > usbdata.txt
```

### 6.5 USB 键盘流量

见《CTF Misc 入门手册》的详细解析脚本。核心：

- 8 字节 HID 报告。
- 第 0 字节：修饰键。
- 第 2-7 字节：键码。

### 6.6 DNS / ICMP 隧道

- DNS：看 `dns.qry.name`，把子域名拼起来，常是 base32/base64。
- ICMP：看 `data.data`，提取 payload。
- HTTP：看 Cookie、User-Agent、URL 里的长字符串。

### 6.7 TLS 解密

如果题目给了 keylog：

Wireshark → 编辑 → 首选项 → Protocols → TLS → (Pre)-Master-Secret log filename。

### 6.8 常见题型

| 题型 | 思路 |
|------|------|
| 网站被黑 | 导出 HTTP 对象，找 webshell |
| 数据外带 | DNS/ICMP 异常 |
| 键盘记录 | USB HID |
| 邮件附件 | SMTP 导出附件 |
| 数据库泄露 | 跟踪 MySQL 流量 |
| 恶意下载 | HTTP 对象里的 exe/dll |

---

## 第七章：浏览器与邮件取证

### 7.1 Chrome / Edge

```text
C:\Users\<user>\AppData\Local\Google\Chrome\User Data\Default\
  History
  Login Data
  Cookies
  Web Data
  Bookmarks
  Cache/
  Code Cache/
```

`History` 是 SQLite：

```sql
SELECT url, title, visit_count, last_visit_time FROM urls ORDER BY last_visit_time DESC;
SELECT url, visit_time FROM visits ORDER BY visit_time DESC;
```

Chrome 时间戳是“1601-01-01 起的微秒数”，需要转换。

`Login Data` 里的密码用 Windows DPAPI 加密，需要用户上下文或主密钥。

### 7.2 Firefox

```text
C:\Users\<user>\AppData\Roaming\Mozilla\Firefox\Profiles\<profile>\
  places.sqlite
  logins.json
  cookies.sqlite
```

`places.sqlite` 里 `moz_places`、`moz_historyvisits`。

### 7.3 邮件

- `.eml`：文本格式，看 `Received`、`From`、`To`、`Subject`、附件。
- `.msg`：Outlook 单封邮件。
- `.pst` / `.ost`：Outlook 邮箱数据库。

```bash
# 提取 pst
readpst -o out mailbox.pst

# 看 eml 头
head -50 mail.eml
```

邮件头重点：

- `Received`：经过哪些服务器。
- `Return-Path`：真实发件人。
- `SPF` / `DKIM` / `DMARC`：是否伪造。
- `Message-ID`：邮件唯一标识。

### 7.4 浏览器缓存里的图片

Chrome 缓存不是直接可读的图片，可以用工具提取：

- `chrome-cache-parser`
- `MZCacheView`
- 或直接 `strings` 找文件头。

---

## 第八章：手机取证

### 8.1 Android

常见数据位置：

```text
/data/data/<package>/         应用私有数据
/data/media/0/                内部存储（照片、下载）
/sdcard/                      同上（软链接）
/system/                      系统分区
```

没有 root 时：

```bash
adb backup -apk -shared -all -f backup.ab
# 用 abe 解包
abe unpack backup.ab backup.tar
```

有 root 时：

```bash
adb shell
su
cp -r /data/data/com.example.app /sdcard/
adb pull /sdcard/com.example.app
```

### 8.2 iOS

iTunes/Finder 备份：

```text
%APPDATA%\Apple Computer\MobileSync\Backup\<UDID>\
  Manifest.db
  Manifest.plist
  <hash 文件>
```

`Manifest.db` 记录文件映射，可以用 `iLEAPP`、`iMazing`、`libimobiledevice` 解析。

### 8.3 CTF 常见手机题

- 给一个 `.ab` 备份，解出聊天记录/图片。
- 给一个 APK 的数据目录，找 SQLite 数据库。
- 给一个 iOS 备份，找 `Manifest.db` 里的关键文件。
- 图片 EXIF 里有 GPS。

### 8.4 数据库分析

```bash
sqlite3 app.db ".tables"
sqlite3 app.db "SELECT * FROM messages LIMIT 10;"
```

常见表：`messages`、`contacts`、`calllog`、`accounts`。

---

## 第九章：云与日志取证

### 9.1 AWS CloudTrail

记录 API 调用：谁、什么时候、从哪个 IP、做了什么。

重点事件：

```text
ConsoleLogin
CreateUser
AttachUserPolicy
CreateAccessKey
RunInstances
DeleteTrail
StopLogging
```

### 9.2 Azure Activity Log

记录订阅级操作：创建资源、修改权限、删除资源。

### 9.3 Microsoft 365 Unified Audit Log

记录 Exchange、SharePoint、Teams、OneDrive 操作。

重点：

```text
UserLoggedIn
New-InboxRule
Set-Mailbox
FileDownloaded
FileAccessed
```

### 9.4 Kubernetes 审计日志

记录 `kubectl` / API 请求：谁创建了 Pod、读了 Secret、执行了 exec。

### 9.5 CTF 里怎么考

- 给一段 CloudTrail JSON，找被盗用的 Access Key。
- 给 M365 日志，找异常登录 IP。
- 给 K8s 审计日志，找创建恶意 Pod 的用户。

---

## 第十章：时间线分析

### 10.1 为什么要做时间线

单看一个文件、一条日志，很难判断因果。把所有事件按时间排好，故事就出来了。

### 10.2 用 plaso 生成时间线

```bash
log2timeline.py timeline.plaso evidence/
psort.py -o timeline.csv timeline.plaso
```

### 10.3 用 mactime

```bash
# 先生成 bodyfile
fls -r -m / -o 2048 disk.img > bodyfile.txt

# 再生成时间线
mactime -b bodyfile.txt -d > timeline.csv
```

### 10.4 时间戳的坑

- 时区：UTC vs 本地时间。
- NTFS 时间戳精度 100ns。
- FAT 时间戳精度 2 秒。
- 浏览器/数据库可能用自己的纪元（Chrome 1601、Unix 1970）。
- 时间戳可以被篡改。

> 💡 看到“文件创建时间早于系统安装时间”“所有文件时间完全一样”“时间戳是整点”，都要怀疑。

---

## 第十一章：反取证与对抗

### 11.1 常见反取证手段

| 手段 | 表现 | 识别 |
|------|------|------|
| 时间戳篡改 | 时间与内容不符 | `$SI`/`$FN` 不一致 |
| 日志清除 | Security 1102 | 事件日志缺失 |
| USN 日志删除 | `$J` 很小/为空 | 文件操作记录缺失 |
| 安全删除 | 数据被覆盖 | 无法恢复 |
| 加密 | 文件不可读 | 熵高、文件头异常 |
| 隐写 | 图片/音频异常 | 隐写分析 |
| 容器逃逸 | 进程/命名空间异常 | 内存/日志分析 |
| 卷影副本删除 | 无法回滚 | `vssadmin` 记录 |

### 11.2 熵分析

```bash
binwalk -E file
```

熵接近 8 通常是加密/压缩数据；突然升高的区域可能是隐藏的加密块。

### 11.3 日志被清空

Windows Security 事件：

- 1102：审计日志被清除。
- 104：日志被清除。

如果关键时间段日志缺失，本身就是线索。

---

## 第十二章：报告与复现

### 12.1 报告结构

```markdown
## 案件/题目信息
- 编号：
- 证据来源：
- 哈希：

## 环境与工具
- 

## 分析过程
1. 镜像/内存/流量获取
2. 文件系统分析
3. 痕迹分析
4. 时间线
5. 关键发现

## 结论
- flag：
- 攻击链：
- 证据列表：

## 复现步骤
（命令）
```

### 12.2 证据记录

每条结论都要能指向：

- 文件名/inode。
- 注册表键。
- 日志事件 ID。
- 时间戳。
- 哈希。

> 💡 写 WP 时不要只写“我找到了 flag”，要写“我在哪个文件、哪条记录、用什么命令找到的”。这既是取证思维，也让别人能复现。

---

## 第十三章：完整演练：从镜像到 flag

下面用一个虚构场景，把流程走一遍。

### 13.1 场景

你拿到 `evidence.img`，怀疑有人用 U 盘拷走了文件，并在系统里执行过恶意程序。

### 13.2 第一步：确认镜像

```bash
file evidence.img
sha256sum evidence.img
mmls evidence.img
```

假设输出显示 NTFS 分区从扇区 2048 开始。

### 13.3 第二步：只读挂载

```bash
sudo mount -o ro,loop,offset=$((2048*512)) evidence.img /mnt/evidence
ls -la /mnt/evidence
```

### 13.4 第三步：找用户目录

```bash
ls /mnt/evidence/Users
```

假设有一个用户 `sajunan`。

### 13.5 第四步：看最近文件

```bash
ls -la "/mnt/evidence/Users/sajunan/AppData/Roaming/Microsoft/Windows/Recent"
```

用 `LECmd` 解析 `.lnk`：

```bash
LECmd.exe -d "Recent" --csv out
```

假设发现 `secret.zip.lnk`，目标路径 `D:\backup\secret.zip`。

### 13.6 第五步：看 Prefetch

```bash
PECmd.exe -d "/mnt/evidence/Windows/Prefetch" --csv out
```

假设发现 `7z.exe` 在凌晨 3 点执行过。

### 13.7 第六步：看 USB 历史

用 Registry Explorer 打开 `SYSTEM`，定位：

```text
SYSTEM\CurrentControlSet\Enum\USBSTOR
```

假设发现一个陌生 U 盘，序列号 `0123456789ABCDEF`。

### 13.8 第七步：找删除的文件

```bash
fls -r -o 2048 evidence.img | grep -i secret
```

或 Autopsy 里看 Deleted Files，恢复 `secret.zip`。

### 13.9 第八步：解压/分析

```bash
unzip secret.zip
file *
strings -a *
```

假设里面是一张图片，`exiftool` 的 `Comment` 里藏着 flag。

### 13.10 第九步：写时间线

把 Prefetch 时间、LNK 时间、USB 插入时间、文件删除时间排在一起，还原：

```text
02:58  U 盘插入
03:01  7z.exe 执行
03:03  secret.zip 被复制到 U 盘
03:05  secret.zip 被删除
```

### 13.11 第十步：出报告

记录哈希、命令、证据、结论。整个流程结束。

---

## 第十四章：工具链与学习路线

### 14.1 学习路线

```text
文件系统基础（NTFS/FAT/ext4）
   ↓
镜像获取 + 只读挂载 + Sleuth Kit
   ↓
Windows 痕迹（注册表、Prefetch、LNK、日志）
   ↓
内存取证（Volatility）
   ↓
流量分析（Wireshark）
   ↓
手机/云/时间线
   ↓
综合题 + 报告
```

### 14.2 推荐练习

- **Autopsy 官方示例镜像**：练文件系统浏览、关键字搜索、时间线。
- **Volatility 官方样本**：练内存取证。
- **CyberDefenders**：蓝队取证题，贴近真实。
- **Blue Team Labs Online**：取证/应急题。
- **picoCTF Forensics**：基础题。
- **BUUCTF Misc/取证**：国内题。
- **DFIR Training / SANS 挑战**：进阶。

### 14.3 常用组合拳

```text
镜像 → mmls → fls → icat
镜像 → Autopsy → 关键字 → 时间线
内存 → pslist/psscan → cmdline → dumpfiles
流量 → 导出对象 → USB/HTTP/DNS
Windows → Registry + Prefetch + LNK + Event Log
```

---

## 附录：速查表与参考资料

### A.1 命令速查

```bash
# 镜像
dcfldd if=/dev/sdb of=disk.img bs=4M hash=sha256 hashlog=hash.txt
sha256sum disk.img
mmls disk.img
fls -r -o 2048 disk.img
icat -o 2048 disk.img 123 > file
mount -o ro,loop,offset=$((2048*512)) disk.img /mnt/evidence

# 内存
vol -f memory.raw windows.info
vol -f memory.raw windows.pslist
vol -f memory.raw windows.psscan
vol -f memory.raw windows.cmdline
vol -f memory.raw windows.filescan
vol -f memory.raw windows.dumpfiles --virtaddr 0x...
vol -f memory.raw windows.netscan
vol -f memory.raw windows.malfind

# 时间线
log2timeline.py timeline.plaso evidence/
psort.py -o timeline.csv timeline.plaso
mactime -b bodyfile.txt -d > timeline.csv

# 流量
tshark -r a.pcap -Y "http.request" -T fields -e http.host -e http.request.uri
tshark -r a.pcap -q -z follow,tcp,ascii,0

# 手机
adb backup -apk -shared -all -f backup.ab
abe unpack backup.ab backup.tar
```

### A.2 Windows 痕迹路径速查

| 痕迹 | 路径 |
|------|------|
| SAM | `C:\Windows\System32\config\SAM` |
| SYSTEM | `C:\Windows\System32\config\SYSTEM` |
| SOFTWARE | `C:\Windows\System32\config\SOFTWARE` |
| NTUSER.DAT | `C:\Users\<user>\NTUSER.DAT` |
| Prefetch | `C:\Windows\Prefetch` |
| Amcache | `C:\Windows\AppCompat\Programs\Amcache.hve` |
| SRUM | `C:\Windows\System32\sru\SRUDB.dat` |
| 事件日志 | `C:\Windows\System32\winevt\Logs` |
| 最近文件 | `C:\Users\<user>\AppData\Roaming\Microsoft\Windows\Recent` |
| 回收站 | `C:\$Recycle.Bin` |
| USB 历史 | `SYSTEM\...\Enum\USBSTOR` |

### A.3 参考资料

- RFC 3227（证据收集与归档指南）：<https://datatracker.ietf.org/doc/html/rfc3227>
- Autopsy：<https://www.autopsy.com/>
- Sleuth Kit：<https://www.sleuthkit.org/sleuthkit/>
- Volatility 3：<https://volatility3.readthedocs.io/>
- Eric Zimmerman Tools：<https://ericzimmerman.github.io/>
- KAPE：<https://www.kroll.com/en/services/cyber-risk/incident-response-litigation-support/kroll-artifact-parser-extractor-kape>
- Wireshark：<https://www.wireshark.org/docs/>
- plaso：<https://plaso.readthedocs.io/>
- CyberDefenders：<https://cyberdefenders.org/>

---

> 📌 **最后一句：** 取证不是“找 flag”，而是“用证据讲一个能复现的故事”。当你的每条结论都能指向具体文件、具体时间、具体命令时，flag 只是故事的结尾。
