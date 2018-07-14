---
title: Java工程师知识必备
date: 2018-07-12 21:16:30
tags: java
---

# 一、基础 

### 1.Throwable，Error，Exception，RuntimeException

* `Error`和`Exception`都是`Throwable`接口的子类
* Java分为“受检查异常”（或`Checked Exception`，这类异常编译器会检查，如果对这类异常既没有`try...catch`也没`throws`时编译不通过）和“未检查异常”（或`Unchecked Exception`，这类异常编译器不检查），`RuntimeException`及其子类都是“未检查异常”，如`NullPointException`，其他为“受检查异常”，如`IOException`。
* `Error`一般是指与虚拟机相关的错误，程序一般无法自身恢复，比如各种内存溢出（如`OutOfMemoryError`，`StackOverflowError`，注意：断言失败是抛出`AssertionError`）；`Exception`则表示程序级别的异常，程序可以处理并恢复。



参考：[谈一谈Java中的Error和Exception](https://blog.csdn.net/goodlixueyong/article/details/47122487)

#### 2.对象克隆 

`Object.clone()`方法是`native`方法，如果类没有实现`Cloneable`接口时将抛出`CloneNotSupportedException`，即便是子类重写了`clone()`，`clone()`方法签名如下：

```java
protected native Object clone() throws CloneNotSupportedException;
```

`Object.clone()`方法由C++来实现，实现原理是先调用`CollectedHeap::array_allocate()`或者`CollectedHeap::obj_allocate()`分配与原对象相同大小的新内存，然后调用`Copy::conjoint_jlongs_atomic()`将原对象的内存数据复制到新内存当中（所以，该复制方法是`浅复制`）。



调用`clone()`来克隆对象时需要调用到`Object.clone()`，所以子类重写`clone()`时，需要调用`super.clone()`。



对象克隆分为`浅克隆`（或`浅复制`）和`深克隆`（或`深复制`），他们的区别就是会不会为引用类型成员变量调用`clone()`来生成新对象。



由于`Object.clone()`方法是复制内存数据来克隆对象的，因此，同样可以使用Java序列化机制来复制内存数据达到克隆的效果，而且是`深克隆`。



参考：[详解Java中的clone方法 -- 原型模式](https://blog.csdn.net/zhangjg_blog/article/details/18369201)，[Java对象克隆（Clone）及Cloneable接口、Serializable接口的深入探讨](https://blog.csdn.net/kenthong/article/details/5758884)

#### 3.强引用与弱引用 

在`java.lang.ref`包中存在三种类型的引用类，分别是`SoftReference`（软引用），`WeakReference`（弱引用），`PhantomReference`（虚引用），Java的引用类型除了这三个，还有一个就是强引用，它们的级别由高到低分别为强引用、软引用、弱引用、虚引用，引用类型可以控制JVM回收对象的时机。



如果一个对象存在强引用，就不会被回收；如果一个对象只存在软引用，当JVM内存不足时，就会被回收；如果一个对象只存在弱引用，当执行垃圾收集时就会被回收；如果一个对象只存在虚引用，任何时候都可能被回收。



引用类型 | 回收时间 | 用途 | 生存时间
---|---|---|---
强引用 | 从来不会 | 对象的一般状态 | JVM停止运行时终止
软引用 | 内存不足时 | 对象缓存 | 内存不足时终止
弱引用 | 垃圾收集时 | 对象缓存 | 垃圾收集时终止
虚引用 | Unknown | Unknown | Unknown



参考：[Java四种引用---强、软、弱、虚的知识点总结](https://blog.csdn.net/l540675759/article/details/73733763)

#### 4.断言 

断言用于程序调试，Java使用`assert`关键字来支持断言操作，且使用`java`命令运行时必须加上`-ea`参数才会执行断言语句。断言有两种表达方式，如下：



```java
assert condition; // 第一种方式
assert condition : expression // 第二种方式
```



当`condition`为`false`时会抛出`AssertionError`错误，`expression`作为`AssertionError`构造函数的参数用来构造`AssertionError`对象。



#### 5.静态初始化和实例初始化 

* 静态初始化

  静态初始化是指`<cinit>`方法的调用，该方法是编译自动生成，代码是静态语句块和静态变量（或者类变量）。JVM会保证在调用某类的`<cinit>`方法时先调用其父类的`<cinit>`方法。`<cinit>`方法只会被JVM调用一次。

* 实例初始化

  实例创建的方式有：`new`、`Class.newInstance()`、`Constructor.newInstance()`、`Object.clone()`、`ObjectInputStream.readObject()`等，从Java虚拟机层面来说是两种：`new`、`invokevirtual`。

  实例初始化是指`<init>`方法的调用，该方法由构造函数、实例代码块和实力变量组成，`<init>`方法的第一行总是会调用父类的`<init>`方法。



参考：[深入理解Java对象的创建过程：类的初始化与实例化](https://blog.csdn.net/justloveyou_/article/details/72466416)，[JVM类生命周期概述：加载时机与加载过程](https://blog.csdn.net/justloveyou_/article/details/72466105)

#### 6.Java8新特性

#### 7.Java序列化与反序列化

#### 8.反射

#### 9.Statement和PreparedStatement的区别，如何防止SQL注入

#### 10.Java命令

* java
* javac
* jar
* jdb
* javah
* javap
* jps
* jinfo
* jstack
* jmap
* jhat
* jconsole
* jvisualvm
* javadoc
* keytool
* rmiregistry

# 二、集合类 

#### 1.HashMap，TreeMap，Hashtable的实现方式 

#### 2.ArrayList和LinkedList实现方式以及SubList实现方式 

#### 3.HashSet实现方式 

#### 4.Set,Queue,List,Map,Stack,Vector 

#### 5.ConcurrentHashMap的实现方式 

#### 6.Collection.synchronized实现方式 

#### 7.hashCode()和equals()方法的作用 

#### 8.Arrays.sort()和Collections.sort()的实现方式

# 三、线程 

#### 1.线程生命周期 

#### 2.wait()、sleep()、notify()区别 

#### 3.ThreadPoolExecutor实现方式 

#### 4.线程池原理及如何使用线程池 

#### 5.ThreadLocal实现方式 

#### 6.多种方式实现生产者和消费者 

#### 7.中断机制 

# 四、并发 

#### 1.happens-before原则 

#### 2.volatile作用 

#### 3.AQS原理 

#### 4.synchronized和lock的区别 

#### 5.CAS 

#### 6.ReentrantLock,Condition,Semaphore,ReadWriteLock,CoundDownLatch,CyclicBarrier,LockSupport的原理 

#### 7.AtomicInteger和AtomicLong

#### 8.锁的升级和降级

# 五、Servlet

#### 1.HttpServlet生命周期

#### 2.Servlet工作原理

#### 3.Session和Cookie的区别

#### 4.Tomcat架构

#### 5.Servlet3.0的异步请求

# 六、jvm 

#### 1.类加载机制和双亲委派模型 

#### 2.JVM内存模型和运行时数据区 

#### 3.引用计数器和GC root 

#### 4.垃圾收集方法 

#### 5.内存泄漏 

#### 6.调优方法 

# 七、mysql 

#### 1.事务隔离级别 

#### 2.三范式 

#### 3.mysql有哪些索引，区别是什么 

#### 4.悲观锁和乐观锁 

#### 5.分布式原理 

#### 6.mysql有哪些锁 

#### 7.mysql实现B+树的原理 

#### 8.数据一致性

#### 9.mysql的innodb、myisam和memory引擎的优缺点

#### 10.什么是nosql，与sql的优缺点

#### 11.TPS压测

#### 12.Page Split

#### 13.SQL优化

# 八、redis 

#### 1.哨兵模式和主从数据同步 

#### 2.集群原理 

#### 3.redis与memcached区别 

#### 4.过期策略 

#### 5.如何保证一致性 

#### 6.缓存热点与击穿 

#### 7.redis内存淘汰机制 

# 九、mybatis 

#### 1.mybaits一级缓存和二级缓存

#### 2.与hibernate优缺点

#### 3.mybatis工作流程

# 十、MessageQueue 

#### 1.重复消费问题 

#### 2.顺序问题

# 十一、设计模式 

#### 1.有哪些原则 

#### 2.常用设计模式

# 十二、网络 

#### 1.TCP和UDP的区别 

#### 2.TCP三次握手 

#### 3.IO多路复用 

#### 4.NIO 

# 十三、Spring 

#### 1.IoC和AOP 

#### 2.Spring Boot的条件’加载’ 

#### 3.Spring容器启动过程 

#### 4.Spring bean生命周期 

#### 5.Spring MVC生命周期 

#### 6.Spring声明性事务实现方式

# 十四、算法 

#### 1.哪些排序算法 

# 十五、架构

#### 1.分布式锁的实现方式

#### 2.流量控制

#### 3.全文搜索引擎