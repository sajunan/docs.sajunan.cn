---
title: "杂项入门"
description: "CTF Misc 方向入门手册：编码转换、图片隐写、压缩包、音频视频、文档、流量分析、内存与磁盘取证、Git 泄漏与二维码。"
---

# 杂项入门

> **杂项不杂：先看文件，再猜套路，最后写脚本**

Misc（杂项）没有固定套路，考的是观察力、工具熟练度和信息串联能力。本文从文件识别、编码转换讲到图片隐写、压缩包、流量、内存与磁盘取证，并给出完整的实战流程。

## 第零章：Misc 到底是什么

Misc（杂项）是 CTF 里最不像“一个方向”的方向：它可能是给你一张图、一个压缩包、一段流量、一块内存镜像、一个奇怪的编码文本，让你从中找出 flag。

它考的不是某个固定漏洞，而是：

1. **观察力**：能不能看出文件不对劲。
2. **工具熟练度**：知道该用哪个工具、参数是什么。
3. **信息串联**：一层套一层，能不能耐着性子剥完。
4. **脚本能力**：批量处理、像素分析、协议解析，基本都要写 Python。

> 💡 **新手建议：** Misc 是最适合入门的方向。它不要求你懂漏洞原理，先从“文件识别 + 编码解码 + 图片/压缩包”这三板斧开始，就能做出不少题。

### 0.1 安全底线

同样一句话：只分析题目附件和自己有权分析的文件。不要拿别人的流量包、磁盘镜像、账号数据练手。

---

## 第一章：拿到文件先做这五件事

### 1.1 看文件类型

```bash
file mystery
```

输出可能是：

```text
mystery: PNG image data, 800 x 600, 8-bit/color RGBA, non-interlaced
mystery: Zip archive data, at least v2.0 to extract
mystery: ELF 64-bit LSB executable
mystery: ASCII text, with very long lines
```

> ⚠️ 不要相信后缀名。`.jpg` 可能其实是 zip，`.txt` 可能其实是 PNG。`file` 看的是文件头，不是扩展名。

### 1.2 看十六进制

```bash
xxd mystery | head -40
```

重点看：

- 文件头（PNG `89 50 4E 47`、JPEG `FF D8 FF`、ZIP `50 4B 03 04`、GIF `47 49 46 38`）。
- 文件尾（PNG `49 45 4E 44 AE 42 60 82`）后面还有没有数据。
- 中间有没有明显的 ASCII 字符串。

### 1.3 提取字符串

```bash
strings -a -n 6 mystery
strings -el mystery        # UTF-16LE（Windows 程序常见）
```

### 1.4 看元数据

```bash
exiftool mystery
```

图片的 GPS、相机型号、软件、注释、缩略图，都可能是线索。

### 1.5 扫隐藏内容

```bash
binwalk mystery
binwalk -e mystery        # 自动提取
foremost mystery          # 按文件头 carving
```

`binwalk` 能发现“文件里套文件”，比如 PNG 尾部藏 ZIP、PDF 里藏图片。

### 1.6 一张万能流程图

```text
file → xxd/010 → strings → exiftool → binwalk
   │
   ├─ 是图片 → zsteg / Stegsolve / pngcheck
   ├─ 是压缩包 → 伪加密 / 明文攻击 / 爆破
   ├─ 是音频 → 频谱 / SSTV / LSB
   ├─ 是流量 → Wireshark / tshark
   ├─ 是内存 → Volatility
   └─ 是文本 → CyberChef 编码识别
```

---

## 第二章：编码与转换

### 2.1 先判断“它像什么”

| 特征 | 可能编码 |
|------|----------|
| 只有 `A-Z a-z 0-9 + / =` | Base64 |
| 只有 `A-Z 2-7 =` | Base32 |
| 只有 `0-9 A-F` | Hex |
| `%XX%XX` | URL 编码 |
| `\uXXXX` | Unicode 转义 |
| `&#xXXXX;` / `&#NNN;` | HTML 实体 |
| `=XX` 大量出现 | Quoted-printable |
| 只有 `[]()!+` | JSFuck |
| 全是 `ﾟωﾟﾉ` 一类颜文字 | AAEncode |
| 全是 `$=~[]` | JJEncode |
| 只有 `+-<>[].,` | Brainfuck |
| 点划/长短 | 摩斯电码 |
| 0/1 五位一组 | 培根密码 |
| 一串汉字但读不通 | 与佛论禅 / 兽音 / 百家姓等 |

