---
layout: post
title: "Bash Shell Script 杂记"
subtitle: "记录和解释难以顺畅使用的Shell"
author: "L.K."
header-img: "img/inpost/post-bash-script.jpg"
tags:
    - bash
    - 笔记
---



shell 作为日常编程中比较方便的工具，通过一些 shell 脚本可以完成很多的事情，但是 shell 本身的一些语法比较古怪，且在不同的系统上所体现的结果有时候也不尽相同。日常使用中我们会遇到一些问题，同时也会积累一些比较好的惯用法，这篇文章主要的目的就是对某些细节进行分析，同时也会对一些习惯用法进行总结。注意本文所提及的语法在 bash 中验证通过，所以如果你使用的是 zsh 或者其他 shell， 在使用脚本的时候建议加上: `#!/usr/bin/env bash` 。 

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

​           

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



###  数学计算

目前推荐使用 `$(( ))` 进行数学计算，虽然使用 `let`  也可以完成相应的工作但是使用前者是比较稳妥的，且不容易出错。

首先在 bash 中，数字的表示形式有如下几种：

+ `0...`  表示为八进制数
+ `0x...` 或者 `0X...`  表示为十六进制数
+ `<BASE>#...`  根据 base 来解析后面的数字。

 其余的则会被当做十进制的数来使用。 对于第三种情形，需要着重说一下，这个语法的目的是为了将其他不同 base 的值转换成十进制的值。 比如：

```shell
$ printf '%s\n' $((16#abc))
2748
$ printf '%s\n' $((2#010101))
21
$ printf '%s\n' $((10#abc))
bash: 10#abc: value too great for base (error token is "10#abc")
```

所以我们可以看到，即使数字没有使用相应的机制标识开头也同样可以转换成十进制。同样的如果想将十进制转换成其他的进制的，可以使用 `printf` 格式化输出。

下面来说一下计算部分，在 `$(())` 中支持大部分的数学计算语法。逻辑运算，位操作，以及常规的数据运算都是支持的。下面通过一些基本的例子展示下数学计算：

```shell
$ printf '%d\n' $((a=2, a+=3))
5
$ printf '%d\n' $((a=1, a++))
1
$ printf '%d\n' $((1+3))
4
```

我们需要注意的是，返回值是最后一个表达式的结果。 除此之外，在 `$(())` 中也是支持变量操作的。比如：

```shell
$ ((a=16#abc, b=16#${a:0:2})); printf '%s, %s\n' $a $b
2748, 39
```

这边涉及到了子串扩展，这个在后面详细说明。 不过并不是所有的操作都是合法的，也有一些情况需要我们注意到。比如下面的例子中：

```shell
$ x=1
$ echo $(($x[0])) # 将会被扩展为 $((1[0]))
bash: 1[0]: syntax error: invalid arithmetic operator (error token is "[0]")
$ printf '%d\n' $((${x[0]})) 
1
$ printf '%d\n' $(("$x" == 1))  # 解析为 $(("1")) 
1
```

上面的这一点我们就得注意下，有些变量并不一一定能在数学扩展中解析出来。此外，我们也可以用变量扩展作为布尔值的判断。比如：

```shell
if ((1 == 2)); then
	echo "true"
else
	echo "false"
fi
# false
```

shell 中的数学计算基本上就是这些，还有其他的内容后续继续增加。



###  正则表达式

我们在写脚本时，可以使用 `sed`, `awk`, `perl` 等工具自带的正则。但是如果不是很复杂的情况， 单独使用 bash 自带的正则就可以了。bash 本身支持的是 ERE 语法，同时也支持 group 匹配。RE 的语法在这里不多说了，正则表达式在有时候会比较清晰一些。比如下面的这个例子：

```shell
$ ls 
1_abc.txt  245_def.txt 
```

将上述的文件名中的数字转换成16进制的。 这个问题解决的办法很多，我们尝试使用 RE 来解决这个问题。

```shell
convert() {
    filename="$1"
    rx='^([0-9]+)_(.*)$'
    if [[ "$filename" =~ $rx ]]; then
    	mv "$filename" "$(printf '%04x%s' "${BASH_REMATCH[1]}" "${BASH_REMATCH[2]}")"
    fi
}
```

匹配到数字部分和字符串部分然后将其通过 `printf` 输出。匹配到的分组部分保存在 `BASH_REMATCH `中， 不过通过 bash 的字符串子串操作也可以轻松完成。不过在获取子串这块语法会存在一些混淆，这个后面再说，现在看看使用子串如何解决这个问题。

```shell
# bad， not safe
convert() {
    number=${1%%_*}
    other=${1##*_}
    mv "$1" "$(printf '%04x%s' "$number" "$other")"
}
```

实际上通过子串分割更加的快捷，不过需要保证每个文件都是统一的格式，不存在其他的格式。所以使用正则的好处是对于不符合条件的文件名不进行分割，在一定程度上更加的安全一些。

###  字符串子串

