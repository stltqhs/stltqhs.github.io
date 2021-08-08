---
title: Java后端开发 - 容器类
date: 2018-10-13 08:14:38
tags: java
toc: true
---

# HashMap，LinkedHashMap，TreeMap，Hashtable的实现方式 

## HashMap

HashMap由数组和链表组成，元素的类型是Map.Entry，其`table[]`字段就是数组类型，而Map.Entry.next字段组成链表。使用链表的原因是存在hash碰撞，即不同key计算出来的hash值一样。HashMap不是线程安全的，也不保证元素插入的顺序。HashMap可以接受`null`键。

  当调用`HashMap.put()`方法时，会先判断`key`是否为`null`，如果是，`value`需要放在`table[0]`的链表中。如果`key`已经存在，替换旧值，否则放在链表头部（代码：`table[indexFor(hash,table.length)] = new Entry<>(hash, key, value, e);size++`）。

  当调用`HashMap.get()`方法时，根据`key`的hash值，通过`indexFor(hash,table.length)`计算索引位置（如果key为`null`，索引位置是0），从链表头部开始搜索，当元素e与`key`满足：`e.hash == hash && ((k = e.key) == key || (key != null && key.equals(k))`时返回e.value，否则通过`e.next`获取下一个元素继续比较。

  每次调用`addEntry`添加一个新元素时，都会通过`(size >= threshold) && (null != table[bucketIndex])`（其中`threshold = (int) Math.min(capacity * loadFactor, MAXIMUM_CAPACITY + 1);`）来判断是否需要扩容，如果需要，则调用`resize(2 * table.length)`创建一个新的table，大小为原来的两倍，通过`transfer`方法将旧数据移动到新的table上来，此时会重新计算这些元素的索引位置，称为重hash。

  当构造一个HashMap时可以指定table数组大小（默认为16），但HashMap要求table数组实际大小必须是2的幂次，即必须是偶数，而不是奇数，原因是计算hash索引的`indexFor()`方法使用的是位运算，而不是模运算，代码是`h & (length-1)`。当length为偶数时，length-1就是奇数，最低位恒为1，与h按位与运算后，值可以是偶数，也可以是奇数。而当length为奇数时length-1就是偶数，最低位恒为0，与h按位与运算后，值是偶数，不可能存在奇数的可能，此时碰撞的概率增加。



  最坏情况下，所有的key通过`indexFor()`计算的值都相同，此时HashMap退化成链表，时间复杂度从O(1)提高到O(n)，Java8对此进行优化，将最坏情况下的时间复杂度降低到O(logn)。原理是当链表长度大于TREEIFY_THRESHOLD时（默认为8），会将链表转换为二叉树，此时查找时间复杂度就是O(logn)。

## LinkedHashMap

LinkedHashMap解决HashMap不保证插入顺序的问题。实现方式是在HashMap的基础上添加一个双向链表（`Entry.after`和`Entry.before`组成双向链表），用来记录插入顺序。LinkedHashMap与HashMap一样可以接受`null`键。

  `LinkedHashMap.put`基本与`HashMap.put`操作一致，只是`LinkedHashMap.put`多了一个将新元素添加到双向链表的尾部（`Entry.addBefore(header)`），如果不是新元素，则调用`Entry.recordAccess()`方法记录访问操作（如果`accessOrder`为`false`则什么也不做，默认为`false`）。

  `LinkedHashMap.get`基本与`HashMap.get`操作一致，如果能找到元素，则调用`Entry.recordAccess`方法记录访问操作。

  `accessOrder`可以用来控制是按照插入顺序还是访问顺序迭代元素，默认是插入顺序，当`accessOrder`为`true`时表示按照访问顺序迭代元素，此时`Entry.recordAcess()`的操作就是将元素添加到双向链表的尾部，表示该元素刚刚被访问过。在`addEntry`方法中添加一个新元素时，会调用`removeEldestEntry(header.after)`来判断是否进行删除长时间未被访问的记录，如果是长时间未被访问则调用`removeEntryForKey(header.after.key)`删除这些记录。不过`removeEldestEntry()`默认返回`false`，该方法可以被重写，用来实现简单的`LRU`算法。

  `LinkedHashMap.resize()`与`HashMap.resize()`一致，但是`LinkedHashMap.transfer()`重写了`HashMap.transfer`方法，重hash时直接遍历双向链表即可。

## TreeMap

