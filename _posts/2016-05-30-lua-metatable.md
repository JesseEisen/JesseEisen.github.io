---
layout: post
title: Lua Metatable Tuorial
updated: 2016-05-30 12:00
description: lua metatable
---

> 本文翻译自：[Lua Metatables Tutorial](http://nova-fusion.com/2011/06/30/lua-metatables-tutorial/)

我将会在这篇教程中讲到在 Lua 中一个非常重要的概念：`metatable` . 知道如何使用 metatable 后会让你更好的使用 Lua。每个 table 都可以有一个 metatable。matetable 是通过某些 key 来改变它所绑定的 table 的一些行为的。让我们来看一个例子：

```lua
t = {} -- 正常定义的一个 table
mt = {} -- metatable， 不过现在没有包含任何东西
setmetatable(t,mt) -- 设置 mt 成为 t 的 metatable
getmetatable(t) -- 这将返回 mt
```

我们可以发现：`getmetatable` 和`setmetatable`是两个主要的函数，很明显从字面就可以看出他们是做什么的。当然，我们可以将上面的前三行写成如下的样子：

```lua
t = setmetatable({},{})
```

`setmetatable` 返回它的第一个参数，所以我们可以用这样的惯用法。

我们可以在 metatable 中放些什么呢？metatable 中可以放任何的东西，但是他们需要和以`__`（两个下划线）开头的 key 对应。比如：`__index` 和`__newindex`. 这些 keys 对应的值通常是函数或者是其他的表。如下：

```lua
t = setmetatable({},{
  __index = function(t, key)
  	if key == "foo" then
  		return 0
  	else
  		return table[key]
  	end
  end
})
```

上面我们将一个函数赋值给了`__index`现在让我们来看看这个关键字到底是关于什么的。

### __index

使用的最多的 metatable 的 key 就是`__index`. 它可以包含其他的函数和表。

当我们用一个 key(table 是 key-value 形式）在表中搜索是，不管这个 key 是什么 (t[4],t.foo 或者 t["foo"]). 当值没有赋给这个 key 的时候，Lua 会在 table 的 metatable 中寻找`__index`, 如果`__index`包含在这个表中，Lua 将在属于`__index`的表中寻找对应的`Key`. 这也许看起来比较的绕，让我们看一个例子：

```lua
other = {foo = 3}
t = setmetatable({}, {__index = other})
t.foo   -- 3
t.bar   -- nil
```

如果`__index`包含一个函数，那这个函数将会被调用。这个函数的参数是我们用于遍历的表以及我们要搜寻的`key`（译注：这边的 key 是指 table 的 key，而不是指 metatable 的关键字）. 我们从上面的倒数第二个例子可以看出，在这个`__index`关键字内我们可以使用条件判断，以及任何 Lua 代码能够做到的。因此，在那个例子中，如果 key 等于"foo", 我们就返回`0`, 否则我们就用那个 key 去搜寻`table`表。我们用`t`作为`table`的一个别名，如果这个 key`foo`被使用就返回 0.

你也许会疑惑为什么 table 会作为`__index`函数的第一个参数，这很好解释，如果你在多个表中使用这个 metatable，他支持了代码的复用以及节约了一定的电脑资源。我们将在`Vector`类中像你展示这些。

### __newindex

下一个是`__newindex`, 这个和`__index`类似。和`__index`一样，他可以包含函数或者其他的表。

当你尝试去给一表设置不存在的键值得时候，Lua 会搜索是否`__newindex`关键字存在于这个 metatable 中，它的步骤和`__index`一样。如果`__newindex`是一个表，那么这个 key 和 value 将会被设置在`__newindex`所指定的表中。

```lua
other = {}
t = setmetatable({},{__newindex = other})
t.foo = 3
other.foo   -- 3
t.foo  -- nil
```

如果`__newindex`是一个函数，那么这个函数将被调用，key 和 value 将作为参数传入

```lua
t = setmetatable({}, {
	__newindex = function(t,key,value)
		if type(value) == "number" then
			rawset(t,key, value * value)
		else
			rawset(t,kye,value)
		end
	end
})

t.foo = "foo"
t.bar = 4
t.la = 10
t.foo  -- "foo"
t.bar -- 16
t.la  -- 100
```

当在 t 中创建一个新的 key 的时候，如果这个 value 是一个 number，那他将被平方，否则他都会被设置。
同时这个例子也引出了`rawget` 和`rawset`

### rawget 和 rawset

他们是在我们需要设置和获取表的 key 的时候，不想让 Lua 将 metatable 引进来。正如你猜的那样，`rawget`允许你获取这个`key`所代表的值，同时让 Lua 不是使用`__index`. `rawset`让你设置这个`key`的 value，同时不会去使用`__newindex`（这些并不会对速度上有神提升，如果使用常规做法的话）。当你陷入无限循环的时候你可能需要使用他们。举个例子，上面的那个代码例子中，`t[key] = value * value` 将会在`__newindex`中设置一遍，这将会让你陷入无线循环中。使用`rawset(t,key, value * value)`会避免这个。

我们需要将目标 table，key 和 value 都传给我们要使用的那些函数。

### 运算符

很多的 metatable 的关键字可以和运算符搭配使用 (+,- 等），允许你让 table 能够让 table 可以支持那些运算符。比如，我们想让 table 可以能够相乘，我么可以这样做：

```lua
t = setmetatable({1,2,3}, {
    __mul = function(t, other)
     new = {}

     for i = 1,other do
         for _,v in ipairs(t) do table.insert(new,v) end
     end

     return new
    end
})

t = t * 2 -- {1,2,3,1,2,3}
```
这让我们可以通过乘法运算来实现将 table 重复指定次数的功能。对应于乘法运算的关键字是`__mul`, 和`__index`和`__newindex`不同的是，和运算符有关的关键字只能接受 function。函数的第一个参数永远是目标 table，然后接下来的参数就是在操作符右边的那个（除了__unm）, 下面是支持的运算符的：

+ __add: 加法 (+)
+ __sub: 减法 (-)
+ __mul: 乘法 (*)
+ __div: 除法 (/)
+ __mod: 取模 (%)
+ __unm: 一元运算 -，取负数
+ __concat: 连接 (..)
+ __eq: 相等 (==)
+ __lt: 小于 (<)
+ __le: 小于等于 (<=)

（这边只有 ==,<,<= 是因为你可以通过这三个实现所有的对比操作的功能，事实上只有 == 和《是必需的）

### __call

下一个是关键字`__call`, 这让你将 table 当作函数来调用，一个代码例子：

```lua
t = setmetatable({},{
    __call = function(t,a,b,c,whatever)
        return (a + b + c) *whatever
    end
})

t(1,2,3,4)  --24
```
这个函数和上面的一样，传入目标 table，后面跟着我们要传入的参数。

`__call`在很多情况下是非常有用的。一个比较常用的就是将一个 call 转发给 table 内的一个函数，我们可以在 [tween.lua](https://github.com/kikito/tween.lua/blob/master/tween.lua#L339) 库里面找到这个用法。`tween.start`能够被`tween`这个表自己调用。另一个例子是 [MiddleClass](https://github.com/kikito/middleclass/blob/master/middleclass.lua#L64), 类的`new`方法能够被类自己调用。

### __tostring
最后一个介绍的是`__tostring`. 如果实现了这个，他将会通过 tostring 将这个 table 转换成一个 string，最常用的就是使用和`print`类似的函数。通常，当你尝试去将一个 table 转换成一个 string 的时候，你一般会得到如下的格式的"0x<hex-code-here>", 但是你可以用`__tostring`来修改它，比如：

```lua
t = setmetatable({1,2,3}, {
    __tostring = function(t)
        sum = 0
        for _, v in pairs(t) do sum = sum + v end
        return "Sum: " .. sum
    end
})

print(t)  -- prints out "Sum:6"
```

### 构建一个 Vector 类

为了将所有的东西包裹起来，我们将写一个类，来封装一个 2D 的 vector（感谢 [hump.vector](https://github.com/vrld/hump/blob/master/vector.lua) 提供了很多的代码）。将他们都放在这儿有点多了，但是你可以在 [gist#1055480](https://gist.github.com/1055480) 处看到所有的代码。我已经优先将所有和 metatable 有关的东西放到了第一个文件里面，这也是最重要的。（注意：如果你是第一次遇到 OOP，也许会让你有一点点困惑）

> 译注：下面的内容需要先看一下代码，或者是对照的代码阅读

```lua
Vector = {}
Vector.__index = Vector
```
这段代码设置一个用于`Vector`类的 table。并且将`__index`关键字指向了它本身。接下来要干嘛呢？你也许注意到了我们将所有的 metatable 的方法都放到了`Vector`类内部了。你现在所看到的就是一个 Lua 中最简单的 OOP。`Vector`表对应着一个类，这个类包含的所有方法，所有的实例都可以用。`Vector.new`创建了这个类的一个新的实例。

```lua
function Vector.new(x, y)
    return setmetatable({x = x or 0, y = y or 0}, Vector)
end
```
这个函数创建了一个新的表，包含了`x`和`y`。 并且设置 metatable 为`Vector`类。我们知道，`Vector`包含所有的元方法以及`__index`关键字。这就表明我们可以在新的 table 里面使用到在`Vector`里面定义到的所有函数。
我们待会儿再谈这个。

另一个比较重要的是最后一行：

```lua
setmetatable(Vector, {__call = function(_, ...) return Vector.new(...) end})
```
这样操作的目的就是让我们可以通过`Vector.new`或者`Vector`来创建`Vector`实例。

最后一个比较重要的东西就是：你有可能不太了解冒号的语法。 我们定义用冒号定义一个函数如下：

```lua
function t:method(a,b,c)
    ...
end
```
我们实际是定义了这样的一个函数：

```lua
function t.method(self, a, b,c)
    ...
end
```

这是一个语法糖，我们可以在使用了冒号语法的时候这样调用函数：

```lua
--these are the same
t:method(1,2,3)
t.method(t,1,2,3)
```

现在我们该怎样使用`Vector`类呢？，见下：

```lua
a = Vector.new(10,10)
b = Vector(20,11)
c = a + b
print(a:len())  --14.142135623731
print(a)  --(10,10)
print(c)  --(30,21)
print(a < c)  -- true
print(a == b) --false
```





