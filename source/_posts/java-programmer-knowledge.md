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



参考：[深入解析Java反射（1） - 基础](https://www.sczyh30.com/posts/Java/java-reflection-1/#%E4%B8%80%E3%80%81%E5%9B%9E%E9%A1%BE%EF%BC%9A%E4%BB%80%E4%B9%88%E6%98%AF%E5%8F%8D%E5%B0%84%EF%BC%9F)，[sun.misc.Unsafe的后启示录](http://www.infoq.com/cn/articles/A-Post-Apocalyptic-sun.misc.Unsafe-World)

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

# 二、容器类 

#### 1.HashMap，LinkedHashMap，TreeMap，Hashtable的实现方式 

* HashMap

  HashMap由数组和链表组成，元素的类型是Map.Entry，其`table[]`字段就是数组类型，而Map.Entry.next字段组成链表。使用链表的原因是存在hash碰撞，即不同key计算出来的hash值一样。HashMap不是线程安全的，也不保证元素插入的顺序。HashMap可以接受`null`键。

  

  当调用`HashMap.put()`方法时，会先判断`key`是否为`null`，如果是，`value`需要放在`table[0]`的链表中。如果`key`已经存在，替换旧值，否则放在链表头部（代码：`table[indexFor(hash,table.length)] = new Entry<>(hash, key, value, e);size++`）。

  

  当调用`HashMap.get()`方法时，根据`key`的hash值，通过`indexFor(hash,table.length)`计算索引位置（如果key为`null`，索引位置是0），从链表头部开始搜索，当元素e与`key`满足：`e.hash == hash && ((k = e.key) == key || (key != null && key.equals(k))`时返回e.value，否则通过`e.next`获取下一个元素继续比较。

  

  每次调用`addEntry`添加一个新元素时，都会通过`(size >= threshold) && (null != table[bucketIndex])`（其中`threshold = (int) Math.min(capacity * loadFactor, MAXIMUM_CAPACITY + 1);`）来判断是否需要扩容，如果需要，则调用`resize(2 * table.length)`创建一个新的table，大小为原来的两倍，通过`transfer`方法将旧数据移动到新的table上来，此时会重新计算这些元素的索引位置，称为重hash。

  

  当构造一个HashMap时可以指定table数组大小（默认为16），但HashMap要求table数组实际大小必须是2的幂次，即必须是偶数，而不是奇数，原因是计算hash索引的`indexFor()`方法使用的是位运算，而不是模运算，代码是`h & (length-1)`。当length为偶数时，length-1就是奇数，最低位恒为1，与h按位与运算后，值可以是偶数，也可以是奇数。而当length为奇数时length-1就是偶数，最低位恒为0，与h按位与运算后，值是偶数，不可能存在奇数的可能，此时碰撞的概率增加。

  

  最坏情况下，所有的key通过`indexFor()`计算的值都相同，此时HashMap退化成链表，时间复杂度从O(1)提高到O(n)，Java8对此进行优化，将最坏情况下的时间复杂度降低到O(logn)。原理是当链表长度大于TREEIFY_THRESHOLD时（默认为8），会将链表转换为二叉树，此时查找时间复杂度就是O(logn)。

  

  参考：[HashMap 的实现原理](http://wiki.jikexueyuan.com/project/java-collection/hashmap.html)，[Java 8：HashMap的性能提升](http://www.importnew.com/14417.html)

* LinkedHashMap

  LinkedHashMap解决HashMap不保证插入顺序的问题。实现方式是在HashMap的基础上添加一个双向链表（`Entry.after`和`Entry.before`组成双向链表），用来记录插入顺序。LinkedHashMap与HashMap一样可以接受`null`键。

  

  `LinkedHashMap.put`基本与`HashMap.put`操作一致，只是`LinkedHashMap.put`多了一个将新元素添加到双向链表的尾部（`Entry.addBefore(header)`），如果不是新元素，则调用`Entry.recordAccess()`方法记录访问操作（如果`accessOrder`为`false`则什么也不做，默认为`false`）。

  

  `LinkedHashMap.get`基本与`HashMap.get`操作一致，如果能找到元素，则调用`Entry.recordAccess`方法记录访问操作。

  

  `accessOrder`可以用来控制是按照插入顺序还是访问顺序迭代元素，默认是插入顺序，当`accessOrder`为`true`时表示按照访问顺序迭代元素，此时`Entry.recordAcess()`的操作就是将元素添加到双向链表的尾部，表示该元素刚刚被访问过。在`addEntry`方法中添加一个新元素时，会调用`removeEldestEntry(header.after)`来判断是否进行删除长时间未被访问的记录，如果是则调用`removeEntryForKey(header.after.key)`删除长时间未被访问的记录。不过`removeEldestEntry()`默认返回`false`，该方法可以被重写，用来实现简单的`LRU`算法。

  

  `LinkedHashMap.resize()`与`HashMap.resize()`一致，但是`LinkedHashMap.transfer()`重写了`HashMap.transfer`方法，重hash时直接遍历双向链表即可。

  

  参考：[Map 综述（二）：彻头彻尾理解 LinkedHashMap](https://blog.csdn.net/justloveyou_/article/details/71713781)

* TreeMap

  TreeMap按照key的顺序排序存储，使用的数据结构时“红黑树”。由于是按照key的顺序排序，所以如果没有通过构造函数指定`Comparator`时，key就需要实现`Comparable`接口。TreeMap不接受`null`键。

  

  当调用`TreeMap.put()`时，以跟节点开始寻找与key相等的节点，如果找到，设置新值，如果不存在，添加新节点，调整树结构。

  参考：[Java提高篇（二七）-----TreeMap](https://blog.csdn.net/chenssy/article/details/26668941)

* Hashtable

  Hashtable与HashMap基本一样，出来Hashtable不接受`null`键也不接受`null`值，且是线程安全的（因为put和get方法都是同步方法）。

  

  参考：[Map 综述（四）：彻头彻尾理解 HashTable](https://blog.csdn.net/justloveyou_/article/details/72862373)

#### 2.ArrayList和LinkedList实现方式以及SubList实现方式 

* ArrayList

  ArrayList是一个动态数组实现的线性表，其容量可以自动增长，新容量为`(原始容量*3)/2 + 1`。`elementData`是用来存在添加的元素，`size`记录元素个数。ArrayList查找元素使用`indexOf(E)`，原理就是遍历所有元素，效率低下。由于其为数组，可以使用索引快速访问（实现了`RandomAccess`接口）。当删除元素时，需要将后面的元素全部向前移动，效率低下。ArrayList不是线程安全的。

  

  参考：[Java 集合系列03之 ArrayList详细介绍(源码解析)和使用示例](https://www.cnblogs.com/skywang12345/p/3308556.html)

* CopyOnWriteArrayList

  CopyOnWriteArrayList是线程安全的ArrayList，通过“写时复制”来提高“读多写少”的并发操作场景。每次对`array`（定义为`private volatile transient Object[] array;`）修改时，首先获取独占锁（`ReentrantLock`），复制数组，新数组的大小为`len+1`，然后赋值给`array`，最后释放锁。它的迭代器也是通过将`array`赋值给`snopshot`，然后迭代`snopshot`的内容，而且`snopshot`被定义为`final`类型。迭代器不支持`remove`操作，或者说所有修改操作都不支持。

  

  参考：[Java 7之多线程并发容器 - CopyOnWriteArrayList](https://blog.csdn.net/mazhimazh/article/details/19210547)

* LinkedList

  LinkedList是一个双向链表实现的线性表。使用索引访问时需要从header开始查找索引位置，效率低下。插入元素时只需要修改元素`next`和`previous`指针即可，不需要移动元素，效率高。

  

  参考：[LinkedList - Java提高篇](http://wiki.jikexueyuan.com/project/java-enhancement/java-twentytwo.html)

* ArrayList.subList和Arrays.asList(T...a)

  ArrayList.subList的返回值类型是ArrayList内部类SubList，该对象表示ArrayList的一个视图，所以修改SubList会影响到ArrayList。如果ArrayList被修改（`modCount`值改变）时，SubList就无效了，因为SubList记录的`modCount`和ArrayList的`modCount`不一致，任何操作都会报`ConcurrentModificationException`。

  

  Arrays.asList使用适配器模式构建一个Arrays.ArrayList类型的数据，不支持添加、删除调整，原数组的修改将会影响到该Arrays.ArrayList。

  

  参考：[使用ArrayList.subList()和Arrays.asList()方法要注意的地方](https://www.jianshu.com/p/d2a69f7dc563)

* Vector

  与ArrayList基本一致，但是Vector是线程安全的（方法为`synchronized`），而且扩容时新容量的大小与`capacityIncrement`有关，如果`capacityIncrement`大于0，则新容量的大小为`oldCapacity + capacityIncrement`，否则为`oldCapacity * 2`。

  

  参考：[Java 集合系列06之 Vector详细介绍(源码解析)和使用示例](https://www.cnblogs.com/skywang12345/p/3308833.html)

#### 3.HashSet实现方式 

HashSet是基于HashMap实现的，`HashSet.add(E e)`内部是通过`map.put(e, PRESENT)`来实现的，map就是HashMap类型，PRESENT为一个Object类型，用来作为HashMap的值对象。



参考：[HashSet 的实现原理](http://wiki.jikexueyuan.com/project/java-collection/hashset.html)

#### 4.Set,Queue,List,Map,Stack 

* Set

  集合，不存在重复元素。当`e1.equals(e2)`时表示e1和e2重复。最多只有一个`null`元素。

* Queue

  FIFO队列，`offer(e)`添加元素，`poll()`移除元素，`peek()`查看元素。

* Deque

  “double ended queue”即Deque，表示双端队列，可以在线性表两端进行添加和删除元素。

* List

  线性表，可以存在重复的元素。

* Map

  Key-value pair

* Stack

  LIFO队列

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

#### 6.中断机制 

# 四、并发 

#### 1.happens-before原则 

#### 2.volatile作用 

#### 3.AQS原理 

#### 4.synchronized和lock的区别 

#### 5.CAS 

#### 6.ReentrantLock,Condition,Semaphore,ReadWriteLock,CountDownLatch,CyclicBarrier,LockSupport的原理 

#### 7.AtomicInteger和AtomicLong

#### 8.锁的升级和降级

#### 9.多种方式实现生产者和消费者

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

# 十四、算法和数据结构 

[Leetcode](https://leetcode.com/)有各种经典算法的代码，也是算法题库。

#### 1.排序算法 

#### 2.查找算法

参考：[Data Structure Visualizations](https://www.cs.usfca.edu/~galles/visualization/Algorithms.html)

#### 3.LRU

# 十五、架构

#### 1.分布式锁的实现方式

#### 2.流量控制

#### 3.全文搜索引擎