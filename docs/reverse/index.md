---
title: "逆向工程入门"
description: "CTF Reverse 方向入门手册：PE/ELF、x86-64 汇编、IDA/Ghidra、动态调试、常见算法、脱壳、跨语言逆向、angr 与 z3。"
---

# 逆向工程入门

> **把黑盒变成白盒：看懂程序在做什么，然后让它说出 flag**

逆向工程是从结果反推过程：没有源码，也要还原程序逻辑。本文从可执行文件格式、汇编基础讲到 IDA/Ghidra、动态调试、脱壳、跨语言逆向与符号执行，并给出完整的 crackme 演练。

## 第零章：逆向是什么

逆向工程（Reverse Engineering）是“从结果反推过程”：给你一个编译好的程序，没有源码，你要通过反汇编、反编译、动态调试，还原它的逻辑。

CTF 逆向题通常长这样：

- 一个 `crackme.exe`，输入正确密码才输出 flag。
- 一个 `checker`，校验你输入的字符串。
- 一个 `.apk`，把 flag 藏在 Java/Kotlin 层。
- 一个 `.pyc`，逻辑被编译成字节码。
- 一个加壳程序，要先脱壳才能看。
- 一个自定义虚拟机，让你逆它的指令集。

### 0.1 需要的基础

| 基础 | 重要程度 | 说明 |
|------|----------|------|
| C 语言 | ★★★★★ | 指针、数组、结构体、函数调用 |
| 汇编 | ★★★★☆ | 能看懂 mov/cmp/jmp/call 就够开始 |
| Linux 命令 | ★★★☆☆ | file、strings、objdump、gdb |
| Python | ★★★☆☆ | 写解密脚本、angr、z3 |
| 数据结构 | ★★☆☆☆ | 栈、队列、树、图 |

> 💡 **不要先学完汇编再做题。** 先做最简单的 crackme，遇到不懂的指令再查，效率高得多。

### 0.2 安全底线

只逆向题目附件、自己写的程序、明确授权的软件。不要逆向商业软件去破解、去绕过授权，那是另一条路。

---

## 第一章：环境与工具

### 1.1 推荐环境

- **Windows 虚拟机**：跑 x64dbg、IDA、查壳工具。
- **Linux 虚拟机（Ubuntu）**：跑 GDB、angr、Python 脚本。
- **快照**：做题前打快照，程序崩了随时回滚。

### 1.2 核心工具

| 工具 | 作用 | 费用 |
|------|------|------|
| IDA Free / IDA Home | 反汇编 + 反编译（F5） | Free 免费，Home 付费 |
| Ghidra | NSA 开源逆向套件，带反编译 | 免费 |
| Binary Ninja | 现代反编译器，插件多 | 付费 |
| radare2 / rizin / Cutter | 命令行/图形化逆向 | 免费 |
| x64dbg / x32dbg | Windows 动态调试 | 免费 |
| GDB + pwndbg / GEF | Linux 动态调试 | 免费 |
| Detect It Easy (DIE) | 查壳、查编译器、查架构 | 免费 |
| PE-bear / CFF Explorer | 查看/编辑 PE 结构 | 免费 |
| 010 Editor / HxD | 十六进制编辑 | 010 付费，HxD 免费 |
| upx | 脱 UPX 壳 | 免费 |
| ScyllaHide | 反反调试插件 | 免费 |
| Process Monitor | 看文件/注册表/网络行为 | 免费 |
| API Monitor | 看 API 调用 | 免费 |

### 1.3 语言相关工具

| 目标 | 工具 |
|------|------|
| .NET | dnSpy、ILSpy、dotPeek |
| Java | jadx、JD-GUI、CFR、Procyon |
| Android APK | jadx、apktool、Frida、objection |
| Python | pycdc、decompyle3、uncompyle6、pyinstxtractor |
| Go | GoReSym、`go tool objdump`、IDA 插件 |
| Rust | `rustfilt`、IDA/Ghidra 符号恢复 |

