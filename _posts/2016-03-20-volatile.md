---
title: Why volatile should not be used
updated: 2016-03-20 19:00
---

> 翻译自[Why the "volatile" type class should not be used](https://www.kernel.org/doc/Documentation/volatile-considered-harmful.txt)

## 为什么“volatile”类型不应该被使用
c程序员经常会认为volatile变量可以在当前线程执行时能够在外部被修改。因此，当有共享的数据结构的时候，他们有时会尝试在kernel代码中使用volatile。换句话说，他们已经把volatile变量当成一个简单的原子变量来操作了。 事实上并不是这样的，在kernel代码中使用volatile变量也几乎没有正确过。本文就来为你揭开原因。

理解volatile的关键一点在于，考虑到volatile的目的是为了防止优化。在kernel中，我们需要保护一个共享的数据结构，防止它被同时访问。所以用于保护不必要并行操作(unwanted concurrency)的进程需要使用一些高效的方法来避免优化相关的问题。

和volatile一样，kernel原生的用于保护并行访问数据安全的(spinlock，mutexes，memory barriers等等)， 这些是设计来防止不必要的优化的。如果正确的使用他们，就没有必要再去使用volatile了。如果volatile仍然有必要使用，那么你的代码里面某处肯定是有bug的。在正确设计的kernel代码中，volatile只是拖后腿的。

下面是一段典型的kernel代码：

```c
spin_lock(&the_lock);
do_something_on(&shared_data);
do_something_else_with(&shared_data);
soin_unlock(&the_lock);
```

如果所有的代码都遵循这样的锁规则，共享数据的值在lock里面是不会被意外的改变的。其他代码想要修改这个数据的需要等lock释放。`spinlock`实质是内存屏障——这是一个显示的写法。这就意味着数据访问不会被优化。 编译器以为他知道共享变量里面有什么，但是加了`spin_lock`的调用后，由于这个调用是起到内存屏障的作用的，将强制让编译器忘掉他知道的一切。从而对于要访问的数据编译器就不会有任何的优化了。

如果共享变量被声明成volatile，加锁仍然是有必要的。但是编译器依然会被禁止在临界区(critical section)内对访问的数据进行优化，即在临界区的时候，没有人可以使用它。 当锁存在时，共享数据就不再是volatile的。 在处理共享数据的时候，正确的加锁会让volatile变得没有并要，有时还有存在潜在的坏处。

> 有关临界区[critical section](https://en.wikipedia.org/wiki/Critical_section) 

volatile存储集原来是给内存映射IO寄存器的。在内核中，寄存器的访问也需要使用锁保护，但是不想在临界区内被编译器去优化寄存器的访问。 在kernel内部，I/O内存访问永远是通过accessor函数来完成的。 通过指针来访问I/O内存已经是不太合适了而且并不能在所有的架构上能够正常工作。 这些accessors的实现都是用来防止不必要的优化的。所以，volatile在这个情况下没有必要使用了。

当处理器对一个变量的值上是busy-waiting的时候，我们可能会尝试用去使用volatile。正确的去使用busy wait应该是下面这样的:

```c
while(my_variable != what_i_want)
  	cpu_relax()
```

`cpu_relax()` 调用能够降低cpu的功耗或者切换到超线程的兄弟处理器。同样这个会处理compiler barrier。 所以，再一次volatile又是没有必要的。当然，busy-waiting是一个常用的anti-social的行为。

仍然有一些比较少的场景下面会使用到volatile：

+ 上面提到的accessor 函数在架构上也许会在直接I/O内存访问的时候用volatile。此外，每一个accessor的调用都会成为他自己的一个临界区，并且保证了访问是按照程序员所期待的方式进行的。
+ 内联汇编会会改变内存，但是它没有其他显示的副作用，会有被GCC删除的风险，所以加上对asm表达式加上volatile关键字会防止被清除掉。
+ jiffies变量是一个特殊的变量，每次引用它的时候都会有一个不同的值。但是它可以在不加任何的锁的情况下被读取。所以jiffies可以使用volatile的。但是除此之外的其他类型不建议这么使用。jiffies被认为是一个`stupid legacy` (linus 说的)。维护他弊大于利。
+ 指向连续内存中的数据结构的指针，可能会被I/O设备修改，所以使用volatile是合理的。一个环形buffer被用作成一个网络适配器。典型的例子是，适配器会修改这个指针表明描述符已经被执行了。

对于大多数代码而言，以上都不是使用volatile的理由。因此使用volatile很有可能被看作是一个bug或者将会带来一些安全问题。开发者在使用volatile的时候需三思。

打补丁去移除掉volatile变量是很受欢迎的，只要他们有一个理由表明并行的情况已经被完全的思考过。

(完)

## Expande Link
1.[何登成-Volatile关键字深度剖析](http://hedengcheng.com/?p=725)



