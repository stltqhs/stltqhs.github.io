---
title: 计算机体系结构——为什么需要内存屏障
date: 2020-12-18 16:59:05
tags: 计算机体系结构
---

# 问题

首先说说两个经典问题。

## Double Checked Locking

双重检查锁模式<sup>[ [1] ](https://en.wikipedia.org/wiki/Double-checked_locking)</sup>可以减少获取锁的开销，但是在多线程模式下，容易出问题，比如：

```java
// Broken multithreaded version
// "Double-Checked Locking" idiom
class Foo {
    private Helper helper;
    public Helper getHelper() {
        if (helper == null) {
            synchronized (this) {
                if (helper == null) {
                    helper = new Helper();
                }
            }
        }
        return helper;
    }

    // other functions and members...
}
```

当线程 A 和线程 B 同时调用 `getHelper()` 方法时，在 `helper = new Helper()`由于“重排序”问题会导致另一个线程看到还未完成初始化的 `helper` 对象。解决的版本是 `helper` 使用 `volatile` 来声明，`volatile` 内部使用内存屏障在防止“重排序”带来的数据不一致。

## RingBuffer

Linux 内核的 kfifo<sup>[ [2] ](https://www.cnblogs.com/anker/p/3481373.html) </sup>是并发无锁的先进先出循环队列，它支持高效的一写多读模式。多线程取数据可以不使用锁，节省开销。kfifo 使用内存屏障解决因“重排序”时的数据一致性问题，其思想也应用在 Java 高效的内存队列库 disruptor<sup>[ [3] ](https://tech.meituan.com/2016/11/18/disruptor.html)</sup>。

# 为什么要重排序

重排序分为：

* 编译器指令重排序（本文不会谈这一点）
* CPU 指令重排序
* CPU 读写缓存重排序

重排序的目的都是为了提高执行效率。CPU 为了提高指令流水线的执行效率，存在依赖关系的内存读写操作需要打散而穿插执行无依赖关系的指令。CPU 多核中的一级缓存一致性约束导致读取和更新还存在一个缓存，即 load buffer 和 store buffer，结果是读写乱序。

结合 JVM 的内存模型和指令重排序已经缓存一致性问题，绘制一张图，如下。



##CPU 指令重排序



## 读写缓存重排序



# 缓存一致性



# volatile 实现原理



# synchronized 实现原理



# happens-before 原则