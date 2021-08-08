---
title: Java后端开发 - Servlet
date: 2018-10-13 08:15:09
tags: java
toc: true
---

# Servlet生命周期与过滤器

Servlet 生命周期可被定义为从创建直到毁灭的整个过程。以下是 Servlet 遵循的过程：

- Servlet 通过调用 **init ()** 方法进行初始化；

  init 方法被设计成只调用一次。它在第一次创建 Servlet 时被调用，在后续每次用户请求时不再调用。因此，它是用于一次性初始化。

- Servlet 调用 **service()** 方法来处理客户端的请求；

  service() 方法是执行实际任务的主要方法。Servlet 容器（即 Web 服务器）调用 service() 方法来处理来自客户端（浏览器）的请求，并把格式化的响应写回给客户端。

  每次服务器接收到一个 Servlet 请求时，服务器会产生一个新的线程并调用服务。service() 方法检查 HTTP 请求类型（GET、POST、PUT、DELETE 等），并在适当的时候调用 doGet、doPost、doPut，doDelete 等方法。

- Servlet 通过调用 **destroy()** 方法终止（结束）；

- 最后，Servlet 是由 JVM 的垃圾回收器进行垃圾回收的。

Servlet中的过滤器Filter是实现了javax.servlet.Filter接口的服务器端程序，只要你在web.xml 文件配置好要拦截的客户端请求，它都会帮你拦截到请求。Filter可认为是Servlet的一种“变种”，它主要用于对用户请求进行预处理，也可以对HttpServletResponse进行后处理，是个典型的处理链。它与Servlet的区别在于：它不能直接向用户生成响应。完整的流程是：Filter对用户请求进行预处理，接着将请求交给Servlet进行处理并生成响应，最后Filter再对服务器响应进行后处理。

# Session和Cookie的区别

Session是在服务端保存的一个数据结构，用来跟踪用户的状态，这个数据可以保存在集群、数据库、文件中；

Cookie是客户端保存用户信息的一种机制，用来记录用户的一些信息，也是实现Session的一种方式。

# Servlet的异步请求

