---
title: Java后端开发 - 并发
date: 2018-10-13 08:14:57
tags: java
---

# 死锁、活锁和饥饿

死锁产生的原因是：

* 系统资源不足
* 进程运行推进的顺序不正确
*  资源分配不当等

如果系统资源充足，进程的资源请求都能够得到满足，死锁出现的可能性就很低，否则就会因争夺有限的资源而陷入死锁。其次，进程运行推进顺序与速度不同，也可能产生死锁。

产生死锁的四个必要条件：

* 互斥条件：一个资源每次只能被一个进程使用
* 请求与保持条件：一个进程因请求资源而阻塞时，对已获得的资源保持不放
* 不剥夺条件:进程已获得的资源，在末使用完之前，不能强行剥夺
* 循环等待条件:若干进程之间形成一种头尾相接的循环等待资源关系

这四个条件是死锁的必要条件，只要系统发生死锁，这些条件必然成立，而只要上述条件之
一不满足，就不会发生死锁。

死锁的解除与预防：
理解了死锁的原因，尤其是产生死锁的四个必要条件，就可以最大可能地避免、预防和
解除死锁。所以，在系统设计、进程调度等方面注意如何不让这四个必要条件成立，如何确
定资源的合理分配算法，避免进程永久占据系统资源。此外，也要防止进程在处于等待状态
的情况下占用资源。因此，对资源的分配要给予合理的规划。

MySQL中比较常出现死锁问题，解决这类问题的方法是保证不同事物加锁的顺序一致，其次，不要使用共享锁（即select ... lock in share mode和外健）。

