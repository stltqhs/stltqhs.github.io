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

# IO模型

## IO多路复用



## Reactor模式



## Proactor模式



## Disruptor模式



[C10K](http://www.kegel.com/c10k.html)问题

基于IO多路复用的高并发学习框架[handy](https://github.com/yedf/handy)