> 💡 工具装最新稳定版即可。IDA 9.x、Ghidra 11.x、x64dbg 2026 稳定版都能覆盖绝大多数 CTF 题。

---

## 第二章：可执行文件格式

### 2.1 三种常见格式

| 系统 | 格式 | 特征 |
|------|------|------|
| Windows | PE（.exe/.dll） | 开头 `MZ`，PE 头 |
| Linux | ELF | 开头 `\x7fELF` |
| macOS | Mach-O | 开头 `\xfeedfacf` 等 |

### 2.2 PE 结构

```text
DOS Header (MZ)
   ↓
DOS Stub
   ↓
NT Headers (PE\0\0)
   ├─ File Header（架构、节数量）
   └─ Optional Header（入口点、镜像基址、节对齐）
   ↓
Section Table
   ├─ .text    代码
   ├─ .data    已初始化全局变量
   ├─ .rdata   只读数据（字符串、导入表）
   ├─ .bss     未初始化全局变量
   └─ .rsrc    资源
```

关键概念：

- **ImageBase**：程序默认加载地址。
- **Entry Point (OEP)**：程序入口。
- **IAT（导入地址表）**：程序调用外部函数时用的地址表。
- **EAT（导出地址表）**：DLL 导出函数表。

### 2.3 ELF 结构

```text
ELF Header
   ↓
Program Headers（运行时怎么加载）
   ↓
Sections
   ├─ .text    代码
   ├─ .rodata  只读数据
   ├─ .data    已初始化数据
   ├─ .bss     未初始化数据
   ├─ .plt     外部函数跳转桩
   ├─ .got     全局偏移表（运行时填真实地址）
   └─ .init/.fini
```

### 2.4 用命令快速看

```bash
file binary
readelf -h binary        # ELF 头
readelf -S binary        # 节表
readelf -s binary        # 符号表
objdump -d binary        # 反汇编
nm binary                # 符号
ldd binary               # 依赖库
```

Windows 可以用 `dumpbin /headers`、`dumpbin /imports`（Visual Studio 自带）。

---

## 第三章：x86-64 汇编速成

### 3.1 寄存器

| 寄存器 | 作用 |
|--------|------|
| `rax` | 累加器，函数返回值 |
| `rbx` | 基址寄存器，常保存指针 |
| `rcx` | 计数，第 4 个参数（SysV） |
| `rdx` | 数据，第 3 个参数 |
| `rsi` | 源索引，第 2 个参数 |
| `rdi` | 目的索引，第 1 个参数 |
| `rbp` | 栈帧基址 |
| `rsp` | 栈顶指针 |
| `r8`-`r15` | 通用寄存器，第 5、6 个参数等 |
| `rip` | 指令指针（下一条指令地址） |
| `eflags` | 标志位（ZF、SF、CF、OF） |

子寄存器：

```text
rax = 64 位
eax = 低 32 位
ax  = 低 16 位
al  = 低 8 位
ah  = 高 8 位（仅 ax）
```

### 3.2 常见指令

| 指令 | 含义 |
|------|------|
| `mov dst, src` | 赋值 |
| `lea dst, [addr]` | 取地址（不访问内存） |
| `push x` | 压栈 |
| `pop x` | 出栈 |
| `add/sub` | 加/减 |
| `imul/idiv` | 乘/除 |
| `and/or/xor/not` | 位运算 |
| `shl/shr` | 左/右移 |
| `cmp a, b` | 比较（做减法但不保存结果） |
| `test a, b` | 按位与，只影响标志位 |
| `jmp` | 无条件跳转 |
| `je/jz` | 相等/为零跳转 |
| `jne/jnz` | 不等/非零跳转 |
| `jg/jl` | 有符号大于/小于 |
| `ja/jb` | 无符号大于/小于 |
| `call` | 调用函数（压返回地址） |
| `ret` | 返回（弹出返回地址） |
| `leave` | `mov rsp, rbp; pop rbp` |
| `nop` | 空操作 |

### 3.3 函数调用与栈帧

