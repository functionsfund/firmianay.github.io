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
$ strings libc.so.6 | grep "GNU C"
GNU C Library (Ubuntu GLIBC 2.23-0ubuntu3) stable release version 2.23, by Roland McGrath et al.
Compiled by GNU CC version 5.3.1 20160413.
```
64 位程序，保护全开，默认开启 ASLR。


## 题目解析

## 漏洞利用

## 参考资料
- https://ctftime.org/task/4811
