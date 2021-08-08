---
title: Java后端开发 - JVM
date: 2018-10-13 08:15:16
tags: java
toc: true
---

# 类加载机制

类从被加载到虚拟机内存到从内存中卸载，它的整个生命周期包括**加载**、**验证**、**准备**、**解析**、**初始化**、**使用**、**卸载**七个阶段，其中类加载过程包括**加载**、**验证**、**准备**、**解析**、**初始化**五个阶段。五个阶段中只有解析阶段是不确定的，因为它可以发生在准备阶段后，也可以发生在初始化阶段后。

- 加载

加载是类加载的第一个阶段，在该阶段，虚拟机需要完成三件事，1、通过一个类的全限定名来获取其定义的二进制字节流；2、将这个字节流所代表的静态存储结构转化为方法区的运行时数据结构；3、在Java堆中生成一个代表这个类的java.lang.Class对象，作为对方法区中这些数据的访问入口。加载阶段是可控性最强的阶段，开发人员可自定义类加载器实现类加载过程，也可以使用系统提供的类加载器。

- 验证

验证的目的是为了确保Class文件中的字节流包含的信息符合当前虚拟机规范要求，而且不会危害虚拟机自身的安全。验证阶段大致包括：**文件格式验证**、**元数据验证**、**字节码验证**、**符号应用验证**。为了提高虚拟机性能，在确保所有的类是安全的情况下，可以使用`-Xverify:none`来关闭验证功能。

- 准备

准备阶段是正式为类变量分配内存并设置类变量初始值的阶段，这些内存都将在方法区中分配。为类变量分配内存时，会将内存清零，所以，对于`public static int value = 3`中的变量`value`，在准备阶段的值是0。如果类字段的字段属性表中存在ConstantValue属性，即同时被final和static修饰（对于同时被static和final修饰的常量，必须在声明的时候就为其显式地赋值，否则编译时不通过），那么在准备阶段变量value就会被初始化为ConstValue属性所指定的值。

- 解析

解析阶段是虚拟机将常量池中的符号引用转化为直接引用的过程，它包括四种类型的解析，分别是**类或接口解析**、**字段解析**、**类方法解析**、**接口方法解析**。所谓符号引用，是指以一组符号来描述所引用的目标，符号可以是任何形式的字面量，只要使用时能无歧义地定位到目标即可，符号引用与虚拟机实现的内存布局无关，引用的目标并不一定已经加载到了内存中。而直接引用是直接指向目标的指针、相对偏移量或是一个能间接定位到目标的句柄，直接引用是与虚拟机实现的内存布局相关的，同一个符号引用在不同虚拟机实例上翻译出来的直接引用一般不会相同，如果有了直接引用，那说明引用的目标必定已经存在于内存之中了。

- 初始化

初始化阶段是执行类构造器`<clinit>()`方法的过程。`<clinit>（）`方法是由编译器自动收集类中的所有类变量的赋值动作和静态语句块中的语句合并产生的，编译器收集的顺序是由语句在源代码中出现的顺序所决定的，静态语句块中只能访问到定义在静态语句块之前的变量，定义在它之后的变量，在前面的静态语句中可以赋值，但是不能访问。

# 双亲委派模型

类加载器包括：

- 启动（Bootstrap）类加载器

负责将 JAVA_HOME/lib下面的类库加载到内存中（比如rt.jar）。由于引导类加载器涉及到虚拟机本地实现细节，开发者无法直接获取到启动类加载器的引用，所以不允许直接通过引用进行操作。该加载器由C++代码实现。

- 标准扩展（Extension）类加载器

是由 Oracle 的 ExtClassLoader（sun.misc.Launcher$ExtClassLoader）实现的。它负责将JAVA_HOME/lib/ext或者由系统变量 java.ext.dir指定位置中的类库加载到内存中。开发者可以直接使用标准扩展类加载器

- 应用程序（Application）类加载器

是由 Oracle 的 AppClassLoader（sun.misc.Launcher$AppClassLoader）实现的。它负责将系统类路径（CLASSPATH）中指定的类库加载到内存中。开发者可以直接使用系统类加载器。由于这个类加载器是ClassLoader中的getSystemClassLoader()方法的返回值，因此一般称为**系统（System）加载器**。

- 自定义类加载器

开发人员自定义的类加载器。



这些类加载器组成一个层级关系，称为**双亲委派模型**，将类加载器的职责分开。而且这种层级关系一般通过组合关系来实现，而不是通过继承。

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

# Java内存模型和运行时数据区

Java内存模型（Java Memory Model，简称JMM）定义了线程和主内存之间的抽象关系：线程之间的共享变量存储在主内存（main memory）中，每个线程都有一个私有的本地内存或工作内存（local memory），本地内存中存储了该线程以读/写共享变量的副本。本地内存是JMM的一个抽象概念，它涵盖了缓存，写缓冲区，寄存器以及其他的硬件和编译器优化的内存重排序。

运行时数据区包括：**程序计数器**、**方法区**、**堆**、**虚拟机栈**、**本地方法栈**。

- 程序计数器

程序计数器（Program Counter Register）是一块较小的内存空间，它的作用可以看做是当前线程所执行的字节码行号指示器。

- 方法区

方法区（Method Area）与Java堆一样，是各个线程共享的内存区域，它用于存储已被虚拟机加载的类信息、常量、静态变量、即时编译器编译后的代码等数据。方法区又称“永久代”(Permanent Generation) ，使用`-XX:MaxPermSize`调整最大值，当方法区无法满足内存分配需求时，将抛出OutOfMemoryError异常。**运行时常量池**（Runtime Constant Pool）是方法区的一部分。Class文件中除了有类的版本、字段、方法、接口等描述信息外，还有一项信息是常量池表（Constant Pool Table），用于存放编译期生成的各种字面量和符号引用，这部分内容将在类加载后存放在方法区的运行时常量池中。在Java8中，方法区已被移除，引进了Metaspace（本地堆内存）来存放类信息，字符串常量池移至堆区。