x86-64 SysV（Linux）参数顺序：

```text
rdi, rsi, rdx, rcx, r8, r9，之后走栈
返回值：rax
```

Windows x64：

```text
rcx, rdx, r8, r9，之后走栈
调用者还要预留 32 字节 shadow space
返回值：rax
```

典型函数开头：

```asm
push rbp
mov rbp, rsp
sub rsp, 0x20
...
leave
ret
```

对应栈帧：

```text
高地址
┌──────────────┐
│ 调用者栈帧     │
├──────────────┤
│ 返回地址       │ ← rsp 在 call 之后
├──────────────┤
│ 保存的 rbp     │ ← rbp
├──────────────┤
│ 局部变量       │
└──────────────┘
低地址
```

### 3.4 看汇编的小技巧

- `cmp a, b` 后面跟 `jne`，通常是在判断“两个值是否相等”。
- `test eax, eax` 后面跟 `jz`，通常是在判断“返回值是否为 0”。
- 字符串比较常调用 `strcmp`、`memcmp`，参数是 `rdi`、`rsi`。
- 循环通常长这样：

```asm
mov ecx, 0        ; i = 0
loop_start:
cmp ecx, 10
jge loop_end
...
inc ecx
jmp loop_start
loop_end:
```

---

## 第四章：静态分析流程

### 4.1 五步走

```text
1. file / DIE 看架构、编译器、是否加壳
2. strings 找明显字符串
3. 看导入表，猜程序用了什么库
4. 找 main / 入口，读反编译伪代码
5. 定位关键比较/分支，还原算法
```

### 4.2 看字符串

```bash
strings -a -n 6 crackme
strings -el crackme        # UTF-16
```

常见线索：

```text
"Correct!"
"Wrong!"
"Input:"
"flag{"
"Usage:"
```

在 IDA/Ghidra 里对这些字符串按 `X` 看交叉引用，能直接跳到校验逻辑。

### 4.3 找 main

- 有符号：直接搜 `main`。
- 无符号：从入口点找 `__libc_start_main` 的第一个参数。
- Windows：从 `EntryPoint` 进，找 `GetCommandLineA`、`__scrt_common_main`，或直接找 `WinMain`。
- 也可以从字符串交叉引用反推。

### 4.4 IDA 常用快捷键

| 快捷键 | 作用 |
|--------|------|
| `F5` | 反编译为伪代码 |
| `N` | 重命名 |
| `X` | 查看交叉引用 |
| `G` | 跳转到地址 |
| `;` | 添加注释 |
| `Space` | 图形视图/文本视图切换 |
| `Esc` | 返回上一视图 |
| `Ctrl+F` | 搜索 |
| `Shift+F12` | 字符串窗口 |
| `F2` | 下断点 |

### 4.5 Ghidra 常用操作

- 双击函数进入，右侧 Decompile 窗口看伪代码。
- `Ctrl+Shift+E` 导出反编译结果（不同版本可能不同）。
- `L` 重命名，`;` 注释。
- 字符串窗口：Window → Defined Strings。
- 交叉引用：右键 → References → Show References to。

### 4.6 读伪代码的技巧

反编译出来的变量名通常是 `v1`、`v2`，不要怕，按这个顺序读：

1. 找输入：`scanf`、`cin`、`fgets`、`read`。
2. 找长度限制：`strlen`、循环次数。
3. 找关键比较：`strcmp`、`memcmp`、`if`。
4. 找变换：异或、加减、查表、base64。
5. 把伪代码翻译成 Python，反向计算。

---

## 第五章：动态调试

### 5.1 什么时候用动态调试

- 静态看不出来（混淆、虚拟机、间接跳转）。
- 想知道某个函数运行时参数。
- 想跳过反调试。
- 想直接改内存/寄存器拿结果。

### 5.2 x64dbg 基础

