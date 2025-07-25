---
title: 面经总结
# description: Welcome to Hugo Theme Stack
slug: mianjingzongjie
date: 2025-04-07 00:00:00+0000
# image: filesystem.jpg
categories:
    - 面试经验
    - 理论知识
tags:
    - 操作系统
    - 网络
    - 数据库
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---
## 操作系统
### 比较管道，共享内存和消息队列

管道和共享内存是两种进程间通信方式。管道是通过在内核空间开辟一块缓冲区，写进程向缓冲区写入数据而读进程从缓冲区读出数据实现通信的，共享内存是开辟一块内核空间，让两个不同进程各自映射到用户空间实现进程通信的。

管道是单向通信的，而共享内存是双向通信的。

管道只适合少量数据传输，因为缓冲区较小。共享内存分配的空间较大，因此适合较多数据传输。

管道的拷贝开销大，需要进程拷贝将待写的数据拷贝到内核管道，另一个进程从内核管道中拷贝出来。而共享内存通过映射的方式避免陷入内核态，因此只需要由写进程将写入共享内存区，读进程不需要复制就可以从共享内存区读出。

共享内存需要锁机制避免数据竞争。

消息队列是不同进程间约定好消息体格式，发送方将消息体写入内核的消息链表中就返回。接收方将消息体复制出来。由于管道是阻塞的，消息队列是非阻塞的，所以消息队列传输效率更高，适合频繁传输。

### mmap文件有哪些刷盘方式

mmap将文件映射到用户的文件映射区，当进程修改文件映射区内容后会产生脏页。一方面，内核会有线程定期写回脏页。另一方面可以主动调用msync刷回指定虚拟内存区域的脏页（可以是同步等待的，也可以是异步即只是通知内核刷盘，而不阻塞的），也可以调用fsync以文件为单位刷回文件所有脏数据和元数据。最后，使用munmap解除映射时内核会将未同步的脏页刷回。

### mmap的文件会马上放到物理内存吗，什么时候会放到物理内存

mmap仅分配虚拟空间，但是不会马上将文件复制进内存。只有访问虚拟地址时触发缺页中断才会将文件读入页缓存中，并将虚拟地址映射到页缓存的物理地址上。

如果文件已经在page cache中，因为缺失PTE表项，所以mmap仍需要触发一次page fault。在异常处理流程中先检查page cache然后绑定相应物理地址。

### open，read，write，unlink，rmdir，truncate函数执行过程

要打开或者创建一个文件，可以使用open系统调用。open系统调用有3个参数。第一个是文件的路径和文件名，第二个是文件的访问模式和操作，第三个是在创建文件时，定义文件的访问权限。

首先，内核需要解析路径名。解析路径名的方式就是从根目录开始，逐级读取目录文件，并确定下一级目录文件的位置。如果目录较长，这个过程可能会很慢。另外，文件系统提供dentry缓存机制加快解析。如果最近访问过某个文件或目录，就会把该文件对应的目录项缓存在内存中，不会逐目录查找。

其次，open第二个参数需要指定文件打开以后的访问模式，包括只读，只写和读写。如果文件不存在，可以用O_CREAT指定创建一个文件，还可以指定是否要追加写、截断等。O_EXCL和O_CREAT合用，表示如果文件本身存在就返回创建失败。O_TRUNC表示如果文件存在就把长度截断到0。O_APPEND表示文件打开后总是往尾部追加写。

创建一个文件的流程包括：路径解析（已经提到），检查该目录是否有权限创建文件，为文件分配inode结构，将文件的名字和inode号更新到目录中。如果需要写入还要分配数据块并更新元数据索引。如果是direct io或者同步写还需要将数据块落盘。

open第三个参数代表权限。在创建文件的模式下，第三个参数指定了文件的访问权限，包括文件创建者，所在组和其他成员的读取，写入，执行权限。

read和write系统调用负责文件的读写。均有三个参数。第一个参数是文件描述符，第二个是缓冲区指针，第三个是读或写的字节数。

read和write都从文件的读写指针开始，可以通过lseek修改读写指针。lessk的三个参数是文件描述符，偏移量和偏移起始地址。偏移起始地址可以是文件开头，末尾或者当前读写指针。

https://blog.csdn.net/weixin_44698673/article/details/125729055

删除文件用unlink系统调用，参数是文件路径。之所以参数是文件路径，是因为不同的文件可能对应同一个inode，也就是硬链接。如果通过文件描述符找到文件，就不知道具体想要删除哪一个硬链接。当删除最后一个硬链接的时候，文件的空间才会真正释放。删除文件的过程包括从目录中删除该文件的表项，减少硬链接计数。如果是最后一个硬链接，就调用具体文件系统的删除操作。

删除目录用rmdir系统调用，参数是目录路径，因为打开目录不会返回一个整数形式的描述符。只能删除空目录，目录不空就要递归删除。

截断文件是指定一个长度，如果文件超过这个长度就把多余部分删除，如果不足这个长度就用\0填充到这个长度。truncate和ftruncate系统调用可以截断。前者的参数是路径名和长度，后者的参数是文件描述符和长度。

区别：truncate之前无需先打开文件，ftruncate需要在打开文件的情况下截断。truncate可能会收到符号链接攻击，意思是高权限的文件被链接到低权限的链接，如果文件系统不完善，那么低权限的用户可能可以操作高权限的文件。

### dentry详细介绍

dentry主要包括文件名和路径到inode的映射。同时dentry在内核中以哈希表和LRU链表的方式存放，从而快速定位到目录项。

### page cache加速原理

Linux上会将文件先读入内存，作为page cache映射到进程的虚拟地址空间。page cache就是struct page中的address_space结构体。每个文件会在内核中只有一份page cache。但是可以有多个进程中的struct file指向。

page cache不仅是缓存基于文件的数据页，还用于元数据的缓存。不同的page有不同的操作函数，定义在address_space_operations操作集中。

page cache的数据结构是基数树，用于快速定位某些状态，如脏的页。基数树是一种利用前缀索引快速检索的数据结构，检索复杂度是Ok，k是所有字符串最大长度。page cache的键是页偏移量，使用分层编码压缩的方式减少长度

### 使用page cache和不使用page cache有什么区别

使用page cache的就是buffer io，不使用page cache的是direct io。他们之间的区别有几点。第一是buffer io的读写性能更好，因为可以在内存中命中，而direct io每次都要访存。第二是buffer io需要占用一定内存存放page cache，但是direct io不需要。第三个是buffer io 不用管理一致性问题和崩溃恢复问题，因为page cache已经实现了这个机制，但是direct io需要自己管理。buffer io适合大多数场景，但是数据库等需要精细控制IO或者是已经实现了自己的缓存的应用就可以跳过page cache。

### 使用direct IO需要注意什么
首先需要手动管理数据的一致性，因为direct io只是保证数据落盘，但不会保证元数据都落盘，需要频繁调用fsync保证所有的元数据都落盘。其次还需要自己实现崩溃恢复，出错重发等机制，然后为了保证性能还需要自己实现批量写入和预读的策略，自己实现读写缓冲区。最后还需要把写入数据块的大小和文件系统数据块的大小对齐。
使用direct io需要在打开文件的时候设置O_DIRECT标志。

### Page Cache的一致性与可靠性
更新page cache上的页就成了脏页。一方面操作系统内核线程定期写回脏页，另一方面用户可以主动调用fsync和sync同步某个文件和整个文件系统的数据。最后内存压力大时会导致回收文件页（丢弃或回写）

3个系统调用：fsync：将fd文件的所有脏数据和脏元数据写回磁盘；fdatasync：将fd文件所有的脏数据和必要的元数据写回磁盘。必要的元数据是对访问文件有关键作用的元数据，比如文件大小。但是文件修改时间就不写回。sync：将整个文件系统的脏数据和脏元数据写回磁盘。

### page fault 的过程

page fault主要包括缺页异常和权限异常。缺页异常是页表项不存在，权限异常是页表项存在但是没有操作权限。

触发page fault时，首先这是一个中断，因此要保存用户态上下文，进入内核态，并根据中断向量表找到中断处理函数。然后检查虚拟内存地址属于哪一个vma，如果不属于任何一个vma说明还没有分配，因此终止进程。
根据vma的类型决定缺页处理逻辑：新匿名页缺页则分配零页或新物理页，文件映射缺页则从磁盘加载到page cache。swap页缺页则从swap分区换入被换出的物理页。

对于匿名页的首次读访问，且表项为空，会映射到一个内容全为0的全局只读零页，后续发生写操作会再次触发page fault权限异常，进入写时拷贝流程。这是为了延迟分配物理页，等到要写入才真正分配。

对于匿名页的首次写访问，会分配新物理页并更新表项。

对于文件页的读访问，会从磁盘上读取文件页，并且会用预读机制优化

