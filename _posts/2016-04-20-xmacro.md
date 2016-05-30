---
title: Xmarco Introduction
updated: 2016-04-20 12:00
---

## General Introduction 
这是一个最早被用在汇编上的一个技术，用来实现批量产生一些代码的目的。使用x-macro有两种形式，这两种形式分别为：独立头文件，以及同一个文件。

## 独立头文件
将我们要用于产生代码的部分放置于一个头文件中：

```c
//in enum_def.h
    #define X(red,"red")
    #define X(green,"green")
    #define X(gray,"gray")
```

然后在我们的`.c`文件中实现相关的代码部分如下：

```c
    typedef enum{
        #define X(member,value) member,
        #include "enum_def.h"
        #undef X
    }colors;
```

同样我们可以定义一个数组：

```c
    char *color[] = {
        #define X(member,value) value,
        #include "enum_def.h"
        #undef X
    }
```

注意在定义头文件的时候，如果同时在一个文件中有多处引用了该头文件，则不要使用`#ifndef _xxx_H` 如果使用了则会让头文件无法多次引用，同时我们还可以将宏扩展一下，以方便于自定义值

```c
    #define X(red, , "red")
    #define X(green, =3, "green")
    #define X(blud, , "blue")
```

那么可以扩展成在定义枚举的时候便可自由的设置值了。

```c
    typedef enum{
        #define X(a,b,c) a b,
        #include "enum_def.h"
        #undef X
    }colors;
```

放在文件中，这个定义适合比较短小的内容，如果内容比较多。在预处理的时候都替换掉，会让文件比较大，有很多无用的宏内容存在。

## 同一个文件

在同一个文件，那么会有一些不同的技巧可以使用：

比如我们可以定义一个结构体：

```c
    #define EXPAND_STRUCT \
        EXPAND_MEMBER(x, int) \
        EXPAND_MEMBER(y, int) \
        EXPAND_MEMBER(z, double)
```

然后我们可以使用如下的定义来实现struct定义：

```c
    typedef struct {
        #define EXPAND_MEMBER(member, type)  type member;
        EXPAND_STRUCT
        #undef EXPAND_MEMBER
    }
```

不过下面这段代码也是不错的：

```c
    #define EXPAND_STRUCT(_, ...)\
        _(x, int ,__VA_ARGS__) \
        _(y, int, __VA_ARGS__) \
        _(z, double, __VA_ARGS__)
```

同时定义如下的宏：

```c
    #define EXPAND_MEMBER (member,type) type member;
```

然后便可使用：

```c
    typedef struct{
        EXPAND_STRUCT(EXPAND_MEMBER,)
    }structs;
```

不仅如此，这个宏还可以继续定制：

```c
    #define FORMAT_(type)  FORMAT_##type
    #define FORMAT_int   "%d"
    #define FORMAT_double "%g"

    #define PRINT_MEMBER(member, type) \
        printf("%s", FORMAT_(type) "\n", #member,obj.member);
```
使用则为：

```c
    ...  //定义的结构体
    EXPAND_STRUCT(PRINT_MEMBER,sturcts)

```

X-macro的基本使用就是这些了。

### reference
1.[wikibook](https://en.wikibooks.org/wiki/C_Programming/Preprocessor#X-Macros)
2.[wikibook again](https://en.wikibooks.org/wiki/C_Programming/Serialization)