TreeMap按照key的顺序排序存储，使用的数据结构时“红黑树”。由于是按照key的顺序排序，所以如果没有通过构造函数指定`Comparator`时，key就需要实现`Comparable`接口。TreeMap不接受`null`键。



  当调用`TreeMap.put()`时，以跟节点开始寻找与key相等的节点，如果找到，设置新值，如果不存在，添加新节点，调整树结构。

## Hashtable

Hashtable与HashMap基本一样，出来Hashtable不接受`null`键也不接受`null`值，且是线程安全的（因为put和get方法都是同步方法）。

# ArrayList和LinkedList实现方式以及SubList实现方式 

## ArrayList

ArrayList是一个动态数组实现的线性表，其容量可以自动增长，新容量为`(原始容量*3)/2 + 1`。`elementData`是用来存放新增的元素，`size`记录元素个数。ArrayList查找元素使用`indexOf(E)`，原理是遍历所有元素，所以效率比较低。由于其为数组，可以使用索引快速访问（实现了`RandomAccess`接口）。当删除元素时，需要将后面的元素全部向前移动，效率比较低。ArrayList不是线程安全的。

## CopyOnWriteArrayList

CopyOnWriteArrayList是线程安全的ArrayList，通过“写时复制”来提高“读多写少”的并发操作场景。每次对`array`（定义为`private volatile transient Object[] array;`）修改时，首先获取独占锁（`ReentrantLock`），复制数组，新数组的大小为`len+1`，然后赋值给`array`，最后释放锁。它的迭代器也是通过将`array`赋值给`snopshot`，然后迭代`snopshot`的内容，而且`snopshot`被定义为`final`类型。迭代器不支持`remove`操作，或者说所有修改操作都不支持。

## LinkedList

LinkedList是一个双向链表实现的线性表。使用索引访问时需要从header开始查找索引位置，效率比较低。插入元素时只需要修改元素`next`和`previous`指针即可，不需要移动元素，效率高。

## ArrayList.subList

ArrayList.subList的返回值类型是ArrayList内部类SubList，该对象表示ArrayList的一个视图，所以修改SubList会影响到ArrayList。如果ArrayList被修改（`modCount`值改变）时，SubList就无效了，因为SubList记录的`modCount`和ArrayList的`modCount`不一致，任何操作都会报`ConcurrentModificationException`。

## Arrays.asList(T...a)

  Arrays.asList使用适配器模式构建一个Arrays.ArrayList类型的数据，不支持添加、删除调整，原数组的修改将会影响到该Arrays.ArrayList。

## Vector

与ArrayList基本一致，但是Vector是线程安全的（方法为`synchronized`），而且扩容时新容量的大小与`capacityIncrement`有关，如果`capacityIncrement`大于0，则新容量的大小为`oldCapacity + capacityIncrement`，否则为`oldCapacity * 2`。

# HashSet实现方式 

HashSet是基于HashMap实现的，`HashSet.add(E e)`内部是通过`map.put(e, PRESENT)`实现，map是HashMap类型，PRESENT为一个Object类型，用来作为HashMap的值对象。

# TreeSet实现

TreeSet基于TreeMap实现。

# Set,Queue,List,Map,Stack 

- Set

  集合，不存在重复元素。当`e1.equals(e2)`时表示e1和e2重复。最多只有一个`null`元素。

- Queue

  FIFO队列，`offer(e)`添加元素，`poll()`移除元素，`peek()`查看元素。

- Deque

  “double ended queue”即Deque，表示双端队列，可以在线性表两端进行添加和删除元素。

- List

  线性表，可以存在重复的元素。

- Map

  Key-value pair

- Stack

  LIFO队列

# ConcurrentHashMap的实现方式 

## JDK1.7实现

ConcurrentHashMap使用`锁分段`的方式来实现高效的HashMap，使用`不变性`和`volatile`来减少加锁操作，提高线程并发。

ConcurrentHashMap包含n个segment（segment是ReentrantLock的子类，初始时n为16，且n为2的幂次），segment的几个重要成员变量定义如下：

```java
transient volatile int count;
transient int modCount;
transient int threshold;
transient volatile HashEntry<K,V>[] table;
final float loadFactor;
```

每个segment由HashEntry组成，HashEntry的几个重要的成员变量的声明如下：

```java
final K key;
final int hash;
volatile V value;
final HashEntry<K,V> next;
```

ConcurrentHashMap数据结构如下图：