对于文件页的写访问，如果是私有文件映射，会写时拷贝避免影响其他进程。如果是共享文件映射，会直接修改物理页。

### page fault 的预读机制

预读机制是触发文件页的缺页中断时，会从磁盘上读取该文件页以及后续若干文件页。预读窗口的大小与访问模式有关，检测到顺序访问会增加窗口，检测到随机访问会缩小窗口。

### 一个进程open的文件，进程异常退出了，脏数据会落盘吗
会，因为文件的page cache在内核中独立于进程，进程崩溃了，page cache依然会回写脏数据。但是如果系统也崩溃，如掉电等，则不能保证。

如果脏数据在用户态缓冲区还没有写入内存page cache，则也会丢失。

### 一个进程在写文件，另一个进程删除该文件，删除会不会成功，为什么，写文件的进程能不能继续写

会成功，可以继续写。

因为每个文件有硬链接计数和进程引用计数。删除文件只是减少硬链接计数，并且从目录中删除该文件的项。只要有进程打开文件，则进程引用计数不为零，数据块仍保留在磁盘中。

写进程可以继续写，因为进程通过文件描述符操作文件，即使没有文件目录也可以写。并且可以正常落盘。只有最后一个进程关闭文件后才会删除。

### 进程线程协程，协程的实现，优缺点

进程是操作系统资源分配的基本单位，线程是CPU调度的基本单位，协程是用户态的轻量级线程。

进程拥有独立的资源，包括地址空间、文件描述符等。进程之间具有隔离性，不同进程需要进程间通信机制交换数据。进程切换的开销大。

同一进程内的不同线程共享地址空间，不需要专门的通信机制，但是需要锁和信号量机制防止数据竞争。线程切换的开销比进程小。

协程是用户态实现的更轻量的执行单位，由用户态的函数库管理，操作系统不知道协程的存在。协程栈只有KB级别，线程栈有MB级别，所以协程可以实现高并发，切换非常快速，而且不需要进入内核态。协程只在等待IO时主动让出CPU而不是被抢占，这是因为内核不能介入协程的执行过程，无法打断协程执行流，并且如果允许抢占则需在任意位置保存全部状态，开销大。协程在线程内串行执行不需要加锁。

但是协程无法利用多核CPU，因为内核不知道协程的存在，所以无法调度。而且协程的使用和调试复杂，需要函数库支持。

进程适合需要隔离多个任务的场景，线程适合需要共享数据、并发的场景，协程适合高并发的场景。

### 创建进程线程的函数
#### 创建进程：

使用fork可以创建一个进程，具体是根据父进程复制一个子进程，创建时不为子进程分配资源，而是采用写时拷贝技术，其页表项设置为只读，只有写入时触发page fault分配新页面并复制。

exec系列函数用于替换当前进程的代码和数据，加载并执行新程序，覆盖原进程的代码段和数据段。经常和fork连用，因为fork的写时拷贝技术避免了复制父进程的无用数据。但是保留文件描述符等。

#### 创建线程：
c语言有基于posix接口的pthread库，pthread_create创建线程，pthread_join等待线程结束并回收资源。线程函数调用pthread_exit主动退出。pthread库中还有互斥锁和条件变量。
C++11中有thread库，是基于pthread实现的，用法更简单，创建一个thread对象的时候就启动线程，主线程中调用线程对象的join方法等待线程返回或者调用detach使子线程脱离主线程。有互斥锁和条件变量，C++14提供读写锁。

### 怎么排查死锁

用户态死锁：

pstack/gdb（C/C++）

对进程执行 pstack <pid> 或 gdb -p <pid>，查看各线程的调用栈和锁状态。

top/htop

观察CPU占用率异常的进程，死锁线程可能处于高CPU或持续等待状态

strace

跟踪进程系统调用，发现卡在 futex 或 pthread_mutex_lock 等锁操作

内核态死锁

内核日志：通过 dmesg 查看 WARNING 或 deadlock 关键字

coredump分析：利用 gdb 分析转储文件中的线程堆栈和锁状态

使用Ftrace分析锁竞争路径

### 操作系统如何保持进程隔离性

操作系统通过虚拟内存机制隔离不同内存，虚拟内存机制使得每个进程只能看到属于自己的连续独占虚拟地址空间，看不到其他进程的虚拟地址空间，所以保证了隔离性。

### mmap是否影响进程隔离性

不影响。mmap可以有共享和私有映射。私有映射仍然是隔离的，共享映射是进程间通信方式。

### 进程线程之间哪些资源共享哪些不共享，线程独有的资源有哪些，为什么独有

线程可以共享地址空间，文件描述符，代码段，数据段、堆、环境变量和用户身份等。

但是每个线程有自己独立的线程id，栈和寄存器。因为线程在并行或并发执行不同任务时用到的局部变量和函数不同，如果栈不独立会导致栈帧被覆盖。

例如，某个线程在栈上分配了局部变量，另一个线程分配了另一个局部变量，假如第一个线程想要释放局部变量，就会导致另一个线程的也释放，因为栈时后进先出的。而堆不会，堆释放了不会立刻归还内存。

不同的线程会嵌套执行多个函数，函数的返回地址需要按顺序压入栈，如果栈不独立会导致函数返回错乱。

### vfs四个关键结构体
要使用某种文件系统，必须先将其注册到 VFS 核心。注册文件系统就是将该文件系统类型插入一个 file_systems 单链表中。目的是向 VFS 提供 get_sb 和 kill_sb 回调函数，从而装载或卸载该类型的文件系统。
VFS中的对象都是仅存在于内存的。具体文件系统的对象会落盘，有盘上结构
1. 超级块super block存放了已挂载文件系统的元数据和控制信息，主要用来指向 fs 超级块（s_fs_info）以及 fs 提供的超级块操作表(s_op)。因为不同文件系统的操作时不同的，这个VFS超级块就是为了向上层提供统一的接口。文件系统层面的操作一般有分配、删除、写回inode、将超级块写入盘上、锁住文件系统和解锁文件系统（比如文件系统要sync）等。
2. inode唯一表示文件，通过inode编号管理，包括操作函数表等。VFS的inode只存在内存中，具体文件系统的inode会落盘。
3. 目录项用来避免重复解析文件路径，加快路径解析速度。
4. file文件对象表示进程打开的文件示例，包括访问模式、偏移量和操作函数表。

### 比较LRU和FIFO

LRU和FIFO是两种常见的缓存替换算法。

LRU优先淘汰最久未被使用的数据，基于时间局部性原理，通常使用哈希表和双向链表实现，可以在O1内完成插入查找和删除。数据局部性较好的时候缓存命中率高。

LRU的缺点第一是实现比较复杂，需要跟踪页面的访问情况，第二是对突然切换工作集的情况不适用，因为可能频繁逐出新工作集中的数据（这是LFU）。第三是不适合随机IO，因为会预读失效，第四是不适合顺序扫描大文件，因为这种模式缺乏时间局部性，每个块看起来都是最近使用的，会污染缓存，也就是缓存被新加载但是不会再被使用的数据填满。

FIFO按照进入队列的顺序，淘汰最先进入队列的数据，使用队列实现，O1。实现简单，但是性能比较差，尤其是不符合先入先出假设的时候。并且还有belady异常，也就是队列增大，命中率反而下降。原因是队列深度可能不足以覆盖整个工作集（工作集就是一段时间内频繁访问的页面集合）

LRU适合对性能要求较高的系统，FIFO适合对性能要求不高，资源有限的系统，或者数据访问本身就比较随机的场景。

### 如何设计LRU

采用数据结构为双向链表和映射unordered_map。需要保存的信息有LRU队列的容量capacity，当前元素数量size和链表的头尾节点。实现的操作有，一是插入，如果待插入的节点已经存在于链表中就更新值并移动到头部。如果没有就在头部新增一个节点。同时如果size大于capacity就需要把尾部逐出。二是访问，如果数据在LRU中就读出来并且移动到头部，如果不在可能就要从别处读取，比如从磁盘中读取。

### 如何改善LRU
1. LRU只记录最近一次访问的时间，可以改成记录最近K次访问情况，比如最近k次访问中至少访问一次。
2. 双队列LRU：将缓存分为两个部分，一个是短期的FIFO队列，另一个是长期的LRU。数据第一次进入短期队列，第二次再进入LRU。可以解决一次性数据带来的预读失效和缓存污染
3. 多级缓冲队列：设置多个优先级不同的队列，每个队列都是LRU的，数据被访问多次后可提升到优先级更高的队列。

### 操作系统有哪些进程和线程的调度算法

先来先服务、短作业优先、高响应比优先、时间片轮转、高优先级、多级反馈队列