> 💡 **CyberChef 的 Magic 功能**可以先自动猜一轮，但不要完全依赖它。知道每种编码的特征，才能判断它猜得对不对。

### 2.2 Base 家族

```text
Base16 = Hex
Base32：A-Z 2-7，通常带 = 填充
Base64：A-Z a-z 0-9 + /，通常带 = 填充
Base64URL：把 + / 换成 - _，去掉 =
Base58：比特币地址常用，没有 0 O I l
Base62：0-9 A-Z a-z 的不同排列，常见于短链
Base85/Ascii85：包含大量符号
```

Base64 解码：

```python
import base64
print(base64.b64decode("ZmxhZ3t0ZXN0fQ==").decode())
```

> ⚠️ Base64 字符串长度不是 4 的倍数时，先补 `=`；换行、空格要先去掉。

### 2.3 常见“套娃”编码

```text
Base64 → Hex → URL → Base64 → ...
```

写个循环自动尝试：

```python
import base64, binascii, urllib.parse

s = "...."
for _ in range(10):
    try:
        s = base64.b64decode(s).decode()
        print("base64:", s)
        continue
    except Exception:
        pass
    try:
        s = bytes.fromhex(s).decode()
        print("hex:", s)
        continue
    except Exception:
        pass
    s = urllib.parse.unquote(s)
    print("url:", s)
```

### 2.4 摩斯电码

```text
.... . .-.. .-.. ---   .-- --- .-. .-.. -..
```

- `.` / `-`，字母之间空格，单词之间 `/`。
- 也有用 `0/1`、`A/B`、长短音表示的变体。
- 工具：CyberChef 的 “From Morse Code”、在线摩斯解码器。

### 2.5 培根密码

五位二进制表示一个字母：

```text
A = aaaaa
B = aaaab
C = aaaba
...
```

题目常把 A/B 藏在两种字体、两种颜色、大小写里。

### 2.6 键盘与图形编码

| 名称 | 特征 |
|------|------|
| 键盘密码 | 字母围成形状、九宫格坐标 |
| 猪圈密码 | 井字/叉字图形 |
| 当铺密码 | 汉字里“口”的数量 |
| 云影密码 | 数字 0-8 分组 |
| 敲击码 | 两个数字表示行列 |
| 曲路密码 | 按路径读字符 |
| 栅栏密码 | 分行后按列读 |

> 💡 图形类编码先用纸笔画一遍，比盯着屏幕猜快得多。

### 2.7 中文网络编码

| 名称 | 特征 |
|------|------|
| 与佛论禅 | 大量“佛曰”“如是我闻” |
| 兽音译者 | “嗷呜啊~”一类 |
| 社会主义核心价值观 | 12 个词随机组合 |
| 百家姓 | 一串姓氏 |
| 颜文字加密 | 大量日文假名/颜文字 |

这些基本都有现成在线解码器，搜名字即可。

---

## 第三章：图片隐写

### 3.1 PNG 结构

PNG = 8 字节签名 + 若干 chunk：

```text
┌────────┬────────┬──────────┬────────┐
│ 长度 4B │ 类型 4B │ 数据 N B │ CRC 4B │
└────────┴────────┴──────────┴────────┘
```

常见 chunk：

| 类型 | 作用 |
|------|------|
| `IHDR` | 宽、高、色深、颜色类型 |
| `PLTE` | 调色板 |
| `IDAT` | 压缩后的像素数据 |
| `IEND` | 文件结束 |
| `tEXt` / `zTXt` / `iTXt` | 文本元数据（常藏 flag） |

检查：

```bash
pngcheck -v image.png
exiftool image.png
strings -a image.png | grep -i flag
```

> ⚠️ `IEND` 后面还有数据，是最常见的“尾巴藏东西”。用 `binwalk` 或 `dd` 切出来。

### 3.2 LSB 隐写

把信息写进像素值的最低位，肉眼看不出变化。

```bash
zsteg -a image.png
```

`zsteg` 会自动尝试多种通道、位平面、位序组合。

也可以用 Stegsolve：

- 切换 RGB 通道。
- 看 bit plane（0-7）。
- 看 frame browser（GIF/APNG）。
- 看 stereo/random color。

### 3.3 图片高度/宽度被改

PNG 的 IHDR 里记录了宽高。题目可能把高度改小，导致下半部分不显示。

