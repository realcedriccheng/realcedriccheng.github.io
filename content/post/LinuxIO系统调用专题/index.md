---
title: Linux IO 系统调用专题
# description: Welcome to Hugo Theme Stack
slug: LinuxIO
date: 2025-04-07 00:00:00+0000
# image: filesystem.jpg
categories:
    - 面试经验
    - 理论知识
tags:
    - Linux
    - IO路径
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---
https://arthurchiao.art/blog/intro-to-io-uring-zh/

• How io_uring and eBPF Will Revolutionize Programming in Linux, ScyllaDB, 2020
• An Introduction to the io_uring Asynchronous I/O Framework, Oracle, 2020

io_uring 是 2019 年 Linux 5.1 内核首次引入的高性能异步 I/O 框架，能显著加速 I/O 密集型应用的性能。
但如果你的应用已经在使用 传统 Linux AIO 了，并且使用方式恰当，那 io_uring并不会带来太大的性能提升 —— 根据原文测试（以及我们自己的复现），即便打开高级特性，也只有 5%。除非你真的需要这 5% 的额外性能，否则切换成 io_uring代价可能也挺大，因为要重写应用来适配 io_uring（或者让依赖的平台或框架去适配，总之需要改代码）。

既然性能跟传统 AIO 差不多，那为什么还称 io_uring 为革命性技术呢？

1. 它首先和最大的贡献在于：统一了 Linux 异步 I/O 框架，
    ◦ Linux AIO 只支持 direct I/O 模式的存储文件（storage file），而且主要用在数据库这一细分领域；
    ◦ io_uring 支持存储文件和网络文件（network sockets），也支持更多的异步系统调用
（accept/openat/stat/...），而非仅限于 read/write 系统调用。

2. 在设计上是真正的异步 I/O，作为对比，Linux AIO 虽然也是异步的，但仍然可能会阻塞，某些情况下的行为也无法预测；

似乎之前 Windows 在这块反而是领先的，更多参考：
    ◦ 浅析开源项目之 io_uring，“分步试存储”专栏，知乎
    ◦ Is there really no asynchronous block I/O on Linux?，stackoverflow

3. 灵活性和可扩展性非常好，甚至能基于 io_uring 重写所有系统调用，而 Linux AIO 设计时就没考虑扩展性。
eBPF 也算是异步框架（事件驱动），但与 io_uring 没有本质联系，二者属于不同子系统，并且在模型上有一个本质区别：eBPF 对用户是透明的，只需升级内核（到合适的版本），应用程序无需任何改造；

4. io_uring 提供了新的系统调用和用户空间 API，因此需要应用程序做改造。
eBPF 作为动态跟踪工具，能够更方便地排查和观测 io_uring 等模块在执行层面的具体问题。

本文介绍 Linux 异步 I/O 的发展历史，io_uring 的原理和功能，并给出了一些程序示例和性能压测结果（我们在 5.10内核做了类似测试，结论与原文差不多）。

很多人可能还没意识到，Linux 内核在过去几年已经发生了一场革命。这场革命源于两个激动人心的新接口的引入：eBPF 和 io_uring。

我们认为，二者将会完全改变应用与内核交互的方式，以及应用开发者思考和看待内核的方式。

本文介绍 io_uring（我们在 ScyllaDB 中有 io_uring 的深入使用经验），并略微提及一下 eBPF。

1 Linux I/O 系统调用演进

1.1 基于 fd 的阻塞式 I/O：read()/write()

作为大家最熟悉的读写方式，Linux 内核提供了基于文件描述符的系统调用，这些描述符指向的可能是存储文件（storage file），也可能是 network sockets：
ssize_t read(int fd,void* buf,size_t count);
ssize_t write(int fd,const void* buf,size_t count);二者称为阻塞式系统调用（blocking system calls），因为程序调用这些函数时会进入 sleep 状态，然后被调度出去（让出处理器），直到 I/O 操作完成：
• 如果数据在文件中，并且文件内容已经缓存在 page cache 中，调用会立即返回；
• 如果数据在另一台机器上，就需要通过网络（例如 TCP）获取，会阻塞一段时间；
• 如果数据在硬盘上，也会阻塞一段时间。
但很容易想到，随着存储设备越来越快，程序越来越复杂，阻塞式（blocking）已经这种最简单的方式已经不适用了。

1.2 非阻塞式 I/O：select()/poll()/epoll()
阻塞式之后，出现了一些新的、非阻塞的系统调用，例如 select()、poll() 以及更新的 epoll()。

