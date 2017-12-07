---
layout: post
title: go file read
date: 2017-10-06 18:00:00
description: go read relate
---

最近在写一个Go的文件搜索程序，涉及到一些文件的读写。所以趁此机会来总结一些。根据不同的场景选择最合适的读写函数。

 <hr>

#### 打开文件

`os` 包里提供了一个`Open`函数，函数签名为：

```
func Open(name string) (*File, error)
```

函数返回一个`*File`， 而`File` 实现了`Read`和`Write`这两个方法， 所以Open的返回值可以直接传入以`io.Reader`和`io.Writer`为参数的函数中。
不过于对于`os.Open`而言，返回的File只能用于读（Os.O_RDONLY）。 因此在需要写的情况下，直接`Open`这个函数就不太适合。 可以使用如下：

```
func OpenFile(name string, flag int, perm FileMode) (*File, error)
```

函数中的`flag`和`perm` 这两个参数，可以参考linux下的`open(2)`,通过这两个指定文件打开的权限和读写设置.


#### 读写文件

在读写文件的方面,可以有多种方式,可以希望都出来是`[]byte` 可以是`string`. 可以按行读, 也可以一次性读完整个文件. 因此我们可以使用不同的包提供的相关函数.
这些包有`io`,`ioutil`,`bufio`以及`File`本身的一些方法.
