---
title: 某手机公司面试题：实现 read/write 接口
# description: Welcome to Hugo Theme Stack
slug: moushoujigongsimianshitirw
date: 2025-02-26 00:00:00+0000
# image: filesystem.jpg
categories:
    - 面经
tags:
    - C/C++
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---
（AI生成）

# 题目

有 2 个 api aligned read 和 write，调用他们实现其中的 read 和 write 调用接口

请使用C语言帮我实现程序，并给出思路和代码的详细注释：

aligned size = 4kb

int aligned read(int fd, void *buf, u64 pos u64 len)

int read(int fd, void *buf，u64 pos, u64 len)

int aligned write(int fd, void *buf, u64 pos, u64 len)

int write(int fd, void *buf，u64 pos, u64 len)

# 思路

为了实现 `read` 和 `write` 接口，我们需要调用 `aligned_read` 和 `aligned_write` API。这两个 API 要求缓冲区和位置都是对齐的（aligned size = 4KB）。因此，我们需要确保在调用 `aligned_read` 和 `aligned_write` 时，缓冲区和位置都是 4KB 对齐的。

对齐检查:

确保传递给 `aligned_read` 和 `aligned_write` 的缓冲区和位置都是 4KB 对齐的。

分段处理:

如果读取或写入的范围不对齐，我们需要分段处理对齐的部分和非对齐的部分。

缓冲区管理:

使用临时缓冲区来处理非对齐的部分。

# 代码实现

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <stdint.h>

#define ALIGN_SIZE 4096

// 模拟 aligned_read 和 aligned_write API
int aligned_read(int fd, void *buf, uint64_t pos, uint64_t len) {
    // 模拟读取操作
    lseek(fd, pos, SEEK_SET);
    return read(fd, buf, len);
}

int aligned_write(int fd, void *buf, uint64_t pos, uint64_t len) {
    // 模拟写入操作
    lseek(fd, pos, SEEK_SET);
    return write(fd, buf, len);
}

// 实现 read 接口
int read(int fd, void *buf, uint64_t pos, uint64_t len) {
    uint64_t aligned_pos = pos & ~(ALIGN_SIZE - 1);
    uint64_t end_pos = pos + len;
    uint64_t aligned_end_pos = (end_pos + ALIGN_SIZE - 1) & ~(ALIGN_SIZE - 1);

    uint64_t aligned_len = aligned_end_pos - aligned_pos;
    void *aligned_buf = aligned_alloc(ALIGN_SIZE, aligned_len);
    if (!aligned_buf) {
        perror("aligned_alloc");
        return -1;
    }

    // 读取对齐的范围
    int ret = aligned_read(fd, aligned_buf, aligned_pos, aligned_len);
    if (ret < 0) {
        free(aligned_buf);
        return ret;
    }

    // 复制所需的数据到目标缓冲区
    memcpy(buf, (char *)aligned_buf + (pos - aligned_pos), len);
    free(aligned_buf);

    return len;
}

// 实现 write 接口
int write(int fd, void *buf, uint64_t pos, uint64_t len) {
    uint64_t aligned_pos = pos & ~(ALIGN_SIZE - 1);
    uint64_t end_pos = pos + len;
    uint64_t aligned_end_pos = (end_pos + ALIGN_SIZE - 1) & ~(ALIGN_SIZE - 1);

    uint64_t aligned_len = aligned_end_pos - aligned_pos;
    void *aligned_buf = aligned_alloc(ALIGN_SIZE, aligned_len);
    if (!aligned_buf) {
        perror("aligned_alloc");
        return -1;
    }

    // 读取对齐的范围
    int ret = aligned_read(fd, aligned_buf, aligned_pos, aligned_len);
    if (ret < 0) {
        free(aligned_buf);
        return ret;
    }

    // 复制新的数据到对齐缓冲区
    memcpy((char *)aligned_buf + (pos - aligned_pos), buf, len);

    // 写入对齐的范围
    ret = aligned_write(fd, aligned_buf, aligned_pos, aligned_len);
    free(aligned_buf);

    return ret < 0 ? ret : len;
}