| 操作 | 快捷键 |
|------|--------|
| 运行 | F9 |
| 暂停 | F12 |
| 单步进入 | F7 |
| 单步跳过 | F8 |
| 到返回 | Ctrl+F9 |
| 下断点 | F2 |
| 跳转地址 | Ctrl+G |
| 跟随表达式 | Ctrl+Shift+G |

常用窗口：

- **CPU**：反汇编、寄存器、栈。
- **Dump**：内存数据。
- **Stack**：调用栈。
- **Breakpoints**：断点列表。
- **Memory Map**：模块和内存段。

### 5.3 下断点的位置

- 字符串比较函数：`strcmp`、`lstrcmpA`、`memcmp`。
- 输入函数：`scanf`、`fgets`、`GetDlgItemTextA`。
- 提示字符串附近。
- 关键跳转前。

### 5.4 改寄存器/内存

在 x64dbg 里双击寄存器值可以直接改；在 Dump 窗口右键 → Modify → 改字节。

常见操作：

- 把 `ZF` 置 1，让 `jne` 不跳。
- 把 `eax` 改成 0，让校验通过。
- 把内存里的比较结果改掉。

### 5.5 GDB 基础

```bash
gdb ./binary
(gdb) b main
(gdb) run
(gdb) ni          # 单步跳过
(gdb) si          # 单步进入
(gdb) x/20i $rip  # 看指令
(gdb) x/s 0x...   # 看字符串
(gdb) info registers
(gdb) set $eax=0
(gdb) continue
```

装了 pwndbg 或 GEF 后，界面会显示寄存器、栈、反汇编，体验更好。

---

## 第六章：常见题型与套路

### 6.1 明文字符串比较

```c
if (strcmp(input, "s3cr3t") == 0) puts("Correct");
```

直接在字符串窗口找 `s3cr3t` 即可。

### 6.2 异或加密

```c
for (int i = 0; i < len; i++) {
    buf[i] = input[i] ^ key[i % key_len];
}
```

逆向时把密文再异或一次同样的 key：

```python
cipher = bytes.fromhex("...")
key = b"secret"
plain = bytes(c ^ key[i % len(key)] for i, c in enumerate(cipher))
print(plain)
```

如果 key 未知，可以用已知明文（`flag{`）反推 key。

### 6.3 简单加减/移位

```c
buf[i] = (input[i] - 3) ^ 0x55;
```

逆向就是反过来：

```python
plain = bytes(((c ^ 0x55) + 3) & 0xFF for c in cipher)
```

> 💡 注意运算顺序和溢出。C 里的 `char` 可能是有符号的，Python 要 `& 0xFF`。

### 6.4 Base64 / RC4 / TEA

- Base64：字符串里出现 `ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/`。
- RC4：有 256 字节的 S 盒初始化。
- TEA/XTEA：特征常数 `0x9E3779B9`、`0x61C88647`。
- AES：S 盒、`0x1B` 乘法、轮密钥。

识别出算法后，直接找现成实现解密。

### 6.5 迷宫题

程序把地图存成二维数组，用 `w/a/s/d` 控制移动，要求走到终点。

做法：

1. 在 IDA 里找到地图数组。
2. 用 Python 提取成二维字符画。
3. 手走或用 BFS 求路径。
4. 路径就是 flag。

### 6.6 数独/方程题

程序要求填入满足条件的数字。

- 提取约束。
- 用 z3 求解。
- 或直接写回溯。

### 6.7 自定义虚拟机

程序里有一个 `switch`，每个 case 对应一条自定义指令（push、add、jmp、print 等）。

做法：

1. 找到字节码数组。
2. 还原每条指令的语义。
3. 用 Python 写一个解释器跑字节码。
4. 或直接把字节码翻译成可读逻辑。

### 6.8 按键序列

程序记录你按下的键，要求特定顺序。

- 动态调试时下断点看比较。
- 或直接在内存里找目标序列。

### 6.9 校验和/CRC

程序对你的输入做 CRC32/MD5 校验。

- 如果只校验长度和 CRC，可以构造碰撞或逆向 CRC。
- 如果还比较内容，就老老实实还原算法。

