---
abbrlink: b151
categories:
  - - note
description: redis线程模式
tags:
  - redis
title: 浅谈redis(2)-线程模式
top_img: 'https://img.lazysun.me/202307301447764.jpg'
cover: 'https://img.lazysun.me/202312161102075.webp'
date: 2023-12-16 11:01:00
updated:aplayer:
aside:
comments:
highlight_shrink:
katex:
keywords:
mathjax:
type:
---
## Redis线程模式
## 单线程模式

Redis采用**Reactor模式**的网络模型

主线程负责数据(键值对)的读写和网络IO传输，而多线程负责大key的删除和后台快照备份等。Redis在处理客户端的请求时包括获取（socket读）、解析、执行、内容返回（socket写）等都由一个**顺序串行的主线程处理**

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
>    返回值中会返回三个集合数据包含 `readfds`,`writefds`以及`exceptfds`文件描述符集合。(`fds`有`1024`限制)
> 2. `poll`函数  - > 同步多路复用`IO`方法
>    返回值中返回对应有响应的`fds`集合。
> 3. `epoll_create`函数  - > 打开`epoll`文件描述符
>    该方法将会返回一个`epoll`实例（该实例用于接收`IO`事件通知）。
> 4. `epoll_ctl`函数  - > `epoll`描述符的控制接口
>    接收`fd`绑定对应事件到`epoll`实例上。
> 5. `epoll_wait`函数  - > 等待`epoll`文件描述符上`IO`事件
>    返回对应有`IO`事件的`fd`。
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
> 5. [Redis 事件机制详解 | 程序员历小冰 (remcarpediem.net)](