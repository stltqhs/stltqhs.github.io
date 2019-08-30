---
title: Java后端开发 - 操作系统
date: 2018-10-13 08:18:11
tags: java
---



# Linux from scratch

[Linux from scratch](http://www.linuxfromscratch.org/)叙述如何制作Linux版本，包括制作引导程序、编译linux内核、制作initrd文件系统。

Linux from scratch对于初学者会比较复杂，可以先阅读[从零开始制作Linux](https://blog.csdn.net/linyt/article/details/80142036)迅速感受一下。

# 操作系统实现

### 1 进程

#### 1.1 调度算法

#### 1.2 上下文切换

### 2 内存

#### 2.1 进程地址空间与地址转换

#### 2.2 段

#### 2.3 页

#### 2.4 TLB

### 3 并发



# 容器

### Namespace

Linux Namespace用于实现系统资源隔离，不同namespace的进程使用的资源相互独立<sup>[1]</sup>。

### API

* clone

clone<sup>2</sup>类似fork，用于创建进程，同时还可以通过 <i>flags</i> 参数指定创建命名空间，实现Linux容器的命名空间有`CLONE_NEWCGROUP`、`CLONE_NEWIPC`、`CLONE_NEWNET`、`CLONE_NEWNS`、`CLONE_NEWPID`、`CLONE_NEWUTS`、`CLONE_NEWUSER`、`CLONE_NEWUTS`<sup>3</sup>。

* chroot
* mount



### 参考

1.[https://lwn.net/Articles/531114/#series_index](https://lwn.net/Articles/531114/#series_index)

2.[http://man7.org/linux/man-pages/man2/clone.2.html](http://man7.org/linux/man-pages/man2/clone.2.html)

3.[http://man7.org/linux/man-pages/man7/namespaces.7.html](http://man7.org/linux/man-pages/man7/namespaces.7.html)

5.[https://coolshell.cn/articles/17010.html](https://coolshell.cn/articles/17010.html)

6.[https://coolshell.cn/articles/17029.html](https://coolshell.cn/articles/17029.html)





轻量级容器实现[cocker](https://github.com/calvinwilliams/cocker)

[Everything You Need to Know about Linux Containers, Part I: Linux Control Groups and Process Isolation](https://www.linuxjournal.com/content/everything-you-need-know-about-linux-containers-part-i-linux-control-groups-and-process)

[Everything You Need to Know about Linux Containers, Part II: Working with Linux Containers (LXC)](https://www.linuxjournal.com/content/everything-you-need-know-about-linux-containers-part-ii-working-linux-containers-lxc)

https://www.cnblogs.com/huxiao-tee/p/4660352.html)