---

## 第七章：脱壳与反调试

### 7.1 查壳

```bash
diec crackme.exe
```

或看节区名：

```text
UPX0 / UPX1        → UPX
.vmp               → VMProtect
Themida            → Themida
```

### 7.2 UPX 脱壳

```bash
upx -d packed.exe -o unpacked.exe
```

如果 UPX 头被改过，`upx -d` 会报错。可以手工找 OEP，或用 `upx` 修复工具。

### 7.3 手动脱壳思路

1. 用 x64dbg 打开，停在入口。
2. 找“POPAD”后的跳转，或“push 地址; ret”。
3. 在目标地址下断点。
4. 运行到 OEP。
5. 用 Scylla 或 x64dbg 插件 dump 内存、修复 IAT。

> 💡 新手先练 UPX，手脱壳等遇到再说。VMProtect/Themida 这类商业壳，CTF 里通常不会让你硬脱，而是找其他突破口。

### 7.4 常见反调试

| 手段 | 表现 |
|------|------|
| `IsDebuggerPresent` | 检测到调试器就退出 |
| `CheckRemoteDebuggerPresent` | 同上 |
| `NtQueryInformationProcess` | 查询调试端口/标志 |
| PEB.BeingDebugged | 直接读 PEB |
| `ptrace(PTRACE_TRACEME)` | Linux 下拒绝二次调试 |
| 时间检测 | 单步太慢就判定调试 |
| 异常处理 | 故意触发异常，调试器不处理就崩 |
| 父进程检测 | 检查父进程是不是调试器 |

### 7.5 反反调试

- **ScyllaHide**：x64dbg 插件，一键隐藏调试器。
- 手工改 PEB、改返回值。
- 用 `hide` 插件、 TitanHide。
- Linux 下 patch `ptrace` 调用，或用 `LD_PRELOAD` 拦截。

### 7.6 加壳程序的突破口

- 找字符串（壳通常不加密所有字符串）。
- 找导入表（壳初始化后会还原）。
- 在 `VirtualAlloc`、`VirtualProtect` 下断点。
- 运行到 OEP 后 dump。

---

## 第八章：跨语言逆向

### 8.1 .NET

工具：dnSpy、ILSpy、dotPeek。

dnSpy 可以直接看 C# 源码、下断点、改代码、保存回 exe。

常见套路：

- 直接搜 `flag`、`Correct`。
- 看 `Main`。
- 如果是混淆过的，用 de4dot 先脱混淆。

### 8.2 Java

```bash
# jar 包
jadx-gui app.jar
# 或
java -jar cfr.jar app.jar --outputdir out
```

看 `Main`、`check`、`decrypt` 方法。

### 8.3 Android APK

```bash
# 反编译 Java/Kotlin
jadx-gui app.apk

# 反编译资源 + smali
apktool d app.apk -o apk_out

# 动态 Hook
frida -U -f com.example.app -l hook.js
```

常见藏点：

- `MainActivity` 里的校验。
- `native-lib.so` 里的 JNI 函数。
- `assets/` 里的加密文件。
- `strings.xml` 里的提示。
- 签名校验/root 检测。

修改 smali 后重新打包签名：

```bash
apktool b apk_out -o patched.apk
uber-apk-signer -a patched.apk
```

### 8.4 Python

**`.pyc` 文件：**

```bash
pycdc challenger.pyc
# 或
decompyle3 challenger.pyc
uncompyle6 challenger.pyc
```

**PyInstaller 打包的 exe：**

```bash
python pyinstxtractor.py app.exe
# 得到 app.exe_extracted/
# 找到主程序的 .pyc，再反编译
```

> ⚠️ Python 版本越高，反编译越容易失败。`pycdc` 支持范围较广，但遇到新版本字节码仍可能出错。可以先用 `python -m dis` 看字节码。

### 8.5 Go

- `GoReSym` 恢复函数名。
- `go tool objdump -s main.main binary`。
- IDA 装 golang loader 插件。
- Go 二进制通常很大，先搜字符串和 `main.main`。