- 用 `pngcheck` 看 CRC 是否报错。
- 用 010 Editor 的 PNG 模板改回正确高度。
- 改完要重算 CRC，或直接用工具修复。

### 3.4 JPEG 隐写

```bash
steghide extract -sf image.jpg
# 需要密码时试空密码、题目提示、常见弱密码
outguess -r image.jpg out.txt
jphide / jpseek
```

JPEG 还可以看 DCT 系数、缩略图、EXIF。

### 3.5 GIF / APNG

```bash
# 拆帧
convert image.gif -coalesce frames/%03d.png
ffmpeg -i image.gif frames/%03d.png

# 看每帧差异
```

多帧图片常把 flag 拆到不同帧，或只在某一帧显示。

### 3.6 盲水印

肉眼看不见，但频域里能恢复：

```bash
# 常见工具
blind_watermark
invisible-watermark
```

题目会给原图和水印图，要求提取或对比。

### 3.7 图片修复

- 文件头损坏：对照正常 PNG/JPEG 头补字节。
- 缺少 IEND：补 `00 00 00 00 49 45 4E 44 AE 42 60 82`。
- 宽高错误：改 IHDR 并重算 CRC。
- 颜色异常：检查颜色类型、位深、调色板。

### 3.8 像素级操作

```python
from PIL import Image

im = Image.open("image.png").convert("RGB")
w, h = im.size
for y in range(h):
    for x in range(w):
        r, g, b = im.getpixel((x, y))
        # 按题目规则处理
```

常见套路：

- 像素值转 ASCII。
- 通道分离后拼图。
- 按坐标排序像素。
- 二维码被切成碎片，按编号拼回。

---

## 第四章：压缩包

### 4.1 先看结构

```bash
file archive.zip
zipdetails archive.zip
unzip -l archive.zip
unzip -z archive.zip     # 看注释
```

### 4.2 ZIP 伪加密

ZIP 的“加密”标志位在：

- 本地文件头偏移 `6-7`（General Purpose Bit Flag）。
- 中央目录偏移 `8-9`。

如果标志位被改成 `1`，但数据其实没加密，就是伪加密。把对应位改回 `0` 即可。

工具：

```bash
ZipCenOp.jar r archive.zip
```

或直接用 010 Editor 改字节。

### 4.3 明文攻击（bkcrack）

ZIP 传统加密（ZipCrypto）在已知明文时可以恢复密钥。要求至少 **12 字节已知明文，其中至少 8 字节连续**。

```bash
# 查看压缩包信息
bkcrack -L archive.zip

# 已知明文在 plain.txt，压缩包内文件名是 secret.txt
bkcrack -C archive.zip -c secret.txt -p plain.txt

# 恢复出内部密钥后，解密
bkcrack -C archive.zip -k <key1> <key2> <key3> -U unlocked.zip password
```

> 💡 题目常给一张“已知图片”和压缩包里的同名图片，这就是标准明文攻击场景。注意只对 ZipCrypto 有效，AES 加密的 ZIP 不行。

### 4.4 密码爆破

```bash
fcrackzip -u -D -p rockyou.txt archive.zip

zip2john archive.zip > hash.txt
john hash.txt --wordlist=rockyou.txt
```

### 4.5 CRC32 相关

- 小文件可以枚举内容使 CRC32 匹配。
- 题目给 CRC 值和要求长度，就是在让你写爆破脚本。
- 工具：`crc32`、`zipcrash`、自己写 Python。

### 4.6 嵌套与损坏

```bash
binwalk -e archive.zip
7z x archive.zip
unrar x archive.rar
```

- 压缩包套压缩包：写循环自动解。
- 文件头损坏：补 `PK\x03\x04`。
- 分卷压缩：把所有分卷放同一目录再解。
- 注释藏 base64：`unzip -z`。

### 4.7 其他压缩格式

| 格式 | 工具 |
|------|------|
| ZIP | unzip、7z、bkcrack |
| RAR | unrar、7z |
| 7z | 7z |
| TAR | tar |
| GZ | gzip |
| XZ | xz |
| CAB | cabextract |

---

## 第五章：音频隐写

### 5.1 先听一遍

很多题听一遍就有感觉：

- 有杂音/电流声 → 可能藏数据。
- 只有左右声道之一有声音 → 分离声道看看。
- 倒放像人声 → 反转。
- 有规律滴滴声 → 摩斯/DTMF/SSTV。

### 5.2 看频谱图

用 Audacity：

