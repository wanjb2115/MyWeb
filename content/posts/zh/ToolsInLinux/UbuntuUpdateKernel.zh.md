---
title: "Ubuntu 22.04更换内核并加载ko"
date: 2024-07-30T20:41:38+08:00
draft: true
tags: ["Linux工具"]
categories: ["内核之旅"]
Summary: "Ubuntu 22.04升级内核，编写并加载ko"
---

在[更改LINUX发行版的内核](http://wanjiabing.top/posts/zh/toolsinlinux/modify-kernel-boot/)了解到我们可以更换Linux发行版的内核。
最近刚好有一台联想笔记本闲置，想着把它改造成我的内核开发本吧。

安装好了Ubuntu 22.04，打算更换内核版本，安装几个自己编写的kernel驱动，谁知踩坑不断，遂把这些坑记录下来

## 第一坑：编译不过
内核编译报错: No rule to make target ‘debian/canonical-certs.pem‘, needed by ‘certs/x509_certificate_list‘

参考链接：[内核编译报错](https://blog.csdn.net/guo__heng/article/details/134334295)

如果报错 canonical-certs.pem：
```bash
scripts/config --disable  SYSTEM_TRUSTED_KEYS
```

如果报错 canonical-revoked-certs.pem：
```bash
scripts/config --disable  SYSTEM_REVOCATION_KEYS
```

## 第二坑：无法安装内核make modules_install install

关闭Secure boot

参考链接； [Key was rejected by service](https://blog.csdn.net/hydront/article/details/130280075)

1、输入代码
```bash
sudo mokutil --disable-validation
```
2、设置密码

3、重启电脑

4、关闭secure boot

5、开机重新编译

## others: 编译模块ko并加载

```bash
make -C . M=drivers/xxx modules
insmod xxx.ko
rmmod xxx.ko
```
