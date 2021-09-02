---
title: CAS
date: 2021-08-11 00:00:00
author: lh
summary: ""
categories: 并发编程
tags: 
    - 并发编程
---

# 什么是CAS 
    CAS compare and swap的缩写，中文翻译成 比较并替换
    
    CAS 操作包含三个操作数 内存位置（V）、预期原值(A) 和新值(B)
    如果内存位置的值与预期原值相匹配，那么处理器会自动将该位置更新为新值。否则，处理器不做任何操作。
    无论哪种情况，它都会在CAS指令之前返回改位置的值。在 CAS 的一些特殊情况下将仅返回 CAS 是否成功，而不提取当前 值。

    CAS 有效地说明了“我认为位置 V 应该包含值 A；如果包含该值，则将 B 放到这个位置；否则，不要更改该位置，只告诉我这个位置现在的值即可。

## CAS的目的
    利用CPU的 CAS指令，同事接祖JNI来完成java的非阻塞算法。
    其他院子操作都是利用类似的特性完成的。
    而这个JUC都是建立在CAS之上的，同时对于synchronized阻塞算法，JUC在性能上有了很大的提升。

## CAS 存在的问题
    CAS 虽然很高效的解决原子操作，但是CAS任然存在三大问题：ABA问题、循环时间长开销大、只能保证一个共享变量的原子操作。
* ABA问题  
    因为CAS在操作值得时候需要判断值有没有发生变化，没有发生变化则更新。  
    但是如果一直原来是A，变成了B，又变成了A，那么使用CAS进行检查时发现值么有发生变化，但是实际值却是变化了。  
    ABA问题的解决思路就是使用版本号。在变量钱追加版本号，每次变量更新的时候版本号加一  
    那么A -B -A 就会变成 1A -2B -3A. 随着jdk版本迭代也推出了atomic原子类进行优化。

* 循环时间长开销大  
    自选CAS如果长时间不成功，会给CPU带来很大的开销。
    如果JVM能支持处理的提供的pause指令，那么效率会有一定的提升。  
    pause 指令的作用
    * 它可以延迟流水线执行指令（de-pipeline）,使CPU不会消耗过多的执行资源，延迟的时间取决于具体实现的版本，在一些处理器上延迟时间是零
    * 它可以避免在退出循环的时候因内存顺序冲突（memory order violation）而引起CPU流水线被清空（CPU pipeline flush），从而提高CPU的执行效率。   

* 只能保证一个共享变量的原子操作  
    当对一个共享变量进行操作时，可以使用循环CAS的方式来保证原子操作，但是对多个共享变量操作时，循环CAS就无法保证操作的原子性。  
    解决办法：可以引入锁机制（比如synchronized），或者将多个共享变量合并成一个共享变量来操作
``` java
public class HelloWorld{
    private int data = 0;

    public synchronized void increment(){
        data++;
    }

    // 多个线程同时调用方法：increment（）;
}
```

## atmoic原子类及其底层原理
    对于简单的data++类的操作，可以换一种做法，JAVA并发包（JUC）下面提供了一系列的Atmoic原子类，比如AtmoicInteger。
    他可以保证多线程并发安全的情况下，高性能的并发更新一个数值。
```java
public class HelloWorld {
    
    private AtomicInteger data = new AtomicInteger(0);

    //多个线程并发的执行：data.incrementAndGet()
}
```
多个线程并发的执行AtmoicInteger的incrementAndGet()方法，意思就是给data的值累加1，接着返回累加后最新的值。

    Atomic 原子类底层用的不是传统意义的锁机制，而是无锁化的 CAS 机制，通过 CAS 机制保证多线程修改一个数值的安全性

