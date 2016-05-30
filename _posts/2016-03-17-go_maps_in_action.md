---
title: Go map in action
updated: 2016-03-17 15:57
---

> 本文翻译自 [Golang Blog|Go map in action](https://blog.golang.org/go-maps-in-action)

## 介绍

在计算机科学里面,hash表是最有用的数据结构之一.hash表有很多的不同实现方式,但是他们都有一些共同的特性:快速查找,添加和删除 Go提供了一个内置的map类型来实现hash表



## 声明和初始化

Go的map类型有如下类似的样子：

```
	map[KeyType]ValueType
```

`KeyType`可以是任何**可比较**的类型,`ValueType`可以是任何类型，包括map类型！

> 可比较类型：[https://golang.org/ref/spec#Comparison_operators](https://golang.org/ref/spec#Comparison_operators)

下面这个变量`m`就是一个键为`string`类型而值为`int`

```
	var m map[string]int
```

`Map`类型是引用类型，像：pointer或者slices. 所以上面的`m`的值为`nil`.它并没有指向一个初始化的map。在读一个`nil`的map，就像是在读一个空map， 但是如果尝试去写一个`nil`的map 的时候，将会引发一个运行时panic。

所以最好不要这么做，**初始化**一个map，可以使用内置的`make`函数：

```
	m = make(map[string]int)
```

`make` 函数分配空间并初始化一个hash map的数据结构，然后返回一个map值指向那个数据结构。
这个数据机构的具体内容是由runtime的细节实现的。不是语言本身的。在本文中将关注如何使用`map`,而不是它的实现。

## 使用maps

Go 提供通用的语法来使用maps， 下面这个表达式而设置了键`route`的值为66:

```
	m["route"] = 66
```

下面的这个表达式重新获取到键`route`的值，并把它赋给一个新的变量`i`

```
	i := m["route"]
```

r如果请求的键不存在，我们将得到一个值类型的`zero value`（不一定就是0），下面这个例子中，值的类型是`int` ，所以它的零值就是`0`

```
	j := m["root"]
	//j == 0
```

内置的函数`len`返回map的项的数量

```
	n := len(m)
```

内置的函数`delete` 从map中移除一个成员

```go
	delete(m."route")
```

函数`delete`不会返回任何东西，所以如果指定的键不存在，函数也不会做任何事儿。

`two-value`表达式测试一个键是否存在

```go
	i, ok := m["route"]
```

在上面的这个表达式中，i将保存键`route`的值，如果这个键不存在，那i就是`zero value`. 第二个值(ok)是一个`bool`, `true`表示这个键存在，`false`表示这个键不存在。

仅仅为了测试这个键的值是否存在，可以使用下划线来代替第一个位置：

```go
	_, ok := m["route"]
```

可以使用`range`来迭代map的的内容：

```go
	for key, value := range m {
		fmt.Println("Key:",key, "Value:",value)
	}
```

想用一些值初始化一个map，可以使用map字面值来进行：

```go
	commits := map[string]int{
		"rsc":3711,
		"r":2138
		"gri":1908
		"adg":912,
	}
```

可以使用相同的语法来初始化一个空的map，这个和使用`make`是一样的。

```
	m = map[string]int{}
```

## 探索zero 值

当一个键不存在时返回一个zero value时非常方便的。

比如，一个布尔值的map可以被用来实现一个`set-like`的数据结构（回忆下，zero value 对布尔值而言是false）。比较典型的例子是，遍历一个链表`Nodes` 并且打印出他们的值。使用`Node`的map来检测一个循环。

``` go
type Node struct {
		Next  *Node
		Value interface{}
}
var first *Node

visited := make(map[*Node]bool)
for n := first; n != nil; n = n.Next {
		if visited[n] {
				fmt.Println("cycle detected")
				break
		}
		visited[n] = true
		fmt.Println(n.Value)
}

```

如果`n`被访问过，那么表达式`visited[n]`就是true。如果这个n不存在，那么这个表达式就是false. 不需要使用`two-value`的形式来检测n是否存在于map中。`zero value`默认为我们做了这些。

另一个使用`zero value` 比较好的例子是有关`slice`的map.往一个`nil`的slice里面添加内容，只需要分配一个新的slice。所以这是一个线性的去往slice的map里面追加值。不需要检查这个这个key是否存在。 

下面的这个例子，people是一个`person`的slice， 每一个`Person` 有一个`Name`和一个Like的slice。这个例子创建了一个map来连接喜好和people slice（people所喜欢的）

``` go 
	 type Person struct {
        Name  string
        Likes []string
    }
    var people []*Person

    likes := make(map[string][]*Person)
    for _, p := range people {
        for _, l := range p.Likes {
            likes[l] = append(likes[l], p)
        }
    }

```

打印出喜欢chess的人：

``` go
	for _, p : reange like["cheese"] {
			fmt.Println(p.Name, "like cheese")
	}
```

打印出喜欢bacon的人的数量：

```
	fmt.Println(len(likes["bacon"]), "people like bacon.")
```

需要注意的是，range和len都将nil slice看作是zero-length的slice。 所以在没有人喜欢cheese或者bacon的时候上面的那两个例子也是可以工作的。

## 键类型

在之前提到过，map的key可以是任何可比较的类型。官方的语言spec很精准的定义了这个概念。 简单的说，可比较类型是：boolean，numeric，string， pointer， channel，和 interface,以及 struct 或者是array。 值得注意的是：slices，maps 和 functions 这些类型是不可以用`==`，所以他们不能作为map的key。

string，int 和其他的基础类型都是可以作为key的，可能对struct作为key而言会有些出乎意料
struct可以在多维的基础上用作key。 比如，下面的这个map的map可以用在匹配不同国家的网页

```
	hits := make(map[string]map[string]int)
```

这是一个string到`string到int map`的map，每个外部map的键都是网页地址对应这它内部的map。 每个内部的map的键是2个字符的国家码。 下面的这个表达式取出了Australian加载了多少次这个网页

```
	n := hits["/doc/"]["au"]
```

不幸的是，这个方法在添加数据的时候变得比较麻烦。对任意一个给出的外部的键，你必须要检查一下它的内部键是否纯在，并且在需要的时候创建它：

``` go
func add(m map[string]map[string]int, path, country string){
		mm, ok := m[path]
		if !ok {
				mm = make(map[string]int)
				m[path] = mm
		}

		mm[country]++
}

add(hits, "/doc/", "au")

```

所以使用一个单个的struct的map就省去了这些复杂的东西了：

```go
type key struct {
		Path, Country string
}

hits := make(map[Key]int)

```

当一个越南(vietnamese)人访问了主页，增加对应的访问量只要一行代码：

```go
	hits[Key{"/","vn"}]++
```

同样，如果想查看多少瑞士人访问了spec：

```go
	n := hits[Key{"/ref/spec","ch"}]
```

## 并发

在并发上使用map是不安全的：当你同时去读和写他们的时候，结果是未定义的。如果你需要从并发执行的goroutines上读一个map和写一个map，你必须使用一些同步机制。一个最常见的去保护map是用[sync.RWMutex](https://golang.org/pkg/sync/#RWMutex)

下面的语句声明了一个`counter`变量，他是一个匿名的结构体，包含了一个map和一个内嵌的`sync.RWMutex`

``` go
	var counter = struct {
			sync.RWMutex
			m mapp[string]int
	}{m: make(map[string]int)}
```

想要从counter中读数据，设置一个读锁

```go
	counter.RLock()
	n := counter.m["some_key"]
	counter.RUnlock()
	fmt.Println("some_key",n)
```

想往counter中写数据，设置一个写锁

```go

	counter.Lock()
	counter.m["some_key"]++
	counter.Unlock()
```

## 迭代顺序

当使用range来迭代一个map的时候，迭代的顺序没有被指定，并不能保证是同样从一个迭代到
另一个。自从GO 1开始，运行时是map用的随机的迭代顺序，而程序员们是需要一个稳定的迭代顺序的。如果你需要一个稳定的迭代顺序，你需要维护一个单独的数据结构来指明这个顺序。

下面的这个例子使用了一个单独的有序的slice作为key去顺序打印`map[int]string`

```go

	import "sort"

	var m map[int]string
	var keys []int

	for k := range m {
			keys = append(keys,k)
	}

	sort.Ints(keys)

	for _,k := range keys {
			fmt.Println("key:",k,"value:",m[k])
	}
```

> 译注：这也许就是为什么key需要是可比较的类型的。