1. 导入音频。
2. 选中轨道。
3. 视图 → 频谱图（Spectrogram）。
4. 调高对比度，看有没有文字/图案。

常见：频谱里直接写着 flag。

### 5.3 SSTV

慢扫描电视，把图片编码成声音。工具：

- RX-SSTV（Windows）
- QSSTV（Linux）
- `sstv` Python 库

把音频播放给解码软件，就能还原图片。

### 5.4 DTMF

电话按键音：

```bash
multimon-ng -a DTMF -t wav audio.wav
```

### 5.5 摩斯

```bash
# 先转成点划文本，再解码
```

Audacity 里看波形长短，或用工具自动识别。

### 5.6 LSB 音频隐写

```bash
silenteye
DeepSound
```

也可以自己写脚本读 WAV 采样值的最低位：

```python
import wave
w = wave.open("audio.wav", "rb")
frames = w.readframes(w.getnframes())
bits = [sample & 1 for sample in frames]
```

### 5.7 声道与反转

```bash
# 分离声道
ffmpeg -i audio.mp3 -map_channel 0.0.0 left.wav -map_channel 0.0.1 right.wav

# 反转
ffmpeg -i audio.mp3 -af areverse reversed.wav
```

左右声道相减，常能抵消人声、留下隐藏信息。

---

## 第六章：视频

### 6.1 抽帧

```bash
ffmpeg -i video.mp4 -vf fps=1 frames/%04d.png
ffmpeg -i video.mp4 -vf fps=10 frames/%04d.png
```

逐帧看，或在帧之间做差分。

### 6.2 看轨道

```bash
ffprobe video.mp4
mkvinfo video.mkv
```

可能有多条音轨、字幕轨、章节信息。

```bash
mkvextract tracks video.mkv 1:video.h264 2:audio.aac 3:subs.srt
```

### 6.3 帧差与拼接

- 两帧相减得到隐藏图案。
- 把每帧某个像素拼成一张图。
- 视频某几帧只显示一瞬间。

### 6.4 视频里的音频

先把音频抽出来，再按音频隐写处理：

```bash
ffmpeg -i video.mp4 -vn -acodec copy audio.aac
```

---

## 第七章：文档与 Office

### 7.1 PDF

```bash
pdfinfo file.pdf
pdftotext file.pdf -
pdfimages -all file.pdf img
pdf-parser.py file.pdf
peepdf file.pdf
qpdf --decrypt file.pdf out.pdf
```

常见藏点：

- 隐藏文字（白色、字号 0）。
- 注释、附件。
- 多层图片。
- JavaScript。
- 元数据。

### 7.2 Office

`.docx` / `.xlsx` / `.pptx` 本质是 zip：

```bash
unzip document.docx -d docx_out
```

看 `word/document.xml`、`xl/sharedStrings.xml`、`ppt/slides/`。

### 7.3 宏

```bash
olevba document.doc
olevba --decode document.doc
```

宏代码里可能有下载地址、加密字符串、flag。

### 7.4 加密文档

```bash
msoffcrypto-tool document.docx decrypted.docx -p password
```

### 7.5 隐藏内容

- 白色字体。
- 隐藏工作表/行/列。
- 批注。
- 文档属性。
- 页眉页脚。
- 修订记录。

---

## 第八章：流量分析

### 8.1 Wireshark 常用过滤

```text
http
http.request.method == "POST"
http.response.code == 200
ip.addr == 192.168.1.1
tcp.stream eq 0
dns
ftp
telnet
usb.capdata
```

### 8.2 导出对象

Wireshark → 文件 → 导出对象 → HTTP / SMB / TFTP。

能直接导出图片、压缩包、可执行文件。

### 8.3 tshark 命令行

```bash
# 列出所有 HTTP 请求
tshark -r traffic.pcap -Y "http.request" -T fields -e http.host -e http.request.uri

# 导出 HTTP 文件数据
tshark -r traffic.pcap -Y "http.file_data" -T fields -e http.file_data

# 跟踪某个 TCP 流
tshark -r traffic.pcap -q -z follow,tcp,ascii,0
```

### 8.4 USB 键盘流量

题目给一段 USB 流量，让你还原键盘输入。

```bash
tshark -r usb.pcap -T fields -e usb.capdata > usbdata.txt
```

8 字节 HID 报告里：

- 第 0 字节：修饰键（Shift、Ctrl 等）。
- 第 2-7 字节：按下的键码。

解析脚本（经典写法）：