![](https://kubpang.gitee.io/sourceFile/Java/并发/atomic-1.jpg)

![](https://kubpang.gitee.io/sourceFile/Java/并发/atomic-2.png)

    上面整个过程就是Atomic原子类的原理，没有基于加锁机制串行化，而是基于CAS机制，。
    先获取一个值，然后发起CAS，比较整个值有没有被改过，如果没有，则更新,CAS 是原子的，不会被打断。

## java8 如何对CAS 进行了优化
    atomic是基于CAS机制来处理数据的，但是CAS也是有缺陷的。
    如果大量的线程同事并发修改一个AtomicInteger，可能有很多的线程会不停的自旋，判断值是否有修改，有修改，然后进入一个空循环中，消耗CPU性能。

    于是JAVA 8 推出了一个新的类 LongAdder.  
    LongAdder是道格·利（Doug Lea的中文名）在java8中发布的类。

    LongAdder也有一个volatile修饰的base值，但是当竞争激烈时，多个线程并不会一直自旋来修改这个值，而是采用了分段的思想。  
    竞争激烈时，各个线程会分散累加到自己所对应的Cell[]数组的某一个数组对象元素中，而不会大家共用一个。

    这样做，可以把不同线程对应到不同的Cell中进行修改，降低了对临界资源的竞争。本质上，是用空间换时间。

    LongAdder是尝试使用分段CAS以及自动分段迁移的方式来大幅提升多线程高并发执行CAS操作的性能，降低了线程间的竞争冲突。

    但是在竞争激烈的情况下，LongAdder 的预期吞吐量要高得多，经过试验，  
    LongAdder 的吞吐量大约是 AtomicLong 的十倍，不过凡事总要付出代价。  
    LongAdder 在保证高效的同时，也需要消耗更多的空间

![](https://kubpang.gitee.io/sourceFile/Java/并发/LongAdder-1.jpg) 

```java
import java.text.NumberFormat;
import java.util.ArrayList;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;
import java.util.concurrent.atomic.LongAdder;

/**
 * <pre>
 * 程序目的：和 AtomicLong 进行性能对比
 * </pre>
 * created at 2020/8/11 06:25
 * @author lerry
 */
public class LongAdderDemo {
 /**
  * 线程池内线程数
  */
 final static int POOL_SIZE = 1000;

 public static void main(String[] args) throws InterruptedException {
    long start = System.currentTimeMillis();

    LongAdder counter = new LongAdder();
    ExecutorService service = Executors.newFixedThreadPool(POOL_SIZE);

    ArrayList<Future> futures = new ArrayList<>(POOL_SIZE);
    for (int i = 0; i < POOL_SIZE * 100; i++) {
    futures.add(service.submit(new LongAdderDemo.Task(counter)));
    }

    // 等待所有线程执行完
    for (Future future : futures) {
    try {
        future.get();
    }
    catch (ExecutionException e) {
        e.printStackTrace();
    }
    }

    NumberFormat numberFormat = NumberFormat.getInstance();
    System.out.printf("统计结果为：[%s]\n", numberFormat.format(counter.sum()));
    System.out.printf("耗时：[%d]毫秒", (System.currentTimeMillis() - start));
    // 关闭线程池
    service.shutdown();
 }

 /**
  * 有一个 LongAdder 成员变量，每次执行N次+1操作
  */
 static class Task implements Runnable {

  private final LongAdder counter;

  public Task(LongAdder counter) {
   this.counter = counter;
  }

  /**
   * 每个线程执行N次+1操作
   */
  @Override
  public void run() {
    for (int i = 0; i < 100; i++) {
        counter.increment();
    }
  }// end run
 }// end class
}
```

[来源参考] https://shishan100.gitee.io/docs/#/./docs/page/page2
[微信参考链接] https://mp.weixin.qq.com/s/NMm7NQt9A1oVwmPgLdzIHg
[LongAdder实践](https://mp.weixin.qq.com/s?__biz=MzU0OTk3ODQ3Ng==&mid=2247483926&idx=1&sn=2a796ef514dea15790e45d79d233833e&chksm=fba6ea15ccd1630387b8738a00a8c1dc6ae0c535305ec4d6e3c76d64eff48bf1d47ae0eaea07&scene=21#wechat_redirect)

