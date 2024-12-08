---
title: 使用F2FS文件系统
# description: Welcome to Hugo Theme Stack
slug: use_f2fs_filesystem
date: 2024-12-08 00:00:00+0000
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
## 挂载F2FS

文件系统可以挂载在盘上，也可以挂载在文件上。有时候为了实验会在文件上挂载一个文件系统。

### 创建一个100M的空文件：

```c
dd if=/dev/zero of=/data/local/tmp/f2fs_100MB.bin bs=1M count=100
```

其中，if是指从zero设备中取数据，取出的都是0。of是指将0写入后面路径的文件中，大小是100个1M。

### 格式化文件系统

```c
make_f2fs -f -d1 -g android -O compression -O extra_attr /data/local/tmp/f2fs_100MB.bin    
```

(参数待查)

### 获取文件系统信息

使用dump.f2fs命令获取文件系统的信息

```c
dump.f2fs -d 1 f2fs_100MB.bin
```

(参数待查)

## 未列出的参考资料

[https://mp.weixin.qq.com/s/FbGyxclb1Gesk8_apEexPQ](https://mp.weixin.qq.com/s/FbGyxclb1Gesk8_apEexPQ)