---
layout: post
title: Coroutine in C
date: 2016-05-18 05:57:00
description: different way to realize a coroutine
---


协程(coroutine)是一种程序组件，比较灵活但是实际使用的不是很多。经典的一个模型就是生产者和消费者的模型。

生产者将相应的"产品"放到队列中，而消费者从这个队列中，获取这个产品。一般使用协程会在合作式多任务，迭代器等地方。

一般我们在使用子例程(可理解成函数)的时候，它只有一个入口，一旦退出这个子例程就表示结束了。而想再次回到这个函数的时候，我们需要重新从子例程的最开始重新执行。同时子例程的生命周期遵循着后进先出。

而协程则有所不同，协程的起始处是第一个入口处，在协程里面，返回点之后便是下一次的入口点。协程的生命期有自己的使用决定。


目前原生支持协程的语言并不是很多，主流的一些语言中，C#，JavaScript，Go等这些是支持的。同时Lua也是支持协程的，Lua的协程比较容易理解，所以下面从Lua的协程的使用来了解一下协程，同时在C语言中实现相应的协程库。

<hr>

上面很书面的说了一下协程的概念，协程拥有自己的栈，局部变量等，同时和其他的协程共享一些全局变量等。这一点和线程比较像，但是和线程有一个很大的区别就是，线程可以同时运行很多个，但是协程则不行，需要其他的协程的配合。正如上面说的生产-消费模型。
协程在某一个时刻会被挂起，然后等待恢复。不过协程相对于线程就显得很轻量了。

Lua中的协程的相关实现函数是放在"coroutine"的table中的。一般提供了如下的几个函数：

+ create()   创建一个协程，参数一般是一个函数。
+ status()   显示当前协程的状态
+ yield()    将当前的协程挂起
+ resume()   唤醒协同程序

一般这样的几个基本的函数。具体的函数的细节就不细究了，这个主要涉及到Lua的相应的语法了, 会在后续有关Lua的文章中解释这些。下面我们就利用这几个函数实现一下生产-消费的模型，以了解协程的基本使用。

{% highlight lua %}

-- function productor
local nProductor

-- create a product,and send it
function productor()
    local i = 0
    while i < 10 do
        i = i + 1
        send(i)
    end
end

function consumer()
    while i < 10 do
        local i = receive()
        print(i)
    end
end

-- here if no error, the resume function will
-- return true, because the yield has a parameter x
-- so the second return value will the parameter x
function receive()
    local status, value = coroutine.resume(nProductor)
    return value
end

-- send the product, and yield the coroutine
-- here the parameter of the yield will be the
-- return value of the resume function
function send(x)
    coroutine.yield(x)
end

--start from consumer,wake up the function
--nProductor
nProductor = coroutine.create(productor)
consumer()

{% endhighlight %}

上面代码中有一些简短的注释，主要是关于Lua里面相关函数的使用的注释。下面是这个程序的一些基本流程：

+ 注册productor成为一个协程
+ 接着调用consumer(),程序执行到receive中的resume时，会从coroutine的第一行开始执行，此处也就是productor的第一行开始。
+ 在productor执行到send时，由于有yield,所以当前的productor被挂起。
+ 此时回到之前的consumer函数中，receive执行结束，将当前获取到的值打印后，进入下一次循环，又遇到了resume函数，此时唤醒被挂起的productor，之前挂起的地方是send函数里面，send函数只有一条语句，已经执行结束，所以继续执行productor的循环，以此类推。
+ 当最后不满足循环条件的时候，退出循环，整个过程结束。

如果你在send很receive中分别加入打印的话，你会看到send后总是receive。所以这之间是一个配合。

如果你熟悉线程的话，不加任何的同步机制，那么结果会是这两个线程争着跑，打印也是很规则的，可以加以类比。

<hr>

通过上面的简单的介绍，协程的整个过程是比较清晰的，就是执行——挂起——唤醒——执行——挂起——唤醒.... 只是协程需要实现的是能够在之前挂起的地方恢复后继续往下执行。这就好比是在函数A的a行处按下暂停，但是函数A的栈空间不能被清除，转而去执行其他函数B，当执行到唤醒的时候，恢复A的现场，继续执行从a行的下一行执行。

C语言函数的调用依赖于栈帧的形式，函数调用是在一个栈结构上，所以这样的结构是没有办法实现平级的调用的，必须是后压入栈中的函数先执行完退出后，才能执行先压入栈中的函数。不过C提供了两个函数`setjmp()  longjmp()` 这两个函数的作用是：setjmp保存了一份程序的计数器和当前的栈顶指针，而longjmp是用来恢复这些值得。 这个很类似于一个全局的goto。这样的话在某种程度上可以用来实现协程的，不过由于setjmp、longjmp的最大用途是在错误恢复，如果在这样的频繁切换中，程序将如何执行有很大的不确定性。

