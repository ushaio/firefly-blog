---
title: RabbitMQ 学习笔记
published: 2025-12-01
description: RabbitMQ 学习笔记
tags:
  - rabbitmq
  - 消息队列
category: 编程
draft: false
---

# RabbitMQ 

![RabbitMQ 概览图 1](https://cdn.nlark.com/yuque/0/2025/png/1452823/1763190591482-efab88cb-65ce-4eeb-8413-6db283c54ec6.png)

![RabbitMQ 概览图 2](https://cdn.nlark.com/yuque/0/2025/png/1452823/1763190781956-f390ecca-12b3-442f-a9bf-a7ee26055264.png)

## 六大模式

1. 简单模式
2. 工作模式
3. 发布/订阅模式：交换机
4. 路由模式
5. 主题模式
6. 发布/确认模式

## 工作原理

![RabbitMQ 工作原理](https://cdn.nlark.com/yuque/0/2025/png/1452823/1763191211598-4067c80b-f393-49b9-9c58-3e97336f9a19.png)

- **Broker**：接受和分发消息的应用，RabbitMQ Server 就是 Message Broker。
- **Virtual host**：
- **Connection**：
- **Channel**：信道。
- **Exchange**：message 到达 broker 的第一站，根据分发规则匹配查询 routing key，分发消息到 queue 中。类型：`direct`、`topic`、`fanout`。
- **Queue**：消息最终被送到这里等待 consumer 取走。
- **Binding**：exchange 和 queue 之间的虚拟连接，binding 中可以包含 routing key。Binding 信息被保存到 exchange 中的查询表中，用于 message 的分发依据。

## 环境

- [Windows 安装 RabbitMQ 消息队列超详细步骤（附加详细操作截屏）- Rainbow-Sea - 博客园](https://www.cnblogs.com/TheMagicalRainbowSea/p/19023068)
- [Downloads - Erlang/OTP](https://www.erlang.org/downloads)
- [Release Information | RabbitMQ](https://www.rabbitmq.com/release-information)

### 操作

```powershell
erl -version

# 开启管理页面：sbin 路径下
rabbitmq-plugins.bat enable rabbitmq_management

# root
# 关闭服务
net stop rabbitmq

# 开启服务
net start rabbitmq
```

管理页面：[http://localhost:15672/#/](http://localhost:15672/#/)

### 添加用户

操作路径：`D:\dev\RabbitMQ Server\rabbitmq_server-4.2.0\sbin`

```shell
# 添加用户
rabbitmqctl add_user admin shai

# 设置角色
rabbitmqctl set_user_tags admin administrator

# 设置权限
rabbitmqctl set_permissions -p "/" admin ".*" ".*" ".*"

# 查看用户
rabbitmqctl list_users
```

![添加用户](https://cdn.nlark.com/yuque/0/2025/png/1452823/1763197964183-9ae13623-6a5e-4b3c-8e89-b4d1a551bdeb.png)

## Java 使用 MQ

### Maven 配置

`pom.xml`：

```xml
<dependencies>
    <!-- 核心 -->
    <dependency>
        <groupId>com.rabbitmq</groupId>
        <artifactId>amqp-client</artifactId>
        <version>5.14.2</version>
    </dependency>
    <dependency>
        <groupId>commons-io</groupId>
        <artifactId>commons-io</artifactId>
        <version>2.20.0</version>
    </dependency>
</dependencies>

<!-- 指定 jdk 编译版本 -->
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <configuration>
                <source>8</source>
                <target>8</target>
            </configuration>
        </plugin>
    </plugins>
</build>
```

### 方法

#### Channel

##### 生产者

- `queueDeclare`：队列声明
  1. 队列名称
  2. 是否持久化
  3. 是否只供一个消费者消费
  4. 是否自动删除
- `basicPublish`：消息发送
  1. 发送到的交换机
  2. 路由 key——队列的名称
  3. 其他参数信息
     - `MessageProperties.OERSISTENT_TEXT_PLAIN`：消息持久化
  4. 发送的消息体
- `confirmSelect`：发布确认
- `addConfirmListener`：消息监听器，异步发布确认

##### ConfirmCallback

- `deliveryTag`：消息的标记
- `multiple`：是否为批量确认

```java
ConfirmCallback ackCallback = (deliveryTag, multiple) -> {
    System.out.println("已确认的消息：" + deliveryTag);
};

ConfirmCallback nackCallback = (deliveryTag, multiple) -> {
    System.out.println("未确认的消息：" + deliveryTag);
};
```

##### 消费者

- `basicConsume`：接收消息
  1. 队列名称
  2. 消费成功后是否自动应答
  3. 消费成功消费的回调：`DeliverCallback`
  4. 消费者取消消费的回调：`CancelCallback`
- `basicAck`：手动应答
  1. 消息的标记
  2. 是否批量应答：建议 `false`
- `basicQos`：策略
  - `1`：不公平分发
  - `2`：预取值是 2（非 1 情况）

### 消息生产者/消费者

1. 生产者发送消息给队列。
2. 消费者接收队列的消息。

![生产者与消费者](https://cdn.nlark.com/yuque/0/2025/png/1452823/1763258973899-1f1ba027-25ab-4efc-9521-1abef8227b3e.png)

#### 生产者

```java
/**
 * 生产者：发消息
 */
public class Producer {
    // 队列名称
    public static final String QUEUE_NAME = "hello";

    // 发消息
    public static void main(String[] args) {
        // 创建一个 ConnectionFactory
        ConnectionFactory factory = new ConnectionFactory();
        // 设置 rabbitmq 的 ip 端口，默认端口是 5672
        factory.setHost("127.0.0.1");
        // 用户名
        factory.setUsername("admin");
        // 密码
        factory.setPassword("shai");

        try {
            // 创建 Connection
            Connection connection = factory.newConnection();
            // 创建 Channel
            Channel channel = connection.createChannel();
            // 创建队列（队列名称、是否持久化、是否只供一个消费者消费、是否自动删除）
            channel.queueDeclare(QUEUE_NAME, false, false, false, null);

            // 发送消息
            String message = "I love u";
            // 发送一个消费（发送到的交换机、路由 key-队列的名称、其他参数信息、发送的消息体）
            channel.basicPublish("", QUEUE_NAME, null, message.getBytes());
            System.out.println("消息发送完毕！");
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
```

![生产者运行结果 1](https://cdn.nlark.com/yuque/0/2025/png/1452823/1763258747639-294ed9ef-4ce1-43c9-b7c3-407b07a970f0.png)

![生产者运行结果 2](https://cdn.nlark.com/yuque/0/2025/png/1452823/1763258669854-3730bdd1-5069-4b43-aef4-3049c278c3cb.png)

#### 消费者

```java
/**
 * 消费者
 */
public class Consumer {
    // 队列名称
    public static final String QUEUE_NAME = "hello";

    // 接收消息
    public static void main(String[] args) throws Exception {
        // 创建一个 ConnectionFactory
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("127.0.0.1");
        factory.setUsername("admin");
        factory.setPassword("shai");

        // 创建 Connection
        Connection connection = factory.newConnection();
        // 创建 Channel
        Channel channel = connection.createChannel();

        // 声明接收消息的回调
        DeliverCallback deliverCallback = (consumerTag, delivery) -> {
            String message = new String(delivery.getBody(), "UTF-8");
            System.out.println("接收到消息：" + message);
        };
        // 声明取消消息的回调
        CancelCallback cancelCallback = (consumerTag) -> {
            System.out.println("消息消费被中断");
        };

        // 接收消息（队列名称、消费成功后是否自动应答、消费成功消费的回调、消费者取消消费的回调）
        channel.basicConsume(QUEUE_NAME, true, deliverCallback, cancelCallback);
        System.out.println("消息接收结束！ ");
    }
}
```

![消费者运行结果](https://cdn.nlark.com/yuque/0/2025/png/1452823/1763260559110-310eb83e-3af2-46ce-b5d8-f54df3902102.png)

接收完消息之后，消费队列的消息也随之清空。

### 原理

工作队列（任务队列）。

**一个消息只能被处理一次，否则造成重复消费。**

![工作队列原理](https://cdn.nlark.com/yuque/0/2025/png/1452823/1763260857588-36e15439-ca95-4952-8077-bda8a503418b.png)

因此，工作线程在接收消息时，使用**轮询方式**处理消息，保证每个消息只被处理一次。

工作线程之间为**竞争关系**。

### 提取工具类

#### RabbitUtils

```java
/**
 * RabbitMQ 工具类
 */
public class RabbitUtils {
    /**
     * 获取连接通道
     *
     * @return
     * @throws Exception
     */
    public static Channel getChannel() throws Exception {
        ConnectionFactory factory = new ConnectionFactory();
        // 设置 rabbitmq 的 ip 端口，默认端口是 5672
        factory.setHost("127.0.0.1");
        // 用户名
        factory.setUsername("admin");
        // 密码
        factory.setPassword("shai");
        // 创建 Connection
        Connection connection = factory.newConnection();
        // 创建 Channel
        Channel channel = connection.createChannel();
        return channel;
    }
}
```

#### SleepUtils

```java
public class SleepUtils {
    public static void sleep(int seconds) {
        try {
            Thread.sleep(seconds * 1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

### 多工作线程 Worker

#### 工作线程

```java
public static void main(String[] args) throws Exception {
    Channel channel = RabbitUtils.getChannel();

    // 消息接收
    DeliverCallback deliverCallback = (consumerTag, delivery) -> {
        String message = new String(delivery.getBody(), "UTF-8");
        System.out.println("接收到的消息：" + message);
    };
    // 消息接收被取消
    CancelCallback cancelCallback = (consumerTag) -> {
        System.out.println(consumerTag + "消息接收被中断");
    };

    // 接收消息（队列名称、消费成功后是否自动应答、消费成功消费的回调、消费者取消消费的回调）
    channel.basicConsume(QUEUE_NAME, true, deliverCallback, cancelCallback);
}
```

添加多个工作线程，实现工作线程竞争关系的状态。

**允许多个实例创建**，以此创建多个工作线程。

![允许多个实例运行](https://cdn.nlark.com/yuque/0/2025/png/1452823/1763273097436-851f89c4-6eef-42a8-a28e-f9ce5cbad3cd.png)

#### 生产者

由控制台输入方式发送消息。

```java
public class Task01 {
    public static final String QUEUE_NAME = "hello";

    // 发送者
    public static void main(String[] args) throws Exception {
        Channel channel = RabbitUtils.getChannel();
        // 队列声明
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);

        // 从控制台接收消息
        Scanner scanner = new Scanner(System.in);
        while (scanner.hasNext()) {
            String message = scanner.next();
            // 发送一个消费（发送到的交换机、路由 key-队列的名称、其他参数信息、发送的消息体）
            channel.basicPublish("", QUEUE_NAME, null, message.getBytes());
            System.out.println("发送消息完成：" + message);
        }
    }
}
```

![多工作线程运行结果 1](https://cdn.nlark.com/yuque/0/2025/png/1452823/1763274038474-a4846d12-aa63-494e-8b47-96a0a721a2a4.png)

![多工作线程运行结果 2](https://cdn.nlark.com/yuque/0/2025/png/1452823/1763274047929-1cddee6c-30cb-4aa8-b552-56031ce67268.png)

![多工作线程运行结果 3](https://cdn.nlark.com/yuque/0/2025/png/1452823/1763274058084-b021d1a3-852b-40d3-85ca-84143220f2f4.png)

## 消息应答

> RabbitMQ 一旦向消费者传递一个消息，该消息就会标记未删除。
>
> 那么……若工作过程中，某一工作线程宕机（在处理一个过程较长的任务，并未全部完成）？

消息应答保证工作线程在接收消息并处理完消息，告诉 RabbitMQ 已经处理了，RabbitMQ 才会删除该消息。

### 自动应答

![自动应答](https://cdn.nlark.com/yuque/0/2025/png/1452823/1763274515391-eb98cec7-a966-4123-abea-ca0edf2de437.png)

接到消息就自动应答，但可能还有其他操作逻辑，无法保证其他逻辑过程中是否出现报错等错误情况。

### 消息应答方法

![消息应答方法](https://cdn.nlark.com/yuque/0/2025/png/1452823/1763274688365-e0bc286f-12ea-49ee-a8fa-0f5b2bf4f096.png?x-oss-process=image%2Fcrop%2Cx_0%2Cy_3%2Cw_654%2Ch_271)

### Multiple 批量应答

**不建议**，默认为 `false`。

可能出现其他消息处理完，但其中某个消息未处理完，已导致该批次的消息全部应答的错误情况。

### 消息自动重新入队

由其他消费者接替进行处理。

![消息重新入队 1](https://cdn.nlark.com/yuque/0/2025/png/1452823/1763275452361-fd61e8ed-ec2c-4a1e-bf6d-fe6353d20380.png)

![消息重新入队 2](https://cdn.nlark.com/yuque/0/2025/png/1452823/1763275466824-27995384-f5a7-4a86-84d1-ddc2b116f2b3.png)

### 消息手动应答

**默认使用自动应答。**需修改。

#### 生产者

```java
public class Task02 {
    public static final String QUEUE_NAME = "ack_queue";

    public static void main(String[] args) throws Exception {
        Channel channel = RabbitUtils.getChannel();
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);

        // 控制台输入消息
        Scanner scanner = new Scanner(System.in);
        while (scanner.hasNext()) {
            String message = scanner.next();
            channel.basicPublish("", QUEUE_NAME, null, message.getBytes());
            System.out.println("Task02发送消息完成：" + message);
        }
    }
}
```

#### 消费者

C1 消费者正常接收，C2 消费者沉睡 30s（**以此模拟消息处理失败的场景，将消息交由 C1 接替处理**）。

```java
public class Worker03 {
    public static final String QUEUE_NAME = "ack_queue";

    public static void main(String[] args) throws Exception {
        Channel channel = RabbitUtils.getChannel();
        System.out.println("C2等待接收消息时间：30s。。。");

        // 接收消息成功应答
        DeliverCallback deliverCallback = (consumerTag, delivery) -> {
            SleepUtils.sleep(30);
            System.out.println("接收到的消息：" + new String(delivery.getBody(), "UTF-8"));
            // 消息完成后，手动应答
            channel.basicAck(delivery.getEnvelope().getDeliveryTag(), false);
        };
        CancelCallback cancelCallback = (consumerTag) -> {
            System.out.println(consumerTag + "消费者取消消费接口回调逻辑");
        };

        boolean autoAck = false;
        channel.basicConsume(QUEUE_NAME, autoAck, deliverCallback, cancelCallback);
    }
}
```

**结论**：轮询到 C2 接收消息时，30s 内（还未接收消息期间）结束 C2，消息将由其他消费者（C1）接替处理。

## 持久化

### 队列持久化

保证消费者崩溃、RabbitMQ 重启等情况消息不丢失。

当前，**未持久化**，重启后，队列就都消失了。

![未持久化的队列](https://cdn.nlark.com/yuque/0/2025/png/1452823/1763278998399-90b6aadd-844a-4fe0-9f38-df7debd29569.png)

**若已存在队列，直接修改为持久化运行，将会报错，因此需要先删除原来队列重新开始。**

![修改队列持久化时报错](https://cdn.nlark.com/yuque/0/2025/png/1452823/1763280100792-d63dcf6a-68cd-4521-a591-39797cf479be.png)

```java
boolean durable = true;
channel.queueDeclare(QUEUE_NAME, durable, false, false, null);
```

![队列持久化结果](https://cdn.nlark.com/yuque/0/2025/png/1452823/1763280337925-8dc9df2e-9959-4628-83b8-f0c5dabd46ce.png)

### 消息持久化

> 经过队列持久化可以发现，RabbitMQ 重启后，其持久化的队列仍然存在，但是由生产者发送的消息并未持久化，在重启过程中丢失了，因为 RabbitMQ 是默认不持久化的……

`MessageProperties.PERSISTENT_TEXT_PLAIN`：消息持久化。

```java
channel.basicPublish("", QUEUE_NAME, MessageProperties.PERSISTENT_TEXT_PLAIN, message.getBytes());
```

### 不公平分发

> 由前面的初实践发现，多工作线程的处理机制是**轮询方式**，其可能导致资源分配不均匀。如给 C2 消费者设置沉睡时间，就需要等待 C2 处理完成，造成堆积，而 C1 早已运行完了。

**由消费者设置该属性。**

`channel.basicQos(1);`

```java
public static void main(String[] args) throws Exception {
    // ...
    // 设置为不公平分发
    channel.basicQos(1);

    boolean autoAck = false;
    channel.basicConsume(QUEUE_NAME, autoAck, deliverCallback, cancelCallback);
}
```

**若有未处理完的持久化消息，重启消费者会自动继续处理。**

### 预取值

> 不同于不公平分发——能者多劳，提前在信道设置好每个消费者的预处理消息。

![预取值示意图 1](https://cdn.nlark.com/yuque/0/2025/png/1452823/1763282795234-cf314902-68b8-49f9-be73-ccb9c3b61b5d.png)

![预取值示意图 2](https://cdn.nlark.com/yuque/0/2025/png/1452823/1763283927804-c2537d3e-3c96-406a-ad0b-1fedaebe75a1.png)

## 发布确认

> 解决发布消息不丢失问题，确保消息保存到了磁盘上（持久化成功）。
>
> **前提：**队列持久化、消息持久化（必须保存到磁盘上）。

**存在问题：**若未保存到磁盘上，则该消息持久化失败。

- 单个发布确认
- 批量确认
- 异步确认

### 单个发布确认

> 发一条等待一条确认，再继续发布。
>
> **缺点：**发布速度特别慢，可能导致阻塞。

```java
/**
 * @description: 单个发布确认
 * @author: shai
 * @date: 2025/11/16 20:27
 * @return: 发布1000个单独确认消息,耗时：127ms
 **/
public static void publishMessageIndividually() throws Exception {
    Channel channel = RabbitUtils.getChannel();
    // 队列声明
    String queueName = UUID.randomUUID().toString();
    channel.queueDeclare(queueName, false, false, false, null);
    // 开启发布确认
    channel.confirmSelect();

    // 开始时间
    long begin = System.currentTimeMillis();
    String message = "hello world";
    for (int i = 0; i < MESSAGE_COUNT; i++) {
        channel.basicPublish("", queueName, null, (message + i).getBytes());
        // 等待消息发送完毕
        if (channel.waitForConfirms()) {
            System.out.println("消息发送成功" + i);
        } else {
            System.out.println("消息发送失败" + i);
            return;
        }
    }

    // 结束时间
    long end = System.currentTimeMillis();
    System.out.println("发布" + MESSAGE_COUNT + "个单独确认消息,耗时：" + (end - begin) + "ms");
}
```

### 批量发布确认

> 和单个发布确认相比，其是先发布一批消息，然后一起确认，可以极大提高吞吐量。
>
> **缺点：**如果其中某个消息发布过程出现问题，就不知道哪个消息出现问题。因此必须将这整个批处理保存在内存中，然后重新发布。

```java
/**
 * @description: 批量发布确认
 * @author: shai
 * @date: 2025/11/16 20:27
 * @return: 批量发布1000个单独确认消息, 耗时：82ms
 **/
public static void publishMessageBatch() throws Exception {
    Channel channel = RabbitUtils.getChannel();
    // 队列声明
    String queueName = UUID.randomUUID().toString();
    channel.queueDeclare(queueName, false, false, false, null);
    // 开启发布确认
    channel.confirmSelect();

    // 批量确认消息大小
    int batchSize = 100;
    long begin = System.currentTimeMillis();
    String message = "hello world";
    for (int i = 0; i < MESSAGE_COUNT; i++) {
        channel.basicPublish("", queueName, null, (message + i).getBytes());
        // 判断达到批量消息大小
        if (i % batchSize == 0) {
            // 批量确认
            channel.waitForConfirms();
        }
    }

    long end = System.currentTimeMillis();
    System.out.println("批量发布" + MESSAGE_COUNT + "个单独确认消息,耗时：" + (end - begin) + "ms");
}
```

### 异步发布确认 🌟

> 如其名，只管发送即可，保留了确认发布机制，以及无需等待其他批次的发布确认结果。对于未成功发布的消息，利用回调函数保证消息发布是否成功。

![异步发布确认](https://cdn.nlark.com/yuque/0/2025/png/1452823/1763298379786-702862ce-0009-40db-aa5f-d227c19c270d.png)

- `deliveryTag`：消息的标记
- `multiple`：是否为批量确认

```java
ConfirmCallback ackCallback = (deliveryTag, multiple) -> {
    System.out.println("已确认的消息：" + deliveryTag);
};

ConfirmCallback nackCallback = (deliveryTag, multiple) -> {
    System.out.println("未确认的消息：" + deliveryTag);
};
```

```java
/**
 * @description: 异步发布确认
 * @author: shai
 * @date: 2025/11/16 20:27
 * @return: 批量发布1000个单独确认消息, 耗时：35ms
 **/
public static void publishMessageAsync() throws Exception {
    Channel channel = RabbitUtils.getChannel();
    // 队列声明
    String queueName = UUID.randomUUID().toString();
    channel.queueDeclare(queueName, false, false, false, null);
    // 开启发布确认
    channel.confirmSelect();

    long begin = System.currentTimeMillis();
    String message = "hello world";
    // 添加一个异步监听器（在发送消息前），回调函数
    ConfirmCallback ackCallback = (deliveryTag, multiple) -> {
        System.out.println("已确认的消息：" + deliveryTag);
    };
    ConfirmCallback nackCallback = (deliveryTag, multiple) -> {
        System.out.println("未确认的消息：" + deliveryTag);
    };
    channel.addConfirmListener(ackCallback, nackCallback);

    for (int i = 0; i < MESSAGE_COUNT; i++) {
        channel.basicPublish("", queueName, null, (message + i).getBytes());
    }

    long end = System.currentTimeMillis();
    System.out.println("异步发布" + MESSAGE_COUNT + "个单独确认消息,耗时：" + (end - begin) + "ms");
}
```

#### 如何处理异步未确认的消息

![处理异步未确认消息](https://cdn.nlark.com/yuque/0/2025/png/1452823/1763299962940-fc2bb1ae-94c7-49f6-8b49-9235f4f42fd0.png)

**ConcurrentSkipListMap**：

- 轻松地将序号和消息进行关联。
- 轻松批量删除条目，只要给到序号。
- 支持高并发（多线程）。

**ConcurrentNavigableMap**：

```java
public static void publishMessageAsync() throws Exception {
    Channel channel = RabbitUtils.getChannel();
    // 队列声明
    String queueName = UUID.randomUUID().toString();
    channel.queueDeclare(queueName, false, false, false, null);
    // 开启发布确认
    channel.confirmSelect();

    long begin = System.currentTimeMillis();
    String message = "hello world";
    // 创建一个线程安全有序的哈希表，适用于高并发
    ConcurrentSkipListMap<Long, String> outstandingConfirms = new ConcurrentSkipListMap<>();

    // 添加一个异步监听器（在发送消息前），回调函数
    ConfirmCallback ackCallback = (deliveryTag, multiple) -> {
        // 批量确认
        if (multiple) {
            // 删除已确认的消息，剩下的就是未确认的消息
            ConcurrentNavigableMap<Long, String> confirmed = outstandingConfirms.headMap(deliveryTag);
            confirmed.clear();
        } else {
            // 单个确认
            outstandingConfirms.remove(deliveryTag);
        }
        System.out.println("已确认的消息：" + deliveryTag);
    };
    ConfirmCallback nackCallback = (deliveryTag, multiple) -> {
        // 打印未确认的消息
        String msg = outstandingConfirms.get(deliveryTag);
        System.out.println("未确认的消息：" + msg + ";;;tag:" + deliveryTag);
    };
    channel.addConfirmListener(ackCallback, nackCallback);

    for (int i = 0; i < MESSAGE_COUNT; i++) {
        channel.basicPublish("", queueName, null, (message + i).getBytes());
        // 添加消息到待确认列表
        outstandingConfirms.put(channel.getNextPublishSeqNo(), message + i);
    }

    long end = System.currentTimeMillis();
    System.out.println("异步发布" + MESSAGE_COUNT + "个单独确认消息,耗时：" + (end - begin) + "ms");
}
```

**ConcurrentLinkedQueue**

### 结论

这时间感觉和教程老师的差别有点大啊 =-=！！！！

按照理论来讲，速度：**异步 > 批量 > 单个**。

## 交换机

![交换机示意图 1](https://cdn.nlark.com/yuque/0/2025/png/1452823/1763301497636-703d1e60-ebcc-40d1-92a6-b45c15ab9667.png)

![交换机示意图 2](https://cdn.nlark.com/yuque/0/2025/png/1452823/1763301515764-fbbc7aa2-5480-498d-8313-11b957514283.png)

生产者发送消息并不是直接发送给消息队列，**只能发送给交换机**。

### Exchanges

#### 交换机类型

直接（`direct`）、主题（`topic`）、扇出（`fanout`）。

#### 无名交换机

![无名交换机](https://cdn.nlark.com/yuque/0/2025/png/1452823/1763301851682-96518017-80a1-4a48-9608-b6c6f5c1a91b.png)

此时，交换机参数为空字符串，即**默认交换机——无名交换机**。

```java
channel.basicPublish("", queueName, null, message.getBytes());
```

![默认交换机配置](https://cdn.nlark.com/yuque/0/2025/png/1452823/1763301800657-154e32b6-251a-497b-9586-7ac3e6cbe4aa.png)

### 临时队列

![临时队列](https://cdn.nlark.com/yuque/0/2025/png/1452823/1763302092967-0f2478aa-07b8-4dab-b1c8-762ebfdcc9cb.png)

```java
String queueName = channel.queueDeclare().getQueue();
```

### 绑定 Bindings

binding 是 exchange 和 queue 之间的桥梁，描述了 exchange 和 queue 之间的绑定关系。

![绑定关系](https://cdn.nlark.com/yuque/0/2025/png/1452823/1763302700679-32e7dde2-7ac1-4128-807b-b778d85bb0ab.png)

根据 RoutingKey 找到对应队列：`RoutingKey → QueueName`。

可以通过 RoutingKey 匹配到指定队列。

### 扇出 Fanout

> 将接收到的所有消息广播到它知道的所有队列中。
>
> **routingKey 设置无效。**

![Fanout 示意图 1](https://cdn.nlark.com/yuque/0/2025/png/1452823/1763302914968-74a95832-c710-4dda-b5f0-91b30e501ca9.png)

![Fanout 示意图 2](https://cdn.nlark.com/yuque/0/2025/png/1452823/1763303031697-eb4b5c59-a960-4a31-bca1-a9e5107895d0.png)

#### 消息接收

```java
/**
 * @Description: 消息接收
 * @Author: shai
 * @CreateTime: 2025-11-16 22:30
 * @Version: 1.0
 */
public class ReceiveLogs01 {
    public static final String EXCHANGE_NAME = "logs";

    public static void main(String[] args) throws Exception {
        Channel channel = RabbitUtils.getChannel();
        // 声明交换机
        channel.exchangeDeclare(EXCHANGE_NAME, "fanout");
        // 声明临时队列（断开连接即删除）
        String queueName = channel.queueDeclare().getQueue();
        // 绑定队列到交换机
        channel.queueBind(queueName, EXCHANGE_NAME, "");
        System.out.println("ReceiveLogs01等待接收消息...");

        // 接收消息
        DeliverCallback deliverCallback = (consumerTag, delivery) -> {
            String message = new String(delivery.getBody(), "UTF-8");
            System.out.println("控制台ReceiveLogs01打印接收到消息：" + message);
        };
        channel.basicConsume(queueName, true, deliverCallback, cancelCallback -> {
        });
    }
}
```

#### 发送消息

```java
/**
 * @Description: 发送消息
 * @Author: shai
 * @CreateTime: 2025-11-16 22:46:00
 * @Version: 1.0
 */
public class EmitLog {
    public static final String EXCHANGE_NAME = "logs";
    public static final String QUEUE_NAME = "ack_queue";

    public static void main(String[] args) throws Exception {
        Channel channel = RabbitUtils.getChannel();
        // 声明交换机
        channel.exchangeDeclare(EXCHANGE_NAME, "fanout");

        // 控制台输入消息
        Scanner scanner = new Scanner(System.in);
        while (scanner.hasNext()) {
            String message = scanner.next();
            channel.basicPublish(EXCHANGE_NAME, "", MessageProperties.PERSISTENT_TEXT_PLAIN, message.getBytes());
            System.out.println("EmitLog发送消息完成：" + message);
        }
    }
}
```

**限制**：只能进行无意识的广播，如果需要记录指定等级（错误级别）的日志，那么该方式将无法达到，并且会浪费磁盘空间。

### 直接 Direct

> 相比于 fanout，direct 可以让消息只去它绑定的 routingKey 队列中。

![Direct 示意图](https://cdn.nlark.com/yuque/0/2025/png/1452823/1763366643557-b234f493-1056-43cc-8206-ee2a306fdac4.png)

#### 绑定关系

**ReceiveLogsDirect01**：`direct_logs → info / warn → console`

```java
/**
 * 接收者1：监听 info、warn 的消息
 */
public class ReceiveLogsDirect01 {
    public static final String EXCHANGE_NAME = "direct_logs";

    public static void main(String[] args) throws Exception {
        Channel channel = RabbitUtils.getChannel();
        // 声明交换机
        channel.exchangeDeclare(EXCHANGE_NAME, BuiltinExchangeType.DIRECT);
        // 声明队列
        channel.queueDeclare("console", false, false, false, null);
        // 绑定
        channel.queueBind("console", EXCHANGE_NAME, "info");
        // 多重绑定
        channel.queueBind("console", EXCHANGE_NAME, "warn");

        // 消费
        DeliverCallback deliverCallback = (consumerTag, message) -> {
            System.out.println("ReceiveLogsDirect01接收消息：" + new String(message.getBody(), StandardCharsets.UTF_8));
        };
        channel.basicConsume("console", deliverCallback, consumerTag -> {
        });
    }
}
```

**ReceiveLogsDirect02**：`direct_logs → error → console`

```java
/**
 * 接收者2：监听 error 级别的消息
 */
public class ReceiveLogsDirect02 {
    public static final String EXCHANGE_NAME = "direct_logs";

    public static void main(String[] args) throws Exception {
        Channel channel = RabbitUtils.getChannel();
        // 声明交换机
        channel.exchangeDeclare(EXCHANGE_NAME, BuiltinExchangeType.DIRECT);
        // 声明队列
        channel.queueDeclare("disk", false, false, false, null);
        // 绑定
        channel.queueBind("disk", EXCHANGE_NAME, "error");

        // 消费
        DeliverCallback deliverCallback = (consumerTag, message) -> {
            System.out.println("ReceiveLogsDirect02接收消息：" + new String(message.getBody(), StandardCharsets.UTF_8));
        };
        channel.basicConsume("disk", deliverCallback, consumerTag -> {
        });
    }
}
```

#### DirectLogs：发送消息

```java
/**
 * @Description: 发送消息
 * @Author: shai
 * @CreateTime: 2025-11-16 22:46:00
 * @Version: 1.0
 */
public class DirectLogs {
    public static final String EXCHANGE_NAME = "direct_logs";

    public static void main(String[] args) throws Exception {
        Channel channel = RabbitUtils.getChannel();
        Scanner scanner = new Scanner(System.in);
        while (scanner.hasNext()) {
            String message = scanner.next();
            channel.basicPublish(EXCHANGE_NAME, "error", MessageProperties.PERSISTENT_TEXT_PLAIN, message.getBytes());
            System.out.println("DirectLogs：" + message);
        }
    }
}
```

生产者发送的消息，消费者只能根据其消息的 routingKey 来接收，如果为不存在的 routingKey，则都接收不到消息。

#### 多重绑定

> Direct 下，一个队列可绑定多个 routingKey，借此，可以接收多个 routingKey 的消息。
>
> **缺点**：无法使用通配符 `*`，管理成本高。

其实 Topic 也可以理解为支持多重绑定吧，形式不一样而已，一个精准匹配，一个支持了通配符格式匹配。

![多重绑定](https://cdn.nlark.com/yuque/0/2025/png/1452823/1763430029222-a22e6037-65e5-4d86-871d-6b8a6e5c4411.png)

```java
// Direct
channel.queueBind(queueName, EXCHANGE_NAME, "info.base");
channel.queueBind(queueName, EXCHANGE_NAME, "info.warn");

// Topic
channel.queueBind(queueName, EXCHANGE_NAME, "*.*.rabbit");
channel.queueBind(queueName, EXCHANGE_NAME, "lazy.#");
```

### 主题 Topic

> 比 fanout 和 direct 更完美。
>
> 对于直接交换机（Direct Exchange），仍然存在局限性的。比如，一个消费者要接收的日志类型有 `info.base` 和 `info.advantage`，而另一个消费者只想接收 `info.base` 的消息，那这个时候 `direct` 就办不到了，只能使用 `topic`。

- **支持通配符**，无需精准匹配每个 routingKey，比 Direct 方便。

![Topic 示意图](https://cdn.nlark.com/yuque/0/2025/png/1452823/1763431190843-60754b71-49a2-40e6-8637-e7ec15903eb2.png)

#### 消费者

**消费者 1**：

```java
public static void main(String[] args) throws Exception {
    Channel channel = RabbitUtils.getChannel();
    // 声明交换机
    channel.exchangeDeclare(EXCHANGE_NAME, BuiltinExchangeType.TOPIC);
    // 声明队列
    String queueName = "Q1";
    channel.queueDeclare(queueName, false, false, false, null);
    // 绑定，匹配规则：*.orange.*
    channel.queueBind(queueName, EXCHANGE_NAME, "*.orange.*");
    System.out.println("ReceiveLogsTopic01等待接收消息...");

    // 消费
    DeliverCallback deliverCallback = (consumerTag, message) -> {
        System.out.println("ReceiveLogsTopic01接收消息：" + new String(message.getBody(), StandardCharsets.UTF_8));
        System.out.println("接收队列：" + queueName + ";;;绑定键：" + message.getEnvelope().getRoutingKey());
    };
    channel.basicConsume(queueName, deliverCallback, consumerTag -> {
    });
}
```

**消费者 2**：

```java
public static void main(String[] args) throws Exception {
    Channel channel = RabbitUtils.getChannel();
    // 声明交换机
    channel.exchangeDeclare(EXCHANGE_NAME, BuiltinExchangeType.TOPIC);
    // 声明队列
    String queueName = "Q2";
    channel.queueDeclare(queueName, false, false, false, null);
    // 绑定，匹配规则：*.orange.*、lazy.#
    channel.queueBind(queueName, EXCHANGE_NAME, "*.*.rabbit");
    channel.queueBind(queueName, EXCHANGE_NAME, "lazy.#");
    System.out.println("ReceiveLogsTopic02等待接收消息...");

    // 消费
    DeliverCallback deliverCallback = (consumerTag, message) -> {
        System.out.println("ReceiveLogsTopic02接收消息：" + new String(message.getBody(), StandardCharsets.UTF_8));
        System.out.println("接收队列：" + queueName + ";;;绑定键：" + message.getEnvelope().getRoutingKey());
    };
    channel.basicConsume(queueName, deliverCallback, consumerTag -> {
    });
}
```

#### 发送者

```java
/**
 * Q1：*.orange.*
 * Q2：*.*.rabbit
 * Q2：lazy.#
 */
public static void main(String[] args) throws Exception {
    Channel channel = RabbitUtils.getChannel();
    Map<String, String> bindingKeyMap = new HashMap<>();
    bindingKeyMap.put("quick.orage.rabbit", "Q1;Q2");
    bindingKeyMap.put("lazy.orange.elephant", "Q1;Q2");
    bindingKeyMap.put("quick.orange.fox", "Q1");
    bindingKeyMap.put("lazy.brown.fox", "Q2");
    bindingKeyMap.put("lazy.pink.rabbit", "虽然满足两个绑定但只被队列Q2接收");
    bindingKeyMap.put("quick.brown.fox", "不匹配任何绑定不会被任何队列接收到会被丢弃");
    bindingKeyMap.put("lazy.orange.male.rabbit", "是四个单词但匹配Q2");

    for (Map.Entry<String, String> bindingKeyEntry : bindingKeyMap.entrySet()) {
        String routingKey = bindingKeyEntry.getKey();
        channel.basicPublish(EXCHANGE_NAME, routingKey, null, bindingKeyEntry.getValue().getBytes());
    }
}
```

#### 结果

```text
# ReceiveLogsTopic01
ReceiveLogsTopic01接收消息：Q1;Q2
接收队列：Q1;;;绑定键：lazy.orange.elephant
ReceiveLogsTopic01接收消息：Q1
接收队列：Q1;;;绑定键：quick.orange.fox

# ReceiveLogsTopic02
ReceiveLogsTopic02接收消息：Q1;Q2
接收队列：Q2;;;绑定键：lazy.orange.elephant
ReceiveLogsTopic02接收消息：Q2
接收队列：Q2;;;绑定键：lazy.brown.fox
ReceiveLogsTopic02接收消息：Q1;Q2
接收队列：Q2;;;绑定键：quick.orage.rabbit
ReceiveLogsTopic02接收消息：虽然满足两个绑定但只被队列Q2接收
接收队列：Q2;;;绑定键：lazy.pink.rabbit
ReceiveLogsTopic02接收消息：是四个单词但匹配Q2
接收队列：Q2;;;绑定键：lazy.orange.male.rabbit
```

## 死信队列

> 无法被消费的消息。

![死信队列](https://cdn.nlark.com/yuque/0/2025/png/1452823/1763441846410-0a8d63e2-e352-4e04-9a1d-277044bde38a.png)

**场景**：

- 保证订单业务的消息数据不丢失，当消息消费异常时，将消息投入死信队列中。
- 用户在商城下单成功并点击去支付后在指定时间未支付，订单自动失效。

**来源**：

1. 消息 TTL 过期。
2. 队列达到最大长度。
3. 消息被拒绝。

![死信队列架构](https://cdn.nlark.com/yuque/0/2025/png/1452823/1763442099402-ebaadb03-4361-4858-8533-2bd28d4a6f0b.png)

> 一个用于处理正常消息的队列，一个专门处理死信的队列。

- 一个生产者
- 两个交换机（正常交换机和死信交换机）
- 两个队列（正常队列和死信队列）
- 两个消费者（正常消费者和死信消费者）

### 消息 TTL 过期

#### 正常队列和死信队列

```java
/**
 * 正常消费者1
 */
public class Consumer1 {
    public static final String NORMAL_EXCHANGE = "normal_exchange";
    public static final String DEAD_EXCHANGE = "dead_exchange";
    public static final String NORMAL_QUEUE = "normal_queue";
    public static final String DEAD_QUEUE = "dead_queue";

    public static void main(String[] args) throws Exception {
        Channel channel = RabbitUtils.getChannel();
        // 声明死信和普通交换机
        channel.exchangeDeclare(NORMAL_EXCHANGE, BuiltinExchangeType.DIRECT);
        channel.exchangeDeclare(DEAD_EXCHANGE, BuiltinExchangeType.DIRECT);

        // 声明死信和普通队列
        Map<String, Object> arguments = new HashMap<>();
        // 绑定死信队列
        arguments.put("x-dead-letter-exchange", DEAD_EXCHANGE);
        // 设置死信 RoutingKey
        arguments.put("x-dead-letter-routing-key", "lisi");
        channel.queueDeclare(NORMAL_QUEUE, false, false, false, arguments);
        channel.queueDeclare(DEAD_QUEUE, false, false, false, null);

        // 绑定普通交换机与普通队列
        channel.queueBind(NORMAL_QUEUE, NORMAL_EXCHANGE, "zhangsan");
        // 绑定死信交换机与死信队列
        channel.queueBind(DEAD_QUEUE, DEAD_EXCHANGE, "lisi");
        System.out.println("Consumer1等待接收消息...");

        // 接收消息
        DeliverCallback deliverCallback = (consumerTag, message) -> {
            System.out.println("Consumer1接收到消息：" + new String(message.getBody(), StandardCharsets.UTF_8));
        };
        channel.basicConsume(NORMAL_QUEUE, deliverCallback, consumerTag -> {
        });
    }
}
```

![死信队列声明结果](https://cdn.nlark.com/yuque/0/2025/png/1452823/1763621760636-3d2984e3-5970-4524-84e6-191c6688e7f1.png)

#### 生产者

```java
/**
 * 生产者
 */
public class Producer {
    public static final String NORMAL_EXCHANGE = "normal_exchange";

    public static void main(String[] args) throws Exception {
        Channel channel = RabbitUtils.getChannel();
        // 延迟（死信）消息，设置 TTL
        // 如果 10s 内消费者 C1 未被接受，则转到死信队列给 C2
        AMQP.BasicProperties properties = new AMQP.BasicProperties()
                .builder().expiration("10000").build();

        for (int i = 0; i < 10; i++) {
            String message = "info-" + i;
            channel.basicPublish(NORMAL_EXCHANGE, "zhangsan", properties, message.getBytes());
        }
        channel.queueDeclare();
    }
}
```

![TTL 过期结果](https://cdn.nlark.com/yuque/0/2025/png/1452823/1763623261623-9712eec4-36fb-4505-942e-239146ac93e4.png)

#### 死信消费者

```java
/**
 * 死信消费者2
 */
public class Consumer2 {
    public static final String DEAD_EXCHANGE = "dead_exchange";
    public static final String DEAD_QUEUE = "dead_queue";

    public static void main(String[] args) throws Exception {
        Channel channel = RabbitUtils.getChannel();
        System.out.println("Consumer2等待接收消息...");

        // 接收消息
        DeliverCallback deliverCallback = (consumerTag, message) -> {
            System.out.println("Consumer2接收到消息：" + new String(message.getBody(), StandardCharsets.UTF_8));
        };
        channel.basicConsume(DEAD_QUEUE, deliverCallback, consumerTag -> {
        });
    }
}
```

```text
Consumer2等待接收消息...
Consumer2接收到消息：info-0
Consumer2接收到消息：info-1
Consumer2接收到消息：info-2
Consumer2接收到消息：info-3
Consumer2接收到消息：info-4
Consumer2接收到消息：info-5
Consumer2接收到消息：info-6
Consumer2接收到消息：info-7
Consumer2接收到消息：info-8
Consumer2接收到消息：info-9
```

### 队列达到最大长度

**Consumer1 修改，启动完成后停止**：

```java
// 声明死信和普通队列
Map<String, Object> arguments = new HashMap<>();
// 绑定死信队列
arguments.put("x-dead-letter-exchange", DEAD_EXCHANGE);
// 设置死信 RoutingKey
arguments.put("x-dead-letter-routing-key", "lisi");
// 设置正常队列长度限制
arguments.put("x-max-length", 6);
channel.queueDeclare(NORMAL_QUEUE, false, false, false, arguments);
```

**Producer**：

```java
// 注释掉
// AMQP.BasicProperties properties = new AMQP.BasicProperties()
//         .builder().expiration("10000").build();
```

![队列达到最大长度](https://cdn.nlark.com/yuque/0/2025/png/1452823/1763876621563-a182af89-62e7-40f5-83d1-1ca66b900a36.png)

### 消息被拒

> 消息如何被拒？

**正常消费者**：

```java
// false：不放回队列
channel.basicReject(message.getEnvelope().getDeliveryTag(), false);
```

```java
// arguments.put("x-max-length", 6);
...

// 接收消息
DeliverCallback deliverCallback = (consumerTag, message) -> {
    String msg = new String(message.getBody(), StandardCharsets.UTF_8);
    if (msg.equals("info-5")) {
        // 拒绝消息处理，移交死信（C1 消费 9 条，C2 消费 1 条）
        System.out.println("Consumer1拒绝了消息❌：" + msg);
        // false：不放回队列
        channel.basicReject(message.getEnvelope().getDeliveryTag(), false);
    } else {
        System.out.println("Consumer1接收到消息：" + msg);
        // false：不批量应答
        channel.basicAck(message.getEnvelope().getDeliveryTag(), false);
    }
};

// 开启手动应答
channel.basicConsume(NORMAL_QUEUE, false, deliverCallback, consumerTag -> {
});
```

![消息被拒后的处理结果](https://cdn.nlark.com/yuque/0/2025/png/1452823/1763877022885-b7fe5aae-ec37-4011-a179-0d7447fdccb3.png)

## 整合 Spring Boot

### 依赖与配置

```xml
<!-- RabbitMQ 测试依赖 -->
<dependency>
    <groupId>org.springframework.amqp</groupId>
    <artifactId>spring-rabbit-test</artifactId>
</dependency>
```

```properties
spring.rabbitmq.host=localhost
spring.rabbitmq.port=5672
spring.rabbitmq.username=admin
spring.rabbitmq.password=shai
```

### 延迟队列

![延迟队列示意图 1](https://cdn.nlark.com/yuque/0/2025/png/1452823/1763877705500-0450c2f3-b621-4636-9523-1d89ce94aa17.png)

![延迟队列示意图 2](https://cdn.nlark.com/yuque/0/2025/png/1452823/1763880057525-31c4b747-05ef-49d5-9b2f-fa1913992a67.png)

![延迟队列示意图 3](https://cdn.nlark.com/yuque/0/2025/png/1452823/1763882670185-b094e64d-7a4e-48e5-bcf8-1d17ac683bbe.png)

### 延迟队列优化

> 基于延迟队列，发现无法自定义控制 TTL 时间。

![延迟队列优化](https://cdn.nlark.com/yuque/0/2025/png/1452823/1763884411274-9d0cd034-b7cc-4fe8-920b-b4457f7fe164.png)

#### 生产者

```java
/**
 * @description: 基于上面的方法，无法设置自定义的 ttl 时间
 * @author: shai
 * @date: 2025/11/23 16:15
 * @param: [ttl, message]
 * @return: java.lang.String
 **/
@RequestMapping("sendExpirationMsg/{ttl}/{message}")
public String sendExpirationMsg(@PathVariable String ttl, @PathVariable String message) {
    log.info("当前时间：{}，发送一条时长 {}ms 的消息给两个TTL队列：{}", new Date(), ttl, message);
    // 通过 XC 实现交换机 X 给 QC 队列发送消息
    rabbitTemplate.convertAndSend("X", "XC", message, msg -> {
        msg.getMessageProperties().setExpiration(ttl);
        return msg;
    });
    return "发送消息成功";
}
```

#### 绑定关系

```java
// 队列（延迟队列优化）
@Bean("queueC")
public Queue queueC() {
    Map<String, Object> arguments = new HashMap<>(3);
    // 绑定死信队列 QD
    arguments.put("x-dead-letter-exchange", Y_DEAD_LETTER_EXCHANGE);
    arguments.put("x-dead-letter-routing-key", "YD");
    return QueueBuilder.durable(QUEUE_C)
            .withArguments(arguments)
            .build();
}

// 绑定 X 交换机 - QC 队列（延迟优化）
@Bean
public Binding queueCBindingX(@Qualifier("queueC") Queue queueC,
                              @Qualifier("xExchange") DirectExchange xEchange) {
    return BindingBuilder
            .bind(queueC)
            .to(xEchange)
            .with("XC");
}
```

#### 消费者

```java
@Slf4j
@Component
public class DeadLetterQueueConsumer {
    // 接收来自队列 QD 的消息
    @RabbitListener(queues = "QD")
    public void reveiveD(Message message, Channel channel) throws Exception {
        String msg = new String(message.getBody());
        log.info("当前时间：{}，收到死信队列的消息：{}", new Date(), msg);
    }
```

#### 死信延迟存在的问题

发送 hello1（TTL = 20s 进入死信队列），发送 hello2（TTL = 2s 进入死信队列）。

预期收到死信队列消息的时间：

| 消息 | 预期时间 | 实际时间 | 结果 |
| --- | --- | --- | --- |
| hello1 | 12 + 20 = 32 | 32 | ✅ |
| hello2 | 17 + 2 = 19 | 32 | ❌ |

出现该问题原因：hello2 实际在 19s 的时候已经收到消息了，但是由于前方还有 hello1 处于等待状态，因此需要等到 hello1 进入到死信队列，才能继续处理 hello2。

![死信延迟问题 1](https://cdn.nlark.com/yuque/0/2025/png/1452823/1764158999329-632e80c9-01ba-41b3-852e-7a391d13d661.png)

![死信延迟问题 2](https://cdn.nlark.com/yuque/0/2025/png/1452823/1764159265187-fab90a1d-4708-4240-a1f8-a56688ad7fec.png)

### 基于插件的延迟队列

#### 插件

> 解决死信延迟存在的问题。

`rabbitmq_delayed_message_exchange` 插件：

[Community Plugins | RabbitMQ](https://www.rabbitmq.com/community-plugins)

![延迟插件](https://cdn.nlark.com/yuque/0/2025/png/1452823/1764212091773-01110d34-8bf4-4057-b8e4-af6ac275e7f6.png)

**启用插件**：

```shell
rabbitmq-plugins enable rabbitmq_delayed_message_exchange
```

![启用延迟插件](https://cdn.nlark.com/yuque/0/2025/png/1452823/1764212133338-3026f118-630a-4fbb-980c-0929dc5aeb4a.png)

**检查生效**：

![检查插件生效](https://cdn.nlark.com/yuque/0/2025/png/1452823/1764212235694-fe588297-44b7-44e9-babf-52705f94658c.png)

**基于死信的**：

![基于死信的延迟](https://cdn.nlark.com/yuque/0/2025/png/1452823/1764214103357-5050ecc1-6352-4550-adf0-9d2393f6da2c.png?x-oss-process=image%2Fcrop%2Cx_0%2Cy_24%2Cw_570%2Ch_313)

**基于插件的**：延迟的位置处于**交换机**。

![基于插件的延迟](https://cdn.nlark.com/yuque/0/2025/png/1452823/1764214142444-392ef559-5ba0-4834-b87f-45ad6d08b66e.png)

#### 实现

![延迟队列实现](https://cdn.nlark.com/yuque/0/2025/png/1452823/1764214436827-6c7f6ec7-ccb0-4a51-97c0-f607f86bc52b.png)

##### 延迟队列 Config

创建自定义交换机 `CustomExchange`：

```java
/**
 * 延迟队列 x_delayed_message
 */
@Configuration
public class DelayedQueueConfig {
    // 交换机
    public static final String DELAYED_QUEUE_NAME = "delayed.queue";
    // 队列
    public static final String DELAYED_EXCHANGE_NAME = "delayed.exchange";
    // routingkey
    public static final String DELAYED_ROUTING_KEY = "delayed.routingkey";

    /**
     * 声明队列
     */
    @Bean
    public Queue delayedQueue() {
        return new Queue(DELAYED_QUEUE_NAME);
    }

    /**
     * 声明交换机
     */
    @Bean
    public CustomExchange delayedExchange() {
        Map<String, Object> arguments = new HashMap<>();
        arguments.put("x-delayed-type", "direct");
        /*
         * 1. 交换机名称
         * 2. 交换机类型
         * 3. 是否持久化
         * 4. 是否自动删除
         * 5. 其他参数
         */
        return new CustomExchange(
                DELAYED_EXCHANGE_NAME,
                "x-delayed-message",
                true,
                false,
                arguments
        );
    }

    /**
     * 延迟队列绑定延迟交换机
     */
    @Bean
    public Binding delayedQueueBindingDelayedExchange(
            @Qualifier("delayedQueue") Queue delayedQueue,
            @Qualifier("delayedExchange") CustomExchange delayedExchange) {
        return BindingBuilder.bind(delayedQueue)
                .to(delayedExchange)
                .with(DELAYED_ROUTING_KEY)
                .noargs();
    }
}
```

##### 生产者

```java
@RequestMapping("/sendDelayedMsg/{message}/delayTime")
public String sendDelayedMsg(@PathVariable String message, @PathVariable Long delayTime) {
    log.info("当前时间：{}，发送一条时长 {}ms 的消息给延迟队列：{}", new Date(), delayTime, message);
    rabbitTemplate.convertAndSend(
            DelayedQueueConfig.DELAYED_EXCHANGE_NAME,
            DelayedQueueConfig.DELAYED_ROUTING_KEY,
            message,
            msg -> {
                // 发送消息，延迟时间（ms）
                msg.getMessageProperties().setDelayLong(delayTime);
                return msg;
            });
    return "DelayedMsg消息发送成功！" + message;
}
```

##### 消费者

```java
/**
 * 消费者：基于插件消息延迟
 */
@Slf4j
@Component
public class DelayQueueConsumer {
    @RabbitListener(queues = DelayedQueueConfig.DELAYED_QUEUE_NAME)
    public void reveiveD(Message message) throws Exception {
        String msg = new String(message.getBody());
        log.info("当前时间：{}，收到延迟队列 {} 的消息：{}", DelayedQueueConfig.DELAYED_QUEUE_NAME, new Date(), msg);
    }
}
```

![插件延迟队列运行结果](https://cdn.nlark.com/yuque/0/2025/png/1452823/1764217063387-0e1bcc1f-9a51-4d4f-a916-66adbf8e5fa9.png)

**不会再收到前方的 hello20 消息，导致后续的消息无法接收到了。**

> **不可以直接被消费者消费，需要转发到消费队列。**

![延迟消息转发](https://cdn.nlark.com/yuque/0/2025/png/1452823/1764221029749-3c4ab072-cd71-49d1-b9a8-54e9c682f403.png)

### 死信队列和延迟队列

![死信队列和延迟队列](https://cdn.nlark.com/yuque/0/2025/png/1452823/1764220271110-b2706d6b-0d5b-4012-af01-5d597e265ad1.png)

| 对比项 | 死信队列（DLQ） | 延迟队列（Delay Queue） |
| --- | --- | --- |
| **作用** | 保存无法被正常消费的“死信”消息 | 让消息在未来某个时间点被投递 |
| **触发方式** | 被动触发：消息变成死信后转入死信队列 | 主动设置：消息设置 TTL 或使用延迟插件 |
| **常见触发条件** | TTL 过期；队列满（被拒绝）；消费失败且拒绝（nack/reject） | 给消息/队列设置 TTL；使用 RabbitMQ 延迟插件（x-delay） |
| **是否一定延迟？** | ❌ 不一定，只要消息死了就进 | ✔️ 一定延迟到固定时间后投递 |
| **使用场景** | 错误消息处理、记录失败消息、重试机制 | 延时任务、订单超时取消、消息定时执行 |
| **是否需要插件** | ❌ 不需要插件（RabbitMQ 原生支持） | ✔️ 使用延迟插件（rabbitmq_delayed_message_exchange）更灵活 |
| **实现方式** | 设置 DLX（x-dead-letter-exchange）、routing-key | TTL + DLX（死信方式实现），或延迟交换机 |
| **消息处理流程** | 普通队列 → 变死信 → 死信交换机 → 死信队列 | 普通队列（或延迟交换机）→ 达到时间后投递到目标队列 |
| **控制级别** | 无法设置具体投递时间 | 可精确控制延迟毫秒级 |
| **能否重投主队列** | 可以（需要业务逻辑实现） | 不需要，到了时间自然投递 |

- **死信队列 = 放失败消息的垃圾桶（错误处理）**
- **延迟队列 = 定时器，让消息晚点处理（延时任务）**

## 发布确认（高级）

> 生产环境，若意外导致 RabbitMQ 重启，重启期间导致生产者消息投递失败，而导致消息丢失了，则需要手动处理和恢复。

集群不可用情况下：

- **如何保证消息投递的可靠性？**
- **消息无法投递该如何处理？**

### 确认机制

![确认机制](https://cdn.nlark.com/yuque/0/2025/png/1452823/1764222427994-857d7d1a-98ba-435a-b6e5-6eacc2ed0392.png)

#### 架构

![发布确认架构](https://cdn.nlark.com/yuque/0/2025/png/1452823/1764222251411-b654beb4-fba6-4530-ba48-be4d372afa0d.png)

##### 发布确认 Config

```java
/**
 * 发布确认（高级）
 */
@Configuration
public class ConfirmConfig {
    // 交换机
    public static final String CONFIRM_QUEUE_NAME = "confirm.queue";
    // 队列
    public static final String CONFIRM_EXCHANGE_NAME = "confirm.exchange";
    // routingkey
    public static final String CONFIRM_ROUTING_KEY = "confirm.routingkey";

    /**
     * 声明队列
     */
    @Bean("confirmQueue")
    public Queue confirmQueue() {
        return new Queue(CONFIRM_QUEUE_NAME);
    }

    /**
     * 声明交换机
     */
    @Bean("confirmExchange")
    public DirectExchange confirmExchange() {
        return new DirectExchange(CONFIRM_EXCHANGE_NAME);
    }

    /**
     * 延迟队列绑定延迟交换机
     */
    @Bean
    public Binding queueBindingExchange(
            @Qualifier("confirmQueue") Queue confirmQueue,
            @Qualifier("confirmExchange") DirectExchange confirmExchange) {
        return BindingBuilder.bind(confirmQueue)
                .to(confirmExchange)
                .with(CONFIRM_ROUTING_KEY);
    }
}
```

##### 生产者

```java
@Slf4j
@RestController
@RequestMapping("/confirm")
public class ProducerController {
    @Autowired
    private RabbitTemplate rabbitTemplate;

    @GetMapping("/sendMessage/{message}")
    public void sendMessage(@PathVariable("message") String message) {
        rabbitTemplate.convertAndSend(
                ConfirmConfig.CONFIRM_EXCHANGE_NAME,
                ConfirmConfig.CONFIRM_ROUTING_KEY,
                message);
        log.info("发送消息内容：{}", message);
    }
}
```

##### 消费者

```java
/**
 * 消费者
 */
@Slf4j
@Component
public class Consumer {
    @RabbitListener(queues = ConfirmConfig.CONFIRM_QUEUE_NAME)
    public void reveiveD(Message message) throws Exception {
        String msg = new String(message.getBody());
        log.info("Consumer：收到 {} 队列QD的消息：{}", ConfirmConfig.CONFIRM_QUEUE_NAME, msg);
    }
}
```

##### 正常情况

[http://localhost:8080/confirm/sendMessage/hello](http://localhost:8080/confirm/sendMessage/hello)

![发布确认正常情况](https://cdn.nlark.com/yuque/0/2025/png/1452823/1764227143894-0179bffd-d1c6-4d2c-96fc-6577be0e1a32.png)

**复现问题场景：交换机 or 队列异常（极端情况）。**

### 回调接口 ConfirmCallback

#### 配置文件

- `none`：默认，禁用发布确认模式。
- `correlated`：发布消息成功到交换机会触发回调方法。
- `simple`：
  - 情况 1：和 `correlated` 一样触发回调。
  - 情况 2：见下图。

![simple 模式情况 2](https://cdn.nlark.com/yuque/0/2025/png/1452823/1764228786015-38882404-d4ef-4e83-9d6a-9febc87fa305.png)

```properties
spring.rabbitmq.publisher-confirm-type=correlated
```

#### 回调实现类

```java
/**
 * 回调实现类
 * 1. 实例化 MyCallback
 * 2. @Autowired 注入
 * 3. @PostConstruct 注入
 */
@Slf4j
@Component
public class MyCallback implements RabbitTemplate.ConfirmCallback {
    @Autowired
    private RabbitTemplate rabbitTemplate;

    @PostConstruct
    public void init() {
        // 注入
        rabbitTemplate.setConfirmCallback(this);
    }

    // ⬇️ 此时还不能调用其 MyCallback 的 confirm 方法，需要注入到 RabbitTemplate ⬆️
    /**
     * @param correlationData 保存回调消息的 ID 及相关信息
     * @param ack 交换机是否收到消息
     * @param cause 失败原因
     * @desc 交换机确认回调方法
     */
    @Override
    public void confirm(CorrelationData correlationData, boolean ack, String cause) {
        String id = correlationData != null ? correlationData.getId() : "";
        if (ack) {
            // 成功
            log.info("MyCallback：交换机收到消息，ID: {}", id);
        } else {
            // 失败
            log.info("MyCallback：交换机未收到消息，ID:{}, CAUSE: {}", id, cause);
        }
    }
}
```

#### 生产者

```java
@GetMapping("/sendMessage/{message}")
public String sendMessage(@PathVariable("message") String message) {
    CorrelationData correlationData = new CorrelationData("233");
    rabbitTemplate.convertAndSend(
            ConfirmConfig.CONFIRM_EXCHANGE_NAME,
            ConfirmConfig.CONFIRM_ROUTING_KEY,
            message,
            correlationData);
    log.info("发送消息内容：{}", message);
    return "发送消息成功：" + message;
}
```

#### 运行结果

**成功**：

![ConfirmCallback 成功](https://cdn.nlark.com/yuque/0/2025/png/1452823/1764230324389-fb2ac113-73f5-4b66-90ea-2b1eec1e37f0.png)

**失败（错误的交换机）**：

![错误的交换机](https://cdn.nlark.com/yuque/0/2025/png/1452823/1764230361274-0a58015f-c612-4808-be95-dffe7a4314a7.png)

**失败（错误的 routingKey）——Consumer 未收到消息**：

> 交换机收到消息，但是队列未收到消息。

![错误的 routingKey](https://cdn.nlark.com/yuque/0/2025/png/1452823/1764230409562-7269a8cb-87ed-41fc-9012-d998e870e243.png)

### 回退消息

#### Mandatory 参数

> 在仅开启生产者确认机制情况下，交换机接收消息后，会直接给消息生产者发送确认消息，**如果发现该消息不可路由**，那么**消息会被直接丢弃**。此时，生产者是不知道消息被丢弃了的。
>
> 通过设置 `mandatory` 参数可以在当消息传递过程中不可达目的地时将消息返回给生产者。

##### 配置

```properties
# 退回消息：回退消息给生产者
spring.rabbitmq.publisher-returns=true
```

##### MyCallback

```java
...
public class MyCallback implements RabbitTemplate.ConfirmCallback, RabbitTemplate.ReturnsCallback {
    ...

    @PostConstruct
    public void init() {
        // 注入
        rabbitTemplate.setConfirmCallback(this);
        rabbitTemplate.setReturnsCallback(this);
    }

    /**
     * @param returnedMessage 退回的消息
     * @desc 队列确认回调方法
     */
    @Override
    public void returnedMessage(ReturnedMessage returnedMessage) {
        log.error("MyCallback：消息 {} 被交换机 {} 退回，退回原因：{}，路由Key：{}",
                new String(returnedMessage.getMessage().getBody()),
                returnedMessage.getExchange(),
                returnedMessage.getReplyText(),
                returnedMessage.getRoutingKey());
    }
}
```

##### 错误的 routingKey

![回退消息结果](https://cdn.nlark.com/yuque/0/2025/png/1452823/1764254882379-152199bd-68f5-4b3e-8b46-cd88748c580e.png)

### 备份交换机

当消息到达交换机但没有任何队列匹配它时，**将这些“无法路由”的消息转发给备份交换机，避免消息丢失**。

![备份交换机](https://cdn.nlark.com/yuque/0/2025/png/1452823/1764255073191-1725e6cc-1900-4018-957e-13048e0f9b06.png)

- [x] 确认交换机（上章节已完成）
- [ ] 备份交换机

#### ConfirmConfig

`fanout` 类型交换机没有 routingKey。

```java
...
@Configuration
public class ConfirmConfig {
    // 交换机
    public static final String CONFIRM_QUEUE_NAME = "confirm_queue";
    // 队列
    public static final String CONFIRM_EXCHANGE_NAME = "confirm_exchange";
    // routingkey
    public static final String CONFIRM_ROUTING_KEY = "key1";
    // 备份交换机
    public static final String BACKUP_EXCHANGE_NAME = "backup_exchange";
    // 备份队列
    public static final String BACKUP_QUEUE_NAME = "backup_queue";
    // 报警队列
    public static final String WARNING_QUEUE_NAME = "warning_queue";

    /**
     * 确认队列
     */
    @Bean("confirmQueue")
    public Queue confirmQueue() {
        return new Queue(CONFIRM_QUEUE_NAME);
    }

    /**
     * 确认交换机：指定报警交换机
     */
    @Bean("confirmExchange")
    public DirectExchange confirmExchange() {
        return ExchangeBuilder.directExchange(CONFIRM_EXCHANGE_NAME)
                .durable(true)
                .withArgument("alternate-exchange", BACKUP_EXCHANGE_NAME)
                .build();
    }

    /**
     * 延迟队列绑定延迟交换机
     */
    @Bean
    public Binding queueBindingExchange(
            @Qualifier("confirmQueue") Queue confirmQueue,
            @Qualifier("confirmExchange") DirectExchange confirmExchange) {
        return BindingBuilder.bind(confirmQueue)
                .to(confirmExchange)
                .with(CONFIRM_ROUTING_KEY);
    }

    /**
     * 备份交换机
     */
    @Bean("backupExchange")
    public FanoutExchange backupExchange() {
        return new FanoutExchange(BACKUP_EXCHANGE_NAME);
    }

    /**
     * 备份队列
     */
    @Bean("backupQueue")
    public Queue backupQueue() {
        return QueueBuilder.durable(BACKUP_QUEUE_NAME).build();
    }

    /**
     * 报警队列
     */
    @Bean("warningQueue")
    public Queue warningQueue() {
        return QueueBuilder.durable(WARNING_QUEUE_NAME).build();
    }

    /**
     * 绑定备份队列
     */
    @Bean
    public Binding backupQueueBindingExchange(
            @Qualifier("backupQueue") Queue backupQueue,
            @Qualifier("backupExchange") FanoutExchange backupExchange) {
        return BindingBuilder.bind(backupQueue)
                .to(backupExchange);
    }

    /**
     * 绑定报警队列
     */
    @Bean
    public Binding warningQueueBindingBackupExchange(
            @Qualifier("warningQueue") Queue warningQueue,
            @Qualifier("backupExchange") FanoutExchange backupExchange) {
        return BindingBuilder.bind(warningQueue)
                .to(backupExchange);
    }
}
```

#### 报警队列消费者 WarningConsumer

```java
/**
 * 报警队列消费者
 */
@Slf4j
@Component
public class WarningConsumer {
    @RabbitListener(queues = ConfirmConfig.WARNING_QUEUE_NAME)
    public void reveiveD(Message message) throws Exception {
        String msg = new String(message.getBody());
        log.info("WarningConsumer：报警发现不可路由消息：{}", msg);
    }
}
```

**需要重新创建 ConfirmExchange。**

#### 结果

> ID：1，正常的 routingKey，消息正常，ConfirmQueue 收到消息并消费。
>
> ID：3，错误的 routingKey，转发给备份交换机，发送给备份队列和报警队列，报警队列用于报警给用户提示。

![备份交换机运行结果](https://cdn.nlark.com/yuque/0/2025/png/1452823/1764405256446-64872363-efcf-4b78-89b5-d798b96d48cc.png)

> 此时并未走进 MyCallback 消息回退，因为消息是可到达的 → **备份交换机**。

**备份队列保留了发送失败的消息。**

![备份队列保留消息](https://cdn.nlark.com/yuque/0/2025/png/1452823/1764405779142-4b399ce1-c17e-48c8-b477-a551db495d09.png)

## 幂等性问题

> 重复提交，**重复消费导致业务重复执行。**

![幂等性问题](https://cdn.nlark.com/yuque/0/2025/png/1452823/1764406021502-99199c14-f989-4442-a6be-28fee330c7e7.png)

### 可能导致的问题

- 下单重复
- 扣款多次
- 重复发货
- 数据重复写入
- 重复积分/奖励
- 状态机回退

### 解决方案

**🌟 Redis 原子性**

1. **业务唯一 ID**
2. 去重表
3. Redis 分布式锁
4. Redis 记录处理状态
5. 数据库乐观锁 / CAS
6. 保证接口本身幂等性
7. 消息队列端做幂等性
8. 数据库唯一键 + 插入判断

## 优先级队列

> 按照普通队列，是按照队列的先进先出顺序出队消费。
>
> 优先级队列，按照其消息权重（对内按照权重排序）进行出队消费。

权重：`0`（默认）～`255`（建议 `10`），越大优先级越高。

### 创建优先级队列

![创建优先级队列](https://cdn.nlark.com/yuque/0/2025/png/1452823/1764407511513-81504ce7-fe23-467a-ab76-f33fb5a1170a.png)

### 生产者

```java
/**
 * 生产者：优先级队列
 */
public class Producer {
    private static final String QUEUE_NAME = "priority_queue";

    public static void main(String[] args) throws Exception {
        Channel channel = RabbitUtils.getChannel();
        Map<String, Object> arguments = new HashMap<>();
        arguments.put("x-max-priority", 10);
        channel.queueDeclare(QUEUE_NAME, false, false, false, arguments);

        // i = 5，设置优先级
        for (int i = 0; i < 10; i++) {
            String message = "消息" + i;
            if (i == 5) {
                AMQP.BasicProperties properties = new AMQP.BasicProperties().builder().priority(5).build();
                channel.basicPublish("", QUEUE_NAME, properties, message.getBytes());
            } else {
                channel.basicPublish("", QUEUE_NAME, null, message.getBytes());
            }
        }
    }
}
```

![优先级消息发送结果](https://cdn.nlark.com/yuque/0/2025/png/1452823/1764408790696-70365b35-8599-4549-91ae-5ca45a53bc3c.png)

### 消费者

```java
/**
 * 消费者
 */
public class Consumer {
    public static final String QUEUE_NAME = "priority_queue";

    public static void main(String[] args) throws Exception {
        Channel channel = RabbitUtils.getChannel();
        System.out.println("PriorityQueue等待接收消息...");

        // 接收消息
        DeliverCallback deliverCallback = (consumerTag, message) -> {
            System.out.println("PriorityQueue接收到消息：" + new String(message.getBody(), StandardCharsets.UTF_8));
        };
        channel.basicConsume(QUEUE_NAME, deliverCallback, consumerTag -> {
        });
    }
}
```

![优先级消息消费结果](https://cdn.nlark.com/yuque/0/2025/png/1452823/1764470792856-31e74421-ef84-4b56-9662-66e5e8838f6f.png?x-oss-process=image%2Fcrop%2Cx_0%2Cy_0%2Cw_1534%2Ch_432)

## 惰性队列

> 正常队列：消息保存在内存中。
>
> **惰性队列：消息保存在磁盘中。**

### 场景

- 消费者下线
- 宕机（繁忙）
- 因维护而关闭

导致大量消息堆积，**防止 RabbitMQ 内存爆炸。**

### 普通队列和惰性队列

| 特性 | 普通队列（classic） | 惰性队列（lazy） |
| --- | --- | --- |
| **消息存储位置** | 优先内存 | 直接磁盘 |
| **堆积能力** | 差（内存压力大） | 强（可堆积百万级） |
| **速度** | 更快 | 稍慢 |
| **消费时** | 从内存取 | 从磁盘读取 |
| **使用场景** | 高实时性消息 | 大量堆积/削峰/不紧急 |

### 惰性队列的两种模式

- `default`（默认）
- `lazy`（惰性队列）

```java
arguments.put("x-queue-mode", "lazy");
```

### 内存开销对比

| 堆积消息量 | 普通队列 Classic Queue | 惰性队列 Lazy Queue |
| --- | --- | --- |
| **10 万条** | ~ 1.5G RAM | ~ 150MB（主要是系统） |
| **50 万条** | ~ 5–8G RAM | ~ 300MB |
| **100 万条** | **直接 OOM 或触发内存报警** | ~ 500MB（一直保持很稳） |
| **500 万条** | 不可能（早挂了） | ~ 600–800MB |

> **最终都是转到内存中再消费。**

## 集群

目标：搭建 3 个 RabbitMQ，组成集群。

![RabbitMQ 集群](https://cdn.nlark.com/yuque/0/2025/png/1452823/1764480863139-805e4568-1c69-456c-94d4-9b3c82e1ee0c.png)

### 1. 集群节点 cookie 一致

```shell
cat /var/lib/rabbitmq/.erlang.cookie
```

### 2. 以节点 1 的 cookie 为准，复制到其他两个节点

```shell
scp /var/lib/rabbitmq/.erlang.cookie root@192.168.1.12:/var/lib/rabbitmq/
scp /var/lib/rabbitmq/.erlang.cookie root@192.168.1.13:/var/lib/rabbitmq/
```

### 3. 权限

```shell
chmod 400 /var/lib/rabbitmq/.erlang.cookie
```

### 4. 设置节点名

配置文件：`/etc/rabbitmq/rabbitmq-env.conf`

```properties
NODENAME=rabbit@rabbit1 # 根据各自节点修改
```

### 5. 加入集群

```shell
# 停止应用
rabbitmqctl stop_app

# 加入节点 1
rabbitmqctl join_cluster rabbit@rabbit1

# 启动
rabbitmqctl start_app
```

### 6. 验证状态

```shell
rabbitmqctl cluster_status
```

## 镜像队列

> 旧方案，高可用。
>
> 为了**某个节点挂了不丢消息、不停机**，实现队列备份。
>
> 新方案：仲裁队列。

## 负载均衡

## Federation Exchange

### 1. 启用插件

```shell
rabbitmq-plugins enable rabbitmq_federation
rabbitmq-plugins enable rabbitmq_federation_management
```

### 2. 配置上游

```shell
rabbitmqctl set_parameter federation-upstream myUpstream \
  '{ "uri": "amqp://admin:pass@rabbit1:5672", "expires": 3600000 }'
```

### 3. 定义联邦策略（policy）

```shell
# 含义：所有名称以 fed-ex 开头的 exchange 都会联邦同步。
rabbitmqctl set_policy federateX "^fed-ex.*" \
  '{"federation-upstream-set":"all"}'
```

### 4. 创建 Federation Exchange

```text
# 创建一个与上游同名的 exchange
# RabbitMQ 会自动绑定 Federation Link。
fed-ex.topic
```

## Federation Queue
