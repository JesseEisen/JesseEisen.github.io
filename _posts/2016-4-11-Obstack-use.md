---
title: GNU Obstack Use
updated: 2016-04-11 12:00
---

## obstack use

obstack主要的功能是用来申请一块大内存，每次从这个内存中分配内存，在内存不够时，会继续自动的扩大内存，
这个适合于在长时间使用一个内存，且需要一次性释放的。

主要的结构：
```c
    struct obstack
```

包含的成员：

+ chunk  <-- The objects in the obstack are packed into large blocks


## 使用前

+ 直接引用头文件：
```c
    #include <obstack.h>
```

+ 如果定义了 `obstack_init` 这个宏
仍需要定义如下的宏
```c
    #define obstack_chunk_alloc  xmalloc
    #define obstack_chunk_free   free
```

**`int obstack_init(struct obstack *obstack_ptr)`**

这个函数用来初始化一个`obstack_ptr` 这个函数是调用了`obstack_chunk_alloc`。如果分配失败，那么会调用`obstack_alloc_failed_handler` 这个函数永远只返回1（意味着不需要检查返回值）

可以这么使用：
```c
    static struct obstack myobstack;
    ....
    obstack_init(&myobstack);
```

还可以使用：
```c
    struct obstack *myobstack_ptr
    = (struct obstack *)malloc(sizeof(struct obstack));
    obstack_init(myobstack_ptr)
```


变量：`obstack_alloc_failed_handler`

这个变量是指向一个函数的，当这个`obstack` 使用 `obstack_chunk_alloc`失败的时候，这个函数一般会调用`exit` `longjmp`或者不返回之类的。

```c
void my_obstack_alloc_failed(void)
...
obstack_alloc_failed_handler = &my_obstack_alloc_failed;
```


## 在obstack上分配空间

使用`void * obstack_alloc(struct obstack *obstack_ptr, int size)`
这个函数在一个obstack上分配一个未初始化的块，大小是size， 返回这个块内存的地址。

如果需要的话，会调用`obstack_chunk_alloc` 来分配一个新的chunk内存，如果分配失败了的话，那么会调用`obstack_alloc_failed_handler`这个函数进行处理。

```c
    struct obstack string_obstack;

    char * copystring(char *string)
    {
        size_t len = strlen(string) + 1;
        char *s = (char *)obstack_alloc(&string_obstack, len);
        memcpy(s, string ,len);
        return s;
    }
```

分配一个有指定内容的block，可以使用`obstack_copy`
`void *obstack_copy(struct obstack *obstack-ptr, void *address, int size)`

复制size大小的内容，内容是从address开始的部分。结果作为返回值。 失败的时候会调用`obstack_alloc_failed_handler`

另一个和`obstack_copy`类似的函数`obstack_copy0`函数，会在结尾加上一个`\0`作为结尾。

```
    char * obstack_savestring(char *addr, int size)
    {
        return obstack_copy0(&myobstack,addr,size);
    }
```

## 释放空间

使用`obstack_free`函数去释放掉在obstack上分配的空间。 释放掉一个，会自动的释放掉在同一个obstack上的最近分配的空间。

`void obstack_free(struct obstack *obstack_ptr, void *object)`
如果object是NULL，那么会释放掉所有的内容。 如果不是NULL，那么object就需要是一个在obstack上分配的指针。

如果使用的是NULL的话，结果就是一个未初始化的obstack， 为了能够进一步使用，最好是用最开始在obstack上分配的那个地址作为object传入函数。

```c
    obstack_free(obstack_ptr, first_object_allocated_ptr)
```

如果一个chunk上的所有object都被free了，那么这个chunk也会被自动的free。
如果其他的obstack或者非obstack分配，可以重复使用其他的chunk。

## 参考
[GNU obstacks manual](http://www.gnu.org/software/libc/manual/html_node/Obstacks.html#Obstacks)