- Servlet3.0的异步请求

  [Servlet3.0规范](http://download.oracle.com/otn-pub/jcp/servlet-3.0-fr-oth-JSpec/servlet-3_0-final-spec.pdf?AuthParam=1535295644_2a1916a1ba6dc83fc114e98f80b21f78)2.3.3.3节的“Asynchronous processing”讲述了异步处理，当启用异步处理时，响应数据可以由其他线程创建，而当前线程可以立即交还给Servlet容器处理其他请求。当要启用异步处理时，首先需要声明HttpServlet支持异步处理，然后才能调用`request.startAsync`方法获得`AsyncContext`，它包含`request`和`response`。调用`startAsync`方法后，程序能保证在退出`service`方法时，不会完成本次请求，即不响应任何数据给客户端。需要在之后调用`AsyncContext.complete`方法来完成本次请求，向客户端写入数据。

- Servlet3.1的NIO

  [Servlet3.1规范](https://javaee.github.io/servlet-spec/downloads/servlet-3.1/Final/servlet-3_1-final.pdf)第3.7节“Non Blocking IO”讲述了非阻塞IO。Servlet 3.0对请求的处理虽然是异步的，但是对InputStream和OutputStream的IO操作却依然是阻塞的，对于数据量大的请求体或者返回体，阻塞IO也将导致不必要的等待，因此在Servlet 3.1中引入了非阻塞IO。

# Tomcat架构

Tomcat的Servlet容器是用来处理请求Servlet资源，并为web客户端填充response对象的模块。在Tomcat中，共有四种类型的容器，分别是：Engine、Host、Context和Wrapper。Servlet容器是[org.apache.catalina.Container](https://github.com/apache/tomcat/blob/TOMCAT_8_0_0/java/org/apache/catalina/Container.java)接口的实例。Container接口及其相关类的UML类图如下：

```text
                                     +----------------+
                                     | <<interface>>  |
                                     |   Container    |
                                     +----------------+
                                             ^       ^
                                             |       |
                                             |       +---------------------------+
                                             |                                   |
       +------------------+------------------+------------------+                |
       |                  |                  |                  |                |
+-------------+    +-------------+    +-------------+    +-------------+   +-------------+
|<<interface>>|    |<<interface>>|    |<<interface>>|    |<<interface>>|   |ContainerBase|
|   Engine    |    |     Host    |    |   Context   |    |   Wrapper   |   |             |
+-------------+    +-------------+    +-------------+    +-------------+   +-------------+
       ^                  ^                   ^                     ^             ^
       |                  |                   |                     |             |
       |  +---------------^---+---------------^-----+---------------^---+---------+
       |  |               |   |               |     |               |   |
+--------------+ INC  +-------------+ INC  +---------------+ INC  +---------------+
|StandardEngine| <>-- | StandartHost| <>-- |StandardContext| <>-- |StandardWrapper|
+--------------+ 1  n +-------------+ 1  n +---------------+ 1  n +---------------+
```

Tomcat的[Pipeline](https://github.com/apache/tomcat/blob/TOMCAT_8_0_0/java/org/apache/catalina/Pipeline.java)管道和[Valve](https://github.com/apache/tomcat/blob/TOMCAT_8_0_0/java/org/apache/catalina/Valve.java)阀的工作机制类似于Servlet的过滤器，在Servlet容器的管道中，有一个基础阀，也可以添加任意数量的阀，基础阀总是最后一个执行。

当Tomcat启动时，相关组件也会一起启动，Tomcat关闭时，这些组件也会随之关闭，这是通过实现[Lifecycle](https://github.com/apache/tomcat/blob/TOMCAT_8_0_0/java/org/apache/catalina/Lifecycle.java)接口达到统一启动/关闭组件的效果。实现了`Lifecycle`接口的组件可以触发一个或多个事件：`BEFORE_START_EVENT`、`START_EVENT`、`AFTER_START_EVENT`、`BEFORE_STOP_EVENT`、`STOP_EVENT`、`AFTER_STOP_EVENT`，事件是[LifecycleEvent](https://github.com/apache/tomcat/blob/TOMCAT_8_0_0/java/org/apache/catalina/LifecycleEvent.java)类的实例，事件监听器是[LifecycleListener](https://github.com/apache/tomcat/blob/TOMCAT_8_0_0/java/org/apache/catalina/LifecycleListener.java)类的实例，[LifecycleSupport](https://github.com/apache/tomcat/blob/TOMCAT_8_0_0/java/org/apache/catalina/util/LifecycleSupport.java)提供了一种简单的方法来触发某个组件的生命周期事件，并对事件监听器进行处理。

Tomcat载入器是[Loader](https://github.com/apache/tomcat/blob/TOMCAT_8_0_0/java/org/apache/catalina/Loader.java)接口的实例，[WebappLoader](https://github.com/apache/tomcat/blob/TOMCAT_8_0_0/java/org/apache/catalina/loader/WebappLoader.java)是负责从`WEB-INF/classes`和`WEB-INF/lib`载入Servlet所需类的载入器，它内部会使用[WebappClassLoader](https://github.com/apache/tomcat/blob/TOMCAT_8_0_0/java/org/apache/catalina/loader/WebappClassLoader.java)作为类载入器（或类加载器）。`Loader`接口使用`modified()`方法来支持类的自动重载，在载入器的具体实现中，如果仓库（`Loader.addRepository()`）中的一个或多个类被修改，那么`modified()`方法必须返回`true`，载入器内部会调用`Context.reload`完成类重新载入的过程。载入类时，`WebappClassLoader`遵循如下规则：

- 因为所有已经载入的类都会缓存起来，所以载入类时要先检查本地缓存；
- 若本地缓存中没有，则检查上一层缓存，即调用`java.lang.ClassLoader`类的`findLoadedClass()`方法；
- 如果两个缓存中都没有，则使用系统的类载入器`ApplicationClassLoader`进行加载，防止Web应用程序中的类覆盖Java EE的类；
- 若启用了`SecurityManager`，则检查是否运行载入该类。若该类是禁止载入的类，抛出`ClassNotFoundException`异常；
- 若打开标志位`delegate`，或者待载入的类是属于包触发器`WebappClassLoader.packageTriggers`中的包名，则调用父类载入器来载入相关类。如果父类载入器为`null`，则使用系统的类载入器；
- 从当前仓库中载入相关类；
- 若当前仓库中没有需要的类，且标志位`delegate`关闭，则使用父类载入器。若父类载入器为`null`，则使用系统的类载入器进行加载；
- 若仍未找到需要的类，则抛出`ClassNotFoundException`异常。

Tomcat使用Session管理器的组件来管理建立的Session对象，该组件由[Manager](https://github.com/apache/tomcat/blob/TOMCAT_8_0_0/java/org/apache/catalina/Manager.java)接口表示。Session管理器需要与一个`Context`容器相关联，且必须与一个`Context`容器关联。Session管理器负责创建、更新、销毁Session对象，当有请求到来时，要返回一个有效的Session对象。Servlet实例通过调用`javax.servlet.http.HttpServletRequest.getSession()`方法获取一个Session对象。在Tomcat的默认连接器中，[Request](https://github.com/apache/tomcat/blob/TOMCAT_8_0_0/java/org/apache/catalina/connector/Request.java)类实现`HttpServletRequest`接口。Session管理器会将其所管理的Session对象放在内存中，或存储在文件中，或通过JDBC写入数据库中。Session对象是[Session](https://github.com/apache/tomcat/blob/TOMCAT_8_0_0/java/org/apache/catalina/Session.java)接口的实例，实现类是[StandardSession](https://github.com/apache/tomcat/blob/TOMCAT_8_0_0/java/org/apache/catalina/session/StandardSession.java)。[StandardManager](https://github.com/apache/tomcat/blob/TOMCAT_8_0_0/java/org/apache/catalina/session/StandardManager.java)是`Manager`接口的实现类，session管理器需要负责销毁已经失效的Session对象，在[ManagerBase](https://github.com/apache/tomcat/blob/TOMCAT_8_0_0/java/org/apache/catalina/session/ManagerBase.java)类的`backgroundProcess`方法负责过期Session对象的检查与销毁。[Store](https://github.com/apache/tomcat/blob/TOMCAT_8_0_0/java/org/apache/catalina/Store.java)接口是为Session管理器的Session对象提供持久化存储器的一个组件，`Store`接口两个重要的方法是`save`和`load`。[StoreBase](https://github.com/apache/tomcat/blob/TOMCAT_8_0_0/java/org/apache/catalina/session/StoreBase.java)是一个抽象类，提供了基本功能，该类的两个子类是[FileStore](https://github.com/apache/tomcat/blob/TOMCAT_8_0_0/java/org/apache/catalina/session/FileStore.java)和[JDBCStore](https://github.com/apache/tomcat/blob/TOMCAT_8_0_0/java/org/apache/catalina/session/JDBCStore.java)。

一般情况下一个`Context`容器包含一个或多个`Wrapper`，而一个`Wrapper`对应一个`HttpServlet`。[StandardWrapper](https://github.com/apache/tomcat/blob/TOMCAT_8_0_0/java/org/apache/catalina/core/StandardWrapper.java)是[Wrapper](https://github.com/apache/tomcat/blob/TOMCAT_8_0_0/java/org/apache/catalina/Wrapper.java)接口的实现类，它的基础阀[StandardWrapperValve](https://github.com/apache/tomcat/blob/TOMCAT_8_0_0/java/org/apache/catalina/core/StandardWrapperValve.java)会调用Servlet的过滤器和Servlet的`service`方法。连接器接收到HTTP请求后，方法调用的协作图如下：

```text
1:[new request] create() ->
2:[new response] create() ->
            +----+
Message1 -> |    | 3:invoke() ->                   4:invoke() ->
         +-----------+           +-----------------+             +-----------------------+
---------| Connector | ----------| StandardContext |-------------|StandardContextPipeline|
         +-----------+           +-----------------+             +-----------------------+
                                                                 5:invoke() |
                                                                     |      |
                           <-7:invoke()              <-6:invoke()    v      |
   +----------------------+          +-----------------+         +----------------------+
   | StandardWrapperValve |----------| StandardWrapper |---------| StandardContextValve |
   +----------------------+          +-----------------+         +----------------------+
              |
8:allocate()  | 11:service()
     |        |      |
     v        |      v
   +----------------------+  9:load()->
   |       Servlet        |---------------
   +----------------------+  10:init()->
```

协作图过程描述如下：

- 1.连接器创建request和response对象；
- 2.连接器调用`StandardContext`实例的`invoke()`方法；
- 3.`StandardContext`实例的`invoke()`方法调用其管道对象的`invoke()`方法。`StandardContext`中管道对象的基础阀是`StandardContextValve`类的实例，因此，`StandardContext`的管道对象会调用`StandardContextValve`实例的`invoke()`方法；
- 4.`StandardContextValve`实例的`invoke()`方法获取相应的`Wrapper`实例处理HTTP请求。调用`Wrapper`实例的`invoke()`方法；
- 5.`StandardWrapper`类是`Wrapper`接口的标准实现，`StandardWrapper`实例的`invoke()`方法会表用其管道对象的`invoke`方法；
- 6.`StandardWrapper`的管道对象中的基础阀是`StandardWrapperValve`类的实例，因此，会调用`StandardWrapperValve`的`invoke()`方法，`StandardWrapperValve`的`invoke()`方法调用`Wrapper`实例的`allocate()`方法获取Servlet实例；
- 7.`allocate()`方法调用`load()`方法载入相应的Servlet类，若已经载入，则无需重复载入；
- 8.`load()`方法调用Servlet实例的`init()`方法；
- 9.`StandardWrapperValve`调用Servlet实例的`service()`方法。

[FilterDef](https://github.com/apache/tomcat/blob/TOMCAT_8_0_0/java/org/apache/tomcat/util/descriptor/web/FilterDef.java)表示一个过滤器的定义，用来说明`web.xml`部署描述符文件中定义的过滤器。[ApplicationFilterConfig](https://github.com/apache/tomcat/blob/TOMCAT_8_0_0/java/org/apache/catalina/core/ApplicationFilterConfig.java)类实现`javax.servlet.FilterConfig`接口，用于管理Web应用程序第一次启动时创建的所有的过滤器实例，其`getFilter()`方法会返回一个`javax.servlet.Filter`对象。[ApplicationFilterChain](https://github.com/apache/tomcat/blob/TOMCAT_8_0_0/java/org/apache/catalina/core/ApplicationFilterChain.java)类实现`javax.servlet.FilterChain`接口，`StandardWrapperValve`类的`invoke()`方法会创建`ApplicationFilterChain`类的一个实例，并调用其`doFilter()`方法。`ApplicationFilterChain`类的`doFilter()`方法会调用过滤器链中第一个过滤器的`doFilter`方法。如果某个过滤器时过滤器链中的最后一个过滤器，则会调用被请求的Servlet类的`service()`方法。

[Host](https://github.com/apache/tomcat/blob/TOMCAT_8_0_0/java/org/apache/catalina/Host.java)接口表示一个虚拟主机，其实现类是[StandardHost](https://github.com/apache/tomcat/blob/TOMCAT_8_0_0/java/org/apache/catalina/core/StandardHost.java)。[Engine](https://github.com/apache/tomcat/blob/TOMCAT_8_0_0/java/org/apache/catalina/Engine.java)表示Servlet引擎，其实现类是[StandardEngine](https://github.com/apache/tomcat/blob/TOMCAT_8_0_0/java/org/apache/catalina/core/StandardEngine.java)。`Host`的父容器只能是`Engine`，而`Engine`是顶级容器，不再有父容器，且子容器只能是`Host`。[Server](https://github.com/apache/tomcat/blob/TOMCAT_8_0_0/java/org/apache/catalina/Server.java)表示整个Servlet引擎，其实现类是[StandardServer](https://github.com/apache/tomcat/blob/TOMCAT_8_0_0/java/org/apache/catalina/core/StandardServer.java)。`Server`几个重要方法是`initialize()`、`start()`、`stop()`、`await()`、`addService()`。`initialize()`方法用来初始化通过`addService()`方法添加到`Server`中的服务组件。`await()`方法则是创建一个`ServerSocket`，等待客户端连接并发送关闭指令，如果收到关闭指令，则调用`stop()`方法关闭`Server`，即整个Servlet引擎。[Service](https://github.com/apache/tomcat/blob/TOMCAT_8_0_0/java/org/apache/catalina/Service.java)接口表示一个服务，其实现类是[StandardService](https://github.com/apache/tomcat/blob/TOMCAT_8_0_0/java/org/apache/catalina/core/StandardService.java)类。`Service`通过`setContainer()`方法关联一个容器，通过`addConnector()`方法，关联多个连接器。`StandardService`内部会将其容器通过`Connector.setContainer`方法设置`Service`的容器给`Connector`。

Tomcat的总体架构图如下：

![Tomcat架构](https://www.ibm.com/developerworks/cn/java/j-lo-tomcat1/image001.gif "Tomcat架构")

当一个处理请求时，连接器`Connector`创建`Request`和`Response`对象，通过[Mapper](https://github.com/apache/tomcat/blob/TOMCAT_8_0_0/java/org/apache/catalina/mapper/Mapper.java)类来选择相应的`Context`和`Wrapper`来处理请求。

Tomcat使用[Digester](https://commons.apache.org/proper/commons-digester/)库来解析`%CATALINA_HOME%/conf/server.xml`文件和web应用的`web.xml`文件。[ContextConfig](https://github.com/apache/tomcat/blob/TOMCAT_8_0_0/java/org/apache/catalina/startup/ContextConfig.java)类的实例负责并解析读取web应用的`web.xml`文件，为每个Servlet元素创建一个`StandardWrapper`实例。`ContextConfig`实现了`LifecycleListener`接口，被添加到`StandardContext`作为其事件监听器。每次`StandardContext`实例触发事件时，会调用`ContextConfig`实例的`lifecycleEvent()`方法，该方法会调用`ContextConfig.start()`方法来解析`web.xml`文件。如果文件解析成功，调用`Context.setConfigured(true)`方法，使得`Context`可以正常启动，否则启动失败。

Tomcat的启动类是[Bootstrap](https://github.com/apache/tomcat/blob/TOMCAT_8_0_0/java/org/apache/catalina/startup/Bootstrap.java)，其`start`方法首先判断是否存在[Catalina](https://github.com/apache/tomcat/blob/TOMCAT_8_0_0/java/org/apache/catalina/startup/Catalina.java)实例，如果存在则初始化一个`Catalina`实例，然后调用`Catalina.start()`解析`%CATALINA_HOME%/conf/server.xml`文件和初始化`Server`。`Catalina`会调用`Runtine.addShutdownHook()`方法来注册一个`CatalinaShutdownHook`类型的关闭钩子，用于安全的释放资源。

`StandardHost`实例使用`HostConfig`对象作为一个生命周期监听器，当`StandardHost`对象启动时，它的`start()`方法会触发一个`START`事件。为了响应`START`事件，[HostConfig](https://github.com/apache/tomcat/blob/TOMCAT_8_0_0/java/org/apache/catalina/startup/HostConfig.java)中的lifecycleEvent方法和`HostConfig`中的事件处理程序调用`start()`方法。`HostConfig`类的`deployApps()`会完成`Host`下各个`Context`的部署工作。

延伸阅读

* [Tomcat 系统架构与设计模式，第 1 部分 工作原理](https://www.ibm.com/developerworks/cn/java/j-lo-tomcat1/index.html)，
* [Tomcat 系统架构与设计模式，第 2 部分 设计模式分析](https://www.ibm.com/developerworks/cn/java/j-lo-tomcat2/)
* [深入剖析Tomcat](https://book.douban.com/subject/10426640/)