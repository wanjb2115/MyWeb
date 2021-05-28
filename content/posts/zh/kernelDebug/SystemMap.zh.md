---
title: "通过System Map调试分析Kernel panics和Kernel Oopses"
date: 2021-05-28T11:28:48+08:00
draft: true
tags: ["LinuxKernel调试"]
categories: ["内核之旅"]
Summary: "通过System Map调试分析Kernel panics和Kernel Oopses"
---

# Linux kernel调试介绍：
---
有多种多样调试kernel的方式，例如打印相关信息，使用kernel符号表，使用kernel Debugger， 但是本文

描述了一些技巧和技术帮助解释一个Oops信息和kernel panic。在这之前，我们应该明白什么是Kernel OOPS和panic。


**Kernel panic** 是当操作系统检测到一个内部致命错误，因为检测到的运行时系统故障，它无法安全地恢复并强制系统挂起/重新引导系统。

Kernel panic的行为是可以通过设置运行时系统配置，例如hung task，使之变为可控的。这就是kernel panic。


**Oops**是执行的kernel 异常处理程序，包括被定义为异常指令的宏，例如BUG()。每一个异常有一个特定的数字。

一些“oops”足够严重，以至于kernel决定执行panic特性立即停止系统运作。这是一次由调用panic的kernel crash。


当一个Kernel Oops在一个运行的kernel中发生，在屏幕上会输出Oops信息，例如 ([ 67.994624] Internal error: Oops: 5 [#1] XXXXXXXXXXX)。 这个Oops信息包含：
- 各个CPU寄存器的值
- 造成这次失败的函数的地址，如PC
- 栈
- 当前正在正在执行的进程名称
通过使用这个OOPS状态，你可以开始调试特定的kernel问题。然而，有些时候，仅根据Oops信息是不够的。

如何通过System.map去解释Oopes
---

在Linux中，Syste.map文件是kernel使用的符号表。

当需要某个符号名的地址，或某个地址的符号名时，System.map文件就是必要的。而且该文件对调试kernel panic和kernel oopes十分有效。当CONFIG_KALLSYMS被启用时，Kernel自己进行address-to-name的翻译，所以一些如ksymoops的工具就不需要了。

更多细节信息可以参考：[System.map](https://en.wikipedia.org/wiki/System.map).

注意：System.map内的地址可能每次都会变化，换句话说，每次构建kernel都会生成新的System.map，但是必须有已报告kernel panic/Oopes的同一Linux内核的System.map才能调试问题。

注意在logs中kernel调用栈内，，kernel会寻找与要分析的地址最接近的符号，由于内联，静态和优化，并非所有的函数符号都是可用的，因此有时候报告的函数名称并不是故障所在的位置。