```python
keys = {
    0x04: 'a', 0x05: 'b', 0x06: 'c', 0x07: 'd', 0x08: 'e',
    0x09: 'f', 0x0a: 'g', 0x0b: 'h', 0x0c: 'i', 0x0d: 'j',
    0x0e: 'k', 0x0f: 'l', 0x10: 'm', 0x11: 'n', 0x12: 'o',
    0x13: 'p', 0x14: 'q', 0x15: 'r', 0x16: 's', 0x17: 't',
    0x18: 'u', 0x19: 'v', 0x1a: 'w', 0x1b: 'x', 0x1c: 'y',
    0x1d: 'z', 0x1e: '1', 0x1f: '2', 0x20: '3', 0x21: '4',
    0x22: '5', 0x23: '6', 0x24: '7', 0x25: '8', 0x26: '9',
    0x27: '0', 0x2d: '-', 0x2e: '=', 0x2f: '[', 0x30: ']',
    0x31: '\\', 0x33: ';', 0x34: "'", 0x36: ',', 0x37: '.',
    0x38: '/', 0x2c: ' ',
}

out = ""
for line in open("usbdata.txt"):
    line = line.strip()
    if len(line) < 6:
        continue
    code = int(line[4:6], 16)
    if code in keys:
        out += keys[code]
print(out)
```

> 💡 新版 tshark 可以直接导出 `usbhid.boot_report.keyboard.keycode_1`，不用自己切字节。不同系统抓的包字段可能不同，先 `tshark -G fields | grep usbhid` 看一下。

### 8.5 DNS / ICMP 隧道

- DNS：看 `dns.qry.name`，把子域名拼起来。
- ICMP：看 `data.data`，把 payload 提取出来。
- HTTP：看 Cookie、User-Agent 里的异常长字符串。

### 8.6 TLS 解密

如果题目给了 `SSLKEYLOGFILE`：

Wireshark → 编辑 → 首选项 → Protocols → TLS → (Pre)-Master-Secret log filename。

### 8.7 常见题型

| 题型 | 思路 |
|------|------|
| 网站被黑 | 导出 HTTP 对象，找 webshell |
| 数据外带 | 看 DNS/ICMP 异常 |
| 键盘记录 | USB HID 解析 |
| 邮件附件 | 导出 SMTP 附件 |
| 数据库泄露 | 跟踪 MySQL 流量 |

---

## 第九章：内存取证

### 9.1 Volatility 3 基本用法

```bash
vol -f memory.raw windows.info
vol -f memory.raw windows.pslist
vol -f memory.raw windows.psscan
vol -f memory.raw windows.cmdline
vol -f memory.raw windows.netscan
vol -f memory.raw windows.filescan
```

### 9.2 找进程

```bash
vol -f memory.raw windows.pslist | grep -i flag
```

- `pslist` 走进程链表，可能被 rootkit 隐藏。
- `psscan` 扫描内存池，能找到被隐藏的进程。
- 两者对比是发现 DKOM 隐藏进程的经典方法。

### 9.3 找文件

```bash
vol -f memory.raw windows.filescan > files.txt
grep -i flag files.txt
vol -f memory.raw windows.dumpfiles --virtaddr 0x...
```

### 9.4 看命令行

```bash
vol -f memory.raw windows.cmdline
```

能还原执行过的命令、启动参数。

### 9.5 看网络

```bash
vol -f memory.raw windows.netscan
```

找可疑外联 IP、端口。

### 9.6 找恶意代码

```bash
vol -f memory.raw windows.malfind
```

列出可疑的内存页（可执行、可写、无文件映射），常用于发现注入的 shellcode。

### 9.7 Linux 内存

```bash
vol -f memory.raw linux.pslist
vol -f memory.raw linux.bash
vol -f memory.raw linux.files
```

### 9.8 其他

- `windows.hashdump`：导出 SAM 哈希。
- `windows.lsadump`：导出 LSA 机密。
- `windows.registry.printkey`：读注册表。
- `windows.svcscan`：服务列表。

---

## 第十章：磁盘与文件恢复

### 10.1 看分区

```bash
fdisk -l disk.img
mmls disk.img
```

### 10.2 列文件

```bash
fls -r -o <offset> disk.img
icat -o <offset> disk.img <inode> > recovered_file
```

Sleuth Kit 的 `fls` / `icat` 可以列出和提取文件，包括被删除的。

### 10.3 恢复删除文件

