---
abbrlink: '1893'
categories:
  - - 日常学习
date: '2023-10-9 21:09:31'
description: redis是一个开源的、内存型的、支持多种数据结构的存储系统，常用来做缓存、数据库、消息代理，本文为作者学习redis的笔记
tags:
  - redis
  - nosql
title: redis日常学习(持续更新)
top_img: 'https://img.lazysun.me/202302152349787.png'
cover: 'https://img.lazysun.me/cover-202310092118802.webp'
updated:
aplayer:
aside:
comments:
highlight_shrink:
katex:
keywords:
mathjax:
type:
---

# Reids持久化机制

## 自测

**怎么保证 Redis 挂掉之后再重启数据可以进行恢复？** ⭐⭐⭐⭐⭐



**什么是 RDB 持久化？** ⭐⭐⭐⭐⭐



**什么是 AOF 持久化？** ⭐⭐⭐⭐⭐



**Redis 4.0 对于持久化机制做了什么优化？** ⭐⭐⭐⭐



## 知识点

使用缓存的时候，我们经常需要对内存中的数据进行持久化也就是将内存中的数据写入到硬盘中。大部分原因是为了之后重用数据（比如重启机器、机器故障之后恢复数据），或者是为了做数据同步（比如 Redis 集群的主从节点通过 RDB 文件同步数据）。



Redis支持三种持久化方式: RDB, AOF, RDB和AOF混合持久化

### RDB方式持久化

默认采用的持久化方式

Redis 可以通过创建快照来获得存储在内存里面的数据在 **某个时间点** 上的副本。Redis 创建快照之后，可以对快照进行备份，可以将快照复制到其他服务器从而创建具有相同数据的服务器副本（Redis 主从结构，主要用来提高 Redis 性能），还可以将快照留在原地以便重启服务器的时候使用。

Redis 提供了两个命令来生成 RDB 文件，分别是 save 和 bgsave，save会堵塞主进程，而bgsave则是fork出一个子进程执行快照备份。
redis.conf相关配置

```cobol
save 900 1           #在900秒(15分钟)之后，如果至少有1个key发生变化，Redis就会自动触发bgsave命令创建快照。

save 300 10          #在300秒(5分钟)之后，如果至少有10个key发生变化，Redis就会自动触发bgsave命令创建快照。

save 60 10000        #在60秒(1分钟)之后，如果至少有10000个key发生变化，Redis就会自动触发bgsave命令创建快照。
```

bgsava 过程中，Redis 依然**可以继续处理操作命令**的，也就是数据是能被修改的。关键的技术就在于**写时复制技术**(***Copy-On-Write, COW***)。

执行 bgsava 命令的时候，会通过 fork() 创建子进程，此时子进程和父进程是共享同一片内存数据的，因为创建子进程的时候，会复制父进程的页表，但是页表指向的物理内存还是一个。

