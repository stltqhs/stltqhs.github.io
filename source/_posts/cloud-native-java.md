---
title: Cloud Native Java阅读笔记
date: 2018-04-10 22:10:31
tags:java
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

本章介绍Restful API开发方式，使用[RestTemplate](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/client/RestTemplate.html)来对接Restful接口。使用`@ControllerAdvice`和`@ExceptionHandler`处理请求数据和异常数据。使用`HAL`和`HATEOAS`来描述资源，使用`Spring RESTDocs`来生成Restful API接口文档，不过已习惯使用`swagger`。

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