所以我们需要找到其他一些可行的办法来实现协程。不过此时我们能明确一下方向：我们需要将协程的上下文保存到其他地方，而不是在堆栈中，同时协程之间相互调用的时候，需要从其他地方恢复当前被挂起的协程的时的上下文即可。

这样的操作也就只能在汇编层面上能够实现的比较好，所以大部分用来实现c协程的都会用到一个`ucontext`的组件。这个组件提供了四个函数，用于实现上面提到的保存上下文以及上下文的切换。

+ getcontext(ucontext_t *)  获取当前上下文
+ setcontext(const unconst_t *) 设置当前上下文
+ void makecontext(ucontext_t *, (void *)(), int, ...) 
+ int  swapcontext(ucontext_t *, const ucontext_t *)切换两个协程上下文

这边需要对上面的四个函数有一定的解释：
ucontext_t的实现对应于不同的平台，不过至少会包含以下四个

{% highlight c %}
ucontext_t *uc_link     pointer to the context that will be 
                        resumed when this context returns
sigset_t    uc_sigmask  the set of signals that are blocked
                        when this context is active
stack_t     uc_stack    the stack used by this context
mcontext_t  uc_mcontext a machine-specific representation of
                        the saved context

```

对于makecontext而言，这个函数会修改通过getconext获取到的上下文,然后给这个上下文设置一下栈空间，以及后继的uc_link.

当上下文通过setcontext或者swapcontext激活后，执行func，即makecontext的是第二个参数，后面的int表示有多少个参数传入到func中，最后是参数序列。

话不多说，还是代码比较明确：

{% highlight c %}
void func(void *arg)
{
	puts("In child routine");	
}


void context_test()
{
	char stack[1024*128];
	ucontext_t child, main;

	getcontext(&child); //获取当前的上下文
	child.uc_stack.ss_sp = stack;
	child.uc_stack.ss_size = sizeof(stack);
	child.uc_stack.ss_flag = 0;
	child.uc_link = &main; //设置后继上下文

	//修改上下文指向的func
	makecontext(&child, (void (*)(void))func,0);
	
	//切换到child的上下文，并将当前的上下文保存到main中
	swapcontext(&main,&child);
	puts("Back to main routine");

}

int main()
{
	context_test();

	return 0;
}

{% endhighlight %}

执行结果是：

{% highlight shell %}
In child routine
Back to main routine
{% endhighlight %}

因为我们设置了后继上下文，所以程序能够再次回到context_test中，如果将后继设置为NULL，那么程序只会打印出`In child routine`。 

从上面的程序中我们会发现，实际上我们的context_test就是一个类似于Lua中的resume的函数。 如果是第一次执行，则从注册的func的开头开始执行。如果我们对这个函数加以封装，便可以得到一个resume。

<hr>

我们主要是仿照Lua的样式，实现出对应的：`cothread_create()`, `cothread_resume()`,`cothread_yield()` `cothread_status` 等一系列的函数。

首先我们需要定义两个结构体，一个是用于保存当前协程信息的，另一个是用于保存调度信息的。

{% highlight c %}
typedef struct cothread{
	ucontext_t ctx;
	Fun func;
	int state;
	void *arg;
	char stack[STACK_SIZE];
}cothread_t;

typedef struct schedule{
	ucontext_t main;
	int nco;
	int cap;
	int isrunning;
	cothread_t **co;
}schedule_t;

{% endhighlight %}


#### create函数
对于create而言，我们需要注册一个函数，同时创建一个协程的信息结构体`cothread_t`,并将这个信息保存到`schedule_t`中.

#### resume函数
一个协程一般有如下几个状态：`free` `runnable` `running` `suspend`. 从状态的名字上可以看出具体的用途。resume的函数需要处理两个状态下。`runnable`和`suspend`.当时runnable的时候，表明是第一次调用，所以需要使用makecontext进行一些修改和绑定。当为`suspend`的时候，表明此刻需要唤醒挂起的协程，所以只要swapcontext即可。

#### yield函数
这个函数也只是需要对两个上下文进行一个转换，swapcontext即可。

#### status函数
通过结构体`cothread_t`维护了一个status的状态，所以每次只要返回当前上下文的status项即可。

我实现了一个简单的协程，基本可用。[https://github.com/JesseEisen/coroutine](https://github.com/JesseEisen/coroutine)

<hr>

还有一个方式可以实现，duff device 利用了C语言的“奇技淫巧” 实现的一个相当之轻量的协程。具体可以参考[protothreads](http://dunkels.com/adam/pt/).主要是利用了switch-case的技巧，然后将其封装成相应的宏。这个可以参考[一个“蝇量级” C 语言协程库](http://coolshell.cn/articles/10975.html) 

至此，浅显的说明了如何在C中实现一个协程。Go在这方面的处理用到了goroutine，不过和coroutine有所不同。有空研究一下是如何实现的。

(全文完)
