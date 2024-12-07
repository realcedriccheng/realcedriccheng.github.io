---
title: linux中查看内核信息总是报fd0错
# description: Welcome to Hugo Theme Stack
slug: fd0
date: 2024-12-07 00:00:00+0000
# image: filesystem.jpg
categories:
    - 疑难经验
tags:
    - 虚拟机
    - linux内核
    - 报错
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---

![](1.png)

开机以后查看内核信息，总是报fd0相关的错误。这是因为系统启动的时候加载了软盘驱动，但是没有软盘。所以fd0是用不了的。可以忽略这些信息，也可以禁用软盘。