先来先服务：先来先服务是最简单的调度算法，原理就是将作业按照提交的顺序假如队列，每次从队列的头部取出作业执行。优点：实现简单；缺点：对长作业有利，导致短作业被长作业卡住，响应慢。适合CPU密集型的任务，因为可以不被干扰一直运行，不适合IO密集型的任务，因为IO经常要等待，就要重新排队。

短作业优先算法：短作业优先是指每次尽量选择执行时间较短的作业。优点：对短作业友好，不会被长作业卡住；缺点：可能导致长作业饥饿

高响应比优先算法：根据服务时间和等待时间综合计算优先级，服务时间加等待时间除以服务时间。优点：兼顾长短作业，对于两种作业来说，都是等待时间越长优先级越高，对于短作业来说服务时间短，所以优先级略高于长作业。这种基于优先级的算法都可以是抢占式也可以是非抢占式。随着运行可能出现优先级更高的其他作业。

时间片轮转算法：给每个作业分配时间片，每个作业只能运行一定时间片，然后要重新排队。优点：实现简单，公平；缺点：时间片不好选择，时间片太短导致频繁切换，时间片太长退化成先来先服务。

高优先级算法：给每个作业分配优先级，每次执行高优先级的作业。优先级可以是静态的也可以是动态的。优点：保证高优先级的作业及时响应；缺点：导致低优先级作业饥饿。

多级反馈队列算法：设置多个优先级队列，优先级越低的队列时间片越长。将第一次调度的作业放到最高优先级的尾部，如果时间片用完了，就移动到下一级队列。每次只有上一个优先级的队列空才会执行下一个队列的作业。优点：兼顾长短作业，长作业虽然优先级低，但是时间片也长；缺点：仍然可能导致长作业饥饿。并且实现复杂。改进：定期将低优先级队列的内容加回高优先级队列。

### Linux块层如何调度IO请求
上层应用调用read或者write发送IO请求，在文件系统中会封装成bio的形式发给块层。块层会将bio合并转化为一个或多个request，并将request插入对应块设备的请求队列中。每个块设备都有自己的请求队列。请求队列会采用一定调度算法进行排序。比如可以先来先服务，也就是不做调度，还可以用deadline机制，也就是排队时间越长优先级越高。

多队列机制：暂时说不上来

### Linux的IO栈
首先是一个用户态的程序调用read write等涉及IO的系统调用，然后是进入内核态，由虚拟文件系统VFS将请求交给具体的文件系统处理。文件系统将请求封装成bio发给块层，块层经过调度之后发给设备驱动，设备驱动
再发给设备。

不同的线程在不同的时间通过page cache去写文件，而这两个IO离得比较近，怎样去做调度

没看懂，应该是要做磁盘调度算法

### 磁盘调度算法

磁盘调度算法的目的是尽量减少磁头移动的距离，减少寻道时间。

先来先服务：这种算法按照IO请求的到来顺序访问磁盘块。当大量进程IO到来的时候可能会导致访问的磁道很分散，寻道时间长。

最短寻道时间优先：先选择离当前磁头最近的位置。问题在于可能导致位置较远的IO饥饿。

扫描算法或者电梯算法：磁头在一个方向上移动，直到到达这个方向上磁道的末尾，然后再换方向。优点是性能较好，不会饥饿，但是缺点是中间的磁道被扫描的频率比两边高。

循环扫描算法：就是单向的电梯算法，只有一个方向是扫描方向，当一次扫描到头以后就回到起点重新扫描。回来的过程不处理IO。优点：每个磁道被访问的机会比较平均；缺点：请求扫描完不会立刻停止，而是扫描到最后一个磁道。

LOOK算法和CLOOK算法：就是针对扫描算法和循环扫描算法的优化，扫描到一个方向的最远请求就回头。CLOOK的缺点：如果一个方向一直到达请求，那么就一直无法回头，导致另一个方向饥饿

deadline算法：为每个请求设置截止时间，优先服务快到截止时间的请求。优点：可以防止饥饿。缺点：需要额外的资源管理请求的截止时间和更新状态。

对于SSD来说没必要做磁盘调度优化，因为SSD不需要寻道。

### 在文件系统中创建一个文件的流程，如果创建的目录比较长会有什么问题

首先是解析路径，内核需要逐层解析路径的每一个目录，直到找到目标目录。

然后是检查是否有在该目录下创建文件的权限

再分配文件的inode

再将文件的名称和inode编号更新到目录文件中

如果文件需要写入，还需要为文件分配实际的数据块。把数据块先放在page cache中，然后异步刷新到磁盘。

如果目录比较大，可能路径解析的过程比较长。因为需要遍历更多的目录条目。我们可以用符号链接只想较深层次的目录，方便快速访问

### 怎样保证文件系统写入数据的一致性和原子性

文件系统写入的一致性是指文件系统在任何时候都应该保证逻辑上的一致状态，比如文件的创建时间应该早于修改时间，文件的索引应该是正确的。我们一般讨论的是文件系统的崩溃一致性，比如断电的情况下，有可能导致文件系统的不一致。

为了保证文件系统的一致性，我们可以采用日志的方式。比如在ext4文件系统中有独立的日志区域。文件系统在执行操作之前，会先将要执行的操作提交到日志去，然后再去完成这些操作，最后清除日志。如果文件系统崩溃，那么在重新挂载以后就扫描日志区域，看有没有未完成的事务。

如果文件系统在写日志的时候崩溃，那么数据是否能恢复取决于日志的状态，如果日志中的某个事务没有提交，那么这个事务就不重新应用。只有提交的事务才能重新应用。

ext4提供了几种日志机制，可以保证不同的一致性等级。最差的是writeback机制，这种机制只把元数据的更新写到日志里。那么在上电之后可能会导致元数据指向的数据还没来得及落盘。第二个是ordered机制，他也只把元数据的更新写到日志里，但是必须保证先把数据落盘，然后再写元数据的日志。这样就不会导致元数据指向不存在的数据。第三种是journal机制，就是把数据和元数据都写入journal。这种机制会造成严重的写放大。

除了日志机制以外，还有写时拷贝技术。在文件系统中就是在修改数据的时候，不在原始的地方修改，而是把数据复制到一个新的区域修改，修改完成后再将元数据指向新的位置。比如btrfs和zfs都支持COW。优点是不用重放日志，上电以后就是最新的状态。缺点是导致写放大，因为复制会有开销，并且带来了额外的元数据更新。还会导致磁盘碎片化。

然后还有日志结构文件系统的机制。日志结构文件系统是把整个盘当作一个大日志，在写入的时候也是先写入数据再写入元数据。上电之后再扫描写过的区域重建文件。这种文件系统会定期做检查点来保证一致性。

### 基于HDD的文件系统和基于SSD的文件系统的区别

基于HDD的文件系统可能采用连续分配来优化大文件的顺序读取。因为HDD更适合顺序IO。连续分配就是尽量给数据块分配连续的地址，这样可以减少寻道时间。但是容易产生外碎片，需要定期做磁盘整理。还可以通过设置预读来减少磁头移动。

然而SSD不用寻道，所以顺序和随机IO都擅长。但是SSD的闪存有读写寿命，所以针对SSD的文件系统需要减少写放大。比如采用日志结构的F2FS文件系统。此外TRIM命令对SSD非常重要，可以告诉SSD哪些数据块不再使用，可以擦除，这样就不用等到写入之前再做GC。

### CPU密集型和IO密集型应用特点

CPU密集型应用是指执行流程中大部分时间用于CPU运算（如大规模矩阵计算），IO密集型应用是指执行流程大部分时间用于IO等待（如磁盘寻道或网络传输）

CPU密集型应用的优化可以：采用多核CPU或者GPU并行计算

IO密集型的应用可以采用IO多路复用机制减少IO的监控开销，还可以采用缓存机制减少IO次数。还可以通过协程加速。因为协程的切换开销小，并发性高。

### 什么是vfork

vfork也是创建一个子进程，但是允许子进程和父进程共享虚拟地址空间，所以可能会导致数据竞争问题。vfork的初衷是避免创建进程时复制地址空间以减少开销，但是现在fork采用写时拷贝技术，不会弄直接复制。因此vfork很少用到。

### 进程间通信方式

进程通过虚拟地址空间实现了隔离性，因此进程之间的协作需要专门的通信机制。

宏内核进程间通信有管道、消息队列、共享内存、信号量、信号和socket机制。
• 管道：字节流、两个进程、单向、匿名和命名
• 消息队列：消息体、多个进程、单向或双向
• 共享内存：内存区间、多进程、单向或双向
• 信号量：计数器、多进程、单向或双向、同步和互斥
• 信号：事件编号、多进程、单向
• 套接字：数据报文、两个进程、单向或双向、网络栈

#### 管道

管道就是一个单向队列，一端发送一端接收。

有匿名管道和命名管道。
• 匿名管道就是bash命令中的竖线，只允许临时将前面的输出传递给后面的输入。
• 命名管道需要用mkfifo创建，将数据写入命名管道后终端会阻塞，直到另一端被读出。
• 管道通信效率低，不适合进程之间频繁交换数据

