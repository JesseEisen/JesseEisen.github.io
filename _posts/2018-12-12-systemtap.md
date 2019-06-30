---
layout: post
title: SystemTap Syntax 初探
subtitle: "动态追踪调试技术试水"
author: "L.K."
header-img: "img/inpost/post-dynamic-bg.jpg"
tags:
    - Linux
    - 动态追踪
---

SystemTap 是一个调试追踪工具，提供了一系列可以用来监测 Linux 性能和细节分析的基本部件。本文作为一个 SystemTap 的基本入门梳理。从安装到正式上手自定义一些脚本从而使用到日常编程调试中。

### 安装

SystemTap 是由 RedHat 开发的，因此在 RedHat 系列的 Linux 发行版上一般会或多或少的直接支持。首先需要明确一点的是：SystemTap 是一款调试追踪工具，支撑其正常工作的条件是需要一定的 debuginfo 存在，一般内核是不会自带这些信息的，因此需要手动安装对应版本的 debuginfo 以更好的使用 SystemTap。

由于不同的操作系统所安装的方式不一样，因此在此就不详细描述安装过程,如果你使用的是 RedHat 系的系统，可以参考: [安装 SystemTap](https://sourceware.org/systemtap/SystemTap_Beginners_Guide/using-systemtap.html#installproper) 和 [Install and Setup](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html-single/systemtap_beginners_guide/index#using-setup) 这两个链接。

如果你的系统对应的版本没有对应的 debuginfo，那需要升级一下 kernel。 具体的升级方式参考对应系统的官网升级方式。

### SystemTap 工作方式

在真正深入到 SystemTap 的具体内容前，我们可以了解一下 SystemTap 的基本工作方式，这样更有利于我们使用。

首先，SystemTap 会检查脚本是否合法，同时替换匹配到 tapset 定义。这一步和 C 语言中的预编译过程一样。 接着 SystemTap 将脚本翻译成 C 代码，同时编译成一个 kernel module。完成之后，SystemTap 加载这些模块，并激活脚本中涉及到的探针。这些探针是原先就存在 Kernel 中，使能之后，当有事件发生，对应的探针就会执行我们脚本定义的这些 handler。当 SystemTap 结束后，这些探针会立即被关闭，同时这些 kernel module 也会被销毁。

从上述的工作原理也能看出， SystemTap 是需要 kernel 支持的。 同时，需要做一步代码翻译和编译，所以在运行速度上，最开始时比较耗时的。

### 运行 SystemTap 脚本

脚本的运行一般是通过 `stap` 命令进行，不过并不是所有的用户都能执行该命令， 如果你有超级用户权限，那自然是可以跑的，如果没有我们可以通过将对应的用户添加到组 `stapdev` 和 `stapusr`。 前者的权限比较大一点，后者的权限只能运行 `staprun`。 

staprun 是运行通过 stap 编译生成的 `instrumentation module`。 其不能直接运行 stap 脚本。

此外 stap 还有几个通用的选项需要了解一下，这样会让你更好的使用 SystemTap。

+ `-v` 

提供更加详细的输出，这个在出错的情况下打开比较有用，可以提供更多的信息。比如 `stap -vvv xxx.stp`

+ `-o filename`

将输出到标准输出的内容写到文件中。

+ `-S size,count`

限制文件大小为 size MB，同时限制文件数为 count 个。 这个选项一般用在管理日志文件上。

+ `-x process id`

设置 `target()` 为指定的 process ID。

+ `-c command`

设置 `target()` 为指定的命令。 注意需要指定完整的命令路径。

+ `-e script`

注意 script 指的是字符串型的 systemtap 语句， 而不是脚本文件。

+ `-F`

后台运行脚本并记录结果，一般有两种模式: in-memory flight recorder、 File Fligh Recorder。 从名称上可以直接看出，前者是内容保存在 memory 中， 另一个是保存在文件中。针对后者我们可以这么用 `stap -F -o /tmp/temp.log -S 1,2 temp.stp`。

+ `-l`

用于列出对应模块或者 kernel 中支持的 probe。比如 `stap -l 'kernel.function("*")'` 会列出当前所有 kernel 的 function probe：

### SystemTap 脚本

systemTap 脚本是由两个最基本的内容组成：*event* 和 *handler* 。这两者我们通常组合在一快称为 `probe`，即探针。

我们书写脚本的目的是更好的处理我们的需求，最基本的就是我们根据需要写出我们想要捕获的事件，并给出对应的 handler。

SystemTap 脚本一般以 `.stp` 结尾，脚本内容一般是由 probe 组成。详细的语法如下：

```shell
probe {kernel|module("module-pattern")}.function("function-pattern")[.{call|return[.maxactive(VALUE)]|inline}]
```

通常一个脚本中的 probe 可以有多个，而同一个 probe 可以有多个事件，事件以逗号隔开。如果同一个 probe 指定了多个事件，那么只要有一个 event 发生，SystemTap 就会执行 handler。statement 的语法和 C 语言的语法比较接近。

针对 `call`, `return`, `maxactive`, `inline` 的解释如下：

> call is used to attach entry point non-inlined function, while .inline is used to attach first instruction of inlined function;

>maxactive specifies how many instances of the specified function can be probed simultaneously. You can leave off .maxactive in most cases, as the default (KRETACTIVE) should be sufficient. However, if you notice an excessive number of skipped probes, try setting .maxactive to incrementally higher values to see if the number of skipped probes decreases.

>.return is used for return points of non-inlined functions;

> empty suffix is treated as combination of .call and .inline suffixes.


上述的 `function-pattern` 的具体定义如下：

```shell
function-name[@source-path[{:line-number|:first-line-last-line|+relative-line-number}]]
```

举个例子：

```shell
# 函数名@文件名：指定行
kernel.function("AUDIT_MODE@../security/apparmor/include/policy.h:401")
```

此外，我们也可以自定义函数，这个好处就是可以避免重复代码。格式如下：

```shell
function function_name(arguments) {statements}
probe event {function_name(arguments)}
```

### target variable

首先 probe event 是映射到代码中的实际位置的。 而target variable 的作用是去获取代码中该位置的可见变量的值。 一般我们可以使用 `-L` 列出当前可以查看的 target variable 值。

```shell
stap -L 'kernel.function("vfs_read")'
#输出
kernel.function("vfs_read@../fs/read_write.c:381") $file:struct file* $buf:char* $count:size_t $pos:loff_t*
```

注意，这个功能需要安装对应的 kernel debuginfo 才能支持。 上面的输出是什么意思呢？每个 target variable 都是以 `$` 开头， `:` 后面跟着变量的类型。

对于 target variable 不属于当前 probe 的本地变量，可以使用 `@var("varname@src/file.c")` 来访问。

正常情况下如果我们要访问某个 target variable， 可以使用 `->` ，同时 target variable 是可以链式使用的。 下面有一例子可以用来展示一下：

```
stap -e 'probe kernel.function("vfs_read") {
           printf ("current files_stat max_files: %d\n",
                   @var("files_stat@fs/file_table.c")->max_files);
           exit(); }'
```

除此之外，如果知道了当前数据的地址，可以通过下面的几个函数，更快的获取到内容：

+ kernel_char(address)       
Obtain the character at address from kernel memory.

+ kernel_short(address)   
Obtain the short at address from kernel memory.

+ kernel_int(address)   
Obtain the int at address from kernel memory.

+ kernel_long(address)    
Obtain the long at address from kernel memory

+ kernel_string(address)    
Obtain the string at address from kernel memory.

+ kernel_string_n(address, n)    
Obtain the string at address from the kernel memory and limits the string to n bytes.

有时如果我们不清楚有哪些变量可以输出，或者变量较多时，可以使用一些快捷的预定义变量进行输出：

+ $$vars  当前定义域内存在的所有变量
+ $$locals  $$vars 的子集，只包含本地变量
+ $$parms   $$vars 的子集，只包含函数变量
+ $$return  只可以在 .return 的 probe 中使用，如果被探测的函数是有返回值的话，则打印出该返回值。

在上述的变量后面还可以添加 `$` 或者 `$$` 这两个后缀。以此来输出更加详细的内容，比如 `$$var$` 或者 `$$var$$`


`@defined` 和 `@choose_defined` 是用检查 target variable 是否存在。后者是前者的一个语法糖。什么意思呢？ 一般代码是会改变的，有些变量可能在某些版本上是不存在的，因此我们可以使用 `@defined` 进行判断。惯用法是：

```shell
write_access = (@defined($flags) ? $flags : $write_access)
```

这和 C 语言中的三元符是一个道理。而 `@choose_define` 则是对三元符的一个语法糖。即：

```shell
@defined($a)?$a:$b  等价于 @choose_defined（$a, $b）
```

### event

SystemTap 将事件大致分为：同步事件和异步事件。所谓的同步事件指的是，在内核代码中某一处的指令被执行后，则会触发的事件。异步事件指的是不和内核代码相互绑定的事件，比如 `timer` 事件。

我们大多数情况下需要使用的是同步事件，大致有如下几种：

+ syscall.system_call

这样的事件是捕捉的系统调用，监测的是指定系统调用的动作。当调用了该系统调用，SystemTap就会执行我们指定的 statement。此外，也可以监控系统调用退出的动作，只需要在后面加上 `.return` 即可。比如： `syscall.close` 和 `syscall.close.return`。如果有返回值，一般会存在 `$return` 中。

+ vfs.file_operation

监测的是虚拟文件系统的相关动作。和 syscall 类似，也有一个 `.return` 的动作。

+ kernel.function("function")

监测内核函数 `function`。当内核中任意的线程调用了 `function`，都会触发对应的 event。同样可以指定 `.return` 来捕获退出时的事件。

`function` 中可以使用 `*`,同样如果想追踪指定文件中的函数，可以使用如下的方式：

```shell
probe kernel.function("*@net/socket.c") { }
probe kernel.function("*@net/socket.c").return {}
```

简单说明一下，第一条语句的意思是： 监控内核源文件 net/socket.c 中所有的函数。 第二条语句的意思是当这些函数退出时捕获对应 event。

+ kernel.trace("tracepoint")

现在的 kernel 中会有一些特定的指令事件。这些事件被静态地标记为 tracepoint。比如 `kernel.trace("kfree_skb")`。 这个追踪点就是表明 network 缓存被释放，此时就会触发对应的事件。

+ module("module").function("function")

系统的内核模块在 `/lib/modules/kernel_version` 并以 `.ko` 结尾。 这些模块也是可以被 SystemTap 追踪的。比如：

```
probe module("ext3").function("*") {}
probe module("ext3").function("*").return {}
```

下面说一说异步事件, 异步事件大致有如下几个：

+ begin
+ end

这两个可以类比 awk 中的 BEGIN 和 END 。 在 SystemTap 开始之前，会执行 begin 中的内容，在结束之前会执行 end 中内容。

+ timer

用来周期性的处理内容，这个一般用在定期的输出收集到的内容。一般有如下的衍生：

    + timer.s(xxx)
    + timer.ms(xxx)
    + timer.us(xxx)
    + timer.ns(xxx)
    + timer.hz(hertz)
    + timer.jiffies(jiffies)

一个基本的例子：

```shell

probe begin
{
    printf("Enter\n")
}

probe timer.s(2)
{
    printf("2s later")
    exit()
}

probe end
{
    printf("Exit")
}

```

我们可以通过 `man stapprobes` 查看 stap 所支持的 event。 同时在 SEE ALSO 中可以看到其他相关的 man page。

### 内置 function

SystemTap 内置了一些可以直接使用的函数，比如 printf() 使用方式和 C 语言中的 printf 一样。除此之外还有如下几个：

+ tid()
+ uid()
+ cpu()  当前 cpu 个数
+ gettimeofday_s()  从 1970/1/1 起的秒
+ ctime() 转换上述时间
+ pp()  用来描述当前被处理的 probe point 的字符串
+ thread_indent() 更好的输出结果，可以接受一个参数，参数表示的是 indent 变化，可正可负。
+ name  标记当前系统调用的名字。
+ target()  在一开始我们提到这个，主要使用与绑定 pid 或者 command 名称


### 基本语法

SystemTap 和 awk 类似，可以自动识别变量的类型，不过需要区分局部变量和全局变量，全局变量需要在 probe 外通过 `global` 关键字定义。局部变量是被初始化过的。 支持 `++` 操作符。

+ 条件语句

条件语句和 C 类似，如下：

```shell
if (condition)
    statement1
else
    statement2
```

如果有多条语句，需要使用 `{}` 包裹。

+ 循环语句

```shell
while(condition)
    statement

# for 
for(initial; condition; increment)
    statement
```

一般我们写脚本需要传入一下命令行的参数， SystemTap 通过 `@1， @2` 等表示命令行参数。 使用时直接传入指定的函数中。

```shell
probe kernel.function(@1) { }
probe kernel.function(@1).return { }
```

#### 关联数组

关联数组和 awk 中的关联数组很像，使用上也没多少区别。 需要注意的一点是，我们需要在使用前通过 `global` 定义下数组。

需要着重说明的是， SystemTap 中支持多个 index。一般情况下，我们的索引是单个的字符串。比如:

```shell
arr["stephen"] = 30
```

但是在 SystemTap 中我们最多可以对同一个 value 设置 9 个 index， 比如：

```shell
device[pid(),execname(),uid(),ppid(),"W"] = devname
```

这么做的好处是什么呢？ 显而易见的是提供了更多的信息，当我们在遍历关联数组时，这些 index 都可以作为我们的信息输出。

比如：

```shell
foreach([pid, execname, uid, ppid, status] in device)
    printf("%4s %5s %d %d %s %s\n", pid, execname, uid, ppid, status, device[pid,execname,uid,ppid,status])
```

数组的读写和我们平常的使用一致，没有太多的区别，同样是通过取下标进行的。如果想要遍历关联数组，可以通过上面的 `foreach` 进行。 使用也比较简单。单个 index 的情况如下使用：

```shell
foreach(index in arrays)
    statements
```

这里面还需要着重说的是，`foreach` 有一个特别的语法，支持排序和个数限制。这样可以让代码更加的简洁。具体的语法如下：

```
foreach(index in array+ limit 10)
#or
foreach(index in array- limit 10)
```

在数组名后面添加 `+` 表示增序或者 `-` 表示逆序。 后面的 limit 表示限制 10 个元素输出。此外如果想要删除数组的元素或者清空整个组可以使用 `delete` 关键字。

我们还可以判断一个元素是否存在于关联数组中，语法如下：

```shell
if(["key"] in array) {
    statement
}
```

#### 总量统计

statistical aggregate 是用来统计一组数据，使用 `<<< value` 将 value 加到集合中。操作的对象可以是一个全局变量也可以是数组的一个元素。 这个统计是独立于操作对象本身的，怎么理解呢？举个例子。

比如我们有一个数组 `readbytes[execname()] <<< 2`。如果我们一直操作这样的语句，最后 `readbytes[execname]` 的值都是初始值，而不是我们想象中的 `2`。`<<<` 不是赋值的意思，他是存在另一个结构中，不会修改操作对象本生。

那如何获取到这个统计的具体内容呢？ SystemTap 提供了几个运算符，可以方便的来获取统计内容，在说到运算符之前，我们简单的看个例子： 假定有一个空集合，我们往里面添加一个 `<<< 1`, 此时集合中就有一个元素了， 我们继续添加一个 `<<< 2`。那此时集合中就有了 1 和 2 两个元素。 有了这样的印象后，下面看看几个获取统计的函数, 语法是：`@extractor(variable/array_index expression)`

extractor 可以是如下几个：

+ count  返回统计集合中元素的个数
+ sum    返回统计集合中元素值的总和
+ min    返回统计集合中元素的最小值
+ max    返回统计集合中元素的最大值
+ avg    返回统计集合中元素的平均值

对于 `count` 和 `sum` 而言，按照上面的例子而言，执行 count 后， 返回的是 2， 而执行 sum 之后，返回的是 3。


### tapset

SystemTap 提供了 tapset 库，可以类比下 C 的 libc。 tapset 中提供了相应的全局变量和函数可以直接在 SystemTap 脚本中使用。 具体的手册可以参考 [tapset](https://sourceware.org/systemtap/tapsets/index.html)



### 其他内容

probe 后一般可以跟上 `?` 或者 `!`。 这两个符号的意思是： `?` 表示 probe 是可选的，如果不存在对应的 probe ，程序也不会出错。而 `!` 表明 probe 一旦解析成功，则不会继续解析后面的 probe。


### 用户态的探针

说了这么多，大多都是涉及到的是内核相关的监测。这些监测会让我们更好的了解系统，但是如何将 SystemTap 应用到 user 模式中呢？ SystemTap 使用 `uprobes` 模块来支持用户态的探测。

一般内核版本在 3.5 以上的都默认包含了 `uprobes`。 可以通过 `grep CONFIG_UPROBES /boot/config-\`uname -r\`` 验证一下是否包含。输出结果为 `CONFIG_UPROBES=y`。

所有的用户态的事件监测都是用过 `process` 打头的。一般有如下的几个调用方式：

```shell
process.begin
process("PATH").begin
process(PID).begin

process.thread.begin
process("PATH").thread.begin
process(PID).thread.begin

process.end
process("PATH").end
process(PID).end

process.thread.end
process("PATH").thread.end
process(PID).thread.end
```

相关的解释如下：

> The .begin variant is called when a new process described by PID or PATH is created. If no PID or PATH argument is specified (for example process.begin), the probe flags any new process being spawned.
> The .thread.begin variant is called when a new thread described by PID or PATH is created.
> The .end variant is called when a process described by PID or PATH dies.
> The .thread.end variant is called when a thread described by PID or PATH dies.

除此之外还有一个 `syscall` 是比较好用的。

```
process.syscall
process("PATH").syscall
process(PID).syscall

process.syscall.return
process("PATH").syscall.return
process(PID).syscall.return
```

在处理 syscall 的 probe 时，我们可以使用 `$syscall` 获取系统调用数。 前六个该系统调用的参数可以通过 `$arg1, $arg2... $arg6` 等等获取到。