---
layout: post
title: C 语言中的 Fat Pointer 
subtitle: "一种隐藏控制信息的指针方法"
author: "L.K."
header-img: "img/inpost/post-c-pointer-bg.jpg"
tags:
    - C
    - Programming
---

在正式讨论这篇文章的主角前，我想讨论一下在 C 编程中经常容易出现的错误，就是当数组传入函数后，实际该数组就退化成指针，而作为数组所具有的维度的属性就丢失了。

通常我们会使用下面的一个宏来计算数组的长度：

```c
#define nelem(x) (sizeof(x)/sizeof((x)[0]))
```

注意这个宏统计的是当前数组所有元素的个数，并不是当前数组被赋值了多少。当我们对函数参数使用该宏的时候，往往得到的结果并不是我们期望的。比如：

```c
size_t getsize(char *s){
    return nelem(s);
}
```

这个结果往往是当前系统上一个指针的长度。对于这种情况，很容易出现溢出或者 segmentation fault。 这也许是指针和数组最直观的不同，很多情况下我们很容易忽略这个。

规避这个问题最简单的方式就是多加一个长度的参数，或者我们使用如下的方式：

```c
struct Element {
    int size;
    int buf[];
}
```

定义一个带有变长数组的结构体，这个结构体所占的大小为仅为第一参数 int 所占的大小，**第二个参数不占结构体空间但是内存紧跟在结构体之后，这就让我们在分配内存的的时候，可以只分配一次内存，即可以使用 buf**。 如果将 buf 定义成指针，则这个指针的地址不一定紧跟在结构体之后，所以一般情况下需要分配两次内存。

当然这种方式你在任何情况下对 buf 使用 nelem 宏，结果都不是你所期望的数组的大小。 其实定义成带有变长数组的方式是一种 trick 的方式。 读过 redis 的源码的都知道，redis 里面的 sds ，这种 hack string 的方式和我们上面定义的这个结果很类似。

redis 通过定义一个 sdshdr 结构，在这个结构中定义了如下内容：

```c
struct sdshdr {
    long len;
    long free;
    char buf[];
};
```
而暴露给其他 API 使用的字符串类型却是： `typedef char *sds;` 。 具体的操作可以简化成下面的代码：

```c
sds newsdslen(void *init, size_t initlen) {
    /* 定义一个 sdshdr 指针并分配相关内存 */
    /* 将 init 中内容复制到 buf 中 */

    return (char *)sh->buf;
}
```

最后返回了是我们常见的字符串，这样做的好处则是可以通用标准库中的 string 操作函数，又可以更好根据 len 和 free 维护当前的 string。

这样的思路其实就是已经是在使用 fat pointer 了。 而使用 fat pointer 的目的是什么呢？ 目的之一就是作为一种对象信息的维护。这个对象的信息在创建一个通用模板函数的情况下比较实用，比如 [Cello](http://libcello.org/home) 这个项目就是大量的使用 fat pointer。将很大一部分的信息隐藏在头部，暴露出来的只是简单的结构体类型。 这个项目作者自己也调侃说：“这是一个使用错误的工具解决错误的问题”。 不过整个代码还是很有学习价值的。

一个简单的 fat pointer 思路是这样的， 我们同样定义一个 header 结构体，这个结构体里面我们只存放一个简单的 type 字段。

```c
struct head {
    void *type;
};
```

Cello 中大量使用了 `void *`， 这个主要是为了实现一个通用的形式。 比如我们要创建一个整型的变量，我们可以这么干：

```c
var header_init(var head, var type) {
  struct Header* self = head;
  self->type = type;
  return ((char*)self) + sizeof(struct Header);
}

#define alloc_stack(T) header_init( \
  (char[sizeof(struct Header) + sizeof(struct T)]){0}, T)

#define $(T, ...) ((struct T*)memcpy( \
  alloc_stack(T), &((struct T){__VA_ARGS__}), sizeof(struct T)))
```

上面的宏是从栈上分配内存，此时我们可以定义这样一个结构体：

```c
void * Int;
struct Int{
    int data;
}
```

此时我们就可以这么使用了：

```c
 void * a = $(Int, 5);
```

这么一通操作看起来很复杂，不过当我们有这样的一个函数，接收的是一个 `void *` 的参数，那这种方式的便利性就出来了。

实际上这个头部可以包含更多的内容，Cello 中包含更加复杂的操作, 使用了 Type classes 的概念，这在函数式编程中比较常见。可以简单的理解为对一系列具有某种性质的 type 的抽象，如果一个type想要成为某个 type class， 就必须成为这个实例并实现 type class 定义的函数。 这么说或许确实挺抽象的，以 Cello 中的一个类型定义为例。

```c
var Help = Cello(Help,
  Instance(Doc,
    Help_Name,       Help_Brief,    Help_Description,
    Help_Definition, Help_Examples, Help_Methods));

#define Instance(I, ...) NULL, #I, &((struct I){__VA_ARGS__})
```

这边 Help 实现了 Doc 这个结构中所定义的所有函数，那么当我们在使用 help 时则可以放到 Doc 的处理函数中。

这种做法，实际上不是特别适合 C，但是这其中的一些处理方式还是很值得我们去学习，这从侧面反应出 Fat pointer 的使用可以达到 Cello 这样抽象的程度，这在让我们在实现一些 API 时可以做到更加的干净。

（全文完）