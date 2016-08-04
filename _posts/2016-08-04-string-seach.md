---
title: String Searching Algorithm -- Part1
updated: 2016-08-04 18:00
---

今天在写一个有关C语言字符串操作的一些通用函数动态库。C语言中对字符串的处理的函数比较少，很多需要自己去实现。比如`substr`,`sub`等,不过好在标准包里面提供了一些不错的函数,所以在这些函数上进行一些封装,那些功能也都能实现。所以将这些基本的函数做成一个库，这样下次使用的时候也会比较的方便。

不过在实现这个这些字符串函数的时候，不免需要碰到字符串查找，所以决定写一篇博客来表达一下自己对这几种的搜索算法的看法。这一篇写两个比较简单的：`朴素查找(暴力查找)`，`Rabin-Karp`。 这两个在实现和理解上是很简单的，后面一篇会谈及`Boyer-Moore（BM）`和`KMP`。 这两个中，前者效率比较高，实际使用的比较多。下面就开始说说最简单的两个。

在此先约定一下我们用于查找的字符串用`s`（source）表示，比如："hello world", 而用于查找的字符串用`t`（target）表示,比如"llo"，二者的长度分别用`slen`,`tlen`来表示。

## 朴素查找

这个方式是最贴近我们认为查找的，思路：从`s`的第一个字符开始，逐个字符的对比`t`中的字符，这个过程中有一个不同，则将`t`往后移动一个字符，从`s`的第二字符开始，重复之前的动作。直到`t`中的所有元素和`s`的某一块连续的子串都匹配上时停止，同时将与`t`的开头对应着的`s`中的字符的`index`返回。如果未能匹配到，则返回`0`.

所以要实现这样的一个算法，需要两个循环，外循环控制着从`s`中哪个位置开始往后与`t`对比。内循环则是做字符串匹配。所以我们可以这么简单的实现如下的算法：

```c
int Navie_Search(const char *s,const char *t){
	int slen = strlen(s);
	int tlen = strlen(t);
	int i, j;

	for(i = 0; i< (slen-tlen);i++){
		for(j = 0; j < tlen; j++){
			if(s[i+j] != t[j])
				break;
		}
		if(j == tlen)
			return i+1; /*index start for 1*/
	}

	return 0;
}
```

这就实现好了朴素查找，这个算法的复杂度是`O((n-m)m)`, 效率是不太高的，不过实现起来很简单。接下来是`Rabin-Karp`算法。

## Rabin-Karp

这个算法的核心思想在于将字符串进行了hash值运算，如果hash值不一样,就没有必要进行对比了。而这个算法的其他部分则和朴素查找没有什么区别。下面简述一下这个算法的过程：

+ 计算出`s`长度和`t`一样的子串的hash值，以及`t`的hash值
+ 遍历`s`,`t`
+ 如果`s`子串的hash值和`t`的hash值不相等，则移动`s`，形成新的子串并重新计算hash。
+ 如果hash值相等，则对比这两个子串是否完全一样。

首先计算出`tlen`长度的`s`和`t`的hash值`hashValS`,`hashValT`，对于`t`而言只要计算一次即可。而`s`由于需要不断的移动，所以每次的子串是不一样的，则hash值需要重新计算。

计算hash值，首先需要选定一个基底,比如`256`,所以对"abc"这个字符串计算hash值则为:`hash("abc") = 97*256^2 + 98*256 + 99`。 如果子串的长度比较长的话，最终这个hash值会很大，比较容易溢出，所以需要考虑将这个值缩小，一般是模上一个素数，比如:`hash("abc") %= 101`。 

现在还需考虑的是更新`s`子串的hash值，这边有一个trick：不需要重头在计算一遍`s`子串的hash，而只要减去头部字符的hash值，加上结尾字符的字符的hash值即可。比如`s`为"abcd"，`t`为"bcd"。先前我们已经计算"abc"的值了。那么接下来计算"bcd"的时候，我们只要这样：

```c
hash("bcd") = 256*(hash("abc")-97*256) + 100
```

这样可以节约一定的时间。 这两个关键部分解决后，这个函数的实现便不是很复杂了：

```c
#define BASE 256

int RK_Search(const char *s, const char *t, int q){
	int slen = strlen(s);
	int tlen = strlen(t);
	int i, j;
	int h = 1;  /* for store the low(256,tlen-1)%q */
	int hashValS = 0;
	int hashValT = 0;

	for(i = 0; i< (tlen-1); i++){
		h = (h*BASE)%q;  /*Notice:h has moded q already*/
	}

	for(i = 0; i < tlen; i++){
	    hashValS = (hashValS*BASE + s[i])%q;
		hashValT = (hashValT*BASE + t[i])%q;
	}

	for(i = 0; i<(slen-tlen); i++){
	  	if(hashValS == hashValT){
			for(j = 0; j<tlen;j++){
		        if(s[i+j] != t[i])
				   break;
		    }

			if(j == tlen)
			   return i+1;
	    }else{	
			hashValS = (BASE*(hashValS - s[i]*h) + s[i+tlen])%q;
			if(hashValS < 0){
				hashValS += q;
			}	
		}
	}

	return 0;
}
```

这个算法的主要损耗在计算hash值上，其他部分和朴素查找是一样的。所以这个算法的时间复杂度，平均而言是`O(m+n)`,而最差的是`O(m(n-m))`,所以在一定程度上,`Rabin-Karp`算法比朴素的算法要快。

不过我在看`golang`的strings部分的源码时发现，它内部也使用了RK算法，也是用来查找子串的，下面顺手也贴一下这部分的代码：

```go
hashsep, pow := hashStr(sep)
h := uint32(0)
for i := 0; i < len(sep); i++ {
	h = h*primeRK + uint32(s[i])
}
lastmatch := 0
if h == hashsep && s[:len(sep)] == sep {
	n++
	lastmatch = len(sep)
}
for i := len(sep); i < len(s); {
	h *= primeRK
	h += uint32(s[i])
	h -= pow * uint32(s[i-len(sep)])
	i++
	if h == hashsep && lastmatch <= i-len(sep) && s[i-len(sep):i] == sep {
		n++
		lastmatch = i
	}
}
```
这段代码是用来统计在字符串`s`中有多少个`sep`子串。

代码中的`hashstr`是计算了字符串sep(也就是我们说的t)的hash值和高位的幂(我实现的代码里面的h)。接下来就是计算`s`的最开始的`len(sep)`长度子串的hash值。接下来先做了一次对比，如果hash值相等且完全匹配，则记录下最后匹配时候的位置，并将次数加一。否则进入一个循环,依次查找。循环内的部分和上面C实现的部分是类似的，不过golang中slice的比较直接使用`==`即可。所以实现的基本过程一样。 不过golang中使用`primeRK = 16777619`。


## Reference

1.[Rabin-Karp Algorithm WikiPedia](https://en.wikipedia.org/wiki/Rabin%E2%80%93Karp_algorithm)
2.[Golang Strings Source Code](https://github.com/golang/go/blob/master/src/strings/strings.go#L97)
3.[Golang Slices Usage and Internal](https://blog.golang.org/go-slices-usage-and-internals)


