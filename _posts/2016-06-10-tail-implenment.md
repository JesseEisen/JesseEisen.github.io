---
title: Tail Command implementation
updated: 2016-6-16 18:00
---

> 原本只是想用C/Python实现下tail，但是在上班的路上，脑子里冒出了一个想法，用Lua/Go/Haskell也来实现一下。
> 但是对Haskell只是初步的学习，还是有点怵，不过还好最后还是实现了。所以下面就会有5个语言的tail -n。^_^

如题，这篇博客是来说说实现`tail`命令，当然了是实现的tail的最基本的功能。即`tail -n`

## C版本

之前在学C的时候，重新实现cat，tail之类的命令，也写了一些。论实现一个tail，粗暴的方式也不少。比如：在C语言中，一次性将所有的文件内容都读到一个buf中，最后只取这个buf的最后几个结果输出。

思路上很简单，不过我们不知道这个文件有多少行，所以这个保存结果的buf的大小该定义多少？也许你会说，使用`wc`来获取到文件的函数，然后动态分配内存。
但是考虑到可移植性的话，系统调用在不同的平台上有所不同，这个方式并不是一个好的选择。

实际上，我们并不需要将文件的所有内容都保存到buf中，我们只要保存指定的`n`行，或者是默认的`10`行即可。这样我们的buf的空间大小可以通过对命令行参数的解析即可确定下来。

接下来就是往这个buf中填充内容：

+ 如果文件比较小，没能填满整个buf，那么就直接输出buf中的内容。
+ 如果文件比较大，buf被填满了，那么下一个读取的内容我们就其放到buf的第一个位置上，以此类推，最后记录下最后一行放进buf中的位置。

读取结束后，我们所要的内容都在这个buf中，只是还需要对这个buf做一个分析

+ buf未满
+ buf满了，最后一行刚好是在buf的最后一个位置上
+ buf满了，最后一行不在buf的最后一个位置上

第1,2两种情况没什么好说的，直接输出相应的内容，对于第3中情况，我们需要先从最后一行放进去的下一个位置开始输出，输出到buf的最后。 接着从buf的开始输出到最后一行放进去的位置。 这样就按照原顺序将文件内容输出了。

下面这段是读入文件内容到buf的简单实现：

```c
while((len = getlines(line,MAXLEN)) > 0){
	if(end <= total){
		lineptr[end] = malloc(sizeof(char) * len);
		strcpy(lineptr[end], line);
		end++;
	}else{
		end = 0;
		isfull = 1;
		lineptr[end] = malloc(sizeof(char) * len);
		strcpy(lineptr[end],line);
		end++;
	}
}

```

`end`记录的是最后一行放进buf中的位置，`isfull`标记这个buf是否满了，这个标记决定接下来的输出。`getlines`是自己实现读入文件的函数。

如果将结果输出，代码如下：

```c
void show_result(char **res, int end, int total,int isfull)
{
	int i;

	if(isfull == 0){
		for( i = 0; i< end; i++)
			printf("%s",res[i]);
	}else{
		if(end == total){
			for(i = 0; i < end; i++)
				printf("%s",res[i]);
		}else{
			for(i = end; i<total; i++)
				printf("%s",res[i]);
			for(i = 0; i< end; i++){
				printf("%s",res[i]);
			}
		}
	}

}

```