- 堆

Java堆是垃圾收集管理的主要战场。根据Java虚拟机规范的规定，Java堆可以处于物理上不连续的内存空间中，只要逻辑上是连续的即可，就像我们的磁盘空间一样。在实现时，既可以实现成固定大小的，也可以是可扩展的，不过当前主流的虚拟机都是按照可扩展来实现的。可通过`-Xmx`和`-Xms`控制Heap大小，如果在堆中没有内存完成实例分配，并且堆也无法再扩展时，将会抛出OutOfMemoryError异常。

- 虚拟机栈

虚拟机栈描述的是Java方法执行的内存模型：每个方法被执行的时候都会同时创建一个栈帧 （Stack Frame）用于存储局部变量表、操作栈、动态链接、方法出口等信息。每一个方法被调用直至执行完成的过程，就对应着一个栈帧在虚拟机栈中从入栈到出栈的过程。如果线程请求的栈深度大于虚拟机所允许的深度，将抛出`StackOverflowError`异常；如果虚拟机栈可以动态扩展，当扩展时无法申请到足够的内存时会抛出OutOfMemoryError异常。可通过`-Xss`调整虚拟机栈大小。

- 本地方法栈

本地方法栈（Native Method Stacks）与虚拟机栈所发挥的作用非常类似，区别在于虚拟机栈为虚拟机执行Java方法服务，而本地方法栈则是为虚拟机使用到的Native方法服务。



**直接内存**不是虚拟机运行时数据区的一部分，在Java1.4引入的NIO中新增了`DirectByteBuffer`对象作为这块内存的引用。该部分的内存大小不受Java堆大小限制，而是受操作系统内存限制。

# 垃圾收集

## 收集算法

判断对象是否存活的算法有两种，一种是**引用计数器算法**，另一种是**根搜索算法**。

引用计数器算法的基本思路是给对象添加一个引用计数器，每当有一个地方引用它时，计数器的值就加1；当引用失效时，计数器的值减1；任何时刻计数器为0的对象就是不可能再被使用。引用计数器算法实现简单、效率高（微软的COM即Componet Object Model就是使用该算法），但存在一个缺点，对于对象之间的相互循环引用的问题无法解决，所以JVM并未使用该算法。

根搜索算法（RC Roots Tracing）的基本思路就是通过一系列的名为“GC Roots”的对象作为起始点，从这些节点开始向下搜索，搜索所走过的路径称为引用链（Reference Chain），当一个对象到“GC Roots”没有任何引用链相连（用图论的话来说就是从“GC Roots”到这个对象不可达）时，则证明此对象是不可用的。在Java语言里，可作为“GC Roots”的对象包括：虚拟机栈（栈帧中的本地变量表）中的引用的对象、方法区中的类静态属性引用的对象、方法区中的常量引用对象、本地方法栈中JNI引用的对象。

在根搜索算法中不可达的对象不会立即清除，至少需要经过两次标记过程：如果对象在进行根搜索后发现没有与GC Roots相连接的引用链，那它将会被第一次标记并且进行一次筛选，筛选的条件是此对象是否有必要执行`finalize()`方法。当对象没有覆盖`finalize()`方法或者`finalize()`方法已经被虚拟机调用过，虚拟机将这两种情况都视为“没有必要执行”。如果这个对象被判断为有必要执行`finalize()`方法，那么这个对象将会被放置在一个名为`F-queue`的队列之中，并在稍后由虚拟机自动建立的、低优先级的`Finalizer`线程去执行（即调用对象的`finalize()`方法）。`finalize()`方法是对象逃脱死亡命运的最后一次机会，稍后GC将对`F-queue`中的对象进行第二次小规模的标记，如果对象在`finalize()`方法中重新与引用链上的任何一个对象建立关联，此时它将被移除“即将回收”的集合；如果对象此时并未与引用链上的任何对象建立关联，则此对象将会被回收。

方法区的垃圾收集主要回收两部分内容：废弃常量和无用的类。类需要满足三个条件才能算是“无用的类”：该类所有的实例都已经回收，也就是Java堆中不存在该类的任何实例；加载该类的ClassLoader已经被回收；该类对应的`java.lang.Class`对象没有任何地方被引用，无法在任何地方通过反射返回该类的方法。

垃圾收集算法包括四类，分别是**标记-清除算法**、**复制算法**、**标记-整理算法**、**分代收集算法**。

- 标记-清除算法（Mark-Sweep）

该算法是最基本的收集算法，它分为“标记”和“清除”两个阶段：首先标记出所有需要回收的对象，在标记完成后，统一回收掉所有被标记的对象。它的主要缺点有两个：一个是效率问题，标记和清除过程的效率都不高；另一个是空间问题，标记清除之后会产生大量不连续的内存碎片，空间碎片太多可能会导致当程序在以后的运行过程中需要分配较大对象时无法找到足够的联系内存而不能不提前触发另一次垃圾收集动作。

- 复制算法（Copying）

该算法可解决“标记-清除算法”的效率问题，它将可用内存分成两块，当其中一块用完了，就将还存活的对象复制到另一块上面，然后再把已经用过的内存空间一次性清理掉。该算法的优点是垃圾回收速度快，不会有空间碎片问题，缺点是缩小了可用内存。新生代中JVM将内存分为较大的Eden空间和两块较小的Survivor空间，工作过程是先标记Eden空间中的可用对象，将这些对象复制到其中一个Survivor空间，然后将Eden空间清除，新对象可以继续分配在Eden空间中，当Eden空间不可用时，再次进行标记，然后将可用对象复制到另一个Survivor空间中，如此反复。

