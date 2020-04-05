---
title: Java后端开发 - 网络
date: 2018-10-13 08:17:59
tags: java
---



# HTTP

HTTP协议规范：

* 1996年的HTTP1.0的[RFC1945](https://tools.ietf.org/html/rfc1945)
* 1999年的HTTP1.1的[RFC2616](https://tools.ietf.org/html/rfc2616)
* 2015年的HTTP/2的[RFC7540](https://tools.ietf.org/html/rfc7540)和[RFC7541](https://tools.ietf.org/html/rfc7541)

## HTTP1.0的问题

HTTP协议的基本特点是“一来一回”，客户端发起一个TCP连接，在连接上面发一个HTTP Request到服务器，服务器返回一个HTTP Response，然后关闭连接。每个请求都要重复这样的操作，存在以下问题：

* 频繁的连接建立与关闭造成的**性能问题**，该问题使用Keep-Alive机制解决
* 服务器推送问题，服务器无法在客户端没有请求的情况下主动向客户端推送消息

## HTTP1.1

### 连接复用与Chunk机制

HTTP1.1默认启用`Connection: Keep-Alive`属性来达到连接复用。HTTP1.0的`Content-Length`属性用来解决Keep-Alive启用时让客户端判断是否数据已经传输完毕，导致的问题是在动态语言生成的HTTP页面内容时，服务端需要在内存中渲染然后再计算长度，效率低。HTTP1.1的解决办法是使用`Transfer-Encoding: chunked`机制，数据分块传输。比如：

```text
HTTP/1.1 200 OK
Content-Type: text/plain
Transfer-Encoding: chunked
0B
Hello World
05
number
0
```

分块传输时也需要告知每块的字节大小。

### Pipeline与Head-of-line Blocking问题

在连接复用中，请求是串性执行，并发度不高。HTTP1.1引入Pipeline机制，在同一个TCP连接上，可以在一个请求发出后响应未发回来之前，发送下一个、再下一个请求，可提高请求的处理效率。然而，Piepline的请求-响应是先近先出，即响应第二个请求时必须发生在响应完第一个请求之后，如果第一个请求耗时长，则第二个请求将会阻塞，这种行为称为Head-of-line Blocking，即“队头阻塞”。由于“队头阻塞”，Pipeline机制一般被关闭状态。

## HTTP/2

### 与HTTP1.1兼容

使用一个转换层SPDY（即HTTP/2）来兼容HTTP1.1协议。

```text
+----------+
| HTTP 1.1 |
+----------+
|  HTTP/2  |
|  (SPDY)  |
+----------+
|    TCP   |
+----------+
```

### 二进制分帧

二进制分帧是HTTP/2为了解决HTTP1.1的“队头阻塞”问题所设计的核心特征。原理是将一个请求的字符内容转换为二进制，并且分成多个帧来发送。这样一来，多个请求被切分后可乱序发送，接收响应时顺序也不确定，从HTTP协议层解决“对头阻塞”。但由于TCP是“先进先出”，它依然存在“队头阻塞”。要彻底解决“队头阻塞”就是不使用TCP，使用Google的QUIC。

Pipeline和HTTP/2请求响应：

![Pipeline和HTTP/2请求响应](/images/http_pipeline.jpg "Pipeline和HTTP/2请求响应")

### 头部压缩

HTTP/2使用HPACK协议和对应的算法对HTTP头部压缩。

# SSL/TLS

明文传输和对称加密传输都不安全，需要使用RSA非对称加密传输数据。RSA分为公钥和私钥，服务端存储私钥，而公钥存储在客户端。为了防止数据被篡改，需要签名，为了防止中间人攻击，需要CA数字证书。

双向非对称加密：

![双向非对称加密](/images/rsa_bidirection.jpg "双向非对称加密")

单向非对称加密：

![单向非对称加密](/images/rsa_direction.jpg "单向非对称加密")

中间人攻击：

![中间人攻击](/images/rsa_mediator_attack.jpg "中间人攻击")

数字证书和CA

![数字证书和CA](/images/rsa_ca.jpg "数字证书和CA")

SSL/TLS四次握手：

![SSL/TLS四次握手](/images/rsa_tls_handshake.jpg "SSL/TLS四次握手")

# TCP

TCP协议规范是[RFC793](https://tools.ietf.org/html/rfc793)。

TCP通过消息顺序编号+客户单重发+服务端顺序ACK实现了客户端到服务器的数据包的不重、不漏、时序不乱。

## 解决不丢问题:ACK+重发

网络的不稳定造成丢包是肯定发生的，解决不丢只有**重发**。服务端每收到一个包，都要对客户端进行确认，如果客户端在超时时间内未收到ACK，则重发。客户端发送时需要对每个数据包编号，号码从小到大单调递增。当服务端收到数据包确认时回复ACK=n表示所有小于或等于n的数据包都已经收到了，可以不用逐一回复。

##  解决不重复的问题

由于服务端回复确认信息时是**顺序ACK**，比如回复ACK=6表示小于或等于6的数据包都已经收到，下次收到5的数据包时，能够判断5这个数据包已经重复。

## 解决时序错乱的问题

假设服务队收到了数据包1、2、3，回复客户端ACK=3，之后收到5、6、7，而数据包4迟迟没有收到，此时服务端会将数据包5、6、7暂存，此时不回复ACK，等待数据包4的到来。如果超时后客户端仍然没有收到ACK，则重发数据包4、5、6、7。当客户端收到数据包4时，就可以回复ACK=7，同时数据包5、6、7重复，丢弃即可。

## 三次握手

TCP建立连接的过程：

![TCP三次握手](/images/tcp_handshake.jpg "TCP三次握手")

* 图中的ACK的意思和前面所讲的稍微有些差异：前文中的ACK=7表示小于或等于7的数据包都收到了；这里的ACK=x+1表示小于或等于x的数据包都收到了，接下来要接收x+1的数据包
* seq=x表示发出去的数据包编号是x，因为TCP是全双工通信，为了优化传输，将seq=y和ACK=x+1合并一个数据包发出去。

为什么需要三次握手？由于网络的两将军问题[ [1] ](https://en.wikipedia.org/wiki/Two_Generals%27_Problem)，使用三次握手恰好可以保证客户端和服务端对自己的发送、接收能力做了一次确认，保证了自己发送的数据对方可以接收。

## 四次挥手

TCP关闭连接的过程：

![TCP四次挥手](/images/tcp_close.jpg "TCP四次挥手")

* 为什么需要四次挥手？因为TCP是全双工通信，第一次和第二次，TCP连接还处于Half-Close状态，需要等到第三次和第四次连接才会处于完全的CLOSE状态
* 为什么服务端收到客户端的FIN请求后，需要分两次发ACK和FIN？服务端需要通知上层应用做清理工作，同时因为下一个原因
* 为什么需要一起进入TIME_WAIT状态？由于网络数据包传输会有延时，当双方都进入CLOSE状态时，仍然可能有数据包还在网络上“闲逛”，此时如果收到了这些闲逛的数据包，丢掉即可，但是问题是连接可能重开，导致闲逛的数据包当作新打开连接的数据包。在整个TCP/IP网络上，定义了一个值叫作MSL（Maximum Segment Lifetime），任何一个IP数据包在网络上逗留的最长时间是MSL，这个默认值是120s。意味者一个数据包必须最多在MSL时间内从源点传输到目的地，如果超出这个时间，中间的路由节点就会把该数据包丢弃。有了这个限定之后，一个连接保持在TIME_WAIT状态，再等待2xMSL的时间进入CLOSE状态，就可以完全避免旧的连接上面存在闲逛的数据包串到新的连接上。
* 为什么是2倍的MSL？因为网络的两将军问题，第四次发送的数据包，服务端是否收到，客户端是不知道的。服务器采取的方法是在无法收到第四次数据包的情况下重发第三次的数据包，客户端重新收到第三次数据包，再次发送第四次数据包。第四次数据包的传输时间+服务器重新发送第三次数据包的时间，最长是两倍的MSL，所以要让客户端在TIME_WAIT状态等待2倍的MSL
* 为什么服务端收到第四次数据包后，马上就进入CLOSE状态，而不用等待两倍的MSL？因为任何一个连接都是4元组，由（客户端IP、客户端Port、服务端IP、服务队Port）组成，在客户端处于TIME_WAIT状态后，意味者这个连接需要到两倍的MSL时间之后才能重新启用，服务端即使想立马使用也无法实现。

## 状态机

```text
                              +---------+ ---------\      active OPEN
                              |  CLOSED |            \    -----------
                              +---------+<---------\   \   create TCB
                                |     ^              \   \  snd SYN
                   passive OPEN |     |   CLOSE        \   \
                   ------------ |     | ----------       \   \
                    create TCB  |     | delete TCB         \   \
                                V     |                      \   \
                              +---------+            CLOSE    |    \
                              |  LISTEN |          ---------- |     |
                              +---------+          delete TCB |     |
                   rcv SYN      |     |     SEND              |     |
                  -----------   |     |    -------            |     V
 +---------+      snd SYN,ACK  /       \   snd SYN          +---------+
 |         |<-----------------           ------------------>|         |
 |   SYN   |                    rcv SYN                     |   SYN   |
 |   RCVD  |<-----------------------------------------------|   SENT  |
 |         |                    snd ACK                     |         |
 |         |------------------           -------------------|         |
 +---------+   rcv ACK of SYN  \       /  rcv SYN,ACK       +---------+
   |           --------------   |     |   -----------
   |                  x         |     |     snd ACK
   |                            V     V
   |  CLOSE                   +---------+
   | -------                  |  ESTAB  |
   | snd FIN                  +---------+
   |                   CLOSE    |     |    rcv FIN
   V                  -------   |     |    -------
 +---------+          snd FIN  /       \   snd ACK          +---------+
 |  FIN    |<-----------------           ------------------>|  CLOSE  |
 | WAIT-1  |------------------                              |   WAIT  |
 +---------+          rcv FIN  \                            +---------+
   | rcv ACK of FIN   -------   |                            CLOSE  |
   | --------------   snd ACK   |                           ------- |
   V        x                   V                           snd FIN V
 +---------+                  +---------+                   +---------+
 |FINWAIT-2|                  | CLOSING |                   | LAST-ACK|
 +---------+                  +---------+                   +---------+
   |                rcv ACK of FIN |                 rcv ACK of FIN |
   |  rcv FIN       -------------- |    Timeout=2MSL -------------- |
   |  -------              x       V    ------------        x       V
    \ snd ACK                 +---------+delete TCB         +---------+
     ------------------------>|TIME WAIT|------------------>| CLOSED  |
                              +---------+                   +---------+

                      TCP Connection State Diagram
```

摘抄至[RFC793](https://tools.ietf.org/html/rfc793)第22页的TCP连接状态图。

## QUIC

QUIC（Quick UDP Internet Connection）是由Google公司提出的基于UDP协议的多路并发传输协议。QUIC可以解决TCP的“队头阻塞”问题。QUIC包含的特征有：

* 不丢包（Raid5和Raid6）
* 更少的RTT
* 连接迁移

# 网络IO模型

## IO多路复用

早起服务端网络编程模型中，使用一个线程启动一个连接LISTEN，等待新的客户端连接请求，当收到请求后，创建一个新线程，该线程阻塞式的读取客户端发送来的消息和发送消息到客户端。这种方式存在严重的性能问题，由于线程数量太多时，操作系统将会花大部分时间在线程上线文切换中，系统资源很容易耗光，不能有效处理客户端请求，造成著名的[C10K](http://www.kegel.com/c10k.html)问题。

为了解决上面的问题，使用IO多路复用，用一个线程监听所有的客户端连接状态，检查是否有数据请求。IO多路复用也称为事件驱动IO。

### select/poll

[select](http://man7.org/linux/man-pages/man2/select.2.html)函数声明：

```c
 int select(int nfds, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout);
```

* 因为fd是一个int值，所以fd_set其实是一个bit数组，每一位表示一个fd是否有读事件或者写事件
* 第一个参数表示fd个数，是readfds或者writefds的下标的最大值+1.因为fd从0开始，+1才表示个数
* 返回结果还在readfds和writefds里面，操作系统会重置所有的bit位，告知应用程序到底哪个fd上面有事件，应用程序需要自己从0到maxfds-1遍历所有的fd，然后执行响应的read/write操作
* 每次当select调用返回后，在下一次调用之前，要重新维护readfds和writefds

**select受FD_SETSIZE的限制**。

[poll](http://man7.org/linux/man-pages/man2/poll.2.html)函数声明：

```c
int poll (struct pollfd *fds, unsigned int nfds, int timeout);
struct pollfd {
    int fd; /* file descriptor */
    short events; /* requested events to watch */
    short revents; /* returned events witnessed */
};
```

通过上面的函数会发现，select、poll每次调用都需要应用程序把fd的数组传进去，这个fd的数组每次都要在用户态和内核态之间传递，影响效率。为此，epoll设计了“逻辑上的epfd”，epfd是一个数字，把fd数组关联到上面，然后每次向内核传递的是epfd这个数字。

select/poll的使用举例见[LINUX – IO MULTIPLEXING – SELECT VS POLL VS EPOLL](https://devarea.com/linux-io-multiplexing-select-vs-poll-vs-epoll/)。

### epoll

[epoll](http://man7.org/linux/man-pages/man7/epoll.7.html)相关函数的声明：

```c
// 创建一个epoll句柄，size用来告诉内核监听的数目一共有多少。
// 其中的size并不要求是准确的数字，只是告诉内核，计划监听多少个fd。
// 实际通过epoll_ctl添加的fd数目可能大于这个值。
int epoll_create(int size);

// 将一个fd增/删/改到epfd里，对应的事件也即读/写
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);

// 其中的maxevents也是可以自定义的。假如有100个fd，而maxevents只设置为64，
// 则其他fd，上面的事件会在下次调用epoll_wait时返回
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
```

整个epoll过程分成三个步骤：

* 事件注册：通过函数epoll_ctl实现。对于服务器而言，是accept、read、write三种事件；对于客户端而言，是connect、read、write三种事件。
* 轮询这三个事件是否就绪：通过函数epoll_wait实现。有事件发生，该函数返回。
* 事件就绪，执行实际的IO操作：通过函数accept、read、write实现。

事件就绪的说明：

* read事件就绪：指远程有新数据来了，socket读取缓冲区里有数据，需要调用read函数处理
* write事件就绪：指本地的socket写缓冲区是否可写。如果写缓冲区没有满，则一直都是可写的，write事件一直是就绪的，可以调用write函数。只有当遇到发送大文件的场景时，socket写缓冲区被占满时，write事件才不是就绪状态
* accept事件就绪：有新的连接你如，需要调用accept函数处理

epoll里面有两种模式：LT（水平触发）和ET（边缘触发）。水平触发又称条件触发，边缘触发又称状态触发。

* 水平触发：读缓冲区只要不为空，就会一直触发读事件；写缓冲区只要不满，就会一直触发写事件
* 边缘触发：读缓冲区的状态，从空转为非空的时候触发一次；写缓冲区的状态从满转为非满时触发一次。

关于LT和ET，有两点需要注意的问题：

* 对于LT模式，要避免“写的死循环”问题：写缓冲区为满的概率很小，即“写的条件”会一直满足，所以当用户注册了写事件却没有数据要写时，它会一直触发，因此在LT模式下写完数据一定要取消写事件
* 对于ET模式，要避免“short read”问题：例如用户收到100个字节，它触发1次，用户只读到了50个字节，剩下的50字节不读，它也不会再次触发。因此在ET模式下，一定要把“读缓冲区”的数据一次性全部读完

在事件开发中，一般倾向于用LT，这也是默认的模式，Java NIO用的也是epoll的LT模式。因为ET容易漏事件，一次触发如果没有处理好，就没有第二次机会了。虽然LT重复触发可能有少许性能损耗，但更安全。



基于IO多路复用的高并发学习框架[handy](https://github.com/yedf/handy)。

### NIO

Java NIO的4个重要抽象API是

* Buffers，数据缓冲区
* Charsets，表示字符到字节的编码和解码
* Channels，表示可执行IO操作的网络连接，或者称其为数据流，可执行读写操作
* Selectors，表示epoll的IO多路复用

Java NIO可以使用一个Selector线程处理所有的Channel连接。Channel和Buffer使用直接缓冲区实现“零拷贝”[ [2] ](https://en.wikipedia.org/wiki/Zero-copy)。

![Java NIO模型](/images/java_nio_abstract.jpg "Java NIO模型")

Selector举例：

```java
Selector selector = Selector.open();

channel.configureBlocking(false);

SelectionKey key = channel.register(selector, SelectionKey.OP_READ);


while(true) {

  int readyChannels = selector.selectNow();

  if(readyChannels == 0) continue;


  Set<SelectionKey> selectedKeys = selector.selectedKeys();

  Iterator<SelectionKey> keyIterator = selectedKeys.iterator();

  while(keyIterator.hasNext()) {

    SelectionKey key = keyIterator.next();

    if(key.isAcceptable()) {
        // a connection was accepted by a ServerSocketChannel.

    } else if (key.isConnectable()) {
        // a connection was established with a remote server.

    } else if (key.isReadable()) {
        // a channel is ready for reading

    } else if (key.isWritable()) {
        // a channel is ready for writing
    }

    keyIterator.remove();
  }
}
```



## Reactor模式

Reactor模式称为主动模式。所谓主动，是指应用程序不断地轮询，访问操作系统或网络框架、IO是否就绪。Linux系统下的select、poll、epoll就属于主动模式，需要应用程序中有一个循环一直轮询；Java中的NIO也属于这种模式。在这种模式下，实际的IO操作还是应用程序执行的。

Reactor的组件：

* Reactor：Reactor是IO事件的派发者
* Acceptor：Acceptor接受client连接，建立对应client的Handler，并向Reactor注册此Handler
* Handler：和一个client通讯的实体，相当于Java NIO中的Channel。

单线程的Reactor模型：

![单线程的Reactor模型](/images/reactor_single_thread.webp "单线程的Reactor模型")

多线程的Reactor模型：

![多线程的Reactor模型](/images/reactor_multiple_thread.webp "多线程的Reactor模型")

主从Reactor模型：

![主从Reactor模型](/images/reactor_master_slave.webp "主从Reactor模型")

多线程的Reactor模型将线程分为IO线程和工作线程。

Netty是Reactor模式的开源框架。

详细说明见[高性能Server---Reactor模型](https://www.jianshu.com/p/2461535c38f3)。

## Proactor模式

Proactor称为被动模式，应用程序把read和write函数操作全部交给操作系统或者网络框架，实际的IO操作由操作系统或者网络框架完成，之后再回调应用程序。[asio](https://think-async.com/Asio/)库就是典型的Proactor模式。

