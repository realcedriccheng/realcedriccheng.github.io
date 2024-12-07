---
title: 文件的相关概念
# description: Welcome to Hugo Theme Stack
slug: concepts_on_file
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
## inode

inode 用来存储文件的元数据，包括普通文件、目录文件、块设备文件和字符设备文件等。读取某个文件之前，需要先将其 inode 读到内存中来，通过 inode 索引其数据块位置。

VFS 中的 inode 就叫 struct inode，具体文件系统的 inode 一般叫做 xxx_inode_info。

除根目录 inode 外，所有 inode 都在一个全局哈希表 inode_hashtable 中，便于查找某个文件系统特定 inode。这是为了管理内存中的 inode。如果 inode 在内存中，就通过哈希表找到，而不是从盘上再读出来。毕竟我们只知道 inode 号。

每个 inode 还包含在所属文件系统的链表中（super_block->s_inodes、inode->i_sb_list），对文件的操作通过 inode 中的 inode 操作表（i_op）和 file 操作表（i_fop）完成。

inode 操作表中提供的操作与文件的元数据有关，file 操作表中的操作与文件内容本身的读写。

## dentry

dentry 用来存储文件在内核文件系统树中的位置。目录、常规文件、符号链接、块设备文件、字符设备文件等文件系统对象都有 dentry。同样分为盘上 dentry、VFS dentry 和文件系统 dentry。

VFS dentry 通过 d_fsdata 指向文件系统 dentry（f2fs 中没有这一项），通过 d_op 指向文件系统提供的 dentry 操作表（f2fs 中也没有这个）。

dentry 和 inode 是多对一的关系，每个 dentry 只有一个 inode，由 d_inode 指向。每个 inode 可能有多个 dentry，由 i_dentry 指向 dentry 链表，例如硬链接。

dentry 之间有一对多的父子关系，d_parent 指向父 dentry，根目录的 dentry 是自己的父 dentry。d_subdirs 指向子 dentry 链表。

除了根 dentry 外，所有内存中的 dentry 加入到 dentry_hashtable 全局哈希表，便于查找给定目录下的文件。

## 孤儿节点 orphan node

orphan node 是无主的 node，orphan inode 是指一个文件不在任何目录中，不能被用户访问到。这样的文件仍然被某些进程占用，当没有进程使用时，该文件就被删除了。

## 文件的地址空间

文件的地址空间用来管理文件映射到内存的页面，将文件系统中 file 的数据与内存 page cache 或者 swap cache 相关页面绑定到一起。这样就可以将盘上实际不连续的数据以页面为单位连续呈现出来。

表示地址空间的数据结构是 address_space 结构体，表示地址空间操作表的数据机构是 address_space_operations（a_ops）。

地址空间的属主（host）是 inode。对于块设备文件是其主 inode。

文件的地址空间以页面为单位组织成基数树。

地址空间的操作一般是对具体 page 的操作。包括 writepage、readpage 等。将内存中的某个 page 落盘或者将盘上某个 page 的数据读取到内存中。

*注意，i_fop 中那些操作都是针对内存的。只有在这里才是真正的 IO。*

读取文件时

- 以文件的f_mapping为参数，通过find_get_page查找page cache
- 查找到page cache且数据是最新的，就通过copy_page_to_iter，将数据拷贝到用户空间
- 没找到，就通过page_cache_alloc分配一个新页面，并将其加入page cache和LRU链表
- 然后调用对应的readpage函数，从磁盘中读入文件数据
- 最后还是通过copy_page_to_iter，将数据拷贝到用户空间

## 文件系统中的扩展属性xattr

文件的基础属性包括 inode ID、创建时间和大小等，比较有限。扩展属性 xattr 是一种允许用户为文件添加自定义属性的方法。xattr 以键值对的方式储存在文件外部。

通过 API 使用 xattr
setfattr 设置属性，getfattr 获取属性。

xattr 的实现，参看文件系统相关概念。

# 参考资料

https://blog.csdn.net/jinking01/article/details/106490467

https://blog.csdn.net/u010039418/article/details/115773253

《文件系统技术内幕》