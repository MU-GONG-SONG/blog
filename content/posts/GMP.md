---
title: "Go sheduler 是什么?"
date: 2025-09-07
categories: ["GMP"]
tags: ["GMP"]
draft: false
---

### Go sheduler是什么?

>Go 程序的执行有两个层面：Go Program 和 Runtime，即用户程序和运行时。它们之间通过函数调用来实现内存管理、channel 通信、goroutine 创建等功能。用户程序进行的系统调用都会被
Runtime 拦截，以此来帮助它进行调度以及垃圾回收相关的工作。

Go scheduler 可以说是 Go 运行时的一个最重要的 部分了。 Runtime 维护所有的 goroutine ，并通过
scheduler 来进行调度。goroutine 和 threads 是独立的， 但是 goroutine 要依赖 threads 才能执行。
Go 程序执行的高效和 scheduler 的调度是分不开的。 
实际上在操作系统看来，所有的程序都是在执行多线程。将 goroutine 调度到线程上执行，仅仅是 runtime
层面的一个概念，在操作系统之上的层面，操作系统并不能感知到 goroutine 的存在。

### G、M、P三个基础的结构体来实现 goroutine 的调度：
~~~
G 代表一个 goroutine，它包含：表示 goroutine 栈的一些字段，指示当前 goroutine 的状态，指示当前运行到的指令地址，也就是 PC 值。
M 表示内核线程，包含正在运行的 goroutine 等字段。
P 代表一个虚拟的CPU Processor，它维护一个处于 Runnable 状态的 goroutine 队列，M 需要获得 P 才能运行 G。
~~~
当然还有一个核心的结构体：sched，它总揽全局，维持整个调度器的运行。
Runtime 起始时会启动一些 G：垃圾回收的 G，执行调度的 G，运行用户代码的 G；并且会创建一
个 M 用来开始 G 的运行。随着时间的推移，G和M的创建数量逐渐增多。
在 Go 的早期版本，并没有 P 这个结构体，M 必须从一个全局的队列里获取要运行的 G，因此需要获取一个全局的锁，当并发量大的时候，锁就成了瓶颈。后来调度器在 Dmitry Vyukov (Go 语言运行时 runtime 和调度器的核心贡献者之一) ，加
上了 P 结构体。每个 P 维护一个处于 Runnable 状态的 G 的队列，解决了原来的全局锁问题。
~~~
Go scheduler 的目标：将 goroutine 调度到内核线程上。
Go scheduler 的核心思想是：
1）重用线程。
2）限制同时运行（不包含阻塞）的线程数为 N，N 等于 CPU 的核心数目。
3）线程私有 runqueues，并且可以从其他线程偷取 goroutine 来运行，线程阻塞后，可以将
runqueues 传递给其他线程。
~~~
### 为什么需要 P 这个组件，直接把 runqueues 放到 M 不行吗？
~~~
需要 P 组件的原因是当一个线程阻塞的时候，将和它绑定的 P 上的 goroutine 转移到其他线程。
例如当线程进行阻塞系统调用的时候，这时它无法再执行其他代码，因此可以将与其相关联的 P 上的
goroutine 分配给其他线程运行。
另外，Go scheduler 会启动一个后台线程 sysmon，用来检测长时间（超过 10 ms）运行的 goroutine，将其“停
靠”到 global runqueues。这是一个全局的 runqueue，优先级比较低，以示惩罚。
~~~
通常讲到 Go scheduler 都会提到 GPM 模型，来一
个个地看。