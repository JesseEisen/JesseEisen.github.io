C 语言中， string 的操作函数不是很多，很大一部分需要我们自己去组合使用。同时 C 语言中的 string 由于有`\0` 的存在，这一点在内存分配以及拷贝时需要格外小心，同时一些库函数在不注意的情况下引入一些 bug。所以总结一下在编程时需要注意的点以及如何更好的使用 string 相关函数。

我们大体上可以将 string 的操作函数分为几类： `mem-` 打头的函数是操作任意的字符，不考虑是否包含尾零； `str-` 打头的函数是会操作带空字符的字符串的，而 `strn-` 打头的函数则只操作非空字符的字符串。 这么说可能不是很具体。 下面通过一些详细的说明来阐述这些问题。

###  常用函数

+ strcpy 和 strncpy

这两个函数的作用都是用来复制字符串的，所不同的是 strcpy 在 dst 的长度小于 src 时，会产生不确定的错误。使用 gcc 编译时，程序不会错误，但是会发生溢出。在其他编译器下，可能运行会产生错误。 strncpy 则是最多拷贝 n 个字符，如果 n 大于 src 的长度，则剩余长度会使用 `\0` 填充，这会产生一些 bug。此外这一点从侧面表明 strncpy 在某些情况下比 strcpy 效率要慢一些。 同时如果 n 等于 src 的长度，则不会在最后添加 `\0` 。所以在使用这两个函数时最好的方式是：

```c
int slen = strlen(src);
char *dst = malloc(slen+1);
strcpy(dst, src);
//strncpy(dst, src, slen+1);
```

后面的 `+1` 作为一种提示，我们是多分配了一个字节用于保存`\0` 的。 而更加安全的复制函数可以使用 `strlcpy` 这个库函数，不过这个只在 freebsd 上支持，其他的系统上还不是很支持。不过我们可以参考 fressbsd 的实现，自己封装一个 strlcpy。

```c
size_t
mstrlcpy(char *dst, const char *src, size_t dsize)
{
    const char *osrc = src;
    size_t nleft     = dsize;

    if (nleft != 0) {
        while (--nleft != 0) {
            if ((*dst++ = *src++) == '\0')
                break;
        }
    }
    if (nleft == 0) {
        if (dsize != 0)
            *dst = '\0';
        while (*src++)
            ;
    }

    return (src - osrc - 1);
}
```

注意函数的**最后一个参数是 dst 的长度**，所以最多会复制 dsize-1 个字符到 dst 中，剩下的则为 `\0` 。 这就规避了 strncpy 中如果 src 的长度比指定的长度要长的情况下，剩下的字符用 `\0` 填充的情况。 同时也避免了 strcpy 中会多复制的情况。

+ strcat 和 strncpy 

这两个函数主要是用来字符串拼接。这两个函数同样有上面的问题，当需要拼接的 dst 的内存不足以容纳 src 时，会产生不期待的结果。即使是 `strncpy` 也是会有安全的问题的。 man 手册上对 strcpy 进行了相关的说明。只要注意在拼接前保证 dst 的空间足够。此外我们可以参考下 `strlcat` 的实现。 

```c
size_t
strlcat(char *dst, const char *src, size_t siz)
{
	char *d = dst;
	const char *s = src;
	size_t n = siz;
	size_t dlen;

	/* Find the end of dst and adjust bytes left but don't go past end */
	while (n-- != 0 && *d != '\0')
		d++;
	dlen = d - dst;
	n = siz - dlen;

	if (n == 0)
		return(dlen + strlen(s));
	while (*s != '\0') {
		if (n != 1) {
			*d++ = *s;
			n--;
		}
		s++;
	}
	*d = '\0';

	return(dlen + (s - src));	/* count does not include NUL */
}
```

这边的 size 是指的 dst 的 size。不过这个函数还是有当 dst 是指针时，想单独通过指针来计算 dst 的实际长度是不太行的。可以通过指定一个长度的参数传入，如果是数组的话，可以通过 sizeof(dst) 计算。 所以在使用这一类的函数时，尤其要小心出现溢出的问题。

