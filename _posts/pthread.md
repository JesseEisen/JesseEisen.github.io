---
layout: post
title: Thread In Linux
date: 2017-10-15 18:00:00
description: fundmental of pthread
---



实际应用中很多地方需要使用的多线程模型，线程的定义如下：

> 线程是[操作系统](https://zh.wikipedia.org/wiki/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F)能够进行运算[调度](https://zh.wikipedia.org/wiki/%E8%B0%83%E5%BA%A6)的最小单位。它被包含在[进程](https://zh.wikipedia.org/wiki/%E8%BF%9B%E7%A8%8B)之中，是[进程](https://zh.wikipedia.org/wiki/%E8%BF%9B%E7%A8%8B)中的实际运作单位。一条线程指的是[进程](https://zh.wikipedia.org/wiki/%E8%BF%9B%E7%A8%8B)中一个单一顺序的控制流，一个进程中可以并发多个线程，每条线程并行执行不同的任务。

本文主要整理 POSIX 线程相关的内容，主要涉及到线程的基本使用、线程同步、以及线程间通信的内容。

<hr>

###  基本概念

每一个线程都有一个标识，这个和每个进程都一个的进程号类似。 我们通过 *pthread_t* 定义一个线程号，将这个参数传入线程创建函数中。线程创建函数的接口为：

```c
int pthread_create(pthread_t *thread, const pthread_attr_t *attr,
                          void *(*start_routine) (void *), void *arg);
```

创建一个线程所需要指定的参数包括：线程id，线程属性，线程处理函数以及传入处理函数中的参数。 首先我们来了解下函数的返回值。**pthread 相关的函数都是成功返回 0， 失败直接返回 error number，而不是去设置 errno**。 

线程创建后，其执行的先后顺序并不能保证，因为线程之间存在着竞争，需要一些方法来避免这些竞争。这个后续再说。

线程实际上是可以通过 `pthread_self()` 获取自己的线程id，同时在比较两个线程id时候，使用`pthread_eqaul(tid1, tid2)` 是更为稳妥一些的。





