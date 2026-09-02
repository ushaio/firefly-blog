# 一、Java

## 如何处理异常

1. try-catch/多catch分支捕获（先捕获子类，再捕获父类）

2. throws（声明抛出）

   `public static void readFile(String path) throws IOException, FileNotFoundException {`

3. throw（手动抛出异常）

   `throw new ArithmeticException("除数不能为0");`

4. **try-with-resources（自动关闭资源）⭐**

   传统写法：`try{}catch(){}finally{}`

   **try-with-resources：**

   ```java
   // 单个资源
   try (BufferedReader reader = new BufferedReader(new FileReader("test.txt"))) {
       String line = reader.readLine();
       System.out.println(line);
   } catch (IOException e) {
       e.printStackTrace();
   }
   
   // 多个资源
   try (
       FileInputStream fis = new FileInputStream("input.txt");
       FileOutputStream fos = new FileOutputStream("output.txt")
   ) {
       fos.write(fis.readAllBytes());
   } catch (IOException e) {
       e.printStackTrace();
   }
   ```

| 方式                 | 适用场景                   | 特点                       |
| -------------------- | -------------------------- | -------------------------- |
| `try-catch`          | 需要在当前方法处理异常     | 最基础，就地处理           |
| `throws`             | 当前方法不处理，交给调用者 | 声明式，受检异常必须声明   |
| `throw`              | 主动抛出异常（如参数校验） | 配合自定义异常使用         |
| `try-with-resources` | IO流/数据库连接等资源操作  | 自动关闭，代码简洁         |
| 自定义异常           | 业务逻辑异常               | 语义清晰，便于分类处理     |
| 全局异常处理         | Web 应用统一兜底           | 统一返回格式，避免重复代码 |

## 反射

- 反射能获取 private 字段吗

  > 反射可以通过`getDeclaredField()`获取包括private的全部字段，但需要添加`field.setAccessible(true)`

- 

## 举例常用异常类型

**RuntimeException**

- **NullPointerException（空指针异常）**
- **ArrayIndexOutOfBoundsException（数组越界异常）**
- **ArithmeticException（算术异常）**
- **ClassCastException（类型转换异常）**
- **NumberFormatException（数字格式异常）**
- **IllegalArgumentException（非法参数异常）**
- **OutOfMemoryError（内存溢出错误）**

## Spring循环依赖及解决

注解方式解决

## @Resource和@Autowired

## 事务

三级缓存

事务失效的场景，如何处理

## 并发

- 并发和并行

  - 并发，同一时间段多个线程交替执行；单核CPU支持
  - 并行，同一时刻多个线程执行；单核CPU不支持

- **多线程一定是并行吗**

  不一定。多线程只是提供了多个执行流，但是否并行取决于 CPU 核心数、线程调度、任务类型等因素。比如在单核 CPU 上，多个线程只能并发交替执行；在多核 CPU 上，多个线程才可能真正并行执行。

# 二、数据库

## join和left join

| 连接类型          | 保留哪些数据                | NULL 来源         |
| ----------------- | --------------------------- | ----------------- |
| `INNER JOIN`      | 两表**都匹配**的行          | 无                |
| `LEFT JOIN`       | **左表全部** + 右表匹配的行 | 右表不匹配 → NULL |
| `RIGHT JOIN`      | **右表全部** + 左表匹配的行 | 左表不匹配 → NULL |
| `FULL OUTER JOIN` | **两表全部**的行            | 两边不匹配 → NULL |

## SQL优化

1. 减少不必要的select 字段，避免使用select *
2. 避免大量表join连接，较多的情况下适当根据业务进行拆分
3. 合理创建索引

## 索引

- 单张表索引不超过**5个**

单列索引

联合索引

```sql
-- 联合索引：多个字段组成一个索引
CREATE INDEX idx_city_age_name ON users(city, age, name);
-- 对应的 B+Tree 结构（先按 city 排序，city 相同按 age 排序，age 相同按 name 排序）
```

**最左前缀原则（核心）⭐⭐⭐**

*左侧最优先，必须自左向右有匹配到*

