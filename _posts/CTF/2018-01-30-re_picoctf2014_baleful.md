---
layout: post
title: re PicoCTF2014 Baleful
category: CTF
tags: ctf
keywords: ctf, binary, hack
description:
---

- [题目解析](#题目解析)
  - [逆向 VM 求解](#逆向-vm-求解)
  - [使用 Pin 求解](#使用-pin-求解)
- [参考资料](#参考资料)


## 题目解析
```
$ file baleful 
baleful: ELF 32-bit LSB executable, Intel 80386, version 1 (GNU/Linux), statically linked, stripped
$ strings baleful | grep -i upx
@UPX!
$Info: This file is packed with the UPX executable packer http://upx.sf.net $
$Id: UPX 3.91 Copyright (C) 1996-2013 the UPX Team. All Rights Reserved. $
UPX!u
UPX!
UPX!
$ upx -d baleful -o baleful_unpack
                       Ultimate Packer for eXecutables
                          Copyright (C) 1996 - 2017
UPX 3.94        Markus Oberhumer, Laszlo Molnar & John Reiser   May 12th 2017

        File size         Ratio      Format      Name
   --------------------   ------   -----------   -----------
    144956 <-      6752    4.66%   linux/i386    baleful_unpack

Unpacked 1 file.
$ file baleful_unpack
baleful_unpack: ELF 32-bit LSB executable, Intel 80386, version 1 (SYSV), dynamically linked, interpreter /lib/ld-linux.so.2, for GNU/Linux 2.6.24, BuildID[sha1]=35d1a373cbe6a675ecbbc904722a86f853f20ce3, stripped
```
经过简单地检查，我们发现二进制文件被加了壳，使用 upx 脱掉就好了。

运行下看看，典型的密码验证题：
```
$ ./baleful_unpack
Please enter your password: ABCD
Sorry, wrong password!
```

#### 逆向 VM 求解
打开 r2 开干吧！
```
[0x08048540]> pdf @ main
/ (fcn) main 96
|   main ();
|           ; var int local_8h @ ebp-0x8
|           ; var int local_10h @ esp+0x10
|           ; var int local_8ch @ esp+0x8c
|              ; DATA XREF from 0x08048557 (entry0)
|           0x08049c82      55             push ebp
|           0x08049c83      89e5           mov ebp, esp
|           0x08049c85      57             push edi
|           0x08049c86      53             push ebx
|           0x08049c87      83e4f0         and esp, 0xfffffff0
|           0x08049c8a      81ec90000000   sub esp, 0x90
|           0x08049c90      65a114000000   mov eax, dword gs:[0x14]    ; [0x14:4]=-1 ; 20
|           0x08049c96      8984248c0000.  mov dword [local_8ch], eax
|           0x08049c9d      31c0           xor eax, eax
|           0x08049c9f      8d442410       lea eax, [local_10h]        ; 0x10 ; 16
|           0x08049ca3      89c3           mov ebx, eax
|           0x08049ca5      b800000000     mov eax, 0
|           0x08049caa      ba1f000000     mov edx, 0x1f               ; 31
|           0x08049caf      89df           mov edi, ebx
|           0x08049cb1      89d1           mov ecx, edx
|           0x08049cb3      f3ab           rep stosd dword es:[edi], eax
|           0x08049cb5      8d442410       lea eax, [local_10h]        ; 0x10 ; 16
|           0x08049cb9      890424         mov dword [esp], eax
|           0x08049cbc      e8caecffff     call fcn.0804898b
|           0x08049cc1      b800000000     mov eax, 0
|           0x08049cc6      8b94248c0000.  mov edx, dword [local_8ch]  ; [0x8c:4]=-1 ; 140
|           0x08049ccd      653315140000.  xor edx, dword gs:[0x14]
|       ,=< 0x08049cd4      7405           je 0x8049cdb
|       |   0x08049cd6      e8e5e7ffff     call sym.imp.__stack_chk_fail ; void __stack_chk_fail(void)
|       |      ; JMP XREF from 0x08049cd4 (main)
|       `-> 0x08049cdb      8d65f8         lea esp, [local_8h]
|           0x08049cde      5b             pop ebx
|           0x08049cdf      5f             pop edi
|           0x08049ce0      5d             pop ebp
\           0x08049ce1      c3             ret
```

`fcn.0804898b` 是程序主要的逻辑所在，很容易看出来它其实是实现了一个虚拟机：
```
[0x08048540]> pdf @ fcn.0804898b
Linear size differs too much from the bbsum, please use pdr instead.
[0x08048540]> pdr @ fcn.0804898b
/ (fcn) fcn.0804898b 226
|   fcn.0804898b (int arg_8h);
| ; var int local_b4h @ ebp-0xb4
| ; var int local_38h @ ebp-0x38
| ; var int local_34h @ ebp-0x34
| ; var int local_30h @ ebp-0x30
| ; var int local_2ch @ ebp-0x2c
| ; var int local_28h @ ebp-0x28
| ; var int local_24h @ ebp-0x24
| ; var int local_20h @ ebp-0x20
| ; var int local_1ch @ ebp-0x1c
| ; var int local_18h @ ebp-0x18
| ; var int local_14h @ ebp-0x14
| ; var int local_10h @ ebp-0x10
| ; var int local_ch @ ebp-0xc
| ; arg int arg_8h @ ebp+0x8
|    ; CALL XREF from 0x08049cbc (main)
| 0x0804898b      55             push ebp
| 0x0804898c      89e5           mov ebp, esp
| 0x0804898e      81ecc8000000   sub esp, 0xc8
| 0x08048994      c745cc001000.  mov dword [local_34h], 0x1000
| 0x0804899b      837d0800       cmp dword [arg_8h], 0
| 0x0804899f      742a           je 0x80489cb
| ----------- true: 0x080489cb  false: 0x080489a1
| 0x080489a1      c745d0000000.  mov dword [local_30h], 0
| 0x080489a8      eb19           jmp 0x80489c3
| ----------- true: 0x080489c3
|    ; JMP XREF from 0x080489c7 (fcn.0804898b)
| 0x080489aa      8b45d0         mov eax, dword [local_30h]
| 0x080489ad      c1e002         shl eax, 2
| 0x080489b0      034508         add eax, dword [arg_8h]
| 0x080489b3      8b10           mov edx, dword [eax]
| 0x080489b5      8b45d0         mov eax, dword [local_30h]
| 0x080489b8      8994854cffff.  mov dword [ebp + eax*4 - 0xb4], edx
| 0x080489bf      8345d001       add dword [local_30h], 1
| ----------- true: 0x080489c3
|    ; JMP XREF from 0x080489a8 (fcn.0804898b)
| 0x080489c3      837dd01e       cmp dword [local_30h], 0x1e           ; [0x1e:4]=-1 ; 30
| 0x080489c7      7ee1           jle 0x80489aa
| ----------- true: 0x080489aa  false: 0x080489c9
| 0x080489c9      eb21           jmp 0x80489ec
| ----------- true: 0x080489ec
|    ; JMP XREF from 0x0804899f (fcn.0804898b)
| 0x080489cb      c745d4000000.  mov dword [local_2ch], 0
| 0x080489d2      eb12           jmp 0x80489e6
| ----------- true: 0x080489e6
|    ; JMP XREF from 0x080489ea (fcn.0804898b)
| 0x080489d4      8b45d4         mov eax, dword [local_2ch]
| 0x080489d7      c784854cffff.  mov dword [ebp + eax*4 - 0xb4], 0
| 0x080489e2      8345d401       add dword [local_2ch], 1
| ----------- true: 0x080489e6
|    ; JMP XREF from 0x080489d2 (fcn.0804898b)
| 0x080489e6      837dd41e       cmp dword [local_2ch], 0x1e           ; [0x1e:4]=-1 ; 30
| 0x080489ea      7ee8           jle 0x80489d4
| ----------- true: 0x080489d4  false: 0x080489ec
|    ; JMP XREF from 0x080489c9 (fcn.0804898b)
| 0x080489ec      c745c800f000.  mov dword [local_38h], 0xf000
| 0x080489f3      c745d8000000.  mov dword [local_28h], 0
| 0x080489fa      c745ec000000.  mov dword [local_14h], 0
| 0x08048a01      c745f0000000.  mov dword [local_10h], 0
| 0x08048a08      c745f4000000.  mov dword [local_ch], 0
| 0x08048a0f      c745e8000000.  mov dword [local_18h], 0
| 0x08048a16      8b45e8         mov eax, dword [local_18h]
| 0x08048a19      8945e4         mov dword [local_1ch], eax
| 0x08048a1c      8b45e4         mov eax, dword [local_1ch]
| 0x08048a1f      8945e0         mov dword [local_20h], eax
| 0x08048a22      8b45e0         mov eax, dword [local_20h]
| 0x08048a25      8945dc         mov dword [local_24h], eax
| 0x08048a28      e93a120000     jmp 0x8049c67
| ----------- true: 0x08049c67
|    ; JMP XREF from 0x08049c74 (fcn.0804898b)
| 0x08048a2d      8b45cc         mov eax, dword [local_34h]
| 0x08048a30      05c0c00408     add eax, 0x804c0c0
| 0x08048a35      0fb600         movzx eax, byte [eax]
| 0x08048a38      0fbec0         movsx eax, al
| 0x08048a3b      83f820         cmp eax, 0x20                         ; 32
| 0x08048a3e      0f871e120000   ja 0x8049c62
| ----------- true: 0x08049c62  false: 0x08048a44
| 0x08048a44      8b0485d49d04.  mov eax, dword [eax*4 + 0x8049dd4]    ; [0x8049dd4:4]=0x8048a4d
| 0x08048a4b      ffe0           jmp eax

|    ; JMP XREF from 0x08048a3e (fcn.0804898b)
| 0x08049c62      8345cc01       add dword [local_34h], 1
| 0x08049c66      90             nop
| ----------- true: 0x08049c67
|    ; XREFS: JMP 0x08048a28  JMP 0x08048a51  JMP 0x08048a8a  JMP 0x08048bbf  JMP 0x08048cf4  JMP 0x08048e2a  JMP 0x08048f8c  JMP 0x080490c1  
|    ; XREFS: JMP 0x080491f6  JMP 0x0804932b  JMP 0x08049462  JMP 0x08049599  JMP 0x080495f0  JMP 0x08049644  JMP 0x08049698  JMP 0x080496cc  
|    ; XREFS: JMP 0x080496e7  JMP 0x08049710  JMP 0x08049739  JMP 0x08049762  JMP 0x0804978b  JMP 0x080497b4  JMP 0x080497dd  JMP 0x080498eb  
|    ; XREFS: JMP 0x080499fd  JMP 0x08049a81  JMP 0x08049ab4  JMP 0x08049ae7  JMP 0x08049b3e  JMP 0x08049b8d  JMP 0x08049bf6  JMP 0x08049c2c  
|    ; XREFS: JMP 0x08049c60  
| 0x08049c67      8b45cc         mov eax, dword [local_34h]
| 0x08049c6a      05c0c00408     add eax, 0x804c0c0
| 0x08049c6f      0fb600         movzx eax, byte [eax]
| 0x08049c72      3c1d           cmp al, 0x1d                          ; 29
| 0x08049c74      0f85b3edffff   jne 0x8048a2d
| ----------- true: 0x08048a2d  false: 0x08049c7a
| 0x08049c7a      8b854cffffff   mov eax, dword [local_b4h]
|    ; JMP XREF from 0x08048a6f (fcn.0804898b + 228)
| 0x08049c80      c9             leave
\ 0x08049c81      c3             ret
```

```
[0x08048540]> px @ 0x0804c060
- offset -   0 1  2 3  4 5  6 7  8 9  A B  C D  E F  0123456789ABCDEF
0x0804c060  7c86 0408 ad86 0408 d486 0408 a987 0408  |...............
0x0804c070  fb86 0408 1b87 0408 4e87 0408 d887 0408  ........N.......
0x0804c080  1986 0408 1388 0408 3488 0408 7b88 0408  ........4...{...
0x0804c090  b688 0408 f188 0408 2c89 0408 6086 0408  ........,...`...
0x0804c0a0  6a86 0408 6789 0408 0000 0000 0000 0000  j...g...........
```


#### 使用 Pin 求解
就像上面那样逆向实在是太难了，不如 Pin 的黑科技。

编译 32 位 pintool：
```
[ManualExamples]$ make obj-ia32/inscount0.so TARGET=
```
随便输入几个长度不同的密码试试：
```
[ManualExamples]$ echo "A" | ../../../pin -t obj-ia32/inscount0.so -o inscount.out -- ~/baleful_unpack ; cat inscount.out 
Please enter your password: Sorry, wrong password!
Count 437603
[ManualExamples]$ echo "AA" | ../../../pin -t obj-ia32/inscount0.so -o inscount.out -- ~/baleful_unpack ; cat inscount.out 
Please enter your password: Sorry, wrong password!
Count 438397
[ManualExamples]$ echo "AAA" | ../../../pin -t obj-ia32/inscount0.so -o inscount.out -- ~/baleful_unpack ; cat inscount.out 
Please enter your password: Sorry, wrong password!
Count 439191
```
```
$ python -c 'print(439191 - 438397)'
794
$ python -c 'print(438397 - 437603)'
794
```
指令执行的次数呈递增趋势，完美，这样只要递增到这个次数有不同时，就可以得到正确的密码长度：
```python
import os

def get_count(flag):
    cmd = "echo " + "\"" + flag + "\"" + " | ../../../pin -t obj-ia32/inscount0.so -o inscount.out -- ~/baleful_unpack"
    os.system(cmd)
    with open("inscount.out") as f:
        count = int(f.read().split(" ")[1])
    return count

flag = "A"
p_count = get_count(flag)
for i in range(50):
    flag += "A"
    count = get_count(flag)
    print("count: ", count)
    diff = count - p_count
    print("diff: ", diff)
    if diff != 794:
        break
    p_count = count
print("length of password: ", len(flag))
```
```
Please enter your password: Sorry, wrong password!
count:  459041
diff:  794
Please enter your password: Sorry, wrong password!
count:  459835
diff:  794
Please enter your password: Sorry, wrong password!
count:  508273
diff:  48438
length of password:  30
```
好，密码长度为 30，接下来是逐字符爆破，首先要确定字符不同对 count 没有影响：
```
[ManualExamples]$ echo "A" | ../../../pin -t obj-ia32/inscount0.so -o inscount.out -- ~/baleful_unpack ; cat inscount.out 
Please enter your password: Sorry, wrong password!
Count 437603
[ManualExamples]$ echo "b" | ../../../pin -t obj-ia32/inscount0.so -o inscount.out -- ~/baleful_unpack ; cat inscount.out 
Please enter your password: Sorry, wrong password!
Count 437603
[ManualExamples]$ echo "_" | ../../../pin -t obj-ia32/inscount0.so -o inscount.out -- ~/baleful_unpack ; cat inscount.out 
Please enter your password: Sorry, wrong password!
Count 437603
```
确实没有，写下脚本：
```python
import os

def get_count(flag):
    cmd = "echo " + "\"" + flag + "\"" + " | ../../../pin -t obj-ia32/inscount0.so -o inscount.out -- ~/baleful_unpack"
    os.system(cmd)
    with open("inscount.out") as f:
        count = int(f.read().split(" ")[1])
    return count

charset = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789_-+*'"

flag = list("A" * 30)
p_count = get_count("".join(flag))
for i in range(30):
    for c in charset:
        flag[i] = c
        print("".join(flag))
        count = get_count("".join(flag))
        print("count: ", count)
        if count != p_count:
            break
    p_count = count
print("password: ", "".join(flag))
```
```
packers_and_vms_and_xors_oh_mx
Please enter your password: Sorry, wrong password!
count:  507925
packers_and_vms_and_xors_oh_my
Please enter your password: Congratulations!
count:  505068
password:  packers_and_vms_and_xors_oh_my
```
简单到想哭。


## 参考资料
- [Pico CTF 2014 : Baleful](https://github.com/ctfs/write-ups-2014/tree/master/pico-ctf-2014/master-challenge/baleful-200)
