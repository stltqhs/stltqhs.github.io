---
title: Java后端开发 - 线程
date: 2018-10-13 08:14:49
tags: java
---

# 线程生命周期 

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

- New（新建）

  当构造一个线程时（如`new Thread()`），则该线程的状态是`New`，此时`Thread.threadStatus = 0`，当调用`Thread.getState()`时返回`Thread.State.NEW`；

- Runnable（运行）

  该状态线的线程可能正在执行或者正在等待线程调度器选中，一般会将该状态拆分为Ready（就绪）和Running（运行），而Ready状态到Runnable状态是操作系统控制；

- Waiting（无限期等待）

  线程处于无限期等待某一个条件触发，处于该状态的原因是调用了`Object.wait()`、`Thread.join()`、`LockSupport.park()`方法。

- Timed_Waiting（限期等待）

  线程处于限期等待，处于该状态的原因是调用了`Object.wait(long)`、`Thread.join(long)`、`LockSupport.parkNanos(long)`、`LockSupport.parkUntil(long)`。

- Blocked（阻塞）

  线程正在等待监视锁以进入到`synchronized`方法和语句快；

- Terminated（结束）

  线程`run()`方法或者主线程`main()`方法结束或者抛出未捕获的异常时；

某些资料或者书籍会将Waiting、Timed_Waiting以及Blocked合并为一个状态，称为Blocked，即阻塞。

  

下图展示线程状态转换流程。

![线程状态机](https://i.stack.imgur.com/FTHOR.png "线程状态机")

参考：[Java多线程学习(三)---线程的生命周期](https://www.cnblogs.com/sunddenly/p/4106562.html)，[一张图让你看懂JAVA线程间的状态转换](https://my.oschina.net/mingdongcheng/blog/139263)，[对Java线程概念的理解](https://blog.csdn.net/fuzhongmin05/article/details/71425191)，[visualvm thread states](https://stackoverflow.com/questions/27406200/visualvm-thread-states)

# wait()、sleep()、notify()区别 

- wait

  使线程处于Waiting或者Timed_Waiting，该线程进入wait queue，而且会释放对象的监视锁。使用wait方法时，该线程必须拥有wait对象的监视锁，即wait方法只能放在对象的同步方法或者同步语句块中，如：

  ```java
  Object mon = ...;
  synchronized (mon) {
      mon.wait();
  }
  ```

  wait方法是在Object中定义的native方法。

- sleep

  使线程处于Waiting或者Timed_Waiting，但是不会使线程释放监视锁，且该方法为Thread的静态方法；

- notify

  释放该对象的监视锁，从对象的wait queue中唤醒一个线程或者多个线程（此时调用notifyAll方法），被唤醒的线程需要重新获取该对象的监视锁，以进入同步块。使用notify方法时，该线程必须拥有notify对象的监视锁，即notify方法只能放在对象的同步方法或者同步语句块中，如：

  ```java
  Object mon = ...;
  synchronized(mon) {
      mon.notify();
  }
  ```

参考：[difference between wait and sleep](https://stackoverflow.com/questions/1036754/difference-between-wait-and-sleep)

# ThreadPoolExecutor原理 

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

# ThreadLocal实现方式 

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

# 中断机制 

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

# 活跃性

- 死锁

  如果多个线程以固定的顺序来获取锁时，那么他们将不回存在死锁的问题。死锁发生表示获取锁的顺序不一致导致互相占用各自需要获取的锁。死锁的解决方法有：1)限制锁的调用顺序，2)缩小锁的范围，3)使用显示锁替换内置锁，灵活控制锁策略。

- 饥饿

  由于线程得不到需要的资源，不能正常执行，就会造成线程饥饿。如对线程的优先级设置不当，造成线程不能获得CPU周期执行导致饥饿，或者其他线程长时间持有锁，导致其他线程长时间等待，造成的饥饿。

- 活锁

  活锁问题不会导致线程阻塞，但是活锁会导致线程不能继续正常执行。比如这样一个消息系统中，从队列里边取的消息，然后执行，但是由于某种业务原因，失败了，那么把它放到队列头，然后再拿出来执行，自然还是失败的，这样线程虽然没有阻塞，但是也不能正常的处理其他的消息。



  当出现活跃性故障时，除了终止应用程序之外没有其他任何机制可以帮助从这种故障恢复过来。

参考：[java并发编程中的活跃性问题](http://study-a-j8.iteye.com/blog/2366489)