```bash
photorec disk.img
testdisk disk.img
```

`photorec` 按文件签名恢复，`testdisk` 修复分区表。

### 10.4 图形化工具

- **Autopsy**：Sleuth Kit 的图形界面，适合浏览文件系统、时间线、关键词。
- **FTK Imager**：挂载镜像、导出文件。
- **X-Ways**：商业工具。

### 10.5 Windows 取证点

| 位置 | 内容 |
|------|------|
| `$MFT` | 文件记录，含文件名、时间戳 |
| `$Recycle.Bin` | 回收站 |
| `NTUSER.DAT` | 用户注册表 |
| `Prefetch` | 程序执行记录 |
| `Amcache` | 程序安装/执行信息 |
| `Event Logs` | 登录、服务、PowerShell 日志 |

---

## 第十一章：Git 与源码泄漏

### 11.1 发现 .git

```text
http://target/.git/HEAD
http://target/.git/config
```

### 11.2 拉取仓库

```bash
git-dumper http://target/.git/ repo
```

### 11.3 找被删的内容

```bash
cd repo
git log --all --oneline
git reflog
git fsck --lost-found
git cat-file -p <commit>
git diff <commit1> <commit2>
```

> 💡 flag 经常在“已经被删除的提交”里。`git log --all` 看所有分支，`git fsck --lost-found` 找悬空对象。

### 11.4 SVN / 备份

```text
/.svn/entries
/.svn/wc.db
```

工具：`svn-extractor`、`dvcs-ripper`。

---

## 第十二章：二维码与条码

### 12.1 识别

```bash
zbarimg qrcode.png
```

Python：

```python
from pyzbar.pyzbar import decode
from PIL import Image

print(decode(Image.open("qrcode.png")))
```

### 12.2 二维码被遮挡/损坏

- 二维码有纠错等级（L/M/Q/H），损坏一部分仍可识别。
- 三个定位角（左上、右上、左下）必须完整。
- 可以手工重建缺失模块，或用工具修复。

### 12.3 二维码结构

| 区域 | 作用 |
|------|------|
| 定位图案 | 三个回字形 |
| 校正图案 | 右下附近的小方块 |
| 格式信息 | 纠错等级、掩码 |
| 数据区 | 编码后的内容 |
| 时序图案 | 黑白相间的一行一列 |

### 12.4 条码

- Code 128、Code 39、EAN-13。
- 工具：`zbarimg` 也支持部分一维码。

---

## 第十三章：杂项常见套路

### 13.1 压缩包套娃

```bash
while file data != "data"; do
  # 根据类型解压
done
```

实际写脚本时，按 `file` 输出分支处理 zip/rar/7z/tar/gz。

### 13.2 文件拼接

```bash
cat part1 part2 > full
```

或图片按上下/左右拼接：

```bash
convert +append a.png b.png out.png
convert -append a.png b.png out.png
```

### 13.3 像素与坐标

- 按题目给的坐标顺序取像素。
- 把一维数组按宽高重排。
- 通道分离后单独看。
- 颜色值转 ASCII。

### 13.4 游戏存档

很多游戏存档是 JSON/XML/二进制：

- 直接看文本。
- base64 解码。
- 找金币、分数、关卡字段。
- 用 010 改数值。

### 13.5 文档隐藏

- Word 白色文字。
- Excel 隐藏行列/工作表。
- PPT 备注。
- PDF 多层内容。
- 文件属性/自定义属性。

### 13.6 区块链

- 题目给交易哈希，去区块浏览器看 input data。
- `cast` / `web3.py` 解码。
- UTF-8/Hex 转换。

### 13.7 虚拟机镜像

- `.vmdk` / `.ova` / `.qcow2`。
- 用 7z 解开，或挂载后看文件系统。
- 找历史命令、浏览器记录、下载文件。

### 13.8 常见“送分点”

```text
strings 里有 flag
exiftool 的 Comment 里有 flag
zip 注释里有 base64
图片 IEND 后面有 zip
PNG 的 tEXt 里有 flag
音频频谱里有 flag
流量 HTTP 导出对象里有 flag
```

---

## 第十四章：工具链与学习路线

### 14.1 工具速查

