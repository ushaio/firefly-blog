---
title: SpringBoot的泪
published: 2026-10-25
description: SpringBoot的泪
tags:
  - java
  - springboot
category: java
draft: true
---
面试被问到SpringBoot和Spring有什么区别，一下子傻了眼，遂回答：依赖注入、控制反转、面向切片编程、自动装配、集成Tomcat和Jetty内置服务，省去大量配置步骤。回答完后，双方对视尴尬一笑。果不其然，回答的可以说是驴头不对马嘴。。。


DI、AOP、IOC...属于SpringFramework的概念，麻了，乱七八糟的了感觉，以前SSE、SpringMVC、Spring晕头转向了要。


# Spring Framework

## 控制反转IOC

>把对象的创建和生命周期交给容器管理





## 依赖注入DI

依赖注入？依赖啥？注入啥？

> **IoC 的具体实现方式**

前置基础代码举例：
```java
    @Override
    public User findById(int id) {
        // 这里写具体的 JDBC / SQL 语句，去 MySQL 查数据返回
        System.out.println("去 MySQL 查了用户数据...");
        return new User();
    }

    @Override
    public void save(User user) {
        System.out.println("把用户存到了 MySQL...");
    }
}
```

**没有依赖注入前：**
```java
public class UserService {
    // 自己控制 UserDao 的创建和生命周期 （多态）
    private UserDao userDao = new UserDaoImpl(); 
}
```

通过依赖注入实现：
```java
@Service
public class UserService {
    private final UserDao userDao;

    // 容器自动把 UserDao 的实现注入进来
    // (构造器方式注入)
    public UserService(UserDao userDao) {
        this.userDao = userDao;
    }
}
```

## 面向切片编程AOP

玄学概念，对着个理解一直感觉很抽象，比较简单的例子，过去开发时，常在Service层业务方法需要简化获取租户ID，即在方法上添加`@TenantScope`即可在方法下的传参。

>AI：AOP 就是“在不修改业务代码的前提下，在方法执行的前后悄悄插入一段通用逻辑”。

```java
@TenantScope // 自动获取tenantId塞进param中
private List<DailyDetail> getDailyDetail(Map<String,Object> param){
	// TODO
}
```
**梳理：**
1. 项目启动时，Spring 扫描到 Bean 中有切点规则（例如标记了自定义注解 `@TenantScope`）；
	> 这也就是为啥每次改注解要重新启动项目的原因了啊，我丢。。。
2. Spring 不会直接把原对象暴露给调用方，而是造了一个“替身”——代理对象（Proxy）（通过 JDK 动态代理或 CGLIB 生成）；
3. 当外部调用该方法时，实际调用的是**代理对象**：
	`调用方 -> [代理对象：执行前置切面逻辑] -> [真实业务方法] -> [代理对象：执行后置/异常/事务逻辑] -> 返回结果`

另一个场景事务注解`@Transactional`
```java
@Transactional
public CMReturn insertRow(DataRow datarow){
	// TODO
}
```
因此，也出现了一个问题：**同一个类中，方法A调用带有事务注解的方法B，会导致事务失效。**
- 外部（如 Controller）调 Service 时，拿到的是**代理对象**，会走 AOP 事务拦截；
- 但如果是类内部调用：`this.b()`，这里的 `this` 是**原始目标对象**，而不是 Spring 包装的**代理对象**，直接**绕过了 AOP 切面，因此事务根本不会生效**。


### 构造器注入

```java
@Service
public class UserService {
    private final UserDao userDao;

    // 容器自动把 UserDao 的实现注入进来
    public UserService(UserDao userDao) {
        this.userDao = userDao;
    }
}
```

### **字段注入**

**`@Autowired`**
- **来源**：Spring 原生注解。
- **匹配规则**：默认优先按照**类型（byType）**匹配；若存在多个同类型 Bean，会退化按**属性名**匹配。
- **指定名称**：搭配 `@Qualifier("beanName")`。
- **特点**：支持 `required = false`（允许未找到 Bean 也不报错）。

**`@Resource`**
- **来源**：Java 标准规范（JSR-250，`javax.annotation` / `jakarta.annotation`）。
- **匹配规则**：默认优先按照**名称（byName）**匹配（未指定 `name` 时取当前变量名作为 Bean 名称）；找不到则退化为按**类型（byType）**匹配。
- **指定名称**：自带 `name` 属性，如 `@Resource(name = "userDaoImpl")`（此时只按名称查，找不到直接报错）。

**`@Inject`**
- **来源**：JSR-330 规范，需单独引依赖（`jakarta.inject`）。
- **匹配规则**：类似 `@Autowired`，默认按**类型（byType）**匹配。
- **指定名称**：搭配 `@Named("beanName")`。

**`@RequiredArgsConstructor` (Lombok，推荐最佳实践)**
- **本质**：构造器注入的优雅实现，告别繁琐的手写构造器与字段注入的弊端。
- **用法**：
  ```java
  @Service
  @RequiredArgsConstructor // 自动生成包含所有 final 字段的构造方法
  public class UserService {
      private final UserDao userDao; // 必须是 final
  }
  ```
- **优势**：
  1. 保证了依赖的不可变性（`final`）；
  2. 便于脱离 Spring 容器进行单元测试；
  3. 彻底避免了字段注入可能引发的循环依赖掩盖与空指针异常。

### Setter 方法注入

在类的公开 `setter` 方法上标注 `@Autowired`（或 `@Resource`），由 Spring 容器在实例化 Bean 后调用方法完成注入。

**代码示例：**
```java
@Service
public class UserService {
    private UserDao userDao;

    // 实例化对象后，Spring 会自动调用该 Setter 方法注入依赖
    @Autowired
    public void setUserDao(UserDao userDao) {
        this.userDao = userDao;
    }
}
```

**特点与优缺点：**
- **适用场景（可选依赖）**：非常适合有**默认值**或**非必需**的依赖。当配合 `@Autowired(required = false)` 时，如果容器中找不到对应的 Bean 也不会启动报错，允许依赖留空或走默认逻辑。
- **支持重新注入**：与构造器注入不同，Setter 方法允许在对象创建后被重新调用修改（具有可变性）。
- **缺点**：
  1. **无法修饰为 `final`**：破坏了依赖的不可变性（Immutability）；
  2. **空指针隐患**：对象实例化与属性注入是分步完成的，如果在使用该对象时依赖尚未注入完全，容易出现 `NullPointerException`；
  3. 样板代码较多（虽然也可用 Lombok `@Setter(onMethod_ = @Autowired)`，但一般首选构造器注入）。



# SpringBoot

面试官为什么会“对视尴尬一笑”？
>“回答：依赖注入、控制反转、面向切片编程、自动装配、集成Tomcat和Jetty内置服务...”

1. 自动配置（Auto-Configuration）：约定优于配置，告别 XML 和样板代码；
2. 起步依赖（Starter POMs）：聚合常用依赖并统一管理版本，解决依赖地狱；
3. 内嵌 Servlet 容器：内置 Tomcat / Jetty / Undertow，打成可执行 jar 直接 `java -jar` 跑；
4. 生产就绪监控（Actuator）：开箱即用的健康检查、指标监控（Metrics）、审计日志等。

### 自动配置

前：
![](assets/Pasted%20image%2020261025214339.png)
![](assets/Pasted%20image%2020261025214352.png)

现在：
零xml，直接在pom引入场景依赖