应用程序在调用这些函数读写时不会阻塞，而是立即返回，返回的是一个已经 ready 的文件描述符列表。

但这种方式存在一个致命缺点：只支持 network sockets 和 pipes ——epoll() 甚至连 storage files 都不支持。

1.3 线程池方式
对于 storage I/O，经典的解决思路是 thread pool：主线程将 I/O 分发给 worker 线程，后者代替主线程进行阻塞式读写，主线程不会阻塞。

这种方式的问题是线程上下文切换开销可能非常大，后面性能压测会看到。

1.4 Direct I/O（数据库软件）：绕过 page cache
随后出现了更加灵活和强大的方式：数据库软件（database software）有时 并不想使用操作系统的 page cache，而是希望打开一个文件后，直接从设备读写这个文件（direct access to the device）。这种方式称为直接访问（direct access）或直接 I/O（direct I/O），

• 需要指定 O_DIRECT flag；
• 需要应用自己管理自己的缓存 —— 这正是数据库软件所希望的；
• 是 zero-copy I/O，因为应用的缓冲数据直接发送到设备，或者直接从设备读取。

1.5 异步 IO（AIO）
前面提到，随着存储设备越来越快，主线程和 worker 线性之间的上下文切换开销占比越来越高。

现在市场上的一些设备，例如 Intel Optane，延迟已经低到和上下文切换一个量级（微秒 us）。换个方式描述，更能让我们感受到这种开销：上下文每切换一次，我们就少一次 dispatch I/O 的机会。

因此，Linux 2.6 内核引入了异步 I/O（asynchronous I/O）接口，

方便起见，本文简写为 linux-aio。AIO 原理是很简单的：

• 用户通过 io_submit() 提交 I/O 请求，

• 过一会再调用 io_getevents() 来检查哪些 events 已经 ready 了。

• 使程序员能编写完全异步的代码。

近期，Linux AIO 甚至支持了epoll()：也就是说不仅能提交 storage I/O 请求，还能提交网络 I/O 请求。照这样发展下去，linux-aio似乎能成为一个王者。但由于它糟糕的演进之路，这个愿望几乎不可能实现了。

Linux AIO 确实问题缠身，

1. 只支持 O_DIRECT 文件，因此对常规的非数据库应用  （normal, non-database applications）几乎是无用的；

2. 接口在设计时并未考虑扩展性。虽然可以扩展 —— 我们也确实这么做了 —— 但每加一个东西都相当复杂；

3. 虽然从技术上说接口是非阻塞的，但实际上有  很多可能的原因都会导致它阻塞，而且引发的方式难以预料。

1.6 小结

以上可以清晰地看出 Linux I/O 的演进：

• 最开始是同步（阻塞式）系统调用；

• 然后随着实际需求和具体场景，不断加入新的异步接口，还要保持与老接口的兼容和协同工作。

另外也看到，在非阻塞式读写的问题上并没有形成统一方案：

1. Network socket 领域：添加一个异步接口，然后去轮询（poll）请求是否完成（readiness）；

2. Storage I/O 领域：只针对某一细分领域（数据库）在某一特定时期的需求，添加了一个定制版的异步接口。

这就是 Linux I/O 的演进历史 —— 只着眼当前，出现一个问题就引入一种设计，而并没有多少前瞻性 —— 直到 io_uring 的出现。

2 io_uring

io_uring 来自资深内核开发者 Jens Axboe 的想法，他在 Linux I/O stack 领域颇有研究。

从最早的 patch aio: support for IO polling可以看出，这项工作始于一个很简单的观察：随着设备越来越快，中断驱动（interrupt-driven）模式效率已经低于轮询模式（polling for completions） —— 这也是高性能领域最常见的主题之一。

• io_uring 的基本逻辑与 linux-aio 是类似的：提供两个接口，一个将I/O 请求提交到内核，一个从内核接收完成事件。

• 但随着开发深入，它逐渐变成了一个完全不同的接口：设计者开始从源头思考如何支持完全异步的操作。

2.1 与 Linux AIO 的不同

io_uring 与 linux-aio 有着本质的不同：

1. 在设计上是真正异步的（truly asynchronous）。只要设置了合适的 flag，它在系统调用上下文中就只是将请求放入队列，不会做其他任何额外的事情，保证了应用永远不会阻塞。

2. 支持任何类型的 I/O：cached files、direct-access files 甚至 blocking sockets。

