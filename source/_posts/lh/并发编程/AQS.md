---
title: AQS
date: 2021-08-27 00:00:00
author: lh
summary: ""
categories: 并发编程
tags: 
    - 并发编程
---

# 什么是AQS
    AQS Abstract Queued Synchronizer 抽象队列同步器
    
    AQS定义了一套多线程访问共享资源的同步器框架，许多同步类实现都依赖于它一个
    
    AQS对象内部的有一个核心变量state， int类型，代表加锁状态。初始化状态下state为0
    AQS 内部还有一个关键变量，用来记录当前加锁的事哪个线程，init下这个变量为null
    
![](https://kubpang.gitee.io/sourceFile/Java/并发/ReentrantLock与AQS_1.jpg)

## java并发与AQS的关系

java 并发api的使用 简单如 

```java
ReentrantLock lock = new ReentrantLock();

lock.lock(); //加锁

//...业务逻辑

lock.unlock();//释放锁

//以上代码：初始化一个lock 对象，然后加锁 和释放锁
```

    以reentrantLock为例， 与AQS的关系主要是因为：
    java并发包下很多API都是基于AQS来实现 加锁和释放锁的等功能的，
    AQS 是java并发包的基础类。


## ReentrantLock-重入锁
    可重入锁即是 可以对一个ReentrantLock对象多次的执行 lock()与unlock()操作。
    也就是可以对一个锁加锁解锁多次

每次线程可重入加锁一次，会判断一下 当前加锁的线程如果是自己，那么就线程就可重入多次加锁，每次加锁都是将state的 值累加1，其他不变化。

## ReentrantLock的加锁和释放锁的底层原理
    当一个ReentrantLock 尝试对一个对象进行lock 操作时 主要有以下操作

1. 线程1通过调用ReentranLock的lock()来参数进行加锁，这里的加锁过程是直接通过CAS操作将 state 由0 变为1。
    如果之前没有线程尝试过获取锁，那么state肯定为0，此时线程1就可以加锁成功
    
    线程1加锁成功后，就可以设置AQS的加锁线程变量设置为自己
    
![线程1尝试加锁](https://kubpang.gitee.io/sourceFile/Java/并发/ReentrantLock与AQS_2.jpg)  

    从上面的图中可以简单的看出 ReentrantLock 其实就是一个外层API，
    内核中实现的锁机制都是用来的AQS实现的。

2. 线程1加锁了后，线程2跑过来加锁时，会通过CAS判断state是否为0，为1则代表了当前对象有线程加锁了。紧接着会去判断，加锁线程是否为自己，是自己则获取锁成功，而当前是线程1获取，则线程2 获取锁失败

![线程2尝试加锁](https://kubpang.gitee.io/sourceFile/Java/并发/ReentrantLock与AQS_3.jpg)  

    接着 线程2 会将自己放入AQS的一个线程的等待队列中等待，
    当线程1是否锁后，可以再次尝试去加锁

![AQS加锁失败等待队列](https://kubpang.gitee.io/sourceFile/Java/并发/ReentrantLock与AQS_4.jpg)  

线程1在执行完业务逻辑后，就会释放锁，释放锁的过程很简单，就是将AQS的state值逐步减1，当state为0时，则彻底释放锁，同时设置加锁线程变量为null
![线程1释放锁](https://kubpang.gitee.io/sourceFile/Java/并发/ReentrantLock与AQS_5.jpg)


3. 接下来就是从等待队列中唤醒线程2尝试重新加锁。
    线程2开始重复步骤2的操作加锁，加锁成功后，将state设置1，并将加锁线程变量设置为自己

![线程2唤醒后尝试加锁](https://kubpang.gitee.io/sourceFile/Java/并发/ReentrantLock与AQS_6.jpg)

# 总结
    本文主要介绍了AQS的作用、ReentrantLock以及AQS与java并发的关系
    AQS 其实就是一个java并发的基础组件，用来实现各种锁、各种同步组件。  
    它包含了：state变量、加锁线程变量、等待队列等并发中的核心组件。

[来源参考]：https://shishan100.gitee.io/docs/#/./docs/page/page3