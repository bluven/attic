---
title: "Go内存模型学习笔记"
date: 2018-01-20T21:19:57+08:00
---

Go内存模型保证了一个Goroutine对某个变量的读取可以观测到另一个Goroutine对同一个变量的写。

>注：内存模型的存在允许编译器进行比较重要的优化，即便是简单的优化也可能会影响到共享变量的读写顺序，从而导致条件竞争（race condition）。没有内存模型的情况下，编译器通常是不允许在多线程程序中进行优化。


>A memory model allows a compiler to perform many important optimizations. Even simple compiler optimizations like loop fusion move statements in the program, which can influence the order of read and write operations of potentially shared variables. Changes in the ordering of reads and writes can cause race conditions. Without a memory model, a compiler is not allowed to apply such optimizations to multi-threaded programs in general, or only in special cases.

Happens Before
-----------
编译器和处理器对一个goroutine中的读写顺序进行重新排序，可能会导致一个goroutine观察到的顺序与另一个goroutine观察到的不一样，例如：一个goroutine执行 `a = 1; b= 2`,而另一个goroutine观察的是b先更新，a随其后。

为了解决这个问题，Go定义了名为“Happens Before”的内存操作局部顺序规范。如果事件e1发生在事件e2前面，我们就说e2发生在e1后，如果e1既没有在发生在e2之前或者之后，那么就说e1跟e2是并发地。

在单goroutine中，happens-before的顺序正如程序中描述的那样。

假设变量是v，对v的读取为r，对v的修改为w，对v的以外修改为w`,r只有在如下条件成立时才能观察到w：

1.r没有发生在w前（可能同时发生）；
2.在r之前w之后， 没有w`操作（可能同时发生）；

只有在以后两种情况下都成立的情况下，r才能观察到w：
1. w发生在r前面。
2. w`发生在w前或者r后面


后2点比前2点更强势，它要求没有与w或r并发的w`操作。

对go中任意一个类型的零值初始化操作也算做一个修改。

对大于一个机器字节的读写操作相当于多个顺序不定的机器字节操作。


同步化（Synchronization）
-----------------------


##初始化

程序初始化是在单goroutine中，但goroutine也能生成其他goroutine，这样就成了同步初始化了。

如果package p 引入 package q，q的init函数结束在p的代码运行前。

main.main发生在所有的init函数结束后。


## goroutine创建

go语句发生在任何goroutine执行前

##goroutine销毁

任何一个goroutine的退出都不保证发生在其他事件前，如下语句：

    var a string

    func hello() {
        go func() { a = "hello" }()
        print(a)
    }
a的赋值语句没有跟随其他同步时间，所以其赋值不保证被其他goroutine观察到。为了保证一个goroutine的操作被其他goroutine观察到，必须使用lock或者channel等同步机机制来建立一个相对顺序。

## channel

channel通信是goroutine间同步的主要的方式，对channel的写发生在对应的读完成之前，而对非缓冲channel的读发生在写完成之前，

## Lock

sync包下提供了sync.Mutex和sync.RWMutex来实现传统的锁机制

## Once

Once可以保证一个函数只被执行一次

## 不正确的同步化

Go Memeory Model一文提到了一些不正确的同步化方式，不过经我实验没有重现文中提到的各种可能情况，只能牢记要保证正确的读取，必须依靠go提供的同步机制。