上面这段代码比较浅显，就不多做解释了。完整的代码放在[JesseEisen Gist](https://gist.github.com/JesseEisen/710fdce0971dc489965f352542865a48)

> 补充：在实现Lua的时候，想到在保存Line的时候，实际上对下标的处理可以改成`i%n` 这样保证下标只会在0～n之间，最后输出的时候也不需要那么的复杂。
> 具体的可以看Lua的实现。

## Python版本

Python的实现就更简单了，使用内置的`collections`模块中的`deque`. `deque`是一个双端队列，好处是能够从头和尾取数据和添加数据。下面引用一下文档上的一个描述：

> If maxlen is not specified or is None, deques may grow to an arbitrary length. Otherwise, the deque is bounded to the specified maximum length. Once a bounded length deque is full, when new items are added, a corresponding number of items are discarded from the opposite end. Bounded length deques provide functionality similar to the tail filter in Unix. They are also useful for tracking transactions and other pools of data where only the most recent activity is of interest.

`class collections.deque([iterable[, maxlen]])` 这是deque的定义， 上面的那段描述，简要的说就是：如果`maxlen`提供了。那么返回的值只包含`maxlen`个值。如果没有指定的话，那就输出所有的结果。 最后也提到了`Bounded length deques provide functionality similar to the tail filter in Unix`.所以使用`deque`是一个正确的方向。

还有的一个好处是：

> Deques support thread-safe, memory efficient appends and pops from either side of the deque with approximately the same O(1) performance in either direction.

复杂度很低，所以这就在文件比较大的时候，性能上能够保证。

下面放上python实现的代码：

```python
import sys
from collections import deque

# first read the file
def tail(filename="default.txt", n=10):
	while True:
		for lines in list(deque(open(filename),n)):
			print lines.strip('\n')
		break

argc = len(sys.argv)
if argc == 1:
	tail()
elif argc == 2:
	tail(sys.argv[1])
else:
	tail(sys.argv[1],int(sys.argv[2]))

```

除去对命令行的解析，实现tail只有5行的代码。有内置模块确实省不少力。这个代码不是很复杂，关键就`list(deque(open(filename),n))`.

## Lua版本

利用Lua实现tail，算是比较顺畅的，没有包的引用，内置函数也比较到位。 思路是和C版本的类似，一行一行的读取文件内容，并将每一行放到buf中，Lua里面是放到table中。table的大小是`n`,命令行解析得到。在将内容放入table中的时候，使用到了一个小的技巧。 直接看代码会更好：

```lua
buffer={}

--解析命令行参数
if #arg == 1 then
	n = tonumber(arg[1])
	if n == nil then
		io.write(string.format("Usage:%s -n input\n"),arg[0])
		os.exit(1)
	end
end

--读取文件，存入buffer中
i = 0
while true do
	local line = io.read()
	if not line then
		break
	end
	buffer[i % n] = line
	i = i + 1
end

--打印出最终结果
for j = 0,n-1 do
	local pos = (j + i) % n
	if buffer[pos] then
		print(buffer[pos])
	end
end
```

实际上上面的C版本设计的有点臃肿。Lua的文件读取，以及命令行处理是非常的简单。真是一个好语言！

## Go版本

在Go中我没有找到类似Python中的`deque`的结构，不过Go也许是更加的直接，此处没有沿用C和Python中的方式，原因是Go里面提供了不少的文件读取函数，通过比较发现，使用包`io/ioutil`的效率比较高。

Go的直接在于：利用`ioutil.ReadFile(filename)` 直接将文件的内容都读入到内存中。之后将返回值变成一个字符串数组即可。所以Go的关键代码就两行:

```go
f,err = ioutil.ReadFile(filename)
... //省略错误检查
s = strings.Split(string(f),'\n')
```
`s`就是最后的字符串数组，从数组中获取最后的n行，应该是相当的简单的了。 完整代码放在[golang version tail implementation](https://gist.github.com/JesseEisen/aec959e6ccf7da486f45fbd19b2bc9e2)
顺便说一下，在处理命令行参数的时候，使用`flag`包和手动处理真的差不少。不过和Lua的那种简单粗暴比还是差了点意思。

> Go学的不是很到家，如果你有什么不错的惯用法，或者高效的处理方式，希望能够分享一下～

## Haskell

实际上，使用Haskell对我而言，只是一个挑战，因为我只是在`ghci`中敲了些命令，同时学习了一些有关`list`的一些相关操作，以及简单的递归函数实现。对类型推断还处于模糊的状态中。实现过程中查阅了有关`文件读写`和`命令行处理`相关的文档。

写tail的目的很明确， 读取文件，输出指定的行数。 所以首先是读取文件，Haskell中有一个`IO`的模块。使用起来也不是很复杂：

```haskell
handler <- openFile "filename.txt" ReadMode   --以读方式打开一个文件,得到一个文件句柄
contents <- hGetContents handler              --利用hGetContents 方法获取到文件的所有内容
let raw = [x | x <-(lines contents)]          --将contents转换成一个list
```

一旦内容变成了list后，剩下的就是list操作。Haskell中操作list的函数还是很多的，比如：`take` `last` `drop` 等等。
这边我们使用`drop`, drop是取列表剩下的元素。一个例子就能明白了：

```haskell
drop 3 [1,2,3,4,5,6]  --返回[4,5,6]
```

所以我们所要做的就是取剩下的n个元素：

```haskell
let total = length(raw)   --获取整个list的元素的个数
drop (total-n) raw        --返回就是最后剩下的元素
```

最后就是打印出list的元素，我用了一个递归函数进行打印的：

```haskell
printItems :: [String] ->IO()
printItems [] = putStr ""     --递归终止时，什么都不打印
printItems (y:ys) = do        --如果list中有元素，则执行下面的打印
	putStr (y ++ "\n")
	printItems ys             --递归调用
```

最后还有一个命令行参数的处理，引入`import System(getArgs)` 接着就是:

```haskell
args <- getArgs
let n = read (args !! 0) :: Int  --将第一个参数转成Int
```

这样Haskell的版本已基本实现了。上面的写法也许不够纯，但是能正确实现，也是挺宽慰了。Haskell的道路才刚刚开始……


## 参考

+ [python document about collections](https://docs.python.org/2/library/collections.html#)
+ [Haskell IO](http://rwh.readthedocs.io/en/latest/chp/7.html)
+ [Haskell Read file](http://stackoverflow.com/questions/7867723/haskell-file-reading)
+ [Golang Command line](http://www.nljb.net/default/Golang-%E4%BD%BF%E7%94%A8%E5%91%BD%E4%BB%A4%E8%A1%8C%E5%8F%82%E6%95%B0/)


(全文完)
