---
title: 彻底搞懂零拷贝
tags: 网络
toc: true
date: 2023-09-10 16:09:19
---


*本文主要参考 [Zero Copy I: User-Mode Perspective](https://www.linuxjournal.com/article/6345?page=0,0)，作为一个笔记。*

# 什么是零拷贝

为了更好的理解问题的解决方案，我们首先需要理解问题的本身。我们先看一段代码，它是读取一个文件，然后将文件内容通过网络的方式发送出去。

*代码块-1*
```c
read(file, tmp_buf, len);
write(socket, tmp_buf, len);
```

站在我们的角度看，read 函数将文件内容读取出来放在了 `tmp_buf` 中，然后又将 `tmp_buf` 的内容通过 write 将其以网络形式发送出去。这两个系统调用至少涉及 4 次数据复制、多次用户/内核上下文切换。请看图-1。

![代码块-1 涉及的系统调用](https://www.linuxjournal.com/files/linuxjournal.com/linuxjournal/articles/063/6345/6345f1.jpg)
*图-1 代码块-1 涉及的系统调用*

图-1的底部显示的是复制操作，顶部显示的是上下文切换。

对图-1每个步骤的解释如下。

* 第一步，调用 `read` 函数时，由于它是系统接口调用，导致一次上下文切换，代码逻辑从用户模式切换到内核模式。在内核模式下，操作系统通过 DMA 的方式，将文件内容复制到内核的缓冲区中，该缓冲区位于内核地址空间中。注意：1）DMA 复制是不需要 CPU 参与，直接由磁盘硬件来完成；2）内核缓冲区不是 `tmp_buf` 所在的位置，`tmp_buf` 是用户程序声明的缓冲区，位置在用户地址区间中。
* 第二步，内核将数据从内核缓冲区复制到用户空间的缓冲区中（即 `tmp_buf` 中），然后 `read` 系统调用函数返回。此时触发一次上下文切换，从内核模式返回到用户模式。继续执行用户代码。该过程的复制是需要 CPU 参与的。
* 第三步，应用程序继续调用 `write` 函数，它也是系统接口调用，导致一次上下文切换，代码逻辑从用户模式切换到内核模式。由于涉及到网络发送，内核会将用户模式下的数据（即 `tmp_buf` 中的数据）复制到 socket 缓冲区（即 [sk_buff](https://docs.kernel.org/networking/skbuff.html)），socket 缓冲区的数据最后会被发送出去。
* 第四步，函数返回时，从内核模式返回到用户模式。注意，此时数据可能还没有发送出去，系统仅仅是将数据放在一个发送队列。具体可以参考 [25 张图，一万字，拆解 Linux 网络包发送过程](https://mp.weixin.qq.com/s/wThfD9th9e_-YGHJJ3HXNQ) 的说明。发送时的数据复制是由 DMA 完成的，不需要 CPU 参与。

# 第一步改进：mmap

图-1中，`tmp_buf` 能否减少一次拷贝呢？也就是不要复制到用户态，直接在内核里面完成复制。答案是 `mmap`。

改进后的代码如下。

*代码块-2*
```c
tmp_buf = mmap(file, len);
write(socket, tmp_buf, len);
```

代码块-2 对应的内存复制流程如图-2所示。

![调用 mmap](https://www.linuxjournal.com/files/linuxjournal.com/linuxjournal/articles/063/6345/6345f2.jpg)
*图-2 调用 mmap*

图-2所示，2 次系统调用和 4 次上下文切换一次都没有少，还是一样，但是内存复制有点不一样。按步骤分解如下。

* 第一步，调用 `mmap` 后，内核会将文件数据复制到缓冲区，这个缓冲区是共享的，用户空间和内核空间都可以访问。这一步已经不像之前那样，需要将数据复制到用户空间的缓冲区中，而是直接返回共享缓冲区的地址。
* 第二步，调用 `write` 函数后，内核直接从共享缓冲区中将数据复制到 sk buff 中。
* 第三步，最后一步之前一样，数据通过 DMA 的方式将数据从 sk buff 从复制到网卡并发送出去。

使用 `mmap` 可以减少一次数据拷贝，也就是将内核缓冲区的数据拷贝到用户空间的缓冲区中，可以提高效率。但是使用 mmap 也是有代价的。代价是并发访问问题导致进程崩溃，当 `write` 访问文件的内存映射地址区间时，文件内容可能被其他任务修改过了，这会导致地址区间也发生变化，导致内存越界，发出 SIGBUS 错误，然后结束进程。不过这也有解决办法，使用 file leasing，这不是本文的重点，就不说了。

# 再次改进：sendfile

为了提高文件和网络之间的传输效率，内核从 2.1 开始引入了 `sendfile` 系统调用。此时直接一行代码就可以搞定。

*代码块-3*
```c
sendfile(socket, file, len);
```

![使用 sendfile](https://www.linuxjournal.com/files/linuxjournal.com/linuxjournal/articles/063/6345/6345f3.jpg)
*图-3 使用 sendfile*

`sendfile` 相比 `mmap` 来说，并没有减少内存复制的操作，但是减少了系统调用，进而减少了上下文切换。

步骤分解如下：
* 第一步，`sendfile` 系统调用之后后，内核将文件内容通过 DMA 的方式复制到内核缓冲区中。然后文件的内容再通过 CPU 被复制到 sk buff 中。
* 第二步，最后一步跟之前一样，数据通过 DMA 的方式将数据从 sk buff 从复制到网卡并发送出去。

# 终极大招：DMA Gather

通过 `mmap` 和 `sendfile`，我们发现与 sk buff 有关的内存复制还存在，并没有消除。

从 Linux 内核 2.4 开始，通过 DMA Gather 技术，可以做到真正的零拷贝。

使用的代码还是一行 `sendfile` 调用，不过内核做了很多事情。

*代码块-4*
```c
sendfile(socket, file, len);
```

![由硬件支持的 gather 操作可以将聚合多个内存地址的数据，减少其他复制](https://www.linuxjournal.com/files/linuxjournal.com/linuxjournal/articles/063/6345/6345f4.jpg)
*图-4 由硬件支持的 gather 操作可以将聚合多个内存地址的数据，减少其他复制*

图-4中，sk buff 不在复制数据，而是仅仅记录了数据的地址和长度，然后交给 DMA。