由于设计上就是异步的（async-by-design nature），因此无需 poll+read/write 来处理 sockets。

 只需提交一个阻塞式读（blocking read），请求完成之后，就会出现在 completion ring。

3. 灵活、可扩展：基于 io_uring 甚至能重写（re-implement）Linux 的每个系统调用。

2.2 原理及核心数据结构：SQ/CQ/SQE/CQE

每个 io_uring 实例都有两个环形队列（ring），在内核和应用程序之间共享：

• 提交队列：submission queue (SQ)

• 完成队列：completion queue (CQ)

这两个队列：

• 都是单生产者、单消费者，size 是 2 的幂次；

• 提供无锁接口（lock-less access interface），内部使用内存屏障做同步（coordinated with memory barriers）。

使用方式：

• 请求

    ◦ 应用创建 SQ entries (SQE)，更新 SQ tail；

    ◦ 内核消费 SQE，更新 SQ head。

• 完成

    ◦ 内核为完成的一个或多个请求创建 CQ entries (CQE)，更新 CQ tail；

    ◦ 应用消费 CQE，更新 CQ head。

    ◦ 完成事件（completion events）可能以任意顺序到达，到总是与特定的 SQE 相关联的。

    ◦ 消费 CQE 过程无需切换到内核态。

2.3 带来的好处

io_uring 这种请求方式还有一个好处是：原来需要多次系统调用（读或写），现在变成批处理一次提交。

还记得 Meltdown 漏洞吗？当时我还写了一篇文章解释为什么我们的 Scylla NoSQL 数据库受影响很小：aio 已经将我们的 I/O 系统调用批处理化了。

io_uring将这种批处理能力带给了 storage I/O 系统调用之外的其他一些系统调用，包括：

• read

• write

• send

• recv

• accept

• openat

• stat

• 专用的一些系统调用，例如 fallocate

此外，io_uring 使异步 I/O 的使用场景也不再仅限于数据库应用，普通的非数据库应用也能用。这一点值得重复一遍：

虽然 io_uring 与 aio 有一些相似之处，但它的扩展性和架构是革命性的：

它将异步操作的强大能力带给了所有应用（及其开发者），而不再仅限于是数据库应用这一细分领域。

我们的 CTO Avi Kivity 在 the Core C++ 2019 event 上 有一次关于 async 的分享。

核心点包括：从延迟上来说，

1. 现代多核、多 CPU 设备，其内部本身就是一个基础网络；

2. CPU 之间是另一个网络；

3. CPU 和磁盘 I/O 之间又是一个网络。

因此网络编程采用异步是明智的，而现在开发自己的应用也应该考虑异步。

这从根本上改变了 Linux 应用的设计方式：

• 之前都是一段顺序代码流，需要系统调用时才执行系统调用，

• 现在需要思考一个文件是否 ready，因而自然地引入 event-loop，不断通过共享 buffer 提交请求和接收结果。

2.4 三种工作模式

io_uring 实例可工作在三种模式：

1. 中断驱动模式（interrupt driven）

默认模式。可通过 io_uring_enter() 提交 I/O 请求，然后直接检查 CQ 状态判断是否完成。

2. 轮询模式（polled）

Busy-waiting for an I/O completion，而不是通过异步 IRQ（Interrupt Request）接收通知。

这种模式需要文件系统（如果有）和块设备（block device）支持轮询功能。

 相比中断驱动方式，这种方式延迟更低（连系统调用都省了）， 但可能会消耗更多 CPU 资源。

目前，只有指定了 O_DIRECT flag 打开的文件描述符，才能使用这种模式。当一个读或写请求提交给轮询上下文（polled context）之后，应用（application）必须调用 io_uring_enter() 来轮询 CQ 队列，判断请求是否已经完成。

对一个 io_uring 实例来说，不支持混合使用轮询和非轮询模式。

3. 内核轮询模式（kernel polled）

这种模式中，会 创建一个内核线程（kernel thread）来执行 SQ 的轮询工作。

使用这种模式的 io_uring 实例， 应用无需切到到内核态 就能触发（issue）I/O 操作。

通过 SQ 来提交 SQE，以及监控 CQ 的完成状态，应用无需任何系统调用，就能提交和收割 I/O（submit and reap I/Os）。

如果内核线程的空闲时间超过了用户的配置值，它会通知应用，然后进入 idle 状态。

这种情况下，应用必须调用 io_uring_enter() 来唤醒内核线程。如果 I/O 一直很繁忙，内核线性是不会 sleep 的。

后面略