---
title: "APT in ubuntu init"
date: 2021-06-01T21:29:38+08:00
draft: true
tags: ["Linux工具"]
categories: ["内核之旅"]
Summary: "在Ubuntu系统上，想要做些什么之前那得做好准备"
---
# 系统初始化必选：
sudo apt-get install pkg-config build-essential zsh tmux htop git fakeroot ncurses-dev xz-utils libssl-dev bc flex libelf-dev bison vim curl net-tools flex

# kernel内核开发：
sudo apt-get install linux-headers-`uname -r`

# 发生patch
sudo apt-get install cscope libncurses-dev git-email

# sparse
sudo apt-get install g++ llvm libgtk-3-dev libxml2 libsqlite3-dev

# 编译Linux内核
sudo apt-get install build-essential bison flex libelf-dev libssl-dev 

# Coccicheck
sudo apt-get install python-dev libpycaml-ocaml-dev libmenhir-ocaml-dev menhir ocaml-native-compilers camlp4-extra ocaml-findlib texlive-fonts-extra opam  automake 