匿名管道通过系统调用pipe，在内核中开辟了一块内存缓冲区，并返回两个文件描述符。文件描述符用于读和写缓冲区。

匿名管道只能用于父子进程和子进程之间的通信，这是因为子进程会继承父进程的文件描述符表，从而找到内核缓冲区。

命名管道通过函数mkfifo创建，可以用于不相关进程之间的通信。因为命名管道有一个具体的路径名，可以像文件一样打开。不过实际上并不存在于磁盘，而是一个内存缓冲区。

由于匿名管道和命名管道文件都是先进先出的，所以不支持lseek操作。

管道传递的是无格式的字节流。

#### 消息队列

消息队列是一个保存在内核的消息链表，链表的每一个节点都是一个消息体。消息的发送方和接收方约定好消息体的格式，发送方将数据放在消息队列中就返回，接收方需要时去队列中取得数据，因此是非阻塞的。

消息队列的生命周期随内核，如果不释放消息队列或关闭操作系统，则消息队列一直存在。但是管道的生命周期随进程，引用管道的进程终止则管道消失。

消息队列适合频繁交换数据，因为是非阻塞的。但缺点是不适合大量数据传输，因为消息体有大小限制，并且消息队列在内核，因此有用户态和内核态之间的数据拷贝开销。管道比消息队列更快，因为不需要消息体的封装与解封装。

#### 共享内存

管道和消息队列都要利用内核缓冲区，因此会有拷贝开销。

共享内存机制是将不同进程的虚拟地址空间映射到相同的物理地址空间。优点是不需要用户态和内核态之间的拷贝，缺点是产生数据竞争。

#### 信号量

为了解决数据竞争引入信号量实现进程之间的互斥与同步。

信号量就是一个整形计数器，因此并不能缓存数据，而是表示能进入临界区的进程数量。

信号量有两个原子操作：P是信号量-1，V是信号量+1.P时信号量<0则P阻塞，V后信号量<=0则表示有进程需要唤醒。P和V必须成对使用。

信号量初始化为1则为互斥锁。信号量初始化为0可以实现进程同步：另一个进程执行完后再V，这是当前进程就可以P然后执行。

#### 信号

信号用于单向的事件通知而不是数据传输。信号量也可以通知，但是需要进程主动查询信号量。信号则可以随时通知另一个进程，并且另一个进程不需要阻塞等待，内核会切换到处理函数。

Linux内核为sigint等信号提供了默认处理函数，也可以自己定义信号处理函数。还可以屏蔽信号。但是sigkill等有些信号是不能屏蔽的，因为用于终止一个进程。

信号处理并不是中断处理。因为处理信号的时机一般是进程从内核态返回用户态之前。信号处理函数一般在用户态执行，上下文会保存在用户栈上。如果信号处理函数中有系统调用则再次进入内核态。

信号与中断的联系与区别：
• 联系：1. 都是异步通信机制；2. 处理完毕时返回原来的断点；3. 有些中断或信号可以屏蔽
• 区别：1.中断有优先级，信号没有优先级；2.信号处理程序在用户态运行，中断处理程序在内核态运行；3. 中断响应是即时的，而信号响应有一定延迟。

#### socket

不同主机进程之间通信需要跨网络。

socket创建参数有协议、报文类型。可以实现TCP、UDP和本地进程间通信。
• TCP通信过程
    ◦ 服务端通过socket创建套接字，并使用bind将套接字绑定到特定的IP地址和端口上。接着调用listen监听客户端的连接请求。
    ◦ 客户端通过socket创建套接字后，调用connect向服务端指定IP地址和端口发起连接请求
    ◦ 服务端正在监听该端口并不超过最大连接数，则完成三次握手并建立连接。调用accept返回一个文件描述符代表与客户端的连接并开始数据交换。
    ◦ 客户端断开连接则调用close，服务端读取数据时读到EOF，处理完数据后调用close表示连接关闭。
    ◦ 注意：监听的socket和建立连接的socket并不一样。建立连接后双方通过read write或send recv向建立连接的socket读写数据。
• UDP通信过程
    ◦ 服务端和客户端分别使用socket创建套接字并绑定到某个端口。
    ◦ 通信时通过sendto和recvfrom。
    ◦ 通信完成后调用close关闭套接字
• 本地socket
    ◦ 可以是有连接的，也可以是无连接的
    ◦ 区别是在socket绑定时绑定一个本地文件系统中的路径作为地址。

### aio与io_uring

https://blog.csdn.net/youzhangjing_/article/details/127848418

https://arthurchiao.art/blog/intro-to-io-uring-zh/

https://kdocs.cn/l/chW272CU3dmZ

aio和io_uring是linux中实现异步io的两种框架。io_uring改善了aio存在的一些问题。
1. 在aio中，提交io操作和获取io操作结果都需要通过系统调用完成，但是系统调用有切换内核态的开销。
2. aio只支持dio，不支持buffer io
3. aio中提交和获取io操作接口存在用户态和内核态之间的拷贝

io_uring中在用户态映射了一块共享内存，从而消除了内核态切换的开销。用户进程向共享内存提交需要发起的IO操作，内核线程从共享内存中读取IO操作，并且写入返回结果。

io_uring创建了3块共享内存，分别是提交队列、完成队列和提交队列项数组。这三个队列都是环形的，类似与NVMe协议中的环形命令队列。提交队列存储待执行的IO操作索引，具体的IO操作参数放在提交队列项数组中。完成队列存储操作结果。

操作流程是用户提交SQE，内核处理，结果写入CQE，用户读取CQE。整个过程不需要用户态和内核态之间的转换和拷贝。

### 如何优化malloc

如果所有的线程都从同一个地方分配内存，会使得竞争激烈，所以可以划分不同的分配区。优化malloc有预分配和线程本地缓存的思路。比如tcmalloc就是每个线程独立维护一个本地缓存，用于快速分配小对象。优点在于线程的本地缓存不需要获取全局锁，线程之间没有竞争，所以速度快。中对象的缓存全局共享，大对象用mmap分配。中和大对象使用自旋锁。

还有jemalloc，将分配区划分更细粒度，每个线程绑定一块区域从而减少全局锁争用。释放空间时还会合并相邻空闲块减少碎片。

tcmalloc小对象分配更快，jemalloc可以控制碎片化，但是内存占用更高

注意：ptmalloc分配时，brk在堆上分配需要加锁，因为线程共用一个堆。mmap在映射区分配空间时，每个线程可以绑定自己的分配区（arena），不需要全局加锁，只需要局部锁。

### linux中fsync过程

用户通过fsync(fd)系统调用发起请求，内核检查fd的有效性，获取对应的struct file对象，根据文件系统调用注册的fsync实现函数。

fsync会将文件的脏页从page cache写回磁盘，还会将元数据同步回磁盘

### 信号量是如何唤醒和阻塞线程的

信号量由一个计数器和等待队列组成。计数器代表当前可用资源的数量，当计数器为正数时，线程可以进入临界区，当计数器为0或负数时，线程需要进入等待队列。等待队列存储因资源不足而被阻塞的线程，调度策略可以是FIFO或按优先级排序的

## 网络

### tcp如何保证可靠

TCP通过握手挥手建立连接、校验和、序列号确认、滑动窗口、动态重传及拥塞控制等机制，在不可靠的IP层上实现了端到端的可靠传输。其设计平衡了效率与安全性，例如通过SACK减少重传冗余、通过拥塞控制适应网络波动。实际应用中需注意半包/粘包问题（需应用层定义消息边界）及校验和的局限性（无法完全检测多比特错误）

### 如何用UDP实现可靠传输/QUIC

在应用层实现序列号和确认应答机制，就是为每个数据包添加递增序列号，接收方通过ack报文确认已接受的数据包。并且超时未确认则重传。通过滑动窗口机制避免乱序。这相当于把tcp在应用层又实现了一遍。

但是也可以采用类似QUIC的设计。quic的头部分为长头部和短头部，只有在建立连接的时候使用长头部，连接建立后使用短头部。从而降低连接开销。

quic采用动态协商的连接ID标识连接，而不是五元组（源、目的IP；源目的端口；协议）表示。这样能够支持无缝切换网络，并且不易被追踪。

quic采用严格递增的序列号，因此重传的序列号也不一样。避免出现TCP中的ACK歧义问题（第一次响请求的ACK在网络中堵塞，重传后又发了ACK，无法知道这两个ACK哪一个是重传的，从而不能准确计算RTT）。并且可以支持乱序确认，因为TCP丢包会导致窗口不滑动，而quic中的序列号是严格递增的，可以通过更大的序列号确认后续重发的包。

