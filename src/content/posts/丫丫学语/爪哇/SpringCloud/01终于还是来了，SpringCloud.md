---
title: 01终于还是来了，SpringCloud
published: 2026/10/25
tags:
  - java
  - SpringCloud
category:
draft: true
---
实在遭不住每次面试被问到，明明简历都没有写，只好来看看到底怎么个事吧，之前一直停留在理念上，实在没过多深入研究，直接AI全家桶整了一遭。


SpringCloud和SpringBoot的区别？

依赖于以前对于若依框架和零零散散的印象，SpringCloud本质可以说就是多个SpringBoot服务组建的一个网络，通过类似于nacos的服务中心，实现各个服务之间的通信，其中包含额外的功能如请求控制、服务崩溃的兜底机制等

里面额外的包含组件如OpenFeign

Q：那各个微服务必须是由SpringBoot作为开发框架吗，如某个服务使用nodejs有的使用fastapi呢？
A：可以的。Nacos/Eureka 的注册协议是 HTTP + JSON，任何语言只要按它的接口注册、拉取，就能接入。这就是为什么有 `nacos-sdk-python`、`nacos-sdk-go`。但代价是：`OpenFeign`、`@LoadBalanced`、`@EnableDiscoveryClient` 这些「无感集成」就都没了，node/fastapi 服务需要自己写服务发现 + 负载均衡 + 重试。实际生产里跨语言通常也是这么做的，Java 服务走 Spring Cloud 全家桶，其他语言服务只做「注册 + 直接调」。


# Nacos

作为SpringCloud的注册中心+配置中心的角色，同样实现其功能的还有别的，如**Eureka、Zookeeper**、，都是用于注册和管理多个SpringBoot服务，实现调度，各个服务之间如何协作



## OpenFeign

> 声明式HTTP客户端，像调用本地接口一样调用远程服务



FeignClient：

```java
@FeignClient(name = "user-service") // 声明该服务调用user-service微服务
public interface UserClient {
    @GetMapping("/user/{id}")
    User getById(@PathVariable("id") Long id);
}
```













