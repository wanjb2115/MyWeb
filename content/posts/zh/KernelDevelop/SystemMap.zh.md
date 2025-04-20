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

当需要某个符号名的地址，或某个地址的符号名时，System.map文件就是必要的。而且该文件对调试kernel panic和kernel oops十分有效。当CONFIG_KALLSYMS被启用时，Kernel自己进行address-to-name的翻译，所以一些如ksymoops的工具就不需要了。

更多细节信息可以参考：[System.map](https://en.wikipedia.org/wiki/System.map).

注意：System.map内的地址可能每次都会变化，换句话说，每次构建kernel都会生成新的System.map，但是必须有已报告kernel panic/Oops的同一Linux内核的System.map才能调试问题。

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

这是一个kernel在“add_range”函数崩溃时的kernel调用栈，让我们来一步一步地分析。

1. Crash每次都发生在下面的位置：
```bash
PC is at add_range +0x14/0x6c
```
2. 筛选(grep)/寻找 add_range()函数在System.map文件中，记录下符号名称对应的地址，例如80049f28
```bash
#Linux-Kernel # grep add_range System.map 
80049f28 T add_range
```
3. 用add_range符号地址替换符号名称。"add_range"+0x14 = 80049f28 + 0x14 = 80049F3C

4. "80049F3C"也应该和kernel调用栈中的PC地址一致，这正说明我正使用的Kernel版本与出错的kernel版本一致。

5. 运行objdump结合vmlinux得到反汇编和程序的更多细节。

**objdump**: 一个展示关于object文件不同种类信息的程序。例如，它可以被用作反汇编器以汇编的形式查看正在运行的程序。

**vmlinux**： 一个包含Linux支持的object文件形式的Linux kernel静态链接可执行文件。vmlinux常用来做Kernel调试，符号表生成和其他用途。

```bash
#objdump -D -S --show-raw-insn --prefix-addresses --line-numbers vmlinux > objdump
```
6. 在vmlinux.object寻找"add_range"，并且寻找上面计算的PC地址。80049F3C
```bash
80049F3C <add_Range+0x14> e5903004  ldr     r3, [r0, #4]
```

7. 崩溃点可以被识别出来，如下：
```bash
ldr     r3, [r0, #4] = r0+4 = 02120bc0+4 = 02120bc4 /*replace r0 with r0 register value from the Back Traces*
```

8. 哇!这与错误日志中的地址相同
```bash
Unable to handle kernel paging request at virtual address 02120bc4
```

总结：这里r0寄存器指向一个无效地址，从反汇编中可以找出r0寄存器指的地址以及查找出为什么r0指向了一个无效的地址。

# 使用GDB寻找你的kernel panic或oops的位置
---
一个寻找出现kernel panic或oops代码行快速和简单的方式是使用GDB的list命令。你可以这样做。

让我们假设你的panic/oops信息如下：
```bash
[  174.507084] Stack:
[  174.507163] 
ce0bd8ac 00000008 00000000 ce4a7e90 c039ce30 ce0bd8ac c0718b04 c07185a0
[  174.507380] 
ce4a7ea0 c0398f22 ce0bd8ac c0718b04 ce4a7eb0 c037deee ce0bd8e0 ce0bd8ac
[  174.507597] 
ce4a7ec0 c037dfe0 c07185a0 ce0bd8ac ce4a7ed4 c037d353 ce0bd8ac ce0bd8ac
[  174.507888] Call Trace:
[  174.508125] 
[<c039ce30>] ? sd_remove+0x20/0x70
[  174.508235] 
[<c0398f22>] ? scsi_bus_remove+0x32/0x40
[  174.508326] 
[<c037deee>] ? __device_release_driver+0x3e/0x70
[  174.508421] 
[<c037dfe0>] ? device_release_driver+0x20/0x40
[  174.508514] 
[<c037d353>] ? bus_remove_device+0x73/0x90
[  174.508606] 
[<c037bccf>] ? device_del+0xef/0x150
[  174.508693] 
[<c0399207>] ? __scsi_remove_device+0x47/0x80
[  174.508786] 
[<c0399262>] ? scsi_remove_device+0x22/0x40
[  174.508877] 
[<c0399324>] ? __scsi_remove_target+0x94/0xd0
[  174.508969] 
[<c03993c0>] ? __remove_child+0x0/0x20
[  174.509060] 
[<c03993d7>] ? __remove_child+0x17/0x20
[  174.509148] 
[<c037b868>] ? device_for_each_child+0x38/0x60
[  174.509241] 
[<c039938f>] ? scsi_remove_target+0x2f/0x60
[  174.509393] 
[<d0c38907>] ? __iscsi_unbind_session+0x77/0xa0
[scsi_transport_iscsi]
[  174.509699] 
[<c015272e>] ? run_workqueue+0x6e/0x140
[  174.509801] 
[<d0c38890>] ? __iscsi_unbind_session+0x0/0xa0
[scsi_transport_iscsi]
[  174.509977] 
[<c0152888>] ? worker_thread+0x88/0xe0
[  174.510047] 
[<c01566a0>] ? autoremove_wake_function+0x0/0x40
```
如果你想知道哪一行代码代表sd_remove+0x20/0x70，进入你的kernel目录树，在包含函数sd_remove()的".o"文件运行gdb，这个例子中是"sd.o"，然后运行gdb的list命令，(gdb) list *(function+0xoffset), 在这个例子中function是sd_remove()，offset是0x20, gdb会告诉你发生panic或oops的行号。在大多数情况下，这都是可信的。
```bash
#gdb sd.o
gdb)list *(sd_remove+0x20)
0x1650 is in sd_remove
(Kernel/linux-xxx/drivers/scsi/sd.c:2125).
2120    static int sd_remove(struct device *dev)
2121    {
2122            struct scsi_disk *sdkp;
2123    
2124            async_synchronize_full();
2125            sdkp = dev_get_drvdata(dev);
2126           
blk_queue_prep_rq(sdkp->device->request_queue, scsi_prep_fn);
2127            device_del(&sdkp->dev);
2128            del_gendisk(sdkp->disk);
2129            sd_shutdown(dev);
(gdb)

so dev_get_drvdata()is the function where crash has been happened and Lets analyze why dev_get_drvdata(dev)is crashing.
```

# 反汇编kernel
---
交叉工具是必须的。

## objdump功能
主要功能
```bash
arm-none-linux-gnueabi-objdump –dr vmlinux  /*If We have object code handy then, we can disassemble the individual object file also like objdump -S panic.o"
```

## gdb在vmlinux
可以使用gdb在vmlinux镜像中反汇编一个已构建的kernel。这在得到一个kernel oops信息和栈转储（一种可以反汇编object代码和查看oops发生位置的信息）时是十分有效的。例如，
```bash
#arm-none-linux-gnueabi-gdb –silent vmlinux
#disassemble printk
Dump of assembler code for function printk:
0xffffffff8023dce0 <printk+0>:  sub    $0xd8,%rsp
0xffffffff8023dce7 <printk+7>:  lea    0xe0(%rsp),%rax
0xffffffff8023dcef <printk+15>: mov    %rsi,0x28(%rsp)
0xffffffff8023dcf4 <printk+20>: mov    %rsp,%rsi
0xffffffff8023dcf7 <printk+23>: mov    %rdx,0x30(%rsp)
0xffffffff8023dcfc <printk+28>: mov    %rcx,0x38(%rsp)
0xffffffff8023dd01 <printk+33>: mov    %rax,0x8(%rsp)
0xffffffff8023dd06 <printk+38>: lea    0x20(%rsp),%rax
0xffffffff8023dd0b <printk+43>: mov    %r8,0x40(%rsp)
0xffffffff8023dd10 <printk+48>: mov    %r9,0x48(%rsp)
0xffffffff8023dd15 <printk+53>: movl   $0x8,(%rsp)
0xffffffff8023dd1c <printk+60>: movl   $0x30,0x4(%rsp)
0xffffffff8023dd24 <printk+68>: mov    %rax,0x10(%rsp)
0xffffffff8023dd29 <printk+73>: callq  0xffffffff8023d980 <vprintk>
0xffffffff8023dd2e <printk+78>: add    $0xd8,%rsp
0xffffffff8023dd35 <printk+85>: retq   
End of assembler dump.
```

# 如何解释汇编语言（EABI C函数调用与ARM寄存器的映射）
首先第一步，我们需要通过使用OBJDUMP或在vmlinux镜像中使用gdb反汇编kernel函数，如上文所述。例如，这是add_range()内核函数的反汇编，我将演示这些是如何工作的。取决于编译器的操作，有些信息可能会有差异，不过它总是有帮助的。
```bash
#gdb disassemble add_range
Dump of assembler code for function add_range:
   0x8004c4d8 <+0>:    mov    r12, sp
   0x8004c4dc <+4>:    push    {r4, r5, r6, r7, r11, r12, lr, pc}
   0x8004c4e0 <+8>:    sub    r11, r12, #4
   0x8004c4e4 <+12>:    ldrd    r6, [r11, #4]
   0x8004c4e8 <+16>:    ldrd    r4, [r11, #12]
   0x8004c4ec <+20>:    cmp    r7, r5
   0x8004c4f0 <+24>:    cmpeq    r6, r4
   0x8004c4f4 <+28>:    bcs    0x8004c510 <add_range+56>
   0x8004c4f8 <+32>:    cmp    r2, r1
   0x8004c4fc <+36>:    lsllt    r3, r2, #4
   0x8004c500 <+40>:    addlt    r2, r2, #1
   0x8004c504 <+44>:    addlt    r1, r0, r3
   0x8004c508 <+48>:    strdlt    r6, [r0, r3]
   0x8004c50c <+52>:    strdlt    r4, [r1, #8]
   0x8004c510 <+56>:    mov    r0, r2
   0x8004c514 <+60>:    ldm    sp, {r4, r5, r6, r7, r11, sp, pc}
End of assembler dump.
```

## 相应的kernel c函数
```c
int add_range(struct range *range, int az, int nr_range, u64 start, u64 end)
{
        if (start >= end)
                return nr_range;
        /* Out of slots: */
        if (nr_range >= az)
                return nr_range;
        range[nr_range].start = start;
        range[nr_range].end = end;
        nr_range++;
        return nr_range;
}
```

让我们来分析前三行，这里r12=IP(Intra-Procedure-call scratch register),r11=FP(Frame pointer).FP保存函数到函数之间的参数轨迹，是函数栈的一个栈帧，可以查看基础帧布局获取更多信息。所以简单来说，SP是当前栈指针的寄存器，而FP是曾经栈的寄存器，和PC和LR的关系类似。

```bash
 0x8004c4d8 <+0>:    mov    r12, sp   /*get a copy of sp*/
 0x8004c4dc <+4>:    push    {r4, r5, r6, r7, r11, r12, lr, pc} /*Save the frame,link register,program counter and other Register on to the stack */
 0x8004c4e0 <+8>:    sub    r11, r12, #4 /*Set the new frame pointer.*/
```

接下来的2条指令，从FP指针传送了4字节和12字节到r6和r4寄存器用于函数调用，换句话说 r11+#4的值在r6中保存，r11+#12的值在r4中保存.

注意：LORD被用来保存双字指令，然而其中的内容将被加载到r7和r5寄存器。这个函数调用被用来处理64位数据，所以64位数据只能在栈中操纵。
```bash
 0x8004c4e4 <+12>:    ldrd    r6, [r11, #4]
 0x8004c4e8 <+16>:    ldrd    r4, [r11, #12]
```
注意：前四个寄存器r0-r3被用来在子函数中传值和从函数中返回值，所以**R0=range,R1=az,R2=nr_range,R3=start,R4=end.**

下一条指令可以很简单的映射到c代码。

注意：Underlying mapping somewhat different from the normal C to Assembly conversion mapping because here 64-bit value is being passed in Function call argument which is u64 start and u64 end and to deal with 64 bit data it has to be stored in register pair and can be retrived using ldrd instruction from stack using  frame pointer.

```bash
 0x8004c4ec <+20>:    cmp    r7, r5    /*first instruction compare r7 and r5 register i.e store 32 bit LSB for start & End whose value is stored in stack.       
 0x8004c4f0 <+24>:    cmpeq    r6, r4  /*This next instruction performs an comparison only if the result of above  { cmp    r7, r5 } instruction found true(i.e r7=r5). 
 0x8004c4f4 <+28>:    bcs    0x8004c510 <add_range+56>
 0x8004c4f8 <+32>:    cmp    r2, r1   /*This instruction compare values stored in resisters r2 and r1 which are passed argument values i.e nr_range and az. 

Corresponding C code is 

if (start >= end)
                return nr_range;
/* Out of slots: */
        if (nr_range >= az)
                return nr_range;
```

让我们看下面的指令：
```bash
0x8004c4fc <+36>:    lsllt    r3, r2, #4 
0x8004c500 <+40>:    addlt    r2, r2, #1
0x8004c504 <+44>:    addlt    r1, r0, r3
0x8004c508 <+48>:    strdlt    r6, [r0, r3]
0x8004c50c <+52>:    strdlt    r4, [r1, #8]

Corresponding C code is 

range[nr_range].start = start;
        range[nr_range].end = end;
        nr_range++;
```

```bash
0x8004c510 <+56>:    mov    r0, r2 /*move r2 content into r0 register which can be return back and As I said R0-R3 are also used to hold return value from function.
0x8004c514 <+60>:    ldm    sp, {r4, r5, r6, r7, r11, sp, pc} /*LDM is used to load multiple instructions and similar to  POP stack instruction.

Corresponding C code is
 
    return nr_range;
```

这里是ARM寄存器的定义，请记住这些寄存器当你映射C函数到ARM寄存器时，更多的信息可以在这里被找到。

注意：不要忘了去看[Tour of ARM Assembly](http://www.coranac.com/tonc/text/asm.htm),这会帮助理解更深入的细节，等你做完了这些，我打赌你应该可以写一些不错的ARM汇编，至少能看懂它。

参考资料：
1. [Procedure Call Standard for the ARM® Architecture](http://infocenter.arm.com/help/topic/com.arm.doc.ihi0042e/IHI0042E_aapcs.pdf)
2. [ARM Procedure Call Standard](https://www.cl.cam.ac.uk/~fms27/teaching/2001-02/arm-project/02-sort/apcs.txt)
3. [Arm Instruction set manual](http://liris.cnrs.fr/~mmrissa/lib/exe/fetch.php?media=armv7-a-r-manual.pdf)
