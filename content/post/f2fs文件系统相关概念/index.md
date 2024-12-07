---
title: F2FS 文件系统相关概念
# description: Welcome to Hugo Theme Stack
slug: concepts_on_f2fs_filesystem
date: 2024-12-07 00:00:00+0000
# image: filesystem.jpg
categories:
    - 技术分析
tags:
    - 操作系统
    - 文件系统
    - Linux内核
    - F2FS
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---
> 可参看https://github.com/RiweiPan/F2FS-NOTES/

## IPU 和 OPU

ipu，in-place-update 就地更新：在原地更新数据。传统文件系统如 ext4 都采用。

opu，out-of-place-update 异地更新：将更新后的数据写在新的地址，修改映射到新地址。

1. 分配一个新的物理地址
2. 将数据写入新的物理地址
3. 将旧的物理地址无效掉，然后等GC回收
4. 更新逻辑地址和物理地址的映射关系

异地更新更适合闪存特性。 OPU 的缺点在于（1）产生无效块，造成 GC 开销；（2）更新元数据的开销；（3）数据碎片化