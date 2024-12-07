---
title: 配置riscv64-unknown-elf-gdb环境
# description: Welcome to Hugo Theme Stack
slug: riscv64-unknown-elf-gdb
date: 2024-12-07 00:00:00+0000
# image: filesystem.jpg
categories:
    - 疑难经验
tags:
    - 虚拟机
    - gdb
    - riscv
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---

由于实验用到的汇编代码是riscv的，不能直接用gdb调试，而要用riscv64-unknown-elf-gdb。这个版本的gdb属于riscv工具链。

要安装riscv工具链，可以下载源代码自己编译，也可以下载预编译的二进制文件。这里我从[ttps://mirror.iscas.ac.cn/riscv-toolchains/release/riscv-collab/riscv-gnu-toolchain/LatestRelease/](https://mirror.iscas.ac.cn/riscv-toolchains/release/riscv-collab/riscv-gnu-toolchain/LatestRelease/)或者[https://github.com/riscv-collab/riscv-gnu-toolchain/releases](https://github.com/riscv-collab/riscv-gnu-toolchain/releases)选择适合的版本下载，我下载的是riscv64-elf-ubuntu-18.04-nightly-2022.11.12-nightly.tar.gz。

在ubuntu中解压后，将riscv目录复制到要安装的地点，再将安装目录下的riscv/bin添加到环境变量。查看版本证明安装成功。

```
riscv64-unknown-elf-gdb --version
```


注意：

从[https://static.dev.sifive.com/dev-tools/freedom-tools/v2020.08/riscv64-unknown-elf-gcc-10.1.0-2020.08.2-x86_64-linux-ubuntu14.tar.gz](https://static.dev.sifive.com/dev-tools/freedom-tools/v2020.08/riscv64-unknown-elf-gcc-10.1.0-2020.08.2-x86_64-linux-ubuntu14.tar.gz)下载的预编译工具不好，没有TUI界面。

要把riscv/bin目录添加到环境变量，而不是riscv

其实解压出来也就是安装了，只是要把用到的二进制文件添加到PATH以便使用
