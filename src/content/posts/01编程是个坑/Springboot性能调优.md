---
title: Springboot性能调优
published: 2026-09-23
description: Springboot性能调优
image: "api"
tags: [编程]
category: 编程
draft: false
---
#### **容器选型与线程池模型匹配**

Springboot，将Tomcat替换使用undertow，会导致原有的Tomcat相关配置失效



网络传输优化

通过对于某些大量响应数据的接口，使用HTTP报文压缩，节省带宽，同时加入压缩触发阈值避免每次都使用报文压缩，给服务器带来压力。



连接池

连接数设置过大，应当根据硬件性能进行估算