### 8.6 Rust

- 符号名被 mangle，用 `rustfilt` 还原。
- Ghidra/IDA 对 Rust 的支持在变好。
- 先找 `main`、`rust_begin_unwind`、字符串。

---

## 第九章：符号执行与约束求解

### 9.1 z3 解方程

题目要求满足一堆约束，直接交给 z3：

```python
from z3 import *

flag = [BitVec(f"c{i}", 8) for i in range(8)]
s = Solver()

s.add(flag[0] + flag[1] == 0x9A)
s.add(flag[2] ^ flag[3] == 0x2F)
s.add(flag[4] * 3 == 0x123)
# ... 按题目条件继续加

if s.check() == sat:
    m = s.model()
    print(bytes([m[c].as_long() for c in flag]))
```

### 9.2 angr 符号执行

angr 可以自动探索程序路径，找到满足条件的输入。

```python
import angr
import claripy

proj = angr.Project("./chall", auto_load_libs=False)

# 100 字节的符号输入
flag = claripy.BVS("flag", 100 * 8)
state = proj.factory.entry_state(
    stdin=angr.SimFileStream(name="stdin", content=flag, has_end=False)
)
simgr = proj.factory.simgr(state)

# 目标是输出 Correct 的地址
simgr.explore(find=0x401234, avoid=0x401256)

if simgr.found:
    found = simgr.found[0]
    print(found.posix.dumps(0))
```

常用参数：

- `find=`：成功地址。
- `avoid=`：失败地址。
- `state.posix.dumps(0)`：读 stdin。
- `state.posix.dumps(1)`：读 stdout。

> 💡 angr 对简单校验题很好用，但路径爆炸时会很慢。先用静态分析缩小范围，再用 angr。

### 9.3 什么时候用 angr

- 程序逻辑是线性校验。
- 输入长度固定。
- 没有复杂循环/系统调用。
- 静态分析太绕，但能找到成功/失败分支。

不适合：大量循环、浮点、复杂数据结构、反调试。

---

## 第十章：补丁与重打包

### 10.1 IDA 打补丁

1. 光标移到要改的指令。
2. Edit → Patch program → Assemble（或 Change byte）。
3. 输入新指令。
4. Apply patches to input file。

例：把 `jne` 改成 `je`，或把 `call check` 改成 `nop`。

### 10.2 x64dbg 打补丁

1. 在 CPU 窗口右键 → Assemble。
2. 输入新指令。
3. 右键 → Patches → Save file。

### 10.3 APK 重打包

```bash
apktool d app.apk -o out
# 修改 smali
apktool b out -o patched.apk
uber-apk-signer -a patched.apk
adb install patched.apk
```

### 10.4 验证补丁

- 重新运行程序，确认逻辑改变。
- 检查文件大小、校验和。
- 如果程序有完整性校验，先绕过校验。

---

## 第十一章：工具链与学习路线

### 11.1 工具速查

| 场景 | 工具 |
|------|------|
| 查壳/查编译器 | Detect It Easy、PEiD |
| 静态反汇编 | IDA、Ghidra、radare2 |
| 反编译 | IDA F5、Ghidra、Binary Ninja |
| 动态调试（Windows） | x64dbg、WinDbg |
| 动态调试（Linux） | GDB + pwndbg/GEF |
| .NET | dnSpy、ILSpy |
| Java | jadx、CFR |
| Android | jadx、apktool、Frida |
| Python | pycdc、pyinstxtractor |
| 符号执行 | angr |
| 约束求解 | z3 |
| 脱壳 | upx、Scylla |
| 反反调试 | ScyllaHide |

### 11.2 学习路线

```text
C + 汇编基础
   ↓
用 IDA 看简单 crackme
   ↓
x64dbg 动态调试
   ↓
异或/Base64/RC4/TEA 算法还原
   ↓
迷宫/数独/VM 题
   ↓
脱壳 + 反调试
   ↓
.NET / APK / Python
   ↓
angr + z3
```