活锁和饥饿见[Starvation and Livelock](https://docs.oracle.com/javase/tutorial/essential/concurrency/starvelive.html)。饥饿是指在没有锁互相等待时也长时间获取不到锁，造成线程等待。活锁是指线程并未互相等待，而是一直在执行“重试”或者“谦让”的过程，线程状态还是RUN。

# happens-before原则 

在共享内存的多处理器体系结构中，每个处理器都拥有自己的缓存，并且定期地与主内存进行协调。此时就存在处理器P1修改变量A时，在同步变量A到主内存之前，处理器P2读取变量A将是一个旧值。此类问题只能由程序来控制这种**内存不一致**的问题。

Java为各种处理器体系不同的内存模型，抽象了Java自身的内存模型（Java Memory Model，或简称JMM）。所以Java内存模型不是一个具体的内存，而是抽象的内存，包括处理器多级缓存、寄存器缓存、处理器指令重排序、JVM指令重排序等。

为了解决上述**内存不一致**的情况，Java需要根据自身的语言要求在特定位置插入内存栅栏来刷新缓存数据，保证内存数据和处理器中的缓存数据一致，或者禁止处理器重排序。

**要想保证执行操作B的线程看到操作A的结果（无论A和B是否在同一个线程中执行），那么在A和B之间必须满足Happens-Before关系**。如果两个操作之间缺乏Happends-Before关系，那么JVM可以对他们任意地重排序。

Happens-Before的规则包括：

- **程序顺序规则** 如果程序中操作A在操作B之前，那么线程中A操作将在B操作之前执行。如果A happens- before B，JMM并不要求A一定要在B之前执行。JMM仅仅要求前一个操作（执行的结果）对后一个操作可见，且前一个操作按顺序排在第二个操作之前；
- **监视器锁规则** 在监视器锁上的解锁操作必须在同一个监视器锁上的加锁操作之前执行；
- **volatile变量规则** 对volatile变量的写入操作必须在对该变量的读取操作之前执行；
- **线程启动规则** 在线程上对`Thread.start()`的调用必须在该线程中执行任何操作之前执行；
- **线程结束规则** 线程中的任何操作都必须在其他线程检测到该线程已经结束之前执行，或者从`Thread.join()`中成功返回，或者在调用`Thread.isAlive()`时返回`false`；
- **中断规则** 当一个线程在另一个线程上调用`interrupt`时，必须在被中断线程检测到`interrupt`调用之前执行（通过抛出`InterruptedException`，或者调用`isInterrupted()`或`interrupted()`）；
- **终结器规则** 对象的构造函数必须在启动该对象的终结器之前执行完成；
- **传递性** 如果操作A在操作B之前执行，并且操作B在操作C之前执行，那么操作A必须在操作C之前执行。

JVM可以保证在调用任何类的静态方法或者静态字段时，JVM都已经对类正确的执行了静态初始化，即调用类的静态初始化函数（`<cinit>`方法）是原子操作。

JVM可以保证`final`字段可以在构造函数退出前完成初始化，不会把`final`字段初始化重排序到构造函数外执行（这里有个前提是在构造函数中没有把`this`逸出）。例如：

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

延伸阅读：

* [深入理解Java内存模型（一）——基础](http://www.infoq.com/cn/articles/java-memory-model-1)
* [深入理解Java内存模型（二）——重排序](http://www.infoq.com/cn/articles/java-memory-model-2)
* [深入理解Java内存模型（三）——顺序一致性](http://www.infoq.com/cn/articles/java-memory-model-3)
* [深入理解Java内存模型（四）——volatile](http://www.infoq.com/cn/articles/java-memory-model-4)
* [深入理解Java内存模型（五）——锁](http://www.infoq.com/cn/articles/java-memory-model-5)
* [深入理解Java内存模型（六）——final](http://www.infoq.com/cn/articles/java-memory-model-6)

# volatile作用 

volatile有两个作用，一个是将long和double类型的读取和写入操作原子化（由于long和double是64位，JVM内部会将long和double的操作分为两个32位的操作，而且不是原子操作），另一个是控制变量线程间的可见性（**volatile变量规则**规定对volatile变量的写入操作必须在对该变量的读取操作之前执行）。



如果一个int类型变量i被volatile修饰，那么`i++`操作并不是线程安全的操作。`volatile`只能保证变量读取和写入是原子性，且读取变量时会刷新缓存，保证线程间的可见性。而`i++`操作包含3个操作，首先是读取变量到临时变量中，然后临时变量加1，再将临时变量写入，这3个操作合起来并不是原子操作，所以不是线程安全的。

# CAS 

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

ABA问题可能会导致灾难性的后果，因此在某些场景需要使用特殊的方法解决**ABA**问题。目前解决**ABA**问题的方法是使用一个修改次数的变量作为版本号。Java提供的`AtomicStampedReference`也是基于版本控制来解决**ABA**问题。`AtomicStampedReference`的比较替换方法如下：

```java
public boolean compareAndSet(V   expectedReference,
                             V   newReference,
                             int expectedStamp,
                             int newStamp) {
    Pair<V> current = pair;
    return
        expectedReference == current.reference &&
        expectedStamp == current.stamp &&
        ((newReference == current.reference &&
          newStamp == current.stamp) ||
         casPair(current, Pair.of(newReference, newStamp)));
}

private boolean casPair(Pair<V> cmp, Pair<V> val) {
    return UNSAFE.compareAndSwapObject(this, pairOffset, cmp, val);
}
```



# LockSupport原理

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

`park`和`unpark`方法分别调用`sun.misc.Unsafe`的`park`和`unpark`方法，这两个方法的签名如下：

```java
public native void unpark(Object thread);
public native void park(boolean isAbsolute, long time);
```

`sun.misc.Unsafe.park`和`sun.misc.Unsafe.unpark`都是native方法，由C++代码调用操作系统API实现。不同操作系统有不同实现方式，本文以Linux平台的实现方式叙述`sun.misc.Unsafe.park`和`sun.misc.Unsafe.unpark`是如何实现的。Linux实现的代码在文件[hotspot/src/os/linux/vm/os_linux.cpp](https://github.com/dmlloyd/openjdk/blob/jdk7u/jdk7u/hotspot/src/os/linux/vm/os_linux.cpp)中，方法定义如下：

```c++
void Parker::park(bool isAbsolute, jlong time) {
  // Ideally we'd do something useful while spinning, such
  // as calling unpackTime().

  // Optional fast-path check:
  // Return immediately if a permit is available.
  // We depend on Atomic::xchg() having full barrier semantics
  // since we are doing a lock-free update to _counter.
  if (Atomic::xchg(0, &_counter) > 0) return;

  Thread* thread = Thread::current();
  assert(thread->is_Java_thread(), "Must be JavaThread");
  JavaThread *jt = (JavaThread *)thread;

  // Optional optimization -- avoid state transitions if there's an interrupt pending.
  // Check interrupt before trying to wait
  if (Thread::is_interrupted(thread, false)) {
    return;
  }

  // Next, demultiplex/decode time arguments
  timespec absTime;
  if (time < 0 || (isAbsolute && time == 0) ) { // don't wait at all
    return;
  }
  if (time > 0) {
    unpackTime(&absTime, isAbsolute, time);
  }


  // Enter safepoint region
  // Beware of deadlocks such as 6317397.
  // The per-thread Parker:: mutex is a classic leaf-lock.
  // In particular a thread must never block on the Threads_lock while
  // holding the Parker:: mutex.  If safepoints are pending both the
  // the ThreadBlockInVM() CTOR and DTOR may grab Threads_lock.
  ThreadBlockInVM tbivm(jt);

  // Don't wait if cannot get lock since interference arises from
  // unblocking.  Also. check interrupt before trying wait
  if (Thread::is_interrupted(thread, false) || pthread_mutex_trylock(_mutex) != 0) {
    return;
  }

  int status ;
  if (_counter > 0)  { // no wait needed
    _counter = 0;
    status = pthread_mutex_unlock(_mutex);
    assert (status == 0, "invariant") ;
    // Paranoia to ensure our locked and lock-free paths interact
    // correctly with each other and Java-level accesses.
    OrderAccess::fence();
    return;
  }

#ifdef ASSERT
  // Don't catch signals while blocked; let the running threads have the signals.
  // (This allows a debugger to break into the running thread.)
  sigset_t oldsigs;
  sigset_t* allowdebug_blocked = os::Linux::allowdebug_blocked_signals();
  pthread_sigmask(SIG_BLOCK, allowdebug_blocked, &oldsigs);
#endif

  OSThreadWaitState osts(thread->osthread(), false /* not Object.wait() */);
  jt->set_suspend_equivalent();
  // cleared by handle_special_suspend_equivalent_condition() or java_suspend_self()
  assert(_cur_index == -1, "invariant");
  if (time == 0) {
    _cur_index = REL_INDEX; // arbitrary choice when not timed
    status = pthread_cond_wait (&_cond[_cur_index], _mutex) ;
  } else {
    _cur_index = isAbsolute ? ABS_INDEX : REL_INDEX;
    status = os::Linux::safe_cond_timedwait (&_cond[_cur_index], _mutex, &absTime) ;
    if (status != 0 && WorkAroundNPTLTimedWaitHang) {
      pthread_cond_destroy (&_cond[_cur_index]) ;
      pthread_cond_init    (&_cond[_cur_index], isAbsolute ? NULL : os::Linux::condAttr());
    }
  }
  _cur_index = -1;

  assert_status(status == 0 || status == EINTR ||
                status == ETIME || status == ETIMEDOUT,
                status, "cond_timedwait");

#ifdef ASSERT
  pthread_sigmask(SIG_SETMASK, &oldsigs, NULL);
#endif

  _counter = 0 ;
  status = pthread_mutex_unlock(_mutex) ;
  assert_status(status == 0, status, "invariant") ;
  // Paranoia to ensure our locked and lock-free paths interact
  // correctly with each other and Java-level accesses.
  OrderAccess::fence();

  // If externally suspended while waiting, re-suspend
  if (jt->handle_special_suspend_equivalent_condition()) {
    jt->java_suspend_self();
  }
}

void Parker::unpark() {
  int s, status ;
  status = pthread_mutex_lock(_mutex);
  assert (status == 0, "invariant") ;
  s = _counter;
  _counter = 1;
  if (s < 1) {
    // thread might be parked
    if (_cur_index != -1) {
      // thread is definitely parked
      if (WorkAroundNPTLTimedWaitHang) {
        status = pthread_cond_signal (&_cond[_cur_index]);
        assert (status == 0, "invariant");
        status = pthread_mutex_unlock(_mutex);
        assert (status == 0, "invariant");
      } else {
        status = pthread_mutex_unlock(_mutex);
        assert (status == 0, "invariant");
        status = pthread_cond_signal (&_cond[_cur_index]);
        assert (status == 0, "invariant");
      }
    } else {
      pthread_mutex_unlock(_mutex);
      assert (status == 0, "invariant") ;
    }
  } else {
    pthread_mutex_unlock(_mutex);
    assert (status == 0, "invariant") ;
  }
}
```

当调用`Parker::unpark`方法时，首先获取`_mutex`的锁，`pthread_mutex_lock`函数是阻塞方法，只有线程获取了`_mutex`的锁时函数才会返回0，否则阻塞，直到获取`_mutex`的锁。获取了`_mutex`的锁后将许可`_counter`备份在本地变量`s`中，将其赋值为1。如果许可`_counter`旧值`s`小于1，表示线程可能调用了`Parker::park`方法。如果`_cur_index`的值大于等于0，表示线程已经调用了`Parker::park`方法并通过系统函数`pthread_cond_wait`将该线程阻塞等待`_cur_index`条件变量的信号通知。此时需要调用`pthread_cond_signal`函数向阻塞的线程发送信号通知，解除线程阻塞。

当调用`Parker::park`方法时，先将许可`_counter`置为0，实现的方式是使用`Atomic::xchg`方法完成，该方法的定义如下：

```cpp
inline jint     Atomic::xchg    (jint     exchange_value, volatile jint*     dest) {
  __asm__ volatile (  "xchgl (%2),%0"
                    : "=r" (exchange_value)
                    : "0" (exchange_value), "r" (dest)
                    : "memory");
  return exchange_value;
}
```

`Atomic::xchg`的方法的工作方式类似于`swap`置换变量的方式：

```text
dest -> TEMP
exchange_value -> dest
TEMP -> exchange_value
```

`xchg`汇编指令描述可参考[Exchange Register / Memory With Register (xchg)](https://docs.oracle.com/cd/E19620-01/805-4693/instructionset-124/index.html)，`__asm__`语法可参考[GCC Inline Assembly HOWTO](http://www.ibiblio.org/gferg/ldp/GCC-Inline-Assembly-HOWTO.html)。

如果`Atomic::xchg(0, &_counter) > 0`为`true`时，表示许可`_counter`值为1，表示该线程处于`unpark`状态，此时函数返回，并未让线程阻塞。这需要上层程序在循环中判断线程是否可以获取某个资源，不能获取资源时则调用`park`方法，比如：

```java
// Block while not first in queue or cannot acquire lock
while (waiters.peek() != current ||
       !locked.compareAndSet(false, true)) {
   LockSupport.park(this);
   if (Thread.interrupted()) // ignore interrupts while waiting
     wasInterrupted = true;
}
```

当第一次执行循环体，调用`LockSupport.park(this);`时线程未阻塞，则`while`循环的条件需要让该线程继续获取资源，如果获取失败，则继续调用`LockSupport.park(this)`方法。此时`Atomic::xchg(0, &_counter) > 0`为`false`，此时`Parker::park`方法将会尝试阻塞该线程。事实上，**`park` 方法还可以在其他任何时间“毫无理由”地返回，因此通常必须在重新检查返回条件的循环里调用此方法**。可以认为当调用`Parker::unpark`时，许可`_counter`置为1，当调用`Parker::park`时，如果许可`_counter`为1，将许可`_counter`置为0并返回，否则等待许可`_counter`。这个过程类似于信号量，不同的是许可不能累加，最大值为1。

`ThreadBlockInVM`的功能是插入内存栅栏，防止CPU对代码进行重排序，将线程的工作内存都刷新。`ThreadBlockInVM`在[hotspot/src/share/vm/runtime/interfaceSupport](https://github.com/dmlloyd/openjdk/blob/jdk7u/jdk7u/hotspot/src/share/vm/runtime/interfaceSupport.hpp)中定义，如下：

```cpp
class ThreadBlockInVM : public ThreadStateTransition {
 public:
  ThreadBlockInVM(JavaThread *thread)
  : ThreadStateTransition(thread) {
    // Once we are blocked vm expects stack to be walkable
    thread->frame_anchor()->make_walkable(thread);
    trans_and_fence(_thread_in_vm, _thread_blocked);
  }
  ~ThreadBlockInVM() {
    trans_and_fence(_thread_blocked, _thread_in_vm);
    // We don't need to clear_walkable because it will happen automagically when we return to java
  }
};
```

`Parker::park`方法会调用`pthread_mutex_trylock`函数尝试获取`_mutex`的锁，该函数并非阻塞模式，因此如果无法获取锁，`Parker::park`方法会立即返回。此时调用`LockSupport.park(this)`的循环体会一直执行，要么该线程能够获得资源，否则继续调用`LockSupport.park(this)`方法，`Parker::park`将再次尝试获取`_mutex`的锁。如果`_mutex`的锁获取成功，检查许可`_counter`的值，如果大于0，表示该线程执行了`Parker::unpark`方法将许可`_counter`置为1，函数释放锁后立即返回，在下次循环中将许可`_counter`置为0；如果小于0，则调用`pthread_cond_wait`函数阻塞该线程，此时完成了线程阻塞操作。

`unpark`方法可以先于`park`调用。使用`os::Linux::safe_cond_timedwait`方法可以设置等待一个互斥变量的超时时间。

# AQS原理 

AQS是`AbstractQueuedSynchronizer`类的简称，它是`java.concurrent.util`包里各种独占锁或者共享锁（包括`ReentrantLock`和`Semaphore`等）实现的基础。

AQS使用一个`volatile int state`表示同步状态，并使用CAS操作保证条件判断与值更新的原子性。线程阻塞和唤醒使用`LockSupport.park()`和`LockSupport.unpark()`完成，使用FIFO队列管理阻塞的线程。

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

AQS的`release`方法的实现如下：

```java
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
private void unparkSuccessor(Node node) {
    /*
     * If status is negative (i.e., possibly needing signal) try
     * to clear in anticipation of signalling.  It is OK if this
     * fails or if status is changed by waiting thread.
     */
    int ws = node.waitStatus;
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);

    /*
     * Thread to unpark is held in successor, which is normally
     * just the next node.  But if cancelled or apparently null,
     * traverse backwards from tail to find the actual
     * non-cancelled successor.
     */
    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
        LockSupport.unpark(s.thread);
}
```

AQS的`acquire`方法的实现如下：

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    // Try the fast path of enq; backup to full enq on failure
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    enq(node);
    return node;
}
private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        if (t == null) { // Must initialize
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);
    return Thread.interrupted();
}
```

AQS的`acquireShare`方法的实现如下：

```java
public final void acquireShared(int arg) {
    if (tryAcquireShared(arg) < 0)
        doAcquireShared(arg);
}
private void doAcquireShared(int arg) {
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    if (interrupted)
                        selfInterrupt();
                    failed = false;
                    return;
                }
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
private void setHeadAndPropagate(Node node, int propagate) {
    Node h = head; // Record old head for check below
    setHead(node);
    /*
     * Try to signal next queued node if:
     *   Propagation was indicated by caller,
     *     or was recorded (as h.waitStatus) by a previous operation
     *     (note: this uses sign-check of waitStatus because
     *      PROPAGATE status may transition to SIGNAL.)
     * and
     *   The next node is waiting in shared mode,
     *     or we don't know, because it appears null
     *
     * The conservatism in both of these checks may cause
     * unnecessary wake-ups, but only when there are multiple
     * racing acquires/releases, so most need signals now or soon
     * anyway.
     */
    if (propagate > 0 || h == null || h.waitStatus < 0) {
        Node s = node.next;
        if (s == null || s.isShared())
            doReleaseShared();
    }
}
```

AQS的`releaseShare`方法的实现如下：

```java
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}
private void doReleaseShared() {
    /*
     * Ensure that a release propagates, even if there are other
     * in-progress acquires/releases.  This proceeds in the usual
     * way of trying to unparkSuccessor of head if it needs
     * signal. But if it does not, status is set to PROPAGATE to
     * ensure that upon release, propagation continues.
     * Additionally, we must loop in case a new node is added
     * while we are doing this. Also, unlike other uses of
     * unparkSuccessor, we need to know if CAS to reset status
     * fails, if so rechecking.
     */
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) {
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                unparkSuccessor(h);
            }
            else if (ws == 0 &&
                     !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;                // loop on failed CAS
        }
        if (h == head)                   // loop if head changed
            break;
    }
}
```

当调用`tryAcquire()`返回true或者`tryAcquireShared()`返回值大于0时，线程不需要阻塞。否则需要向FIFO队列添加一个节点（包括当前线程对象信息），阻塞该线程，然后进入`acquireQueued`循环，不停的尝试获取锁。当获取锁成功时，需要退出`acquireQueued`，同时需要判断后续节点是否为共享模式，如果是，需要将后续线程也唤醒。

当调用`tryRelease()`返回true或者`tryReleaseShared()`返回值大于0时，唤醒FIFO队列head的线程。

`Condition`是AQS定义的内部类`ConditionObject`，必须与独占锁一起使用，它提供了`await`、`signal`和`signalAll`方法来弥补`Object.wait()`、`Object.notify()`和`Object.notifyAll()`的缺陷。`Condition`内部也提供了一个FIFO队列。当调用`await`方法时，释放锁，将当前线程添加到`Condition`的FIFO队列中，阻塞线程，当线程唤醒时需要判断该节点是否进入了AQS的FIFO锁等待队列，如果已入队列，则进入`acquireQueued`循环获取锁，否则线程继续阻塞。当调用`signal`时，需要将`Condition`的FIFO队列的第一个线程移动到AQS的FIFO队列中，进入锁等待队列。

`ConditionObject`的`wait`方法和`signal`方法实现如下：

```java
public final void await() throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    Node node = addConditionWaiter();
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    while (!isOnSyncQueue(node)) {
        LockSupport.park(this);
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}
public final void signal() {
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    Node first = firstWaiter;
    if (first != null)
        doSignal(first);
}
private void doSignal(Node first) {
    do {
        if ( (firstWaiter = first.nextWaiter) == null)
            lastWaiter = null;
        first.nextWaiter = null;
    } while (!transferForSignal(first) &&
             (first = firstWaiter) != null);
}
final boolean transferForSignal(Node node) {
    /*
     * If cannot change waitStatus, the node has been cancelled.
     */
    if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
        return false;

    /*
     * Splice onto queue and try to set waitStatus of predecessor to
     * indicate that thread is (probably) waiting. If cancelled or
     * attempt to set waitStatus fails, wake up to resync (in which
     * case the waitStatus can be transiently and harmlessly wrong).
     */
    Node p = enq(node);
    int ws = p.waitStatus;
    if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
        LockSupport.unpark(node.thread);
    return true;
}
```



# ReentrantLock,Semaphore,ReadWriteLock,CountDownLatch,CyclicBarrier的原理 

**1).ReentrantLock**

`ReentrantLock`是可重入互斥锁，使用AQS独占锁实现。`ReentrantLock`的成员变量`Sync`实现了`AbstractQueuedSynchronizier`，内部类`FairSync`和`NonfairSync`分别实现了公平锁和非公平锁。可重入机制需要使用`setExclusiveOwnerThread`和`getExclusiveOwnerThread`方法设置和获取独占锁的线程。

非公平锁的实现方式比较简单，首先尝试抢占锁（`compareAndSetState(0, 1)`），如果抢占失败时进入等待队列；如果抢占成功则不进入等待队列，线程继续执行。

公平锁则需要检查等待队列是否存在前驱节点，如果存在，则进入等待队列，否则尝试获取锁。

从效率上来说，非公平锁高于公平锁，因为非公平锁如果抢占成功就少了入队操作，也少了线程阻塞和唤醒的操作系统API调用过程。

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

**6).StampedLock**

`ReadWriteLock`是基于悲观锁的设计，如果有线程正在读，写线程需要等待读线程释放锁后才能获取写锁，即读的过程中不允许写。

乐观锁的意思就是乐观地估计读的过程中大概率不会有写入，因此被称为乐观锁。反过来，悲观锁则是读的过程中拒绝有写入，也就是写入必须等待。显然乐观锁的并发效率更高，但一旦有小概率的写入导致读取的数据不一致，需要能检测出来，再读一遍就行。

JDK8对`ReadWriteLock`的优化是`StampedLock`，它将读锁分为乐观读锁和悲观读锁，获取乐观读锁时，实际上是获取版本号，使用者最后还需要验证在访问受保护资源时版本号是否变更，如果有变更，则获取悲观读锁，悲观读锁与写锁互斥，使用`StampedLock`时还需要注意与线程中断带来的CPU使用率高的问题[ [1] ](https://blog.csdn.net/zcl_love_wx/article/details/94856005)。

# BlockingQueue原理

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

# synchronized原理 

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

hash就是`Object.hashCode()`的返回值，age表示对象在垃圾收集过程中幸存的年龄，biased_lock表示是否是偏向锁，lock表示标志位。`Mark Word`是JVM实现同步机制的基础。程序进入临界区时需要获取的锁的结构是[ObjectMonitor](https://github.com/dmlloyd/openjdk/blob/jdk7u/jdk7u/hotspot/src/share/vm/runtime/objectMonitor.hpp)，称为监视锁。监视锁是重量级锁，因为它需要调用操作系统方法来完成，涉及到操作系统“用户态”向“内核态”的切换，需要一些开销。JVM对锁进行了一系列优化来降低使用重量级锁的开销，在没有必要使用重量级锁的场景时使用其他锁来完成同步操作，其他锁包括偏向锁、轻量级锁。使用biased_lock和lock位来表示锁的状态，锁的状态从低到高分别是无锁状态、偏向锁、轻量级锁、重量级锁。`ObjectMonitor`的3个重要字段为`_count`，它是记录获取锁的数量，因为JVM同步锁机制支持重入，每次重入，该计数器都要加1；`_WaitSet`，它是等待线程的集合，调用`Object.wait()`时线程被放入该集合中；`_cxq`，它是FILO竞争队列，应对多线程竞争锁的时候，使用CAS操作替换队列头部；`_EntryList`，cxq中的合适线程可以被放入EntryList，Wait Set中的线程被notify()之后，也会放入EntryList中，准备竞争锁。

各种锁状态的变化过程如下：

- 偏向锁

  引入偏向锁是为了在无多线程竞争的情况下尽量减少不必要的轻量级锁执行路径，因为轻量级锁的获取及释放依赖多次CAS原子指令，而偏向锁只需要在置换ThreadID的时候依赖一次CAS原子指令（由于一旦出现多线程竞争的情况就必须撤销偏向锁，所以偏向锁的撤销操作的性能损耗必须小于节省下来的CAS原子指令的性能消耗）。

  获取偏向锁的过程如下：

  （1）访问Mark Word中偏向锁的标识是否设置成1，锁标志位是否为01——确认为可偏向状态；

  （2）如果为可偏向状态，则检查线程ID是否指向当前线程，如果是，进入步骤（5），否则进入步骤（3）；

  （3）如果线程ID并未指向当前线程，则通过CAS操作竞争锁。如果竞争成功，则将Mark Word中线程ID设置为当前线程ID，然后执行（5）；如果竞争失败，执行（4）；

  （4）如果CAS获取偏向锁失败，则表示有竞争。当到达全局安全点（safepoint）时获得偏向锁的线程被挂起，偏向锁升级为轻量级锁，然后被阻塞在安全点的线程继续往下执行同步代码；

  （5）执行同步代码。

  偏向锁的释放：

  偏向锁只有遇到其他线程尝试竞争偏向锁时，持有偏向锁的线程才会释放锁，线程不会主动去释放偏向锁。偏向锁的撤销，需要等待全局安全点（在这个时间点上没有字节码正在执行），它会首先暂停拥有偏向锁的线程，判断锁对象是否处于被锁定状态，撤销偏向锁后恢复到未锁定（标志位为“01”）或轻量级锁（标志位为“00”）的状态。

- 轻量级锁

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

- 重量级锁

  重量级锁的实现在[ObjectMonitor.cpp](https://github.com/dmlloyd/openjdk/blob/jdk7u/jdk7u/hotspot/src/share/vm/runtime/objectMonitor.cpp)中完成，获取锁的方法为`void ATTR ObjectMonitor::enter(TRAPS) `，释放锁的方法为`void ATTR ObjectMonitor::exit(bool not_suspended, TRAPS) `。

  获取锁的过程如下：

  （1）设置`ObjectMonitor`的`_owner`字段为当前线程，如果设置失败时需要检查是否重入，设置成功时则表示获取锁成功；

  （2）通过自旋执行`void ATTR ObjectMonitor::EnterI (TRAPS) `方法等待锁的释放进入方法中。该方法的逻辑是

  ```
     a.当前线程被封装成ObjectWaiter对象node，状态设置成ObjectWaiter::TS_CXQ；
    
     b.在for循环中，通过CAS把node节点push到`_cxq`列表中，同一时刻可能有多个线程把自己的node节点push到`_cxq`列表中；
    
     c.node节点push到`_cxq`列表之后，通过自旋尝试获取锁，如果还是没有获取到锁，则通过park将当前线程挂起，等待被唤醒；
    
     d.当该线程被唤醒时，会从挂起的点继续执行，通过`ObjectMonitor::TryLock`尝试获取锁。
  ```

  其本质就是通过CAS设置monitor的_owner字段为当前线程，如果CAS成功，则表示该线程获取了锁，跳出自旋操作，执行同步代码，否则继续被挂起；

  当某个持有锁的线程执行完同步代码块时，会进行锁的释放，给其它线程机会执行同步代码，在HotSpot中，通过退出monitor的方式实现锁的释放，并通知被阻塞的线程，具体实现位于`ObjectMonitor::exit`方法中。

  释放锁的过程如下：

  （1）如果是重量级锁的释放，monitor中的_owner指向当前线程，即THREAD == _owner；

  （2）根据不同的策略（由QMode指定），从cxq或EntryList中获取头节点，通过`ObjectMonitor::ExitEpilog`方法唤醒该节点封装的线程，唤醒操作最终由unpark完成；

  （3）被唤醒的线程，继续执行monitor的竞争；

为了减少重量级锁的操作，引进了偏向锁和轻量级锁。在某些场景下还可以对获取锁的过程做进一步优化，如下：

- 适应性自旋

  当线程在获取轻量级锁的过程中执行CAS操作失败时，是要通过自旋来获取重量级锁的。自旋的意思循环检查是否可以获取重量级做。JVM内部根据运行时信息决定自旋的次数，即循环次数。适应性自旋，简单来说就是线程如果自旋成功了，则下次自旋的次数会更多，如果自旋失败了，则自旋的次数就会减少。

- 锁粗化

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

- 锁消除

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

除了上面JVM对锁的优化，在程序端还可以使用锁分段（如ConcurrentHashMap实现）和锁分离（ReadWriteLock实现）的技术提高并发。

# 锁的升级和降级

锁降级在`ReentrantReadWriteLock`中的意思是从写锁降级为读锁，但是`ReentrantReadWriteLock`不能从读锁升级为写锁。

锁降级在JVM中是指重量级锁降级为轻量级锁或者偏向锁。

锁升级在JVM中是指偏向锁升级为轻量级锁，轻量级锁升级为重量级锁。

# 多种方式实现生产者和消费者模式

- wait/notify

- ReentrantLock/Condition

- BlockingQueue

- Semaphore

- PipedInputStream/PipedOutputStream

  该方法不是基于线程同步完成，因此只能满足于一个生产者和一个消费者。原理是先创建一个管道输入流和管道输出流，然后将输入流和输出流进行连接，用生产者线程往管道输出流中写入数据，消费者在管道输入流中读取数据，这样就可以实现了不同线程间的相互通讯，但是这种方式在生产者和生产者、消费者和消费者之间不能保证同步，也就是说在一个生产者和一个消费者的情况下是可以生产者和消费者之间交替运行的，多个生成者和多个消费者者之间则不行

延伸阅读：

* [Java实现生产者和消费者的5种方式](https://juejin.im/entry/596343686fb9a06bbd6f888c)



# RingBuffer

RingBuffer是一种更高效的数据并发访问的保护机制，它不使用CAS实现锁机制，而是使用CPU缓存一致性来实现高效的一个线程读一个线程写的并发操作。CAS低效的原因是除了存在CPU缓存失效还有一个活锁的问题，即当CAS会放在一个循环内，重试去设置期望的值。

高性能消息框架[Disruptor](https://github.com/LMAX-Exchange/disruptor)使用的就是RingBuffer[ [2] ](https://www.jianshu.com/p/3da1bd3b29cd)。