| 场景 | 工具 |
|------|------|
| 文件识别 | file、Detect It Easy、TrID |
| 十六进制 | 010 Editor、HxD、xxd |
| 字符串 | strings |
| 元数据 | exiftool |
| 文件提取 | binwalk、foremost、dd |
| 图片隐写 | zsteg、Stegsolve、pngcheck、tweakpng |
| JPEG | steghide、outguess、jphide |
| 盲水印 | blind_watermark、invisible-watermark |
| 音频 | Audacity、Sonic Visualiser、sox、ffmpeg |
| 压缩包 | 7z、unzip、bkcrack、fcrackzip、john |
| 流量 | Wireshark、tshark、NetworkMiner |
| 内存 | Volatility 3 |
| 磁盘 | Sleuth Kit、Autopsy、FTK Imager、photorec |
| 文档 | pdf-parser、peepdf、oletools、msoffcrypto-tool |
| 二维码 | zbarimg、pyzbar |
| 编码 | CyberChef |
| 脚本 | Python + PIL + requests |

### 14.2 学习路线

```text
文件识别 + 编码解码
   ↓
图片隐写 + 压缩包
   ↓
音频/视频
   ↓
流量分析
   ↓
内存/磁盘取证
   ↓
综合套娃题
```

### 14.3 推荐靶场

- **CTF Wiki Misc 篇**：分类清晰，适合查漏。
- **BUUCTF Misc**：题量大，适合刷手感。
- **CTFHub**：技能树形式，适合系统练。
- **picoCTF Forensics**：英文题，基础扎实。
- **NSSCTF**：国内比赛题复现。

### 14.4 WP 模板

```markdown
## 题目信息
- 名称：
- 类型：
- 考点：

## 解题过程
1. 文件识别：
2. 发现的线索：
3. 逐层剥离：
4. 拿到 flag：

## 知识点总结
- 
```

---

## 附录 A：速查表

```text
# 文件识别
file mystery
xxd mystery | head
strings -a -n 6 mystery
exiftool mystery
binwalk -e mystery

# 图片
pngcheck -v image.png
zsteg -a image.png
steghide extract -sf image.jpg
convert image.gif -coalesce frames/%03d.png

# 压缩包
zipdetails archive.zip
unzip -z archive.zip
bkcrack -L archive.zip
fcrackzip -u -D -p rockyou.txt archive.zip

# 流量
tshark -r a.pcap -Y "http.request" -T fields -e http.host -e http.request.uri
tshark -r usb.pcap -T fields -e usb.capdata

# 内存
vol -f memory.raw windows.pslist
vol -f memory.raw windows.filescan
vol -f memory.raw windows.dumpfiles --virtaddr 0x...

# 文档
pdfinfo file.pdf
pdftotext file.pdf -
olevba document.doc
unzip document.docx -d docx_out

# 编码
base64 -d
xxd -r -p
python3 -c "import urllib.parse; print(urllib.parse.unquote('...'))"
```

## 附录 B：参考资料

- CTF Wiki Misc：<https://ctf-wiki.org/misc/>
- CTF Wiki 流量分析：<https://ctf-wiki.org/misc/traffic/>
- Volatility 3 文档：<https://volatility3.readthedocs.io/>
- Wireshark 文档：<https://www.wireshark.org/docs/>
- bkcrack：<https://github.com/kimci86/bkcrack>
- CyberChef：<https://gchq.github.io/CyberChef/>
- zsteg：<https://github.com/zed-0xff/zsteg>

---

> 📌 **最后一句：** Misc 没有“万能 payload”，但有万能习惯：**先 `file`，再 `strings`，再 `binwalk`，最后才动手猜**。把这三步养成肌肉记忆，你已经能解决一半的 Misc 题。

---

## 附录 C：新手 14 天练习计划

| 天数 | 任务 | 目标 |
|------|------|------|
| 第 1 天 | 装 CyberChef、010、binwalk | 能识别常见文件头 |
| 第 2 天 | Base64/Base32/Hex/URL | 能手解 10 层编码 |
| 第 3 天 | 摩斯/培根/栅栏 | 能识别并解码 |
| 第 4 天 | PNG 结构 + pngcheck | 能看懂 chunk |
| 第 5 天 | zsteg + Stegsolve | 能做 LSB 题 |
| 第 6 天 | exiftool + 文件头修复 | 能修一张坏图 |
| 第 7 天 | ZIP 伪加密 + 爆破 | 能打开加密压缩包 |
| 第 8 天 | bkcrack 明文攻击 | 能复现一个明文攻击 |
| 第 9 天 | Audacity 频谱 | 能从音频看 flag |
| 第 10 天 | Wireshark 导出对象 | 能从流量拿文件 |
| 第 11 天 | USB HID 解析 | 能还原键盘输入 |
| 第 12 天 | Volatility 基础 | 能列进程、导文件 |
| 第 13 天 | Git 泄漏 | 能恢复被删提交 |
| 第 14 天 | BUUCTF Misc 5 题 | 完整写 WP |

