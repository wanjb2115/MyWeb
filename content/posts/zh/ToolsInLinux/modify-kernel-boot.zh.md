---
title: "更改Linux发行版的内核"
date: 2024-02-11T21:42:38+08:00
draft: true
tags: ["Linux工具"]
categories: ["内核之旅"]
Summary: "下载编译内核，并通过命令更改发行版的boot"
---

### 参考链接
[LinuxFundation:building-and-installing-your-first-kernel](https://trainingportal.linuxfoundation.org/learn/course/a-beginners-guide-to-linux-kernel-development-lfd103/building-and-installing-your-first-kernel/building-and-installing-your-first-kernel)


## 1、克隆内核源代码
Start by cloning the stable kernel git, building and installing the latest stable kernel. The stable cloning step below will create a new directory named linux-stable and populate it with the sources. The stable repository has several branches going back to linux-2.6.11.y. Let’s start with the latest stable release branch. As of this writing (April 2021), it is linux-5.12.y, which is used in the following example. You can find the latest stable or a recent active kernel release.

```bash
git clone https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux-stable.git linux_stable
cd linux_stable
git branch -a | grep linux-5
    remotes/origin/linux-5.12.y
    remotes/origin/linux-5.11.y
    remotes/origin/linux-5.10.y

​git checkout linux-5.12.y
```

## 2、查看内核配置文件
Copying the Configuration for Current Kernel from /boot


Starting out with the distribution configuration file is the safest approach for the very first kernel install on any system. You can do so by copying the configuration for your current kernel from /proc/config.gz or /boot. As an example, I am running Ubuntu 19.04 and config-5.0.0-21-generic is the configuration I have in /boot on my system. Pick the latest configuration you have on your system and copy that to linux_stable/.config. In the following example, config-5.0.0-21-generic is the latest kernel configuration.

```bash
ls /boot
config-5.0.0-20-generic        memtest86+.bin
config-5.0.0-21-generic        memtest86+.elf
efi                            memtest86+_multiboot.bin
grub                           System.map-5.0.0-20-generic
initrd.img-5.0.0-20-generic    System.map-5.0.0-21-generic
initrd.img-5.0.0-21-generic    vmlinuz-5.0.0-20-generic
lost+found                     vmlinuz-5.0.0-21-generic

cp /boot/<config-5.0.0-21-generic> .config
```

## 3、编译内核
Compiling the Kernel


Run the following command to generate a kernel configuration file based on the current configuration. This step is important to configure the kernel, which has a good chance to work correctly on your system. You will be prompted to tune the configuration to enable new features and drivers that have been added since Ubuntu snapshot the kernel from the mainline. make all will invoke make oldconfig in any case. I am showing these two steps separately just to call out the configuration file generation step.

```bash
make oldconfig

Another way to trim down the kernel and tailor it to your system is by using make localmodconfig. This option creates a configuration file based on the list of modules currently loaded on your system.

lsmod > /tmp/my-lsmod
make LSMOD=/tmp/my-lsmod localmodconfig

Once this step is complete, it is time to compile the kernel. Using the -j option helps the compiles go faster. The -j option specifies the number of jobs (make commands) to run simultaneously:

make -j3 all
```

## 4、更换内核版本

Installing the New Kernel


Once the kernel compilation is complete, install the new kernel:

```bash
su -c "make modules_install install"
```

The above command will install the new kernel and run update-grub to add the new kernel to the grub menu. Now it is time to reboot the system to boot the newly installed kernel. Before we do that, let's save logs from the current kernel to compare and look for regressions and new errors, if any. Using the -t option allows us to generate dmesg logs without the timestamps, and makes it easier to compare the old and the new.

```bash
dmesg -t > dmesg_current
dmesg -t -k > dmesg_kernel
dmesg -t -l emerg > dmesg_current_emerg
dmesg -t -l alert > dmesg_current_alert
dmesg -t -l crit > dmesg_current_crit
dmesg -t -l err > dmesg_current_err
dmesg -t -l warn > dmesg_current_warn
dmesg -t -l info > dmesg_current_info
```

In general, dmesg should be clean, with no emerg, alert, crit, and err level messages. If you see any of these, it might indicate some hardware and/or kernel problem.

## 5、启动新版本的内核
Booting the Kernel

Run update-grub to update the grub configuration in /boot

sudo update-grub

Now, it’s time to restart the system. Once the new kernel comes up, compare the saved dmesg from the old kernel with the new one, and see if there are any regressions. If the newly installed kernel fails to boot, you will have to boot a good kernel, and then investigate why the new kernel failed to boot.​

These steps are not specific to stable kernels. You can check out linux mainline or linux-next and follow the same recipe of generating a new configuration from an oldconfig, build, and install the mainline or linux-next kernels