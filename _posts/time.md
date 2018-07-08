---
layout: post
title: Time In C
date: 2018-03-28 18:00:00
description: timer use in c
---

C 语言提供了一些基本的 time 相关函数，通过这些函数我们可以获取到相关的时间信息，这些信息可以用于 Log 输出，或者性能测试时的统计。所以本篇博客主要是来整理一下 C 中的时间函数的使用，同时还会涉及到简单的 timer 的设计和使用。

### 获取时间

通常我们获取时间是通过 `time(3)` 函数，出于简单的时间统计，我们可能会这样做：

```c
t1 = time(NULL);
//do something
t2 = time(NULL);

double elapsed = difftime(t2, t1);
```

如果中间部分的代码耗时并不多，`elapsed` 往往就是 0.0 ，因为 time 函数只精确到秒。 difftime 也是一个系统调用，比较两个日历时间的差值，所以当耗时在毫秒甚至更低的时候，往往是统计不出来的。

所以更加精确的时间可以通过 `gettimeofday` 来获取。 这个函数支持到微秒，首先看下函数签名：

```c
int gettimeofday(struct timeval *restrict tp, void *restrict tzp)
```

其中 `struct timeval` 里面定义了所支持的精度：

```c
struct timeval {
             time_t       tv_sec;   /* seconds since Jan. 1, 1970 */
             suseconds_t  tv_usec;  /* and microseconds */
     };
```
我们能看到第二个参数表示的是微秒，不过我们知道微妙和秒之间的换算是 1 s = 10^6 us, 所以如果想要计算出两个时间的差值，可以将这个结构体中的 `tv_sec` 转成 us，并加上微秒的值。 此时需要将结果保存到 `long long` 中，否则会发生溢出。

一般将精度提高到微秒级别的，说明操作的间隔应该是很短的，如果时间很长，一般情况下不太建议使用微秒操作，因为转换出来的结果都会比较大，计算上会稍微慢些。所以建议折衷的做法，转成毫秒进行比对。