### 11.3 推荐练习

- **crackmes.one**：大量 crackme，难度分级。
- **picoCTF Reverse**：基础题友好。
- **Reversing.kr**：经典逆向题。
- **Flare-On**：每年一轮，难度较高。
- **CTF Wiki Reverse**：中文系统资料。
- **BUUCTF Reverse**：国内题多。

### 11.4 WP 模板

```markdown
## 题目信息
- 名称：
- 类型：
- 架构/语言：
- 保护：

## 分析过程
1. 查壳/查信息：
2. 静态分析：
3. 动态调试：
4. 算法还原：
5. 求解脚本：

## 知识点总结
- 
```

---

## 附录 A：汇编速查

```asm
; 数据传送
mov rax, rbx
lea rax, [rbx + 8]
push rax
pop rbx

; 运算
add rax, 1
sub rax, 1
imul rax, rbx
xor rax, rax        ; 清零
and rax, 0xFF
shl rax, 4
shr rax, 4

; 比较与跳转
cmp rax, rbx
test rax, rax
je label
jne label
jg label
jl label
jmp label

; 函数
call func
ret
leave
```

## 附录 B：参考资料

- CTF Wiki Reverse：<https://ctf-wiki.org/reverse/>
- Ghidra：<https://ghidra-sre.org/>
- IDA：<https://hex-rays.com/ida-free/>
- x64dbg：<https://x64dbg.com/>
- angr 文档：<https://docs.angr.io/>
- z3 文档：<https://z3prover.github.io/api/html/>
- Frida：<https://frida.re/docs/home/>
- crackmes.one：<https://crackmes.one/>

---

> 📌 **最后一句：** 逆向不是“从头读到尾”，而是“先找 flag 相关的地方，再往回推”。字符串、导入表、关键跳转，就是你的三盏路灯。

---

## 附录 C：新手 14 天练习计划

| 天数 | 任务 | 目标 |
|------|------|------|
| 第 1 天 | 装 IDA Free/Ghidra/x64dbg | 能打开一个 exe |
| 第 2 天 | 寄存器 + 常见指令 | 能看懂 20 行汇编 |
| 第 3 天 | 函数调用 + 栈帧 | 能画出栈帧图 |
| 第 4 天 | strings + 导入表 | 能找到关键字符串 |
| 第 5 天 | IDA F5 + 交叉引用 | 能定位校验函数 |
| 第 6 天 | x64dbg 断点/单步 | 能改寄存器跳过校验 |
| 第 7 天 | 异或/加减还原 | 能写 Python 解密 |
| 第 8 天 | Base64/RC4/TEA | 能识别算法特征 |
| 第 9 天 | 迷宫/数独 | 能提取数据并求解 |
| 第 10 天 | UPX 脱壳 | 能脱一个 UPX |
| 第 11 天 | .NET/APK | 能用 dnSpy/jadx 看逻辑 |
| 第 12 天 | Python pyc | 能反编译并还原 |
| 第 13 天 | angr/z3 | 能解一个简单约束 |
| 第 14 天 | crackmes.one 5 题 | 完整写 WP |

---

## 附录 D：完整演练：一个 crackme 从 0 到 key

### D.1 题目代码

假设题目给了这个程序（没有源码）：

```c
#include <stdio.h>
#include <string.h>

int main() {
    char buf[64];
    printf("password: ");
    scanf("%63s", buf);
    if (strcmp(buf, "s3cr3t_k3y") == 0)
        puts("Correct!");
    else
        puts("Wrong!");
    return 0;
}
```

### D.2 第一步：看文件

```bash
file crackme
# ELF 64-bit LSB pie executable, x86-64
checksec --file=crackme
# NX enabled, PIE enabled, Canary found
```

### D.3 第二步：看字符串

```bash
strings -a crackme | grep -E "password|Correct|Wrong|s3cr3t"
```

输出：

```text
password: 
Correct!
Wrong!
```

如果没有直接看到密码，说明密码是运行时生成的，或者被编码了。