```sql
Text
联合索引 (city, age, name) 的匹配规则：

查询条件                          是否走索引    使用了哪些列
─────────────────────────────────────────────────────────────
WHERE city = '北京'               ✅           city
WHERE city = '北京' AND age=25    ✅           city, age
WHERE city = '北京' AND age=25    ✅           city, age, name（全部）
  AND name = '张三'

WHERE age = 25                   ❌            跳过了city，索引失效
WHERE name = '张三'              ❌            跳过了city和age，索引失效
WHERE age = 25 AND name = '张三'  ❌            跳过了city，索引失效
```

### 索引失效场景

1. 函数/运算导致索引失效
   1. 对索引列使用函数
   2. 对索引列做运算
   3. 隐式类型转换
2. LIKE 以通配符开头，如：`like ‘%san’`
3. OR 条件中有非索引列，如：`WHERE name = '张三' OR address = '北京'; -- address没有索引，可以拆分改成union`
4. NOT / != / <> / NOT IN

```sql
-- ❌ 大部分情况不走索引
SELECT * FROM test_index WHERE age != 25;
SELECT * FROM test_index WHERE age NOT IN (20, 25, 30);
-- 但也有例外：如果 MySQL 优化器评估走索引更优时仍可能使用
```

1. IS NULL / IS NOT NULL

```sql
-- 视数据分布而定
-- 如果 NULL 值占比很小，可能走索引
-- 如果大部分都是 NULL，优化器会选择全表扫描
SELECT*FROM test_indexWHEREnameISNULL;
SELECT*FROM test_indexWHEREnameIS NOT NULL;
```

### 索引优缺点

优点：

- 提升查询速度
- 保证数据唯一性
- 加速表join连接
- 加速排序和分组
- 联合索引包含全部查询字段时，可以直接从索引fanhui
- 锁优化

缺点：

- 占用额外的空间
- 降低写操作性能，每次insert/update/delete，索引都要维护
- 索引过多增加优化器负担

## union和union all

| **对比维度** | **UNION**                                    | **UNION ALL**                          |
| ------------ | -------------------------------------------- | -------------------------------------- |
| **结果去重** | **会**去除重复行（类似 `SELECT DISTINCT`）   | **不会**去除重复行，保留所有记录       |
| **执行性能** | **慢**（需要额外进行排序和去重运算）         | **快**（直接追加数据，无需额外计算）   |
| **排序行为** | 默认会对最终结果集进行**排序**（通常按首列） | 不排序，按照查询顺序或底层存储顺序输出 |
| **内存消耗** | 高（需要维护去重哈希表或排序缓冲区）         | 低（仅仅是数据流的拼接）               |

## 数据量大优化方案

**1. 查询优化**

- 禁止`SELECT *`，只查询需要的字段
- **避免在WHERE中对列进行函数转换**（如`WHERE date(create_time)='2024-01-01'`），改用范围查询
- **用`IN`代替`OR`**（IN值不超过500个）
- 避免`ORDER BY RAND()`，建议在程序中生成随机值

**2. 写入优化**

- 批量写操作（超100万行）要**分批多次**进行，避免大事务锁表
- 使用`pt-online-schema-change`工具修改大表结构，避免锁表
- 禁止在线上做数据库压力测试

**3. 子查询与JOIN**

- 避免复杂子查询，优化为JOIN操作
- JOIN关联表不超过**5个**，避免内存溢出

1. 分批次写入，意味着可以避免大事务锁表，但需要对失败数据特殊处理，保证后续失败的数据重试写入，保证全部成功
2. 事务包裹，意味着事务一致性，要么成功要么失败，但可能会出现锁表

- **结论分析：**

  ```jsx
  两种策略的本质是：一致性 vs 性能 的权衡
  
  策略一：分批次写入（每批独立事务）
  ├── 本质：牺牲原子性，换取短事务、短锁持有
  ├── 优点：锁持有时间短，不会阻塞其他操作
  ├── 代价：失败后需要重试 + 幂等保证，才能最终全部成功
  └── 适用：大数据量、允许最终一致性的场景
  
  策略二：事务包裹（一个大事务）
  ├── 本质：牺牲性能，换取强一致性
  ├── 优点：要么全部成功，要么全部回滚，数据绝对一致
  ├── 代价：长事务长时间持有行锁，高并发下可能导致大量阻塞
  └── 适用：小数据量、要求强一致性的场景
  ```

  "这两种策略本质是**一致性和性能的权衡**：

  分批次写入用短事务换取性能，代价是失去原子性，需要配合**失败重试 + 幂等设计**来保证最终全部成功；

  事务包裹用长事务换取强一致性，代价是**长时间持有行锁**，数据量大时会导致大量阻塞。

  实际生产中，我会根据数据量和一致性要求来选择——小数据量强一致用事务包裹，大数据量用分批写入加重试幂等。"

