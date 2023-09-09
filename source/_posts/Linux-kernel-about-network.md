---
title: Linux 网络简说（上）
tags: 网络
toc: true
date: 2023-09-09 16:20:24
---


*Linux 网络的知识点是非常多的，不可能一篇文章就可能讲清楚。本文参考了 [开发内功修炼](https://github.com/yanfeizhang/coder-kung-fu) 的文章，做了简单叙述，算是一个笔记。*

# 网络分层

![网络分层](/images/network_kernel_layer.jpg)
*图-1 网络分层*

在 Linux 内核实现中，链路层协议靠网卡驱动来实现，内核协议栈来实现网络层和传输层。内核对更上层的应用层提供 socket 接口来供用户进程访问。我们用 Linux 的视⻆来看到的 TCP/IP 网络分层模型应该是图-1的样子。

内核和网络设备驱动是通过中断的方式来处理的。当设备上有数据到达的时候，会给 CPU 的相关引脚上触发一个电压变化，以通知 CPU 来处理数据。

Linux 中断处理函数是分上半部和下半部的。上半部是只进行最简单的工作，快速处理然后释放 CPU ，接着 CPU 就可以允许其它中断进来。剩下将绝大部分的工作都放到下半部中，可以慢慢从容处理。2.4 以后的内核版本采用的下半部实现方式是软中断，由 ksoftirqd 内核线程全权处理。

# Linux 收包流程

![Linux 收包流程](/images/network_kernel_receive_with_hardware.jpg)
*图-2 Linux 收包流程*

## 启动和初始化

### ksoftirqd 进程初始化

Linux 的软中断都是在专⻔的内核线程(ksoftirqd)中进行的，该进程数量不是 1 个,而是 N 个,其中 N 等于你的机器的核数。系统初始化的时候在 kernel/smpboot.c 中调用了 `smpboot_register_percpu_thread`， 该函数进一步会执行到 `spawn_ksoftirqd`（位于 kernel/softirq.c）来创建出 softirqd 进程。当 ksoftirqd 被创建出来以后,它就会进入自己的线程循环函数 `ksoftirqd_should_run` 和 `run_ksoftirqd` 了。不停地判断有没有软中断需要被处理。

![ksoftirqd 创建流程](/images/network_kernel_ksoftirqd.jpg)
*图-3 ksoftirqd 创建流程*

### 网络子系统初始化

![网络子系统初始化](/images/network_kernel_subnetwork.jpg)
*图-4 网络子系统初始化*

linux 内核通过调用 `subsys_initcall`  来初始化各个子系统，在这个函数里，会为每个 CPU 都申请一个 softnet_data  数据结构，在这个数据结构里的 poll_list  是等待驱动程序将其 poll 函数注册进来。另外 `open_softirq` 注册了每一种软中断都注册一个处理函数。 `NET_TX_SOFTIRQ` 的处理函数为 `net_tx_action`，`NET_RX_SOFTIRQ` 的为 `net_rx_action`。

### 协议栈注册

![协议栈注册](/images/network_kernel_stack_register.jpg)
*图-5 协议栈注册*

内核实现了网络层的 ip 协议，也实现了传输层的 tcp 协议和 udp 协议。 这些协议对应的实现函数分别是 `ip_rcv()`, `tcp_v4_rcv()`和 `udp_rcv()`。通过 `inet_init` ,将这些函数注册到了 `inet_protos` 和 `ptype_base` 数据结构中了。`inet_protos` 记录着 udp 和 tcp 的处理函数地址，`ptype_base` 存储着 `ip_rcv()` 函数的处理地址。

如果看一下 `ip_rcv` 和 `udp_rcv` 等函数的代码能看到很多协议的处理过程。例如 `ip_rcv` 中会处理 netfilter 和 iptable 过滤，如果你有很多或者很复杂的 netfilter 或 iptables 规则，这些规则都是在软中断的上下文中执行的，会加大网络延迟。再例如 `udp_rcv` 中会判断 socket 接收队列是否满了。对应的相关内核参数是 `net.core.rmem_max` 和 `net.core.rmem_default`。

### 网卡初始化

![网卡初始化](/images/network_kernel_netinterface_init.jpg)
*图-6 网卡初始化*

### 启动网卡
![启动网卡](/images/network_kernal_start_netinterface.jpg)
*图-6 启动网卡*

## 接收数据

### 硬中断处理

![硬中断处理](/images/network_kernal_hardware_interrupt.jpg)
*图-7 硬中断处理*

首先当数据帧从网线到达网卡上的时候,第一站是网卡的接收队列。网卡在分配给自己的 RingBuffer 中寻找可用的内存位置,找到后 DMA 引擎会把数据 DMA 到网卡之前关联的内存里,这个时候 CPU 都是无感的。当 DMA 操作完成以后,网卡会向 CPU 发起一个硬中断,通知 CPU 有数据到达。

*注意:当RingBuffer满的时候,新来的数据包将给丢弃。ifconfig查看网卡的时候,可以里面有个overruns,表示因为环形队列满被丢弃的包。如果发现有丢包,可能需要通过ethtool命令来加大环形队列的⻓度。*

### ksoftirqd 软中断处理

![ksoftirqd 软中断处理](/images/network_kernel_ksoftirqd_process.jpg)
*图-8 ksoftirqd 软中断处理*

硬中断中设置软中断标记,和 ksoftirq 的判断是否有软中断到达,都是基于 `smp_processor_id()` 的。这意味着只要硬中断在哪个 CPU 上被响应,那么软中断也是在这个 CPU 上处理的。所以说,如果你发现你的 Linux 软中断 CPU 消耗都集中在一个核上的话,做法是要把调整硬中断的 CPU 亲和性,来将硬中断打散到不同的 CPU 核上去。

使用的 `time_limit` 和 `budget` 来控制 `net_rx_action` 函数主动退出，目的是保证网络包的接收不霸占 CPU 不放。

`net_rx_action` 这个函数中剩下的核心逻辑是获取到当前 CPU 变量 `softnet_data	`，对其 `poll_list` 进行遍历, 然后执行到网卡驱动注册到的 `poll` 函数。

### 网络协议栈处理

![网络协议栈处理](/images/network_kernel_stack_process.jpg)
*图-9 网络协议栈处理*

在 `__netif_receive_skb_core`  中，包含原来经常使用的 tcpdump 的抓包。接着 `__netif_receive_skb_core`  取出 protocol,它会从数据包中取出协议信息,然后遍历注册在这个协议上的回调函数列表。 `ptype_base`  是一个 hash table,在协议注册小节我们提到过。`ip_rcv` 函数地址就是存在这个 hash table 中的。

### IP 协议层处理

`ip_recv` -> `ip_rcv_finish` -> `ip_route_input_noref` -> `ip_route_input_mc` -> `ip_local_deliver`。

`ip_local_deliver` 是路由子系统

## 协议处理
### UDP

![UDP 收包流程](/images/network_kernel_recvfrom_udp.jpg)
*图-10 UDP 收包流程*

所谓的读取过程,就是访问 `sk>sk_receive_queue` 。如果没有数据,且用户也允许等待,则将调用 `wait_for_more_packets() `执行等待操作,它加入会让用户进程进入睡眠状态。相关代码如下：

```c
//file: net/core/datagram.c:EXPORT_SYMBOL(__skb_recv_datagram); 
struct sk_buff *__skb_recv_datagram(struct sock *sk, unsigned int flags, int *peeked, int *off, int *err) {
	// ......
	do {
		struct sk_buff_head *queue = &sk->sk_receive_queue;
		skb_queue_walk(queue, skb) {
			// ......
	    }
	    /* User doesn't want to wait */
	    error = -EAGAIN;
	    if (!timeo) goto no_packet;
	} while (!wait_for_more_packets(sk, err, &timeo, last));
}
```

### TCP

![进程收包流程](/images/network_kernel_receive_wait.jpg)
*图-11 进程收包流程*

![socket 结构](/images/network_kernel_socket_struct.jpg)
*图-12 socket 结构*

```c
//file: net/core/sock.c 
void sock_init_data(struct socket *sock, struct sock *sk) {
	sk->sk_data_ready   =   sock_def_readable;
	sk->sk_write_space  =   sock_def_write_space;
	sk->sk_error_report =   sock_def_error_report;
}
```

当调用 `sock_create` 时会创建一些列的 socket 相关对象，在 `sock_init_data` 会注册一些函数（比如上面的代码）。当软中断上收到数据包时会通过调用 `sk_data_ready` 函数指针(实际被设置成了 `sock_def_readable()`) 来唤醒在 sock 上等待的进程。

### 读阻塞原理

![进程阻塞原理](/images/network_kernel_receive_process.jpg)
*图-13 进程阻塞原理*

```c
//file: net/core/sock.c 
int sk_wait_data(struct sock *sk, long *timeo) {
	//当前进程(current)关联到所定义的等待队列项上  
	DEFINE_WAIT(wait);
	// 调用 sk_sleep 获取 sock 对象下的 wait
	// 并准备挂起,将进程状态设置为可打断 INTERRUPTIBLE   
	prepare_to_wait(sk_sleep(sk), &wait, TASK_INTERRUPTIBLE);
	set_bit(SOCK_ASYNC_WAITDATA, &sk->sk_socket->flags);
	// 通过调用schedule_timeout让出CPU,然后进行睡眠
	rc = sk_wait_event(sk, timeo, !skb_queue_empty(&sk>sk_receive_queue));
	...
}
```

首先在 `DEFINE_WAIT` 宏下,定义了一个等待队列项 `wait`。 在这个新的等待队列项上,注册了回调函数 `autoremove_wake_function`,并把当前进程描述符 `current` 关联到其 `.private` 成员上。

紧接着在 `sk_wait_data` 中 调用 `sk_sleep` 获取 sock 对象下的等待队列列表头 `wait_queue_head_t`。

接着调用 `prepare_to_wait` 来把新定义的等待队列项 `wait` 插入到 `sock` 对象的等待队列下。

这样后面当内核收完数据产生就绪时间的时候,就可以查找 socket 等待队列上的等待项,进而就可以找到回调函数和在等待该 socket 就绪事件的进程了。   
最后再调用 `sk_wait_event` 让出 CPU,进程将进入睡眠状态,这会导致一次进程上下文的开销。

### 进程唤醒原理

![TCP 收包流程](/images/network_kernel_tcp_recv.jpg)
*图-14 TCP 收包流程*

软中断(也就是 Linux 里的 ksoftirqd 进程)里收到数据包以后,发现是 tcp 的包的话就会执行到 `tcp_v4_rcv` 函数。接着走,如果是 ESTABLISH 状态下的数据包,则最终会把数据拆出来放到对应 socket 的接收队列中。然后调用 sk_data_ready 来唤醒用户进程。

在 `tcp_v4_rcv` 中首先根据收到的网络包的 header 里的 source 和 dest 信息来在本机上查询对应的 socket。找到以后,我们直接进入接收的主体函数 `tcp_v4_do_rcv` 来看。

```c
//file: net/ipv4/tcp_input.c 
int tcp_rcv_established(struct sock *sk, struct sk_buff *skb, const struct tcphdr *th, unsigned int len) {
   ......
  // 接收数据到队列中  
  eaten = tcp_queue_rcv(sk, skb, tcp_header_len, &fragstolen);   
  // 数据 ready,唤醒 socket 上阻塞掉的进程  
  sk->sk_data_ready(sk, 0);
```

在 `tcp_rcv_established` 中通过调用  tcp_queue_rcv 函数中完成了将接收数据放到 socket 的接收队列上。

```c
//file: net/ipv4/tcp_input.c 
static int __must_check tcp_queue_rcv(struct sock *sk, struct sk_buff *skb, int hdrlen, bool *fragstolen) {   
	// 把接收到的数据放到 socket 的接收队列的尾部
	if (!eaten) {
		__skb_queue_tail(&sk->sk_receive_queue, skb);
		skb_set_owner_r(skb, sk);
	}
	return eaten;
}
```

调用 `tcp_queue_rcv` 接收完成之后,接着再调用 `sk_data_ready` 来唤醒在socket上等待的用户进程。

回想上面我们在 创建 socket 流程里执行到的 `sock_init_data` 函数,在这个函数里已经把 `sk_data_ready` 设置成 `sock_def_readable` 函数了。

```c
// file: net/core/sock.c static 
void sock_def_readable(struct sock *sk, int len) {
	struct socket_wq *wq;
	rcu_read_lock();
	wq = rcu_dereference(sk->sk_wq);
	// 有进程在此 socket 的等待队列
	if (wq_has_sleeper(wq))     //唤醒等待队列上的进程
		wake_up_interruptible_sync_poll(&wq->wait, POLLIN | POLLPRI | POLLRDNORM | POLLRDBAND);

	sk_wake_async(sk, SOCK_WAKE_WAITD, POLL_IN);
	rcu_read_unlock();
}
```

`wake_up_interruptible_sync_poll` 会调用 `__wake_up_sync_key` 方法，在此继续调用 `__wake_up_common` 方法。`__wake_up_common` 实现唤醒。这里注意下, 该函数调用是参数 `nr_exclusive` 传入的是 1,这里指的是即使是有多个进程都阻塞在同一个 socket 上,也只唤醒 1 个进程。其作用是为了避免惊群。

```c
//file: kernel/sched/core.c 
static void __wake_up_common(wait_queue_head_t *q, unsigned int mode, int nr_exclusive, int wake_flags, void *key) {
	wait_queue_t *curr, *next;
	list_for_each_entry_safe(curr, next, &q->task_list, task_list) {
		unsigned flags = curr->flags;
		if (curr->func(curr, mode, wake_flags, key) && (flags & WQ_FLAG_EXCLUSIVE) && !--nr_exclusive) break;
	}
}
```

在 `__wake_up_common` 中找出一个等待队列项 `curr`,然后调用其 `curr->func`。回忆我们前面在 `recv` 函数执行的时候,使用 `DEFINE_WAIT()` 定义等待队列项的细节,内核把 `curr>func` 设置成了 `autoremove_wake_function`。

在 `autoremove_wake_function` 中,调用了 `default_wake_function`。

```c
//file: kernel/sched/core.c
int default_wake_function(wait_queue_t *curr, unsigned mode, int wake_flags, void *key) {
	return try_to_wake_up(curr->private, mode, wake_flags);
}
```

调用 `try_to_wake_up` 时传入的 `task_struct` 是 `curr->private`。这个就是当时因为等待而被阻塞的进程项。 当这个函数执行完的时候,在 socket 上等待而被阻塞的进程就被推入到可运行队列里了,这又将是一次进程上下文切换的开销。

## IO 多路复用 epoll 原理

在 Linux 上多路复用方案有 select、poll、epoll。 它们三个中 epoll 的性能表现是最优秀的,能支持的并发量也最大。

epoll 相关的函数是如下三个:
* `epoll_create`:创建一个 epoll 对象
* `epoll_ctl`:向 epoll 对象中添加要管理的连接
* `epoll_wait`:等待其管理的连接上的 IO 事件

简单示例代码如下：
```c
int main(){
	listen(lfd, ...);
	cfd1 = accept(...);
	cfd2 = accept(...);
	efd = epoll_create(...);
	epoll_ctl(efd, EPOLL_CTL_ADD, cfd1, ...);
	epoll_ctl(efd, EPOLL_CTL_ADD, cfd2, ...);
	epoll_wait(efd, ...);
}
```

### accept 创建新 socket

当 accept 之后,进程会创建一个新的 socket 出来,专⻔用于和对应的客户端通信,然后把它放到当前进程的打开文件列表中。

![进程文件列表](/images/network_kernel_task_files.jpg)
*图-15 进程文件列表*

其中一条连接的 socket 内核对象更为具体一点的结构图如下。

![socket 文件](/images/network_kernel_task_one_file.jpg)
*图-16 socket 文件*

#### 初始化 struct socket 对象

首先是调用 `sock_alloc` 申请一个 struct socket 对象出来。然后接着把 `listen` 状态的 socket 对象上的协议操作函数集合 ops 赋值给新的 socket。

![socket 操作函数赋值](/images/network_kernel_init_socket.jpg)
*图-17 socket 操作函数赋值*

其中 inet_stream_ops 的定义如下

```c
//file: net/ipv4/af_inet.c
const struct proto_ops inet_stream_ops = {
	...
    .accept        = inet_accept,
    .listen        = inet_listen,
    .sendmsg       = inet_sendmsg,
    .recvmsg       = inet_recvmsg,
    ...
}
```

#### 为新 socket 对象申请 file

struct socket 对象中有一个重要的成员 -- file 内核对象指针。这个指针初始化的时候是空的。 在 accept 方法里会调用 `sock_alloc_file` 来申请内存并初始化。 然后将新 file 对象设置到 `sock->file` 上。`sock_alloc_file` 又会接着调用到 `alloc_file`。注意在 `alloc_file` 方法中,把 `socket_file_ops` 函数集合一并赋到了新 `file->f_op` 里了。在 `accept` 里创建的新 socket 里的 `file->f_op->poll` 函数指向的是 `sock_poll`。

![socket 文件](/images/network_kernel_socket_file.jpg)
*图-18 socket 文件*

#### 接收连接

在 socket 内核对象中除了 file 对象指针以外,有一个核心成员 sock。这个 struct sock 数据结构非常大,是 socket 的核心内核对象。发送队列、接收队列、等待队列等核心数据结构都位于此。在 `accept` 的源码中会调用 `sock->ops->accept` 方法，它对应的方法是 `inet_accept`。它执行的时候会从握手队列里直接获取创建好的 sock。

#### 添加新文件到当前进程的打开文件列表

当 file、socket、sock 等关键内核对象创建完毕以后,剩下要做的一件事情就是把它挂到当前进程的打开文件列表中就行了。

```c
//file: fs/file.c 
void fd_install(unsigned int fd, struct file *file) {
	__fd_install(current->files, fd, file);
}
```

```c
void __fd_install(struct files_struct *files, unsigned int fd, struct file *file) {
	...
    fdt = files_fdtable(files);
    BUG_ON(fdt->fd[fd] != NULL);
    rcu_assign_pointer(fdt->fd[fd], file);
}
```

### epoll_create 实现

在用户进程调用 `epoll_create` 时,内核会创建一个 struct eventpoll 的内核对象。并同样把它关联到当前进程的已打开文件列表中。

![进程中的 eventpoll 文件](/images/network_kernel_eventpoll_file.jpg)
*图-19 进程中的 eventpoll 文件*

对于 struct eventpoll 对象,更详细的结构如下(同样只列出和主题相关的成员)。

![eventpoll 文件说明](/images/network_kernel_eventpoll_file_desc.jpg)
*图-20 eventpoll 文件说明*

eventpoll 这个结构体中的几个成员的含义如下: 
* wq: 等待队列链表。软中断数据就绪的时候会通过 wq 来找到阻塞在 epoll 对象上的用户进程。
* rbr: 一棵红黑树。为了支持对海量连接的高效查找、插入和删除,eventpoll 内部使用了一棵红黑树。通过这棵树来管理用户进程下添加进来的所有 socket 连接。  
* rdllist: 就绪的描述符的链表。当有的连接就绪的时候,内核会把就绪的连接放到 rdllist 链表里。这样应用进程只需要判断链表就能找出就绪进程,而不用去遍历整棵树。

### epoll_ctl 添加 socket

在使用 `epoll_ctl` 注册每一个 socket 的时候,内核会做如下三件事情
* 分配一个红黑树节点对象 epitem, 
* 添加等待事件到 socket 的等待队列中,其回调函数是 `ep_poll_callback`
* 将 epitem 插入到 epoll 对象的红黑树里

通过 epoll_ctl 添加两个 socket 以后,这些内核数据结构最终在进程中的关系图大致如下。

![注册两个 socket 的文件列表](/images/network_kernel_epctl.jpg)
*图-21 注册两个 socket 的文件列表*

### epoll_wait 等待接收

`epoll_wait` 做的事情不复杂,当它被调用时它观察 `eventpoll->rdllist` 链表里有没有数据即可。有数据就返回,没有数据就创建一个等待队列项,将其添加到 eventpoll 的等待队列上,然后把自己阻塞掉就完事。

![epoll_wait 流程](/images/network_kernel_epoll_wait.jpg)
*图-22 epoll_wait 流程*

### 数据来啦

当 socket 上数据就绪时候,内核将以 `sock_def_readable` 这个函数为入口,找到 epoll_ctl 添加 socket 时在其上设置的回调函数 `ep_poll_callback`。

`ep_poll_callback` 函数的逻辑如下：
```c
//file: fs/eventpoll.c 
static int ep_poll_callback(wait_queue_t *wait, unsigned mode, int sync, void *key)
{     
	// 获取 wait 对应的 epitem     
	struct epitem *epi = ep_item_from_wait(wait);     
	// 获取 epitem 对应的 eventpoll 结构体    
	struct eventpoll *ep = epi->ep;     
	// 1. 将当前epitem 添加到 eventpoll 的就绪队列中
	list_add_tail(&epi->rdllink, &ep->rdllist);
	// 2. 查看 eventpoll 的等待队列上是否有在等待
	if (waitqueue_active(&ep->wq))
		wake_up_locked(&ep->wq);
```

在 `ep_poll_callback` 根据等待任务队列项上的额外的 base 指针可以找到 epitem, 进而也可以找到 eventpoll对象。

首先它做的第一件事就是把自己的 epitem 添加到 epoll 的就绪队列中。

接着它又会查看 eventpoll 对象上的等待队列里是否有等待项(epoll_wait 执行的时候会设置)。

如果没执行软中断的事情就做完了。如果有等待项,那就查找到等待项里设置的回调函数。

![eventpoll 等待队列](/images/network_kernel_epoll_wakeup.jpg)
*图-23 eventpoll 等待队列*

在 `default_wake_function` 中找到等待队列项里的进程描述符,然后唤醒之。

等待队列项 `curr->private` 指针是在 epoll 对象上等待而被阻塞掉的进程。

将 `epoll_wait` 进程推入可运行队列,等待内核重新调度进程。然后 `epoll_wait` 对应的这个进程重新运行后,就从 schedule 恢复当进程醒来后,继续从 `epoll_wait` 时暂停的代码继续执行。把 `rdlist` 中就绪的事件返回给用户进程。

## 网络包处理之 CPU 开销汇总
* 系统态 CPU 开销
* 硬中断、软中断开销
* 进程上下文切换

# Linux 发包流程

## LINUX 网络发送过程总览

![发包总览](/images/network_kernel_send_overview.jpg)
*图-24 发包总览*

注意，我们今天的主题虽然是发送数据,但是硬中断最终触发的软中断却是 `NET_RX_SOFTIRQ`,而并不是 `NET_TX_SOFTIRQ` !!!(T 是 transmit 的缩写,R 表示 receive)

## 网卡启动

现在的服务器上的网卡一般都是支持多队列的。每一个队列上都是由一个 RingBuffer 表示的,开启了多队列以后的的网卡就会对应有多个 RingBuffer。

网卡在启动时最重要的任务之一就是分配和初始化 RingBuffer。

![发送环形队列](/images/network_kernel_ring_buffer.jpg)
*图-25 发送环形队列*

有两个 RingBuffer，分别是:
* igb_tx_buffer 数组:这个数组是内核使用的,通过 vzalloc 申请的；
* e1000_adv_tx_desc 数组:这个数组是网卡硬件使用的,硬件是可以通过 DMA 直接访问这块内存,通过 `dma_alloc_coherent` 分配。

## 发送数据

### send 系统调用实现

![消息发送](/images/network_kernel_send.jpg)
*图-26 消息发送*

### 传输层处理

#### 传输层拷⻉
![TCP 消息发送](/images/network_kernel_tcp_send.jpg)
*图-27 TCP 消息发送*

在 tcp_sendmsg 函数里，内核会申请一个内核态的 skb 内存,将用户待发送的数据拷⻉进去。注意这个时候不一定会真正开始发送,如果没有达到发送条件的话很可能这次调用直接就返回了。

![发送队列](/images/network_kernel_skb.jpg)
*图-28 发送队列*

#### 传输层发送

![传输层发送](/images/network_kernel_xmit.jpg)
*图-29 传输层发送*

#### 网络层发送处理
![网络层发送](/images/network_kernel_ip_out.jpg)
*图-30 网络层发送*

#### 邻居子系统

![邻居子系统设置 mac 地址](/images/network_kernel_neighbor.jpg)
*图-31 邻居子系统设置 mac 地址*

#### 网络设备子系统
![网络设备子系统](/images/network_kernel_netdev.jpg)
*图-32 网络设备子系统*

图-32所示，如果系统态 CPU 发送网络包不够用的时候,会调用 `__netif_schedule` 触发一个软中断。该函数会进入到 `__netif_reschedule`,由它来实际发出 `NET_TX_SOFTIRQ` 类型软中断。

#### 软中断调度

![软中断发送](/images/network_kernel_soft_tx.jpg)
*图-33 软中断发送*

#### igb 网卡驱动发送
![igb 网卡驱动发送](/images/network_kernel_igb.jpg)
*图-34 igb 网卡驱动发送*

#### 发送完成硬中断
![发送完成后响应硬中断并进行清理](/images/network_kernel_dev_send_complete.jpg)
*图-35 发送完成后响应硬中断并进行清理*

#### 总结

![发送流程总结](/images/network_kernel_send_desc.jpg)
*图-36 发送流程总结*