---
title: 什么是trim命令
# description: Welcome to Hugo Theme Stack
slug: what_is_trim_command
date: 2024-12-08 00:00:00+0000
# image: filesystem.jpg
categories:
    - 疑难经验
tags:
    - 命令
    - 操作系统
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---

操作系统通过trim命令通知SSD哪些块不再使用，可以擦除或回收。

文件系统在删除文件时，仅将数据块标记为未使用。因此文件所在的位置只有下一次写入时才会被覆盖。但是对于SSD，数据页在重写之前必须擦除其所在块。因此存在垃圾回收和写放大。

trim命令使得SSD知道哪些数据页是无效的，在做垃圾回收时不搬移这些页。

因此，在SSD做过垃圾回收之后几乎不可能恢复数据。但是HDD中被删除的数据仍有可能恢复。