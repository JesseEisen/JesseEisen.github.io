---
layout: post
title: "命令行解析"
subtitle: "分析市面上常用的几种命令行解析方式"
author: "L.K."
header-img: "img/inpost/post-commandline-bg.jpg"
tags:
    - Linux
    - Programming
---

我们在日常编程中，对命令行参数的解析是不可避免的。一般情况下，不同的编程语言都提供了相应的库函数来解析命令行参数。这些库函数有些是比较顺手的，有些则比较原始，通过了解一下不同的解析思路，利用不同语言自有的特性实现一些顺手的命令行解析工具。

---

###  命令行基本概念

首先我们得了解一下命令行的基本格式。比如 GNU 的命令行参数的语法惯例如下：

+ 命令行参数若是以 `-` 开头的则认为是选项（options）
+ 多个选项可以在单个的 `-`  后面（注意选项应为不带参数的形式）
+ 选项的名字是一个单个的字符数字的形式（alphanumeric）
+ 选项后面可以带参数
+ 选项和他的参数之间可以有或者不存在分隔符， 比如 `-o foo`  和 `-ofoo`  可以认为是相同的
+ 选项一般在其他非选项的参数前面出现
+ `--`  表示参数终止， 在这之后的参数都会被认为是非选项，即使是带了 `-`  的
+ 单独的一个 `-`  一般用来表示读取或者写向标准输入输出
+ 选项可以以任意顺序出现，同一个选项也可以出现多次

上述的语法，大体上说明了一个基本的命令行该有的形式，但是还有一些细节没有涉及到。 比如一个选项是否接受一个可选参数或者必须带有一个参数。 我们在看一些命令的手册时，往往会看到这样的几个符号：

+ `[]`
+ `<>`
+ `()`
+ `|`
+ `...`

这几个符号要细究下来，组合的形式比较多，所表示的含义也不尽相同。所以想要定义一个比较标准的命令行 Usage，对于这些参数的组合使用是必须了解的。

+ 参数

一般我们使用  `<argument>`  表示参数， 这个参数即为选项后面所需要带的参数，或者是正常命令行参数。 比如：

```shell
 program <arg1> <arg2>
 program -o <file>
 program --input=<source>
```

这些都是表示参数，一般参数提供出来，未加修饰符都默认是必须提供的参数。

+ 可选项

```shell
program [-w option] [-f value]
program [-f] [-o] [<argument>]
```

使用 `[]` 表明这个参数或者选项是可选的，可选的意思即为在运行命令时这些是可提供或者不提供的。

+ 互斥

```shell
program [-a | -b]
类似于如下写法：
program [-a]
program [-b]
```

互斥表明这两个参数或者多个参数只能选择其中之一，如果都出现了，则会报错, 上面的实例中表示可以都不出现。

+ 必选项

```shell
program: <argument>
program: (-a <good> | <bad>)
```

使用 `()` 是为了来表示必选，或者是作为一个组合的意思。 上面的例子指的是两个选项必须出现其中之一。

+ 重复

```shell
ls: [FILE]...
program: (<from> <to>)...
```

重复的意思即为这个参数或者这一组参数可以出现多次。最直观的的就是 `ls`  可以有参数, 可以同时有多个参数等。

当我们打开 man 手册的时候，往往会发现在  `SYNOPSIS`  这一小节中的内容其实很简单。比如:

```shell
ls [OPTION]... [FILE]...
```

这边使用 `OPTION`  做一个占位符，在后续将选项的具体形式极其使用说明进行详细描述。 这也为我们在编写 Usage 函数时提供了一个思路。

一个简单的 usage 示例：

```shell
Usage: ./test [options]
Options:
        -V, --version                 output program version
	-h, --help                    output help information
	-v, --verbose                 enable verbose stuff
	-r, --required <arg>          required arg
	-o, --optional [arg]          optional arg
```