之所以支持乱序确认，是因为数据包内容的顺序不是依赖packet number，而是依赖stream id和stream offset表示的。数据接收方根据stream id将数据归类，再根据stream offset重新排序。

流量控制方面，quic对每个流限制了最大偏移量，防止一个流数据量过大耗尽缓冲区。quic对所有的流的总数据量也有限制，避免整体资源超出限制。根据RTT时间和缓冲区的处理速率动态调整每个流的最大偏移量和所有流的总数据量。

拥塞控制和TCP差不多，但是可以根据不同应用选择不同拥塞控制算法，而TCP只能用同一套拥塞控制算法。

### 队头阻塞问题

有哪些拥塞控制算法

经典的有慢启动、拥塞避免、拥塞发生、快恢复。这是reno算法

还有更多的，但是这里看不完了。

### 三次握手的过程

三次握手的过程是1. 服务器监听某个端口2.客户端发送第一次握手报文，包括随机初始化序列号、SYN标志并进入syn sent状态3.客户端返回第二次握手syn ack，是对第一次握手的确认，包括确认应答号是第一次握手的序列号+1，随机初始化序列号，SYN和ACK标志并进入syn rcvd状态3. 客户端返回第三次握手报文，包括确认应答号是序列号+1，而且可以携带数据包。进入established状态，4. 服务器收到第三次回收进入established状态

注意序列号代表报文第一个字节的编号，下一个序列号是上一个序列号+上一个报文长度。确认应答号是期望收到的字节编号。例如上一个请求序列号是1，长度1000，则成功收到的确认应答号是1001，下一个报文序列号是1001。

### 每一次握手丢失会怎么样

首先，tcp有超时重传机制，如果迟迟收不到应答会重发请求或者响应。重发次数可以设置，每一次重发间隔是指数增长的。不过，第一次握手、第二次握手、普通报文和挥手阶段重传的参数是不一样的。

如果第一次握手丢失，会导致一直收不到ack应答，导致触发重传。一直收不到应答则最终建立连接失败。

如果第二次握手丢失，会导致服务端收不到第三次握手，服务端会重传ACK。此外客户端会认为是第一次握手丢失，所以会各自重传第一次握手。注意，如果客户端重传第一次握手后收到迟到的ACK响应，会与当前期望的ack不匹配，认为是旧连接，从而丢弃迟到的ack报文。如果重传次数耗尽，就会关闭连接。

如果第三次握手丢失，服务器会以为第二次握手丢失从而重传第二次握手，客户端根据重传的第二次握手重新发送第三次握手，但是客户端不会主动重传第三次握手。

### 什么是syn flood 攻击，有什么危害，如何解决

如果攻击者伪造不同ip大量发起第一次握手而不发送第三次握手，就会导致服务器处于大量syn rcvd状态，从而占用服务器的系统资源。

处于syn rcvd的连接放在服务器的半连接队列中，半连接队列过多会导致无法为新连接分配资源，导致服务器无法响应正常请求。

可以增大半连接队列、减少第二次握手的重传次数以快速释放无效连接，或者启用syn cookie技术，就是服务器收到第一次握手后不为连接分配资源，而是再响应报文中嵌入一个cookie值，客户端返回携带正确cookie的第三次握手才会为连接分配资源。

### 什么是半连接队列和全连接队列

半连接队列是服务端收到第一次握手后，处于syn rcvd状态的连接队列，全连接队列是服务端收到第三次握手后处于established状态的连接队列。

半连接队列用于临时存放未完成握手的连接请求，全连接队列用于缓存已经建立但是来不及被处理的连接。

半连接队列采用哈希表存储，为了收到第三次握手后快速根据tcp四元组定位特定连接。全连接队列使用链表存储，只需要先进先出取出队列头部即可。

### 四次挥手的过程

连接双方都可以主动关闭连接，1.主动关闭放发送第一次挥手，就是设置FIN位的报文，表示主动关闭方不再发送数据但是可以接收数据，进入fin wait 1状态2.被动关闭方发送第二次握手，就是一个ack报文，表示知道关闭请求，但是自己可能还有数据要发送，可以接着发送数据。如果没有数据发送也可以将第三次挥手合并到第二次挥手一起发送。第二次挥手发送后被动关闭放进入close wait状态。3.被动关闭方没有数据要发送了，则发送携带FIN标志位的第三次挥手并进入last ack状态4.主动关闭方收到第三次挥手并发送ack报文，进入time wait状态5. 被动关闭方收到第四次挥手进入closed状态。

### 每一次挥手丢失会怎么样

第一次挥手丢失，会导致主动关闭方收不到ack响应，重发第一次挥手。超过最大重传次数后就强制断开连接

第二次挥手丢失，主动关闭方会认为第一次挥手丢失，从而重传第一次挥手，被动关闭方收到重发的第一次挥手，就再发第二次挥手。

主动关闭方收到第二次挥手后进入fin wait2状态，如果主动关闭方用close关闭，则代表不再收发数据，因此close wait会有超时时间，如果超时时间内未收到第三次挥手则直接关闭连接。如果是shutdown关闭连接则是不发，但是可以收，因此主动关闭方可以一直处于finwait2.

第三次挥手丢失则收不到ack，被动关闭方会重发第三次挥手。和第一次挥手相似。

第四次挥手丢失，则被动发送方认为第三次挥手丢失，则重发第三次挥手。

### time wait和close wait过多的原因，怎么解决

time wait是四次挥手结束后，主动发起关闭的一方进入的状态。作用是1. 防止旧连接干扰，吸收网络中残留的数据包，避免影响后续新建的同端口连接（序列号虽然随机但是可以回绕，如果序列号刚好符合下一个同端口连接则会导致数据错乱）2.如果最后一次ACK丢失则另一方会重发FIN，这时可以重发ACK避免异常关闭连接，持续2MSL，认为数据包全部死亡。

time wait过多说明连接频繁断开，可能是因为频繁使用短连接、或者大量客户端建立连接后不发送数据导致长连接超时、或者http长连接请求的数量达到上限（每个长连接只能处理上限次请求就要关闭）。

timewait过多会导致占用系统资源和端口资源。占用系统资源指会占用文件描述符、内存和CPU。占用端口资源是指timewait时无法对相同IP相同端口发起连接。对于客户端来说，端口资源有限，占满会导致新连接报错。对于服务端，虽然监听单一端口，但是可以处理不同客户端，只是不能和刚才这个客户端马上建立连接。但是都会占用系统资源。

解决timewait过多可以开启长连接代替短连接、排查是否有网络问题导致大量连接超时、允许复用timewait窗口（开启tcp_te_reuse和tcp_timestamp，通过引入时间戳区分新旧数据包）、调高长连接请求上限。

close wait是被动关闭方发送第二次挥手ACK后进入的状态，该状态允许被动关闭方继续发送数据。

close wait过多的原因是被动关闭方可能由于代码缺陷没有正确关闭连接。

close wait过多会导致占用系统资源（文件描述符、内存等）、占用端口。

如果暂时close wait过多可以重启服务强制释放连接。但是根本在于排查代码的缺陷，需要正确关闭连接。

### close和shutdown关闭

close是关闭socket的文件描述符，并减少引用计数，当引用计数减为0就会真正释放资源并出发4次挥手。close会完全关闭双向通信，无法会通过套接字进行读或写操作

shutdown是关闭指定方向的通信，而不是减少引用计数。

close会立即发送fin包，可能丢弃缓冲区的未发送数据，导致数据丢失。如果接收缓冲区有未读数据则会发rst而不是fin。shutdown会等待缓冲区处理完毕再发fin。

主动调用close后，直接进入timewait，只有3次挥手，调用shutdown有4次

### 什么时候会用rst报文
1. 目标端口未开放，则服务器直接回复rst拒绝连接
2. 重复syn报文，例如一个syn报文在网络阻塞，然后重传新的syn并且建立了连接，就会返回rst终止无效请求
3. 连接一方因为异常终止连接，又重启，收到对方继续发的数据就会返回rst
4. 接收到数据包序列号不在滑动窗口范围内，或者数据包校验错误，会发rst
5. 尝试连接timewait状态的服务器会rst
6. 接收方缓冲区溢出，处理不过来
7. tcp保活机制检测长时间无活动

### Linux查看已连接socket、端口的命令

ss命令可以看，参数-a是所有，-t是tcp，-u是udp，-l是监听端口，-p是展示进程信息

netstat可以看，参数一样，但是性能没有ss好

### TCP和UDP区别

tcp是面向连接的有序可靠传输，udp是无连接的不可靠传输。

tcp需要复杂的重传和建立释放连接开销，头部大。udp传输效率高但是可能丢失，头部小。

tcp适合文件传输、网页浏览等需要可靠性的场景，udp适合实时性高，如视频通话等场景。

### 在电梯间内网络不稳定，如何解决音视频信号传输卡顿

可以动态调整码率，减少传输数据量。

