---
title: Linux中的数据完整性（Data Integrity）
# description: Welcome to Hugo Theme Stack
slug: data_integrity_in_linux
date: 2024-12-07 00:00:00+0000
# image: filesystem.jpg
categories:
    - 技术分析
tags:
    - 操作系统
    - 文件系统
    - Linux内核
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---
# Linux中的数据完整性（Data Integrity）

文件系统在检验数据完整性的时候，总是在读取数据的时候才会计算元数据的校验和（checksum），以查验数据是否有损坏。然而此时可能数据刚刚写入，也可能数据已经写入几个月甚至更久了。如果数据刚写入不久，也许还能找到原始数据重新写入。如果间隔太久了，就只能丢失了（或采用纠删码技术等）。

为了解决这一问题，SCSI协议中增加了完整性元数据（integrity metadata, IMD)，也叫保护信息(Protective Information, PI)。也就是说在写入数据的时候，向每个扇区附加8个字节的保护信息。

注意：在这里，扇区（sector）、块（block）、页（page）可以理解成同样的概念。为了兼容以前HDD时代的叫法，虽然SSD中没有扇区的概念，但我们还是把硬盘中的最小单位叫做扇区。也就是说，扇区是硬盘中的概念，在数据从硬盘读到内存的过程中，会经过文件系统。而在文件系统中，数据的最小处理单位是块。当数据读到内存中以后，便以页的形式表现，并由内存管理系统中的段、页管理系统等管理。而SSD的扇区大小通常是4KB，文件系统中的块、内存中的页通常也是4KB。总而言之，这三个概念只是同一段数据在不同载体上的不同名称。

附加保护信息以后，IO链路中的每个节点，如文件系统，SCSI控制器等收到上层发来的文件数据以后，都可以计算校验码检测数据是否完整。

在实现中会遇到一些问题，一个是传统的块大小都是4KB，如果加上8B变成4104B，不方便操作系统处理。另一个是软件方法计算CRC校验码的开销很大。

针对第一个问题，可以采用聚散列表（scatter-gather list）的方式管理保护信息。控制器将在写时交叉（interleave）缓冲区，在读时分割（split）缓冲区。这意味着Linux可以在不改变页缓存（page cache）的情况下将数据缓冲区DMA到主机内存。这个具体的实现要之后看一下代码才知道。

针对第二个问题，可以在操作系统层面采用开销较小的校验和，如IP校验和等。

数据和完整性元数据缓冲区的分离，以及校验和的选择被称为数据完整性扩展（Data Integrity Extensions）

附：Linux关于数据完整性的文档

https://www.kernel.org/doc/Documentation/block/data-integrity.rst
