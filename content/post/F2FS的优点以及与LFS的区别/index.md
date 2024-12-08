---
title: F2FS的优点以及与LFS的区别
# description: Welcome to Hugo Theme Stack
slug: f2fs_lfs
date: 2024-12-08 00:00:00+0000
# image: filesystem.jpg
categories:
    - 技术分析
tags:
    - 操作系统
    - 文件系统
    - Linux内核
    - F2FS
    - LFS
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---
## 什么是LFS

LFS即日志结构文件系统（log-structured file system）。日志结构文件系统是一种只允许顺序写的文件系统。原始的LFS叫做Sprite(精灵) LFS，是 Sprite 网络操作系统的一部分。

## 为什么要将文件系统设计成日志结构的

LFS的基本假设是IO 瓶颈在写不在读，因为文件在内存有 cache。

在写入许多小文件时，将许多同步小写转化成一个大的异步写，从而充分利用磁盘带宽。

## 日志结构文件系统和日志型文件系统的区别

日志型文件系统的 log 仅用作临时存储，在崩溃恢复时使用

日志结构文件系统将 log 作为主要存储区域，并且磁盘上没有其他的结构（这是原始的 LFS）

## 怎样实现LFS

### 仅允许顺序写

将文件的改动暂存在 file cache 中，并向磁盘一次将所有的数据顺序写到 log 中（包括数据及元数据）。

### 支持随机读

每一个文件有对应的 inode，inode 包含访问控制等信息以及指向起始 10 个数据块地址的指针、指向其他数据块地址或者其他 indirect block 的 indirect block。

### 空闲空间的管理

- 将磁盘划分为一系列固定大小的 segment（512KB，这样使得找到一个 segment 不会比遍历 segment 本身更慢）
- segment 中的有效数据搬移之后才能重用（垃圾回收）
- 有些 segment 中存放寿命较长的数据，可以在分配空间的时候跳过，以免重复搬移（冷热数据分离）

### 垃圾回收

将一些 segment 读入内存，识别有效数据，并将有效数据写回干净的 segment

每个 segment 都有一个或多个 segment summary block，包含一个块属于哪个文件（ino）以及 index（为了 GC 修改映射关系）。用于识别有效数据（trivial： 检查文件 index 处的指针是否指向这个块；sprite lfs：检查版本号）

在Sprite LFS中写几十个 seg 就清理。

### 崩溃一致性

LFS 采用 checkpoints 和前滚恢复保证崩溃一致性。

崩溃恢复快，只需扫描最近的 log。

（待续）

## LFS的问题

### wandering tree 问题/滚雪球式更新

在 LFS 中，修改一个文件的数据块会导致其位置发生变化，即追加到尾部。这就导致指向该数据块的直接指针需要修改。然而修改其指针会导致指向直接指针的间接指针也需要修改。因此 inode、inode map 和 cp block 都需要递归修改。

### 清理开销

由于 LFS 的顺序写和异地更新特性，更新一个块后原来的块就作废了。这样导致盘上存在大量作废的垃圾块，需要做垃圾回收。垃圾回收的开销需要对用户隐藏，并且移动的数据量应该尽可能少，移动应该尽可能快。

LFS 中的垃圾回收严重影响性能，缩短 SSD 寿命（写放大）

_SFS: Random write considered harmful in solid state drives FAST 12_

## F2FS的优点以及与LFS的区别

### 日志结构文件系统的固有优点

f2fs 采用顺序写，因此具有适合闪存介质特性的特点。

- 闪存介质只支持异地更新，不支持就地更新。
- 随机写导致闪存内部碎片化。

### 解决了wandering tree问题

在 LFS 中，修改一个文件的数据块会导致其位置发生变化，即追加到尾部。这就导致指向该数据块的直接指针需要修改。然而修改其指针会导致指向直接指针的间接指针也需要修改。因此 inode、inode map 和 cp block 都需要递归修改。

在F2FS中，增加了一个随机写的元数据区域。其中，引入 NAT 表记录 node 位置，切断递归更新。

- 更新文件数据块->更新 dnode 内容->更新 NAT 表中 dnode位置->结束
### 解决了LFS的高GC开销问题

#### 采用multi-head logging实现冷热数据分离

F2FS中存在6个log，即{Hot, Warm, Cold}* {Node, Data}。

LFS中只有一个全局的大log，而F2FS中通过将空间划分为6个log实现了冷热数据分离。冷热数据分离就是为了减少GC开销。

#### 自适应切换写入方式

当空间利用率不高时，F2FS采用append方式顺序写入。

当空间利用率太高时，为了找到干净的segment需要频繁GC。这时F2FS可以采用threaded logging方式写入数据。

threaded logging是指直接在脏segment里面利用内碎片接着写，不用提前清理。

#### GC单位与FTL操作单元对齐

采用Section作为GC单位，与FTL操作单元对齐。

在ZNS SSD上，Section就和Zone对齐，因此只需要做一次GC。

## 未列出的参考资料

[https://github.com/RiweiPan/F2FS-NOTES/blob/master/F2FS-Layout/总体结构.md](https://github.com/RiweiPan/F2FS-NOTES/blob/master/F2FS-Layout/%E6%80%BB%E4%BD%93%E7%BB%93%E6%9E%84.md)

The design and implementation of a log-structured file system