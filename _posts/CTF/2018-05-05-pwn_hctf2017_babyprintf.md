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
|       |   ; JMP XREF from 0x00400832 (main)
|      .--> 0x004007d0      mov edi, eax
|      :|   0x004007d2      call sym.imp.malloc                        ;  void *malloc(size_t size)
|      :|   0x004007d7      mov esi, str.string:                       ; 0x400aa4 ; "string: "
|      :|   0x004007dc      mov rbx, rax
|      :|   0x004007df      mov edi, 1
|      :|   0x004007e4      xor eax, eax
|      :|   0x004007e6      call sym.imp.__printf_chk
|      :|   0x004007eb      mov rdi, rbx
|      :|   0x004007ee      xor eax, eax
|      :|   0x004007f0      call sym.imp.gets                          ; char*gets(char *s)
|      :|   0x004007f5      mov esi, str.result:                       ; 0x400aad ; "result: "
|      :|   0x004007fa      mov edi, 1
|      :|   0x004007ff      xor eax, eax
|      :|   0x00400801      call sym.imp.__printf_chk
|      :|   0x00400806      mov rsi, rbx
|      :|   0x00400809      mov edi, 1
|      :|   0x0040080e      xor eax, eax
|      :|   0x00400810      call sym.imp.__printf_chk
|      :|   ; JMP XREF from 0x004007c8 (main)
|      :`-> 0x00400815      mov esi, str.size:                         ; 0x400a94 ; "size: "
|      :    0x0040081a      mov edi, 1
|      :    0x0040081f      xor eax, eax
|      :    0x00400821      call sym.imp.__printf_chk
|      :    0x00400826      xor eax, eax
|      :    0x00400828      call sub._IO_getc_990
|      :    0x0040082d      cmp eax, 0x1000
|      `==< 0x00400832      jbe 0x4007d0
|           0x00400834      mov edi, str.too_long                      ; 0x400a9b ; "too long"
|           0x00400839      call sym.imp.puts                          ; int puts(const char *s)
|           0x0040083e      mov edi, 1
\           0x00400843      call sym.imp.exit                          ; void exit(int status)
```

#### read
```
[0x00400850]> pdf @ sub._IO_getc_990 
/ (fcn) sub._IO_getc_990 122
|   sub._IO_getc_990 ();
|           ; CALL XREF from 0x00400828 (main)
|           0x00400990      push rbp
|           0x00400991      push rbx
|           0x00400992      xor ebx, ebx
|           0x00400994      sub rsp, 0x28                              ; '('
|           0x00400998      mov rax, qword fs:[0x28]                   ; [0x28:8]=-1 ; '(' ; 40
|           0x004009a1      mov qword [rsp + 0x18], rax
|           0x004009a6      xor eax, eax
|           0x004009a8      nop dword [rax + rax]
|           ; JMP XREF from 0x004009ce (sub._IO_getc_990)
|       .-> 0x004009b0      mov rdi, qword [obj.stdin]                 ; [0x601090:8]=0
|       :   0x004009b7      movsxd rbp, ebx
|       :   0x004009ba      call sym.imp._IO_getc
|       :   0x004009bf      cmp al, 0xa                                ; 10
|       :   0x004009c1      mov byte [rsp + rbx], al
|      ,==< 0x004009c4      je 0x400a05
|      |:   0x004009c6      add rbx, 1
|      |:   0x004009ca      cmp rbx, 9                                 ; 9
|      |`=< 0x004009ce      jne 0x4009b0
|      |    0x004009d0      cmp byte [rsp + 9], 0xa                    ; [0xa:1]=255 ; 10
|      |,=< 0x004009d5      je 0x400a00
|      ||   ; JMP XREF from 0x00400a09 (sub._IO_getc_990)
|     .---> 0x004009d7      xor edx, edx
|     :||   0x004009d9      xor esi, esi
|     :||   0x004009db      mov rdi, rsp
|     :||   0x004009de      call sym.imp.strtoul                       ; long strtoul(const char *str, char**endptr, int base)
|     :||   0x004009e3      mov rcx, qword [rsp + 0x18]                ; [0x18:8]=-1 ; 24
|     :||   0x004009e8      xor rcx, qword fs:[0x28]
|    ,====< 0x004009f1      jne 0x400a0b
|    |:||   0x004009f3      add rsp, 0x28                              ; '('
|    |:||   0x004009f7      pop rbx
|    |:||   0x004009f8      pop rbp
|    |:||   0x004009f9      ret
     |:||   0x004009fa      nop word [rax + rax]
|    |:||   ; JMP XREF from 0x004009d5 (sub._IO_getc_990)
|    |:|`-> 0x00400a00      mov ebp, 9
|    |:|    ; JMP XREF from 0x004009c4 (sub._IO_getc_990)
|    |:`--> 0x00400a05      mov byte [rsp + rbp], 0
|    |`===< 0x00400a09      jmp 0x4009d7
|    |      ; JMP XREF from 0x004009f1 (sub._IO_getc_990)
\    `----> 0x00400a0b      call sym.imp.__stack_chk_fail              ; void __stack_chk_fail(void)
```


## 漏洞利用

开启 ASLR，Bingo!!!
```

```

#### exploit
完整 exp 如下：
```python

```


## 参考资料
- https://github.com/spineee/hctf/tree/master/2017/babyprintf
