---
title: volatile
date: 2021-08-10 00:00:00
author: lh
summary: ""
categories: 并发编程
tags: 
    - 并发编程
---

# 为什么用volatile
    假设 线程1 修改了data的变量为1，然后将这个修改写入到了自己的本地工作内存中。
    那么此时，线程1的工作内存中data的值为1，而主内存和线程2中的data的值任然是1!

   ![](https://kubpang.github.io/sourceFile/Java/并发/volatile-1.jpg)

    这可尴尬了，那接下来，在线程 1 的代码运行过程中，他可以直接读到 data 最新的值是 1，但是线程 2 的代码运行过程中读到的 data 的值还是 0！
    这就导致，线程 1 和线程 2 其实都是在操作一个变量 data，但是线程 1 修改了 data 变量的值之后，线程 2 是看不到的，
    一直都是看到自己本地工作内存中的一个旧的副本的值！

    这就是所谓的 java 并发编程中的可见性问题:
    多个线程并发读写一个共享变量的时候，有可能某个线程修改了变量的值，但是其他线程看不到！也就是对其他线程不可见！

# volatile的作用及背后原理
    要解决上面的问题，引入volatile既可以解决并发编程中的可见性问题。
比如下面的这样的代码，在加了 volatile 之后，会有啥作用呢？
![](https://kubpang.github.io/sourceFile/Java/并发/volatile-2.jpg)

## volatile的作用
    1. volatile修饰的共享变量data，线程1修改data的值，就会在修改本地工作内存的data值之后，强制将data变量最新的值刷回主内存，
    让主内存里的data的值立马变成最新的值

![](https://kubpang.github.io/sourceFile/Java/并发/volatile-3.jpg)

    2. 如果此时如果其他的线程中也存有这个data变量的本地缓存，那么会强制让其他线程的工作内存中的 data 变量缓存直接失效过期，不允许再次读取和使用了！
![](https://kubpang.github.io/sourceFile/Java/并发/volatile-4.jpg)

    3. 如果其他线程想再次获取data时，尝试获取本地工作内存的data变量值，发现失效了，此时，就必须从主内存中获取data变量最新的值。

![](https://kubpang.github.io/sourceFile/Java/并发/volatile-5.jpg)

## volatile的特殊规则
    read、load、use动作必须连续出现。
    assign、store、write动作必须连续出现。


## 内存屏障
JVM 中内存屏障是一组处理器指令，用来实现对内存操作的顺序限制（避免了重排序）。它可以分为下面几种：
* LoadLoad（Load1; LoadLoad；Load2）：Load2 及后续读操作之前，保证 Load1 先读取完。

* StoreStore（Store1; StoreStore; Store2）：Store2 及后续写入操作之前，保证 Store1 的写入对其他处理器可见。

* LoadStore（Load1; LoadLoad；Store2）：Store2 及后续写操作之前，保证 Load1 先读取完。

* StoreLoad（Store1; StoreStore; Load2）：Load2 及后续读操作之前，保证 Store1 的写入对其他处理器可见。
    * StoreLoad 是一个“全能型”的屏障，它同时具有其他 3 个屏障的效果。


需要注意的是：volatile 写是在前面和后面分别插入内存屏障，而 volatile 读操作是在后面插入两个内存屏障。

![](https://kubpang.github.io/sourceFile/Java/并发/volatile-6.png)


# 总结
    每次读取前必须先从主内存刷新到最新的值。
    每次写入后必须立即同步回主内存当中。

    最后给大家提一嘴，volatile 主要作用是保证可见性以及有序性。

    有序性涉及到较为复杂的指令重排、内存屏障等概念，本文没提及，但是 volatile 是不能保证原子性的！

    也就是说，volatile 主要解决的是一个线程修改变量值之后，其他线程立马可以读到最新的值，是解决这个问题的，也就是可见性！

    但是如果是多个线程同时修改一个变量的值，那还是可能出现多线程并发的安全问题，导致数据值修改错乱，
    volatile 是不负责解决这个问题的，也就是不负责解决原子性问题！

    原子性问题，得依赖 synchronized、ReentrantLock 等加锁机制来解决。