![img](https://img.lazysun.me/202310092117416.webp)

只有在发生修改内存数据的情况时，物理内存才会被复制一份。

![img](https://img.lazysun.me/202310092116866.webp)

当主线程和子线程都只是读共享数据时，内存没有任何变化，但是如果主线程发生了写操作，如主线程要**修改共享数据里的某一块数据**（比如键值对 A）时，就会发生写时复制，于是这块数据的**物理内存就会被复制一份（键值对** **A'）**，然后**主线程在这个数据副本（键值对A'）进行修改操作**。与此同时，**bgsave 子进程可以继续把原来的数据（键值对** **A）写入到 RDB 文件**。

写时复制可以保证bgsave生成的RDB文件是某个时间点的全量快照，但是再某种极端情况下，如**键值对被大量修改**，会复制出大量副本，占用许多内存空间。因此，在写操作比较频繁时，需要监控redis的内存变化

### AOF方式持久化

AOF 持久化的实时性更好。默认情况下 Redis 没有开启 AOF（append only file）方式的持久化（Redis 6.0 之后已经默认是开启了），可以通过 `appendonly` 参数开启：

```cobol
appendonly yes
```

开启AOF后，每执行一条修改数据的命令后都会将该命令写入到AOF缓冲区`server.aof_buf`中，然后再写入AOF文件中，最后根据fsync策略决定何时进行刷盘。

#### AOF的基本流程

1. 命令追加(appen)：每一条修改数据的命令都会被添加到AOF缓冲区
2. 文件写入(write)：由AOF缓冲区写入AOF文件，使用`write`函数，此时数据被写入系统内核缓冲区，需要等待系统调度或者 `fsync` 方法强制刷新缓冲区，同步到磁盘
3. 文件同步(fsync)：AOF 缓冲区根据对应的持久化方式（ `fsync` 策略）向硬盘做同步操作。这一步需要调用 `fsync` 函数（系统调用）， `fsync` 针对单个文件操作，对其进行强制硬盘同步，`fsync` 将阻塞直到写入磁盘完成后返回，保证了数据持久化。
4. 文件重写(rewrite)：AOF文件过大时，会进行压缩。
5. 重写加载(load)：重启redis时，读取AOF文件的数据进行恢复

#### AOF持久化方式(fsync策略)

1. **appendfsync always**：每一次调用write方法后调用fsync同步到磁盘，会严重降低redis的性能。
2. **appendfsync everysec**：执行命令，调用 write 方法后立即返回，每隔1s调用fsync进行刷盘。
3. **appendfsync no**：执行命令，调用 write 方法后立即返回，让系统自行同步到磁盘。

#### AOF重写

当 AOF 变得太大时，Redis 能够在后台自动重写 AOF 产生一个新的 AOF 文件，这个新的 AOF 文件和原有的 AOF 文件所保存的数据库状态一样，但体积更小。

![image-20231009205745562](https://img.lazysun.me/202310092114801.png)

#### AOF校检机制

AOF 校验机制是 Redis 在启动时对 AOF 文件进行检查，以判断文件是否完整，是否有损坏或者丢失的数据。这个机制的原理其实非常简单，就是通过使用一种叫做 **校验和（checksum）** 的数字来验证 AOF 文件。这个校验和是通过对整个 AOF 文件内容进行 CRC64 算法计算得出的数字。如果文件内容发生了变化，那么校验和也会随之改变。因此，Redis 在启动时会比较计算出的校验和与文件末尾保存的校验和（计算的时候会把最后一行保存校验和的内容给忽略点），从而判断 AOF 文件是否完整。如果发现文件有问题，Redis 就会拒绝启动并提供相应的错误信息。AOF 校验机制十分简单有效，可以提高 Redis 数据的可靠性。

#### RDB-AOF混合方式

fork出来的子进程将内存副本全量已RDB的方式写入到AOP临时文件，再将重写缓冲区的增量以AOF的方式添加到临时文件末尾，最后将RDB + AOF混合的AOF文件取代旧的AOF文件。


# Redis线程模式

## 自测

**Redis 单线程模型了解吗？ 既然是单线程，那怎么监听大量的客户端连接呢？、为什么 Redis 这么快？**⭐⭐⭐⭐⭐



**Redis6.0 之前为什么不使用多线程？** ⭐⭐⭐



**Redis6.0 之后为何引入了多线程？** ⭐⭐⭐⭐



## 单线程模式

Redis采用**Reactor模式**的网络模型

主线程负责数据(键值对)的读写和网络IO传输，而多线程负责大key的删除和后台快照备份等。Redis在处理客户端的请求时包括获取（socket读）、解析、执行、内容返回（socket写）等都由一个**顺序串行的主线程处理**，

### 为什么选择单线程

- 开发和维护更简单
- 也可以并发处理客户端的请求(IO多路复用和非堵塞IO)
- 主要瓶颈是内存和网络带宽，而非CPU

### 为什么这么快

- 基于内存操作
- 数据结构简单，数据操作基本上都是O(1)复杂度
- IO多路复用和epoll函数
- 避免上下文切换

## 浅谈IO多路复用和NIO

因为IO多路复用和NIO有点类似，这里就放在一起谈了

#### NIO（NonBlocking IO）

`NIO`这里要进行区分：

1. JAVA中代表 `New IO`。
2. 系统操作层面代表`NonBlocking`

这里所说的并非java的NIO(New IO)

同步非阻塞 IO 模型中，应用程序会一直发起 read 调用，等待数据从内核空间拷贝到用户空间的这段时间里，线程依然是阻塞的，直到在内核把数据拷贝到用户空间。**应用程序不断进行 I/O 系统调用轮询数据是否已经准备好的过程是十分消耗 CPU 资源的**，会造成CPU空转。

#### IO多路复用 

> 1. `select`函数 - > 同步多路复用`IO`方法
>     返回值中会返回三个集合数据包含 `readfds`,`writefds`以及`exceptfds`文件描述符集合。(`fds`有`1024`限制)
> 2. `poll`函数  - > 同步多路复用`IO`方法
>     返回值中返回对应有响应的`fds`集合。
> 3. `epoll_create`函数  - > 打开`epoll`文件描述符
>     该方法将会返回一个`epoll`实例（该实例用于接收`IO`事件通知）。
> 4. `epoll_ctl`函数  - > `epoll`描述符的控制接口
>     接收`fd`绑定对应事件到`epoll`实例上。
> 5. `epoll_wait`函数  - > 等待`epoll`文件描述符上`IO`事件
>     返回对应有`IO`事件的`fd`。
>
>   上述方法中，前两个都是基于多路复用进行的，下面三个方法则完全归属于`epoll`方式（个人觉得他也使用到了多路复用，但是更偏向于`Reactor`模型）。

IO多路复用，是建立在内核上提供的多路分离函数select基础之上的，使用select函数可以避免同步非阻塞IO模型中轮询等待的问题；用户将需要进行IO操作的socket添加到select中，然后阻塞等待select系统调用返回。当数据到达时，socket被激活，select函数返回，用户线程发起read请求，读取数据并继续执行。使用select函数进行IO请求与同步阻塞模型并无太大区别，甚至多添加监视socket，select函数额外操作，使用优势主要在于用户可以在一个线程内同时处理多个socket的IO请求，用户可以注册多个socket，然后不断调用select读取被激活的socket，即可达到在同一个线程内同时处理多个IO请求的目的。

和NIO的区别是select获取到的都是有数据的。

## redis事件机制

Redis中的事件驱动库只关注网络IO，以及定时器。该事件库处理下面两类事件：

- 文件事件(file event)：用于处理 Redis 服务器和客户端之间的网络IO。
- 时间事件(time eveat)：Redis 服务器中的一些操作（比如serverCron函数）需要在给定的时间点执行，而时间事件就是处理这类定时操作的。

![事件管理器示意图](https://img.lazysun.me/202310101950780.png)

`aeEventLoop`是整个事件驱动的核心，它管理着文件事件表和时间事件列表，不断地循环处理着就绪的文件事件和到期的时间事件。

尽管多个文件事件可能会并发地出现，但I/O多路复用程序总是会将所有产生的套接字都放到同一个队列(也就是后文中描述的`aeEventLoop`的`fired`就绪事件表)里边，然后文件事件处理器会以有序、同步、单个套接字的方式处理该队列中的套接字，也就是处理就绪的文件事件。

一次 Redis 客户端与服务器进行连接并且发送命令的过程如上图所示。

- 客户端向服务端发起建立 socket 连接的请求，那么监听套接字将产生 AE_READABLE 事件，触发**连接应答处理器**执行。处理器会对客户端的连接请求进行应答，然后创建客户端套接字，以及客户端状态，并将客户端套接字的 AE_READABLE 事件与**命令请求处理器**关联。
- 客户端建立连接后，向服务器发送命令，那么客户端套接字将产生 AE_READABLE 事件，触发**命令请求处理器**执行，处理器读取客户端命令，然后传递给相关程序去执行。
- 执行命令获得相应的命令回复，为了将命令回复传递给客户端，服务器将客户端套接字的 AE_WRITEABLE 事件与**命令回复处理器**关联。当客户端试图读取命令回复时，客户端套接字产生 AE_WRITEABLE 事件，触发命令**回复处理器**将命令回复全部写入到套接字中。

事件处理：

`aeMain`函数以一个无限循环不断地调用`aeProcessEvents`函数来处理所有的事件。

```c
void aeMain(aeEventLoop *eventLoop) {
    eventLoop->stop = 0;
    while (!eventLoop->stop) {
        /* 如果有需要在事件处理前执行的函数，那么执行它 */
        if (eventLoop->beforesleep != NULL)
            eventLoop->beforesleep(eventLoop);
        /* 开始处理事件*/
        aeProcessEvents(eventLoop, AE_ALL_EVENTS|AE_CALL_AFTER_SLEEP);
    }
}
```



> 参考文章
>
> 1. [请不要再说NIO和多路复用IO是同一个东西了（内含BIO、NIO、多路复用、Netty、AIO案例测试代码）_io多路复用和nio的关系_默辨的博客-CSDN博客](https://blog.csdn.net/qq_44377709/article/details/123724519)
> 2. [图解BIO、NIO、AIO、多路复用IO的区别-腾讯云开发者社区-腾讯云 (tencent.com)](https://cloud.tencent.com/developer/article/1745899)
> 3. [【Redis】高级篇： 一篇文章讲清楚Redis的单线程和多线程 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/646111642)
> 4. [Redis常见面试题总结(上) | JavaGuide(Java面试 + 学习指南)](https://javaguide.cn/database/redis/redis-questions-01.html#redis-单线程模型了解吗)
> 5. [Redis 事件机制详解 | 程序员历小冰 (remcarpediem.net)](http://remcarpediem.net/article/1aa2da89/)

# Redis内存管理

待更新.....

# Redis数据结构类型

待更新.....