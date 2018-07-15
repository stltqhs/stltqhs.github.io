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

* Lambda表达式和函数式接口

  Lambda表达式（也称为闭包）可以将函数作为参数传递，语法参考[Lambda Expression](https://docs.oracle.com/javase/tutorial/java/javaOO/lambdaexpressions.html#syntax)；函数式接口指只有一个抽象方法的接口，可以使用`@FunctionalInterface`标记函数式接口，让编译器检查接口是否为有效的函数式接口。

  常用的函数接口有：[java.lang.Runnable](https://docs.oracle.com/javase/8/docs/api/java/lang/Runnable.html)，[java.util.concurrent.Callable](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/Callable.html)，[java.util.function.Function](https://docs.oracle.com/javase/8/docs/api/java/util/function/Function.html)，[java.util.function.Predicate](https://docs.oracle.com/javase/8/docs/api/java/util/function/Predicate.html)，[java.util.function.Supplier](https://docs.oracle.com/javase/8/docs/api/java/util/function/Supplier.html)，[java.util.function.Consumer](https://docs.oracle.com/javase/8/docs/api/java/util/function/Consumer.html)

* 接口默认方法和静态方法

  接口的默认方法使用`default`修饰并且包括方法体，它不需要接口实现类实现，但可以重写；静态方法的语法同一般类的静态方法。

* 方法引用

  方法引用必须与Lambda表达式结合使用，不能直接将方法引用赋值给一个变量。方法应用的类型有：

类型|示例
--- | ---
引用静态方法 | ContainingClasss::staticMethodName
引用实例方法 | containingObject::instanceMethodName
引用任意类型的实例方法 | ContainingType::methodName
引用构造器方法 | ClassName::new


* 重复注解

  可使用`@Repeatable`添加多个相同的注解

* 更好的类型推断

  编译器增强了类型推送功能，比如：

  ```java
  public class Value< T > {
      public static< T > T defaultValue() { 
          return null; 
      }
      
      public T getOrDefault( T value, T defaultValue ) {
          return ( value != null ) ? value : defaultValue;
      }
  }
  
  public class TypeInference {
      public static void main(String[] args) {
          final Value< String > value = new Value<>();
          value.getOrDefault( "22", Value.defaultValue() ); // 这里不需要强制转换类型，编译器可以推断出来
      }
  }
  ```

  

* 拓宽注解应用场景

  Java8添加了`ElementType.TYPE_USER`和`ElementType.TYPE_PARAMETER`用来描述注解的使用场景。

* 参数名称

  当编译时加上`-parameters`参数时，编译器不会擦除方法的参数名称，此时可以使用`Parameter.getName()`获取参数名称。

* Optional

  为了解决大量的`NullPointerException`添加了一个容器类型`Optional`，其常用方法如下：

方法名 | 说明
------|----
isPresent() | 如果`Optional`实例持有一个非空值，该方法返回`true`，否则返回`false`
orElseGet() | 如果`Optional`实例持有一个空值，则返回lambda表达式的值，如：`name.orElseGet(() -> "anonymous")`
map()       | map()可以将现有的`Optional`实例转换为新值，如：`name.map(n -> 'Hi ' + n + "!").orElse("Hi Stranger!")`
orElse()    | `orElse()`与`orElseGet()`类似，区别是`orElse()`参数为一个类型T，`orElseGet()`为一个lambda表达式。

* Streams

  Java8新增了Stream API，可以方便灵活处理容器的`filter`、`map`、`reduce`、`sort`等串行和并行的聚合操作，Stream由流管道组成，一个流管道由源数据（如数组和集合）、中间操作（产生新的stream对象）、终止操作（产生一个结果）详细参考[java.util.stream.Stream](https://docs.oracle.com/javase/8/docs/api/java/util/stream/Stream.html)。

* Date/Time API

  时间可以纳秒级表示。

* Nashorn Javascript引擎

  添加了`jjs`来运行javascript代码文件，也可以使用Java代码运行javascript内容，如：

  ```java
  ScriptEngineManager manager = new ScriptEngineManager();
  ScriptEngine engine = manager.getEngineByName( "JavaScript" );
          
  System.out.println( engine.getClass().getName() );
  System.out.println( "Result:" + engine.eval( "function f() { return 1; }; f() + 1;" ) );
  ```

  

* Base64

  添加了`java.util.Base64`，支持`Base64`功能。

* 并行数组

  可支持并行数组处理，如`Arrays.parallelSetAll()`、`Arrays.parallelSort()`。

* 并发性

  改进了`java.util.concurrent.ConcurrentHashMap`，添加了`java.util.concurrent.locks.StampedLock`替代`java.util.concurrent.locks.ReadWriteLock`。

* JVM新特性

  使用Metaspace替代PermGen Space，

参考：[Java 8的新特性—终极版](https://www.jianshu.com/p/5b800057f2d8)

#### 7.Java序列化与反序列化

Java序列化允许将Java对象保存为一组字节，之后可以读取这组字节组建一个新的对象（该操作为反序列化），对象序列化不会关注类中的静态变量，被`transient`修饰的变量不会被序列化存储。在RMI和RPC中经常使用到Java序列化和反序列化。



只要类实现了`java.io.Serializable`接口时，其对象就可以序列化，需要用到的流对象是`java.io.ObjectInputStream`（反序列化）和`java.io.ObjectOutputStream`（序列化）。



虚拟机是否允许反序列化，不仅取决于类路径和功能代码是否一致，一个非常重要的一点是两个类的序列化 ID 是否一致（就是 `private static final long serialVersionUID`）。



参考：[深入分析Java的序列化与反序列化](http://www.hollischuang.com/archives/1140)，[java序列化和反序列化](https://www.jianshu.com/p/5a85011de960)

#### 8.反射

反射机制是程序可以在运行时获取类型或者实例的字段和方法，然后进行操作。通过反射的方式可以获取对象字段的值，也可以使用`sun.misc.Unsafe`来快速读取对象的字段值。



参考：[深入分析Java的序列化与反序列化](https://www.sczyh30.com/posts/Java/java-reflection-1/#%E4%B8%80%E3%80%81%E5%9B%9E%E9%A1%BE%EF%BC%9A%E4%BB%80%E4%B9%88%E6%98%AF%E5%8F%8D%E5%B0%84%EF%BC%9F)，[sun.misc.Unsafe的后启示录](http://www.infoq.com/cn/articles/A-Post-Apocalyptic-sun.misc.Unsafe-World)

#### 9.Statement和PreparedStatement的区别，如何防止SQL注入

JDBC执行SQL语句可以使用三个类，分别是`Statement`、`PreparedStatement`、`CallableStatement`，描述如下表：

类名 | 作用
----|----
Statement | 通用查询
PreparedStatemnt | 预编译语句查询，可以解决SQL注入问题，由于是预编译，所以可以减少数据库对SQL代码的解析操作，提高效率
CallableStatement | 存储过程查询

参考：[JDBC为什么要使用PreparedStatement而不是Statement](http://www.importnew.com/5006.html)

#### 10.Java命令

* java

  启动JVM运行Java字节码的工具；

* javac

  编译Java代码的工具，或者称为java编译器（javac就是java compiler的缩写）；

* jar

  操作jar文件的工具；

* jdb

  java调试器工具，作用和用法与gnu的gdb类型；

* javah

  开发JNI时使用的工具，它可以根据Java代码声明的`native`方法自动生成C的头文件；

* javap

  java字节码反编译工具；

* jps

  Java Virtual Machine Process Status Tool，用来查看java虚拟机进程状态；

* jjs

  运行javascript代码的工具；

* jdeps

  分析java依赖的工具；

* jinfo

  查看JVM配置信息的工具；

* jstack

  查看JVM线程栈工具；

* jmap

  导出JVM内存的工具；

* jhat

  分析jmap导出的文件的工具；

* jconsole

  查看虚拟机内存、线程等信息的工具；

* jvisualvm

  jconsole的增强工具；

* javadoc

  生成java api文档的工具；

* keytool

  操作RSA或者证书相关的工具；

* rmiregistry

  RMI相关工具。



参考：[JDK Tools and Utilities](https://docs.oracle.com/javase/7/docs/technotes/tools/index.html)

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

参考：[java7和java8的垃圾回收](https://blog.csdn.net/high2011/article/details/53138202)

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