### D.4 第三步：IDA 打开

1. 导入 `crackme`。
2. 字符串窗口（Shift+F12）找 `Correct!`。
3. 双击，按 `X` 看引用。
4. 进入 `main`，按 `F5`。

伪代码大概：

```c
int __cdecl main(int argc, const char **argv, const char **envp) {
  char buf[64];
  printf("password: ");
  scanf("%63s", buf);
  if (!strcmp(buf, "s3cr3t_k3y"))
    puts("Correct!");
  else
    puts("Wrong!");
  return 0;
}
```

密码直接出现在伪代码里。

### D.5 第四步：动态验证

x64dbg 打开：

1. `Ctrl+G` 跳到 `main`。
2. 在 `strcmp` 调用后下断点。
3. 输入一个错误密码。
4. 断下后看 `rdi`、`rsi`：

```text
rdi = 你输入的字符串
rsi = s3cr3t_k3y
```

确认比较的是这两个值。

### D.6 第五步：如果密码被加密

假设伪代码是：

```c
for (i = 0; i < len; i++)
    buf[i] ^= 0x5A;
if (!strcmp(buf, cipher))
    ...
```

提取 `cipher` 字节，写脚本：

```python
cipher = bytes.fromhex("292e39392e3a3e")
print(bytes(c ^ 0x5A for c in cipher))
# b's3cr3t_k3y'
```

### D.7 第六步：写 keygen

```python
def keygen():
    return "s3cr3t_k3y"

print(keygen())
```

### D.8 第七步：验证

```bash
echo "s3cr3t_k3y" | ./crackme
# Correct!
```

### D.9 你学到了什么

- `strings` 找线索。
- IDA 交叉引用定位逻辑。
- 动态调试确认参数。
- 异或还原。
- keygen 思路。

---

## 附录 E：常见报错与解决

| 现象 | 原因 | 解决 |
|------|------|------|
| IDA F5 没反应 | 没有反编译器/架构不支持 | 用 IDA Home/Ghidra |
| `unknown processor type` | 文件头损坏/加壳 | 查壳，脱壳后再打开 |
| x64dbg 打不开 | 32/64 位不匹配/加壳 | 换 x32dbg，先脱壳 |
| 断点不触发 | PIE/ASLR/没走到 | 关 ASLR，或在入口下断 |
| Ghidra 分析很慢 | 文件太大 | 只导入代码段，关自动分析 |
| Ghidra 地址不对 | 基址设置错 | 改 Image Base |
| `upx -d` 报错 | 不是 UPX/头被改 | 查壳，手脱 |
| `strings` 乱码 | UTF-16/编码 | 用 `strings -el`，或转码 |
| pycdc 失败 | Python 版本太新 | 用同版本 Python 的 `dis` |
| APK 重打包失败 | 签名/资源问题 | 用 uber-apk-signer，检查 smali |
| angr 跑不动 | 路径爆炸 | 缩小 find/avoid，限制输入 |
| z3 返回 unknown | 约束太复杂/非线性 | 简化约束，换求解器 |

---

## 附录 F：术语表

| 术语 | 解释 |
|------|------|
| PE/ELF | Windows/Linux 可执行文件格式 |
| OEP | 程序原始入口点 |
| IAT/GOT | 导入地址表/全局偏移表 |
| RVA | 相对虚拟地址 |
| ImageBase | 镜像基址 |
| PIE/ASLR | 地址随机化 |
| Packer | 加壳程序 |
| Stub | 壳的解压代码 |
| Anti-debug | 反调试 |
| Patch | 修改指令/文件 |
| Keygen | 注册机 |
| Crackme | 逆向练习程序 |
| F5 | IDA 反编译快捷键 |
| Xref | 交叉引用 |
| Mangle | C++ 符号修饰 |
| Demangle | 还原符号名 |
| Bytecode | 字节码 |
| AOT | 提前编译 |
| JNI | Java 调用 Native |
| Smali | Android 字节码反汇编 |
