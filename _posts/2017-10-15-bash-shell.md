---
layout: post
title: bash shell detail
date: 2017-10-15 18:00:00
description: some details for bash using
---



shell 作为日常编程中比较方便的工具，通过一些 shell 脚本可以完成很多的事情，但是 shell 本身的一些语法比较古怪，且在不同的系统上所体现的结果有时候也不尽相同。日常使用中我们会遇到一些问题，同时也会积累一些比较好的惯用法，这篇文章主要的目的就是对某些细节进行分析，同时也会对一些习惯用法进行总结。这一篇会不断的更新和增加。

<hr>

###  positional parameter 简述

在 bash 手册中，位置参数是这样定义的：

> A positional parameter is a parameter denoted by one or more digits, other than the single  digit  0.
> Positional  parameters  are  assigned from the shell's arguments when it is invoked, and may be reassigned using the set builtin command.

通常我们最熟悉的就是使用 `$1` 这样的数字代表的变量。如果超过了10个后，我们可以这样使用 `${11}` 。此外还有一些特殊的位置变量。`$*` ,`$@`, `$#` ,`$FUNCNAME`。 最后一个变量其实在一定情况下和 $0 是相同的。

操作位置参数的方式可以通过内置命令 `shift` 。顾名思义就是切换位置参数，如果 shift 不带参数的话，则按位一个一个的切换，**同时也可以指定参数，一次性切换多个**，下面通过一个例子说明一下：

```shell
while [ "${1+defined}" ]; do
	# do something 
	shift
done
```

遍历整个位置参数，同时还防止当 `$1` 为空的时候会提前停止的情况，defined 是我们根据实际情况预先定义的。

我们还可以使用 for 循环来遍历位置参数，比如：

```shell
for arg; do
	 echo "$arg"
done
```

这个是使用的位置参数本身作为遍历对象，只会读取位置参数而不会修改，这个算是一个比较安全的操作。

提到位置参数，有两个经常容易混淆的参数必须要对比到，根据不同的情况的需要，我们可以灵活的使用相应的参数及其形式。 下面通过一个表来展示下这两者的差别。

​    

| 变量   | 输出结果                    |
| :----- | :-------------------------- |
| `$*`   | `$1 $2 $3 ... ${N}`         |
| `$@`   | `$1 $2 $3 ... ${N}`         |
| `"$*"` | `"$1c$2c$3c...c${N}"`       |
| `"$@"` | `"$1" "$2" "$3" ... "${N}"` |

   

这边的 `c` 是表示的 IFS 的第一个参数。一般情况下还是建议使用 `"$@"` 。 也许你会从中发现一个小的陷阱，即当我们在使用遍历位置参数的时候，如果使用 `"$*"` 将位置参数传到函数中，循环只会执行一次。并不是我们所期望的多次执行。 此外位置参数不添加引号时会自动扩展引号，举个例子就明白了：

```shell
#!/bin/bash
# script.sh
showcount() {
    echo "we get" $# 'parameter(s)'
}

showcount $@
showcount $*
showcount "$@"
showcount "$*"

$ ./script.sh 'a b' c d
we get 4 parameter(s)
we get 4 parameter(s)
we get 3 parameter(s)
we get 1 parameter(s)
```

 可以看出前两者将引号中的内容扩展了。同时这个也展示了使用 `"$*"` 作为函数参数传入是，变量个数为 `1`。

此外在数组那块还会提到和位置参数相关的内容，现在暂时不过多涉及。



### 使用 set 设置 positional parameter

set 的一个作用就是用来设置位置参数，现在不讨论 set 在脚本调试和其他功效上的内容，只讨论在设置位置参数时的作用。首先先看一个简单的例子：

```shell
$ set a b c
$ echo "$1" "$2" "$3"
a b c
```

这边的 `$1,$2,$3` 就是位置参数。 我们可以通过 `set` 来修改位置参数，比如：

```shell
$ set b c
$ echo "$1" "$2"
b c
$ set a "$@"
$ echo "$1" "$2" "$3"
a b c
```

我们后面修改了该位置参数，在原有的基础上，增加了一个新的参数 `a` ， 通过这个属性，我们可以做更多的事情，比如 `double` 一下所有的位置参数之类的。 我们再来看下一个例子：

```shell
$ set -- " hello,   world  "
$ echo $*
hello, world
$ echo $1
hello, world
```

这个例子从侧面看可以去掉多个空格，实际上的原理很简单，利用了 `$*` 的默认输出的是 `$1 $2 $3 ...`，  注意这里没有对变量添加引号，且我们可以看到位置参数并没有被分隔开。这一定程度上可以用做 `trim` 函数。

### Here documents 和 Here strings

这两者都是重定向的一种形式。**Here documents** 的语法很简单，不过由于也有两种形式，所以在使用的过程中也有一定的差别。语法如下：

```shell
command <<[-]word
...
...
word
```

一般用 here documents 的比较多的是在 usage 函数中。在这中间的所有的内容都是直接输出的, 不会做任何的修改，不过注意一点的是，如果使用的是 `<<-` 那么会将输出的文本中的前导tab去除掉。下面的几个例子展示了 here documents 的实际使用：

+ 抑制变量的扩展

```shell
$ cat << 'EOF'
> This is my name $name
> EOF
This is my name $name       # 这个变量没有被转换成相应的内容
```

+ 在管道中使用

```shell
$ cat << 'EOF' | sed 's/a/b/g'
> abc
> nab
> EOF
bbc
nbb
```

<hr>

**Here strings** 的基本语法是 `<<<`。它的主要作用是可以替代管道进行输入，我们知道使用管道的时候实际是操作是在 subshell 中进行的，所以有些变量在退出了 subshell 的环境后就不存在了。比如：

```shell
$ echo "Hello world" | read first second
$ echo $first $second
# nothing will output
```

遇到这种情况，一种折中的办法就是在 subshell 中进行输出，使用 group command 进行。但是如果我们想在当前的环境中使用变量，这种方式就不适合了。因此现在使用 Here strings 就比较合适了。

```shell
$ read first second <<< "Hello World"
$ echo $first $second
Hello World
```

looks good！很完美的解决了这个问题。

### 多使用 printf 而不是 echo

shell 中的 printf 在一定程度上和 C 语言中的 printf 类似，语法上基本上差别不大。熟悉 C 的话，使用这些还是很简单的。echo 默认输出换行符，且在不同的系统上的表现也不同，同时 `-n` 是没办法输出来的。

```
$ echo -n
$ echo '-n'  # cannot output
$ echo -e '\055n'
-n
```

通过 printf 可以做一些比较 cool 的事儿。

```shell
# 格式化输出
printf '%d | %0o | 0x%x' "126" "126" "126"
# 绘制水平线
printf '%.0s-' {1..20}; echo
```