- **什么是行锁，这里面什么情况是行锁，什么情况是锁表**

  | 维度     | 行锁                       | 表锁                         |
  | -------- | -------------------------- | ---------------------------- |
  | 锁定范围 | 只锁定被操作的行           | 锁定整张表                   |
  | 并发度   | 高（不同行互不影响）       | 低（同一时刻只能一个事务写） |
  | 加锁开销 | 大（需要维护锁的数据结构） | 小（一次性锁整表）           |
  | 死锁概率 | 较高（多行加锁顺序不同）   | 较低                         |
  | 支持引擎 | InnoDB                     | MyISAM、InnoDB（特定场景）   |

  锁表：

  - 条件列没走索引，`where age = 25`

    > 需要每个条件都走到索引吗？

    不需要。只要有一个索引命中即可。

  - between导致的间隙锁，`WHERE age BETWEEN 20 AND 30;` 导致该区间的记录被锁

  - 

- **什么是幂等和非幂等？**

  SQL

  ```jsx
  -- ❌ 非幂等：重试会插入重复数据
  INSERT INTO t_user (id, username) VALUES (1, '张三');
  
  -- ✅ 幂等：重试不会产生重复
  INSERT IGNORE INTO t_user (id, username) VALUES (1, '张三');
  -- 或
  INSERT INTO t_user (id, username) VALUES (1, '张三')
  ON DUPLICATE KEY UPDATE username = VALUES(username);
  ```

  > 但很多情况下id是根据函数自动生成的

  ——这会导致ignore失效，那如何保证幂等性，不重复插入？

  > 

  | 方案                   | 核心思路                  | 优点                             | 缺点                               |
  | ---------------------- | ------------------------- | -------------------------------- | ---------------------------------- |
  | 应用层预生成 ID        | ID 生成动作提到重试循环外 | 简单直接                         | 需要改造调用代码结构               |
  | 序列单独查询后固定使用 | 把"取值"和"使用"分离      | 兼容传统数据库序列               | 取值这一步本身也要防止被重复调用   |
  | **业务唯一键兜底**     | 不依赖 id，依赖业务字段   | **最稳妥，不受 id 生成方式影响** | 需要额外设计并维护一个唯一索引字段 |

  HTTP