采用quic协议，首次握手时间仅需1rtt，后续可以0rtt握手。并且支持多路复用，允许并行传输数据流，支持前向纠错，发送冗余数据包，可以减少重传次数。并且切换网络可以保持连接不中断，避免重建连接。最后内置tls加密，不需要专门加密握手。

因为弱网条件丢包率高容易卡顿，需要通过quic改善可靠性。

### TCP拥塞控制过程

TCP拥塞控制的主要过程可以简化为以下几个阶段：

​慢启动（Slow Start）​
        TCP连接建立后，拥塞窗口（cwnd）初始化为1个MSS（最大报文段长度）。
        每收到一个ACK确认，cwnd增加1个MSS，因此cwnd呈指数增长。
        当cwnd达到慢启动阈值（ssthresh）时，进入拥塞避免阶段

拥塞避免（Congestion Avoidance）​
    在拥塞避免阶段，cwnd每经过一个RTT（往返时间）增加1个MSS，呈线性增长。
    目的是避免cwnd增长过快导致网络拥塞

​拥塞发生（Congestion Detection）​
    如果发生超时，TCP认为网络拥塞严重，将ssthresh设置为cwnd的一半，cwnd重置为1，重新进入慢启动阶段。
    如果收到3个重复ACK（快速重传），TCP认为发生部分丢包，进入快速恢复阶段

​快速恢复（Fast Recovery）​
    在快速恢复阶段，ssthresh设置为cwnd的一半，cwnd减半并加上3个MSS，然后进入拥塞避免阶段。
    目的是在部分丢包的情况下快速恢复传输效率

总结：TCP拥塞控制通过慢启动、拥塞避免、拥塞发生和快速恢复四个阶段动态调整发送速率，避免网络拥塞并提高传输效率

### 两个进程可以用同一个端口吗

如果是不同协议，如一个是TCP，一个是UDP，则各自端口空间独立，可以重用。

如果是同一协议，但是不是同一IP，也可以重用。因为可以配置多个网卡，转发到不同应用。

内核通过四元组表示连接，如果四元组相同，默认不能显式绑定同一IP地址的同一端口。

如果开启SO_REUSEADDR选项，则可以复用timewait状态的端口。或者开启tcp_tw_reuse和tcp时间戳，也可以重用time wait状态的端口

如果开启SO_REUSEPORT选项，则可以多个进程绑定同一个IP和端口。绑定同样IP和端口的连接放到一条哈希冲突链上，

### socket怎么重用连接

为了避免频繁创建和销毁socket，可以使用socket连接池。原理是预创建多个socket，需要用到连接时取出，用完归还。

初始化时创建一组socket放入池中，避免后面请求时同步的三次握手建立连接。需要使用连接时从池中获取空闲连接，用完后归还到池中而不是关闭。从而允许下次再用。

需要定期检查连接的有效性，发送心跳包探测连接是否断开。

连接池可以动态扩展和收缩，根据当前负载和资源消耗。

空闲连接超时后需要释放，避免资源泄漏。

### 一条TCP连接上可以发多少个http请求

短连接模式下，每个http请求结束后会立即释放tcp连接。

长连接模式下，多个http请求可以复用一条tcp连接。理论上只要tcp存在，则可以发的http请求没有上限，但是服务端会限制单个tcp连接处理的http请求次数，这个参数可以调整。

socket连接中，如果客户端在read()的时候服务器挂掉，会怎么做？客户端有感知吗

1. 如果是服务器进程崩溃了，则内核会回收其socket资源，并发送FIN报文，触发四次挥手。客户端的read会立刻返回0，表示对方已经关闭连接，所以客户端有感知
2. 如果服务器宕机，
    a. 如果开启了tcp保活机制，则一段时间后会发送探测报文，探测报文多次无响应会断开连接
    b. 如果没有开启则会一直阻塞，直到客户端下一次给服务器发送报文得不到响应，并且超时重传也得不到响应才会断开

## 数据库

### 事务的特性，如何保证

事务的特性是ACID，即原子性、一致性、隔离性、持久性。

原子性是指一个事务中的所有操作，要么全部完成，要么全都不完成。

一致性是指事务操作前后，数据满足完整性约束，不能出现我账户的钱减少而对方账户钱不增加的情况。

隔离性是指，数据库允许多个并发事务读写数据，需要隔离多个事务读写数据时的相互干扰。

持久性是指，数据库事务提交后，对数据的修改是永久的，即使系统故障也不会丢失。

原子性通过undo log回滚保证，在事务执行之前，数据库会记录操作前的数据状态到undo log，如果事务失败则根据undo log逆向操作，将数据恢复事务之前的状态。

一致性是由原子性隔离性和持久性保证的

隔离性可以通过锁机制和多版本并发控制保证。

持久性可以通过redo log保证。

### undo log和redo log什么时候起作用

undo log用来保证事务的原子性，保证事务失败时可以通过undo log回滚到事务发生之前的状态。redo log用来保证事务的持久性，保证系统故障时可以根据redo log重放操作。
事务在commit之前不会落盘，因此这个阶段主要是根据undo log保证能够回滚。事务在commit之后保证落盘，这个阶段如果出问题就根据redo log重放操作。

### 事务的隔离级别

四个级别：读未提交、读已提交、可重复读、串行化

读未提交是一个事务的修改在提交之前就可以被其他事务看到。

读已提交是一个事务的修改只有在提交之后才能被其他事务看到。

可重复读是一个事务执行过程中看到的数据一直和事务启动时看到的数据是一致的。是innodb引擎的默认隔离级别

串行化是对记录加读写锁，多个事务通过读写锁访问数据。

脏读是指读取到其他事务还未提交的数据，这些数据是有可能回滚的。

不可重复读是一个事务多次读取同一个数据，但是前后读到的不一样。比如数据被其他事务提交，但是这个数据和本事务开始时看到的不一样

幻读是一个事务中满足查询记录的条目数量会因为其他事务提交数据而变化。

读未提交可能发生脏读、幻读、不可重复读

读已提交可能发生不可重复读和幻读

可重复读可能发生幻读

串行化不会发生这些现象。

### 可重复读的实现原理

在可重复读级别下，每个事务在第一次查询时会生成一个readview，后续的查询语句会利用read view找到开始查询时的数据，所以每次查询都是一样的。

### 可重复读不能解决幻读

事务的读取有快照读和当前读。快照读是普通的select语句，基于事务启动时的read view。当前读是select dor update、update或insert等显式要求读取最新数据或者可能修改数据的操作。

事务如果更新了其他事务插入的数据（虽然看不到），会导致更新快照，后续查询会包含新数据。

可以在开启事务后，马上执行当前读语句，这样会给记录加next key锁，避免其他事务插入新记录。

### MVCC版本链实现、记录的可见性

读提交在每个select执行之前重新生成read view

可重复读在执行第一个select时生成一个read view，然后该事务都用这个read view

read view有4个字段，包括创建read view 的事务id，创建read view时当前数据库中未提交的事务id列表，创建readview时当前数据库中未提交的事务中最小事务的id，创建read view时当前数据库预留的下一个事务的id

innodb存储引擎的数据表记录中，每条记录会有隐藏列，包括事务id和指向旧版本记录的指针。指针的意思是说，每次改动记录时会将旧版本的记录写到undo日志中，这个指针就可以找到旧版本记录。

如果记录的事务id小于read view中未提交事务的最小id，就说明记录在创建read view之前已经提交，所以可见。

如果记录的事务id大于read view预留的下一个id，说明事务是在创建read view 之后创建，所以不可见。

如果记录的事务id在两者之间，则检查read view的未提交事务列表，如果在列表则说明未提交，所以不可见，如果不在则已经提交，所以（对读提交来说）可见

### 事务没有commit之前会不会持久化，为什么需要undo log

事务在commit之前，数据的持久化程度取决于redo log的刷盘策略。

在事务的运行过程中，mysql会把日志写到redo log buffer中，等到事务真正提交的时候，再将buffer中的内容写到redo log文件中。但是这个过程没有绕过page cache，也就是说提交的时候只是写到了pagecache，还需调用fsync保证落盘。

但是，在事务还未提交的时候，redolog也可能落盘，因为innodb有一个后台线程，会定期将redolog buffer中的日志写到pagecache，然后调用fsync持久化，因此事务没有commit也会部分落盘。并且其他事务提交时，可以设置将redo log buffer全部落盘，因此可能被带着持久化。第三种情况是事务的redo log buffer超过指定大小，这时候也需要写盘。

因为事务在未提交时候也可能部分落盘，所以需要undo log。

### 一条update是不是原子性的

update是原子性的，主要通过锁和undolog日志保证。执行update时会加行级锁，保证一个事务更新一条记录时不会被其他事务打扰。事务执行过程中会生成undolog，如果执行失败可以根据undolog回滚。