- 标记-整理算法（Mark-Compact）

该算法解决“标记-清除算法”的空间碎片问题，第一步使用“标记-清除算法”，第二步将可用对象向前移动。

- 分代收集算法

根据对象存活周期不同将内存划分为几块，一般是将Java堆划分为新生代和老年代，其中，新生代使用复制算法，老年代使用标记整理算法。

## 收集器

JVM垃圾收集器分为串行和并行两类，新生代可用的垃圾收集器有**Serial**、**ParNew**、**Parallel Scavenge**，老年代可用的垃圾收集器有**CMS**、**Serial Old**、**Parallel Old**。

- Serial收集器

它是一个单线程垃圾收集器，且在进行垃圾收集时，必须暂停其他所有的工作线程（称为“Stop the world”），直到它收集结束。

- ParNew收集器

它是Serial收集器的多线程版本，可以与CMS收集器配合使用。

- Parallel Scavenge收集器

该收集器可控制吞吐量（`吞吐量=运行用户代码时间/(运行用户代码时间+垃圾收集时间`)，减少垃圾收集时间就可以提高运行用户代码时间。可使用`-XX:MaxGCPauseMillis`控制最大垃圾收集停顿时间，`-XX:GCTimeRatio`可直接设置吞吐量大小。

- Serial Old收集器

它是Serial收集器的老年代版本，也是一个单线程垃圾收集器，可以与Parallel Scavenge配合使用。使用“标记-整理”算法实现。

- Parallel Old收集器

它是Parallel Scavenge收集器的老年代版本。使用“标记-整理”算法实现。

- CMS收集器

CMS（Conccurrent Mark Sweep）收集器是一种以获取最短回收停顿时间为目标的收集器，使用“标记-清除”算法实现。它的运作过程分为4个阶段：初始标记、并发标记、重新标记、并发清除。其中初始标记和重新标记需要“Stop The World”，并发标记和并发清除可以与用户线程并发执行。耗时最长的也是并发标记和并发清除，而这两个阶段可以与用户线程一起执行，减少了停顿时间。CMS收集器有3个缺点：对CPU资源敏感（并发阶段会占用一部分线程导致应用线程变慢，总吞吐量会降低）、无法处理浮动垃圾（并发清理阶段用户线程还在运行，此时会产生新的垃圾）、存在空间碎片（由于“标记-清除算法”）。当可用空间不足或者垃圾碎片太多没有足够空间容纳晋升对象时会发生并发模式失败，此时老年代将进行垃圾收集以释放可用空间，同时也会以整理压缩以消除碎片，这个操作需要停止所有的Java应用进程，并且需要执行相当长时间。与Parallel Scavenge收集器相比，CMS老年代停顿变短了，但代价是新生代停顿略微拉长、吞吐量有所降低，堆的大小有所增加，并且由于并发，垃圾收集还会占用应用的CPU周期。

- G1收集器

**G1收集器**的设计目标是取代CMS收集器，不会产生很多内存碎片，Stop The World(STW)更可控，G1在停顿时间上添加了预测机制，用户可以指定期望停顿时间，使用复制算法实现。G1收集器将堆划分为多个内存区域（Region），每一块区域都可以作为Eden或者Survivor或者Tenured或者是大对象区域。内存区域大小可以通过参数`-XX:G1HeapRegionSize`设定，取值范围从1M到32M，且是2的指数。一次收集其中一部分，这样的方式又叫做增量收集(incremental collection)，分代收集也可以看成一种特殊的增量收集。

内存泄漏是指无用对象（不再使用的对象）持续占有内存或无用对象的内存得不到及时释放，从而造成内存空间的浪费称为内存泄漏。长生命周期的对象持有短生命周期对象的引用就很可能发生内存泄漏，尽管短生命周期对象已经不再需要，但是**因为长生命周期持有它的引用而导致不能被回收，这就是Java中内存泄漏的发生场景**。

## JVM异常退出

`OutOfMemoryError`是常见的JVM致命错误。当JVM因为致命错误而崩溃时，会生产Hotspot错误日志文件，名为`hs_err_pid<pid>.log`，这里`<pid>`是崩溃JVM进程的id，`hs_err_pid<pid>.log`文件生成在JVM的启动目录下。`hs_err_pid<pid>.log`错误日志文件包括内存镜像、操作系统级别动态库调用栈等数据，可以根据这些信息定位是哪个库方法调用时导致了JVM崩溃。`hs_err_pid<pid>.log`错误日志文件名可以使用`-XX:ErrorFile`配置。当发生`OutOfMemoryError`时，可以通过配置`-XX:+HeapDumpOnOutOfMemory`将堆信息导出到文件中。

# JVM关闭钩子

首先JVM的关闭方式可以分为三种：

- 正常关闭：当最后一个非守护线程结束或者调用了System.exit或者通过其他特定平台的方法关闭（发送SIGINT，SIGTERM信号等）
- 强制关闭：通过调用Runtime.halt方法或者是在操作系统中直接kill(发送SIGKILL信号)掉JVM进程
- 异常关闭：运行中遇到RuntimeException异常等。

JVM提供了关闭钩子（shutdown hooks）来做些扫尾的工作，比如删除临时文件、停止日志服务以及内存数据写到磁盘等，为此JVM提供了关闭钩子（shutdown hooks）来做这些事情。关闭钩子本质上是一个线程（也称为Hook线程），用来监听JVM的关闭。通过使用`Runtime`的`addShutdownHook(Thread hook)`可以向JVM注册一个关闭钩子。Hook线程在JVM **正常关闭**才会执行，在强制关闭时不会执行。对于一个JVM中注册的多个关闭钩子它们将会并发执行，所以JVM并不能保证它的执行顺行。当所有的Hook线程执行完毕后，如果此时runFinalizersOnExit为true，那么JVM将先运行终结器，然后停止。

