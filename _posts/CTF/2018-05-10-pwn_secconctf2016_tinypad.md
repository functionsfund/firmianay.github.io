---
layout: post
title: pwn SECCONCTF2016 tinypad
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
$ file tinypad 
tinypad: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 2.6.32, BuildID[sha1]=1333a912c440e714599a86192a918178f187d378, not stripped
$ checksec -f tinypad
RELRO           STACK CANARY      NX            PIE             RPATH      RUNPATH      FORTIFY Fortified Fortifiable  FILE
Full RELRO      Canary found      NX enabled    No PIE          No RPATH   No RUNPATH   Yes     0               4       tinypad
$ strings libc-2.19.so | grep "GNU C"
GNU C Library (Ubuntu EGLIBC 2.19-0ubuntu6.9) stable release version 2.19, by Roland McGrath et al.
Compiled by GNU CC version 4.8.4.
```


## 题目解析

## 漏洞利用

## 参考资料
- https://ctftime.org/task/3189
