---
title: Java后台开发大全
date: 2018-07-12 21:16:30
tags: java
---

# 目录

### [基础](#一、基础)

- [Throwable，Error，Exception，RuntimeException](#1-Throwable，Error，Exception，RuntimeException)
- [对象克隆](#2-对象克隆)
- [强引用与弱引用](#3-强引用与弱引用)
- [断言](#4-断言)
- [静态初始化和实例初始化](#5-静态初始化和实例初始化)
- [Java8新特性](#6-Java8新特性)
- [Java序列化与反序列化](#7-Java序列化与反序列化)
- [反射](#8-反射)
- [Statement和PreparedStatement的区别，如何防止SQL注入](#9-Statement和PreparedStatement的区别，如何防止SQL注入)
- [Java命令](#10-Java命令)
- [NoClassDefFoundError和ClassNotFoundException](#11-NoClassDefFoundError和ClassNotFoundException)
- [方法动态绑定原理](#12-方法动态绑定原理)
- [异常处理原理](#13-异常处理原理)
- [泛型原理](#14-泛型原理)
- [线程栈](#15-线程栈)

### [容器类](#二、容器类)

- [HashMap，LinkedHashMap，TreeMap，Hashtable的实现方式](#1-HashMap，LinkedHashMap，TreeMap，Hashtable的实现方式)
- [ArrayList和LinkedList实现方式以及SubList实现方式](#2-ArrayList和LinkedList实现方式以及SubList实现方式)
- [HashSet实现方式](#3-HashSet实现方式)
- [Set,Queue,List,Map,Stack](#4-Set-Queue-List-Map-Stack)
- [ConcurrentHashMap的实现方式](#5-ConcurrentHashMap的实现方式)
- [Collections.synchronizedMap实现方式](#6-Collections-synchronizedMap实现方式)
- [hashCode()和equals()方法的作用](#7-hashCode-和equals-方法的作用)
- [Arrays.sort()和Collections.sort()的实现方式](#8-Arrays-sort-和Collections-sort-的实现方式)

### [线程](#三、线程)

- [线程生命周期](#1-线程生命周期)
- [wait()、sleep()、notify()区别](#2-wait-、sleep-、notify-区别)
- [ThreadPoolExecutor原理](#3-ThreadPoolExecutor原理)
- [ThreadLocal实现方式](#4-ThreadLocal实现方式)
- [中断机制](#5-中断机制)
- [活跃性](#6-活跃性)

### [并发](#四、并发)

- [happens-before原则](#1-happens-before原则)
- [volatile作用](#2-volatile作用)
- [CAS](#3-CAS)
- [LockSupport原理](#4-LockSupport原理)
- [AQS原理](#5-AQS原理)
- [ReentrantLock,Semaphore,ReadWriteLock,CountDownLatch,CyclicBarrier的原理](#6-ReentrantLock-Semaphore-ReadWriteLock-CountDownLatch-CyclicBarrier的原理)
- [BlockingQueue原理](#7-BlockingQueue原理)
- [synchronized原理](#8-synchronized原理)
- [锁的升级和降级](#9-锁的升级和降级)
- [多种方式实现生产者和消费者](#10-多种方式实现生产者和消费者)

### [Servlet](#五、Servlet)

* [Servlet生命周期与过滤器](#1-Servlet生命周期与过滤器)
* [Session和Cookie的区别](#2-Session和Cookie的区别)
* [Servlet的异步请求](#3-Servlet的异步请求)
* [Tomcat架构](#4-Tomcat架构)

### [JVM](#六、JVM)

- [类加载机制和双亲委派模型](#1-类加载机制和双亲委派模型)
- [Java内存模型和运行时数据区](#2-Java内存模型和运行时数据区)
- [垃圾收集](#3-垃圾收集)

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

断言也可以作为一种设计模式，在很多开源框架中都存在使用此类方法来提高代码可读性和安全性，比如Spring中的代码：

```java
public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
	Assert.notNull(beanFactory, "BeanFactory must not be null");
	this.beanFactory = beanFactory;
	if (this.resourceFactory == null) {
		this.resourceFactory = beanFactory;
	}
}
```

`Assert.notNull`是一个方法，方法的实现方式为：

```java
public static void notNull(Object object, String message) {
	if (object == null) {
		throw new IllegalArgumentException(message);
	}
}
```



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

  编译器增强了类型推断功能，比如：

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

以反射方法调用`Object.hashCode`方法为例，叙述反射的原理，样例代码如下：

```java
Object o = new Object();
Class clazz = o.getClass();
Method hashCode = clazz.getMethod("hashCode", null);
System.out.println(hashCode.invoke(o, null));
```

要通过反射获取对象的方法或者字段，首先需要获取Class对象。在静态编码情况下，可以确定Class对象，比如样例代码第二行，可以直接写成`Class clazz = Object.class`，当在运行情况下，可以通过调用对象的`getClass`方法获取对象的Class对象。`getClass`方法在`Object`类中声明，方法签名如下：

```java
public final native Class<?> getClass();
```

`getClass`方法是本地方法，由C++来实现，C++方法定义在[Object.c](https://github.com/dmlloyd/openjdk/blob/jdk7u/jdk7u/jdk/src/share/native/java/lang/Object.c)文件中，方法定义如下：

```c++
JNIEXPORT jclass JNICALL
Java_java_lang_Object_getClass(JNIEnv *env, jobject this)
{
    if (this == NULL) {
        JNU_ThrowNullPointerException(env, NULL);
        return 0;
    } else {
        return (*env)->GetObjectClass(env, this);
    }
}
```

`GetObjectClass`方法的核心代码如下：

```c++
klassOop k = JNIHandles::resolve_non_null(obj)->klass();
jclass ret =
  (jclass) JNIHandles::make_local(env, Klass::cast(k)->java_mirror());
```

指向对象的指针称为`Ordinary Object Pointer`(OOP)，Java实例对象使用C++中的[oopDesc](https://github.com/dmlloyd/openjdk/blob/jdk7u/jdk7u/hotspot/src/share/vm/oops/oop.hpp)来表示，`oop`就是指向`oopDesc`类型的指针。在JVM中，Java对象的头部有下列两个字段组成：

```c++
volatile markOop  _mark;
  union _metadata {
    wideKlassOop    _klass;
    narrowOop       _compressed_klass;
  } _metadata;
```

`_metadata`就是指向实例对象的`java.lang.Class`的对象，对应C++中的`klassOop`。`JNIHandles::resolve_non_null(obj)->klass()`就是要获取`_metadata`的`klassOop`对象，它指向方法区中的类实例对象，类实例对象就是`Class`对象。所以`o.getClass()`方法的原理就是`o`对象存在指向类实例对象的引用，通过该引用可以获取`o`对象的`Class`对象。

获得了`Class`对象，就可以通过`getMethod`获得`Method`方法引用。`getMethod`的调用链为`Class.getMethod`->`Class.getMethod0`->`Class.privateGetMethodRecursive`->`Class.privateGetDeclaredMethods`->`Class.searchMethods`。在`privateGetDeclaredMethods`方法中使用了一个重要的字段，`private volatile transient SoftReference<ReflectionData<T>> reflectionData`，`ReflectionData`是`Class`的内部类，定义如下：

```java
private static class ReflectionData<T> {
    volatile Field[] declaredFields;
    volatile Field[] publicFields;
    volatile Method[] declaredMethods;
    volatile Method[] publicMethods;
    volatile Constructor<T>[] declaredConstructors;
    volatile Constructor<T>[] publicConstructors;
    // Intermediate results for getFields and getMethods
    volatile Field[] declaredPublicFields;
    volatile Method[] declaredPublicMethods;
    volatile Class<?>[] interfaces;

    // Value of classRedefinedCount when we created this ReflectionData instance
    final int redefinedCount;

    ReflectionData(int redefinedCount) {
        this.redefinedCount = redefinedCount;
    }
}
```

首先，`reflectionData`字段是软引用，而`reflectionData`指向的`ReflectionData`实例对象在内存使用苛刻时就会被GC回收。因此，在获取类对象的方法或者字段时，都需要判断`reflectionData`执行的对象是否为`null`，如果为`null`，表示被回收，需要重新创建，然后通过`Atomic.casReflectionData(this, oldReflectionData, new SoftReference<>(rd))`（CAS操作）设置`reflectionData`字段。其次，`privateGetDeclaredMethods`会判断`ReflectionData`对象的`declaredPublicMethods`或者`declaredMethods`字段是否为`null`，如果为`null`，表示还未获取类对象的方法，需要调用`getDeclaredMethods0`方法获取类对象的方法，并设置到`ReflectionData`对象的`declaredPublicMethods`或者`declaredMethods`字段。

通过`privateGetDeclaredMethods`方法可以获取到类实例的方法表，接下来通过`searchMethods`搜索需要获取的方法，该方法定义如下：

```java
private static Method searchMethods(Method[] methods,
                                    String name,
                                    Class<?>[] parameterTypes)
{
    Method res = null;
    String internedName = name.intern();
    for (int i = 0; i < methods.length; i++) {
        Method m = methods[i];
        if (m.getName() == internedName
            && arrayContentsEq(parameterTypes, m.getParameterTypes())
            && (res == null
                || res.getReturnType().isAssignableFrom(m.getReturnType())))
            res = m;
    }

    return (res == null ? res : getReflectionFactory().copyMethod(res));
}
```

`searchMethods`方法首先迭代方法表，如果期望的方法被找到，需要将方法复制一个新对象，然后返回给样例代码中的`hashcode`变量。复制的方法`copy`定义在`Method.java`中。由此可见，每次通过调用`getMethod`方法返回的Method对象其实都是一个新的对象，如果调用频繁最好缓存起来。

获取到`Method`对象后，接下来就是调用`invoke`方法，与`invoke`方法相关的字段和方法如下：

```java
private Method              root;
private volatile MethodAccessor methodAccessor;
@CallerSensitive
public Object invoke(Object obj, Object... args)
    throws IllegalAccessException, IllegalArgumentException,
       InvocationTargetException
{
    if (!override) {
        if (!Reflection.quickCheckMemberAccess(clazz, modifiers)) {
            Class<?> caller = Reflection.getCallerClass();
            checkAccess(caller, clazz, obj, modifiers);
        }
    }
    MethodAccessor ma = methodAccessor;             // read volatile
    if (ma == null) {
        ma = acquireMethodAccessor();
    }
    return ma.invoke(obj, args);
}
```

`root`字段指向`ReflectionData`对象中的方法表的某个`Method`实例（因为获得的`Method`对象是从`ReflectionData`复制而来，所以`root`保留了被复制的对象或者说原对象）。`Method.invoke`方法就是调用`methodAccessor`的`invoke`方法，`methodAccessor`这个属性如果`root`本身已经有了，那就直接用root的`methodAccessor`赋值过来，否则的话就创建一个。`MethodAccessor`本身就是一个接口，其主要有三种实现

- DelegatingMethodAccessorImpl

- NativeMethodAccessorImpl

- GeneratedMethodAccessorXXX

其中`DelegatingMethodAccessorImpl`是最终注入给`Method`的`methodAccessor`的，也就是某个`Method`的所有的`invoke`方法都会调用到这个`DelegatingMethodAccessorImpl.invoke`，正如其名一样的，是做代理的，也就是真正的实现可以是`NativeMethodAccessorImpl`和`GeneratedMethodAccessorXXX`两种。如果是`NativeMethodAccessorImpl`，那顾名思义，该实现主要是native实现的，而`GeneratedMethodAccessorXXX`是为每个需要反射调用的`Method`动态生成的类，后的XXX是一个不断递增的数字。并且所有的方法反射都是先走`NativeMethodAccessorImpl`，默认调了15次之后，才生成一个`GeneratedMethodAccessorXXX`类，生成好之后就会走这个生成的类的`invoke`方法。`NativeMethodAccessorImpl`的`invoke`方法定义如下：

```java
public Object invoke(Object obj, Object[] args)
    throws IllegalArgumentException, InvocationTargetException
{
    if (++numInvocations > ReflectionFactory.inflationThreshold()) {
        MethodAccessorImpl acc = (MethodAccessorImpl)
            new MethodAccessorGenerator().
                generateMethod(method.getDeclaringClass(),
                               method.getName(),
                               method.getParameterTypes(),
                               method.getReturnType(),
                               method.getExceptionTypes(),
                               method.getModifiers());
        parent.setDelegate(acc);
    }

    return invoke0(method, obj, args);
}
```

`generateMethod`会生成一个`GeneratedMethodAccessorXXX`实例，它的`invoke`方法就是样例代码中的`o.hashCode()`，直接调用对象的方法，这种生成方法是Java字节码拼接技术，用来在运行时生成Java类。[javassist](http://www.javassist.org/)和[cglib](https://github.com/cglib/cglib)都是基于字节码拼接技术，在`Spring`中就是使用`cglib`来动态的生成代理类，实现`AOP`功能。`GeneratedMethodAccessorXXX`的类加载器是一个`DelegatingClassLoader`类加载器，使用新的类加载器是为了性能考虑，在某些情况下可以卸载这些生成的类。`invoke0`方法是本地方法，由C实现，方法定义在[NativeAccessors.c](https://github.com/dmlloyd/openjdk/blob/jdk7u/jdk7u/jdk/src/share/native/sun/reflect/NativeAccessors.c)如下：

```c
JNIEXPORT jobject JNICALL Java_sun_reflect_NativeMethodAccessorImpl_invoke0
(JNIEnv *env, jclass unused, jobject m, jobject obj, jobjectArray args)
{
    return JVM_InvokeMethod(env, m, obj, args);
}
```

实际方法调用交给JVM执行即可。

当执行命令`java JavaApplication`时，JVM内部也是通过反射获取`JavaApplication`的`main`方法的`Method`对象，然后调用`Method.invoke`方法执行`main`方法的代码。

参考：[深入解析Java反射（1） - 基础](https://www.sczyh30.com/posts/Java/java-reflection-1/#%E4%B8%80%E3%80%81%E5%9B%9E%E9%A1%BE%EF%BC%9A%E4%BB%80%E4%B9%88%E6%98%AF%E5%8F%8D%E5%B0%84%EF%BC%9F)，[sun.misc.Unsafe的后启示录](http://www.infoq.com/cn/articles/A-Post-Apocalyptic-sun.misc.Unsafe-World)，[从一起GC血案谈到反射原理](https://mp.weixin.qq.com/s/5H6UHcP6kvR2X5hTj_SBjA)

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

#### 11.NoClassDefFoundError和ClassNotFoundException

`NoClassDefFoundError`和`ClassNotFoundException`都是由于在`CLASSPATH`下找不到对应的类而引起的，通常是缺少对应的jar包或者jar包冲突导致，具体如下：

* `ClassNotFoundException`是在代码中显示调用加载类的方法导致，如`Class.forName`、`ClassLoader.findSystemClass()`和`ClassLoader.loadClass()`等；

* `NoClassDefFoundError`是JVM链接时找不到类时抛出的错误，如`new`一个实例或者调用静态方法等。

  

  参考：[NoClassDefFoundError和ClassNotFoundException的不同](https://www.jianshu.com/p/93d0db07d2e3)

#### 12.方法动态绑定原理

在Java中， `final`，`static`，`private`和构造方法与类的绑定关系是在编译期确定了的，所以我们称之为“先期绑定”或者“静态绑定”，对于实例“静态绑定”的方法，采用`invokespecial`指令调用。对于其他实例方法，则需要在运行时根据对象类型再行决议，我们称之为“后期绑定”或“动态绑定”，采用`invokevirtual`指令调用方法。

以下列代码为例，叙述方法动态绑定原理：

```java
public class VtableExample {
    static class Father {
        protected int i = 1;
        public Father() {
            print();
        }
        public void print() {
            System.out.println("i=" + i);
        }
    }
    
    static class Son extends Father {
        protected int i = 2;
        public Son() {
            
        }
        public void print() {
            System.out.println("i=" + i);
        }
    }
    
    public static void main(String[] args) {
        Father fa = new Son();
        fa.print();
    }
}
```

上述代码的输出结果为

```text
i=0
i=2
```

Java方法的动态绑定原理源自C++的虚方法调用，使用一张虚方法表`vtable`来控制方法调用，所不同的是C++在编译时就创建了类的虚方法表，而Java在运行时加载类才创建虚方法表。

当第一次加载类时，JVM会调用[classFileParser.cpp::parseClassFile()](https://github.com/dmlloyd/openjdk/blob/jdk7u/jdk7u/hotspot/src/share/vm/classfile/classFileParser.cpp)函数对类的字节码解析，调用`parseMethods()`函数解析类的所有方法，之后再调用`klassVtable::compute_vtable_size_and_num_mirandas()`函数计算当前类的`vtable`大小。计算`vtable`大小的过程为首先获取父类的`vtable`大小，再循环当前类的方法，调用`needs_new_vtable_entry`方法判断方法是否需要加入到`vtable`（如果方法被声明为`public`或者`protected`且不是`static`或者`final`时，称此方法为虚方法，此时该方法返回`true`），如果返回true，则`vtable`大小加1。当类解析完成后，就需要调用[InstanceKlass::allocate_instance_klass()](https://github.com/dmlloyd/openjdk/blob/jdk7u/jdk7u/hotspot/src/share/vm/oops/instanceKlassKlass.cpp)方法分配内存，存储类信息，这些信息就包括`vtable`大小。当类信息创建完成后就可以准备方法调用了。在执行真正的方法调用前，需要调用[instanceKlass::link_class](https://github.com/dmlloyd/openjdk/blob/jdk7u/jdk7u/hotspot/src/share/vm/oops/instanceKlass.cpp)进行方法链接，此时将会初始化虚方法表。初始化虚方法表的方法在`instanceKlass::link_class_impl`中执行，代码如下：

```c++
// Initialize the vtable and interface table after
// methods have been rewritten since rewrite may
// fabricate new methodOops.
// also does loader constraint checking
if (!this_oop()->is_shared()) {
  ResourceMark rm(THREAD);
  this_oop->vtable()->initialize_vtable(true, CHECK_false);
  this_oop->itable()->initialize_itable(true, CHECK_false);
}
```

在[initialize_vtable](https://github.com/dmlloyd/openjdk/blob/jdk7u/jdk7u/hotspot/src/share/vm/oops/klassVtable.cpp)方法中，先复制父类的虚方法表到当前类的虚方法表。然后在`update_inherited_vtable`方法中将子类重写的方法入口地址通过`klassVtable::put_method_at(Method* m, int index)`方法写回到虚方法表中，以替换父类方法地址。如果不是重写父类的虚方法，需要在虚方法表中插入一个新元素。

当执行`invokevirtual`调用虚方法时，由[LinkResolver::resolve_invoke](https://github.com/dmlloyd/openjdk/blob/jdk7u/jdk7u/hotspot/src/share/vm/interpreter/linkResolver.cpp)完成解析任务，该方法定义如下：

```c++
void LinkResolver::resolve_invoke(CallInfo& result, Handle recv, constantPoolHandle pool, int index, Bytecodes::Code byte, TRAPS) {
  switch (byte) {
    case Bytecodes::_invokestatic   : resolve_invokestatic   (result,       pool, index, CHECK); break;
    case Bytecodes::_invokespecial  : resolve_invokespecial  (result,       pool, index, CHECK); break;
    case Bytecodes::_invokevirtual  : resolve_invokevirtual  (result, recv, pool, index, CHECK); break;
    case Bytecodes::_invokehandle   : resolve_invokehandle   (result,       pool, index, CHECK); break;
    case Bytecodes::_invokedynamic  : resolve_invokedynamic  (result,       pool, index, CHECK); break;
    case Bytecodes::_invokeinterface: resolve_invokeinterface(result, recv, pool, index, CHECK); break;
  }
  return;
}
```

在样例代码中，当执行`new Son()`时，先创建`Father`的虚方法表，假设`print`方法在虚方法表位置为`n`，父类初始化完成后，开始初始化子类`Son`，然后创建`Son`的虚方法表。创建`Son`的虚方法表时，先将父类的虚方法表复制到子类的虚方法表中，此时子类虚方法表位置为`n`的方法是`Father.print`。当执行`update_inherited_vtable`方法时会将子类的`print`方法入口写入到虚方法表位置为`n`的地方，此时虚方法表位置为`n`的方法是`Son.print`。所有类信息构造完成后，开始执行`Son`的构造函数，它首先调用`Father`的构造函数，在此函数中，会调用`print`方法，实际上是`invokevirtual print`指令。通过`instanceKlass::uncached_lookup_method`方法在`Father`类中查询`print`方法，可以找到该方法，该方法使用[methodOopDesc*](https://github.com/dmlloyd/openjdk/blob/jdk7u/jdk7u/hotspot/src/share/vm/oops/methodOop.cpp)表示，即`methodOop`指针，指向`Father.print`，它记录了通过`klassVtable::put_method_at(Method* m, int index)`放入虚方法表的位置`n`。然后在`LinkResolver::runtime_resolve_virtual_method`方法中通过位置`n`在`Son`的虚方法表中找到真正要执行的方法，即`Son.print`。最后调用`Son.print`方法。

参考：[Getting Started with HotSpot and OpenJDK](https://www.infoq.com/articles/Introduction-to-HotSpot)，[從虛擬機角度看Java多態->（重寫override）的實現原理](https://hk.saowen.com/a/1e9e6f8665515e390f8338884a78aba61a91d1efb7ffcdf9d11aad3524c5083e)

#### 13.异常处理原理

当使用`javac`编译java源码时，会为方法内的`try/catch/finally`语句块生成一个异常表（`exception_table`），异常表指定了当出现异常时代码需要跳转到何处执行。

以如下代码为例：

```java
public static void main(String[] args) throws Exception {
      try {
          throw new Exception();
      } catch (Exception e) {
          System.out.print("Caught!");
      } finally {
          System.out.print("Finally!");
      }
  }
```

当使用`javac`编译时，会生成如下字节码(`javap`)：

```text
     0: new           #17                 // class java/lang/Exception
     3: dup
     4: invokespecial #19                 // Method java/lang/Exception."<init>":()V
     7: athrow
     8: astore_1
     9: getstatic     #20                 // Field java/lang/System.out:Ljava/io/PrintStream;
    12: ldc           #26                 // String Caught!
    14: invokevirtual #28                 // Method java/io/PrintStream.print:(Ljava/lang/String;)V
    17: getstatic     #20                 // Field java/lang/System.out:Ljava/io/PrintStream;
    20: ldc           #34                 // String Finally!
    22: invokevirtual #28                 // Method java/io/PrintStream.print:(Ljava/lang/String;)V
    25: goto          39
    28: astore_2
    29: getstatic     #20                 // Field java/lang/System.out:Ljava/io/PrintStream;
    32: ldc           #34                 // String Finally!
    34: invokevirtual #28                 // Method java/io/PrintStream.print:(Ljava/lang/String;)V
    37: aload_2
    38: athrow
    39: return
```

异常表内容为：

```text
from  to  target type
0     8     8   Class java/lang/Exception
0    17    28   any
```

异常表的内容由四部分组成：

* from 抛出异常的开始行
* to 抛出异常的结束行
* target 需要跳转的行
* type 异常类型

异常表内容的第一行表示如果在第0行（此处的行指程序地址，比如第3行是指程序地址为3的指令，即`dup`）和第8行抛出`java.lang.Exception`异常时，跳转到第8行执行代码。如果在第0行和第17号抛出代码时，跳转到第28行执行代码。如果当前方法未找到合适的异常处理时，当前方法弹栈，交给栈顶方法处理。如果线程栈方法全部弹出也未找到异常处理，则线程结束。

参考：[The secret life of Java exceptions and JVM internals: Level up your Java knowledge](https://blog.takipi.com/the-surprising-truth-of-java-exceptions-what-is-really-going-on-under-the-hood/)

#### 14. 泛型原理

泛型可以对类和接口的类型参数化，使用比较多的是容器类，比如`List<String>`就是将`List`的元素参数化为`String`。使用泛型的作用有：

* 编译器强类型检查
* 去掉强制类型转换的代码
* 支持泛型算法的实现

类型参数的命名习惯如下：

- E - Element (used extensively by the Java Collections Framework)
- K - Key
- N - Number
- T - Type
- V - Value
- S,U,V etc. - 2nd, 3rd, 4th types



泛型示例代码如下：

```java
public class Generic<T> {
    private T t;
    public static void main(String[] args) {
        Generic<String> g = new Generic<String>();
        g.t = "Hello";
    }
}
```

编译上述代码，然后反编译后得到如下信息：

```text
public class Generic<T extends java.lang.Object> extends java.lang.Object
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #6.#20         // java/lang/Object."<init>":()V
   #2 = Class              #21            // Generic
   #3 = Methodref          #2.#20         // Generic."<init>":()V
   #4 = String             #22            // Hello
   #5 = Fieldref           #2.#23         // Generic.t:Ljava/lang/Object;
   #6 = Class              #24            // java/lang/Object
   ...
{
  public Generic();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 1: 0

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=2, args_size=1
         0: new           #2                  // class Generic
         3: dup
         4: invokespecial #3                  // Method "<init>":()V
         7: astore_1
         8: aload_1
         9: ldc           #4                  // String Hello
        11: putfield      #5                  // Field t:Ljava/lang/Object;
        14: return
}

```

从反编译代码中可以看到，字段`t`的类型是`Object`，而代码`Generic<String> g = new Generic<String>();`的泛型信息被擦除，变为`Generic g = new Generic();`，此时已经看不到泛型信息。对于非局部变量，即类、字段和方法（包括参数和返回值）不会擦除泛型信息，因此可以通过反射来获取泛型信息。上述代码`Generic<T>`变编译后，泛型信息是`Generic<T extends java.lang.Object>`，而变量`g`由于泛型类型被擦除，无法通过反射获取其泛型类型。可以通过`new Generic<String>(){}`构造一个子类，此时可以通过反射获取泛型信息。这种方式使用的比较多的是在json解析中，当要解析一串json文本为带有泛型的类型时使用如fastjson的用法`JSONObject.parseObject(json, new TypeReference<List<Person>>(){})`。

参考：[java泛型（二）、泛型的内部原理：类型擦除以及类型擦除带来的问题](https://blog.csdn.net/lonelyroamer/article/details/7868820)，[Generics](https://docs.oracle.com/javase/tutorial/java/generics/index.html)

#### 15.线程栈

参考：[JVM Internals](http://blog.jamesdbloom.com/)


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

ConcurrentHashMap使用`锁分段`的方式来实现高效的HashMap，使用`不变性`和`volatile`来减少加锁操作，提高线程并发。

ConcurrentHashMap包含n个segment（segment是ReentrantLock的子类，初始时n为16，且n为2的幂次），segment的3个重要成员变量定义如下：

```java
transient volatile int count;
transient int modCount;
transient int threshold;
transient volatile HashEntry<K,V>[] table;
final float loadFactor;
```

每个segment由HashEntry组成，HashEntry的3个重要的成员变量的声明如下：

```java
final K key;
final int hash;
volatile V value;
final HashEntry<K,V> next;
```

ConcurrentHashMap数据结构如下图：
![ConcurrentHashMap结构](http://static.zybuluo.com/Rico123/zaqxg0nmu1qq79lm6wpwss2f/ConcurrentHashMap.jpg "ConcurrentHashMap结构")

由于`HashEntry.next`是`final`类型，链表的`put`操作需要在表头添加一个新元素，该操作不影响读取或者对链表的遍历操作，因此读取可以不用加锁（除了`value`为`null`时需要加锁再读一次），`remove`操作是复制被删除节点的前驱节点构造新链表，同时将被删除节点的`next`值复制到该链表的尾节点的`next`，该操作不影响读取或者对链表的遍历操作。对于`size()`操作，先使用不加锁的方式计算每个segment的count，同时比较计算前和计算后的`modCount`值是否改变，如果改变，表示计算期间存在修改情况，此时再加锁计算。

参考：[探索 ConcurrentHashMap 高并发性的实现机制](https://www.ibm.com/developerworks/cn/java/java-lo-concurrenthashmap/index.html)，[Map 综述（三）：彻头彻尾理解 ConcurrentHashMap](https://blog.csdn.net/justloveyou_/article/details/72783008)

#### 6.Collections.synchronizedMap实现方式 

使用`装饰模式`封装相关方法，且方法为同步方法。相关代码如下：

```java
public static <K,V> Map<K,V> synchronizedMap(Map<K,V> m) {
        return new SynchronizedMap<>(m);
}

// SyncrhonizedMap
private static class SynchronizedMap<K,V>
        implements Map<K,V>, Serializable {
       private static final long serialVersionUID = 1978198479659022715L;

        private final Map<K,V> m;     // Backing Map
        final Object      mutex;        // Object on which to synchronize

        SynchronizedMap(Map<K,V> m) {
            if (m==null)
                throw new NullPointerException();
            this.m = m;
            mutex = this;
        }

        SynchronizedMap(Map<K,V> m, Object mutex) {
            this.m = m;
            this.mutex = mutex;
        }

        public int size() {
            synchronized (mutex) {return m.size();}
        }
        /* ... */
}
```



参考：[Wrapper Implementations](https://docs.oracle.com/javase/tutorial/collections/implementations/wrapper.html)

#### 7.hashCode()和equals()方法的作用 

hashCode和equals方法在Object类中定义，其中hashCode方法为native方法，equals方法定义如下：

```java
public boolean equals(Object obj) {
        return (this == obj);
    }
```

在不重写equals方法时，equals和==操作符是等价的。

hashCode方法只有在hash表才用到，比如HashSet，HashMap等，此时hashCode和equals方法存在关联，因为查找元素时先获得对象的hashCode，定位hash索引，然后调用equals方法查找对象。

重写equals方法需要遵循如下要求：

- **自反性**（reflexive）。对于任意不为`null`的引用值x，`x.equals(x)`一定是`true`。
- **对称性**（symmetric）。对于任意不为`null`的引用值`x`和`y`，当且仅当`x.equals(y)`是`true`时，`y.equals(x)`也是`true`。
- **传递性**（transitive）。对于任意不为`null`的引用值`x`、`y`和`z`，如果`x.equals(y)`是`true`，同时`y.equals(z)`是`true`，那么`x.equals(z)`一定是`true`。
- **一致性**（consistent）。对于任意不为`null`的引用值`x`和`y`，如果用于equals比较的对象信息没有被修改的话，多次调用时`x.equals(y)`要么一致地返回`true`要么一致地返回`false`。
- 对于任意不为`null`的引用值`x`，`x.equals(null)`返回`false`。

重写hashCode方法时需要遵循如下要求：

- 在一个Java应用的执行期间，如果一个对象提供给equals做比较的信息没有被修改的话，该对象多次调用`hashCode()`方法，该方法必须始终如一返回同一个integer。

- 如果两个对象根据`equals(Object)`方法是相等的，那么调用二者各自的`hashCode()`方法必须产生同一个integer结果。

  

  基于上述两个性质，一般是重写equals方法就要重写hashCode方法。

参考：[Java hashCode() 和 equals()的若干问题解答](https://www.cnblogs.com/skywang12345/p/3324958.html)，[Java提高篇——equals()与hashCode()方法详解](https://www.cnblogs.com/Qian123/p/5703507.html)

#### 8.Arrays.sort()和Collections.sort()的实现方式

`Arrays.sort(int[])`排序算法使用`DualPivotQuicksort.sort`方法，称为“双轴快速排序“，该方法会根据数组长度和数值的连续性来使用不同的排序算法。如果数组长度大于等于286且连续性好的话，就用归并排序，如果大于等于286且连续性不好的话就用双轴快速排序。如果长度小于286且大于等于47的话就用双轴快速排序，如果长度小于47的话就用插入排序。

`Collections.sort`内部会使用`Arrays.sort(T[] a, Comparator<? super T> c)`方法排序，方法定义如下：

```java
public static <T> void sort(T[] a, Comparator<? super T> c) {
    if (c == null) {
        sort(a);
    } else {
        if (LegacyMergeSort.userRequested)
            legacyMergeSort(a, c);
        else
            TimSort.sort(a, 0, a.length, c, null, 0, 0);
    }
}
```

该方法内部使用了TimSort排序（如果`LegacyMergeSort.userRequested`为`true`，则使用旧版的归并排序）。Timsort是结合了合并排序（merge sort）和插入排序（insertion sort）而得出的排序算法。



注意：如果没有实现好`Comparator`接口时可能存在问题，详情见[JDK7中的排序算法详解--Collections.sort和Arrays.sort](http://blog.sina.com.cn/s/blog_8e6f1b330101h7fa.html)。

参考：[Collections.sort()和Arrays.sort()排序算法选择](https://blog.csdn.net/TimHeath/article/details/68930482)

# 三、线程 

#### 1.线程生命周期 

  `Thread.threadStatus`变量记录了线程状态，包括是否处于等待监视锁，是否处于等待`Object.wait()`，可选的值如下：

```c++
// https://github.com/stltqhs/openjdk/blob/jdk7u/jdk7u/jdk/src/share/javavm/export/jvmti.h
enum {
    JVMTI_THREAD_STATE_ALIVE = 0x0001,
    JVMTI_THREAD_STATE_TERMINATED = 0x0002,
    JVMTI_THREAD_STATE_RUNNABLE = 0x0004,
    JVMTI_THREAD_STATE_BLOCKED_ON_MONITOR_ENTER = 0x0400,
    JVMTI_THREAD_STATE_WAITING = 0x0080,
    JVMTI_THREAD_STATE_WAITING_INDEFINITELY = 0x0010,
    JVMTI_THREAD_STATE_WAITING_WITH_TIMEOUT = 0x0020,
    JVMTI_THREAD_STATE_SLEEPING = 0x0040,
    JVMTI_THREAD_STATE_IN_OBJECT_WAIT = 0x0100,
    JVMTI_THREAD_STATE_PARKED = 0x0200,
    JVMTI_THREAD_STATE_SUSPENDED = 0x100000,
    JVMTI_THREAD_STATE_INTERRUPTED = 0x200000,
    JVMTI_THREAD_STATE_IN_NATIVE = 0x400000,
    JVMTI_THREAD_STATE_VENDOR_1 = 0x10000000,
    JVMTI_THREAD_STATE_VENDOR_2 = 0x20000000,
    JVMTI_THREAD_STATE_VENDOR_3 = 0x40000000
};
```



`Thread.threadStatus`与`Thread.State`的转换规则如下：

```c++
// https://github.com/stltqhs/openjdk/blob/jdk7u/jdk7u/jdk/src/share/javavm/export/jvmti.h
enum {
    JVMTI_JAVA_LANG_THREAD_STATE_MASK = JVMTI_THREAD_STATE_TERMINATED | JVMTI_THREAD_STATE_ALIVE | JVMTI_THREAD_STATE_RUNNABLE | JVMTI_THREAD_STATE_BLOCKED_ON_MONITOR_ENTER | JVMTI_THREAD_STATE_WAITING | JVMTI_THREAD_STATE_WAITING_INDEFINITELY | JVMTI_THREAD_STATE_WAITING_WITH_TIMEOUT,
    JVMTI_JAVA_LANG_THREAD_STATE_NEW = 0,
    JVMTI_JAVA_LANG_THREAD_STATE_TERMINATED = JVMTI_THREAD_STATE_TERMINATED,
    JVMTI_JAVA_LANG_THREAD_STATE_RUNNABLE = JVMTI_THREAD_STATE_ALIVE | JVMTI_THREAD_STATE_RUNNABLE,
    JVMTI_JAVA_LANG_THREAD_STATE_BLOCKED = JVMTI_THREAD_STATE_ALIVE | JVMTI_THREAD_STATE_BLOCKED_ON_MONITOR_ENTER,
    JVMTI_JAVA_LANG_THREAD_STATE_WAITING = JVMTI_THREAD_STATE_ALIVE | JVMTI_THREAD_STATE_WAITING | JVMTI_THREAD_STATE_WAITING_INDEFINITELY,
    JVMTI_JAVA_LANG_THREAD_STATE_TIMED_WAITING = JVMTI_THREAD_STATE_ALIVE | JVMTI_THREAD_STATE_WAITING | JVMTI_THREAD_STATE_WAITING_WITH_TIMEOUT
};
```



  从上可以看到`Thread.State`定义了六种状态，分别为：

* New（新建）

  当构造一个线程时（如`new Thread()`），则该线程的状态是`New`，此时`Thread.threadStatus = 0`，当调用`Thread.getState()`时返回`Thread.State.NEW`；

* Runnable（运行）

  该状态线的线程可能正在执行或者正在等待线程调度器选中，一般会将该状态拆分为Ready（就绪）和Running（运行），而Ready状态到Runnable状态是操作系统控制；

* Waiting（无限期等待）

  线程处于无限期等待某一个条件触发，处于该状态的原因是调用了`Object.wait()`、`Thread.join()`、`LockSupport.park()`方法。

* Timed_Waiting（限期等待）

  线程处于限期等待，处于该状态的原因是调用了`Object.wait(long)`、`Thread.join(long)`、`LockSupport.parkNanos(long)`、`LockSupport.parkUntil(long)`。

* Blocked（阻塞）

  线程正在等待监视锁以进入到`synchronized`方法和语句快；

* Terminated（结束）

  线程`run()`方法或者主线程`main()`方法结束或者抛出未捕获的异常时；


某些资料或者书籍会将Waiting、Timed_Waiting以及Blocked合并为一个状态，称为Blocked，即阻塞。

  

下图展示线程状态转换流程。

![线程状态机](https://i.stack.imgur.com/FTHOR.png "线程状态机")

参考：[Java多线程学习(三)---线程的生命周期](https://www.cnblogs.com/sunddenly/p/4106562.html)，[一张图让你看懂JAVA线程间的状态转换](https://my.oschina.net/mingdongcheng/blog/139263)，[对Java线程概念的理解](https://blog.csdn.net/fuzhongmin05/article/details/71425191)，[visualvm thread states](https://stackoverflow.com/questions/27406200/visualvm-thread-states)

#### 2.wait()、sleep()、notify()区别 

* wait

  使线程处于Waiting或者Timed_Waiting，该线程进入wait queue，而且会释放对象的监视锁。使用wait方法时，该线程必须拥有wait对象的监视锁，即wait方法只能放在对象的同步方法或者同步语句块中，如：

  ```java
  Object mon = ...;
  synchronized (mon) {
      mon.wait();
  }
  ```

  wait方法是在Object中定义的native方法。

* sleep

  使线程处于Waiting或者Timed_Waiting，但是不会使线程释放监视锁，且该方法为Thread的静态方法；

* notify

  释放该对象的监视锁，从对象的wait queue中唤醒一个线程或者多个线程（此时调用notifyAll方法），被唤醒的线程需要重新获取该对象的监视锁，以进入同步块。使用notify方法时，该线程必须拥有notify对象的监视锁，即notify方法只能放在对象的同步方法或者同步语句块中，如：

  ```java
  Object mon = ...;
  synchronized(mon) {
      mon.notify();
  }
  ```


参考：[difference between wait and sleep](https://stackoverflow.com/questions/1036754/difference-between-wait-and-sleep)

#### 3.ThreadPoolExecutor原理 

线程的创建和销毁是一个重量级操作，涉及到操作系统底层调用和资源分配问题。操作系统可创建的线程数量有内存限制，即当线程越多消耗的内存越大，且操作系统线程调度的开销也越多。依靠线程池技术可以有效解决这些问题。



Java提供了两个线程池实现类，分别是ThreadPoolExecutor和ScheduledThreadPoolExecutor，继承关系如下：

```java
                +-------------------+
                |     Executor      |
                +-------------------+
                          ^
                          |
              +-----------+------------+
              |    ExecutorService     |
              +------------------------+
                          ^
                          |
             +------------+--------------+
             |  AbstractExecutorService  |
             +---------------------------+
                          ^
                          |
         +----------------+-------------------+
         |                                    |
+--------+-------+                  +---------+----------+
|  ForkJoinPool  |                  | ThreadPoolExecutor |
+----------------+                  +--------------------+
                                              ^
                                              |
                                +-------------+----------------+       
                                | ScheduledThreadPoolExecutor  |
                                +------------------------------+
```

其中Executor和ExecutorService为接口。



ThreadPoolExecutor的构造函数如下：

```java
public ThreadPoolExecutor(int corePoolSize, // 线程池基本大小
                              int maximumPoolSize, // 线程池最大线程数
                              long keepAliveTime, // 空闲线程允许存活的时间
                              TimeUnit unit, // keepAliveTime的时间单位
                              BlockingQueue<Runnable> workQueue, // 任务队列
                              ThreadFactory threadFactory, // 线程工厂，用于创建线程
                              RejectedExecutionHandler handler/*workQueue满时的拒绝策略*/) {
        // ...
    }
```

当调用`ThreadPoolExecutor.execute(Runnable command)`时，先检查`corePoolSize`是否已满，如果不是，创建一个线程执行`command`任务，如果已满，检查`workQueue`是否已满，如果不是，则将`command`加入到`workQueue`中，如果已满，检查`maximumPoolSize`是否已满，如果是，执行拒绝策略，如果不是，创建新线程执行`command`任务。



当创建新线程执行`command`任务时，需要构建一个`Worker`对象，`Worker`的声明如下：

```java
private final class Worker
        extends AbstractQueuedSynchronizer
        implements Runnable {
        // ...
        /** Thread this worker is running in.  Null if factory fails. */
        final Thread thread;
        /** Initial task to run.  Possibly null. */
        Runnable firstTask;
        /** Per-thread task counter */
        volatile long completedTasks;

        /**
         * Creates with given first task and thread from ThreadFactory.
         * @param firstTask the first task (null if none)
         */
        Worker(Runnable firstTask) {
            setState(-1); // inhibit interrupts until runWorker
            this.firstTask = firstTask;
            this.thread = getThreadFactory().newThread(this);
        }

        /** Delegates main run loop to outer runWorker  */
        public void run() {
            runWorker(this);
        }
        // ..
        }
```

`Worker`继承了`AbstractQueuedSynchronizer`，所以`Worker`也具备锁的功能，且`Worker`实现了`Runnable`接口，可以用于创建线程。从`this.thread = getThreadFactory().newThread(this);`可以看到，线程运行时将会执行`Worker`的`run`方法。`runWorker`方法在`ThreadPoolExecutor`中定义，作用是循环的向`workQueue`中取出任务并运行任务（实现`Runnable`接口的任务，执行该任务时直接调用`run`方法即可）。



`AbstractExecutorService.submit(Callable<T> Task)`用于执行带返回值的任务，该方法内部会执行`new FutureTask<T>(callable)`来构造一个`Runnable`任务，调用`execute`方法执行。`FutureTask`同时实现了`Runnable`和`Future`接口，其中`Future`接口表示一个异步计算的结果。



线程池线程数量的规划需要根据任务的性质决定。如果是CPU密集型任务，应配置尽可能小的线程数，一般配置Ncpu+1个线程的线程池；如果是IO密集型任务，由于IO密集型任务线程并不是一直在执行任务，则应配置尽可能多的线程，如2*Ncpu。混合型的任务，如果可以拆分，将其拆分成一个CPU密集型任务  和一个IO密集型任务，只要这两个任务执行的时间相差不是太大，那么分解后执行的吞吐量将高于串行执行的吞吐量。如果这两个任务执行时间相差太大，则没必要进行分解。

参考：[线程池的实现原理](https://blog.csdn.net/wzq6578702/article/details/68926320)

#### 4.ThreadLocal实现方式 

在多线程环境中，如果多个线程并发操作同一个共享变量，由于Java内存模型的原因，存在脏读、丢失更新等内存数据不一致的情况，解决该问题的方式是使用互斥锁。但是使用互斥锁时会降低线程执行的效率，因此在某些情况下可以不使用共享变量，而是使用局部变量。对于方法级别的局部变量，不存在`race condition`（竟态条件），因为该变量只能被该线程访问，但是会造成对象创建过多，导致垃圾回收效率慢。如果可以将变量与某个线程绑定，该变量同样只能被该线程访问，不存在`race condition`，而且对象创建的数量也不会太多，这样线程的执行的效率将会大大提高。

`Thread`类中定义了`ThreadLocal.ThreadLocalMap threadLocals = null;`，表示一个线程的局部变量。`ThreadLocal.ThreadLocalMap`的实现方式类似`HashMap`，该类的`Entry`继承了`WeakReference`，定义如下：

```java
static class Entry extends WeakReference<ThreadLocal> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal k, Object v) {
                super(k);
                value = v;
            }
        }
```

从上可知，`ThreadLocal.ThreadLocalMap`的键是`ThreadLocal`，而且键是**弱引用**，值为强引用。

调用`ThreadLocal.get()`方法时，首先获取当前线程的`threadLocals`变量，类型是`ThreadLocal.ThreadLocalMap`，如果为`null`，则调用`createMap`创建（`t.threadLocals = new ThreadLocalMap(this, firstValue);`），然后返回初始值。如果`threadLocals`不为`null`，则调用`ThreadLocal.ThreadLocalMap.getEntry(ThreadLocal)`方法，根据`ThreadLocal`的哈希值获取`Entry`，如果`Entry.get()`（由于`Entry`继承了`WeakReference`，`Entry.get()`实际上是调用了`WeakReference.get()`方法，由于`Entry`在构造函数中调用了`super(k)`，将键作为弱引用对象，所以`Entry.get()`返回的是键的引用）返回为`null`，表示垃圾收集器回收了键对象，为避免**内存泄漏**，需要调用`ThreadLocal.ThreadLocalMap.expungeStaleEntry`方法回收`Entry`所占用的`slot`，将`value`清空，然后再重哈希。

`ThreadLocal.set()`方法类似`ThreadLocal.get()`方法。

参考：[Java进阶（七）正确理解Thread Local的原理与适用场景](http://www.jasongj.com/java/threadlocal/)

#### 5.中断机制 

Java中断机制是一种协作方式，仅仅是通知一个线程可以中断执行，但是线程本身也可以忽略中断信息什么也不做。从设计上来说，一个线程不应该由其他线程来强制中断或停止，而是应该由线程自己自行停止。所以使用中断机制来替代`Thread.stop()`, `Thread.suspend()`, `Thread.resume()`。



当调用`Thread.interrupt()`方法时，可以为某个线程对象设置中断标记位，即`Thread.threadStatus`的`JVMTI_THREAD_STATE_INTERRUPTED`标记位被打开，此时被设置中断标记位的线程调用`Thread.isInterrupted()`方法会返回`true`。此时如果被设置中断标记位的线程调用`Thread.interrupted()`方法时，返回`true`的同时还会清除中断标记位。



线程需要自己检查是否被中断，检查的方法一般是调用`Thread.interrupted()`方法。如果检查到自己被中断了，可以根据业务规则处理该中断，也可以将中断信息往线程调用栈上报，方式是抛出`InterruptedException`。如果捕获到了`InterruptedException`时，不能什么也不做（除非自己知道业务无需关注中断），要么将异常再抛出，要么再设置中断信息，如下：

```java
try {
   // ...
} catch (InterruptedException e) {
    Thread.currentThread().interrupt(); // or throw e;
}
```



一般来说，当方法声明了要跑出`InterruptedException`则暗示该方法可中断，即方法内部会检查中断标记位，然后抛出异常，并且清除中断标记位。Java中很多阻塞方法如`BlockingQueue.put`、`BlockingQueue.take`、`Object.wait`、`Thread.sleep`。

参考：[详细分析Java中断机制](http://www.infoq.com/cn/articles/java-interrupt-mechanism)

#### 6.活跃性

* 死锁

  如果多个线程以固定的顺序来获取锁时，那么他们将不回存在死锁的问题。死锁发生表示获取锁的顺序不一致导致互相占用各自需要获取的锁。死锁的解决方法有：1)限制锁的调用顺序，2)缩小锁的范围，3)使用显示锁替换内置锁，灵活控制锁策略。

* 饥饿

  由于线程得不到需要的资源，不能正常执行，就会造成线程饥饿。如对线程的优先级设置不当，造成线程不能获得CPU周期执行导致饥饿，或者其他线程长时间持有锁，导致其他线程长时间等待，造成的饥饿。

* 活锁

  活锁问题不会导致线程阻塞，但是活锁会导致线程不能继续正常执行。比如这样一个消息系统中，从队列里边取的消息，然后执行，但是由于某种业务原因，失败了，那么把它放到队列头，然后再拿出来执行，自然还是失败的，这样线程虽然没有阻塞，但是也不能正常的处理其他的消息。

  

  当出现活跃性故障时，除了终止应用程序之外没有其他任何机制可以帮助从这种故障恢复过来。

参考：[java并发编程中的活跃性问题](http://study-a-j8.iteye.com/blog/2366489)

# 四、并发 

#### 1.happens-before原则 

在共享内存的多处理器体系结构中，每个处理器都拥有自己的缓存，并且定期地与主内存进行协调。此时就存在处理器P1修改变量A时，在同步变量A到主内存之前，处理器P2读取变量A将是一个旧值。此类问题只能由程序来控制这种**内存不一致**的问题。

Java为各种处理器体系不同的内存模型，抽象了Java自身的内存模型（Java Memory Model，或简称JMM）。所以Java内存模型不是一个具体的内存，而是抽象的内存，包括处理器多级缓存、寄存器缓存、处理器指令重排序、JVM指令重排序等。

为了解决上述**内存不一致**的情况，Java需要根据自身的语言要求在特定位置插入内存栅栏来刷新缓存数据，保证内存数据和处理器中的缓存数据一致，或者禁止处理器重排序。

Java内存模型为程序中所有的操作定义了一个偏序关系（偏序关系 π 是集合上的一种关系，具有反对称、自反和传递属性，但对于任意两个元素x、y来说，并不需要一定满足x π y或者y π x的关系），称之为Happens-Before。**要想保证执行操作B的线程看到操作A的结果（无论A和B是否在同一个线程中执行），那么在A和B之间必须满足Happens-Before关系**。如果两个操作之前缺乏Happends-Before关系，那么JVM可以对他们任意地重排序。

Happens-Before的规则包括：

* **程序顺序规则** 如果程序中操作A在操作B之前，那么线程中A操作将在B操作之前执行。如果A happens- before B，JMM并不要求A一定要在B之前执行。JMM仅仅要求前一个操作（执行的结果）对后一个操作可见，且前一个操作按顺序排在第二个操作之前；
* **监视器锁规则** 在监视器锁上的解锁操作必须在同一个监视器锁上的加锁操作之前执行；
* **volatile变量规则** 对volatile变量的写入操作必须在对该变量的读取操作之前执行；
* **线程启动规则** 在线程上对`Thread.start()`的调用必须在该线程中执行任何操作之前执行；
* **线程结束规则** 线程中的任何操作都必须在其他线程检测到该线程已经结束之前执行，或者从`Thread.join()`中成功返回，或者在调用`Thread.isAlive()`时返回`false`；
* **中断规则** 当一个线程在另一个线程上调用`interrupt`时，必须在被中断线程检测到`interrupt`调用之前执行（通过抛出`InterruptedException`，或者调用`isInterrupted()`或`interrupted()`）；
* **终结器规则** 对象的构造函数必须在启动该对象的终结器之前执行完成；
* **传递性** 如果操作A在操作B之前执行，并且操作B在操作C之前执行，那么操作A必须在操作C之前执行。

JVM可以保证在调用任何类的静态方法或者静态字段时，JVM都已经对类正确的执行了静态初始化，即调用类的静态初始化函数（`<cinit>`方法）是原子操作。

JVM可以保证`final`字段可以在构造函数退出前完成初始化，不会把`final`字段初始化重排序的构造函数外执行（这里有个前提是在构造函数中没有把`this`逸出）。例如：

```java
public class FinalFieldExample {
    private int i;
    private final int j;
    private FinalFieldExample self;
    private FinalFieldExample obj;
    public FinalFieldExample() {
        i = 2; // JVM可能将 i = 2 重排序到构造函数外
        j = 5; // JVM不回将 j = 5 重排序到构造函数外
        self = this; // this逸出，不安全的操作，因为该操作可能被重排序到 j = 5之前执行
    }
    
    public static void write() {
        obj = new FinalFieldExample();
    }
    
    public static void read() {
        while (obj == null);
        System.out.println(obj.i);
    }
    
    public static void main(String args[]) {
        Thread t1 = new Thread(new Runnable() {
            FinalFieldExample.write();
        });
        t1.start();
        Thread t2 = new Thread(new Runnable() {
            FinalFieldExample.read();
        });
        t2.start();
        // t2线程调用read时会检查obj是否为null，如果不为null，打印变量i的值
        // t1线程调用write，由于write操作可能执行的是先将引用赋值给obj，然后执行 i = 2的初始化
        // 而当t1线程执行将引用赋值给obj值执行t2，t2读到obj引用存在，于是打印i，但此时i还未初始化
    }
}
```

由于指令重排序的原因，对于[“双重检查加锁“（double check lock）](https://blog.csdn.net/jiankunking/article/details/73012648)也是不安全的操作。

参考：[深入理解Java内存模型（一）——基础](http://www.infoq.com/cn/articles/java-memory-model-1)，[深入理解Java内存模型（二）——重排序](http://www.infoq.com/cn/articles/java-memory-model-2)，[深入理解Java内存模型（三）——顺序一致性](http://www.infoq.com/cn/articles/java-memory-model-3)，[深入理解Java内存模型（四）——volatile](http://www.infoq.com/cn/articles/java-memory-model-4)，[深入理解Java内存模型（五）——锁](http://www.infoq.com/cn/articles/java-memory-model-5)，[深入理解Java内存模型(六)——final](http://www.infoq.com/cn/articles/java-memory-model-6)，[Java并发编程实战](https://book.douban.com/subject/10484692/)

#### 2.volatile作用 

volatile有两个作用，一个是将long和double类型的读取和写入操作原子化（由于long和double是64位，JVM内部会将long和double的操作分为两个32位的操作，而且不是原子操作），另一个是控制变量线程间的可见性（**volatile变量规则**规定对volatile变量的写入操作必须在对该变量的读取操作之前执行）。



如果一个int类型变量i被volatile修饰，那么`i++`操作并不是线程安全的操作。`volatile`只能保证变量读取和写入是原子性，且读取变量时会刷新缓存，保证线程间的可见性。而`i++`操作包含3个操作，首先是读取变量到临时变量中，然后临时变量加1，再将临时变量写入，这3个操作合起来并不是原子操作，所以不是线程安全的。

参考：[深入理解Java内存模型（四）——volatile](http://www.infoq.com/cn/articles/java-memory-model-4)

#### 3.CAS 

CAS是“Compare And Swap”的简称，中文含义是“比较并交换”。CAS操作基于CPU提供的原子操作指令实现。

从intel的[CMPXCHG](https://c9x.me/x86/html/file_module_x86_id_41.html)指令来看，CAS的操作流程是比较旧值是否与期望的值一致，如果一致将ZF程序状态字打开，将新值写入。如果与期望的值不一致将ZF程序状态字清除。因此程序可以通过检查ZF来判断CAS操作是否为预期操作，如果不是，读取新写入的值，再次进行CAS操作。所以CAS操作一般都是放在循环中执行。

Java实现CAS操作依然是靠底层处理器来完成，CAS操作方法定义在`sun.misc.Unsafe`，如下：

```java
    /**
     * Atomically update Java variable to <tt>x</tt> if it is currently
     * holding <tt>expected</tt>.
     * @return <tt>true</tt> if successful
     */
    public final native boolean compareAndSwapObject(Object o, long offset,
                                                     Object expected,
                                                     Object x);

    /**
     * Atomically update Java variable to <tt>x</tt> if it is currently
     * holding <tt>expected</tt>.
     * @return <tt>true</tt> if successful
     */
    public final native boolean compareAndSwapInt(Object o, long offset,
                                                  int expected,
                                                  int x);

    /**
     * Atomically update Java variable to <tt>x</tt> if it is currently
     * holding <tt>expected</tt>.
     * @return <tt>true</tt> if successful
     */
    public final native boolean compareAndSwapLong(Object o, long offset,
                                                   long expected,
                                                   long x);
```

方法都是`native`，会调用C++代码来完成。

程序中会要求很多计算操作（比如自增）为原子操作，而就算一个int类型变量被声明为`volatile`也不能保证`++`操作是原子性，一般就会使用加锁的方式来保证原子性。但是加锁效率太低，因此需要使用CAS操作，因为CAS操作是无锁操作，并发性高。Java提供了`AtomicInteger`（还有`AtomicLong`）类，它提供了很多CAS操作（比如自增）。`getAndIncrement`是自增操作的方法，定义如下：

```java
	public final int getAndIncrement() {
        for (;;) {
            int current = get();
            int next = current + 1;
            if (compareAndSet(current, next))
                return current;
        }
    }
```

上面的代码表明CAS操作是放在一个for循环中完成，每次循环都是获取变量的最新值，然后加1，CAS操作，如果不成功，再循环一次，总有一次会成功。

**ABA**问题是无锁结构实现中常见的一种问题，可基本表述为：

1. 进程P1读取了一个数值A

2. P1被挂起(时间片耗尽、中断等)，进程P2开始执行

3. P2修改数值A为数值B，然后又修改回A

4. P1被唤醒，比较后发现数值A没有变化，程序继续执行。

ABA问题可能会导致灾难性的后果，因此在某些场景需要使用特殊的方法解决**ABA**问题。目前解决**ABA**问题的方法是使用一个修改次数的变量作为版本号。Java提供的`AtomicStampedReference`也是基于版本控制来解决**ABA**问题。

参考：[深入浅出CAS](https://www.jianshu.com/p/fb6e91b013cc)，[比较并交换](https://zh.wikipedia.org/wiki/%E6%AF%94%E8%BE%83%E5%B9%B6%E4%BA%A4%E6%8D%A2)，[JAVA中CAS-ABA的问题解决方案AtomicStampedReference](https://juejin.im/entry/5a7288645188255a8817fe26)

#### 4.LockSupport原理

LockSupport提供了`park`和`unpark`方法用于阻塞线程和解除线程阻塞，调用`park`方法时还可以传一个`Blocker`参数，指明线程阻塞的对象，可以用于线程调试。使用`jstack`来dump线程栈信息时看到`parking to wait for  <0x0000000708f32990>`，0x0000000708f32990这个地址的对象就是`Blocker`，如下所示：

```text
"scheduler-200" prio=10 tid=0x00007fbfc8018000 nid=0x48d9 waiting on condition [0x00007fbe6d7d6000]
   java.lang.Thread.State: WAITING (parking)
        at sun.misc.Unsafe.park(Native Method)
        - parking to wait for  <0x0000000708f32990> (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
        at java.util.concurrent.locks.LockSupport.park(LockSupport.java:186)
        at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2043)
        at java.util.concurrent.ScheduledThreadPoolExecutor$DelayedWorkQueue.take(ScheduledThreadPoolExecutor.java:1085)
        at java.util.concurrent.ScheduledThreadPoolExecutor$DelayedWorkQueue.take(ScheduledThreadPoolExecutor.java:807)
        at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1068)
        at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1130)
        at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:615)
        at java.lang.Thread.run(Thread.java:745)
```

`park`和`unpark`方法都是调用`sun.misc.Unsafe`的相关方法实现，这些方法的签名如下：

```java
public native void unpark(Object thread);
public native void park(boolean isAbsolute, long time);
```

这两个方法都是native方法，由底层C++代码实现。不同操作系统平台有不同实现方式，这里以linux平台为例。linux实现的代码在文件`hotspot/src/os/linux/vm/os_linux.cpp `中，方法声明如下：

```c++
void Parker::park(bool isAbsolute, jlong time);
void Parker::unpark();
```

每个线程都有一个许可，但调用`unpark`时，许可置为1，但调用`park`时，如果许可为1，将许可置为0并返回，否则等待许可（由`unpark`释放许可）。这个过程类似于信号量，不同的是许可不能累加，最大值为1。需要特别注意的一点：**`park` 方法还可以在其他任何时间“毫无理由”地返回，因此通常必须在重新检查返回条件的循环里调用此方法**。`unpark`方法可以先于`park`调用。

`Parker`类使用`_counter`字段表示许可，当调用`park`方法时，先尝试将`_counter`置为0（`if (Atomic::xchg(0, &_counter) > 0) return;`当`_counter`大于0时才会成功地设置为0），如果成功，`park`方法返回，如果不成功，调用`pthread_mutex_trylock`方法尝试锁住互斥变量`_mutex `。如果获取锁不成功，`park`方法返回，需要由上层代码继续调用`park`方法。如果获取锁成功，检查`_counter`是否大于0，如果大于0，将`_counter`置为0，调用`pthread_mutex_unlock`方法解除互斥锁，然后返回。如果不大于0，表示没有许可，调用`pthread_cond_wait`方法等待许可。

当调用`unpark`方法时，调用`pthread_mutex_lock`方法获取互斥锁，将`_counter`置为1，即添加许可。如果`_counter`之前为0，则调用`pthread_cond_signal `通知其他线程可以。最后调用`pthread_mutex_unlock`释放互斥锁，`unpark`方法返回。

使用`os::Linux::safe_cond_timedwait`方法可以设置等待一个互斥变量的超时时间。

参考： [浅谈Java并发编程系列（八）—— LockSupport原理剖析](https://segmentfault.com/a/1190000008420938)，[Java的LockSupport.park()实现分析](https://blog.csdn.net/hengyunabc/article/details/28126139)

#### 5.AQS原理 

AQS是`AbstractQueuedSynchronizer`类的简称，它是`java.concurrent.util`包里各种独占锁或者共享锁（包括`ReentrantLock`和`Semaphore`等）实现的基础。

AQS使用一个`volatile int state`表示同步状态，并使用CAS操作保证条件判断与动作更新的原子性。线程阻塞和唤醒使用`LockSupport.park()`和`LockSupport.unpark()`完成，使用FIFO队列管理阻塞的线程。

AQS使用下列方法实现独占锁和共享锁：

```java
// 独占锁
public final void acquire(int arg);
public final boolean release(int arg) ;

// 共享锁
public final void acquireShared(int arg) ;
public final boolean releaseShared(int arg);
```

独占锁或者共享锁是否能够获取（`acquire`或者`acquireShared`）或者释放（`release`或者`releaseShared`），需要由子类实现下列抽象方法：

```java
// 独占锁
protected boolean tryAcquire(int arg);
protected boolean tryRelease(int arg);

// 共享锁
protected int tryAcquireShared(int arg);
protected boolean tryReleaseShared(int arg);
```

这是典型的`模板方法`使用案例。

操作`state`的方法如下：

```java
protected final int getState();
protected final boolean compareAndSetState(int expect, int update);
protected final void setState(int newState);
```

当调用`tryAcquire()`返回true或者`tryAcquireShared()`返回值大于0时，线程不需要阻塞。否则需要向FIFO队列添加一个节点（包括当前线程），阻塞该线程，然后进入`acquireQueued`循环，不停的尝试获取锁。当获取锁时，需要退出`acquireQueued`，同时需要判断后续节点是否为共享模式，如果是，需要将后续线程也唤醒。

当调用`tryRelease()`返回true或者`tryReleaseShared()`返回值大于0时，唤醒FIFO队列head的线程。

`Condition`是AQS定义的内部类`ConditionObject`，必须与独占锁一起使用，它提供了`await`、`signal`和`signalAll`方法来弥补`Object.wait()`、`Object.notify()`和`Object.notifyAll()`的缺陷。`Condition`内部也提供了一个FIFO队列。当调用`await`方法时，释放锁，将当前线程添加到`Condition`的FIFO队列中，阻塞线程，当线程唤醒时需要判断该节点是否进入了AQS的FIFO锁等待队列，如果时，则进入acquire循环获取锁，否则线程继续阻塞。当调用`signal`时，需要将`Condition`的FIFO队列的第一个线程移动到AQS的FIFO队列中，进入锁等待队列。

参考：[AQS 和 高级同步器](http://novoland.github.io/%E5%B9%B6%E5%8F%91/2014/07/26/AQS%20%E5%92%8C%20%E9%AB%98%E7%BA%A7%E5%90%8C%E6%AD%A5%E5%99%A8.html)

#### 6.ReentrantLock,Semaphore,ReadWriteLock,CountDownLatch,CyclicBarrier的原理 

**1).ReentrantLock**

`ReentrantLock`是可重入互斥锁，使用AQS独占锁实现。`ReentrantLock`的成员变量`Sync`实现了`AbstractQueuedSynchronizier`，内部类`FairSync`和`NonfairSync`分别实现了公平锁和非公平锁。可重入机制需要使用`setExclusiveOwnerThread`和`getExclusiveOwnerThread`方法设置和获取独占锁的线程。

非公平锁的实现方式比较简单，首先尝试抢占锁（`compareAndSetState(0, 1)`），如果抢占失败就获取锁，获取失败时进入等待队列。

公平锁则需要检查等待队列是否存在前驱节点，如果存在，则进入等待队列，否则尝试获取锁。

从效率上来说，非公平锁高于公平锁，因为非公平锁如果抢占成功就少了入队操作，也少了线程阻塞和唤醒的操作系统调用过程。

**2).Semaphore**

信号量Semaphore使用AQS共享锁实现，维护一个许可集，获取n个许可时，许可集减少n，当小于0时则线程阻塞。释放n个许可时，许可集加上n，同时需要唤醒等待许可的线程。

Semaphore使用AQS的state来表示许可集，Semaphore的构造函数接收一个许可集初始容量大小的值。

**3).ReadWriteLock**

`ReadWriteLock`是读写锁的接口，实现该接口的类有`ReentrantReadWriteLock`，这里讲述`ReentrantReadWriteLock`的实现方式。写锁是独占锁，读锁是共享锁，而且读、写锁互斥。`ReentrantReadWriteLock`使用AQS的state字段的高16位为读锁计数器，低16位为写锁计数器。`ReentrantReadWriteLock`内部存在两个实现了`Lock`接口的内部类，分别是`ReadLock`和`WriteLock`，表示读锁和写锁。`ReadLock`的获取和释放锁的方法如下：

```java
public void lock() {
    sync.acquireShared(1);
}
public  void unlock() {
    sync.releaseShared(1);
}
```

从上面的方法调用可以看到`ReadLock`是调用AQS的共享锁方法。获取读锁时要求state的低16位必须为0，即写锁没有被任何线程获取。如果低16位大于0，表示写锁被线程获取，如果获取写锁的线程不是自己，则线程阻塞。如果线程可以获取读锁，则state的高16位加1（`compareAndSetState(c, c + SHARED_UNIT)`，其中`SHARED_UNIT`为`1 << SHARED_SHIFT`），同时线程局部变量`HoldCounter`（`ThreadLocal`）的计数器count字段需要加1，用来保存线程占用的共享读锁的数量。`HoldCounter`保存的计数器的作用是用来判断当前线程在获得写锁的情况下，是否又再获取读锁，此时读锁能够被获取，支持重入机制。当写锁释放时，该线程就降级为只获取读锁。

`WriteLock`的获取和释放锁的方法如下：

```java
public void lock() {
    sync.acquire(1);
}
public void unlock() {
    sync.release(1);
}
```

获取写锁时，需要判断`state`是否存在写锁或者读锁，如果存在且不是当前线程获取时，当前线程需要阻塞。

**4).CountDownLatch**

闭锁CountDownLatch能够使一个线程等待其他线程完成各自的工作后再执行。

CountDownLatch使用AQS共享锁实现，构造函数的参数`count`作为AQS的state的值。调用`await`方法时，如果state不为0则线程阻塞。调用`count`方法时，state减小，当state为0时则唤醒等待的线程。

**5).CyclicBarrier**

循环屏障CyclicBarrier类似一个可循环使用的CountDownLatch，可以让一组线程达到一个屏障时被阻塞，直到最后一个线程达到屏障时，所有被阻塞的线程才能继续执行。 CyclicBarrier好比一扇门，默认情况下是关闭状态，堵住了线程执行的道路，直到所有线程都就位，门才打开，让所有线程一起通过。

CyclicBarrier几个重要的成员变量如下：

```java
/** The lock for guarding barrier entry */
private final ReentrantLock lock = new ReentrantLock();
/** Condition to wait on until tripped */
private final Condition trip = lock.newCondition();
/** The number of parties */
private final int parties;
/* The command to run when tripped */
private final Runnable barrierCommand;
/** The current generation */
private Generation generation = new Generation();

/**
 * Number of parties still waiting. Counts down from parties to 0
 * on each generation.  It is reset to parties on each new
 * generation or when broken.
 */
private int count;
```

`lock`用于保护屏障信息，`trip`用于阻塞线程，`parties`表示屏障数量，`count`表示当前消耗的屏障数量。当调用`wait`方法时，`lock`需要调用`lock()`方法获取锁，然后将count减少，如果count不为0，则当前线程进入trip的Condition等待队列。如果count为0，需要生成一个新的`generation`对象，表示新的一轮循环屏障，同时会调用`condition.signalAll()`方法通知所有等待线程。

参考：[Java并发之ReentrantLock详解](https://blog.csdn.net/lipeng_bigdata/article/details/52154637)，[什么时候使用CountDownLatch](http://www.importnew.com/15731.html)，[JAVA多线程--信号量(Semaphore)](https://my.oschina.net/cloudcoder/blog/362974)，[深入浅出java CyclicBarrier](https://www.jianshu.com/p/424374d71b67)，[Java多线程（十）之ReentrantReadWriteLock深入分析](https://my.oschina.net/adan1/blog/158107)

#### 7.BlockingQueue原理

BlockingQueue接口有两个重要方法，分别是取元素`take`和加入元素`put`。以ArrayBlockingQueue为例，`put`方法如下：

```java
public void put(E e) throws InterruptedException {
    checkNotNull(e);
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        while (count == items.length)
            notFull.await();
        enqueue(e);
    } finally {
        lock.unlock();
    }
}
private void enqueue(E x) {
    // assert lock.getHoldCount() == 1;
    // assert items[putIndex] == null;
    final Object[] items = this.items;
    items[putIndex] = x;
    if (++putIndex == items.length)
        putIndex = 0;
    count++;
    notEmpty.signal();
}
```

从上可以看到`put`方法使用`ReentrantLock`来实现线程安全操作，当数组元素数量为数组长度时，表示队列已经满了，需要等待队列不满，所以`put`方法也是阻塞方法，而等待的方式使用的是AQS的等待队列。`ArrayBlockingQueue`使用两个变量来存储等待队列不为空和等待队列不为满的线程。如下：

```java
notEmpty = lock.newCondition();
notFull =  lock.newCondition();
```

当添加元素成功时，需要通知`notEmpty`等待的线程，表示队列不是空的。

`take`方法如下：

```java
public E take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        while (count == 0)
            notEmpty.await();
        return dequeue();
    } finally {
        lock.unlock();
    }
}
private E dequeue() {
    // assert lock.getHoldCount() == 1;
    // assert items[takeIndex] != null;
    final Object[] items = this.items;
    @SuppressWarnings("unchecked")
    E x = (E) items[takeIndex];
    items[takeIndex] = null;
    if (++takeIndex == items.length)
        takeIndex = 0;
    count--;
    if (itrs != null)
        itrs.elementDequeued();
    notFull.signal();
    return x;
}
```

从上可以看到`take`方法使用`ReentrantLock`来实现线程安全操作，当数组元素数量为0时，表示队列已经为空，需要等待队列不为空，所以`take`方法也是阻塞方法。当取出元素成功时，需要通知`notFull`表示线程不是满的。

参考：[Java并发编程-阻塞队列(BlockingQueue)的实现原理](https://blog.csdn.net/chenchaofuck1/article/details/51660119)

#### 8.synchronized原理 

Java同步机制使用`synchronized`关键字实现。`synchronized`有两种用法，第一种是修饰方法，即同步方法块，第二种是同步代码块，同步代码块和同步方法块被称为临界区。

```java
synchronized void syncMethodBlock() {} // 同步方法块
void syncCodeBlock() {
    synchronized(this) { // 同步代码块
    }
}
```

对于同步代码块，当源代码编译成字节码时，会存在`monitorenter`和`monitorexit`两个字节指令，所表示的意思就是进入临界区和退出临界区。而同步方法块没有这两个指令，由JVM内部判断方法修饰符是否存在`ACC_SYNCHRONIZED`标志，如果存在，JVM内部处理进入临界区和退出临界区的逻辑。



在JVM中，Java对象在内存中的布局分为三块：对象头，实例数据和对齐填充数据（字节对齐在计算机中经常使用，它的作用有解决不同处机器架构内存访问的问题、提高内存访问速度）。指向对象的指针称为`Ordinary Object Pointer`(OOP)，Java对象使用C++中的[oopDesc](https://github.com/dmlloyd/openjdk/blob/jdk7u/jdk7u/hotspot/src/share/vm/oops/oop.hpp)来表示，该类定义了一个变量`volatile markOop  _mark`，而`markOop`是一个指向[markOopDesc](https://github.com/dmlloyd/openjdk/blob/jdk7u/jdk7u/hotspot/src/share/vm/oops/markOop.hpp)类型的指针，`markOopDesc`就是上述所说的对象头，称为`Mark Word`。在32位JVM中，`markOopDesc`所表示的字节是32位，布局如下：

```text
hash:25 —>| age:4 biased_lock:1 lock:2
```

hash就是`Object.hashCode()`的返回值，age表示对象在垃圾收集过程中幸存的年龄，biased_lock表示是否是偏向锁，lock表示锁标状态。`Mark Word`是Java实现同步机制的基础。程序进入临界区时需要获取的锁的结构是[ObjectMonitor](https://github.com/dmlloyd/openjdk/blob/jdk7u/jdk7u/hotspot/src/share/vm/runtime/objectMonitor.hpp)，称为监视锁。监视锁是重量级锁，因为它需要调用操作系统方法来完成，涉及到操作系统“用户态”向“内核态”的切换，需要一些开销。Java对锁进行了一系列优化来降低使用重量级锁的开销，在没有必要使用重量级锁的场景时使用其他锁来完成同步操作，其他锁包括偏向锁、轻量级锁。使用biased_lock和lock位来表示锁的状态，锁的状态从低到高分别是无锁状态、偏向锁、轻量级锁、重量级锁，~~锁状态的变化只能是从低到高，不能从高到低，即只存在锁升级，不存在锁降级~~<sup>[[HotSpot VM重量级锁降级机制的实现原理]](https://zhuanlan.zhihu.com/p/28505703)</sup>。`ObjectMonitor`的3个重要字段为`_count`，它是记录获取锁的数量，因为Java同步锁机制支持重入，每次重入，该计数器都要加1；`_WaitSet`，它是等待线程的集合，调用`Object.wait()`时线程被放入该集合中；`_cxq`，它是FILO竞争队列，应对多线程竞争锁的时候，使用CAS操作替换队列头部；`_EntryList`，cxq中的合适线程可以被放入EntryList，Wait Set中的线程被notify()之后，也会放入EntryList中，准备竞争锁<sup>[[java重量锁，轻量锁，偏向锁]](https://focusvirtualization.blogspot.com/2017/02/linux-85-java.html)</sup>。

各种锁状态的变化过程如下：

* 偏向锁

  引入偏向锁是为了在无多线程竞争的情况下尽量减少不必要的轻量级锁执行路径，因为轻量级锁的获取及释放依赖多次CAS原子指令，而偏向锁只需要在置换ThreadID的时候依赖一次CAS原子指令（由于一旦出现多线程竞争的情况就必须撤销偏向锁，所以偏向锁的撤销操作的性能损耗必须小于节省下来的CAS原子指令的性能消耗）。

  获取偏向锁的过程如下：

  （1）访问Mark Word中偏向锁的标识是否设置成1，锁标志位是否为01——确认为可偏向状态；

  （2）如果为可偏向状态，则测试线程ID是否指向当前线程，如果是，进入步骤（5），否则进入步骤（3）；

  （3）如果线程ID并未指向当前线程，则通过CAS操作竞争锁。如果竞争成功，则将Mark Word中线程ID设置为当前线程ID，然后执行（5）；如果竞争失败，执行（4）；

  （4）如果CAS获取偏向锁失败，则表示有竞争。当到达全局安全点（safepoint）时获得偏向锁的线程被挂起，偏向锁升级为轻量级锁，然后被阻塞在安全点的线程继续往下执行同步代码；

  （5）执行同步代码。

  偏向锁的释放：

  偏向锁只有遇到其他线程尝试竞争偏向锁时，持有偏向锁的线程才会释放锁，线程不会主动去释放偏向锁。偏向锁的撤销，需要等待全局安全点（在这个时间点上没有字节码正在执行），它会首先暂停拥有偏向锁的线程，判断锁对象是否处于被锁定状态，撤销偏向锁后恢复到未锁定（标志位为“01”）或轻量级锁（标志位为“00”）的状态。

* 轻量级锁

  “轻量级”是相对于使用操作系统互斥量来实现的传统锁而言的（重量级锁锁），轻量级锁并不是用来代替重量级锁的，它的本意是在没有多线程竞争的前提下，减少传统的重量级锁使用产生的性能消耗。轻量级锁所适应的场景是线程交替执行同步块的情况，如果存在同一时间访问同一锁的情况，就会导致轻量级锁膨胀为重量级锁。

  轻量级锁的加锁过程如下：

  （1）在代码进入同步块的时候，如果同步对象锁状态为无锁状态（锁标志位为“01”状态，是否为偏向锁为“0”），虚拟机首先将在当前线程的栈帧中建立一个名为锁记录（Lock Record）的空间，用于存储锁对象目前的`Mark Word`的拷贝，称之为 `Displaced Mark Word`；

  （2）拷贝对象头中的`Mark Word`复制到锁记录中；

  （3）拷贝成功后，虚拟机将使用CAS操作尝试将对象的`Mark Word`更新为指向Lock Record的指针，并将Lock record里的owner指针指向object mark word。如果更新成功，则执行步骤（4），否则执行步骤（5）；

  （4）如果这个更新动作成功了，那么这个线程就拥有了该对象的锁，并且对象`Mark Word`的锁标志位设置为“00”，即表示此对象处于轻量级锁定状态；

  （5）如果这个更新操作失败了，虚拟机首先会检查对象的`Mark Word`是否指向当前线程的栈帧，如果是就说明当前线程已经拥有了这个对象的锁，那就可以直接进入同步块继续执行。否则说明多个线程竞争锁，轻量级锁就要膨胀为重量级锁，锁标志的状态值变为“10”，`Mark Word`中存储的就是指向重量级锁（互斥量）的指针，后面等待锁的线程也要进入阻塞状态。 而当前线程便尝试使用自旋来获取锁，自旋就是为了不让线程阻塞，而采用循环去获取锁的过程。

  轻量级锁的释放过程如下：

  （1）通过CAS操作尝试把线程中复制的Displaced Mark Word对象替换当前的`Mark Word`。

  （2）如果替换成功，整个同步过程就完成了。

  （3）如果替换失败，说明有其他线程尝试过获取该锁（此时锁已膨胀），那就要在释放锁的同时，唤醒被挂起的线程。

* 重量级锁

  重量级锁的实现在[ObjectMonitor.cpp](https://github.com/dmlloyd/openjdk/blob/jdk7u/jdk7u/hotspot/src/share/vm/runtime/objectMonitor.cpp)中完成，获取锁的方法为`void ATTR ObjectMonitor::enter(TRAPS) `，释放锁的方法为`void ATTR ObjectMonitor::exit(bool not_suspended, TRAPS) `。

  获取锁的过程如下：

  （1）设置`ObjectMonitor`的`_owner`字段为当前线程，如果设置失败时需要检查是否重入，设置成功时则表示获取锁成功；

  （2）通过自旋执行`void ATTR ObjectMonitor::EnterI (TRAPS) `方法等待锁的释放进入方法中。该方法的逻辑是

         a.当前线程被封装成ObjectWaiter对象node，状态设置成ObjectWaiter::TS_CXQ；
        
         b.在for循环中，通过CAS把node节点push到`_cxq`列表中，同一时刻可能有多个线程把自己的node节点push到`_cxq`列表中；
        
         c.node节点push到`_cxq`列表之后，通过自旋尝试获取锁，如果还是没有获取到锁，则通过park将当前线程挂起，等待被唤醒；
        
         d.当该线程被唤醒时，会从挂起的点继续执行，通过`ObjectMonitor::TryLock`尝试获取锁。

  其本质就是通过CAS设置monitor的_owner字段为当前线程，如果CAS成功，则表示该线程获取了锁，跳出自旋操作，执行同步代码，否则继续被挂起；

  当某个持有锁的线程执行完同步代码块时，会进行锁的释放，给其它线程机会执行同步代码，在HotSpot中，通过退出monitor的方式实现锁的释放，并通知被阻塞的线程，具体实现位于`ObjectMonitor::exit`方法中。

  释放锁的过程如下：

  （1）如果是重量级锁的释放，monitor中的_owner指向当前线程，即THREAD == _owner；

  （2）根据不同的策略（由QMode指定），从cxq或EntryList中获取头节点，通过`ObjectMonitor::ExitEpilog`方法唤醒该节点封装的线程，唤醒操作最终由unpark完成；

  （3）被唤醒的线程，继续执行monitor的竞争；

为了减少重量级锁的操作，引进了偏向锁和轻量级锁。在某些场景下还可以对获取锁的过程做进一步优化，如下：

* 适应性自旋

  当线程在获取轻量级锁的过程中执行CAS操作失败时，是要通过自旋来获取重量级锁的。自旋的意思循环检查是否可以获取重量级做。JVM内部根据运行时信息觉得自旋的次数，即循环次数。适应性自旋，简单来说就是线程如果自旋成功了，则下次自旋的次数会更多，如果自旋失败了，则自旋的次数就会减少。

* 锁粗化

  锁粗化的就是将多次连接在一起的加锁、解锁操作合并为一次，将多个连续的锁扩展成一个范围更大的锁。例如：

  ```java
  public class StringBufferTest {
      StringBuffer stringBuffer = new StringBuffer();
  
      public void append(){
          stringBuffer.append("a");
          stringBuffer.append("b");
          stringBuffer.append("c");
      }
  }
  ```

  这里每次调用stringBuffer.append方法都需要加锁和解锁，如果虚拟机检测到有一系列连串的对同一个对象加锁和解锁操作，就会将其合并成一次范围更大的加锁和解锁操作，即在第一次append方法时进行加锁，最后一次append方法结束后进行解锁。

* 锁消除

  锁消除即删除不必要的加锁操作。根据代码逃逸技术，如果判断到一段代码中，堆上的数据不会逃逸出当前线程，那么可以认为这段代码是线程安全的，不必要加锁。例如：

  ```java
  public class SynchronizedTest02 {
      public static void main(String[] args) {
          SynchronizedTest02 test02 = new SynchronizedTest02();
          //启动预热
          for (int i = 0; i < 10000; i++) {
              i++;
          }
          long start = System.currentTimeMillis();
          for (int i = 0; i < 100000000; i++) {
              test02.append("abc", "def");
          }
          System.out.println("Time=" + (System.currentTimeMillis() - start));
      }
  
      public void append(String str1, String str2) {
          StringBuffer sb = new StringBuffer();
          sb.append(str1).append(str2);
      }
  }
  ```

  虽然StringBuffer的append是一个同步方法，但是这段程序中的StringBuffer属于一个局部变量，并且不会从该方法中逃逸出去，所以其实这过程是线程安全的，可以将锁消除。

参考：[深入理解Java并发之synchronized实现原理](https://blog.csdn.net/javazejian/article/details/72828483)，[Getting Started with HotSpot and OpenJDK](https://www.infoq.com/articles/introduction-to-hotspot)，[Java并发编程：Synchronized底层优化（偏向锁、轻量级锁）](https://www.cnblogs.com/paddix/p/5405678.html)，[JVM源码分析之synchronized实现](https://www.jianshu.com/p/c5058b6fe8e5)

#### 9.锁的升级和降级

锁降级在`ReentrantReadWriteLock`中的意思是从写锁降级为读锁，但是`ReentrantReadWriteLock`不能从读锁升级为写锁。

锁降级在JVM中是指重量级锁降级为轻量级锁或者偏向锁。

锁升级在JVM中是指偏向锁升级为轻量级锁，轻量级锁升级为重量级锁。

#### 10.多种方式实现生产者和消费者模式

* wait/notify

* ReentrantLock/Condition

* BlockingQueue

* Semaphore

* PipedInputStream/PipedOutputStream

  该方法不是基于线程同步完成，因此只能满足于一个生产者和一个消费者。原理是先创建一个管道输入流和管道输出流，然后将输入流和输出流进行连接，用生产者线程往管道输出流中写入数据，消费者在管道输入流中读取数据，这样就可以实现了不同线程间的相互通讯，但是这种方式在生产者和生产者、消费者和消费者之间不能保证同步，也就是说在一个生产者和一个消费者的情况下是可以生产者和消费者之间交替运行的，多个生成者和多个消费者者之间则不行

参考：[Java实现生产者和消费者的5种方式](https://juejin.im/entry/596343686fb9a06bbd6f888c)

# 五、Servlet

#### 1.Servlet生命周期与过滤器

Servlet 生命周期可被定义为从创建直到毁灭的整个过程。以下是 Servlet 遵循的过程：

- Servlet 通过调用 **init ()** 方法进行初始化；

  init 方法被设计成只调用一次。它在第一次创建 Servlet 时被调用，在后续每次用户请求时不再调用。因此，它是用于一次性初始化。

- Servlet 调用 **service()** 方法来处理客户端的请求；

  service() 方法是执行实际任务的主要方法。Servlet 容器（即 Web 服务器）调用 service() 方法来处理来自客户端（浏览器）的请求，并把格式化的响应写回给客户端。

  每次服务器接收到一个 Servlet 请求时，服务器会产生一个新的线程并调用服务。service() 方法检查 HTTP 请求类型（GET、POST、PUT、DELETE 等），并在适当的时候调用 doGet、doPost、doPut，doDelete 等方法。

- Servlet 通过调用 **destroy()** 方法终止（结束）；

- 最后，Servlet 是由 JVM 的垃圾回收器进行垃圾回收的。

Servlet中的过滤器Filter是实现了javax.servlet.Filter接口的服务器端程序，只要你在web.xml 文件配置好要拦截的客户端请求，它都会帮你拦截到请求。Filter可认为是Servlet的一种“变种”，它主要用于对用户请求进行预处理，也可以对HttpServletResponse进行后处理，是个典型的处理链。它与Servlet的区别在于：它不能直接向用户生成响应。完整的流程是：Filter对用户请求进行预处理，接着将请求交给Servlet进行处理并生成响应，最后Filter再对服务器响应进行后处理。

参考：[Servlet 生命周期](http://www.runoob.com/servlet/servlet-life-cycle.html)，[Servlet,Filter,Listener,Interceptor的作用和区别](https://my.oschina.net/hapier/blog/699193)

#### 2.Session和Cookie的区别

Session是在服务端保存的一个数据结构，用来跟踪用户的状态，这个数据可以保存在集群、数据库、文件中；

Cookie是客户端保存用户信息的一种机制，用来记录用户的一些信息，也是实现Session的一种方式。

参考：[看完就彻底懂了session和cookie](https://www.jianshu.com/p/25802021be63)

#### 3.Servlet的异步请求

* Servlet3.0的异步请求

  [Servlet3.0规范](http://download.oracle.com/otn-pub/jcp/servlet-3.0-fr-oth-JSpec/servlet-3_0-final-spec.pdf?AuthParam=1535295644_2a1916a1ba6dc83fc114e98f80b21f78)2.3.3.3节的“Asynchronous processing”讲述了异步处理，当启用异步处理时，响应数据可以由其他线程创建，而当前线程可以立即交还给Servlet容器处理其他请求。当要启用异步处理时，首先需要声明HttpServlet支持异步处理，然后才能调用`request.startAsync`方法获得`AsyncContext`，它包含`request`和`response`。调用`startAsync`方法后，程序能保证在退出`service`方法时，不会完成本次请求，即不响应任何数据给客户端。需要在之后调用`AsyncContext.complete`方法来完成本次请求，向客户端写入数据。

* Servlet3.1的NIO

  [Servlet3.1规范](https://javaee.github.io/servlet-spec/downloads/servlet-3.1/Final/servlet-3_1-final.pdf)第3.7节“Non Blocking IO”讲述了非阻塞IO。Servlet 3.0对请求的处理虽然是异步的，但是对InputStream和OutputStream的IO操作却依然是阻塞的，对于数据量大的请求体或者返回体，阻塞IO也将导致不必要的等待，因此在Servlet 3.1中引入了非阻塞IO。

参考：[使用异步Servlet改进应用性能](http://www.infoq.com/cn/news/2013/11/use-asynchronous-servlet-improve)，[servlet3异步原理与实践](https://www.jianshu.com/p/c23ca9d26f64)

#### 4.Tomcat架构

Tomcat的Servlet容器是用来处理请求servlet资源，并为web客户端填充response对象的模块。在Tomcat中，共有四种类型的容器，分别是：Engine、Host、Context和Wrapper。servlet容器是[org.apache.catalina.Container](https://github.com/apache/tomcat/blob/TOMCAT_8_0_0/java/org/apache/catalina/Container.java)接口的实例。Container接口及其相关类的UML类图如下：

```text
                                     +----------------+
                                     | <<interface>>  |
                                     |   Container    |
                                     +----------------+
                                             ^       ^
                                             |       |
                                             |       +---------------------------+
                                             |                                   |
       +------------------+------------------+------------------+                |
       |                  |                  |                  |                |
+-------------+    +-------------+    +-------------+    +-------------+   +-------------+
|<<interface>>|    |<<interface>>|    |<<interface>>|    |<<interface>>|   |ContainerBase|
|   Engine    |    |     Host    |    |   Context   |    |   Wrapper   |   |             |
+-------------+    +-------------+    +-------------+    +-------------+   +-------------+
       ^                  ^                   ^                     ^             ^
       |                  |                   |                     |             |
       +------------------+-------------------+---------------------+-------------+
       |                  |                   |                     |
+--------------+ INC  +-------------+ INC  +---------------+ INC  +---------------+
|StandardEngine| <>-- | StandartHost| <>-- |StandardContext| <>-- |StandardWrapper|
+--------------+ 1  n +-------------+ 1  n +---------------+ 1  n +---------------+
```

Tomcat的[Pipeline](https://github.com/apache/tomcat/blob/TOMCAT_8_0_0/java/org/apache/catalina/Pipeline.java)管道和[Valve](https://github.com/apache/tomcat/blob/TOMCAT_8_0_0/java/org/apache/catalina/Valve.java)阀的工作机制类似于servlet的过滤器，在servlet容器的管道中，有一个基础阀，也可以添加任意数量的阀，基础阀总是最后一个执行。

当Tomcat启动时，相关组件也会一起启动，Tomcat关闭时，这些组件也会随之关闭，这是通过实现[Lifecycle](https://github.com/apache/tomcat/blob/TOMCAT_8_0_0/java/org/apache/catalina/Lifecycle.java)接口达到统一启动/关闭组件的效果。实现了`Lifecycle`接口的组件可以触发一个或多个事件：`BEFORE_START_EVENT`、`START_EVENT`、`AFTER_START_EVENT`、`BEFORE_STOP_EVENT`、`STOP_EVENT`、`AFTER_STOP_EVENT`，事件是[LifecycleEvent](https://github.com/apache/tomcat/blob/TOMCAT_8_0_0/java/org/apache/catalina/LifecycleEvent.java)类的实例，事件监听器是[LifecycleListener](https://github.com/apache/tomcat/blob/TOMCAT_8_0_0/java/org/apache/catalina/LifecycleListener.java)类的实例，[LifecycleSupport](https://github.com/apache/tomcat/blob/TOMCAT_8_0_0/java/org/apache/catalina/util/LifecycleSupport.java)提供了一种简单的方法来触发某个组件的生命周期事件，并对事件监听器进行处理。

Tomcat载入器是[Loader](https://github.com/apache/tomcat/blob/TOMCAT_8_0_0/java/org/apache/catalina/Loader.java)接口的实例，[WebappLoader](https://github.com/apache/tomcat/blob/TOMCAT_8_0_0/java/org/apache/catalina/loader/WebappLoader.java)是负责从`WEB-INF/classes`和`WEB-INF/lib`载入servlet所需的类的载入器，它内部会使用[WebappClassLoader](https://github.com/apache/tomcat/blob/TOMCAT_8_0_0/java/org/apache/catalina/loader/WebappClassLoader.java)作为类载入器（或类加载器）。`Loader`接口使用`modified()`方法来支持类的自动重载，在载入器的具体实现中，如果仓库（`Loader.addRepository()`）中的一个或多个类被修改，那么`modified()`方法必须返回`true`，载入器内部会调用`Context.reload`完成类重新载入的过程。载入类时，`WebappClassLoader`遵循如下规则：

* 因为所有已经载入的类都会缓存起来，所以载入类时要先检查本地缓存；
* 若本地缓存中没有，则检查上一层缓存，即调用`java.lang.ClassLoader`类的`findLoadedClass()`方法；
* 如果两个缓存中都没有，则使用系统的类载入器`ApplicationClassLoader`进行加载，防止Web应用程序中的类覆盖Java EE的类；
* 若启用了`SecurityManager`，则检查是否运行载入该类。若该类是禁止载入的类，抛出`ClassNotFoundException`异常；
* 若打开标志位`delegate`，或者待载入的类是属于包触发器`WebappClassLoader.packageTriggers`中的包名，则调用父类载入器来载入相关类。如果父类载入器为`null`，则使用系统的类载入器；
* 从当前仓库中载入相关类；
* 若当前仓库中没有需要的类，且标志位`delegate`关闭，则使用父类载入器。若父类载入器为`null`，则使用系统的类载入器进行加载；
* 若仍未找到需要的类，则抛出`ClassNotFoundException`异常。

Tomcat使用Session管理器的组件来管理建立的Session对象，该组件由[Manager](https://github.com/apache/tomcat/blob/TOMCAT_8_0_0/java/org/apache/catalina/Manager.java)接口表示。Session管理器需要与一个`Context`容器相关联，且必须与一个`Context`容器关联。Session管理器负责创建、更新、销毁Session对象，当有请求到来时，要返回一个有效的Session对象。Servlet实例通过调用`javax.servlet.http.HttpServletRequest.getSession()`方法获取一个Session对象。在Tomcat的默认连接器中，[Request](https://github.com/apache/tomcat/blob/TOMCAT_8_0_0/java/org/apache/catalina/connector/Request.java)类实现`HttpServletRequest`接口。Session管理器会将其所管理的Session对象放在内存中，或存储在文件中，或通过JDBC写入数据库中。Session对象是[Session](https://github.com/apache/tomcat/blob/TOMCAT_8_0_0/java/org/apache/catalina/Session.java)接口的实例，实现类是[StandardSession](https://github.com/apache/tomcat/blob/TOMCAT_8_0_0/java/org/apache/catalina/session/StandardSession.java)。[StandardManager](https://github.com/apache/tomcat/blob/TOMCAT_8_0_0/java/org/apache/catalina/session/StandardManager.java)是`Manager`接口的实现类，session管理器需要负责销毁已经失效的Session对象，在[ManagerBase](https://github.com/apache/tomcat/blob/TOMCAT_8_0_0/java/org/apache/catalina/session/ManagerBase.java)类的`backgroundProcess`方法负责过期Session对象的检查与销毁。[Store](https://github.com/apache/tomcat/blob/TOMCAT_8_0_0/java/org/apache/catalina/Store.java)接口是为Session管理器的Session对象提供持久化存储器的一个组件，`Store`接口两个重要的方法是`save`和`load`。[StoreBase](https://github.com/apache/tomcat/blob/TOMCAT_8_0_0/java/org/apache/catalina/session/StoreBase.java)是一个抽象类，提供了基本功能，该类的两个子类是[FileStore](https://github.com/apache/tomcat/blob/TOMCAT_8_0_0/java/org/apache/catalina/session/FileStore.java)和[JDBCStore](https://github.com/apache/tomcat/blob/TOMCAT_8_0_0/java/org/apache/catalina/session/JDBCStore.java)。

一般情况下一个`Context`容器包含一个或多个`Wrapper`，而一个`Wrapper`对应一个`HttpServlet`。[StandardWrapper](https://github.com/apache/tomcat/blob/TOMCAT_8_0_0/java/org/apache/catalina/core/StandardWrapper.java)是[Wrapper](https://github.com/apache/tomcat/blob/TOMCAT_8_0_0/java/org/apache/catalina/Wrapper.java)接口的实现类，它的基础阀[StandardWrapperValve](https://github.com/apache/tomcat/blob/TOMCAT_8_0_0/java/org/apache/catalina/core/StandardWrapperValve.java)会调用servlet的过滤器和servlet的`service`方法。连接器接收到HTTP请求后，方法调用的协作图如下：

```text
1:[new request] create() ->
2:[new response] create() ->
            +----+
Message1 -> |    | 3:invoke() ->                   4:invoke() ->
         +-----------+           +-----------------+             +-----------------------+
---------| Connector | ----------| StandardContext |-------------|StandardContextPipeline|
         +-----------+           +-----------------+             +-----------------------+
                                                                 5:invoke() |
                                                                     |      |
                           <-7:invoke()              <-6:invoke()    v      |
   +----------------------+          +-----------------+         +----------------------+
   | StandardWrapperValve |----------| StandardWrapper |---------| StandardContextValve |
   +----------------------+          +-----------------+         +----------------------+
              |
8:allocate()  | 11:service()
     |        |      |
     v        |      v
   +----------------------+  9:load()->
   |       Servlet        |---------------
   +----------------------+  10:init()->
```

协作图过程描述如下：

* 1.连接器创建request和response对象；
* 2.连接器调用`StandardContext`实例的`invoke()`方法；
* 3.`StandardContext`实例的`invoke()`方法调用其管道对象的`invoke()`方法。`StandardContext`中管道对象的基础阀是`StandardContextValve`类的实例，因此，`StandardContext`的管道对象会调用`StandardContextValve`实例的`invoke()`方法；
* 4.`StandardContextValve`实例的`invoke()`方法获取相应的`Wrapper`实例处理HTTP请求。调用`Wrapper`实例的`invoke()`方法；
* 5.`StandardWrapper`类是`Wrapper`接口的标准实现，`StandardWrapper`实例的`invoke()`方法会表用其管道对象的`invoke`方法；
* 6.`StandardWrapper`的管道对象中的基础阀是`StandardWrapperValve`类的实例，因此，会调用`StandardWrapperValve`的`invoke()`方法，`StandardWrapperValve`的`invoke()`方法调用`Wrapper`实例的`allocate()`方法获取servlet实例；
* 7.`allocate()`方法调用`load()`方法载入相应的servlet类，若已经载入，则无需重复载入；
* 8.`load()`方法调用servlet实例的`init()`方法；
* 9.`StandardWrapperValve`调用servlet实例的`service()`方法。

[FilterDef](https://github.com/apache/tomcat/blob/TOMCAT_8_0_0/java/org/apache/tomcat/util/descriptor/web/FilterDef.java)表示一个过滤器的定义，用来说明`web.xml`部署描述符文件中定义的过滤器。[ApplicationFilterConfig](https://github.com/apache/tomcat/blob/TOMCAT_8_0_0/java/org/apache/catalina/core/ApplicationFilterConfig.java)类实现`javax.servlet.FilterConfig`接口，用于管理Web应用程序第一次启动时创建的所有的过滤器实例，其`getFilter()`方法会返回一个`javax.servlet.Filter`对象。[ApplicationFilterChain](https://github.com/apache/tomcat/blob/TOMCAT_8_0_0/java/org/apache/catalina/core/ApplicationFilterChain.java)类实现`javax.servlet.FilterChain`接口，`StandardWrapperValve`类的`invoke()`方法会创建`ApplicationFilterChain`类的一个实例，并调用其`doFilter()`方法。`ApplicationFilterChain`类的`doFilter()`方法会调用过滤器链中第一个过滤器的`doFilter`方法。如果某个过滤器时过滤器链中的最后一个过滤器，则会调用被请求的servlet类的`service()`方法。

[Host](https://github.com/apache/tomcat/blob/TOMCAT_8_0_0/java/org/apache/catalina/Host.java)接口表示一个虚拟主机，其实现类是[StandardHost](https://github.com/apache/tomcat/blob/TOMCAT_8_0_0/java/org/apache/catalina/core/StandardHost.java)。[Engine](https://github.com/apache/tomcat/blob/TOMCAT_8_0_0/java/org/apache/catalina/Engine.java)表示servlet引擎，其实现类是[StandardEngine](https://github.com/apache/tomcat/blob/TOMCAT_8_0_0/java/org/apache/catalina/core/StandardEngine.java)。`Host`的父容器只能是`Engine`，而`Engine`是顶级容器，不再有父容器，且子容器只能是`Host`。[Server](https://github.com/apache/tomcat/blob/TOMCAT_8_0_0/java/org/apache/catalina/Server.java)表示整个servlet引擎，其实现类是[StandardServer](https://github.com/apache/tomcat/blob/TOMCAT_8_0_0/java/org/apache/catalina/core/StandardServer.java)。`Server`几个重要方法是`initialize()`、`start()`、`stop()`、`await()`、`addService()`。`initialize()`方法用来初始化通过`addService()`方法添加到`Server`中的服务组件。`await()`方法则是创建一个`ServerSocket`，等待客户端连接并发送关闭指令，如果收到关闭指令，则调用`stop()`方法关闭`Server`，即整个servlet引擎。[Service](https://github.com/apache/tomcat/blob/TOMCAT_8_0_0/java/org/apache/catalina/Service.java)接口表示一个服务，其实现类是[StandardService](https://github.com/apache/tomcat/blob/TOMCAT_8_0_0/java/org/apache/catalina/core/StandardService.java)类。`Service`通过`setContainer()`方法关联一个容器，通过`addConnector()`方法，关联多个连接器。`StandardService`内部会将其容器通过`Connector.setContainer`方法设置`Service`的容器给`Connector`。

Tomcat的总体架构图如下：

![Tomcat架构](https://www.ibm.com/developerworks/cn/java/j-lo-tomcat1/image001.gif "Tomcat架构")

当一个处理请求时，连接器`Connector`创建`Request`和`Response`对象，通过[Mapper](https://github.com/apache/tomcat/blob/TOMCAT_8_0_0/java/org/apache/catalina/mapper/Mapper.java)类来选择相应的`Context`和`Wrapper`来处理请求。

Tomcat使用[Digester](https://commons.apache.org/proper/commons-digester/)库来解析`%CATALINA_HOME%/conf/server.xml`文件和web应用的`web.xml`文件。[ContextConfig](https://github.com/apache/tomcat/blob/TOMCAT_8_0_0/java/org/apache/catalina/startup/ContextConfig.java)类的实例负责并解析读取web应用的`web.xml`文件，为每个servlet元素创建一个`StandardWrapper`实例。`ContextConfig`实现了`LifecycleListener`接口，被添加到`StandardContext`作为其事件监听器。每次`StandardContext`实例触发事件时，会调用`ContextConfig`实例的`lifecycleEvent()`方法，该方法会调用`ContextConfig.start()`方法来解析`web.xml`文件。如果文件解析成功，调用`Context.setConfigured(true)`方法，使得`Context`可以正常启动，否则启动失败。

Tomcat的启动类是[Bootstrap](https://github.com/apache/tomcat/blob/TOMCAT_8_0_0/java/org/apache/catalina/startup/Bootstrap.java)，其`start`方法首先判断是否存在[Catalina](https://github.com/apache/tomcat/blob/TOMCAT_8_0_0/java/org/apache/catalina/startup/Catalina.java)实例，如果存在则初始化一个`Catalina`实例，然后调用`Catalina.start()`解析`%CATALINA_HOME%/conf/server.xml`文件和初始化`Server`。`Catalina`会调用`Runtine.addShutdownHook()`方法来注册一个`CatalinaShutdownHook`类型的关闭钩子，用于安全的释放资源。

`StandardHost`实例使用`HostConfig`对象作为一个生命周期监听器，当`StandardHost`对象启动时，它的`start()`方法会触发一个`START`事件。为了响应`START`事件，[HostConfig](https://github.com/apache/tomcat/blob/TOMCAT_8_0_0/java/org/apache/catalina/startup/HostConfig.java)中的lifecycleEvent方法和`HostConfig`中的事件处理程序调用`start()`方法。`HostConfig`类的`deployApps()`会完成`Host`下各个`Context`的部署工作。

参考：[Tomcat 系统架构与设计模式，第 1 部分 工作原理](https://www.ibm.com/developerworks/cn/java/j-lo-tomcat1/index.html)，[Tomcat 系统架构与设计模式，第 2 部分 设计模式分析](https://www.ibm.com/developerworks/cn/java/j-lo-tomcat2/)，[深入剖析Tomcat](https://book.douban.com/subject/10426640/)，[Tomcat源码debug环境](https://www.jianshu.com/p/d05ef74694f7)

# 六、JVM

#### 1.类加载机制和双亲委派模型

类从被加载到虚拟机内存到从内存中卸载，它的整个生命周期包括**加载**、**验证**、**准备**、**解析**、**初始化**、**使用**、**卸载**七个阶段，其中类加载过程包括**加载**、**验证**、**准备**、**解析**、**初始化**五个阶段。五个阶段中只有解析阶段是不确定的，因为它可以发生在准备阶段后，也可以发生在初始化阶段后。

* 加载

加载是类加载的第一个阶段，在该阶段，虚拟机需要完成三件事，1、通过一个类的全限定名来获取其定义的二进制字节流；2、将这个字节流所代表的静态存储结构转化为方法区的运行时数据结构；3、在Java堆中生成一个代表这个类的java.lang.Class对象，作为对方法区中这些数据的访问入口。加载阶段是可控性最强的阶段，开发人员可自定义类加载器实现类加载过程，也可以使用系统提供的类加载器。

* 验证

验证的目的是为了确保Class文件中的字节流包含的信息符合当前虚拟机规范要求，而且不会危害虚拟机自身的安全。验证阶段大致包括：**文件格式验证**、**元数据验证**、**字节码验证**、**符号应用验证**。为了提高虚拟机性能，在确保所有的类是安全的情况下，可以使用`-Xverify:none`来关闭验证功能。

* 准备

准备阶段是正式为类变量分配内存并设置类变量初始值的阶段，这些内存都将在方法区中分配。为类变量分配内存时，会将内存清零，所以，对于`public static int value = 3`中的变量`value`，在准备阶段的值是0。如果类字段的字段属性表中存在ConstantValue属性，即同时被final和static修饰（对于同时被static和final修饰的常量，必须在声明的时候就为其显式地赋值，否则编译时不通过），那么在准备阶段变量value就会被初始化为ConstValue属性所指定的值。解析阶段包括四种类型的解析，分别是**类或接口解析**、**字段解析**、**类方法解析**、**接口方法解析**。

* 解析

解析阶段是虚拟机将常量池中的符号引用转化为直接引用的过程。所谓符号引用，是指以一组符号来描述所引用的目标，符号可以是任何形式的字面量，只要使用时能无歧义地定位到目标即可，符号引用与虚拟机实现的内存布局无关，引用的目标并不一定已经加载到了内存中。而直接引用是指直接引用可以是直接指向目标的指针、相对偏移量或是一个能间接定位到目标的句柄，直接引用是与虚拟机实现的内存布局相关的，同一个符号引用在不同虚拟机实例上翻译出来的直接引用一般不会相同，如果有了直接引用，那说明引用的目标必定已经存在于内存之中了。

* 初始化

初始化阶段是执行类构造器`<clinit>()`方法的过程。`<clinit>（）`方法是由编译器自动收集类中的所有类变量的赋值动作和静态语句块中的语句合并产生的，编译器收集的顺序是由语句在源文件中出现的顺序所决定的，静态语句块中只能访问到定义在静态语句块之前的变量，定义在它之后的变量，在前面的静态语句中可以赋值，但是不能访问。



---



类加载器包括：

* 启动（Bootstrap）类加载器

负责将 Java_Home/lib下面的类库加载到内存中（比如rt.jar）。由于引导类加载器涉及到虚拟机本地实现细节，开发者无法直接获取到启动类加载器的引用，所以不允许直接通过引用进行操作。该加载器由C++代码实现。

* 标准扩展（Extension）类加载器

是由 Oracle 的 ExtClassLoader（sun.misc.Launcher$ExtClassLoader）实现的。它负责将Java_Home /lib/ext或者由系统变量 java.ext.dir指定位置中的类库加载到内存中。开发者可以直接使用标准扩展类加载器

* 应用程序（Application）类加载器

是由 Oracle 的 AppClassLoader（sun.misc.Launcher$AppClassLoader）实现的。它负责将系统类路径（CLASSPATH）中指定的类库加载到内存中。开发者可以直接使用系统类加载器。由于这个类加载器是ClassLoader中的getSystemClassLoader()方法的返回值，因此一般称为**系统（System）加载器**。

* 自定义类加载器

开发人员自定义的类加载器。



这些类加载器组成一个层级关系，称为**双亲委派模型**，将类加载器的职责分开。而且这种层级关系一般通过组合关系来实现，而不是通过继承。层级关系如下图：

![双亲委派模型](https://upload-images.jianshu.io/upload_images/4491294-8edc15f60a58bd0b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/468/format/webp "双亲委派模型")

**双亲委派模型**的过程是首先将加载任务委托给父类加载器，依次递归，如果父类加载器可以完成类加载任务，就成功返回；只有父类加载器无法完成此加载任务时，才自己去加载。使用双亲委派模型的好处在于Java类随着它的类加载器一起具备了一种带有优先级的层次关系。**双亲委派模型**的实现代码是`java.lang.ClassLoader`的`loadClass()`方法，如下：

```java
protected synchronized Class<?> loadClass(String name,boolean resolve)throws ClassNotFoundException{
    //check the class has been loaded or not
    Class c = findLoadedClass(name);
    if(c == null){
        try{
            if(parent != null){
                c = parent.loadClass(name,false);
            }else{
                c = findBootstrapClassOrNull(name);
            }
        }catch(ClassNotFoundException e){
            //if throws the exception ,the father can not complete the load
        }
        if(c == null){
            c = findClass(name);
        }
    }
    if(resolve){
        resolveClass(c);
    }
    return c;
}
```

自定义类加载器时只需要重写`findClass()`方法即可。

线程上下文类加载器（Thread Context Class Loader）可以通过`java.lang.Thread`类的`setContextClassLoader()`方法进行设置，如果创建线程时还未设置，它将会从父线程中继承一个；如果在应用程序的全局范围内都没有设置过，那么这个类加载器默认就是应用程序类加载器。这种行为实际上是打破了**双亲委派模型**的层次结构。

参考：[深入理解Java虚拟机](https://book.douban.com/subject/6522893/)，[【深入Java虚拟机】之四：类加载机制](https://blog.csdn.net/ns_code/article/details/17881581)，[【深入理解JVM】：类加载器与双亲委派模型](https://blog.csdn.net/u011080472/article/details/51332866)

#### 2.Java内存模型和运行时数据区

Java内存模型（Java Memory Model，简称JMM）定义了线程和主内存之间的抽象关系：线程之间的共享变量存储在主内存（main memory）中，每个线程都有一个私有的本地内存或工作内存（local memory），本地内存中存储了该线程以读/写共享变量的副本。本地内存是JMM的一个抽象概念，并不真实存在，它涵盖了缓存，写缓冲区，寄存器以及其他的硬件和编译器优化。JMM如下图所示：

![Java内存模型](/images/jmm.jpg "Java内存模型")

运行时数据区包括：**程序计数器**、**方法区**、**堆**、**虚拟机栈**、**本地方法栈**。

* 程序计数器

程序计数器（Program Counter Register）是一块较小的内存空间，它的作用可以看做是当前线程所执行的字节码行号指示器。

* 方法区

方法区（Method Area）与Java堆一样，是各个线程共享的内存区域，它用于存储已被虚拟机加载的类信息、常量、静态变量、即时编译器编译后的代码等数据。方法区又称“永久代”(Permanent Generation) ，使用`-XX:MaxPermSize`调整最大值，当方法区无法满足内存分配需求时，将抛出OutOfMemoryError异常。**运行时常量池**（Runtime Constant Pool）是方法区的一部分。Class文件中除了有类的版本、字段、方法、接口等描述信息外，还有一项信息是常量池表（Constant Pool Table），用于存放编译期生成的各种字面量和符号引用，这部分内容将在类加载后进入方法区的运行时常量池中存放。

* 堆

Java堆是垃圾收集管理的主要战场。根据Java虚拟机规范的规定，Java堆可以处于物理上不连续的内存空间中，只要逻辑上是连续的即可，就像我们的磁盘空间一样。在实现时，既可以实现成固定大小的，也可以是可扩展的，不过当前主流的虚拟机都是按照可扩展来实现的。可 通过`-Xmx`和`-Xms`控制Heap大小，如果在堆中没有内存完成实例分配，并且堆也无法再扩展时，将会抛出OutOfMemoryError异常。
* 虚拟机栈

虚拟机栈描述的是Java方法执行的内存模型：每个方法被执行的时候都会同时创建一个栈帧 （Stack Frame）用于存储局部变量表、操作栈、动态链接、方法出口等信息。每一个方法被调用直至执行完成的过程，就对应着一个栈帧在虚拟机栈中从入栈到出栈的过程。如果线程请求的栈深度大于虚拟机所允许的深度，将抛出`StackOverflowError`异常；如果虚拟机栈可以动态扩展，当扩展时无法申请到足够的内存时会抛出OutOfMemoryError异常。可通过`-Xss`调整虚拟机栈大小。

* 本地方法栈

本地方法栈（Native Method Stacks）与虚拟机栈所发挥的作用非常类似，区别在于虚拟机栈为虚拟机执行Java方法服务，而本地方法栈则是为虚拟机使用到的Native方法服务。



**直接内存**不是虚拟机运行时数据区的一部分，在Java1.4引入的NIO中新增了`DirectByteBuffer`对象作为这块内存的引用。该部分的内存大小不受Java堆大小限制，而是受操作系统内存限制。

参考：[全面理解Java内存模型](https://blog.csdn.net/suifeng3051/article/details/52611310)，[JVM内存区域分析](http://sparkyuan.me/2016/04/22/JVM%E8%BF%90%E8%A1%8C%E6%97%B6%E6%95%B0%E6%8D%AE%E5%8C%BA%E5%9F%9F/)

#### 3.垃圾收集

判断对象是否存活的算法有两种，一种是**引用计数器算法**，另一种是**根搜索算法**。

引用计数器算法的基本思路是给对象添加一个引用计数器，每当有一个地方引用它时，计数器的值就加1；当引用失效时，计数器的值减1；任何时刻计数器为0的对象就是不可能再被使用。引用计数器算法实现简单、效率高（微软的COM即Componet Object Model就是使用该算法），但存在一个缺点，对于对象之间的相互循环引用的问题无法解决，所以JVM并未使用该算法。

根搜索算法（RC Roots Tracing）的基本思路就是通过一系列的名为“GC Roots”的对象作为起始点，从这些节点开始向下搜索，搜索所走过的路径称为引用链（Reference Chain），当一个对象到“GC Roots”没有任何引用链相连（用图论的话来说就是从“GC Roots”到这个对象不可达）时，则证明此对象是不可用的。在Java语言里，可作为“GC Roots”的对象包括：虚拟机栈（栈帧中的本地变量表）中的引用的对象、方法区中的类静态属性引用的对象、方法区中的常量引用对象、本地方法栈中JNI引用的对象。

在根搜索算法中不可达的对象不会立即清除，至少需要经过两次标记过程：如果对象在进行根搜索后发现没有与GC Roots相连接的引用链，那它将会被第一次标记并且进行一次筛选，筛选的条件是此对象是否有必要执行`finalize()`方法。当对象没有覆盖`finalize()`方法或者`finalize()`方法已经被虚拟机调用过，虚拟机将这两种情况都视为“没有必要执行”。如果这个对象被判断为有必要执行`finalize()`方法，那么这个对象将会被放置在一个名为`F-queue`的队列之中，并在稍后由虚拟机自动建立的、低优先级的`Finalizer`线程去执行（即调用对象的`finalize()`方法）。`finalize()`方法是对象逃脱死亡命运的最后一次机会，稍后GC将对`F-queue`中的对象进行第二次小规模的标记，如果对象在`finalize()`方法中重新与引用链上的任何一个对象建立关联，此时它将被移除“即将回收”的集合；如果对象此时并未与引用链上的任何对象建立关联，则此对象将会被回收。

内存泄漏是指无用对象（不再使用的对象）持续占有内存或无用对象的内存得不到及时释放，从而造成内存空间的浪费称为内存泄漏。长生命周期的对象持有短生命周期对象的引用就很可能发生内存泄漏，尽管短生命周期对象已经不再需要，但是**因为长生命周期持有它的引用而导致不能被回收，这就是Java中内存泄漏的发生场景**。

参考：[java7和java8的垃圾回收](https://blog.csdn.net/high2011/article/details/53138202)，[Java内存泄漏分析和解决](https://www.jianshu.com/p/54b5da7c6816)

#### 4.JVM关闭钩子

参考：[深入JVM关闭与关闭钩子](https://blog.csdn.net/dd864140130/article/details/49155179)

#### 5.Java Agent

[HotswapAgent](https://github.com/HotswapProjects/HotswapAgent)

参考：[JVM源码分析之javaagent原理完全解读](http://www.infoq.com/cn/articles/javaagent-illustrated)

#### 6.Hotswap

#### 7.调优方法@2018-08-13

参考：[Java性能优化权威指南](https://book.douban.com/subject/25828043/)，[Arthas](https://github.com/alibaba/arthas)

# 七、mysql 

#### 1.事务隔离级别

#### 2.三范式@2018-08-14

#### 3.mysql有哪些索引，区别是什么

#### 4.悲观锁和乐观锁@2018-08-15

#### 5.分布式原理

#### 6.mysql有哪些锁@2018-08-16

参考：[Understanding Innodb locks and deadlocks](https://www.percona.com/live/mysql-conference-2015/sites/default/files/slides/understandinginnodblocksanddeadlocks.pdf)

#### 7.mysql实现B+树的原理

#### 8.数据一致性@2018-08-17

#### 9.mysql的innodb、myisam和memory引擎的优缺点

#### 10.什么是nosql，与sql的优缺点@2018-08-18

#### 11.TPS压测

#### 12.Page Split@2018-08-19

#### 13.SQL优化



# 八、redis 

#### 1.哨兵模式和主从数据同步@2018-08-20

参考：[Redis 深度历险：核心原理与应用实践](https://juejin.im/book/5afc2e5f6fb9a07a9b362527/section)，[[深入学习Redis（1）：Redis内存模型](https://www.cnblogs.com/kismetv/p/8654978.html)](https://www.cnblogs.com/kismetv/p/8654978.html)

#### 2.集群原理

#### 3.redis与memcached区别@2018-08-21

#### 4.过期策略

#### 5.如何保证一致性@2018-08-22

#### 6.缓存热点与击穿

#### 7.redis内存淘汰机制@2018-08-23



# 九、mybatis 

#### 1.mybatis一级缓存和二级缓存

#### 2.mybatis与hibernate优缺点@2018-08-24

#### 3.mybatis工作流程



# 十、Spring

#### 1.IoC

#### 2.AOP@2018-08-29

#### 3.Spring Boot的条件’加载’

#### 4.Spring容器启动过程@2018-08-30

#### 5.Spring bean生命周期

#### 6.Spring MVC生命周期@2018-08-31

#### 7.Spring声明性事务管理实现方式



# 十一、MessageQueue 

#### 1.重复消费问题@2018-08-25 

#### 2.顺序问题



# 十二、设计模式 

#### 1.有哪些原则@2018-08-26 

参考：[Head First 设计模式（中文版）](https://book.douban.com/subject/2243615/)

#### 2.常用设计模式

参考：[Software design pattern](https://en.wikipedia.org/wiki/Software_design_pattern)



# 十三、网络 

#### 1.TCP和UDP的区别@2018-08-27 

参考：https://my.oschina.net/fzyz999/blog/704510

#### 2.TCP三次握手 

#### 3.IO多路复用@2018-08-28 

#### 4.NIO 



# 十四、算法和数据结构 

[Leetcode](https://leetcode.com/)有各种经典算法的代码，也是算法题库。

#### 1.排序算法@2018-09-01 

#### 2.查找算法

参考：[Data Structure Visualizations](https://www.cs.usfca.edu/~galles/visualization/Algorithms.html)

#### 3.LRU@2018-09-02

#### 4.一致性哈希算法

#### 5.GeoHash

参考：[JAVA实现空间索引编码（GeoHash）](https://blog.csdn.net/xiaojimanman/article/details/50358506)

#### 6.选举算法

#### 7.流量控制算法



# 十五、架构

#### 1.分布式锁的实现方式

#### 2.流量控制@2018-09-03

#### 3.全文搜索引擎@2018-09-04



# 十六、操作系统

#### 1.内存映射

参考：[认真分析mmap：是什么 为什么 怎么用](https://www.cnblogs.com/huxiao-tee/p/4660352.html)