bash 中有不少的扩展，其中包括 `paramater expand` 。 先不讨论有关变量赋值的扩展，只讨论和子串有关的部分。首先和其他脚本语言中比较类似的切片语法。

+ ${parameter:offset[:length]}

这个扩展很好理解，返回值为 parameter 从 offset 处开始的 length 个字符。 这个 offset 可以为负数，其中 length 是可选。 parameter 可以为位置参数，在这种情况下索引是默认从 1 开始的。其他的情况下索引是从 0 开始的。一个简单的例子展示一下这个是如何扩展的。

```shell
$ cat substring.sh
#!/usr/bin/env bash
str="This is a string"
printf "%s\n" "${@:2:2}"
printf "%s\n" "${str:5}"
$ ./substring.sh 1 2 3 4 5
2 
3
is a string
```

这个扩展对数组也是可以使用的，不过如果使用在关系型数组上，其结果是未定义的。

+ ${parameter#word}
+ ${parameter##word}

移除掉匹配的前缀，所谓的前缀就是 word 所表示的。word 部分可以使用通配符来表示，有关通配符的内容见后面。 这个匹配一般在获取后缀名或者是目录名比较有用。 比如下面的例子：

```shell
$ mypath=$(pwd)
$ echo "$mypath"
/home/user/work/shell
$ echo ${mypath#*/}
home/user/work/shell
$ echo ${mypath##*/}
shell
```

一个 `#` 表明匹配最短的前缀； 两个 `##` 表明匹配最长的前缀。 前缀部分的内容需根据实际情况来决定。

+ ${parameter%word}
+ ${parameter%%word}

移除掉后缀，原理同上面的。word 部分支持通配符的表示。这个一般在获取文件名或者目录的 path 比较有用。所以我们可以这样使用：

```shell
$ myfile=base.name.txt
$ echo "${myfile%.*}"
base.name
$ echo "${myfile%%.*}"
base
```

但是也能看到这个扩展的局限性，就是匹配的结果要么最少，要么最多，没有达到一个平衡，不过这些都需要我们自己去衡量与选择。

+ ${parameter/pattern/string}

模式替换，实际上就是将匹配到的内容用 string 替换掉。这个功能有一些使用的技巧，整理如下：

| 场景               | 结果                         |
| :----------------- | ---------------------------- |
| pattern 以`/ `开头 | 替换掉所有匹配的 pattern     |
| pattern 以`#` 开头 | 必须是头部就匹配到 pattern   |
| pattern 以`%` 开头 | 必须是尾部开始匹配到 pattern |
| string 为空        | 删除掉匹配到的 pattern       |

除此之外，如果参数是 `@` 或者  `*` 的话，则对每一个参数都使用这样的匹配。下面通过几个例子来简单说明下:

```shell
$ target="This is a string"
$ echo "${target/is/IS}"  # only first matched will be replaced
ThIS is a string
$ echo "${target//is/IS}" # each matched will be replaced
ThIS IS a string
$ echo "${target/#T/t}" # replace first character
this is a string
$ echo "${target/%sting/STRING}" # repalce end if the string
This is a STRING
$ echo "${target/is}" # delete first matched
Th is a string
$ echo "${target//is}" # delete all matched
Th  a string
```

主要是需要记住一些符号的使用，其他的内容，现在不在这边过多的涉及，讨论过多后更加容易混淆，这边只介绍和子串相关的内容，至于大小写转换的功能，这些提及的比较少，所以也不整理了。

###  通配符

要完整的理清统配符的内容，得单独理出一篇。不过在这边主要是说明一些最基本的。至于通过 `shopt` 打开相关的扩展之类的，暂时不涉及。

实际上通配符只是一个低配版的 RE，语法标记上简化了很多。注意  `.` 在通配符中没有特殊含义，仍然代表的是原本的含义。 在 bash 中一般是如下的几个标记：

+ `*`  匹配一个或者多个字符
+ `?`  匹配一个字符
+ `[...]` 匹配这个集合中的任意一个字符

我们使用的最多的是 `*` ，这在遍历文件是最常用到：

```shell
for file in *; do
	......
done
```

除此之外，我们在 case 中经常会使用到这个，比如验证输入Yes/NO。 我们可以这样做：

```shell
case "$input" in
	[Yy]|'')  end=1;
	[Nn]*)    end=0;
	*)        echo "unknown character"
esac
```

实际上我们在使用  `[[`时也可以使用通配符，比如下面这样的方式：

```shell
if [[ $input = [Ee]rror ]]; then 
	... 
fi
```

此外还有一些范围相关的通配符，比如: `[a-h]`, `[[:alnum:]]`，`[[:digit:]_.]` 等，这些用法是合法的。如果你熟悉 RE，这些应该不会太陌生。 至于取反操作有的是 `!` , 表示非。 余下的扩展部分可以看之前整理的[shell-glob](http://jesseeisen.github.io/2016/08/16/shell-glob.html)， 这里面有对扩展部分的例子。