```text
+-------------+         +------------+
| segments[0] |         | entries[0] |
+-------------+         +------------+         +------+-+    +-----+-+
| segments[1] | ------> | entries[1] | ------> |      |-+--> |     | |
+-------------+         +------------+         +------+-+    +-----+-+
| segments[2] |
+-------------+
|     ...     |
+-------------+
| segments[n] |
+-------------+
```

不管是put还是get操作，都需要通过key计算segment的位置。由于put操作需要改segment结构，所以put操作需要加锁，加锁方法如下：

```java
HashEntry<K,V> node = tryLock() ? null :
        scanAndLockForPut(key, hash, value);
```

首先尝试获取锁（`tryLock()`），当获取锁失败时，则会进入`scanAndLockForPut`方法，该方法会自旋获取锁。自旋的实现是`while(tryLock())`，当自旋次数超过了`MAX_SCAN_RETRIES`时，则使用阻塞锁获取（`lock()`）。

由于`HashEntry.next`是`final`类型，链表的`put`操作需要在表头添加一个新元素，该操作不影响读取或者对链表的遍历操作，因此读取可以不用加锁（除了`value`为`null`时需要加锁再读一次），`remove`操作是复制被删除节点的前驱节点构造新链表，同时将被删除节点的`next`值复制到该链表的尾节点的`next`，该操作不影响读取或者对链表的遍历操作。对于`size()`操作，先使用不加锁的方式计算每个segment的count，同时比较计算前和计算后的`modCount`值是否改变，如果改变，表示计算期间存在修改情况，此时再加锁计算。

## JDK 1.8实现

JDK1.8的ConcurrentHashMap的实现抛弃了1.7使用的分段锁，改用“CAS+synchronized“，`Node.next`字段为`volatile`类型，而不是1.7的`final`类型。

```text
   table
+---------+        
| Node[0] |        
+---------+         +------+-+    +-----+-+
| Node[1] | ------> |      |-+--> |     | |
+---------+         +------+-+    +-----+-+
| Node[2] |
+---------+
|   ...   |
+---------+
| Node[n] |
+---------+
```

当`put`一个Key-Value时，先检查Node[n]是否为`null`，如果为`null`则使用`casTabAt`将元素放在头部。如果不为`null`，则`synchronized(Node[n])`将元素加入到队尾，此时可能对链表转换为红黑树。

# Collections.synchronizedMap实现方式 

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



# hashCode()和equals()方法的作用 

`hashCode`和`equals`方法在`Object`类中定义，其中`hashCode`方法为`native`方法，`equals`方法定义如下：

```java
public boolean equals(Object obj) {
        return (this == obj);
    }
```

在不重写`equals`方法时，`equals`和`==`操作符是等价的。

`hashCode`方法只有在`hash`表才用到，比如`HashSet`，`HashMap`等，此时`hashCode`和`equals`方法存在关联，因为查找元素时先获得对象的`hashCode`，定位`hash`索引，然后调用`equals`方法查找对象。

重写`equals`方法需要遵循如下要求：

- **自反性**（reflexive）。对于任意不为`null`的引用值x，`x.equals(x)`一定是`true`。
- **对称性**（symmetric）。对于任意不为`null`的引用值`x`和`y`，当且仅当`x.equals(y)`是`true`时，`y.equals(x)`也是`true`。
- **传递性**（transitive）。对于任意不为`null`的引用值`x`、`y`和`z`，如果`x.equals(y)`是`true`，同时`y.equals(z)`是`true`，那么`x.equals(z)`一定是`true`。
- **一致性**（consistent）。对于任意不为`null`的引用值`x`和`y`，如果用于equals比较的对象信息没有被修改的话，多次调用时`x.equals(y)`要么一致地返回`true`要么一致地返回`false`。
- 对于任意不为`null`的引用值`x`，`x.equals(null)`返回`false`。

重写`hashCode`方法时需要遵循如下要求：

- 在一个Java应用的执行期间，如果一个对象提供给equals做比较的信息没有被修改的话，该对象多次调用`hashCode()`方法，该方法必须始终如一返回同一个integer。

- 如果两个对象根据`equals(Object)`方法是相等的，那么调用二者各自的`hashCode()`方法必须产生同一个`int`结果。



  基于上述两个性质，一般是重写`equals`方法就要重写`hashCode`方法。

# Arrays.sort()和Collections.sort()的实现方式

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
