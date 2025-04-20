---
title: "Linux内核samples/bpf编译拆解与实践"
date: 2023-12-10T16:09:38+08:00
draft: true
tags: ["Linux内核"]
categories: ["内核之旅"]
Summary: "编译samples/bpf目录下的文件，调试、输出并解析"
---
# 准备工作
## 下载内核源码并编译内核
### 1、查看内核版本并下载对应的内核代码压缩包
运行uname -r命令查看内核版本
```bash
uname -r
```
看到以下结果，可以知道当前机器的内核版本是5.15.133
```bash
5.15.133.1-microsoft-standard-WSL2
```
去[内核源码网页](https://www.kernel.org/pub/linux/kernel)下载对应的内核压缩包，这里我们的大版本是5，所以在v5.x目录中找到linux-5.15.133.tar.gz压缩包并下载到机器中,并解压
```bash
wget https://mirrors.edge.kernel.org/pub/linux/kernel/v5.x/linux-5.15.133.tar.gz
tar -xzvf linux-5.15.133.tar.gz
```

### 2、根据已有配置编译内核
进入Linux内核源码路径之后，需要对内核config文件进行配置,当前机器内核编译的config文件保存在/proc/config.gz文件中，我们只需要复制一份到.config文件即可

```bash
zcat /proc/
zcat /proc/config.gz > .config
make ARCH=x86 oldconfig
make ARCH=x86 prepare
make -j4
```

### 3、编译samples/bpf目录
由于bpf字节码需要通过clang编译，所以在编译之前可以先安装clang和llvm编译器，运行以下命令
```bash
make M=samples/bpf
```
可以看到在编译过程中有一些依赖库没有安装，可能会丢失一些特性，这里整理了一份安装bpftrace时的依赖库供参考
```bash
apt-get install libbfd-dev libcap-dev libzstd-dev libperl-dev libpython2-dev  libunwind-dev lzma libnuma-dev
```
再次运行make M=samples/bpf可以看到基本编译选项都已打开
```bash
Auto-detecting system features:
...                        libbfd: [ on  ]
...        disassembler-four-args: [ on  ]
...                          zlib: [ on  ]
...                        libcap: [ on  ]
...               clang-bpf-co-re: [ on  ]
```

至此，编译samples/bpf的任务基本完成

参考链接：
[Bpf trace install.md](https://github.com/iovisor/bpftrace/blob/master/INSTALL.md)

# 实践探究
## 编译结果
samples/bpf编译完成后，我们看一下结果
```bash
ls -lt


total 78568
...
-rwxr-xr-x 1 root root 1393768 Dec 10 12:05 tracex7
-rwxr-xr-x 1 root root 1394176 Dec 10 12:05 tracex6
-rwxr-xr-x 1 root root 1394384 Dec 10 12:05 tracex5
-rwxr-xr-x 1 root root 1393848 Dec 10 12:05 tracex4
-rwxr-xr-x 1 root root 1393768 Dec 10 12:05 tracepoint
-rwxr-xr-x 1 root root 1394232 Dec 10 12:05 tracex3
-rwxr-xr-x 1 root root 1394112 Dec 10 12:05 tracex2
-rwxr-xr-x 1 root root 1394296 Dec 10 12:05 tracex1
-rwxr-xr-x 1 root root 1394000 Dec 10 12:05 sockex3
-rwxr-xr-x 1 root root 1393952 Dec 10 12:05 sockex2
-rwxr-xr-x 1 root root 1393912 Dec 10 12:05 sockex1
...
```
可以看到生成了很多可执行文件，我们可以根据这些可执行文件的源码去看每个文件的编写的样例，并去验证相应的功能，例如运行tracex3,可以获得bio的延迟

```bash
➜  bpf ./tracex3
find ./tracex3.bpf.o  heatmap of IO latency
    - many events with this latency
    - few events
|1us      |10us     |100us    |1ms      |10ms     |100ms    |1s       |10s
                                                                        # 0
                                                                        # 4
^C
```

## 编译过程
看到结果并且可以成功将bpf挂载到对应的内核event，但是我比较好奇samples/bpf的编译流程，因为不想每次想修改代码验证都要重新make一遍，比较麻烦

看到目录下的每个文件都是由.bpf.c和.c组成的，加上对内核bpf虚拟机的理解，很容易的推导出了每个trace的案例其实由两部分组成，一个是输入到内核的bpf程序字节码，另一个是在用户空间去控制和装载该bpf程序的用户程序。

```bash
cd samples/bpf
make V=1
```
通过在编译时添加V=1我们可以看到详细的编译流程，打开Makefile和查看日志可以看到编译由几个部分组成

### 1、编译内核libbpf库和bpftool工具
编译前的samples/bpf目录下只有gnu一个文件夹，在编译过后目录多出了两个文件夹，一个是bpftool文件夹源码来自于tools/bpf/bpftool，另一个是libbpf文件夹源码来自于tools/lib/bpf

生成这两个文件夹是在编译正式程序之前的准备工作，需要提供各种库和头文件，如bpf/bpf_helpers.h


### 2、导出vmlinux.h

查看Makefile文件，可以看到生成vmlinux.h的过程
```bash
...
endif
	$(Q)$(BPFTOOL) btf dump file $(VMLINUX_BTF) format c > $@
else
	$(Q)cp "$(VMLINUX_H)" $@
endif
...
```
大概就是找到vmlinux之后，使用bpftool工具将头文件导出来，具体原因还不清楚，等待进一步学习探究，可以复刻了一下，通过以下命令即可完成。
```bash
bpftool btf dump file ../../vmlinux format c > test.h
```

### 3、编译内核bpf字节码
查看Makefile文件，可以看到生成xxx.bpf.o即bpf字节码的过程
```bash
$(obj)/%.bpf.o: $(src)/%.bpf.c $(obj)/vmlinux.h $(src)/xdp_sample.bpf.h $(src)/xdp_sample_shared.h
	@echo "  CLANG-BPF " $@
	$(Q)$(CLANG) -g -O2 -target bpf -D__TARGET_ARCH_$(SRCARCH) \
		-Wno-compare-distinct-pointer-types -I$(srctree)/include \
		-I$(srctree)/samples/bpf -I$(srctree)/tools/include \
		-I$(LIBBPF_INCLUDE) $(CLANG_SYS_INCLUDES) \
		-c $(filter %.bpf.c,$^) -o $@
```
可以看到，bpf字节码的编译过程是使用clang并指定target为bpf实现的，同时指定对应的头文件路径来编译，复刻一下，可以通过以下命令完成bpf字节码的编译
```bash
clang -g -O2 -target bpf -I/usr/src/linux/linux-5.15.133/include -I/usr/src/linux/linux-5.15.133/samples/bpf -I/usr/src/linux/linux-5.15.133/tools/include -I/usr/src/linux/linux-5.15.133/samples/bpf/libbpf/include -c tracex3.bpf.c -o tracex3.bpf.o
```

### 3、编译用户程序
查看make V=1输出的日志，可以看到链接samples/bpf用户程序的过程
```bash
gcc -Wp,-MD,/usr/src/linux/linux-5.15.133/samples/bpf/.tracex4.d -Wall -O2 -Wmissing-prototypes -Wstrict-prototypes -I./usr/include -I./tools/testing/selftests/bpf/ -I/usr/src/linux/linux-5.15.133/samples/bpf/libbpf/include -I./tools/include -I./tools/perf -DHAVE_ATTR_TEST=0   -o /usr/src/linux/linux-5.15.133/samples/bpf/tracex4 /usr/src/linux/linux-5.15.133/samples/bpf/tracex4_user.o /usr/src/linux/linux-5.15.133/samples/bpf/libbpf/libbpf.a -lelf -lz -lrt
```

可以看到用户程序使用gcc编译链接的，然后同样需要指定头文件路径，依赖库文件路径以及依赖的库

同样复刻一下编译过程：
```bash
gcc -Wall -O2 -I./usr/include -I./tools/testing/selftests/bpf/ -I/usr/src/linux/linux-5.15.133/samples/bpf/libbpf/include -I./tools/include -I./tools/perf -DHAVE_ATTR_TEST=0   -o tracex3 tracex3_user.o /usr/src/linux/linux-5.15.133/samples/bpf/libbpf/libbpf.a -lelf -lz
```

## 总结
至此我们知道了每个bpf的sample都有两个部分组成，一个部分是bpf程序代码（生成bpf字节码，由clang编译），另一部分是bpf程序控制代码（用户程序，负责将bpf字节码加载并注入到内核，由gcc编译）

除此之外还需要前提条件，生成bpftool工具和bpf程序依赖的libbpf库，并且需要得到编译的vmlinux并通过bpftool将btf数据定义从vmlinux取出。

完成这些之后，我们便可以通过clang和gcc灵活编译和验证bpf程序了。



