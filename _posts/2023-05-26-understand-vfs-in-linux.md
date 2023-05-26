---
title: Understand vfs in linux
date: 2023-05-26 10:00:00 +0800
categories: [Notes, Learning]
tags: [File System, OS]
img_path: /assets/img/fs/
---

## 前言

## POSIX File System

[POSIX](https://unix.org/version3/online.html) 想必所有写过代码的人都很熟悉，就是 IEEE 定义的一套标准，用于定义操作系统的编程接口，从而使得不同的操作系统能够兼容同一套代码。

## File and Directory related api/syscall

### mount

一直以来 linux 中使用的 mount 系统调用都是旧的版本：

- mount: sys_mount
- umount2: sys_umount

在2018年的LSFMM大会上 Al Viro 等人提出了[新 mount 系统调用](https://patchwork.kernel.org/project/linux-security-module/cover/153754740781.17872.7869536526927736855.stgit@warthog.procyon.org.uk/)的想法。

fsopen: sys_fsopen

fsmount: sys_fsmount

fsconfig: sys_fsconfig

move_mount: sys_move_mount

mount_setattr: sys_mount_setattr

可以用 glibc 提供的 syscall 函数来调用新的系统调用：

### files

- open: sys_open -> do_sys_open -> do_sys_openat2 -> do_filp_open -> path_openat -> link_path_walk -> walk_component ->
  - lookup_slow
  - step_into -> handle_mounts -> traverse_mounts
