---
title: "ArchLinux安装syzkaller简记"
date: 2021-07-01T21:29:38+08:00
draft: true
tags: ["Linux工具"]
categories: ["内核之旅"]
Summary: "尝试ArchLinux上运行syzkaller"
---

# qemu
## 下载qemu


## 安装必要工具和依赖库
必要
```bash
pacman -S make pkg-config ninja pixman
pacman -S --needed base-devel
```

可选
```bash
pacman -S libssh snappy libusb libaio libnfs libiscsi gtk3 libpng fufe3 capstone python-sphinx libgcrypt libslirp 
```

# 文件系统
## 安装必要工具和依赖库
必要
```bash
pacman -S bc rsync cpio 
```