当我们在设计一个命令行工具的时候，一些选项的命名实际上是可以遵循一些惯例的，比如 `--help` , `--version` 等输出基本程序信息。 GNU 列出了一个在 GNU 软件中所使用到的长选项的[说明](https://www.gnu.org/prep/standards/html_node/Option-Table.html#Option-Table)。 目的是为了后续的一个兼容，这对我们日常的编写程序也是有一定的指导意义的。

上面说的这些并不能完全概括所有的情形，只能说作为在 `*nix` 上工作的人所习惯的一个范式，不过有了理论基础，实际操作上，不同的语言或者不同的库所提供的解析方式有很大的不同。我们总想解析能做到简单易用，指哪打哪儿的效果，但是现实往往并不能如愿，下面就从我说熟悉几个语言上说一说几种不同的命令行解析思路。

<hr>

###  Awk 语言篇

之所以将 awk 放在最开始说，是因为 awk 本身并没有提供命令行解析的相关函数或者库， 即使是 gawk 也没有提供相应的库，所以需要我们自己去实现。正因为如此，将其放在最开始说是很有必要的，因为我们需要去从头到尾的实现下 `getopt` 函数, 通过这个过程来了解一下 getopt 的基本原理。

getopt 一般是在一个循环中使用。 基本的使用框架如下：

```c
getopt(argc, argv, options)
使用方式：
while((flag = getopt(argc, argv, "ab:cd")) != -1){
	switch(flag){
		...
	}
}
```

options 是预先定义的，argc 和 argv 表示的是命令行参数，getopt 最简单的思路如下：

+ 从命令行中取出合法的 option （短或者长的参数），解析出 flag
+ 在 options 中找到对应的 flag， 并检查是否带有参数
+ 若是需要带参数的，则将下一个命令行参数设置到 Optarg 中，否则该 flag 解析完成， 返回该 flag。

一般 getopt 会维护两个全局变量， Optind 和 Optarg， 前者表示解析到第几个命令行参数， 后者表示该 flag 带有的参数。 getopt 使用 `:`  表示一个 flag 是带有参数的。

下面是 GNU awk 对 getopt 的一个简单实现：

```shell
function getopt(argc, argv, options,   thisopt, i)
{
        if(length(options) == 0){
                return -1
        }
		# no support  --
        if(argv[Optind] == "--"){
                Optind++
                _opti = 0
                return -1
        }else if(argv[Optind] !~ /^-[^: ]/){
        # flag must start by - and not contain : and space
                _opti = 0
                return -1
        }

        if(_opti == 0)
                _opti = 2
        thisopt = substr(argv[Optind], _opti, 1)
        Optopt = thisopt
        i = index(options, thisopt)
        if(i == 0){
                if(Opterr)
                        printf("%c -- invalid option\n", thisopt) > "/dev/stderr"
                if(_opti >= length(argv[Optind])){
                        Optind++
                        _opti = 0
                }else
                        _opti++
                return "?"
        }

        if(substr(options, i+1, 1) == ":"){
                if(length(substr(argv[Optind], _opti + 1)) > 0)
                        Optarg = substr(argv[Optind], _opti + 1)  # -abcfoo
                else
                        Optarg = argv[++Optind]
                _opti = 0
        }else
                Optarg = ""

        if(_opti == 0 || _opti >= length(argv[Optind])){
                Optind++
                _opti = 0
        }else
                _opti++
        return thisopt
}
```

这个 getopt 的实现不支持长参数解析，同时使用上也很有局限性，但是很清楚的说明了 getopt 的基本原理。 我们可以这么使用：

```shell
BEGIN {
	Opterr = 1
    Optind = 1

    while((flag = getopt(ARGC, ARGV, "ab:cd")) != -1)
    	printf("flag=%c , optarg=%s\n", flag, Optarg)

    for(; Optind < ARGC; Optind++)
    	printf("Non-option argument:[%d] %s\n". Optind, ARGV[Optind])
}
```

使用 getopt 主体在结果处理上，每次解析出来的 flag 和 optval 可以按照我们想要的方式进行处理。由于 getopt 的使用方式是通用的，如果你熟悉 getopt ，只要支持 getopt 的情况下，一般都能做到上手就用，最多也就是阅读下 man 手册上的一些细节和 feature。

上面实现的 getopt 不支持长参数，若要加入对长参数的解析并不是很困难，只需要在执行解析前将长短 flag 做一个映射即可，这在 awk 中可以直接使用关系型数组即可，比如：

 ```shell
longopts["create"] = "c"
longopts["verbose"] = "V"
...
 ```

设置这个 map 后，当我们解析出是长参数时，判断一下当前定义的长参数是否在 longopts 中，可以这么做：

```shell
if(!(lopt in longopts)){
    # error print
    return "?"
}

opt = longopts[lopt]
# then we just return the opt
```

一般命令行操作都会提供长短参数配套使用机制的。但是也有需求是只提供长参数，并不与短参数绑定，所以只需要将这个 map 的值设置成我们期望返回的即可。

awk 的好处就在于能够快速搭建一个原型，这对我们后续对 C 的分析有很大的帮助。

<hr>

### C 语言篇

C 语言中我们大多使用的就是 `getopt` 和 `getopt_long`  这两个标准库中的函数，我们所以做的就是在定义好 flag 后需要自己去解析，一方面很自由，如何去处理 flag 所带的参数是由开发者自己去实现；另一方面又显得很繁琐， 因为有时候我只想解析后就能直接使用 。

C 中的 getopt 和上述的 awk 方式类似，所以这边不多加叙述。 按照一种更好的做法，在 C 中最好是使用 getopt_long，因为有对长参数的支持。下面就来说一说 getopt_long 的基本使用情况。

在 AWK 篇的最后我们简单的说了下实现对长参数的解析的方式，C 中没有关系型数组，但是 C 可以定义结构，因此，当我们使用 getopt_long 的时候，我们需要初始化一个结构体。

```c
struct option {
    const char *name;
    int 	has_arg;
    int        *flag;
    int 	val;
};
```

这四个参数中最后两个参数比较灵活一些，前面两个从字面意义上也能看：第一个是长标签的名字，第二个为是否有参数。

`flag` 如果设置为 NULL， 那么 getopt_long 就将返回 val， 如果 flag 不为空，而是指向一个变量，那么 getopt_long 返回 0， 并将 val 赋值给 flag 指向的变量。`val` 的作用就是作为 getopt_long 的返回值，或者是给 flag 指向的变量赋值。

而如果我们想将长短参数进行绑定的话，只需要将 flag 设置为 NULL， 并将 val 设置成短参数的值，这样 getopt_long 返回的值和是短参数时一致。至于实现原理，基本上和 getopt 类似，只是需要多加一个对长参数的解析以及寻找到与之绑定的短参数(或者直接返回指定的val)。

具体的使用例子就不在此描述了，可以看下 getopt_long 的 man 手册，里面提供了相关的示例。

<hr>

回过头来想一想， getopt 的操作并没有特别的繁琐，只是在 flag 很多的情况下， switch 的判断分支会很多。从上面我们会看到使用 getopt 我们要事无巨细的写所有的处理。实际上我们可以做的更加的集中一些， 比如 TJ 的 [commander](https://github.com/clibs/commander) 以及 GNU 的 [Argp](https://www.gnu.org/software/libc/manual/html_node/Argp.html#Argp) 。 后者的使用还是稍重一些。前者就显的比较 modern ， 提供的接口也比较简单，使用起来并不复杂。

```c
  command_t cmd;
  command_init(&cmd, argv[0], "0.0.1");
  command_option(&cmd, "-v", "--verbose", "enable verbose stuff", verbose);
  command_option(&cmd, "-r", "--required <arg>", "required arg", required);
  command_option(&cmd, "-o", "--optional [arg]", "optional arg", optional);
  command_parse(&cmd, argc, argv);
```

不过 commander 写的比较多的是最后的回调，如果参数很多，而参数处理很简单的话，整个处理的篇幅也比较大，不过这一套实现命令行的处理机制是值得我们去学习的，结构化处理易于维护。

现在我们来看一看 commander 的实现原理：

command_option 函数将所定义的长短参数放入对应的 struct 域中， 着重说一下在长参数处理上。需要将这个字符串拆分成长参数和 arguments。比如：`--required <arg>`  要解析成 `require`  和 `<arg>`  并将这两个分别存储在 `option->large`  和  `option->argname`  中。 并根据  `<`  和 `[`  判定是否是可选还是必须参数。

在 command_parse 中，首先将 argv 的参数标准化， 即将短参数组合的情况拆分开，比如 `-abc` 需要转成 `-a -b -c`  的形式。接着就是遍历 argv 和 我们之前解析保存的 option 结构，一个一个的比对，如果匹配成功，调用 cb 函数处理结果。

基本的解析过程就是这样，思路很简洁，没有特别绕的地方，如果有兴趣，建议走读一遍代码。

在实际使用中， commander 仍然有一些不能处理情况，比如：当使负数时，负数会被误认为是 flag， 同时不支持 `--` 终止操作, getopt 和 getopt_long 是支持 `--`  的， 所以这里需要 hack 一下 commander 的代码，由于代码的结构清晰，所以自己扩展一下使用很方便。这是我之前提交的一个 [PR](https://github.com/clibs/commander/pull/23) 用来支持 `--` 的， 但是未被合并 :(  。

还有一种情况——子命令， 最典型的例子就是 `Git`。Git 提供了很多的子命令，每个子命令又有自己的 flag，所以此时一种更加通用的解析方式很必要。接下来我们理解一下 Git 是如何处理命令行参数的。

首先 Git 是从 `common_cmd.c`  中的 main 启动，之后执行 `git.c`  中的 `cmd_main()` 函数。在 cmd_main 函数中包含对命令行的处理。首先将 Git 的子命令或者说是内置命令定义在一个结构中： ` struct cmd_struct commands[] ` 。这个表中定义的是子命令的名称以及该命令对应的处理函数，这些处理函数的实现是在单个的文件中。 Git 会优先处理内置命令，由于子命令是紧接着 `git`  出现的，所以先从 commands 表中找到对应命令，如果有则通过  `handle_builtin()` 函数进行，如果不是内置命令，则再去处理其他参数，Git 的处理 flag 的过程是直接解析，并没有采用 getopt 的形式，所谓的直接解析指的是在一个循环中通过 `strcmp` 对参数进行匹配。

在处理子命令时， git 采用类似的思路，即子命令本身的 flag 解析则是将该命令的 flag 定义到 `struct option `   中，将这个 options 传入 `parse_options()` 函数中进行解析， 接着再回到子命令的处理函数中继续处理。 这种处理方式的关键部分是这个 option 结构体，我们可以看一下 struct option 的结构定义：

```c
struct option {
	enum parse_opt_type type;
	int short_name;
	const char *long_name;
	void *value;
	const char *argh;
	const char *help;

	int flags;
	parse_opt_cb *callback;
	intptr_t defval;
};
```

结构的中的前面几个域是比较一目了然的，需要说一下的是 flags， 这个 flags 里面定义该参数的一些行为， 比如： `PARSE_OPT_OPTARG `, `PARSE_OPT_NOARG `  等， 这里面严格区分了是参数可选，有参数， bool型 flag 等。此外，对于某些参数的处理也可以直接调用 callback 进行更复杂的处理。

在 parse_options() 中，根据预定义的 options 表，进行更加细致的处理， 在这里是 parse_options 是一个通用的处理函数。可以类比成 getopt 函数的作用。所有的动作都通过结构体中定义的项进行，相当高效。Git 的这一套机制在后续添加新的命令时，更加的灵活，通过提供一套解析的模板，只需要实现相应的命令各自的处理函数即可，耦合度低。

再回到 C 本身，对子命令的处理也提供一个方案。 在 `stdlib.h`  中提供了一个 `getsubopt`  函数用来处理 子option 的参数，需要搭配 `getopt` 使用。具体的例子可以参考 man 手册中的 EXAMPLE 小节。

从 C 的角度而言，解析的活儿都是自己来干的，这也让我们更加清晰的认识到解析命令行参数的一个基本思路，剩下的就是按需设计想要的方式。如果不想这么麻烦，我们可以参考借鉴一下 Go 以及其他语言的用法，然后自己实现一套。

<hr>

### Go 语言篇

来到高级语言这部分之后，对于命令行的处理似乎能够更加的方便快捷了。 Go 语言中提供的 flag 包，支持一般参数的形式解析以及子命令的解析，在解析的扩展方面 Go 也是很灵活的。

Go 的 flag 标准库基本能满足基本的日常使用，但是仍有一些不太畅快的地方。 毕竟命令行的使用没有一个通用的方式，不同的系统或者说不同的软件所使用的命令行的风格也有所不同。Go 所支持的命令行语法比较简单：

``` go
-flag
-flag=x
-flag x  // non-boolean flags only
```

同时默认为所有的 flag 添加了长 flag 语法。由于这样的语法定义很简单，因此就不支持短 flag 的组合，以及多参数的情况。我们阅读 flag 包的源码就能发现，在解析参数时采用的是一个 for 循环，从头到尾的解析所有的参数，所以 flag 都是定义死的，所以短参数的组合是不能识别的，flag 包会直接给出 help 的信息。硬解析就是这样，只要 flags 不在预定义的 flags 中，则认为错误。

我所说的多参数指的是下面的这样的情况：

```go
$ ./hello -s str1 -s str2
[str2]
```

我们期望 `-s` 能返回 `[str1, str2]` , Go 标准包提供的参数类型中没有复合类型，所以会导致后面解析的参数会覆盖掉前面已经解析保存的参数。好在官方文档中有提供一个自定义类型的解析的示例，这个扩展得益于 Go 的 interface 机制。 所以我们只需要将自定义的类型实现 `Value`  接口即可，这个 interface 的定义为：

```go
type Value interface {
  	String() string
  	Set(string) error
  }
```

也就是说我们需要对自定义的类型实现下 `String 和 Set` 方法。那上面所说的多参数问题，就可以使用定义成 slice 或者其他可以保存多个数据的类型，如何保存就需要在 Set 方法中实现，这一点是很方便的。

此外，flag 包还支持子命令的定义，需要先创建一个 flagset 类型， 然后将子命令的定义添加到刚定义的 flagset 类型中，所使用的方法定义和我们直接使用 flag 包级别的方法是一致的，这个大大降低了开发的心智。

还有一点就是，Go 的 flag 包没有命令行的可选参数和必选参数的概念。有的只是如果没有设置这个 flag 那么就是用默认值，设置了就将该 flag 紧接着的 paramater 解析为需要的参数。这种做法有利有弊，好处在于规范命令行的使用，flag 要么设置（除了布尔型的 flag）， 设置了就必须有值。 但是也带来了一个错误, 比如对后面需要带有参数的 flag 未设置参数，那会导致后面的 flag 被误认为是参数。

总之 Go 这么定义 flag 包，在没有特殊需求的时候可以很快速的完成简单命令行的设计。 对比 `getopt` 就能发现，一旦我们定义好了 flag 的规则，那么解析后返回的结果就可以使用了，而不再需要一个循环去处理每次的返回值。这一点是极大的提高了开发时的效率，但这个效率也是建立在你的命令行处理不复杂的情况下。

Go 标准包功能只是一个基础，功能上并不丰富，所以第三方的包也就有较大的选择空间，一个没有太多学习成本且是 for hunman 的包就很受欢迎，目前比较流行的是 [pflag](https://github.com/ogier/pflag) 、[kingpin](https://github.com/alecthomas/kingpin) 这两种。 两者各有各的特色，需要自己根据实际需求去取舍一下， 在这里就不详细的去叙述两者的实现原理，只补充一点流式函数的使用能够让写代码的人很爽，这一点在 C 中就很难体验到。 同时省去了一大部分的内存管理的烦恼，使用高级语言来编程确实舒服。

最后还有一个比较不错的框架推荐下，[cobra](https://github.com/spf13/cobra)  。 这是一个用来开发命令行工具的框架，一些大的项目也用它来开发，比如 Kubernetes。通过命令直接初始化好一个项目，增加子命令以及在子命令中添加 flag 都非常的方便。 有兴趣的话，可以研究研究。

<hr>

### Python 篇

到了 Python 这边，上面的这些解析的方式 Python 都能驾驭，然而当 [dotopt](http://docopt.org/) 出现的后，基本上算是秒杀其他。用法就是你只要按照规范写好 Usage 信息即可，剩下的你就不用管了。解析之后你将得到一个 dictionary， key 是你 Usage 里面出现的所有和 flag 相关的内容。

基本思路是先将 Usage 部分先解析出 option，之后再根据 argv 来解析实际的参数，这一块实际和前面的解析方法是一致的。所以基本上你能看到使用 dotopt 后，代码就非常的简洁：(代码来自官方 repo 示例)

```python
"""Naval Fate.

Usage:
  naval_fate.py ship new <name>...
  naval_fate.py ship <name> move <x> <y> [--speed=<kn>]
  naval_fate.py ship shoot <x> <y>
  naval_fate.py mine (set|remove) <x> <y> [--moored | --drifting]
  naval_fate.py (-h | --help)
  naval_fate.py --version

Options:
  -h --help     Show this screen.
  --version     Show version.
  --speed=<kn>  Speed in knots [default: 10].
  --moored      Moored (anchored) mine.
  --drifting    Drifting mine.

"""
from docopt import docopt


if __name__ == '__main__':
    arguments = docopt(__doc__, version='Naval Fate 2.0')
    print(arguments)
```

所有的一些就都在 arguments 这个 dictionary 中了，我们只需要按照需求来获取对应 flag 的参数。 使用 dotopt 的前提是你需要熟悉并且按照 POSIX 的方式来定义你的 usage 。 这算是一个学习的成本，一旦掌握了这个定义的方式，那剩下的就是一行代码的事儿。 这基本上可以说是秒杀了一众其他的解析。 作者的一个视频里面也调侃了目前 Python 中使用的比较多的几个解析命令行工具。确实，在 docopt 的实例里面，对 git 的命令解析做了一个示范，确实简洁了很多，我们无需关系解析的细节，只需要关系最终的结果。

此外这一套解析方式已经被移植到其他的语言上了，可以参考这个[链接](https://github.com/docopt)。 出此之外，我个人觉得在 Python 更加容易上手的一个工具是 argparse。 形式类似于上面的 Commander， 简单示例如下：

```python
parser = argparse.ArgumentParser()
parser.add_argument('-s', action='store', dest='src', help="seach a pattern")
parser.add_argument('-p',action='store',default=False, dest='print_lately',type=int,nargs='?',
			help='print n lately words')
...
results = parser.parse_args()
```

这其中 action 表示的是 flag 的属性，是带参数的还是一个布尔型的。 dest 是对每一个 flag 所做的操作的保存，比如 `result.src` ， 此外还有其他的属性可以设置。具体的可以参考 argparse 的手册。 同时我们也能发现这样的做法实际上和我们在 C 中定义结构体的方式是类似的，只是不同的语言所实现的方式不一样罢了。

<hr>

### 总结

经过上面这一通整理，对命令行的解析以及相关工具的使用应该是比较清晰了。使用现成的库或者包进行命令行解析，可以提高效率，减轻一些心智负担，但是通过手动去写解析方式能够带来更高的灵活性。工具只是一种解决问题的方式，明白其中的实现原理，造轮子也只是时间的事儿。



(全文完)



