---
title: Redis和数据库一致性
published: 2026-11-20
description: Redis和数据库一致性
tags:
  - redis
category: 八股文学
draft: false
---
什么情况会出现数据不一致？

-  读写并发
**对数据库和Redis数据的写删的先后顺序**

## 先操作缓存

**先删除缓存，然后写数据库**

![](assets/Pasted%20image%2020261120221138.png)

**数据不一致出现情况**
Redis缓存删除后，更新数据库期间有较大延迟时，导致其他线程请求Redis发现数据不存在，进入到数据库查询数据从而导致查询的是旧的数据。

### 最终一致性

其他线程可能会查询到旧的值（来自于线程1更新数据库时延期间），但最终线程1更新数据库后，还是会将新值推到Redis中（先删后加，但需要延迟删除），保障两者数据一致性。

**延迟双删：**
- 如果不延迟，线程1更新完数据库就立马删除Redis数据，从而可能导致线程2将旧值写回Redis，出现数据不一致。

![](assets/Pasted%20image%2020261120221459.png)

### 思考方案（强一致性）

按照“击穿”场景的处理方式，在读数据库这个步骤添加锁，当前仅删除缓存的那个线程才能将数据库的对应数据写回Redis，只允许单线程进来，实现的是**强一致性**，但丢失了性能。

## 先操作数据库

  ![](assets/Pasted%20image%2020261120225654.png)

数据不一致情况：
删缓存失败，导致Redis的数据还是旧值

![](assets/Pasted%20image%2020261120225737.png)


### 删除重试

引入消息队列，由对应服务处理，但导致代码复杂度提高，耦合度变高
![](assets/Pasted%20image%2020261120230030.png)

解耦：**cannal**（阿里开源框架）
>读取binlog异步删除

当MySQL的数据发生变动，canal通知canal
![](assets/Pasted%20image%2020261120230459.png)




---
>来源：https://www.bilibili.com/video/BV11F411C77V?spm_id_from=333.788.videopod.sections&vd_source=d4cf9a1e1c6dd336272779f2e79469bf