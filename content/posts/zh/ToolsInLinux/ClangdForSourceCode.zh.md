---
title: "使用Vscode中的clangd插件阅读linux源码"
date: 2024-07-21T00:41:38+08:00
draft: true
tags: ["Linux工具"]
categories: ["内核之旅"]
Summary: "使用Vscode中的clangd插件阅读linux源码"
---

## 背景
Linux kernel代码编译选项多，体系结构多，查看源码时很难找到某个函数的定义，这里介绍一个clang工具的拓展插件clangd，最新内核已经支持了clangd的代码检索功能，本文提供了一种方法，该方法可以将vscode打造成为一个检索Linux源码的利器。

## 步骤
### 1、使用ssh连接到服务器

### 2、安装clangd

### 3、编译内核

### 4、调用内核scripts/clang-tools/gen_compile_commands.py工具生成compile_command.json

等待内核编译好了后，查看源码根目录是否有.cmd相关文件，如果有相关文件，则进一步调用脚本文件生成json文件
```bash
./scripts/clang-tools/gen_compile_commands.py -d .
```

### 5、Vscode安装clangd插件

### 6、等待vscode索引完成，开始检索代码

