---
title: IO模型笔记
date: 2018-05-20 10:19:15
tags: OS
categories: 笔记
---

# IO Model
常见IO模型
- blocking IO
- nonblocking IO
- IO mutiplexing(select & poll)
- signal driven IO
- asynchronous IO(the POSIX aio_functions)

> 因为七牛bucket删除了，图示没了，重新作图日后补上

### 一个基本的IO，它会涉及到两个系统对象，一个就是系统内核，另一个是调用这个IO的对象。当一个read操作发生时，会经历以下阶段：
- 通过read系统调用向内核发起读请求
- 内核向硬件发送读指令，并等待读就绪
- 内核把将要读取的数据复制到描述符所指向的内核缓存区
- 将数据从内核缓存区拷贝到用户进程空间中

### 同步与异步
- 同步和异步关注的是消息通信机制
- 注：同步IO过程由进程处理，异步IO交由内核处理IO

### 阻塞与非阻塞
- 阻塞和非阻塞关注的是程序在等待调用结果时的状态

1，2，3，4属于同步IO，5属于异步IO

AIO异步非阻塞IO，适用于连接数多且IO时间长的架构，如相册服务器，JDK7开始支持

NIO同步非阻塞IO，适用于连接数多且轻操作（IO?）的架构

BIO同步则色IO，适用于连接数少且固定的架构

一般来说，IO主要有两种情况（服务器）：一是来自网络的IO，二是文件的IO。windows提供异步IO，Linux提供epoll模型给网络IO，文件IO则提供AIO

### epoll,select/poll
- 本质上都是同步IO，自己处理IO过程，但是epoll优化了轮询操作，使用callback机制响应。
- 额外提供Edge Triggered，用户空间可能缓存IO状态，减少epoll-wait／epoll-pwait调用

level triggered & edge triggered
- LT事件不会丢弃，只要读buffer里面有数据可以让用户读，就会不停的通知。而ET则只发在事件发生之时通知。

### select缺点：
- 每次调用，都需要把fd集合从用户态拷贝到内核态，fd多事件开销大
- 每次都要遍历进入内核的fd，开销也很大
- select支持fd数量太小，默认1024
- poll与select只在fd集合的结构上面有区别

### epoll对select缺点的改进
- 新事件注册到epoll句柄中，会把所有的fd拷贝进内核，而不是在epoll_wait时重复拷贝
- 为fd指定回调函数，设备就绪，调用回调函数唤醒等待者，并将fd加入就绪链表
- 最大的限制很大程度上跟系统内存大小有关
- 消息传递：mmap加速

# 总结
- select、poll都需要轮询遍历所有fd，而epoll只需要读取就绪链表
- select、poll每次调用都copy一次fd，并往设备队列上挂。而epoll只要copy一次fd，并只往自己的等待队列上挂一次即可
