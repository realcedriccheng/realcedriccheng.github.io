---
title: 文件系统相关概念
# description: Welcome to Hugo Theme Stack
slug: concepts_on_filesystem
date: 2024-12-07 00:00:00+0000
image: filesystem.jpg
categories:
    - 技术分析
tags:
    - 操作系统
    - 文件系统
    - Linux内核
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---
> Cover image from [Unsplash](https://unsplash.com/photos/assorted-color-folder-lot-o6GEPQXnqMY)

# 文件系统相关概念

## 硬盘、文件系统和 VFS 上的数据结构

很多数据结构都有三个版本，即持久化到盘上的版本、内存中文件系统的版本和内存中 VFS 使用的版本。例如超级块就有盘上的超级块、VFS 使用的 super_block 和文件系统使用的 xxx_sb_info。 这是因为 Linux 使用 VFS 向应用提供统一的文件系统操作接口，而每个文件系统具体的实现不一样。因此应用通过 VFS 的超级块来访问 xxx_sb_info 中的文件系统元数据和调用具体文件系统的操作函数。 

但是上面的结构都是在内存中的，掉电后就没有了。而文件系统存在的意义就是管理持久化的数据。因此还需要将整个文件系统的元数据也保存在盘上。这就是盘上的超级块。这个超级块保存在盘上特定的区域，这样一上电就能读出来。

### 文件系统类型

要使用某种文件系统，必须先将其注册到 VFS 核心。注册的内容是文件系统类型 file_system_type。注册之后才能编译、装载。不再使用的文件系统类型应从 VFS 核心注销。

注册文件系统就是将该文件系统类型插入一个 file_systems 单链表中。目的是向 VFS 提供 get_sb 和 kill_sb 回调函数，从而装载或卸载该类型的文件系统。

### 超级块
超级块用来保存整个文件系统的元数据。

超级块有盘上的和内存中的。内存中的超级块有 VFS 使用的共有的超级块和文件系统特有的超级块。前者叫做 super_block，后者一般叫做 xxx_sb_info。

VFS 超级块可以指向 fs 超级块（s_fs_info）以及 fs 提供的超级块操作表(s_op)。file_system_type 结构体将该类型文件系统实例对应的超级块实例放在 fs_supers 链表中，并通过 VFS 超级块的 s_instances 成员访问该链表。所有的 VFS 超级块对象还存在于另一个循环双链表中，通过 s_list 域链接。

对文件系统的操作通过 VFS 超级块调用具体文件系统提供的操作表（s_op）中的回调函数完成。文件系统提供的操作一般有
- 分配 inode：alloc_inode。文件系统可能有自己的 inode 结构 xxx_inode_info，也要一起分配。
- 销毁 inode：destroy_inode。若有则同时销毁 xxx_inode_info。
- 置脏 inode：dirty_inode。标记修改过的 inode ，便于写回操作。
- 写回 inode：write_inode。将磁盘上的 inode 读出来，用内存中的 inode 去更新它。然后往往不会执行 IO 写回，而是将更新过的 inode 置脏。
- 在删除 inode 之前做善后工作：drop_inode
- 删除 inode：delete_inode
- 释放超级块：put_super
- 将超级块写到磁盘上：write_super
- 做 cp：sync_fs
- 锁住和解锁文件系统：freeze_fs 和 unfreeze_fs
- 省略

### inode
inode 用来存储文件系统对象的元数据，包括普通文件、目录文件、块设备文件和字符设备文件等。读取某个文件之前，需要先将其 inode 读到内存中来，通过 inode 索引其数据块位置。

VFS 中的 inode 就叫 struct inode，具体文件系统的 inode 一般叫做 xxx_inode_info。

除根目录 inode 外，所有 inode 都在一个全局哈希表 inode_hashtable 中，便于查找某个文件系统特定 inode。这是为了管理内存中的 inode。如果 inode 在内存中，就通过哈希表找到，而不是从盘上再读出来。毕竟我们只知道 inode 号。

每个 inode 还包含在所属文件系统的链表中（super_block->s_inodes、inode->i_sb_list），对文件的操作通过 inode 中的 inode 操作表（i_op）和 file 操作表（i_fop）完成。

inode 操作表中提供的操作与文件的元数据有关，file 操作表中的操作与文件内容本身的读写。

### dentry

dentry 用来存储文件在内核文件系统树中的位置。目录、常规文件、符号链接、块设备文件、字符设备文件等文件系统对象都有 dentry。同样分为盘上 dentry、VFS dentry 和文件系统 dentry。

VFS dentry 通过 d_fsdata 指向文件系统 dentry（f2fs 中没有这一项），通过 d_op 指向文件系统提供的 dentry 操作表（f2fs 中也没有这个）。

dentry 和 inode 是多对一的关系，每个 dentry 只有一个 inode，由 d_inode 指向。每个 inode 可能有多个 dentry，由 i_dentry 指向 dentry 链表，例如硬链接。

dentry 之间有一对多的父子关系，d_parent 指向父 dentry，根目录的 dentry 是自己的父 dentry。d_subdirs 指向子 dentry 链表。

除了根 dentry 外，所有内存中的 dentry 加入到 dentry_hashtable 全局哈希表，便于查找给定目录下的文件。

### vfsmount
vfsmount 反映了一个已经装载的文件系统实例，将文件系统连接到全局文件系统树。

vfsmount 也有父子关系，例如 linux 的根文件系统是 ext4，将 f2fs 装载到某个目录下，则 ext4 是 f2fs 的父文件系统。

除了根 vfsmount 外，所有的 vfsmount 也加入 mount_hashtable 全局哈希表，为了查找装载到特定装载点的文件系统。

在 linux 内核中一个文件的位置需要<vfsmount, dentry>二元组来确定，先确定挂载点，再从挂载点的相对位置找到文件。

## xattr 在文件系统中的实现

xattr参看文件相关概念。

### NTFS
在 Windows 的 NTFS 文件系统中没有扩展属性的概念，而是有 ADS（Alternate Data Stream）的概念。

### Ext2
在 Linux 中，以 Ext2 为例。xattr 的内容存储在一个单独的逻辑块中，由描述头、扩展属性项和值组成。 534520072.jpg entry 存储了键和值的偏移量等信息。键向下生长、值向上生长。但是由于键和值的长度是可变的，因此 entry 和 value 的长度也是可变的，必须通过遍历 entry 才能找到键，通过 offs 才能找到 value。

### F2FS
为了搞清楚 F2FS 中 xattr 的实现方法，必须解答三个问题：xattr 怎样存储，怎样设置，怎样查询。

#### 数据结构
f2fs 中也存在 xattr 的 header 和 entry。

``` C
struct f2fs_xattr_header {
    __le32  h_magic;        /* magic number for identification */
    __le32  h_refcount;     /* reference count */
    __u32   h_reserved[4];  /* zero right now */
};

struct f2fs_xattr_entry {
    __u8    e_name_index;
    __u8    e_name_len;
    __le16  e_value_size;   /* size of attribute value */
    char    e_name[];      /* attribute name */
};
```
#### 接口
xattr 是 linux 的文件系统中广泛支持的功能，VFS 通过一系列函数接口调用具体文件系统的实现。f2fs/xattr.c 中的f2fs_xattr_user_handler、f2fs_xattr_trusted_handler、f2fs_xattr_advise_handler 和f2fs_xattr_security_handler 是 f2fs 中针对 xattr 的接口。

## 装载文件系统
装载子文件系统时，装载点目录不一定为空。装载后原来的子目录和文件都无法访问，只有卸载子文件系统后才能重新访问。

- 检查装载目录路径有效性
- 构造子文件系统装载实例
- 使用现有的或创建新的超级块实例，关联到 vfsmount 对象
- 填充超级块 fill_super
- 将该装载实例挂到父文件系统的装载点，添加到内核的文件系统树，实际就是一些链表操作和填充变量操作
