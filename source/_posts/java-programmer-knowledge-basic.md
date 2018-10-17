---
title: Java后端开发 - 基础
date: 2018-10-13 08:14:22
tags: java
---

# Throwable，Error，Exception，RuntimeException

- `Error`和`Exception`都是`Throwable`接口的子类
- Java分为“受检查异常”（或`Checked Exception`，这类异常编译器会检查，如果对这类异常既没有`try...catch`也没`throws`时编译将不通过）和“未检查异常”（或`Unchecked Exception`，这类异常编译器不检查），`RuntimeException`及其子类都是“未检查异常”，如`NullPointException`，其他为“受检查异常”，如`IOException`。
- `Error`一般是指与虚拟机相关的错误，程序一般无法自身恢复，比如各种内存溢出（如`OutOfMemoryError`，`StackOverflowError`，注意：断言失败是抛出`AssertionError`）；`Exception`则表示程序级别的异常，程序可以处理并恢复。



参考：[谈一谈Java中的Error和Exception](https://blog.csdn.net/goodlixueyong/article/details/47122487)

# 对象克隆 

`Object.clone()`方法是`native`方法，如果类没有实现`Cloneable`接口时将抛出`CloneNotSupportedException`，即便是子类重写了`clone()`，`clone()`方法签名如下：

```java
protected native Object clone() throws CloneNotSupportedException;
```

`Object.clone()`方法由C++来实现，实现原理是先调用`CollectedHeap::array_allocate()`或者`CollectedHeap::obj_allocate()`分配与原对象相同大小的新内存，然后调用`Copy::conjoint_jlongs_atomic()`将原对象的内存数据复制到新内存当中（所以，该复制方法是`浅复制`）。



调用`clone()`来克隆对象时需要调用到`Object.clone()`，所以子类重写`clone()`时，需要调用`super.clone()`。



对象克隆分为`浅克隆`（或`浅复制`）和`深克隆`（或`深复制`），他们的区别就是会不会为引用类型成员变量调用`clone()`来生成新对象。



由于`Object.clone()`方法是复制内存数据来克隆对象的，因此，同样可以使用Java序列化机制来复制内存数据达到克隆的效果，而且是`深克隆`。



参考：[详解Java中的clone方法 -- 原型模式](https://blog.csdn.net/zhangjg_blog/article/details/18369201)，[Java对象克隆（Clone）及Cloneable接口、Serializable接口的深入探讨](https://blog.csdn.net/kenthong/article/details/5758884)

# 强引用与弱引用 

在`java.lang.ref`包中存在三种类型的引用类，分别是`SoftReference`（软引用），`WeakReference`（弱引用），`PhantomReference`（虚引用），Java的引用类型除了这三个，还有一个就是强引用（即Java程序中使用等号赋值的引用），它们的级别由高到低分别为强引用、软引用、弱引用、虚引用，引用类型可以控制JVM回收对象的时机。



如果一个对象存在强引用，就不会被回收；如果一个对象只存在软引用，当JVM内存不足时，就会被回收；如果一个对象只存在弱引用，当执行垃圾收集时就会被回收；如果一个对象只存在虚引用，任何时候都可能被回收。



| 引用类型 | 回收时间   | 用途           | 生存时间          |
| -------- | ---------- | -------------- | ----------------- |
| 强引用   | 从来不会   | 对象的一般状态 | JVM停止运行时终止 |
| 软引用   | 内存不足时 | 对象缓存       | 内存不足时终止    |
| 弱引用   | 垃圾收集时 | 对象缓存       | 垃圾收集时终止    |
| 虚引用   | Unknown    | Unknown        | Unknown           |



参考：[Java四种引用---强、软、弱、虚的知识点总结](https://blog.csdn.net/l540675759/article/details/73733763)

# 断言 

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



# 静态初始化和实例初始化 

- 静态初始化

  静态初始化是指`<cinit>`方法的调用，该方法是编译器自动生成，代码是静态语句块和静态变量（或者类变量）。JVM会保证在调用某类的`<cinit>`方法时先调用其父类的`<cinit>`方法。`<cinit>`方法只会被JVM调用一次。

- 实例初始化

  实例创建的方式有：`new`、`Class.newInstance()`、`Constructor.newInstance()`、`Object.clone()`、`ObjectInputStream.readObject()`等，从Java虚拟机层面来说是两种：`new`、`invokevirtual`。

  实例初始化是指`<init>`方法的调用，该方法由构造函数、实例代码块和实力变量组成，`<init>`方法的第一行总是会调用父类的`<init>`方法。



参考：[深入理解Java对象的创建过程：类的初始化与实例化](https://blog.csdn.net/justloveyou_/article/details/72466416)，[JVM类生命周期概述：加载时机与加载过程](https://blog.csdn.net/justloveyou_/article/details/72466105)

# Java8新特性

- Lambda表达式和函数式接口

  Lambda表达式（也称为闭包）可以将函数作为参数传递，语法参考[Lambda Expression](https://docs.oracle.com/javase/tutorial/java/javaOO/lambdaexpressions.html#syntax)；函数式接口指只有一个抽象方法的接口，可以使用`@FunctionalInterface`标记函数式接口，让编译器检查接口是否为有效的函数式接口。

  常用的函数接口有：[java.lang.Runnable](https://docs.oracle.com/javase/8/docs/api/java/lang/Runnable.html)，[java.util.concurrent.Callable](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/Callable.html)，[java.util.function.Function](https://docs.oracle.com/javase/8/docs/api/java/util/function/Function.html)，[java.util.function.Predicate](https://docs.oracle.com/javase/8/docs/api/java/util/function/Predicate.html)，[java.util.function.Supplier](https://docs.oracle.com/javase/8/docs/api/java/util/function/Supplier.html)，[java.util.function.Consumer](https://docs.oracle.com/javase/8/docs/api/java/util/function/Consumer.html)

- 接口默认方法和静态方法

  接口的默认方法使用`default`修饰并且包括方法体，它不需要接口实现类实现，但可以重写；静态方法的语法同一般类的静态方法。

- 方法引用

  方法引用必须与Lambda表达式结合使用，不能直接将方法引用赋值给一个变量。方法引用的类型有：

| 类型                   | 示例                                 |
| ---------------------- | ------------------------------------ |
| 引用静态方法           | ContainingClasss::staticMethodName   |
| 引用实例方法           | containingObject::instanceMethodName |
| 引用任意类型的实例方法 | ContainingType::methodName           |
| 引用构造器方法         | ClassName::new                       |

- 重复注解

  可使用`@Repeatable`添加多个相同的注解

- 更好的类型推断

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

- 拓宽注解应用场景

  Java8添加了`ElementType.TYPE_USER`和`ElementType.TYPE_PARAMETER`用来描述注解的使用场景。

- 参数名称

  当编译时加上`-parameters`参数时，编译器不会擦除方法的参数名称，此时可以使用`Parameter.getName()`获取参数名称。

- Optional

  为了解决大量的`NullPointerException`添加了一个容器类型`Optional`，其常用方法如下：

| 方法名      | 说明                                                         |
| ----------- | ------------------------------------------------------------ |
| isPresent() | 如果`Optional`实例持有一个非空值，该方法返回`true`，否则返回`false` |
| orElseGet() | 如果`Optional`实例持有一个空值，则返回lambda表达式的值，如：`name.orElseGet(() -> "anonymous")` |
| map()       | map()可以将现有的`Optional`实例转换为新值，如：`name.map(n -> 'Hi ' + n + "!").orElse("Hi Stranger!")` |
| orElse()    | `orElse()`与`orElseGet()`类似，区别是`orElse()`参数为一个类型T，`orElseGet()`为一个lambda表达式。 |

- Streams

  Java8新增了Stream API，可以方便灵活处理容器的`filter`、`map`、`reduce`、`sort`等串行和并行的聚合操作，Stream由流管道组成，一个流管道由源数据（如数组和集合）、中间操作（产生新的stream对象）、终止操作（产生一个结果）详细参考[java.util.stream.Stream](https://docs.oracle.com/javase/8/docs/api/java/util/stream/Stream.html)。

- Date/Time API

  时间支持以纳秒级表示。

- Nashorn Javascript引擎

  添加了`jjs`来运行javascript代码文件，也可以运行javascript字符串格式，如：

  ```java
  ScriptEngineManager manager = new ScriptEngineManager();
  ScriptEngine engine = manager.getEngineByName( "JavaScript" );
          
  System.out.println( engine.getClass().getName() );
  System.out.println( "Result:" + engine.eval( "function f() { return 1; }; f() + 1;" ) );
  ```

- Base64

  添加了`java.util.Base64`，支持`Base64`功能。

- 并行数组

  可支持并行数组处理，如`Arrays.parallelSetAll()`、`Arrays.parallelSort()`。

- 并发性

  改进了`java.util.concurrent.ConcurrentHashMap`，添加了`java.util.concurrent.locks.StampedLock`替代`java.util.concurrent.locks.ReadWriteLock`。

- JVM新特性

  使用Metaspace替代PermGen Space，

参考：[Java 8的新特性—终极版](https://www.jianshu.com/p/5b800057f2d8)

# Java序列化与反序列化

Java序列化允许将Java对象保存为一组字节，之后可以读取这组字节组建一个新的对象（该操作为反序列化），对象序列化不会关注类中的静态变量，被`transient`修饰的变量不会被序列化存储。在RMI和RPC中经常使用到Java序列化和反序列化。



只要类实现了`java.io.Serializable`接口时，其对象就可以序列化，需要用到的流对象是`java.io.ObjectInputStream`（反序列化）和`java.io.ObjectOutputStream`（序列化）。



虚拟机是否允许反序列化，不仅取决于类路径和功能代码是否一致，一个非常重要的一点是两个类的序列化 ID 是否一致（就是 `private static final long serialVersionUID`）。



参考：[深入分析Java的序列化与反序列化](http://www.hollischuang.com/archives/1140)，[java序列化和反序列化](https://www.jianshu.com/p/5a85011de960)

# 反射

反射机制是程序可以在运行时获取类型或者实例的字段和方法，然后进行操作。通过反射可以获取对象字段的值，也可以使用`sun.misc.Unsafe`来快速读取对象的字段值。

以反射调用`Object.hashCode`方法为例，叙述反射的原理，样例代码如下：

```java
Object o = new Object();
Class clazz = o.getClass();
Method hashCode = clazz.getMethod("hashCode", null);
System.out.println(hashCode.invoke(o, null));
```

要通过反射获取对象的方法或者字段，首先需要获取Class对象。在静态编码情况下，可以确定Class对象，比如样例代码第二行，可以直接写成`Class clazz = Object.class`，当在运行时情况下，可以通过调用对象的`getClass`方法获取对象的Class对象。`getClass`方法在`Object`类中声明，方法签名如下：

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

指向对象的指针称为`Ordinary Object Pointer`(OOP)，Java实例对象使用C++中的[oopDesc](https://github.com/dmlloyd/openjdk/blob/jdk7u/jdk7u/hotspot/src/share/vm/oops/oop.hpp)来表示，`oop`就是指向`oopDesc`类型的指针。在JVM中，Java对象的头部由下列两个字段组成：

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

`searchMethods`方法首先迭代`Method[]`方法表，如果期望的`Method`方法被找到，需要将`Method`方法复制一份，然后返回给样例代码中的`hashcode`变量。复制的实现方法`copy`定义在`Method.java`中。由此可见，每次通过调用`getMethod`方法返回的Method对象其实都是一个新的对象，如果调用频繁最好缓存起来。

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

`root`字段指向`ReflectionData`对象中方法表的某个`Method`实例（因为获得的`Method`对象是从`ReflectionData`复制而来，所以`root`保留了被复制的对象或者说原对象）。`Method.invoke`方法调用`methodAccessor`的`invoke`方法。如果`root`的`methodAccessor`存在，则赋值给`methodAccessor`这个属性，否则就创建一个。`MethodAccessor`本身就是一个接口，其主要有三种实现

- DelegatingMethodAccessorImpl
- NativeMethodAccessorImpl
- GeneratedMethodAccessorXXX

`Method`的`methodAccessor`的对象类型就是`DelegatingMethodAccessorImpl`，也就是某个`Method`的所有的`invoke`方法都会调用到这个`DelegatingMethodAccessorImpl.invoke`，正如其名一样的，是做代理的，也就是真正的实现可以是`NativeMethodAccessorImpl`和`GeneratedMethodAccessorXXX`两种。如果是`NativeMethodAccessorImpl`，顾名思义，由本地代码C++实现，而`GeneratedMethodAccessorXXX`是为每个需要反射调用的`Method`动态生成的类，后缀XXX是一个不断递增的数值。并且所有的方法反射都是先走`NativeMethodAccessorImpl`，默认调了15次之后，才生成一个`GeneratedMethodAccessorXXX`类，生成好之后就会走这个生成的类的`invoke`方法。`NativeMethodAccessorImpl`的`invoke`方法定义如下：

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

`generateMethod`会生成一个`GeneratedMethodAccessorXXX`实例，它的`invoke`方法就是调用样例代码中的`o.hashCode()`。通过字节码直接调用对象的方法，称为Java字节码拼接技术，用来在运行时生成Java类。[javassist](http://www.javassist.org/)和[cglib](https://github.com/cglib/cglib)都是基于字节码拼接技术实现，在`Spring`中就是使用`cglib`来动态的生成代理类，实现`AOP`功能。`GeneratedMethodAccessorXXX`的类加载器是一个`DelegatingClassLoader`类加载器，使用新的类加载器是为了性能考虑，在某些情况下可以卸载这些生成的类。`invoke0`方法是本地方法，由C实现，方法定义在[NativeAccessors.c](https://github.com/dmlloyd/openjdk/blob/jdk7u/jdk7u/jdk/src/share/native/sun/reflect/NativeAccessors.c)如下：

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

# Statement和PreparedStatement的区别，如何防止SQL注入

JDBC执行SQL语句可以使用三个类，分别是`Statement`、`PreparedStatement`、`CallableStatement`，描述如下表：

| 类名              | 作用                                                         |
| ----------------- | ------------------------------------------------------------ |
| Statement         | 通用查询                                                     |
| PreparedStatemnt  | 预编译语句查询，可以解决SQL注入问题，由于是预编译，所以可以减少数据库对SQL代码的解析操作，提高效率 |
| CallableStatement | 存储过程查询                                                 |

参考：[JDBC为什么要使用PreparedStatement而不是Statement](http://www.importnew.com/5006.html)

# Java命令

- java

  启动JVM运行Java字节码的工具；

- javac

  编译Java代码的工具，或者称为java编译器（javac就是java compiler的缩写）；

- jar

  操作jar文件的工具；

- jdb

  java调试器工具，作用和用法与gnu的gdb类似；

- javah

  开发JNI时使用的工具，它可以根据Java代码声明的`native`方法自动生成C的头文件；

- javap

  java字节码反编译工具；

- jps

  Java Virtual Machine Process Status Tool，用来查看java虚拟机进程状态；

- jjs

  运行javascript代码的工具；

- jdeps

  分析java依赖的工具；

- jinfo

  查看JVM配置信息的工具；

- jstack

  查看JVM线程栈工具；

- jmap

  导出JVM内存的工具；

- jhat

  分析jmap导出的文件的工具；

- jconsole

  查看虚拟机内存、线程等信息的工具；

- jvisualvm

  jconsole的增强工具；

- javadoc

  生成java api文档的工具；

- keytool

  操作RSA或者证书相关的工具；

- rmiregistry

  RMI相关工具。



参考：[JDK Tools and Utilities](https://docs.oracle.com/javase/7/docs/technotes/tools/index.html)

# NoClassDefFoundError和ClassNotFoundException

`NoClassDefFoundError`和`ClassNotFoundException`都是由于在`CLASSPATH`下找不到对应的类而引起的，通常是缺少对应的jar包或者jar包冲突导致，具体如下：

- `ClassNotFoundException`是在代码中显示调用加载类的方法导致，如`Class.forName`、`ClassLoader.findSystemClass()`和`ClassLoader.loadClass()`等；

- `NoClassDefFoundError`是JVM链接时找不到类时抛出的错误，如`new`一个实例或者调用静态方法等。



  参考：[NoClassDefFoundError和ClassNotFoundException的不同](https://www.jianshu.com/p/93d0db07d2e3)

# 方法动态绑定原理

在Java中， `final`，`static`，`private`以及构造方法与类的绑定关系是在编译期确定，称之为“前期绑定”或者“静态绑定”，对于实例“静态绑定”的方法，采用`invokespecial`指令调用。对于其他实例方法，则需要在运行时根据对象类型再行决议，我们称之为“后期绑定”或“动态绑定”，采用`invokevirtual`指令调用方法。

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

当第一次加载类时，JVM会调用[classFileParser.cpp::parseClassFile()](https://github.com/dmlloyd/openjdk/blob/jdk7u/jdk7u/hotspot/src/share/vm/classfile/classFileParser.cpp)函数对类的字节码解析，调用`parseMethods()`函数解析类的所有方法，之后再调用`klassVtable::compute_vtable_size_and_num_mirandas()`函数计算当前类的`vtable`大小。计算`vtable`大小的过程是首先获取父类的`vtable`大小，再循环当前类的方法，调用`needs_new_vtable_entry`方法判断方法是否需要加入到`vtable`（如果方法被声明为`public`或者`protected`且不是`static`或者`final`时，称此方法为虚方法，此时该方法返回`true`），如果返回true，则`vtable`大小加1。当类解析完成后，就需要调用[InstanceKlass::allocate_instance_klass()](https://github.com/dmlloyd/openjdk/blob/jdk7u/jdk7u/hotspot/src/share/vm/oops/instanceKlassKlass.cpp)方法分配内存，存储类信息，这些信息就包括`vtable`大小。当类信息创建完成后就可以准备方法调用了。在执行真正的方法调用前，需要调用[instanceKlass::link_class](https://github.com/dmlloyd/openjdk/blob/jdk7u/jdk7u/hotspot/src/share/vm/oops/instanceKlass.cpp)进行方法链接，此时将会初始化虚方法表。初始化虚方法表的方法在`instanceKlass::link_class_impl`中执行，代码如下：

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

# 异常处理原理

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

- from 抛出异常的开始行
- to 抛出异常的结束行
- target 需要跳转的行
- type 异常类型

异常表内容的第一行表示如果在第0行（此处的行指程序地址，比如第3行是指程序地址为3的指令，即`dup`）和第8行抛出`java.lang.Exception`异常时，跳转到第8行执行代码。如果在第0行和第17号抛出代码时，跳转到第28行执行代码。如果当前方法未找到合适的异常处理器时，当前方法弹栈，交给栈顶方法处理。如果线程栈方法全部弹出也未找到异常处理器，则线程结束。

参考：[The secret life of Java exceptions and JVM internals: Level up your Java knowledge](https://blog.takipi.com/the-surprising-truth-of-java-exceptions-what-is-really-going-on-under-the-hood/)

# 泛型原理

泛型可以对类和接口的类型参数化，使用比较多的是容器类，比如`List<String>`就是将`List`的元素参数化为`String`。使用泛型的作用有：

- 编译器强类型检查
- 去掉强制类型转换的代码
- 支持泛型算法的实现

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

从反编译代码中可以看到，字段`t`的类型是`Object`，而代码`Generic<String> g = new Generic<String>();`的泛型信息被擦除，变为`Generic g = new Generic();`，此时已经看不到泛型信息。对于类型元信息，即类、字段和方法（包括参数和返回值）不会擦除泛型信息，因此可以通过反射来获取泛型信息。上述代码`Generic<T>`变编译后，泛型信息是`Generic<T extends java.lang.Object>`，而变量`g`由于泛型类型被擦除，无法通过反射获取其泛型类型。可以通过`new Generic<String>(){}`构造一个子类，此时可以通过反射获取泛型信息。这种方式使用的比较多的是在json解析中，当要解析一串json文本为带有泛型的类型时使用如fastjson的用法`JSONObject.parseObject(json, new TypeReference<List<Person>>(){})`。

参考：[java泛型（二）、泛型的内部原理：类型擦除以及类型擦除带来的问题](https://blog.csdn.net/lonelyroamer/article/details/7868820)，[Generics](https://docs.oracle.com/javase/tutorial/java/generics/index.html)

# 线程栈

参考：[JVM Internals](http://blog.jamesdbloom.com/)