### 一个事务的语句特别多会怎样

为了保证事务的原子性和一致性需要加锁，如果语句过多会导致锁住太多数据，使得数据库性能下降，并可能会锁超时。其次需要保存的回滚记录太多占用空间。最后这个事务的执行时间长，会导致主从延迟（？）

### mysql的锁

mysql中有全局锁、表级锁和行级锁。

全局锁会将整个数据库锁住，用于全库备份。

表级锁：
• 表锁：lock tables 表名 read/write，read就是读锁（共享锁，其他线程只读），write就是写锁（独占锁，其他线程不能访问）。注意加锁之后在解锁之前本线程不允许访问其他表。表锁的颗粒度大，影响并发性能
• 元数据锁：这里的元数据是指的表结构，操作数据表时会自动加元数据锁。CRUD加读锁，更改表结构加写锁。注意这里写锁申请不到，会导致所有事务不能申请读锁。阻塞住
• 意向锁：当事务需要对某行加锁时，先申请表级的意向共享锁或者意向排他锁，这样其他事务可以快速判断该表中是否有行级锁，避免逐行扫描。比如一张表获取了行级独占锁，那么另一个事务就不能对这张表加锁。
• auto inc锁用于管理表内的自增列，保证自增字段的唯一性和连续性。auto inc锁是插入语句执行完就释放，而不是等待事务结束再释放。还有一种更轻量的自增锁，将自增字段赋值之后就释放了，而不是等到插入语句完成。

行级锁：innodb引擎支持，myisam不支持，需要先加表级意向锁避免逐行扫描
• 记录锁：
    ◦ 锁住的是记录，可以有共享和独占两种。一个事务对一行加共享锁后，其他事务还可以对这一行加共享锁，但是不能加独占锁。如果加了独占锁，其他事务不能对这行加锁。
• 间隙锁
    ◦ 间隙锁是在可重复读级别下引入的一种行级锁，锁定的是一个区间而不是具体的记录。加锁之后不能对区间中做插入操作，从而避免幻读。
• next key锁
    ◦ 结合了记录锁和间隙锁，既能锁定记录本身不被修改，又不允许在范围内插入记录。

1. Level DB
2. join table 实现方式
3. 数据库如何处理null
4. 数据库如何判断事务的可见性
• 数据库存储引擎格式
• B+树上数据记录1 1 1，一个操作将其更新为 1 2 2，如何操作？
    ◦ 记录日志
• mysql中事务A读 1 1 1，另一个事务B将其update为1 2 2，但没有提交，那么当前处理器，B+树中存了哪些数据，页面的值是什么，哪些信息发生了变化
• 版本号存在哪里，如何组织
• RocksDB
• 主键索引和联合索引的区别
• 主键索引在索引表上只存有索引数据吗

## 分布式

RDMA

raft协议

GFS如何保证高可用

存储系统如何做容灾备份

## Linux命令
• 查看磁盘使用量的命令
    ◦ df命令查看文件系统的整体磁盘空间占用情况，-h转换成MBGB显示，-i显示inode使用情况，-T显示文件系统类型
    ◦ du命令查看目录或文件的磁盘占用
    ◦ 有隐藏文件怎么办
        ▪ 隐藏文件以一个点开头，可以指定通配符：.[!.]*
            • .*表示以.开头的所有文件
            • [!.]表示第二个不为. 即不包括父目录
    ◦ df统计文件系统整体空间，包括已删除但被进程占用的文件，du统计现存文件的占用。两者差异过大可以用lsof列出系统打开文件进行排查
• 查看某个端口被哪个服务占用的命令
    ◦ ss命令
    ◦ netstat命令
查看dd命令进度的方法
在较新版本的dd中，可直接通过status=progress参数显示实时进度

## git场景用法

## 数据结构与算法

B+树、基数树、红黑树、还有哪些树结构、比较

口述一下用两个栈模拟队列

用两个栈模拟队列就是需要两次翻转。两个栈分别是入队栈和出队栈。比如将12345加入入队栈，在出栈的时候先将其压入出队栈变成54321，最后弹出出队栈就变成了12345.

### 八个常见排序算法
排序算法分为基于比较的和非比较的。非比较的排序算法中一个数的值就决定了应该放在哪个位置
基于比较的排序算法有 选择排序、冒泡排序、插入排序、希尔排序、归并排序、快速排序、堆排序
非比较的排序算法有计数排序、基数排序、桶排序。
非比较的排序算法很有局限性，对数据有很多限定。例如计数排序需要小范围整数、基数排序需要数据是相同位数、桶排序需要数据均匀分布。并且非比较算法通常需要较多辅助空间。

选择排序：每次扫描数组可以选出本轮最小的元素，交换到数组的开头。因此需要On2。选择排序的执行时间与数据无关。
冒泡排序：每一轮扫描将较大的数据和后面相邻元素交换。On2如果某一轮没有数据交换，则说明已经有序可以退出。因此顺序数组可以实现On的复杂度
插入排序：类似扑克牌方式，数组的前端是有序数组，每次将当前的元素插入有序部分，也就是逐个交换，直到前面的元素比该元素小。如果本身有序则On。插入排序比选择排序好，因为对于有序数组可以降为On，而且内层循环很有可能提前终止，而冒泡排序提前终止比较少见。插入排序很适合接近有序或规模较小的数组，因为这些数组很容易提前终止内层循环。归并排序和快速排序拆分到较小子区间的时候可以转为使用插入排序，更快。
希尔排序：比快排差但是不需要递归，可以用在不支持递归的环境。希尔排序是分组插入排序，将数组按间隔分为多个组（比如0，4，8为1组，1，5，9为一组等），对每组执行插入排序。接下来将间隔缩减为一半，再执行插入。因此数据会越来越有序，适合插入排序。最后间隔为1.

快速排序：随机选择区间内某一个元素作为基准，将区间内所有小于基准的交换到基准之前，将所有大于基准的交换到基准之后。因此基准就在应该在的地方。分别再排序pivot左边和pivot右边区间
    之所以随机选择是因为对于有序数组来说会使得左边的子区间特别小、右边子区间特别大，使得递归树倾斜
    双路快排是pivot相等元素过多时，避免重复元素堆积到单侧，导致容易出现有序情况，从而递归树倾斜。每次将等于pivot的元素平均分到pivot两侧：
1. 选择基准值（通常为左端元素，但推荐随机化选择
2. i指针向右移动，直到遇到大于基准值的元素；j指针向左移动，直到遇到小于基准值的元素。
3. 交换i和j的元素，继续扫描直至两指针相遇。
4. 将基准值与j指针位置交换，完成分区。
优势：通过双向扫描，减少单侧元素堆积，使分区更均衡，降低递归深度
    三路快排是每次将等于pivot的元素都排到一起，可以避免接下来的区间中再出现pivot，从而减少了左右递归区间的长度
• 指针定义与初始化
    ◦ lt（左边界）：初始指向l，左侧区域的上界。
    ◦ gt（右边界）：初始指向r+1，右侧区域的下界。
    ◦ i（遍历指针）：初始指向l+1，从左向右扫描元素。
• 遍历与交换逻辑
    ◦ 当前元素 < 基准值：与lt+1位置的元素交换，lt和i均右移。
    ◦ 当前元素 > 基准值：与gt-1位置的元素交换，gt左移（i不动，需重新检查交换后的元素）。
    ◦ 当前元素 = 基准值：仅i右移。
• 终止条件：当i与gt相遇时，遍历结束。
• 基准值归位
    ◦ 将基准值（原左端元素）与lt位置的元素交换，此时基准值位于中间区域的左边界。
    ◦ 递归处理左区域（l至lt-1）和右区域（gt至r），中间区域无需处理

###  B+树 增、删、查Olog_m^n n为总数据量，m为每个节点的最大子节点树

B+树是一种多路平衡查找树，所有数据记录在叶子节点，非叶子节点只存键和子结点指针。叶子节点形成一个链表，便于范围查找和遍历。由于多路，节点存储密度高，适合磁盘，能够减少磁盘IO次数，例如EXT4中的extent。传统inode使用多级间接索引，而B+树可以减少查询次数，提高效率

B树 增、删、查Olog_m^n n为总数据量，m为每个节点的最大子节点树

B树也是一种多路平衡查找树，数据放在叶子节点和内部节点上，叶子节点既存储键值又存储数据本身或者数据的指针。并且叶子节点没有链式连接，遍历不方便（这是因为数据在非叶子节点上，有链表也没用）

与B+树相比，B树不适合范围查询，因为需要回溯遍历。B树不如B+树扁平，因为B+树的非叶子节点不存储数据，可以放更多子节点指针。B树的删除插入节点可能在任意层分裂和调整。而B+树只影响叶子节点，复杂度较小。

B树的优点在于键值唯一，因为B+树只有叶子节点才放值，可能存储重复的键

