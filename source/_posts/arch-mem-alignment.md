---
title: 计算机体系结构——内存对齐原理
date: 2020-12-18 16:58:37
tags: 计算机体系结构
---

# 问题

在使用 C 语言定义一个结构体时，使用 `sizeof` 计算结构体的字节大小时与理论计算的不一致。比如：

```c
#include <stdio.h>

typedef struct {
	int a;
	char b;
	int c;
} A;

int main(int argc, char *args[])
{
	A a;
	printf("%d", sizeof(A));
}
```

结构体 `A` 的理论大小应该是 4 + 1 + 4 = 9 字节，当我们执行 `sizeof(A)` 或者 `sizeof(a)` 时，返回值是 12。

# 原因

为了提高程序读写内存的效率，编译器将字段 b 多填充了 3 个字节，这3个字节都是 0。填充空白就是内存对齐，结构体 `A` 是按照 4 字节对齐。一般来说，结构体会按照内存宽度最长的那个字段来做字节对齐。

为什么要字节对齐呢？

# 原理

对于大多数处理器，包括兼容 x86 协议的处理器，允许一个内存总线周期（one memory bus cycle）访问的字节宽度是 1 字节、2 字节、4 字节，而且访问起始地址必须如下图<sup>[ [1] ](https://microchipdeveloper.com/32bit:mz-arch-memory-alignment)</sup>左侧所示：

![内存对齐的数据传输](/images/memory-alignment.png "内存对齐的数据传输")

简言之：

![一次读取](/images/memory-alignment-one-cycle-access.png "一次读取")

下面的方式不能一次读取：

![多次次读取](/images/memory-alignment-more-cycle-access.png "多次读取")

如果读取的数据不是字节对齐，处理器需要多次访问内存，然后做移位操作，合并数据。正如 Intel 的技术手册[《Intel® 64 and IA-32 Architectures Software Developer’s Manual Volume 1: Basic Architecture》](https://software.intel.com/content/www/us/en/develop/download/intel-64-and-ia-32-architectures-software-developers-manual-volume-1-basic-architecture.html)第4.1.1节“Alignment of Words, Doublewords, Quadwords, and Double Quadwords”所说的：

*However, to improve the performance of programs, data  structures (especially stacks) should be aligned on natural boundaries whenever possible. The reason for this is  that the processor requires two memory accesses to make an unaligned memory access; aligned accesses require  only one memory access. A word or doubleword operand that crosses a 4-byte boundary or a quadword operand  that crosses an 8-byte boundary is considered unaligned and requires two separate memory bus cycles for access.*

通过上面的介绍，我们可以知道数据结构 `B` 占用的内存大小是 3。

```c
#include <stdio.h>

typedef struct {
	char a;
	char b;
	char c;
} B;

int main(int argc, char *args[])
{
	B b;
	printf("%d", sizeof(B));
}
```



# 优点

内存对齐有 3 个优点<sup>[ [2] ](https://stackoverflow.com/questions/381244/purpose-of-memory-alignment)</sup>：

* 提高访问速度
* 原子读写
* 提高内存访问范围：*For any given address space, if the architecture can assume that the 2 LSBs are always 0 (e.g., 32-bit machines) then it can access 4 times more memory (the 2 saved bits can represent 4 distinct states), or the same amount of memory with 2 bits for something like flags.*这种取巧方式也用在JVM的偏向锁存储`JavaThread*`线程指针，通过将线程对象按照 128 字节对齐，可以腾出7位的空间存储 `age` 和 `epoch `<sup>[ [3] ](http://hg.openjdk.java.net/jdk8/jdk8/hotspot/file/87ee5ee27509/src/share/vm/oops/markOop.hpp)</sup>。