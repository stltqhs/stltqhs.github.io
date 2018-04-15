---
title: Cloud Native Java阅读笔记
date: 2018-04-10 22:10:31
tags: java
---

《Cloud Native Java》主要介绍使用Spring Cloud开发微服务应用，包括业务逻辑层，缓存，分布式事务及数据持久化。

### 第1章-第2章

这两章主要介绍Spring Boot的使用，不过叙述太过简单，关于Spring Boot的使用可以阅读《[Spring Boot In Action](https://www.manning.com/books/spring-boot-in-action?)》。Spring Boot比较重要的就是条件配置。

### 第3章

本章主要介绍集中化配置。从Spring的属性配置`PropertyPlaceHolderConfigurer`到`Environment`到Spring Boot的`Profile`，这些都是基于硬编码或者命令行级别的配置方式。Spring Cloud支持的配置中心化是`Spring Cloud Config Server`，且使用`@RefreshScope`注解可以刷新应用的配置而不需要重启应用。



使用`@EnableConfigServer`来创建配置服务，在客户端使用`bootstrap.yml`配置文件指定配置服务来从配置服务中获取配置信息。

### 第4章

本章主要介绍测试相关的内容。

### 第5章

本章主要介绍如何使用Spring Cloud将应用迁移到云平台中（如Cloud Foundry）。本章还介绍了使用`X/Open XA`协议和`JTA`来实现分布式事务。

### 第6章

本章介绍Restful API开发方式，使用[RestTemplate](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/client/RestTemplate.html)来对接Restful接口。使用`@ControllerAdvice`和`@ExceptionHandler`处理请求数据和异常数据。使用`HAL`和`HATEOAS`来描述资源，使用`Spring RESTDocs`来生成Restful API接口文档。

### 第7章

本章介绍分布式系统服务的寻址路由，包括服务注册、发现、负载均衡。使用`@EnableEurekaServer`启动注册中心，而且还提供了管控台。使用`@EnableDiscoveryClient`启动`服务提供者`，`服务消费者`（或者客户端）同样适用`@EnableDiscoveryClient`来使用eureka注册中心，可以注入`DiscoveryClient`实例来获取已经注册的服务的信息，由于服务会存在多个，可以使用客户端级别的负载均衡组件`Netflix Ribbon`来选取需要使用的服务，例如：

```java
@Autowired
private DiscoveryClient discoveryClient;
// ...
String serviceId = "greetings-service";
List<Server> servers = discoveryClient.getInstances(serviceId)
    .stream()
    .map(si -> new Server(si.getHost(), si.getPort()))
    .collect(Collectors.toList());

IRule roundRobinRule = new RoundRobinRule();

BaseLoadBalancer loadBalancer = LoadBalancerBuilder.newBuilder()
    .withRule(roundRobinRule)
    .buildFixedServerListLoadBalancer(servers);

Server server = loadBalancer.chooseServer();

ResponseEntity<JsonNode> response = this.restTemplate.getForEntity("http://" + server.getHost() + ":" + server.getPort() + "/hi/{name}", JsonNode.class, variables);
```

或者直接注入`LoadBalancerClient`，例如：

```java
@Autowired
private LoadBalancerClient loadBalancerClient;
// ...
String serviceId = "greetings-service";

ServiceInstance server = loadBalancerClient.choose(serviceId);

ResponseEntity<JsonNode> response = this.restTemplate.getForEntity("http://" + server.getHost() + ":" + server.getPort() + "/hi/{name}", JsonNode.class, variables);
```

也可以使用`@LoadBalance`来注入`RestTemplate`实例，使用`RestTemplate`调用服务接口时Spring Cloud内部会帮我们做负载均衡，例如：

```java
@LoadBalance
@Autowired
private RestTemplate restTemplate;
// ...
ResponseEntity<JsonNode> response = this.restTemplate.getForEntity("http://greetings-service/hi/{name}", JsonNode.class, variables);
```

其中`greetings-service`是服务id，Spring Cloud的拦截器会拦截请求的url，获取服务id，根据它来选取某一个服务提供者。

### 第8章

本章介绍`服务消费者`调用`服务提供者`的方式，有3种，如下：

##### 1.带负载均衡的RestTemplate

第7章已经介绍，这种方式编码比较多，是一种原始的方法。

##### 2.Netflix Feign

该方法可以将显示的RestTemplate调用过程抽象化成一个接口，由Spring Cloud内部处理具体的调用过程，例如：

```java
@FeignClient(serviceId = "greetings-service")
interface GreetingsClient {
    @RequestMapping(method = RequestMethod.GET, value = "/greet/{name}")
    Map<String, String> greet(@PathVariable("name") String name);
}

// ...
@Autowired
GreetingsClient greetingsClient;

// ...
greetingsClient.greet(name);
```

##### 3.Netflix Zuul

Zuul的核心是代理或者过滤器。使用`@EnableZuulProxy`和`Zulu.routes.*`来配置zuul代理，使`边界服务`（`Edge Service`）直连`服务提供者`，例如：

```java
// ZuulConfiguration.java
@Configuration
@EnableZuulProxy
class ZuulConfiguration {
    @Bean
    CommandLineRunner commandLineRunner(RouteLocator routeLocator) {
        Log log = LogFactory.getLog(getClass());
        return args -> routeLocator.getRoutes().forEach(
        	r -> log.info(String.format("%s (%s) %s", r.getId(), r.getLocation(), r.getFullPath()));
        )
    }
}
```

```properties
# application.properties
zuul.routes.hi.path = /lets/**
zuul.routes.hi.serviceId = greetings-service
```

从浏览器到Zuul代理的边界服务到服务提供者的流程如下：

```text
     +---------+               +-------------------+            +-------------------+
     | Browser | ---access-->  | Zuul Edge Service |  --access->| greetings-service |
     +---------+               +-------------------+            +-------------------+
http://edge-service/lets/greet/{name} --zuulproxy--> http://greetings-service/greet/{name}
                    +----------------+       +-------------+
                    | Servlet Filter |  ---> | Zuul Filter |
                    +----------------+       +-------------+
```

自定义Zuul过滤器的方式是继承`ZuulFilter`，Zuul过滤器有4种类型：`pre`，`routing`，`post`，`error`。比如实现一个限流的Zuul过滤器[ThrottlingZuulFilter.java](https://github.com/cloud-native-java/edge/blob/master/edge-service/src/main/java/greetings/ThrottlingZuulFilter.java)。



维护可用服务列表的方式是监听`HeartbeatEvent`事件，例如：

```java
@EventListener(HeartbeatEvent.class)
public void onHeartbeantEvent(HeartbeatEvent e) {
    discoveryClient.getServices().forEach(
    	svc -> map.put(svc, discoveryClient.getInstances(svc))
    )
}
```

详细代码可以参考[RoutesListener](https://github.com/cloud-native-java/edge/blob/master/edge-service/src/main/java/greetings/RoutesListener.java)。

本章还介绍了使用`Spring Security`给服务提供者添加权限相关的特性。

### 第9章

本章主要介绍数据存储的API使用，包括Spring JPA相关内容（如：[CrudRepository](https://docs.spring.io/spring-data/commons/docs/current/api/org/springframework/data/repository/CrudRepository.html) ，使用`@Document`标记的实体可以使Spring访问`MongoDB`），数据库存取API（[JdbcTemplate](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/jdbc/core/JdbcTemplate.html)），Redis存取API（[RedisTemplate](https://docs.spring.io/spring-data/redis/docs/current/api/org/springframework/data/redis/core/RedisTemplate.html)），以及缓存控制（[@CacheEvict](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/cache/annotation/CacheEvict.html)，[@Cacheable](https://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/cache/annotation/Cacheable.html)，[@CachePut](https://docs.spring.io/spring-framework/docs/4.0.x/javadoc-api/org/springframework/cache/annotation/CachePut.html)）。



另外，本章还介绍了基于图结构的`Nosql`数据存取工具——[Neo4j](https://neo4j.com/docs/developer-manual/current/)，该数据结构可以用在推荐系统中。

### 第10章

本章介绍消息中间件和[Spring Cloud Stream](https://docs.spring.io/spring-cloud-stream/docs/current/reference/htmlsingle/)。使用`Spring Cloud Stream`可以简化消息中间件的使用，如生产者（基于[Channel](https://github.com/cloud-native-java/messaging/blob/master/stream/stream-producer/src/main/java/stream/producer/channels/StreamProducer.java)或者基于[Gateway](https://github.com/cloud-native-java/messaging/blob/master/stream/stream-producer/src/main/java/stream/producer/ProducerChannels.java)），[消费者](https://github.com/cloud-native-java/messaging/blob/master/stream/stream-consumer/src/main/java/stream/consumer/integration/StreamConsumer.java)。



本章还介绍了Spring Data Flow，比如实现如下命令的功能：

```shell
cat input.txt | grep ERROR | wc -l > output.txt
```

Java代码的实现参考[这里](https://github.com/cloud-native-java/messaging/blob/master/batch-and-integration/src/main/java/edabatch/EtlFlowConfiguration.java)。

### 第11章

本章介绍Spring[批处理](https://docs.spring.io/spring-batch/4.0.x/reference/html/index.html)、任务、定时器的功能，代码示例参考[这里](https://github.com/cloud-native-java/integration)。

### 第12章

本章介绍分布式系统下的数据集成。有关分布式的相关理论如下：

1.CAP

[C(Consistency)A(Availability)P(Partition)](https://zh.wikipedia.org/wiki/CAP%E5%AE%9A%E7%90%86)是分布式系统数据完整性相关的理论，其中：

* Consistency：一致性，等同于所有节点访问同一份最新的数据副本；
* Availability：可用性，每次请求都能获得非错的响应，但不能保证获取的数据为最新数据；
* Partition：分区容错性，以实际效果而演言，分区相当于对通信的时限要求。系统如果不能在时限内达成数据一致性，就意味着发生了分区的情况，必须就当前操作在C和A之间做出选择。

![CAP理论](/images/cap.jpg "CAP理论")

![CAP之协议](http://book.mixu.net/distsys/images/CAP.png "CAP理论")

该理论规定分布式系统只能满足三项中的两项，不可能全部满足。该理论的[猜想](http://www.cs.berkeley.edu/~brewer/cs262b-2004/PODC-keynote.pdf)于2000年提出，在2002年得到[证明](http://lpd.epfl.ch/sgilbert/pubs/BrewesConjecture-SigAct.pdf)。

2.BASE

[Basically Availability,Soft State,Eventual Consistency](https://en.wikipedia.org/wiki/Eventual_consistency)是CAP的延伸，表示即使无法做到强一致性，但可以保证最终一致性，与传统的[ACID](https://en.wikipedia.org/wiki/ACID)形成对比，是两种不同的设计理念，其中：

* Basically Availability：基本可用，指系统允许部分功能可用，保证核心功能可能，如`SOA`下的服务降级；
* Soft State：软状态，指允许系统存在中间状态，而该中间状态不会影响系统整体可用性。分布式存储中一般一份数据至少会有三个副本，允许不同节点间副本同步的延时就是软状态的体现。mysql replication的异步复制也是一种体现；
* Eventual Consistency：最终一致性，指系统中的所有数据副本经过一定时间后，最终能够达到一致的状态。

[一致性模型](https://en.wikipedia.org/wiki/Consistency_model)可以大致分为：

* 强一致性：当更新操作完成之后，任何多个后续进程或者线程的访问都会返回最新的更新过的值。这种是对用户最友好的，就是用户上一次写什么，下一次就保证能读到什么。但是这种实现对性能影响较大；
* 弱一致性：系统并不保证续进程或者线程的访问都会返回最新的更新过的值。系统在数据写入成功之后，不承诺立即可以读到最新写入的值，也不会具体的承诺多久之后可以读到。但会尽可能保证在某个时间级别（比如秒级别）之后，可以让数据达到一致性状态；
* 最终一致性：弱一致性的特定形式。系统保证在没有后续更新的前提下，系统最终返回上一次更新操作的值。在没有故障发生的前提下，不一致窗口的时间主要受通信延迟，系统负载和复制副本的个数影响。DNS是一个典型的最终一致性系统。最终一致性的变种有因果一致性、读已所写一致性、会话一致性、单调读一致性、单调写一致性。

对于大多数应用，并不需要强一致性，因此牺牲一致性而换取高可用性。

3.Paxos

[Paxos](https://zh.wikipedia.org/wiki/Paxos%E7%AE%97%E6%B3%95)一种基于消息传递且具有高度容错特性的一致性算法。

4.2PC

[2PC](https://zh.wikipedia.org/wiki/%E4%BA%8C%E9%98%B6%E6%AE%B5%E6%8F%90%E4%BA%A4)（Two-Phase Commit），二阶段提交，指为了使基于分布式系统架构下的所有节点在进行事务提交时保持一致性而设计的一种算法。最大缺点就在于它的执行过程中间，节点都处于阻塞状态。

5.Saga

Saga模式用来处理long-lived事务，将事务分成很小单元的事务，每个小单元事务都有一个补偿事务，用来回滚事务，保证一致性。SEC（Saga execution Coordinator）存储saga日志，saga日志存储了进行中事务的记录。

6.CQRS

CQRS(Command Query Responsibility Segregation)提供一种解决方案即separate the writes from the reads。Command驱使微服务更新，Query用来查询结果。任意时刻一个Command更新数据时会触发一个事件依靠Query来更新所有相关服务更新自己的数据。大致流程如下：

```text
+--------+          +-----------+         +-------------+          +-----------------+
| client | ---1---> | Service 1 | ---2--> | Command Bus | ---3---> | Command Handler |
+--------+          +-----------+         +-------------+          +-------+---------+
                                                                           |
                                                                           4
                                                                           |
                                                                           V
                                             +-----------+         +----------------+
                                             | Service 2 |<---5--- | Event Handler  |
                                             +-----------+         +----------------+
```

步骤说明如下：

(1)：用户访问一个一个接口，可能是Rest API，使Service 1更新了某条数据；

(2)：Service 1更新某条数据后，向Command总线发送了一个Command；

(3)：Command处理器处理来自Command总线的新Command，验证有效性后发送一个或者多个事件；

(4)：事件处理器处理Command处理器发来的事件，它可能会产生更多的事件并发送给Service 2，event-sourcing类型的事件支持重放功能；

(5)：Service 2消耗事件，对系统做出响应的修改。

CQRS的实现方式可用直接使用`Spring Cloud Stream`和`Spring Data`，本章介绍使用[Axon](https://docs.axonframework.org/)框架实现，示例代码参考[这里](https://github.com/cloud-native-java/data-integration)，比较重要的类有CommandGateway，@TargetAggregateHandler，@CommandHandler，@EventSourcingHandler，@EventListener，@EventHandler。

7.SEDA

SEDA(Staged event-driven architectures)指将基于事件驱动的应用程序拆分为基于消息队列的场景集合(a set of stages connected by queues)，是云端服务需要足够健壮的平衡负载和扩容的理想设计模式。本章介绍使用`Spring Cloud Data Flow`来实现这种设计模式或者架构。



本章还介绍了[Hystrix](https://github.com/Netflix/Hystrix/wiki)，它是一种分布式容错系统。

### 第13章

本章介绍服务监控，使用[Spring Boot Actuator](https://docs.spring.io/spring-boot/docs/current/reference/html/production-ready.html)捕获系统当前状态。`Actuator`的接口见下表（部分）：

| Endpoint  | Usage                                                        |
| --------- | ------------------------------------------------------------ |
| /info     | Exposes information about current service.                   |
| /metrics  | Exposes quantifiable values about service.                   |
| /beans    | Exposes a graph of all the objects that Spring Boot has created for you. |
| /health   | A description of the state of components in the system: UP,DOWN,etc.Also returns HTTP status codes. |
| /mappings | Exposes all the HTTP endpoints that Spring Boot is aware of in this application as well as any other metadata(such as specified content-types or HTTP verbs in the Spring MVC mapping). |
| /env      | returns all of the known environment properties, such as those in the operating system's environment variables or the results of System.getProperties(). |

使用CounterService来增加自己的指标数据，使用GaugeService获取相关数据，示例如下：

```java
@Autowired
private CounterService counterService;

@Autowired
private GaugeService gaugeService;
// ...

counterService.increment("customers.read.not-found"/*metric name*/);
```

使用[Spring Cloud Sleuth](http://cloud.spring.io/spring-cloud-sleuth/1.3.x/)建立分布式跟踪服务，帮助我们建立系统context，编译监测系统真实状态。Trace id是第一次请求时系统自动生成，贯穿整个请求流程（跨多个组件或者服务），当请求进入到新的组件或者服务时一个新的span id会生成。`Spring Cloud Sleuth`可用在日志中记录trace id和span id，不管请求是来自于消息中间件、Spring MVC、Netflix Zuul、RestTemplate、Netflix Feign Rest客户端，日志格式如下：

```text
2016-02-11 17:12:45.404 INFO [my-service-id,73b62c0f90d11e06,73b6etydf90d11e06,false] 85184 --- [nio-8080-exec-1] com.example.MysimpleComponentMakingARequest : ...
```

my-service-id是spring.application.name指定的服务id，73b62c0f90d11e06是trace id，73b6etydf90d11e06是span id。

本章介绍了[OpenZipkin](https://github.com/openzipkin/zipkin)，它是一种分布式日志跟踪系统，提供用户界面查看系统各个服务的运行状态。Spring Boot Admin和Ordina提供了其他方式来查看服务状态。

### 第14章

本章介绍Cloud Foundry的服务化。

### 第15章

本章介绍持续发布，介绍的工具是[Concourse](https://github.com/concourse/concourse)，它与[Jenkins](https://jenkins.io/doc/)或者Travis有两点不同，即容器和管道，使用示例参考[这里](https://github.com/cloud-native-java/continuous-delivery)。