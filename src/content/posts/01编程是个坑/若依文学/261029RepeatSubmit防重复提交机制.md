---
title: 261029RepeatSubmit防重复提交机制
published: 2026-10-29
description: 261029RepeatSubmit防重复提交机制
tags: []
category: 
draft: true
---
来自若依

发现一个自定义注解`@RepeatSubmit`，其描述：

>防止重复提交(参考美团GTIS防重系统)

检索全文其使用入口：

![](assets/Pasted%20image%2020261029141812.png)
![](assets/Pasted%20image%2020261029141857.png)
...

在维护入口前调用

RepeatSubmitAspect

进入before，首先校验@RepeatSubmit内是否有interval参数值，默认5000ms，如果不符合最低间隔时长1000ms，返回报错。