- **那如果业务要求的是需要一次性全部成功呢，而非部分成功（后续补充完成失败的记录）**

  <aside>
   💡

  1. **数据预校验，再事务包裹一次性写入**
  2. **分批次写入临时表，再事务包裹一次性写入**
      </aside>

  ## **核心思路：把"写入"和"提交"分离**

  既然大事务不可行，分批写入又破坏原子性，那就把整个过程拆成两个阶段：

  ```
  阶段一：数据准备（可以失败、可以重试、不影响生产表）
  阶段二：原子提交（一次性、快速、要么全成功要么全回滚）
  ```

  ------

  ## **方案一：临时表 + 原子切换（最推荐）**

  ### **原理**

  先把所有数据写入一张**临时表/暂存表**，这个阶段可以分批写入、可以失败重试，因为不影响生产数据。等全部数据准备完毕后，再用一个事务做**原子性的 INSERT...SELECT**，从临时表导入正式表。

  ```sql
  -- 第一步：创建临时表（结构和正式表一致）
  CREATE TABLE t_user_temp LIKE t_user;
  
  -- 第二步：分批写入临时表（可以失败重试，不影响生产）
  -- Java 代码分批 insert into t_user_temp ...
  -- 这一步失败了直接 truncate 重来，毫无压力
  
  -- 第三步：原子提交（一个事务，从临时表导入正式表）
  START TRANSACTION;
  INSERT INTO t_user SELECT * FROM t_user_temp;
  COMMIT;
  
  -- 第四步：清理临时表
  TRUNCATE TABLE t_user_temp;
  ```

  ### **为什么可行？**

  | 阶段     | 特点                           | 风险                             |
  | -------- | ------------------------------ | -------------------------------- |
  | 写临时表 | 分批写入、可重试、不影响生产   | 无风险，失败直接重来             |
  | 原子提交 | 单条 INSERT...SELECT，一个事务 | 事务时间短，因为只是表间数据搬运 |

  **关键点**：第三步的 `INSERT...SELECT` 虽然也是一个事务，但它只是**表到表的数据搬运**，不涉及网络传输、业务计算，速度非常快，即使百万级数据也能在几秒内完成，锁持有时间可控。

  ### **Java 代码示例**

  ```java
  @Service
  public class BatchImportService {
  
      @Autowired
      private UserMapper userMapper;
  
      public void importAllOrNothing(List<User> userList) {
          // 阶段一：分批写入临时表（可重试）
          userMapper.createTempTable();
          int batchSize = 1000;
          for (int i = 0; i < userList.size(); i += batchSize) {
              List<User> batch = userList.subList(i, Math.min(i + batchSize, userList.size()));
              userMapper.batchInsertTemp(batch);  // 写入临时表
          }
  
          // 校验：临时表数据量是否等于源数据量
          int tempCount = userMapper.countTempTable();
          if (tempCount != userList.size()) {
              userMapper.truncateTempTable();
              throw new RuntimeException("数据准备不完整，已回滚");
          }
  
          // 阶段二：原子提交（一个事务）
          userMapper.atomicCopyFromTemp();  // INSERT INTO t_user SELECT * FROM t_user_temp
  
          // 清理
          userMapper.truncateTempTable();
      }
  }
  ```

  ```xml
  <!-- 原子提交：一个事务完成 -->
  <insert id="atomicCopyFromTemp">
      INSERT INTO t_user (username, age, email)
      SELECT username, age, email FROM t_user_temp
  </insert>
  ```

  ------

  ## **方案二：预校验 + 事务包裹**

  ### **原理**

  大事务之所以危险，很大程度是因为**中途失败导致长时间持有锁后还要回滚**。如果能在进入事务前**把所有可能失败的因素排除掉**，事务本身就会非常快。

  ```java
  @Service
  public class BatchImportService {
  
      @Transactional(rollbackFor = Exception.class)
      public void importWithPreCheck(List<User> userList) {
          // 阶段一：预校验（事务外完成，这里只是举例）
          // - 检查主键是否冲突
          // - 检查外键引用是否存在
          // - 检查字段格式是否合法
          preValidate(userList);
  
          // 阶段二：事务内快速写入（因为已排除失败因素，几乎不会失败）
          int batchSize = 1000;
          for (int i = 0; i < userList.size(); i += batchSize) {
              List<User> batch = userList.subList(i, Math.min(i + batchSize, userList.size()));
              userMapper.batchInsertUsers(batch);
          }
      }
  
      private void preValidate(List<User> userList) {
          // 批量查询已存在的主键，检查冲突
          Set<Long> existingIds = userMapper.findExistingIds(
              userList.stream().map(User::getId).collect(Collectors.toList())
          );
          for (User user : userList) {
              if (existingIds.contains(user.getId())) {
                  throw new BusinessException("主键冲突: " + user.getId());
              }
              // 其他校验...
          }
      }
  }
  ```

  **适用场景**：数据量中等（几万到十几万条），失败原因主要是数据本身的问题（主键冲突、格式错误等）。

  ------

  ## **方案三：Seata 分布式事务（微服务场景）**

  你的笔记里提到了 Seata，如果涉及**跨库、跨服务**的原子性，就需要分布式事务框架。

  ```mermaid
  flowchart LR
      A[订单服务] -->|Try: 预留资源| B[库存服务]
      A -->|Try: 预留资源| C[账户服务]
      B -->|Confirm: 确认扣减| D[完成]
      C -->|Confirm: 确认扣减| D
      B -.->|Cancel: 回滚| E[失败回滚]
      C -.->|Cancel: 回滚| E
  ```

  Seata 有几种模式：

  | 模式          | 原理                             | 适用场景                 |
  | ------------- | -------------------------------- | ------------------------ |
  | **AT 模式**   | 自动生成回滚SQL，对业务无侵入    | 单体应用跨库、大多数场景 |
  | **TCC 模式**  | 手写 Try/Confirm/Cancel 三个方法 | 高性能、跨服务           |
  | **Saga 模式** | 长事务，通过补偿回滚             | 流程长的业务             |
  | **XA 模式**   | 数据库原生XA协议                 | 强一致性要求极高         |

  ------

  ## **方案四：Saga 补偿事务（长流程场景）**

  如果流程很长（比如订单 → 扣库存 → 扣余额 → 生成物流），不适合用一个事务包裹，可以用 Saga 模式：

  ```
  正向流程：  创建订单 → 扣库存 → 扣余额 → 生成物流
                      ↓ 某步失败
  补偿流程：  取消订单 ← 恢复库存 ← 恢复余额
  ```

  每一步都是独立事务，失败时按反序执行补偿操作。**不是数据库层面的回滚，而是业务层面的补偿**。

  ------

  ## **总结对比**

  | 方案                  | 原子性     | 性能   | 复杂度 | 适用场景                   |
  | --------------------- | ---------- | ------ | ------ | -------------------------- |
  | **临时表 + 原子切换** | ✅ 强       | ✅ 好   | 中     | 单表大数据量导入（最推荐） |
  | **预校验 + 事务包裹** | ✅ 强       | ⚠️ 中等 | 低     | 中等数据量、失败原因可预判 |
  | **Seata AT/TCC**      | ✅ 强       | ⚠️ 中等 | 高     | 跨库跨服务                 |
  | **Saga 补偿**         | ✅ 最终一致 | ✅ 好   | 高     | 长流程业务                 |

  ------

