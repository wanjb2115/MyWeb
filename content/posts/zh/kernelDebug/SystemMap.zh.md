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

如何调试kernel panics和Oopes
---
```bash
67.994406] Unable to handle kernel paging request at virtual address 02120bc4
[   67.994495] pgd = 94240000
[   67.994553] [02120bc4] *pgd=00000000
[   67.994624] Internal error: Oops: 5 [#1] PREEMPT SMP ARM
[   67.994926] CPU: 0    Not tainted  (3.8.13.23-XXXXXXXX #1)
[   67.994996] PC is at add_range+0x14/0x6c
[   67.995056] LR is at XXXXXXX+0x38/0x44
[   67.995117] pc : [<80049F3C>]    lr : [<8004a1ec>]    psr: 20000013
[   67.995117] sp : 9423fd90  ip : 9423fda8  fp : 9423fda4
[   67.995176] r10: 00000000  r9 : 9423ff60  r8 : 8000da84
[   67.995233] r7 : 000041fd  r6 : 00000081  r5 : aa068088  r4 : aa068088
[   67.995290] r3 : ac8ceb80  r2 : 021ab618  r1 : 00000000  r0 : 02120bc0
[   67.995348] Flags: nzCv  IRQs on  FIQs on  Mode SVC_32  ISA ARM  Segment user
[   67.995406] Control: 10c5387d  Table: 2424004a  DAC: 00000015
[   67.995462] Process cat (pid: 1352, stack limit = 0x9423e238)
[   67.995518] Stack: (0x9423fd90 to 0x94240000)
[   67.995577] fd80:                                     aa068088 aa068088 9423fdb4 9423fda8

```

这是一个kernel在“add_range”函数崩溃时的kernel回溯，让我们来一步一步地分析。

1. Crash每次都发生在下面的位置：
```bash
PC is at add_range +0x14/0x6c
```
2. 筛选(grep)/寻找 add_range()函数在System.map文件中，记录下符号名称对应的地址，例如80049f28
```bash
#Linux-Kernel # grep add_range System.map 
80049f28 T add_range
```