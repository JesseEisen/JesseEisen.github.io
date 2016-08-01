---
title: Understanding Redirection In Bash
updated: 2016-08-01 19:00
---

## Reason

在日常的bash脚本或者命令行的使用中，我们经常会使用到重定向,我们会使用`>`,`>>` 这样简单的基于标准文件描述符的重定向。如果使用到一些比较复杂的重定向时，重定向也会因为顺序的不同而产生不同的结果。希望通过这篇blog，能够理清一下重定向的相关问题。

## The basics

在linux上,当我们打开一个terminal的时候,,默认打开了三个文件描述符:

+ standard input:  **值为0**  可用`stdin`表示
+ standard output: **值为1**  可用`stdout`表示
+ standard error: **值为2**  可用`stderr`表示

除了这三个标准的文件描述符，我们还可以打开更多的文件描述符。通过`exec` 来打开。·


### 1.Output Redirection

输出重定向,基本语法`n> file`(n表示描述符的值)。一般使用它是`command > file`（同样可以使用`command 1>file`）,它主要是将原本输出到`stdout`上的内容重定向到了`file`。改变的是标准输出。 

如果想将标准错误的结果重定向,可以使用`command 2>file`。这就将原来输出到标准错误的内容，重定向到`file`中。同样`command 3>file`这个命令是将`3`这个文件描述符重定向到`file`.

### 2.Input Redirection

输入重定向，基本语法`n<file`(n表示描述符的值)。一般用它来改变`stdin`的值。可以使用`command <file`将file重定向到标准输入。同样`command 3<file` 将file重定向到`3`上。

### 3.Pipe

管道`|` 主要是将连接了标准输出和标准输入， `|`左边是标准输出， 右边是标准输入。

## Duplicating

上面介绍的是我们比较常用的三个方式，接下来是**复制文件描述符**。我们经常会看到这样的用法`2>&1`。这个重定向的意思是:**写到文件描述符2的内容将写到文件描述符1指向的地方**(不说是stdout/stderr是因为这两个可以在此之前已经被重定向了)。 

这边所谓的复制可以这样理解。在`2>&1`后，这两个描述符都指向了同一个地方。所以复制文件描述符可以抽象成`m>&n`,其中`m`和`n`是两个描述符。

通过一个例子更见明确一下复制：

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
可以看到在执行了`4>&3`后,使用描述符`4`同样能读到`3`指向的那个文件的下一行。这就表明将`3`复制给了`4`。

## Order Of Redirection

使用重定向的顺序也是需要注意的，比如一个很经典的问题:

> Q:what's the difference between 2>&1 >foo and >foo 2>&1, and when do I use which?

利用这个问题来说明一下不同的顺序使用重定向会带来什么影响。首先从：

+ 2>&1 >foo

首先`2>&1`将描述符`1`复制给描述符`2`,这样`1`和`2`都指向了同一个地方。 接着将`1` 重定向到`file`。 但是此时`2`还是指向是**之前`1`指向的地方**。这相当是将`1`备份了一下。

+ >foo 2>&1

首先`>foo`将`1`重定向到`foo`, 接着`2>&1`, 将`1`复制给`2`。此时`1`和`2`都被重定向到了`foo`上。

下面的一个例子便能很好的说明这两个的区别：

```bash
f() {
	echo "This is stdout"
	echo "This is stderr" 1>&2
}

f >file 2>&1   # nothing printed out
f 2>&1 >file   # print "This is stderr" only
```
·
这两种方式没有对错，可以在有需求的时候使用对应的顺序。

## Some Pratical Usages

在一些情况下，使用重定向会引发一些错误，我们可以使用一些方式去规避它。

+ sed命令

`sed 's/a/A/g' file > file`。这个用法估计很多人在不经意间都会用到，实际上这个用法是不会有作用的。我们之所以这么做是想将sed做出的修改写入到文件中。所以将标准输出重定向到该文件上。问题就在这儿！ 在sed命令执行之前，file先被重定向，这时file就被截断，内容已被清空了，所以sed在读文件的时候，什么都读不到。 而正确的做法是使用`-i` 选项。

> 注： 千万不要将重定向的文件指定成即将用于输入的那个文件。

+ read命令

在bash中，我们习惯性使用如下方式来读取文件的内容：

```bash
while read -r line; do
  echo "$line" 
done < file

```

这个用法是没有问题的，也可以很好的执行。不过如果我们在while循环体内使用再次使用read呢？比如：

```bash
while read -r line; do
   echo "$line"
   read -p "Continue to read?" -n 1
done < file
```

这个情况下就会出错了。出错在于，此时循环体内的标准输入已经被重定向到了`file`, 而`read`有要从标准输入中读取，这时只能从file中读取了。这和我们所期望的就有所差别了。
所以此时我们可以使用如下方式：

```bash
exec 3<file
while read -u 3 line;do
	echo "$line"
	read -p "Continue to read?" -n 1
done 
```

## Create More File Descriptor

上面简单的提了一下使用`exec`创建描述符，现在稍微介绍一下如何创建合适类型的文件描述符。我们直接从例子来说明创建的过程：

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

> 注意在使用`3`这个描述符的时候，我们需要利用复制的方式，不然bash会将`3`理解成普通的文件。如果当前目录下没有`3`这个文件,
> bash便会报错:No such file or directory

因此我们需要用到如下的方式：

```bash
$ exec 3<>foo  # create a read and write fd
$ read -n 5 word <&3 #read seek the file to position 5
$ echo -n / >&3
$ cat foo
hello/world
```

这会儿的`3`便可读可写。

## Close File Descriptor

如果这个描述符不需要再使用了，可以关闭这个描述符。bash 提供了如下的关闭方式：

+ n<&-  关闭一个输入的描述符
+ 0<&-, <&-  关闭标准输入
+ n>&-  关闭一个输出的描述符
+ 1>&-, >&-  关闭标准输出


## Some Abbreviations

+ |&    ==>  `2>&1 |`  added in bash4
+ &>/dev/null  ==> `>/dev/null 2>&1`

这些缩写可以了解一下，目的是为了能够在别人有使用的时候可以能读懂。

## Reference

1. [Difference between some redirections](http://unix.stackexchange.com/questions/70963/difference-between-2-2-dev-null-dev-null-and-dev-null-21/70971#70971)   
2. [IO Redirection](http://www.tldp.org/LDP/abs/html/io-redirection.html)

