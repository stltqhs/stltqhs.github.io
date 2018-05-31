---
title: mysql编译与调试
date: 2018-03-31 15:48:24
tags: mysql
---

## 一、源码

`mysql-server`的源码放在github中，代码仓库路径是：`https://github.com/mysql/mysql-server.git`，使用下面的命令将代码复制到本地计算机中。

```
git clone https://github.com/mysql/mysql-server.git
```
这可能会花费1小时的时间，因为文件总共有670M左右，可能最快的方式是直接获取源代码，而不是从github中下载。



## 二、Windows

### 1.生成`visual studio`项目配置文件

需要使用`cmake`命令和`bison`的支持，需要将这2项下载好，添加到PATH环境变量中。
执行下面的命令生成VS2013的项目配置文件：
```
cmake -G "Visual Studio 12 2013"
```
### 2.VS编译
（1）编译时可能出现`sql_locale.cc`编译失败，出现错误的字符等等。可以将该文件转换为`Unicode(UTF8 BOM)`型文件；
（2）SIGABRT问题需要将`sql/mysqld.cc`中的 `DBUG_ASSERT(0)` 注释掉；
（3）因为权限问题无法操作注册表，可以以管理员身份运行`VS`。
设置`mysqld`为默认启动项，然后编译，由于`mysqld`启动时需要加载`data`目录（使用`mysqld --initialize --datadir=<datadir> --lc-messages-dir=<mysql-server-source-dir>/sql/share` 初始化`data`目录），因此可以设置启动参数如下：
```
--defaults-file=<my.ini>
```
其中`<my.ini>`为`mysql`的配置文件，此时就可以调试。
需要配置`lc_messages_dir=<mysql-server-source-dir>/sql/share`

总结：
1.初始数据文件
```
mysqld --initialize --datadir=<datadir> --lc-messages-dir=<mysql-server-source-dir>/sql/share
```
2.设置启动参数
```
--defaults-file=<my.ini>
```
3.将日志调试时不要隐藏窗口
```
--console
```



## 三、OS X

### 1.Developer目录设置

当安装了Xcode后，需要设置Developer目录，默认是/Library/Developer，需要设置到Xcode的Developer目录，执行命令：

```sh
sudo xcode-select -s /Applications/Xcode.app/Contents/Developer
```

###  2.生成Xcode工程

使用cmake的`out-of-source`方式构建，执行命令：

```sh
mkdir bld_debug
cd bld_debug
cmake .. -G 'Xcode' -DCMAKE_BUILD_TYPE=Debug
```

### 3.编译

使用Xcode打开mysql-server工程，自动生成Schema，开始编译。编译完成后，选择mysqld这个Schema，需要编辑该Schema，设置mysqld命令的参数。

1).使用`mysqld --initialize --datadir=<datadir> --lc-messages-dir=<mysql-server-source-dir>/sql/share` 初始化`data`目录，这里`data`目录设置为`${PRODUCT_DIR}/data`；

2).编辑mysqld的Schema，选择左侧的Run，点击右侧的Arguments，设置`datadir`；

3).可能还需要设置`lc-messages-dir`；

此时就可以运行mysqld程序。

### 4.调试

当以Debug模式编译成功后，运行mysqld时会生成`/tmp/mysqld.trace`，显示`mysqld`的执行细节。



打开sql模块下的sql_select.cc文件，在handle_query方法下的某一行打下断点，然后在mysql客户端执行`select `就会被断下来。

## 参考：

1.mysql5.6在windows下的编译与调试，http://www.voidcn.com/blog/fei33423/article/p-4830278.html；

2.使用cmake编译，https://dev.mysql.com/doc/internals/en/cmake.html；