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



![Pipeline和HTTP/2请求响应](/images/http_pipeline.jpg)

### 头部压缩

HTTP/2使用HPACK协议和对应的算法对HTTP头部压缩。

# TCP

TCP协议规范是[RFC793](https://tools.ietf.org/html/rfc793)。

## 三次握手

## 四次挥手





# IO多路复用

[C10K](http://www.kegel.com/c10k.html)问题

基于IO多路复用的高并发学习框架[handy](https://github.com/yedf/handy)

