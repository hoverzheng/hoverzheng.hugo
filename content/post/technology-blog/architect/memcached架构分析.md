---
title: "Memcached架构分析"
date: 2018-03-18T20:52:36+08:00
draft: false
categories: "Architect"
tags: ["memcached", "架构分析"]
---

## 介绍
本文通过阅读memcached的源代码，剖析了memcached的主流程处理的架构设计。包括主流程的线程模型和网络事件处理机制。
该架构通过一个主线程来分发客户端的连接任务，worker接管连接后负责处理客户端的各种请求。
注：本文是我在2013年写的[一篇博客](http://blog.chinaunix.net/uid-20498361-id-3838477.html)整理而写成的。

## memcached简介
memcached是一个基于内存的缓存系统，具有很高的性能，广泛的使用于网站的各种内容的缓存。一般用它来缓存一些较小的内容(<1M)，这是由它的内存限制和分配策略决定的。
memcached系统是利用libevent事件驱动库实现的一个多线程K/V缓存系统，支持TCP和UDP。

在memcached中，可以大致把线程分为两种：

* 一种是分发线程(主线程)
* 一种是worker线程，也就是进行后续命令处理的线程。

## 主架构实现分析

memcached的内部实现的架构，如下图所示：
![说明](/post/technology-blog/architect/img/memcached_archtect.png)

主线程(main thread)会和每个worker线程都建立一个管道(pipe)，当client连接memcached时，会先把连接请求发送给主线程，并和主线程完成连接的建立，让后主线程会选择一个管道，也就是选择了一个worker thread发送一个字符: ‘c’，并把创建一个新的连接实体放到连接队列中，此时阻塞在管道读区端的worker线程被唤醒，worker线程从连接队列中取出连接实体，并对完成连接的socket注册读事件处理函数，最后进入命令处理流程来处理client端的发送命令和各种连接异常的事件。


### 主线程
在memcached中主线程负责监听client端的连接请求，接收并创建和client的tcp连接。并选择一个worker线程来处理该client端的后续请求。

#### 主线程初始化
初始化时memcached的主线程主要完成以下几项工作：

* 创建N个worker线程，并和每个线程创建一个双向管道(pipe)
* 为每个woker线程创建保存conn_queue_item对象的队列：new_conn_queue
* 为管道的读端fd(代码中的变量是：notify_receive_fd)，注册读事件处理函数：thread_libevent_process
* 创建conn实体，并初始化状态为conn_listening
* 创建绑定(bind)的socket:sfd，并为该socket注册读事件处理函数：event_handler

### worker线程
worker线程负责处理client端发送的命令，连接超时等。

#### worker线程初始化
worker线程的初始化完成的工作如下：

* 为管道的读fd:notify_receive_fd，注册读事件监听函数thread_libevent_process
该函数会阻塞在管道的notify_receive_fd描述符上进行读取。


### 事件处理

#### 处理流程概要
处理客户端创建连接请求的处理流程如下：

* 主线程和客户端完成tcp连接的建立
* 主线程创建conn对象，并把该对象放到连接队列中
* 主线程向worker线程的管道中发送字符：'c'
* worker线程从管道中读取命令'c'，并从连接队列中取出一个conn实体
* worker线程创建一个新的conn实体，并把最新的sfd(已完成tcp连接的socket)读事件注册到event_handler
* event_handler会调用drive_machine处理客户端的各种命令和事件

#### 详细说明
当客户端向memcached发送连接请求时会触发sfd的读事件处理函数event_handler的执行。该函数会调用drive_machine函数，在该函数中完成与client端的tcp连接建立。

当主线程接收到客户端的连接请求时，会选择一个worker线程的管道，选择哪一个worker线程呢，规则如下：
```
int tid = (last_thread + 1) % settings.num_threads;
```
可以看到，其实是按轮训的方式来选择worker线程。这样可以保证每个worker线程服务的client的数量基本相同。

通过以上方式，选择好一个线程的管道后，创建一个conn_queue_item对象，并把该对象放到连接队列new_conn_queue中，然后向该管道发送'c'字符，该字符表示客户端要创建连接了。
此时会触发worker线程的管道读事件处理函数thread_libevent_process的执行，当worker线程的从管道中接收到该字符时，会从事件队列中取出conn_queue_item对象，根据该实体的信息创建一个新的conn实体，并为已经完成连接的socket描述，注册event_handler读事件处理函数。

此时已经由worker线程接管了client的连接，后续的客户端发送的命令都会由event_handler事件处理函数进行处理。

event_handler函数会最终调用driver_machine()函数来处理客户端所有的请求。


## 小结
通过该文的分析，我们知道memcached是通过一个线程池来处理客户端请求，和redis的单进程架构不一样。这种基于libevent的多线程架构，可以把client端的负载均匀分担到多个处理线程中，使用的还是比较多。

参考：

* [github.com/memcached](https://github.com/memcached/memcached)
* [我的2013年博客](http://blog.chinaunix.net/uid-20498361-id-3838477.html)