---

## 附录 D：完整演练：一张 PNG 从 0 到 flag

### D.1 拿到文件

```bash
file image.png
# image.png: PNG image data, 800 x 600, 8-bit/color RGBA, non-interlaced
```

### D.2 看十六进制

```bash
xxd image.png | head -5
xxd image.png | tail -5
```

文件头正常：`89 50 4E 47 0D 0A 1A 0A`。

文件尾如果 `IEND` 后面还有数据，就要注意。

### D.3 看元数据

```bash
exiftool image.png
```

假设输出：

```text
Comment: ZmxhZ3tleGlmX2lzX2Vhc3l9
```

这是 Base64：

```bash
echo ZmxhZ3tleGlmX2lzX2Vhc3l9 | base64 -d
# flag{exif_is_easy}
```

### D.4 看 chunk

```bash
pngcheck -v image.png
```

假设看到：

```text
chunk tEXt at offset 0x00021, length 24
```

`tEXt` 里可能藏东西：

```bash
strings -a image.png | grep -i -A2 "tEXt"
```

### D.5 看 LSB

```bash
zsteg -a image.png
```

假设输出：

```text
b1,rgb,lsb,xy      .. text: "flag{lsb_hidden}"
```

### D.6 看尾部

```bash
binwalk image.png
```

假设输出：

```text
DECIMAL       HEXADECIMAL     DESCRIPTION
0             0x0             PNG image
1240          0x4D8           Zip archive data
```

把 ZIP 切出来：

```bash
dd if=image.png of=hidden.zip bs=1 skip=1240
unzip hidden.zip
```

### D.7 如果 ZIP 加密

```bash
zipdetails hidden.zip
unzip -z hidden.zip
```

伪加密就改标志位；真加密就尝试 `bkcrack` 或字典。

### D.8 如果图片高度不对

```bash
pngcheck -v image.png
# CRC error in IHDR
```

用 010 改回高度，重算 CRC，或直接用 `tweakpng`。

### D.9 总结

一张 PNG 可能同时藏：

- EXIF 注释。
- tEXt chunk。
- LSB。
- 尾部 ZIP。
- 高度/CRC 异常。

按“元数据 → chunk → LSB → 尾部 → 修复”的顺序走，基本不会漏。

---

## 附录 E：常见报错与解决

| 现象 | 原因 | 解决 |
|------|------|------|
| `file` 说是 data | 文件头损坏/加密 | `xxd` 看头，补文件头 |
| `unzip` 报错 | 伪加密/损坏/分卷 | 改标志位、`zip -FF`、补分卷 |
| `zsteg` 没结果 | 不是 LSB/通道不对 | 换 `-a`、换通道、Stegsolve |
| `binwalk -e` 没提取 | 签名不认识/嵌套 | 手动 `dd`，看偏移 |
| 图片只显示一半 | 高度被改 | 改 IHDR + CRC |
| 音频没声音 | 声道/格式 | `ffmpeg` 转 wav，分离声道 |
| Wireshark 看不到 HTTP | 不是 HTTP/加密 | 看 TCP 流、TLS keylog |
| USB 解析乱码 | 键码表不对/修饰键 | 换键码表，处理 Shift |
| Volatility 报错 | 镜像版本/配置 | 换 Volatility 3，先 `windows.info` |
| Git 拉不下来 | `.git` 不完整 | 检查 `HEAD`、`config`，换工具 |

---

## 附录 F：术语表

| 术语 | 解释 |
|------|------|
| Stego | 隐写 |
| LSB | 最低有效位 |
| Chunk | PNG 数据块 |
| CRC | 循环冗余校验 |
| Carving | 按文件头提取文件 |
| Magic bytes | 文件头签名 |
| Payload | 隐藏的数据 |
| Cover | 载体文件 |
| Pseudo-encryption | ZIP 伪加密 |
| Known plaintext | 已知明文攻击 |
| OOB | 带外 |
| Timeline | 时间线 |
| Artifact | 痕迹/工件 |
| Hive | 注册表配置单元 |
| MFT | NTFS 主文件表 |
| VSS | 卷影副本 |
| IOC | 入侵指标 |
