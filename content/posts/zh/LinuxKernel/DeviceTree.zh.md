---
title: "Device Tree"
date: 2021-09-14T15:00:38+08:00
draft: true
tags: ["Linux内核"]
categories: ["内核之旅"]
Summary: "Linux设备树学习"
---
# 术语

- DTS：设备树符号表，一个可读的设备树文件
- DTB：设备树的二进制表示
- DTC：设备树编译器，一个将DTS文件编译为DTB文件的开源工具

# 节点名与路径名

** node-name@unit-address **

node-name：表示该节点的名称，描述设备的类型。

unit-address：特定于节点所在的总线类型。必须匹配reg属性中的第一个地址，如果节点没有reg属性，则unit-address应该被忽略

设备树中的任意一个节点可用完整路径来表示，根目录为'/'，例如/node-name-1/node-name-2/node-name-N

# 标准属性
## compatible
值类型：<stringlist>

描述：

## model

## phandle

## status

## #address-cells 和 #size-cells

## reg

## virtual-reg

## ranges

## dma-ranges

## name(不建议)

## device_type(不建议)


