---
title: Tail Command implementation
updated: 2016-6-10 18:00
---

如题，这篇博客是来说说实现`tail`命令，当然了是实现的tail的最基本的功能。即`tail -n`


## C语言版本

之前在学C的时候，重新实现cat，tail之类的命令，也写了一些。论实现一个tail，粗暴的方式也不少。比如：在C语言中，一次性将所有的文件内容都读到一个buf中，最后只取这个buf的最后几个结果输出。

思路上很简单，不过我们不知道这个文件有多少行，所以这个保存结果的buf的大小该定义多少？也许你会说，使用`wc`来获取到文件的函数，然后动态分配内存。
但是考虑到可移植性的话，系统调用在不同的平台上有所不同，这个方式并不是一个好的选择。

实际上，我们并不需要将文件的所有内容都保存到buf中，我们只要保存指定的`n`行，或者是默认的`10`行即可。这样我们的buf的空间大小就可以通过对命令行参数的解析即可确定下来。

接下来就是往这个buf中填充内容：

1.如果文件比较小，没能填满整个buf，那么就直接输出buf中的内容。
2.如果文件比较大，buf被填满了，那么下一个读取的内容我们就其放到buf的第一个位置上，以此类推，最后记录下最后一行放进buf中的位置。

读取结束后，我们所要的内容都在这个buf中，只是还需要对这个buf做一个分析

1.buf未满
2.buf满了，最后一行刚好是在buf的最后一个位置上
3.buf满了，最后一行不在buf的最后一个位置上

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


## 参考

1.[python document about collections](https://docs.python.org/2/library/collections.html#)



(全文完)