// 测试函数
int main() {
    // 示例文件操作
    int fd = open("testfile.txt", O_RDWR | O_CREAT, 0644);
    if (fd < 0) {
        perror("open");
        return 1;
    }

    char write_buf[] = "Hello, this is a test for aligned write and read.";
    char read_buf[sizeof(write_buf)];

    // 写入数据
    if (write(fd, write_buf, 0, sizeof(write_buf)) < 0) {
        perror("write");
        close(fd);
        return 1;
    }

    // 读取数据
    if (read(fd, read_buf, 0, sizeof(read_buf)) < 0) {
        perror("read");
        close(fd);
        return 1;
    }

    printf("读取的数据: %s\n", read_buf);

    close(fd);
    return 0;
}
```

# 详细注释

1. aligned_read 和 aligned_write 模拟:
- 这两个函数模拟了对齐的读取和写入操作，实际实现中可能需要根据具体需求进行调整。
2. read 接口:
- 对齐计算:
- 计算对齐的起始位置 `aligned_pos` 和结束位置 `aligned_end_pos`。
- 分配对齐缓冲区:
- 使用 `aligned_alloc` 分配对齐的缓冲区。
- 读取对齐的范围:
- 调用 `aligned_read` 读取对齐范围的数据。
- 复制数据:
- 从对齐缓冲区复制所需的数据到目标缓冲区。
- 释放缓冲区:
- 释放分配的对齐缓冲区。
3. write 接口:
- 对齐计算:
- 计算对齐的起始位置 `aligned_pos` 和结束位置 `aligned_end_pos`。
- 分配对齐缓冲区:
- 使用 `aligned_alloc` 分配对齐的缓冲区。
- 读取对齐的范围:
- 调用 `aligned_read` 读取对齐范围的数据。
- 复制数据:
- 将新的数据复制到对齐缓冲区的相应位置。
- 写入对齐的范围:
- 调用 `aligned_write` 写入对齐范围的数据。
- 释放缓冲区:
- 释放分配的对齐缓冲区。

通过上述方法，我们可以确保在调用 `aligned_read` 和 `aligned_write` 时，缓冲区和位置都是 4KB 对齐的，从而实现高效的读取和写入操作。

# 追加问题

如果系统内存是有限的，读和写可以做哪些优化？

在内存有限的情况下，我们可以对读和写操作进行一些优化，以减少内存使用和提高性能。以下是一些优化策略：

### 1. 分块处理

对于大文件的读写操作，可以将文件分成多个小块进行处理，每次只处理一个小块，从而减少内存消耗。

### 2. 避免不必要的内存分配

尽量避免频繁的内存分配和释放操作，可以复用已经分配的内存缓冲区。

### 3. 使用内存映射文件（mmap）

内存映射文件可以将文件的一部分映射到内存中，从而减少内存拷贝的开销，提高读写性能。

### 优化后的代码实现

以下是优化后的代码实现，使用分块处理来减少内存消耗：

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <fcntl.h>
#include <stdint.h>
#include <errno.h>

#define ALIGN_SIZE 4096

// 模拟 aligned_read 和 aligned_write API
int aligned_read(int fd, void *buf, uint64_t pos, uint64_t len) {
    // 模拟读取操作
    lseek(fd, pos, SEEK_SET);
    return read(fd, buf, len);
}

int aligned_write(int fd, void *buf, uint64_t pos, uint64_t len) {
    // 模拟写入操作
    lseek(fd, pos, SEEK_SET);
    return write(fd, buf, len);
}

// 实现 read 接口
int read(int fd, void *buf, uint64_t pos, uint64_t len) {
    uint64_t aligned_pos = pos & ~(ALIGN_SIZE - 1);
    uint64_t end_pos = pos + len;
    uint64_t aligned_end_pos = (end_pos + ALIGN_SIZE - 1) & ~(ALIGN_SIZE - 1);

    uint64_t aligned_len = aligned_end_pos - aligned_pos;
    void *aligned_buf = aligned_alloc(ALIGN_SIZE, ALIGN_SIZE);
    if (!aligned_buf) {
        perror("aligned_alloc");
        return -1;
    }

    uint64_t bytes_read = 0;
    while (bytes_read < len) {
        uint64_t chunk_pos = aligned_pos + bytes_read;
        uint64_t chunk_len = ALIGN_SIZE;

        if (chunk_pos + chunk_len > end_pos) {
            chunk_len = end_pos - chunk_pos;
        }

        // 读取对齐的范围
        int ret = aligned_read(fd, aligned_buf, chunk_pos, chunk_len);
        if (ret < 0) {
            free(aligned_buf);
            return ret;
        }

        // 复制所需的数据到目标缓冲区
        uint64_t copy_start = (chunk_pos < pos) ? (pos - chunk_pos) : 0;
        uint64_t copy_len = (chunk_len - copy_start > len - bytes_read) ? (len - bytes_read) : (chunk_len - copy_start);
        memcpy((char *)buf + bytes_read, (char *)aligned_buf + copy_start, copy_len);

        bytes_read += copy_len;
    }

    free(aligned_buf);
    return bytes_read;
}

// 实现 write 接口
int write(int fd, void *buf, uint64_t pos, uint64_t len) {
    uint64_t aligned_pos = pos & ~(ALIGN_SIZE - 1);
    uint64_t end_pos = pos + len;
    uint64_t aligned_end_pos = (end_pos + ALIGN_SIZE - 1) & ~(ALIGN_SIZE - 1);

    uint64_t aligned_len = aligned_end_pos - aligned_pos;
    void *aligned_buf = aligned_alloc(ALIGN_SIZE, ALIGN_SIZE);
    if (!aligned_buf) {
        perror("aligned_alloc");
        return -1;
    }

    uint64_t bytes_written = 0;
    while (bytes_written < len) {
        uint64_t chunk_pos = aligned_pos + bytes_written;
        uint64_t chunk_len = ALIGN_SIZE;

        if (chunk_pos + chunk_len > end_pos) {
            chunk_len = end_pos - chunk_pos;
        }

        // 读取对齐的范围
        int ret = aligned_read(fd, aligned_buf, chunk_pos, chunk_len);
        if (ret < 0) {
            free(aligned_buf);
            return ret;
        }

        // 复制新的数据到对齐缓冲区
        uint64_t copy_start = (chunk_pos < pos) ? (pos - chunk_pos) : 0;
        uint64_t copy_len = (chunk_len - copy_start > len - bytes_written) ? (len - bytes_written) : (chunk_len - copy_start);
        memcpy((char *)aligned_buf + copy_start, (char *)buf + bytes_written, copy_len);

        // 写入对齐的范围
        ret = aligned_write(fd, aligned_buf, chunk_pos, chunk_len);
        if (ret < 0) {
            free(aligned_buf);
            return ret;
        }

        bytes_written += copy_len;
    }

    free(aligned_buf);
    return bytes_written;
}

// 测试函数
int main() {
    // 示例文件操作
    int fd = open("testfile.txt", O_RDWR | O_CREAT, 0644);
    if (fd < 0) {
        perror("open");
        return 1;
    }

    char write_buf[] = "Hello, this is a test for aligned write and read.";
    char read_buf[sizeof(write_buf)];

    // 写入数据
    if (write(fd, write_buf, 0, sizeof(write_buf)) < 0) {
        perror("write");
        close(fd);
        return 1;
    }

    // 读取数据
    if (read(fd, read_buf, 0, sizeof(read_buf)) < 0) {
        perror("read");
        close(fd);
        return 1;
    }

    printf("读取的数据: %s\n", read_buf);

    close(fd);
    return 0;
}
```

