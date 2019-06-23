---
layout: post
title: "Bash 重定向"
subtitle: "重定向的基本原理和使用"
author: "L.K."
header-img: "img/inpost/post-bash-redirect-bg.jpg"
tags:
    - bash
    - 基础
---

在日常的 bash 脚本或者命令行的使用中，我们经常会使用到重定向，我们会使用`>`,`>>` 这样简单的基于标准文件描述符的重定向。如果使用到一些比较复杂的重定向时，重定向也会因为顺序的不同而产生不同的结果。所以有必要来理一理重定向的相关问题。

### The basics

在 linux 上，当我们打开一个 terminal 的时候，, 默认打开了三个文件描述符：

+ standard input:  **值为 0**  可用`stdin`表示
+ standard output: **值为 1**  可用`stdout`表示
+ standard error: **值为 2**  可用`stderr`表示

除了这三个标准的文件描述符，我们还可以通过 `exec` 来打开更多的文件描述符。


### 1.Output Redirection

输出重定向，基本语法 `n> file` (n 表示描述符的值）。一般使用它是 `command > file`（同样可以使用`command 1>file`）, 它主要是将原本输出到`stdout`上的内容重定向到了`file`，改变的是标准输出。

如果想将标准错误的结果重定向，可以使用`command 2>file`。这就将原来输出到标准错误的内容，重定向到`file`中。同样`command 3>file`这个命令是将`3`这个文件描述符重定向到`file`.

### 2.Input Redirection

输入重定向，基本语法`n< file`(n 表示描述符的值）。一般用它来改变`stdin`的值。可以使用`command <file`将 file 重定向到标准输入。同样`command 3<file` 将 file 重定向到`3`上。

### 3.Pipe

管道 `|` 主要是将连接了标准输出和标准输入， `|` 左边是标准输出， 右边是标准输入。

### Duplicating

上面介绍的是我们比较常用的三个方式，接下来是**复制文件描述符**。我们经常会看到这样的用法`2>&1`。这个重定向的意思是：**写到文件描述符 2 的内容将写到文件描述符 1 指向的地方**（不说 stdout/stderr 是因为这两个可以在此之前已经被重定向了）。

这边所谓的复制可以这样理解。在`2>&1`后，这两个描述符都指向了同一个地方。复制文件描述符可以抽象成`m>&n`, 其中`m`和`n`是两个描述符。

通过一个例子理解下复制：

```bash
$ cat file
line 1
line 2
$ exec 3<file
$ read -u 3 line
$ echo "$line"
line 1
$ exec 4>&3
$ read -u 4 line
$ echo "$line"
line 2
```
可以看到在执行了`4>&3`后，使用描述符`4`同样能读到`3`指向的那个文件的下一行。这就表明将`3`复制给了`4`。

### Order Of Redirection

使用重定向的顺序也是需要注意的，比如一个很经典的问题：

> Q:what's the difference between 2>&1 >foo and >foo 2>&1, and when do I use which?

利用这个问题来说明一下不同的顺序使用重定向会带来什么影响。首先从：

+ `2>&1 >foo`

首先`2>&1`将描述符`1`复制给描述符`2`, 这样`1`和`2`都指向了同一个地方。 接着将`1` 重定向到`file`。 但是此时`2`还是指向是**之前`1`指向的地方**。所以此时`2`和`1`不是指向的同一个地方，这相当是将`1`备份了一下。

+ `>foo 2>&1`

首先`>foo`将`1`重定向到`foo`, 接着`2>&1`, 将`1`复制给`2`。此时`1`和`2`都被重定向到了`foo`上。

下面的一个例子便能很好的说明这两个的区别：

```bash
f() {
	echo "This is stdout"
	echo "This is stderr" 1>&2
}

f >foo 2>&1   # nothing printed out
f 2>&1 >foo   # print "This is stderr" only
```

这两种方式没有对错，在不同的需求下使用对应的顺序。

### Some Pratical Usages

在一些情况下，使用重定向会引发一些错误，我们可以使用一些方式去规避它。

+ sed 命令

`sed 's/a/A/g' file > file`。这个用法估计很多人在不经意间都会用到，实际上这个用法是不会有作用的。我们之所以这么做是想将 sed 做出的修改写入到文件中。所以将标准输出重定向到该文件上。问题就在这儿！ 在 sed 命令执行之前，file 先被重定向，这时 file 就被截断，内容已被清空了，所以 sed 在读文件的时候，什么都读不到。 而正确的做法是使用`-i` 选项。

> 注： 千万不要在 sed 中将重定向的文件指定成即将用于输入的那个文件。

+ read 命令

在 bash 中，我们习惯性使用如下方式来读取文件的内容：

```bash
while read -r line; do
  echo "$line"
done < file

```

这个用法是没有问题的，也可以很好的执行。不过如果我们在 while 循环体内使用再次使用 read 呢？比如：

```bash
while read -r line; do
   echo "$line"
   read -p "Continue to read?" -n 1
done < file
```

这个情况下就会出错了。出错在于，此时循环体内的标准输入已经被重定向到了`file`, 而`read`是要从标准输入中读取，而这时只能从 file 中读取了。这和我们所期望的就有所不同了，
此时可以使用如下方式：

```bash
exec 3<file
while read -u 3 line;do
	echo "$line"
	read -p "Continue to read?" -n 1
done
```
将文件在描述符`3`上打开，通过 read 指定读取时的描述符，这样就避免了标准输入被重定向的问题。


### Create More File Descriptor

上面简单的提了一下使用`exec`创建描述符，现在介绍一下如何创建合适类型的文件描述符。我们直接从例子来说明创建的过程：

```bash
# we have a file named foo
$ cat foo
hello world
$ exec 3<foo    # create a fd only can used to read.
$ read -n 5  word <&3 # read 5 character from foo
$ echo "$word"
hello
$ echo "good" >&3
-bash: echo: write error: Bad file descriptor
```

我们可以看到，再往`3`里面写入的时候报错了，说明此时`3`只是一个只读的文件描述符。

> 注意在使用`3`这个描述符的时候，我们需要利用复制的方式，不然 bash 会将`3`理解成普通的文件。如果当前目录下没有`3`这个文件，
> bash 便会报错：No such file or directory

因此我们需要用到如下的方式：

```bash
$ exec 3<>foo  # create a read and write fd
$ read -n 5 word <&3 #read seek the file to position 5
$ echo -n / >&3
$ cat foo
hello/world
```

这会儿的`3`便可读可写。同样的，可以使用`exec 3>foo` 创建一个可写的文件描述符。（这个可以结合 Linux 系统编程上的 open 函数理解）


### Close File Descriptor

如果这个描述符不需要再使用了，可以关闭这个描述符。bash 提供了如下的关闭方式：

+ `n<&-`  关闭一个用于输入的描述符
+ `0<&-, <&-`  关闭标准输入
+ `n>&-`  关闭一个用于输出的描述符
+ `1>&-, >&-`  关闭标准输出


### Some Abbreviations

+ `|&`  abbr  `2>&1 |`  added in bash4
+ `&>/dev/null` abbr `>/dev/null 2>&1`

这些缩写可以了解一下，目的是为了能够在别人有使用的时候可以能读懂。

### Small Example

当我们想在脚本中，希望 log 能够一边输出到终端上，一边又能写入文件中。这时候也可以使用到重定向，在 linux 中有一个命令`tee`是可以将内容输出到标准输出和文件的。我们可以用`|`来实现，比如：`echo "pass" | tee log`。不过如果有很多的 log，每条都用`|tee log` 会比较繁琐。所以可以结合`process substitute` 和重定向来简化这个过程。

```bash
exec > >(tee log)
echo "pass"
```

这样只要往标准输出的内容，都会被丢给`tee`. 不过这还不能很完美的工作。原因在于`echo`是带有缓冲的，所以如果 log 只是有关标准输出的，那么这么使用是没有问题的。不过我们一般也会将标准错误的内容也保存到 log 中的话，可能会出现打印出来的顺序和实际代码中 log 输出的顺序不一致。

```bash
exec 1> >(tee log)
exec 2> >(tee log)

echo "case 1 pass"
echo "case 2 error" >&2
echo "case 3 pass"
echo "case 4 error" >&2

#will output
case 1 pass
case 3 pass
case 2 error
case 4 error
```

如果是对 log 顺序有要求的话，这样的输出明显是不符合条件的。 好在 linux 中提供了两个命令`stdbuf`和`unbuffer`。这两个命令的原理是不同的，具体的可自行 goole。

```bash
echo_unbuf{
	stdbuf -O0 echo "$@"
	#unbuffer echo "$@"
}

echo_unbuf "case 1 pass"
echo_unbuf "case 2 error" >&2
echo_unbuf "case 3 pass"
echo_unbuf "case 4 error" >&2

#will output like output
```

通过封装一个`echo_unbuf`，这样便能保证了 log 输出的顺序是正确的。最后如果出现了后台命令在程序结束后打印，则可以使用`sync`来同步一下。

## Reference

1. [Difference between some redirections](http://unix.stackexchange.com/questions/70963/difference-between-2-2-dev-null-dev-null-and-dev-null-21/70971#70971)
2. [IO Redirection](http://www.tldp.org/LDP/abs/html/io-redirection.html)

