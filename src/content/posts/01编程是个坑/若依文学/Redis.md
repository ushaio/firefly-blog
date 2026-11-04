---
title: 未命名
published: 2026-10-29
description: 未命名
tags: []
category:
draft: true
---

# Redis 使用

> **一句话总结**：以 Redisson 为核心，覆盖缓存（本地 + 分布式二级）、会话、防重、限流、分布式锁、消息广播、发号器。

## 1. 基础设施

| 组件           | 说明                                                                                                             |
| ------------ | -------------------------------------------------------------------------------------------------------------- |
| Redisson 客户端 | `RedisConfiguration` 定制序列化（Jackson3 JSON codec、`LocalDateTime` 格式）、单机 / 集群模式、连接池、Lua 脚本缓存                      |
| Key 前缀       | `KeyPrefixHandler` + `redisson.keyPrefix`，自动给所有 key 加前缀；业务常量统一用 `GlobalConstants.GLOBAL_REDIS_KEY = "global:"` |
| 异常处理         | `RedisExceptionHandler`                                                                                        |
| 工具类          | `RedisUtils`：封装 `String` / `List` / `Set` / `Map` / `RMapCache` / 原子数 / keys 扫描 / 发布订阅 / 限流 / 监听器              |

## 2. Spring Cache 多级缓存（重点）

- `CacheConfiguration`（`:19` 标注 `@EnableCaching`）+ `PlusSpringCacheManager`：以 Redisson `RMap` / `RMapCache` 作为分布式缓存，外层再套 Caffeine 本地缓存（默认 30s），形成二级缓存。
- `cacheName` 支持参数化配置，格式为：

  ```
  名称#TTL#最大空闲#最大容量#是否启用本地缓存
  ```

  示例：

  ```
  demo:cache#60s#10m#20#1
  ```

  实现见 `PlusSpringCacheManager.java:150-180`。
- `CacheUtils` 提供 `get` / `put` / `evict` / `clear`。
- 业务落点：`@Cacheable` / `@CachePut` / `@CacheEvict` 用于字典、参数配置、部门、角色自定义权限、用户昵称 / 用户名、OSS、客户端、流程分类等（如 `SysDictTypeServiceImpl`、`SysConfigServiceImpl`、`SysDeptServiceImpl`）。

## 3. 认证与会话

- **Sa-Token 持久层** `PlusSaTokenDao`：Caffeine（5s）+ Redis 存储 token / session，登录态、踢人、续期全靠它。
- **在线用户**：`ONLINE_TOKEN_KEY`（`online_tokens:`）记录 `UserOnline`，登出 / 被踢时删除（`UserLoginSuccessListener`、`UserActionListener`）。
- **登录失败锁定**：`PWD_ERR_CNT_KEY`（`pwd_err_cnt:`）累计错误次数并设置 TTL（`SysLoginService.java:199-222`）。
- **验证码**：图形验证码（`CaptchaController.java:86`）、短信 / 邮件验证码（`SysSmsController`、`SysEmailController`）。
- **社交登录 state**：`AuthRedisStateCache`。

## 4. 防重 / 限流 / 分布式锁

| 注解 | 实现 |
| --- | --- |
| `@RepeatSubmit` | `SET NX PX`（Redisson `setIfAbsent`） |
| `@RateLimiter` | Redisson `RRateLimiter`（集群 / 单机维度） |
| `@Lock4j` | Lock4j + Redisson 分布式锁，`Lock4jConfiguration` 自定义抢锁失败策略 |

## 5. 发布订阅 / 集群广播

- `RedisUtils.publish` / `subscribe`：SSE、WebSocket 在集群下把消息广播到所有节点（`SseEmitterSessionManager.java:191,283`、`WebSocketSessionManager.java:154,217`）。
- **Spring Cloud Bus**：`ruoyi-common-bus` 自定义 `RedisMessageChannelBinder`，基于 Redisson `RStream` 实现总线（配置刷新等）。
- `RedisUtils.addObjectListener` / `addMapListener` / … 支持 key 变更回调。

## 6. 其他

| 场景 | 实现 |
| --- | --- |
| 发号器 | `SequenceUtils`：Redisson `RIdGenerator`，支持按天 / 分钟重置（订单号等） |
| 短信验证码存储 | `PlusSmsDao` |
| OSS 默认配置 | `OssConstant.DEFAULT_CONFIG_KEY` 存当前默认配置，`OssConfigChangeListener` 同步 |
| 演示代码 | `RedisCacheController`（Spring Cache）、`RedisPubSubController`（发布订阅）、`RedisLockController`（分布式锁） |
