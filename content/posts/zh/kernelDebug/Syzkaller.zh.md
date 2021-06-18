---
title: "Arm64上的Syzkaller之旅"
date: 2021-06-17T20:28:48+08:00
draft: true
tags: ["LinuxKernel调试"]
categories: ["内核之旅"]
Summary: "如何基于Qemu使用Syzkaller对Arm64平台Linux Kernel进行测试"
---

# Syzkaller

syzkaller ([siːzˈkɔːlə]) 是一个无监督的，全覆盖的，引导内核的模糊测试器。
支持的操作系统：Akaros、FreeBSD、Fuchsia、gVisor、Linux、NetBSD、OpenBSD、Windows。

Github链接：[Syzkaller](https://github.com/google/syzkaller).

# 安装Go

参考链接：[Go安装](https://github.com/google/syzkaller/blob/master/docs/linux/setup.md#install)

```bash
wget https://dl.google.com/go/go1.14.2.linux-amd64.tar.gz       
tar -xf go1.14.2.linux-amd64.tar.gz                          
mv go goroot
mkdir gopath
export GOPATH=`pwd`/gopath  # Syzkaller使用
export GOROOT=`pwd`/goroot  # Syzkaller使用
export PATH=$GOPATH/bin:$PATH # 添加环境变量
export PATH=$GOROOT/bin:$PATH # 添加环境变量
```

下载新版本go压缩包，解压后，在文件夹*bin*目录可得到go编译器，把当前目录添加到环境变量PATH中，即可全局使用go编译器。

使用
```bash
go version
```
查看go编译器版本

# 安装Qemu

参考链接：[Qemu安装](https://github.com/google/syzkaller/blob/master/docs/linux/setup_linux-host_qemu-vm_arm64-kernel.md)

```bash
wget https://download.qemu.org/qemu-6.0.0.tar.xz
tar -xvf qemu-6.0.0.tar.xz
```

下载新版本qemu压缩包，解压后，进入目录运行：
```bash
./configure
make -j4
```

>若出现“ERROR: Cannot find Ninja”
>安装

>*sudo apt-get install ninja-build*

*出现其他错误，安装相应的依赖库*

make编译好后，在 *build/aarch64-softmmu* 文件夹内可以找到qemu模拟器程序 *qemu-system-aarch64*.

# 安装交叉编译链

```bash
wget https://releases.linaro.org/components/toolchain/binaries/latest-7/aarch64-linux-gnu/gcc-linaro-7.5.0-2019.12-x86_64_aarch64-linux-gnu.tar.xz
tar -xvf gcc-linaro-7.5.0-2019.12-x86_64_aarch64-linux-gnu.tar.xz
```

下载新版本*aarch64-linux-gnu-*交叉编译工具包，解压后，在文件夹bin目录可得到交叉编译器，把bin目录添加到环境变量PATH中，即可全局使用交叉编译工具。

# 制作磁盘映像

参考链接：[制作磁盘映像](https://github.com/google/syzkaller/blob/master/docs/linux/setup_linux-host_qemu-vm_arm64-kernel.md)

```bash
git clone git://git.buildroot.net/buildroot         
cd buildroot
make menuconfig
```

选择
```bash
Target options
    Target Architecture - Aarch64 (little endian)
Toolchain type
    External toolchain - Linaro AArch64
System Configuration
[*] Enable root login with password
        ( ) Root password = set your password using this option
[*] Run a getty (login prompt) after boot  --->
    TTY port - ttyAMA0
Target packages
    [*]   Show packages that are also provided by busybox
    Networking applications
        [*] dhcpcd
        [*] iproute2
        [*] openssh
Filesystem images
    [*] ext2/3/4 root filesystem
        ext2/3/4 variant - ext3
        exact size in blocks - 6000000
    [*] tar the root filesystem
```

运行
```
make
```

在编译之后，确保 *output/images/rootfs.ext3* 文件存在。


# 编译内核

进入Linux源码目录，运行

```bash
ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- make defconfig
vim .config
```
修改配置文件中的如下选项：
```bash
CONFIG_KCOV=y
CONFIG_KASAN=y
CONFIG_DEBUG_INFO=y
CONFIG_CMDLINE="console=ttyAMA0"
CONFIG_KCOV_INSTRUMENT_ALL=y
CONFIG_DEBUG_FS=y
CONFIG_NET_9P=y
CONFIG_NET_9P_VIRTIO=y
CONFIG_CROSS_COMPILE="aarch64-linux-gnu-"
```

编译内核
```bash
ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- make -j4
```

在编译成功后，你应该可以看到 *arch/arm64/boot/Image* 文件生成。


# 测试Qemu虚拟机

在qemu模拟器目录下运行：
```bash
./aarch64-softmmu/qemu-system-aarch64 \
  -machine virt \
  -cpu cortex-a57 \
  -nographic -smp 1 \
  -hda /你的磁盘映像目录/rootfs.ext3 \
  -kernel /你的Linux源码目录/arch/arm64/boot/Image \
  -append "console=ttyAMA0 root=/dev/vda oops=panic panic_on_warn=1 panic=-1 ftrace_dump_on_oops=orig_cpu debug earlyprintk=serial slub_debug=UZ" \
  -m 2048 \
  -net user,hostfwd=tcp::10023-:22 -net nic
```

如果顺利的话，可以通过qemu模拟器运行终端，现在你有一个Linux终端了。

# 配置Qemu虚拟机
## 配置虚拟机网络和kcov
在 */etc/init.d/S50sshd* 文件的最上面添加以下配置：
```bash
ifconfig eth0 up
dhcpcd
mount -t debugfs none /sys/kernel/debug
chmod 777 /sys/kernel/debug/kcov
```
注释以下语句：
```bash
/usr/bin/ssh-keygen -A
```

## 配置sshd_config
打开 */etc/ssh/sshd_config* ，修改成如下配置：
```bash
PermitRootLogin yes
PubkeyAuthentication yes
AuthorizedKeysFile      /root/.ssh/authorized_keys
PasswordAuthentication yes
```

## 重启虚拟机

```bash
reboot
```

## 测试主机与虚拟机的通讯

参考链接：[使用密钥对登录服务器](https://www.jianshu.com/p/fab3252b3192)

返回主机，新开终端，保持虚拟机运行，使用
```bash
ssh-keygen -o 
```
生成密钥对，生成的密钥对在 *~/.ssh* 文件夹中

通过命令:
```bash
ssh root@localhost -p 10023
```
输入虚拟机密码，查看能否成功连接虚拟机。

通过命令：
```bash
scp -P 10023 ~/.ssh/id_rsa.pub root@localhost:/root/.ssh/authorized_keys
```
将公钥拷贝至虚拟机中

通过命令:
```bash
ssh -i /home/wanjb/.ssh/id_rsa root@localhost -p 10023
```
查看能否成功连接虚拟机。

# 编译Syzkaller
下载Syzkaller:
```bash
go get -u -d github.com/google/syzkaller/prog
cd gopath/src/github.com/google/syzkaller/
CC=gcc-linaro-6.3.1-2017.05-x86_64_aarch64-linux-gnu/bin/aarch64-linux-gnu-g++ # 改成你的交叉编译链
make TARGETARCH=arm64
```

编译完成后在 *bin* 目录得到syz-manager.

# 配置Syzkaller

添加syzkaller配置文件：

```bash
vi my.cfg
```

```bash
{
    "name": "QEMU-aarch64",
    "target": "linux/arm64",
    "http": ":56700",
    "workdir": "/path/to/a/dir/to/store/syzkaller/corpus", # 工作路径，可以自定义
    "kernel_obj": "/path/to/linux/build/dir", # Linux源代码路径
    "syzkaller": "/path/to/syzkaller/arm64/", # Syzkaller文件夹路径
    "image": "/path/to/rootfs.ext3", # 磁盘映像
    "sshkey": "/path/to/ida_rsa", # 主机ssh公钥
    "procs": 8,
    "type": "qemu",
    "vm": {
        "count": 1,
        "qemu": "/path/to/qemu-system-aarch64", # qemu模拟器路径
        "cmdline": "console=ttyAMA0 root=/dev/vda",
        "kernel": "/path/to/Image", # 编译好的Linux内核镜像
        "cpu": 2,
        "mem": 2048
    }
}
```

# 使用Syzkaller
运行
```bash
./syz-manager -config my.cfg
```

之后访问 *http://localhost:56700/* ,等待一段时间，便可查看syzkaller运行结果。

*注意：在运行syzkaller之前需要退出qemu，让出磁盘映像。*

Enjoy~![捂脸](http://wanjiabing.top/emoj/2018new_wu_org.png)