### 红黑树

红黑树是一种自平衡的二叉查找树，最长路径不超过最短路径的两倍，特点是插入删除的时候，旋转次数少，保证增删改查的时间复杂度是Ologn，适合内存中的数据结构，比如虚拟内存管理中的VMA
通过颜色翻转和很少的旋转可以保持平衡，常数时间

B+树如果插入导致叶子节点分裂需要递归调整父节点的索引键

### B+树插入流程

插入步骤

• 定位叶子节点

从根节点开始，根据关键字大小向下查找，直到找到对应的叶子节点（插入操作仅在叶子层进行）

• 直接插入（未溢出）

若叶子节点当前关键字数 k<m−1，直接按序插入新关键字并更新指针，操作结束

• 分裂叶子节点（溢出处理）

若插入后关键字数 k=m，则分裂为两个节点：

• 左节点：保留前 ⌈m/2⌉ 个关键字。

• 右节点：包含剩余 ⌊m/2⌋ 个关键字。

• 中间关键字上浮：将第 ⌈m/2⌉ 个关键字（即左节点的最大值）复制到父节点作为索引

• 递归调整父节点
    ◦ 若父节点因接收上浮关键字导致 k=m，则继续分裂父节点，直至根节点。
    ◦ 若根节点分裂，则生成新根（树高增加1），新根包含1个关键字和2个子指针

示例（4阶B+树）

1. 插入43：根节点（也是叶子节点）变为 [43]。

2. 插入48, 36：叶子节点变为 [36, 43, 48]。

3. 插入32：节点分裂为 [32, 36] 和 [43, 48]，关键字36上浮到父节点

关键点

• 分裂策略：偶数阶（如4阶）取 ⌈m/2⌉（即第2个关键字）上浮；奇数阶（如3阶）取中间关键字

• 索引更新：内部节点仅存储关键字副本（或最大值）和子指针，不存储实际数据

### AVL树

AVL树也是一种自平衡二叉查找树，可以保证时间复杂度为logn。AVL树要求每个节点的左右子树高度差不超过1，否则需要旋转调整。树的高度比红黑树小，因为红黑树只是保证最长路径不超过最短路径的两倍。但是旋转调整的开销大。红黑树由于放宽了平衡限制，可以保证旋转操作不超过3次，所以适合频繁修改的场景。

### 基数树 Ok

基数树是一种前缀树，结点存储字符或部分字符串，路径从根到叶子标识完整键，适合需要按前缀快速检索字符串的场景，通过合并公共前缀减少存储冗余，例如page cache就是基数树

### 三种树比较

B+树和红黑树查询都是Ologn，但是红黑树插入删除的开销小。

B+树是为磁盘优化的，一是树的高度小，便于减少访存，适合大数据量场景，二是结点与磁盘页面对齐，可以预读其他结点，三是双向链表便于范围查找和遍历。

红黑树的高度高于B+树，但是插入删除结点的旋转的次数少，开销小。而B+树需要合并和分裂结点，操作复杂度为Ologn

B+树用于文件系统和数据库索引，大规模数据树高可控。

红黑树适合内存中经常修改删除的场景，能够减少更新开销。比如stl中的map和set。

基数树能够通过共享前缀减少存储量。例如两个字符串共享前缀，则前缀放在父节点中，后面不同的部分放在两个子节点。通过前缀的匹配能够快速查询字符串，Ok，k是最长字符串长度。比如在page cache中将文件偏移量映射到物理页，偏移量作为键，4K页的地址作为值。

### 完全二叉树

只允许最后一排不满， 中间不可以有空缺，最后一排必须从左往右排序。

### 堆

堆是一种完全二叉树。

大根堆是指堆中每个节点都要大于等于子节点，小根堆是堆中每个节点都要小于等于子节点。

堆通常用数组的形式实现，按照层序遍历存放在数组中

建堆：1. 自下而上：将新元素插入叶子节点并且向上过滤；2.自上而下： 将新元素插入根节点并向下过滤

向上过滤：将某个叶子节点与父节点比较，如果大于/小于父节点就交换，直到根节点。

向下过滤：将父节点与子节点比较，若小于子节点就和最大/最小的子节点交换，直到叶子节点。

堆排序

从小到大排序就是小顶堆（每次弹出最小的根节点），从大到小排序就是大顶堆。

小顶堆/大顶堆建堆：第一步将数组调整为堆，On

• 从最后一个非叶子节点开始逆序下沉/上浮

排序：每次将堆顶元素交换到末尾，并缩小堆范围重新建堆，Onlogn

## 其他

### 实时系统优化

可以从任务调度、高可靠性和资源利用方面展开。比如调度进程的时候可以分配优先级，并且采用抢占式调度，让更紧急的任务提前执行。

内存优化
• 预分配内存池：避免动态内存分配导致的碎片和延迟。例如，在启动阶段为实时任务预留固定内存区域
• 大页内存（HugePage）：减少TLB缺失率，提升内存访问效率（适用于高频访问的数据缓冲区）

### 将ZNS用在云存储上

https://mp.weixin.qq.com/s?__biz=Mzg3MjY5MTc0Ng==&mid=2247485054&idx=1&sn=3606a61e043dee464899134109496a9d&chksm=cf0c4dc9e45288826fc3e866dd3a67f92afebb709f0e3c6141842d9eb2f891da28327af6f6cf#rd

字节跳动做过相关的研究，实际上效果并不好。一方面，使用ZNS需要对软件栈做较大的修改，比如这个顺序写限制。那么对于现有的生态适配是一个挑战。第二个，就是ZNS本身的优化也有点虚。比如，他最原始的那篇文章中是将文件系统层次的GC也当作正常的IO带宽来算，因为盘并不知道文件系统的GC操作，而GC也就是把数据从盘上读出来再写到另一个地方去，那你带宽当然是大的。但是垃圾回收的带宽并不是用户真是想要的，所以ZNS的性能优化本身就是存疑的。第三点，ZNS将盘内管理的工作交给主机去控制，从某种程度来说是给主机增加了负担，因为盘上本身是有一个主控的。对于云服务来说更是这样，本身存储节点也有一定计算能力。那现在将这些结点上的计算能力浪费了。因此不是特别合适。但是我手机上没这个问题，因为手机上一方面用的本身是F2FS，他的管理方式本来就适合ZNS，不会因为使用ZNS额外增加开销。另一方面手机的UFS本身没有计算能力，确实比较适合ZNS。但是对于云服务来说，我看不是特别适合。

### 怎么解决多 CPU 下同时访问自旋锁的性能问题

无锁数据结构替代：对高频访问场景（如计数器），优先使用原子变量或无锁队列

混合锁策略：对长临界区代码切换为互斥锁（如pthread_mutex），避免CPU资源浪费

读写锁分离（Read-Write Lock）

对读多写少场景，使用读写自旋锁（如rwlock），允许多个读线程并发访问，减少写线程阻塞时间

缓存行填充（Cache Line Padding）

将锁变量独占一个缓存行（如64字节对齐），避免与其他变量共享缓存行。例如Linux内核的spinlock_t结构体通过__attribute__((aligned(64)))强制对齐

分片锁（Sharded Lock）

将全局锁拆分为多个子锁，细化锁的粒度，通过哈希函数将资源映射到不同子锁，降低单个锁的争用率。例如，天翼云在高并发CDN场景中采用外层锁+内层锁的分级架构，通过哈希算法将80个Worker进程的竞争分散到多个外层锁，最终仅少数进程进入内层锁竞争

### 缓存行乒乓效应

缓存行乒乓效应（Cache Line Ping-Pong Effect）是多核系统中因多个处理器频繁读写同一缓存行导致的高性能损耗现象，其本质是缓存一致性协议（如MESI）的频繁触发。以下从原理、影响及优化策略展开分析：
共享缓存行的争用

当多个CPU核心同时访问同一缓存行内的数据时（即使数据不同），会触发缓存一致性协议（如MESI）的广播机制。例如：

假共享（False Sharing）：不同线程访问同一缓存行中的独立数据（如数组相邻元素），导致缓存行被频繁无效化

硬件机制的限制

缓存行粒度：缓存以固定大小（通常64字节）的缓存行为单位管理，无法区分同一缓存行内的不同数据
MESI协议开销：每次缓存行状态变更（如从Modified到Invalid）需跨核心同步，导致总线带宽和CPU周期浪费

二、性能影响

吞吐量下降

单核场景下操作耗时若为1单位，双核可能因乒乓效应增至2-3单位

例如，全局计数器在多核下的吞吐量可能随线程数增加而指数级下降

资源浪费

缓存带宽占用：缓存行频繁传递占用内存控制器带宽；

CPU空闲等待：核心因等待缓存行同步而无法执行有效指令

三、优化策略

将高频访问的变量独占一个缓存行，避免假共享。例如：