---
title: Prisma
published: 2026-11-05
description: Prisma
tags: []
category: 
draft: true
---
目前Prisam用到的场景，MO Gallery Web开源、SaaS版（官网）。其主要分为三层职能，其中Client是业务里被频繁使用的一部分：

1. Schema（数据模型定义）：唯一模型定义来源，描述表、字段、关系、索引
2. Migrate（数据库结构变更历史）：把 schema 变化变成带版本号、可复现的 DDL 并应用，同时维护 `_prisma_migrations` 历史表
3. **Client（类型安全查询客户端）**：根据 schema 生成带类型的查询 API，业务代码不用写 SQL

Clinet的使用：

```ts
// 类型安全 + 关联一次写完，不用管外键顺序
const photo = await prisma.photo.create({
  data: {
    title, url,
    exif: { create: { width, height } },
    tags: { connect: [{ id: tagId }] },
  },
  include: { exif: true, tags: true },
})
```


# 遇到的一些坑

之前未注意其增量脚本的流程，导致dev和生产环境的迁移历史不一致，dev生成脚本的顺序和数据库的迁移历史的顺序不一致，最后导致prisma client生成报错。




- [ ] 补充。。。 (@2026-11-05)