- 关于其重试机制

  ```jsx
  重试机制设计
  ├── 1. 什么时候重试？（触发条件）
  ├── 2. 重试几次？（次数限制）
  ├── 3. 每次间隔多久？（退避策略）
  ├── 4. 重试时如何避免重复数据？（幂等保证）
  └── 5. 全部失败后怎么办？（兜底处理）
  ```

# SC

如何集中管理application的配置参数

```
ConfigurationProperties
```

jdk17+，使用record

推荐路线：

1. 先完成分层：[`Controller`](http://localhost:12345/spring-cloud-learn/order-service/src/main/java/org/example/order/OrderController.java:12) → [`Service`](http://localhost:12345/spring-cloud-learn/order-service/src/main/java/org/example/order/OrderController.java:34) → Client。
2. 再新增 [`goods-service`](http://localhost:12345/spring-cloud-learn/pom.xml)，让订单服务同时调用用户服务和商品服务。
3. 再引入注册中心，例如 Nacos 或 Eureka，不再写 [`http://localhost:8081`](http://localhost:12345/spring-cloud-learn/order-service/src/main/resources/application.yml:10)。
4. 再学习 OpenFeign，把 [`RestClient`](http://localhost:12345/spring-cloud-learn/order-service/src/main/java/org/example/order/OrderController.java:16) 调用改成接口式调用。
5. 最后学习 Gateway、配置中心、熔断限流、链路追踪。

- 父工程先管理 Spring Cloud Alibaba 版本：在 [`pom.xml`](http://localhost:12345/spring-cloud-learn/pom.xml:31) 的 [`dependencyManagement`](http://localhost:12345/spring-cloud-learn/pom.xml:31) 中额外导入 [`spring-cloud-alibaba-dependencies`](http://localhost:12345/spring-cloud-learn/pom.xml:31)，建议版本用兼容 Spring Boot 3.x / Spring Cloud 2024.x 的新版本，例如 `2023.0.3.3`。
- 三个服务都加注册发现依赖：在 [`user-service/pom.xml`](http://localhost:12345/spring-cloud-learn/user-service/pom.xml:16)、[`order-service/pom.xml`](http://localhost:12345/spring-cloud-learn/order-service/pom.xml:16)、[`goods-service/pom.xml`](http://localhost:12345/spring-cloud-learn/goods-service/pom.xml:16) 加 [`spring-cloud-starter-alibaba-nacos-discovery`](http://localhost:12345/spring-cloud-learn/pom.xml:31)。
- 三个服务都配置 Nacos 地址：在 [`application.yml`](http://localhost:12345/spring-cloud-learn/user-service/src/main/resources/application.yml:5)、[`application.yml`](http://localhost:12345/spring-cloud-learn/order-service/src/main/resources/application.yml:5)、[`application.yml`](http://localhost:12345/spring-cloud-learn/goods-service/src/main/resources/application.yml:5) 的 [`spring.cloud.nacos.discovery.server-addr`](http://localhost:12345/spring-cloud-learn/order-service/src/main/resources/application.yml:5) 配成 `127.0.0.1:8848`。
- 订单服务需要支持按服务名负载均衡调用：在 [`order-service/pom.xml`](http://localhost:12345/spring-cloud-learn/order-service/pom.xml:16) 加 [`spring-cloud-starter-loadbalancer`](http://localhost:12345/spring-cloud-learn/order-service/pom.xml:16)，然后把 [`RestClient.Builder`](http://localhost:12345/spring-cloud-learn/order-service/src/main/java/org/example/order/client/UserClient.java:14) 配成带 [`@LoadBalanced`](http://localhost:12345/spring-cloud-learn/order-service/src/main/java/org/example/order/client/UserClient.java:14) 的 Bean。
- 最后把固定地址改成服务名：[`UserClient`](http://localhost:12345/spring-cloud-learn/order-service/src/main/java/org/example/order/client/UserClient.java:19) 调用 `http://user-service/users/{id}`，[`GoodsClient`](http://localhost:12345/spring-cloud-learn/order-service/src/main/java/org/example/order/client/GoodsClient.java:19) 调用 `http://goods-service/goods/{id}`，然后删除 [`service.user-url`](http://localhost:12345/spring-cloud-learn/order-service/src/main/resources/application.yml:10) 和 [`service.goods-url`](http://localhost:12345/spring-cloud-learn/order-service/src/main/resources/application.yml:11)。

启动顺序：先启动本地 Nacos，再启动 [`user-service`](http://localhost:12345/spring-cloud-learn/user-service/src/main/resources/application.yml:7)、[`goods-service`](http://localhost:12345/spring-cloud-learn/goods-service/src/main/resources/application.yml:7)、[`order-service`](http://localhost:12345/spring-cloud-learn/order-service/src/main/resources/application.yml:7)。Nacos 控制台地址是 `http://127.0.0.1:8848/nacos`，能看到三个服务注册成功后，再访问订单详情接口。

```
RestClient
```

Nacos

自动重试retry

```
@LoadBalancer
```

使用去OpenFeign前：

application.yml声明各服务名，在业务代码使用RestClient使用对应服务名调用对应微服务接口

## `OpenFeign`

声明式 HTTP 客户端，用接口方式调用远程服务

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

- EnableFeignClients
- FeignClient

### Gateway

第 1 步：Gateway 基础路由

1. Gateway 添加请求头

   ```yaml
   gateway:
         routes:
           - id: order-service
             uri: lb://order-service
             predicates:
               - Path=/api/orders/**
               - AddRequestHeader=X-Source, gateway-service # 添加请求头，用于记录请求来源（X-Source: gateway-service）
             filters:
               - StripPrefix=1
   ```

2. Gateway 全局过滤器 GlobalFilter

3. Gateway 统一 traceId

4. Gateway 简单鉴权

5. Gateway 跨域 CORS

第 2 步：Gateway + Nacos 动态服务发现
 第 3 步：Gateway Filter：StripPrefix、AddRequestHeader
 第 4 步：GlobalFilter：统一日志、traceId
 第 5 步：Gateway 鉴权
 第 6 步：Nacos Config 配置中心
 第 7 步：Sentinel 限流熔断
 第 8 步：Seata 分布式事务

# JWT

**防篡改**

JAVA17

```
record
```

# 实战

## 生产上对数据库数据问题如何排查

```java
生产数据问题
├── 1. 数据丢失（记录莫名消失）
├── 2. 数据错误（字段值不对）
├── 3. 数据重复（脏数据）
├── 4. 数据不一致（多表/多系统间数据对不上）
├── 5. 性能问题（慢查询、死锁）
└── 6. 数据库连接问题（连接池耗尽、超时）
```

## 如何处理Git冲突

ry-with-resource