# Agent

javaagent的主要功能如下：

- 可以在加载class文件之前做拦截，对字节码做修改
- 可以在运行期对已加载类的字节码做变更，但是这种情况下会有很多的限制，后面会详细说
- 还有其他一些小众的功能
  - 获取所有已经加载过的类
  - 获取所有已经初始化过的类（执行过clinit方法，是上面的一个子集）
  - 获取某个对象的大小
  - 将某个jar加入到bootstrap classpath里作为高优先级被bootstrapClassloader加载
  - 将某个jar加入到classpath里供AppClassloard去加载
  - 设置某些native方法的前缀，主要在查找native方法的时候做规则匹配

[JVMTI](http://docs.oracle.com/javase/7/docs/platform/jvmti/jvmti.html)全称JVM Tool Interface，是JVM暴露出来的一些供用户扩展的接口集合。JVMTI是基于事件驱动的，JVM每执行到一定的逻辑就会调用一些事件的回调接口（如果有的话），这些接口可以供开发者扩展自己的逻辑。JVMTIAgent其实就是一个动态库，利用JVMTI暴露出来的一些接口来干一些我们想做、但是正常情况下又做不到的事情。不过为了和普通的动态库进行区分，它一般会实现如下的一个或者多个函数：

```cpp
JNIEXPORT jint JNICALL
Agent_OnLoad(JavaVM *vm, char *options, void *reserved);

JNIEXPORT jint JNICALL
Agent_OnAttach(JavaVM* vm, char* options, void* reserved);

JNIEXPORT void JNICALL
Agent_OnUnload(JavaVM *vm); 
```

- Agent_OnLoad函数，如果agent是在启动时加载的，也就是在vm参数里通过-agentlib来指定的，那在启动过程中就会去执行这个agent里的Agent_OnLoad函数。
- Agent_OnAttach函数，如果agent不是在启动时加载的，而是我们先attach到目标进程上，然后给对应的目标进程发送load命令来加载，则在加载过程中会调用Agent_OnAttach函数。
- Agent_OnUnload函数，在agent卸载时调用，不过貌似基本上很少实现它。

JVMTIAgent在开发过程中经常会使用到，比如调试功能（使用`-agentlib:jdwp=transport=dt_socket,suspend=y,address=localhost:61349`来设置），它会查找一个动态库（Linux系统下为`libjdwp.so`）并加载（调用`Agent_OnLoad`方法）。

instrument实现了JVMTIAgent（动态库为`libinstrument.so`），它称为javaagent，别名JPLISAgent(Java Programming Language Instrumentation Services Agent)。instrument的使用方式是通过在启动命令上添加`-javaagent:xxx.jar`的方式加载一个被称为agent的jar包，jar包的META-INF/MANIFEST.MF中应当声明Premain-Class或Main-Class。启动时JVM会寻找这个类中的`public static void premain(String agentArgs, Instrumentation instrumentation)`, `Instrumentation`对象中可以添加自己的类修改逻辑进行字节码修改。另外当通过attach到一个运行中的JVM的方式时，可以调用`agentmain()`方法来获取`Instrumentation`对象进行类的重定义。

使用javagent实现的一些知名的库有[Btrace](https://github.com/btraceio/btrace)，[HotswapAgent](https://github.com/HotswapProjects/HotswapAgent)。

# Hotswap

热部署（Hotswap）是在不重启 Java 虚拟机的前提下，能自动侦测到 class 文件的变化，更新运行时 class 的行为。目前的 Java 虚拟机只能实现方法体的修改热部署（只有在Debug模式下才能使用），对于整个类的结构修改，仍然需要重启虚拟机，对类重新加载才能完成更新操作。默认的虚拟机行为只会在启动时加载类，如果后期有一个类需要更新的话，单纯替换编译的 class 文件，Java 虚拟机是不会更新正在运行的 class。为了实现热部署，可以使用自定义的 classloader 来加载需要监听的 class，这样就能控制类加载的时机，从而实现热部署。由于同一个类加载器无法同时加载两个相同名称的类，不论类的结构如何发生变化，生成的类名不会变，而 classloader 只能在虚拟机停止前销毁已经加载的类，这样 classloader 就无法加载更新后的类了。解决的办法是让每次加载的类都保存成一个带有版本信息的 class，比如加载 Test.class 时，保存在内存中的类是 Test_v1.class，当类发生改变时，重新加载的类名是 Test_v2.class，使用该方法后，实例化对象的方式都需要使用反射，不能使用`new`关键字。

Tomcat支持热部署，通过配置Context的`realodable`为`true`告知Tomcat某个Context支持重新加载被修改的类，这里以Tomcat的Context的reload功能为例说明Tomcat的热部署的实现方式。Tomcat的Context标准实现类是`org.apache.catalina.core.StandardContext`，在`org.apache.catalina.loader.WebappLoader`的`backgroundProcess()`方法会周期性地检查是否存在被修改的类，该方法定义如下：

```java
public void backgroundProcess() {
    if (reloadable && modified()) {
        try {
            Thread.currentThread().setContextClassLoader
                (WebappLoader.class.getClassLoader());
            if (context != null) {
                context.reload();
            }
        } finally {
            if (context != null && context.getLoader() != null) {
                Thread.currentThread().setContextClassLoader
                    (context.getLoader().getClassLoader());
            }
        }
    }
}
```

该方法会继续调用`StandardContext.reload()`方法。

`StandardContext.reload()`方法定义如下：

```java
public synchronized void reload() {
    // Validate our current component state
    if (!getState().isAvailable())
        throw new IllegalStateException
            (sm.getString("standardContext.notStarted", getName()));
    if(log.isInfoEnabled())
        log.info(sm.getString("standardContext.reloadingStarted",
                getName()));
    // Stop accepting requests temporarily.
    setPaused(true);
    try {
        stop();
    } catch (LifecycleException e) {
        log.error(
            sm.getString("standardContext.stoppingContext", getName()), e);
    }
    try {
        start();
    } catch (LifecycleException e) {
        log.error(
            sm.getString("standardContext.startingContext", getName()), e);
    }
    setPaused(false);
    if(log.isInfoEnabled())
        log.info(sm.getString("standardContext.reloadingCompleted",
                getName()));
}
```

它先调用`stop()`方法，然后再调用`start()`方法。`StandardContext.start()`之后的调用链会是`StandardContext.startInternal()`，`WebappLoader.start()`，`WebappLoader.startInternal()`。`WebappLoader`封装了`ClassLoader`，`StandardContext`将要使用到的类加载器就是`WebppLoader.classLoader`，它在`WebappLoader.startInternal()`方法内调用`createClassLoader()`方法实例化一个`ClassLoader`，代码如下：

```java
private WebappClassLoader createClassLoader()
    throws Exception {

    Class<?> clazz = Class.forName(loaderClass);
    WebappClassLoader classLoader = null;

    if (parentClassLoader == null) {
        parentClassLoader = context.getParentClassLoader();
    }
    Class<?>[] argTypes = { ClassLoader.class };
    Object[] args = { parentClassLoader };
    Constructor<?> constr = clazz.getConstructor(argTypes);
    classLoader = (WebappClassLoader) constr.newInstance(args);

    return classLoader;
}
```

Context每次reload后，`WebappLoader`都会创建一个新的`ClassLoader`，这个`ClassLoader`会重新加载Servlet相关组件的类，完成热部署的效果。

# 调优方法

对于一套应用系统来说，性能优化的内容有：架构调优、代码调优（算法和数据结构）、JVM调优、数据库调优（结构优化和SQL优化）、操作系统调优，而JVM调优主要在垃圾收集方面。通过打印GC日志（使用参数`-XX:+PrintGCDetails`），结合系统业务特征，设置新生代大小（`-Xmn`），设置堆大小（`-Xms`和`-Xmx`），设置不同的垃圾收集器。通过打印应用程序停顿时间（使用参赛`-XX:+PrintGCApplicationStoppedTime`）检查应用程序代码问题，该部分内容涉及到GC Safepoint，将在下节介绍。

对JVM性能调优前，首先需要对应用进行必要的监控，然后根据监控信息来调整相关配置，继续监控调整配置后的应用运行性能。由于JVM调优主要在垃圾收集方面，因此需要对JVM的垃圾收集进行监控，可以使用参数`-XX:+PrintGCDetails`让JVM输出垃圾收集日志，可以使用参数`-Xloggc`指定垃圾收集日志输出到文件。其他参数还包括：`-verbose:gc`（基本垃圾收集日志）、`-XX:+PrintGCTimeStamps`（打印垃圾收集触发时间戳）、`-XX:+PrintGCApplicationConcurrentTime`（垃圾收集时应用线程并发执行的时间）、`-XX:+PrintGCApplicationStoppedTime`（垃圾收集时应用线程停止执行的时间）、`-XX:+PrintTenuringDistribution`（输出每次Minor GC时晋升分布情况）。可以使用GUI工具[GCHisto](https://github.com/jewes/gchisto)分析垃圾收集日志文件。垃圾收集日志中重点需要关注这些数据指标：当前使用的垃圾收集器、Java堆大小、新生代和老年代的大小、永久代的大小、Minor GC的频率、Minor GC的空间回收量、Full GC的执行时间、Full GC的频率、每个并发垃圾收集周期内的空间回收量、垃圾收集前后Java堆的占用量、垃圾收集前后新生代和老年代的占用量、垃圾收集前后永久代的占用量、是否老年代或永久代的占用触发了Full GC、应用是否显示调用了`System.gc()`。每种收集器打印的垃圾收集日志会有差异，可以参考[Diagnosing a Garbage Collection problem](https://www.oracle.com/technetwork/java/example-141412.html)说明的日志格式，分析日志文件。

除了设置JVM输出垃圾收集日志外，在测试环境中（注意不是生产环境），还可以使用[Oracle Solaris Studio Performance Analyzer](https://www.oracle.com/technetwork/cn/server-storage/developerstudio/features/performance-analyzer-2292312-zhs.html)和[NetBeans profiler](https://profiler.netbeans.org/)收集应用运行的性能数据，包括方法执行时间、耗时最长的方法系统态CPU时间和用户态CPU时间。

JVM运行模式分为Client模式和Server模式，Client模式的特点是启动快、占用内存少、JIT编译器生成代码的速度也快，Server模式则提供了更复杂的生成代码优化功能。使用32位JVM还是64位JVM是由应用程序的内存占用决定的，同时需要考虑应用程序中的第三方库是否支持64位JVM、Java应用程序中是否使用了本地组件。早期的64位JVM会占用太多内存，原因是指针是有64位存储，而且性能也比32位JVM差，之后，JVM引入了指针压缩（`-XX:+UseCompressdOops`）的方式解决该问题。

调整年轻代、老年代空间容量大小可以控制垃圾收集次数，影响垃圾收集效率。当使用`-XX:+PrintTenuringDistribution`参数观察到对象提前进入老年代，说明Survivor空间不足，此时需要增大Survivor空间。调整Survivor空间容量的一个重要原则是：调整Survivor空间容量时，如果新生代空间大小不变，增大Survivor空间会减少Eden空间；而减少Eden空间会增加Minor GC的频率。为了同时满足应用程序Minor GC频率的要求，就需要增大当前新生代空间的大小；即增大Survivor空间大小时，Eden空间大小应该保持不变。所以当增加Survivor空间时，新生代也要增大，新生代使用`-Xmn`参数设置，可以根据如下公式计算：

```text
survivor空间大小 = -Xmn<value> / (-XX:SurvivorRatio=<ratio> + 2)
```

如果发现Minor GC持续的时间过长，就应该减少新生代空间的大小，持续调整，直到满足Minor GC的持续时间要求。

CMS垃圾收集器是最复杂（除了替换CMS的G1垃圾收集器）的垃圾收集器，对其优化过程极为复杂和困难。成功的CMS收集器调优要能以对象从新生代提升到老年代的同等速度对老年代中的对象进行垃圾收集，达不到这个标准则称为“失速”（Lost the Race），“失速”的结果是发生Stop The World压缩式垃圾收集，避免“失速”的关键是要结合足够大的老年代空间和足够快的初始化CMS垃圾收集周期，让它以比提升速率更快的速度回收空间。如果出现并发模式失效就会发生Stop The World压缩式垃圾收集，此时可以使用参数`-XX:CMSInitiatingOccupancyFraction=<percent>`设置CMS垃圾收集周期在老年代空间占用达到多少百分比时启动。并发模式失效可以通过`-XX:+PrintGCDetails`打印的日志：

```text
174.445: [GC 174.446: [ParNew: 66408K->66408K(66416K), 0.0000618 secs]174.446: [CMS(concurrent mode failure):161928K->162118K(175104K), 4.0975124 secs] 228336K->162118K(241520K)]
```

其中关键字`concurrent mode failure`表示并发模式失效。可以使用`-XX:+UseCMSInitiatingOccupancyOnly`告知Hotspot VM总是使用`-XX:CMSInitiatingOccupancyFraction`设定的值作为启动CMS周期的老年代空间占用阈值。如果不使用`-XX:+UseCMSInitiatingOccupancyOnly`，Hotspot VM仅在第一个CMS周期里使用`-XX:CMSInitiatingOccupancyFraction`设定的值作为占用比率，之后的周期中又转向自适应地启动CMS周期。`-XX:CMSInitiatingOccupancyFraction`设置的一个通用原则是老年代占用百分比应该至少是活跃数据大小的1.5倍，活跃数据大小是应用程序运行于稳定态时，长期存活的对象在Java堆中占用的空间大小，也就是Full GC后Java堆中老年代和年轻代（《Java性能优化权威指南》说是永久代，应该是书中的错误，因为Java堆使用`-Xms`或者`-Xmx`设置，而永久代使用`-XX:MaxPermSize`设置，且永久代也不属于Java堆）。CMS周期中有两个阶段是Stop The World，处于这两个阶段的应用程序会被阻塞，这两个阶段分别是初始标记阶段和重新标记阶段。虽然初始标记阶段是单线程的，却极少占用很长的时间，通常情况下远小于其他的垃圾收集停顿时间。重新标记阶段是多线程的，通过`-XX:ParallelGCThreads=<n>`可以控制重新标记阶段使用的线程数。使用`-XX:+CMSScavengeBeforeRemark`强制Hotspot VM在进入CMS重新标记阶段之前先进行一次Minor GC，Minor GC可以减少引用老年代空间的新生代对象数目，将重新标记阶段的工作量减到最少。CMS收集器与ParNew收集器是结合使用，以如下垃圾收集日志为例，说明CMS收集器的工作方式：

```text
[GC [ParNew: 64576K->960K(64576K), 0.0377639 secs] 140122K->78078K(261184K), 0.0379598 secs]
[GC [ParNew: 64576K->960K(64576K), 0.0329313 secs] 141694K->79533K(261184K), 0.0331324 secs]
[GC [ParNew: 64576K->960K(64576K), 0.0413880 secs] 143149K->81128K(261184K), 0.0416101 secs]
[GC [1 CMS-initial-mark: 80168K(196608K)] 81144K(261184K), 0.0059036 secs]
[CMS-concurrent-mark: 0.129/0.129 secs]
[CMS-concurrent-preclean: 0.007/0.007 secs]
[GC[ Rescan (non-parallel) [ grey object rescan, 0.0020879 secs][root rescan, 0.0144199 secs], 0.016
6258 secs][weak refs processing, 0.0000411 secs] [1 CMS-remark: 80168K(196608K)] 82493K(261184K),
0.0168943 secs]
[CMS-concurrent-sweep: 1.208/1.208 secs]
[CMS-concurrent-reset: 0.036/0.036 secs]
[GC [ParNew: 64576K->960K(64576K), 0.0311520 secs] 66308K->4171K(261184K), 0.0313513 secs]
[GC [ParNew: 64576K->960K(64576K), 0.0348341 secs] 67787K->5695K(261184K), 0.0350776 secs]
[GC [ParNew: 64576K->960K(64576K), 0.0359806 secs] 69311K->7154K(261184K), 0.0362064 secs]
```

CMS收集器的垃圾收集周期以`CMS-initial-mark`开始，到`CMS-concurrent-reset`结束。在上面的日志中，`CMS-initial-mark`开始时老年代占用80168K，最后一次ParNew Minor GC显示堆占用81128K，可以确定数据都在老年代，年轻代的数据基本被回收。当`CMS-concurrent-reset`结束时的第一次ParNew Minor GC显示堆占用4171K，说明运行一次CMS老年代垃圾收集回收了非常多的对象，内存释放了将近74M（(81128 - 4171) / 1024）。

对Parallel Old收集器进行吞吐量性能调优的目标是尽可能避免发生Full GC，或者更理想的情况下在稳定态时永远不发生Full GC。Parallel Old收集器使用`-XX:+UseParallelOldGC`和`-XX:+UseParallelGC`选项开启，它提供的吞吐量性能是Hotspot VM诸多垃圾收集器中最好的。Parallel Old收集器默认使用自适应大小调整Eden和Survivor空间，可以使用`-XX:-UseAdaptiveSizePolicy`禁用自适应大小调整。使用`-XX:+PrintAdaptiveSizePolicy`可以打印自适应大小调整的日志，日志内容如下：

```text
2010-12-16T21:44:11.444-0600:
	[GCAdaptiveSizePolicy::compute_survivor_space_size_and_thresh:
		survived: 224408984
		promoted: 10904856
		overflow: false
		[PSYongGen: 6515579K->219149K(9437184K)]
	8946490K->2660709K(13631488K), 0.0725945 secs]
	[Times : user=0.56 sys=0.00, real=0.07 secs]
```

`survived`标签的右边是“To”Survivor空间中存活对象的大小，`promoted`标签右边是由新生代提升至老年代空间的对象大小。`overflow`标签右边的文字表明是否有survivor空间的对象溢出到了老年代空间。如果Survivor空间占用没有通过`-XX:TargetSurvivorRation=<percent>`设定，目标Survivor空间则使用默认值50%，表示如果Survivor空间占用超过该设定值时，对象在未到达他们的最大年龄之前就会被提升至老年代。

通过分代垃圾收集原理和各垃圾收集器的特点，可以解决如下问题：

* 对象提升到老年代的速度太快

通过`-XX:+PrintTenuringDistribution`选项观察到对象在未达到最大年龄时就进入老年代，对象提前进入老年代的原因是年轻代垃圾收集时Survivor空间不够，此时需要增加Survivor空间（还需要增加堆空间，保持Eden空间不变）。

* 经过一次垃圾收集堆使用率未降低

触发垃圾收集的原因是年轻代或者老年代没有足够的空间容纳新分配的对象，当执行一次垃圾收集时堆使用率未降低说明并未回收任何对象。存在两种情况，第一种是年轻代全部进入老年代，此时堆空间使用率未降低；第二种是在对象晋升不明显的情况下，堆空间使用率依然未降低，这表示对象是长期存活的，此时需要检查是否存在**内存泄漏**。内存泄漏可以使用jmap命令检查存活对象，分析这些对象为何长期存活。比如使用`jmap -histo:live 348`查看348进程存活的对象，该命令输出时会按照对象类型对应的实例数量降序排序。如果发现内存泄漏则优化程序，如果不是内存泄漏，则可以增加堆空间，减少垃圾收集次数。

* GC线程CPU使用率过高

造成此类情况一般时垃圾收集频繁发生且垃圾回收数量低，可以根据“经过一次垃圾收集堆使用率未降低”来解决。

* 垃圾收集的次数太多

触发垃圾收集的原因是年轻代或者老年代没有足够的空间容纳新分配的对象，此时可以增加堆内存。

* 应用无响应或者进入假死状态

使用`jstack`观察应用程序进程的线程信息，查看现场执行情况，检查是否存在IO等待和锁等待，优化应用程序。

* 系统态CPU执行时间（或使用率）大于用户态CPU执行时间（或使用率）

当应用程序调用系统函数时，此时操作系统从用户态进入内核态，在内核态运行的时间就是系统态CPU执行时间。由于系统调用的开销比较大，而应用系统很难避免系统调用，此时就需要优化应用系统。对于Java应用程序，发生系统调用此时比较多的是监视锁功能，JVM的优化是使用首先使用轻量级锁，如果锁竞争激烈，转入重量级锁，使用重量级锁就是使用操作系统函数完成。除此外，还有锁消除等优化。文件操作也会调用系统函数，对文件操作时应当使用缓存（在`java.io`包下以`Buffer`开发头类名），包括读取和写入，缓存大小应该是操作系统内存页的整数倍，在LInux系统中，内存页可以使用命令`pagesize`或者`getconf PAGESIZE`获得。

* 并行性优化

现代CPU都是多核多线程架构，应用程序可以使用该特点尽可能地使用更多的CPU资源。比如在一个大集合排序中，可以将集合划分多个区域，使用多个CPU资源（就是使用多线程）排序，然后使用归并排序算法完成整个大集合的排序。

使用`-XX:+PrintGCApplicationStoppedTime`打印的日志如下：

```text
Total time for which application threads were stopped: 0.0051000 seconds  
Total time for which application threads were stopped: 0.0041930 seconds  
Total time for which application threads were stopped: 0.0051210 seconds  
Total time for which application threads were stopped: 0.0050940 seconds  
Total time for which application threads were stopped: 0.0058720 seconds  
Total time for which application threads were stopped: 5.1298200 seconds
Total time for which application threads were stopped: 0.0197290 seconds  
Total time for which application threads were stopped: 0.0087590 seconds 
```

其中有一次应用程序停顿的时间非常长，可能的问题是应用程序设计不当，导致某个或者某些线程在垃圾回收期间无法立即进入到GC Safepoint，不当的情况有：1.大循环体导致JVM不能插入check safepoint代码，2.大IO时，操作系统需要读取或者写入文件时，线程需要等待操作系统完成才能继续执行代码，才能执行check safepoint代码。打印停顿实现时还需要配合`-XX:+PrintSafepointStatistics`和`-XX:PrintSafepointStatisticsCount=1`两个参数以便查看停顿时系统正在执行什么VM操作。相关案例可参考 [ParNew 应用暂停时间偶尔会出现好几秒的情况](https://hllvm-group.iteye.com/group/topic/38836)和[Eliminating Large JVM GC Pauses Caused by Background IO Traffic](https://engineering.linkedin.com/blog/2016/02/eliminating-large-jvm-gc-pauses-caused-by-background-io-traffic)。

# Safepoint

当JVM执行垃圾收集时，需要所有的用户线程暂停，不要操作堆内存，只有这样，才能让GC安全的访问堆内存对象。除了GC时需要暂停用户线程，包括jstack和jmap这样的使用JVMTI的工具也需要访问堆内存对象，同样需要暂停用户线程。

OpenJDK对Safepoint的描述是“A point during program execution at which all GC roots are known and all heap object contents are consistent. ”[ [1] ](https://openjdk.java.net/groups/hotspot/docs/HotSpotGlossary.html)。

`VMOperationQueue`队列存放的是操作JVM的请求，当需要GC、jstack、jmap时，需要向`VMOperationQueue`队列添加一个消息。`VMThread`线程在`loop`循环内不停地取出`VMOperationQueue`队列的消息，设置Safepoint标记，让用户线程去检查这个标记，然后用户线程暂停[ [2] ](https://openjdk.java.net/groups/hotspot/docs/RuntimeOverview.html)。

那么`VMThread`做了哪些操作，用户线程又该如何检查Safepoint标记？

[vmThread.cpp](http://hg.openjdk.java.net/jdk7/jdk7/hotspot/file/f03d0a26bf83/src/share/vm/runtime/vmThread.cpp)定义了`VMOperationQueue`为环形的双向链表，`VMThread`在`loop`方法的`while`语句中循环的取出`VMOperation`类型的消息。当取出一个`VMOperation`对象时，使用`_cur_vm_operation->evaluate_at_safepoint()`判断处理此VM操作请求时是否需要进入Safepoint状态。如果需要进入Safepoint状态，则调用`SafepointSynchronize::begin()`方法为用户线程进入Safepoint做准备。`begin`方法定义在[safepoint.cpp](http://hg.openjdk.java.net/jdk8/jdk8/hotspot/file/87ee5ee27509/src/share/vm/runtime/safepoint.cpp)中。根据`SafepointSynchronize::begin()`方法的源码注释，可以知道JVM需要在五个地方检查标记，注释原文如下：

```java
  // Begin the process of bringing the system to a safepoint.
  // Java threads can be in several different states and are
  // stopped by different mechanisms:
  //
  //  1. Running interpreted
  //     The interpeter dispatch table is changed to force it to
  //     check for a safepoint condition between bytecodes.
  //  2. Running in native code
  //     When returning from the native code, a Java thread must check
  //     the safepoint _state to see if we must block.  If the
  //     VM thread sees a Java thread in native, it does
  //     not wait for this thread to block.  The order of the memory
  //     writes and reads of both the safepoint state and the Java
  //     threads state is critical.  In order to guarantee that the
  //     memory writes are serialized with respect to each other,
  //     the VM thread issues a memory barrier instruction
  //     (on MP systems).  In order to avoid the overhead of issuing
  //     a memory barrier for each Java thread making native calls, each Java
  //     thread performs a write to a single memory page after changing
  //     the thread state.  The VM thread performs a sequence of
  //     mprotect OS calls which forces all previous writes from all
  //     Java threads to be serialized.  This is done in the
  //     os::serialize_thread_states() call.  This has proven to be
  //     much more efficient than executing a membar instruction
  //     on every call to native code.
  //  3. Running compiled Code
  //     Compiled code reads a global (Safepoint Polling) page that
  //     is set to fault if we are trying to get to a safepoint.
  //  4. Blocked
  //     A thread which is blocked will not be allowed to return from the
  //     block condition until the safepoint operation is complete.
  //  5. In VM or Transitioning between states
  //     If a Java thread is currently running in the VM or transitioning
  //     between states, the safepointing code will wait for the thread to
  //     block itself when it attempts transitions to a new state.
  //
```

Safepoint的准备和处理：

* Java字节码解释器：调用`TemplateInterpreter::notice_safepoints()`修改dispatch table。dispatch table用来记录方法地址，类型是`DispatchTable`，TemplateInterpreter定义了三个dispatch table，分别是`_active_table`、 `_normal_table`和 `_safept_table`。`_active_table`是正在解释运行的线程使用的dispatch table，`_normal_table`就是正常运行的初始化的dispatch table，`_safept_table`是safe point需要的dispatch table。解释运行的线程一直都在使用`_active_table`，在进入saftpoint 的时候，用`_safept_table`替换`_active_table`, 在退出saftpoint 的时候，使用`_normal_table`来替换`_active_table`。`notice_safepoints`方法内部调用`copy_table`来处理dispatch table替换的操作。当新的dispatch table被访问时，就会访问到进入Safepoint[ [3] ](https://yemablog.com/posts/safepoint-on-interpreter-mode)。
* 当执行流程从JNI返回到JVM时，也会因为dispatch table被替换的原因进入到Safepoint。
* 对于C1/C2编译器编译的代码，编译器会在无限循环体内插入访问Safepoint页的代码，被插入的代码实际就是`test   %eax,PAGE_ADDRESS`[ [4] ](https://www.jianshu.com/p/c79c5e02ebe6)，被称为poll操作。`SafepointSynchronize::begin()`会调用`os::make_polling_page_unreadable()`方法使得Safepoint页不可读，`test`访问到不能读的内存时，操作系统将执行流程陷入到中断处理，JVM的`JVM_handle_linux_signal`方法处理中断请求，进入到Safepoint[ [5] ](https://blog.csdn.net/raintungli/article/details/7162468)。注意，由于是无限循环体，如果是在一个大循环内，即多层for循环，此时会出现GC时间过长的问题[ [6] ](http://psy-lob-saw.blogspot.com/2015/12/safepoints.html)。
* 如果线程处于Blocked状态，继续让其处于Blocked状态，直到Safepoint处理结束。
* 线程状态发生改变时，也会检查Safepoint标记并且进入Safepoint。



# 动态追踪

[BTrace](https://github.com/btraceio/btrace)是

相关文章：[动态追踪技术漫谈](http://openresty.org/posts/dynamic-tracing/)，[BTrace原理浅析](https://www.rowkey.me/blog/2016/09/20/btrace/)