### 详细注释

1. aligned_read 和 aligned_write 模拟:
- 这两个函数模拟了对齐的读取和写入操作，实际实现中可能需要根据具体需求进行调整。
2. read 接口:
- 对齐计算:
- 计算对齐的起始位置 `aligned_pos` 和结束位置 `aligned_end_pos`。
- 分配对齐缓冲区:
- 使用 `aligned_alloc` 分配对齐的缓冲区，每次只分配 4KB。
- 分块读取:
- 使用 while 循环分块读取数据，每次读取 4KB。
- 调用 `aligned_read` 读取对齐范围的数据。
- 复制所需的数据到目标缓冲区。
- 释放缓冲区:
- 释放分配的对齐缓冲区。
3. write 接口:
- 对齐计算:
- 计算对齐的起始位置 `aligned_pos` 和结束位置 `aligned_end_pos`。
- 分配对齐缓冲区:
- 使用 `aligned_alloc` 分配对齐的缓冲区，每次只分配 4KB。
- 分块写入:
- 使用 while 循环分块写入数据，每次写入 4KB。
- 调用 `aligned_read` 读取对齐范围的数据。
- 将新的数据复制到对齐缓冲区的相应位置。
- 调用 `aligned_write` 写入对齐范围的数据。
- 释放缓冲区:
- 释放分配的对齐缓冲区。

通过这些优化，我们可以在内存有限的情况下高效地进行读取和写入操作。分块处理减少了内存消耗，避免了频繁的内存分配和释放操作，从而提高了性能。
