---
title: "ArchLinux安装简记"
date: 2021-07-01T21:29:38+08:00
draft: true
tags: ["Linux工具"]
categories: ["内核之旅"]
Summary: "尝试ArchLinux"
---
ArchLinux发行版有非常完整的wiki说明指南，强大的指南可以指导我们迅速上手。

[ArchWiki](https://wiki.archlinux.org)

# 下载ArchLinux镜像文件
在ArchLinux下载界面选择合适的下载节点，下载所需的镜像文件。

[Arch下载](https://archlinux.org/download/)

# 制作启动盘
windows系统使用rufus软件制作u盘启动盘，将安装镜像写入u盘中。

[rufus](https://rufus.ie/en_US/)

# 使用ESC键进入uefi启动
重启计算机，按"esc"键选择u盘启动

* 注意需要关闭Secure boot *

# 进入ArchLinux安装命令行

ArchWiki有完整的安装流程，这里仅记录一些要点。

[ArchLinux安装指南](https://wiki.archlinux.org/title/Installation_guide_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87))

## 连接无线网
使用iwctl连接无线网

```bash
iwctl
station xxx connect xxx
```

退出后，测试网络连接是否正常。

```bash
ping archlinux.org
```

## 清理磁盘和进行磁盘分区
使用fdisk清理磁盘
```bash
fdisk -l
fdisk /dec/sdxx
```

## 建立文件系统
磁盘分区完成后为每个磁盘创建相应文件系统。

uefi分区创建FAT格式;
swap分区创建SWAP格式;
主分区可以自定义.

## 挂载磁盘和交换分区
swapon 设置交换分区;主分区挂载到/mnt;uefi分区挂载到/mnt/boot.

× 如果没有boot文件夹可以自己创建 × 

## 安装核心文件到挂载磁盘
根据wiki指南安装应用程序到/mnt中，建议安装

```
pacstrap /mnt base linux linux-firmware dhcpd iwd nano vim
```

## 配置自动挂载

用以下命令生成 fstab 文件 (用 -U 或 -L 选项设置UUID 或卷标)：
```bash
genfstab -U /mnt >> /mnt/etc/fstab
```
强烈建议在执行完以上命令后，后检查一下生成的 /mnt/etc/fstab 文件是否正确。 

## 进入磁盘镜像
```bash
arch-chroot /mnt
```

## 设置相关参数

设置时区，本地化，网络配置等等。

本地化语言强烈推荐按照wiki指南设置为en_US.UTF-8

## 安装必要软件

```bash
pacman -S git zsh htop
```

## 设置grub启动选项

### 安装初始化grub

```bash
pacman -S grub
grub-install --target=x86_64-efi --efi-directory=esp --bootloader-id=GRUB
```

### 安装处理器微码

- AMD：amd-ucode
- Intel：intel-ucode

### 更新GRUB配置信息

```bash
grub-mkconfig -o /boot/grub/grub.cfg
```

## 设置pacman最快镜像列表

```bash
pacman -S reflector
reflector --country China --age 12 --protocol https --sort rate --save /etc/pacman.d/mirrorlist
```


## 重启系统安装完成

### 退出arch-chroot

### reboot

## 其他

- 重启系统后对于无线网可使用systemctl start iwd,开启iwd服务