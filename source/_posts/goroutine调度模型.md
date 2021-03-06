---
title: goroutine调度模型
date: 2019-03-08 14:39:39
tags: [go,goroutine]
---

## Goroutine调度模型

现在讲Goroutine调度，定个前提是在go 1.10及以前的版本。Goroutine协程调度模型经历过G-M模型和G-P-M模型。当前和很长一段时间内采用的都是G-P-M模型。G， P， M更为详细的介绍可以看雨枫老师的《go语言学习笔记》。



### P

P是CPU core的抽象， 我们常说某个CPU是4核的，意思是说这个CPU有4个工作单元，可以并行的执行4个进程。此外，现代CPU的发展，衍生出了超线程的概念，这意味这一个core可以同时执行一条以上的指令。比如，假如这个4核CPU每一个核都能同时执行两条指令，实际上就是4核8线程，大概就可以认为这个cpu是八核的。在Go中，一个P(Logical Processor)代表了一个虚拟的核或者是硬件线程。P的数量就与这些硬件线程的数量相当。使用下面的程序可以得到P的数量:

```go
package main

import (
	"fmt"
	"runtime"
)

func main() {
	// NumCPU returns the number of logical
	// CPUs usable by the current process.
    fmt.Println(runtime.NumCPU())
}
```



### M

M需要绑定到一个P上才能进行工作。这个M可以理解为我们平常所说的线程，由操作系统统一管理。



## G

go程序启动时，会启动一个初始化的Goroutine("G"), 即包含main函数的Goroutine。每个Goroutine本质上都是协程。Goroutine与操作系统线程有很多类似之处，区别在于Goroutine的上下文切换在用户态中进行，操作系统线程的切换则发生在内核态。线程切换的本质就是抢占CPU时间片，而Goroutine切换的本质是"抢占"绑定了P的M。当然，当前Goroutine的切换并非是抢占式的， 这个稍后会说。



### GRQ和LRQ

在G-P-M模型中还有一个队列模块。队列分为全局队列(GRQ)和本地队列(LRQ)两种。每一个P会拥有一个LRQ, 用来管理将在这个P上执行的G。Goroutine 将会轮流在绑定了这个P的M上进行上下文切换。GRQ保留了那些还未被绑定到P上的Goroutine。



### 协作式调度

操作系统的线程调度是抢占式的，因此我们无法预测下一次执行的线程是哪一个。go的协程的调度是在go的运行时(runtime)中实现的。go的协程调度发生在用户态，由go的运行时来调度所有的协程， 但这种调度并不是抢占式的，这点非常重要。



### 协程状态

同系统线程一样，协程同样具有状态： 等待状态(WAITING), 就绪状态(RUNABLE), 运行状态(Executing)

WAITING: WAITING状态下，goroutine等待某个操作完成，比如系统调用或者是其他的同步调用(原子操作，锁等)。这会导致程序出现短暂延迟 (latency)，性能问题的根本原因就在于此。

RUNABLE: 这种状态下，协程等待被调度。如果有大量的goroutine处于这种状态，同样会导致latency，引发性能问题。

EXECUTING: goroutine处于运行状态。



### 上下文切换

goroutine的调度需要定义用户空间的事件， 这些事件将在安全埋点上触发，安全埋点就是允许发生上下文切换的地方。这些安全点预埋在goroutine内部的函数调用上，这些函数调用就是goroutine安全调度的关键。在Go1.11及更早的版本中，如果运行一个没有函数调用的循环(或者循环内部只包含一些简单指令)， 就会导致调度和gc出现延迟。不过在Go1.12这些将会改变：

> *Note: There is a proposal for 1.12 that was accepted to apply non-cooperative preemption techniques inside the Go scheduler to allow for the preemption of tight loops.*

在go中有4中事件将会导致协程调度，当然，并不是说一旦这些事件发生，就一定会触发协程调度，只是具备了协程调度的机会.

- 在 go 关键字处
- GC
- 系统调用
- 同步阻塞调用(atomic, mutex, channel 操作，或者是IO)



### 异步系统调用

如果在一个goroutine内发生异步调用，那么就会将这个goroutine放到事件循环中(net poller), P将会从LRQ中取出下一个goroutine，继续在M上执行。而事件循环中的goroutine就绪以后，会再一次放回到P的LRQ，等待下一次被调度。

### 同步系统调用

在很多情况下，比如文件IO操作都是同步的。当发生同步调用时，P就会与当前的M解绑(此时Goroutine还在旧的M上)，重新和一个空闲的M进行绑定，然后从LRQ中取出下一个goroutine执行。在同步调用结束以后，之前的goroutine被放回到LRQ中等待下一次被调用。



### 在P上做“负载均衡”



**相关资源**

[Scalable Go Scheduler Design Doc](https://docs.google.com/document/d/1TTj4T2JO42uD5ID9e89oa0sLKhJYD0Y_kqxDv3I3XMw/edit#heading=h.mmq8lm48qfcw)

[《go语言学习笔记》-- 雨痕老师](https://github.com/qyuhen/book)

[也谈goroutine调度器](https://tonybai.com/2017/06/23/an-intro-about-goroutine-scheduler/)

[Scheduling In Go - Part I](https://www.ardanlabs.com/blog/2018/08/scheduling-in-go-part1.html)

[Scheduling In Go - Part II](https://www.ardanlabs.com/blog/2018/08/scheduling-in-go-part2.html)

[Goroutine并发调度模型深度解析之手撸一个协程池](https://segmentfault.com/a/1190000015464889)




