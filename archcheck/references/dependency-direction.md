# 维度 1:依赖方向 (Dependency Direction)

> Uncle Bob 整本《架构整洁之道》的核心规则:**源代码依赖只能朝内**。

这是五个维度里最硬的一条。其他维度可以妥协,这一条妥协了整个架构就垮。

## 核心规则

### 规则 1.1:业务核心不依赖框架

**业务核心包不能 import 任何框架/外部库**(除了 `java.*`、`slf4j` 这种纯接口的)。

具体扫描清单(Java/Spring):

| Import 模式 | 严重度 | 说明 |
|------------|--------|------|
| `org.springframework.*`(在 domain/usecase 里) | 🔴 严重 | 业务核心被 Spring 污染 |
| `javax.persistence.*` / `jakarta.persistence.*` | 🔴 严重 | 业务核心被 JPA 污染 |
| `com.fasterxml.jackson.*` | 🟡 中等 | 序列化框架污染 |
| `javax.servlet.*` / `jakarta.servlet.*` | 🔴 严重 | HTTP 协议污染业务 |
| `javax.persistence.EntityManager` | 🔴 严重 | ORM 直接出现在业务里 |
| `org.hibernate.*` | 🔴 严重 | Hibernate 污染 |
| `com.alibaba.fastjson.*` | 🟡 中等 | JSON 库污染 |
| `redis.clients.*` / `org.redisson.*` | 🟡 中等 | 缓存客户端污染 |
| `org.apache.kafka.*` | 🟡 中等 | MQ 客户端污染 |
| `feign.*` / `org.apache.http.*` | 🟡 中等 | HTTP 客户端污染 |
| `java.sql.*` | 🔴 严重 | JDBC 直接出现在业务里 |

允许的:

| Import 模式 | 说明 |
|------------|------|
| `java.*` | JDK 标准库 |
| `org.slf4j.*` | 纯日志接口 |
| `lombok.*` | 编译期注解,不影响运行时 |
| 自家 domain/usecase 包 | 内部依赖 |

### 规则 1.2:接口归属正确

这是 DIP 的精髓。**Repository / Gateway / Client 接口必须定义在业务核心里**,实现在 adapter 里。

判定方法:

```
查找所有 interface XxxRepository / XxxGateway / XxxClient / XxxQuery
对每一个:
  if (接口定义文件路径 ∈ adapter/infrastructure 包) {
    扣分:接口归属错误,违反 DIP
  } else if (接口定义文件路径 ∈ domain/usecase/application 包) {
    加分:接口归属正确
  }
```

错误示例:
```
src/
└── infrastructure/
    └── repository/
        ├── OrderRepository.java       ❌ 接口
        └── OrderRepositoryImpl.java       实现
```

正确示例:
```
src/
├── domain/
│   └── port/
│       └── OrderRepository.java       ✓ 接口在业务核心
└── infrastructure/
    └── repository/
        └── OrderRepositoryImpl.java       实现
```

### 规则 1.3:adapter 朝向正确

adapter 包**应该 import** 业务核心(实现端口接口)。
business core **不应该 import** adapter。

扫描:

```
查找 domain/usecase 下所有文件的 import 列表
  if (import com.xxx.adapter.* 或 com.xxx.infrastructure.*) {
    🔴 严重:依赖方向反了
  }
```

### 规则 1.4:删 adapter 测试

最硬核的判定:**临时把 adapter 包删掉,domain + usecase 还能不能编译?**

实操方法(报告里要给出这个测试的预期结果):

```
Step 1: 列出 domain + usecase 包的所有 import 列表
Step 2: 看这些 import 里有没有任何 adapter / infrastructure 包
Step 3: 看有没有 framework 包(框架强依赖)
Step 4: 给出"删 adapter 后能否编译"的判断
```

## Java/Spring 项目典型问题

### 问题 1.A:Service 类直接 @Autowired Repository 实现

```java
// ❌ Bad
@Service
public class OrderService {
    @Autowired
    private OrderJpaRepository repository;  // JPA Repository 直接出现
}
```

```java
// ✓ Good
public class OrderUseCase {
    private final OrderRepository repository;  // 端口接口
    
    public OrderUseCase(OrderRepository repository) {
        this.repository = repository;
    }
}
```

### 问题 1.B:Entity 上有 JPA 注解

```java
// ❌ Bad - domain 对象直接是 JPA 实体
@Entity
@Table(name = "orders")
public class Order {
    @Id @GeneratedValue
    private Long id;
}
```

```java
// ✓ Good - domain 对象纯净,JPA 实体单独
// domain/Order.java
public class Order {  // 纯 Java
    private OrderId id;
}

// adapter/persistence/OrderJpaEntity.java
@Entity
@Table(name = "orders")
class OrderJpaEntity {
    @Id @GeneratedValue
    private Long id;
}
```

### 问题 1.C:Domain 对象上有序列化注解

```java
// ❌ Bad
public class Order {
    @JsonProperty("order_id")
    private OrderId id;
}
```

序列化是 Web 边界的事,不该污染 domain。应该有单独的 Response DTO。

### 问题 1.D:业务代码里出现 HttpServletRequest

```java
// ❌ Bad
public class PlaceOrderUseCase {
    public Order placeOrder(HttpServletRequest request) {  // HTTP 入侵业务
        ...
    }
}
```

业务核心不应该知道 HTTP 存在。Controller 应该把 Request 翻译成业务命令再传给 Use Case。

## 评分标准

| 字母 | 标准 |
|------|------|
| **A** | 业务核心包零框架 import;接口归属 100% 正确;删 adapter 后业务核心可编译 |
| **B** | 业务核心包仅有 1-2 处轻度污染;接口归属基本正确(>90%);删 adapter 后基本可编译 |
| **C** | 业务核心有明显框架依赖,但已经分包;接口归属一半一半;典型 Spring 三层项目水平 |
| **D** | domain 包里大量 `@Entity` / `@Service` / `@Autowired`;接口和实现混在一起;删除任何一层都崩 |
| **F** | 没有 domain 概念;所有代码混在 service 包里;Controller 直接调 JPA |

## 输出格式

```markdown
## 维度 1:依赖方向

### 评分:[A/B/C/D/F]

### ✓ 做对了的
- 检测到 domain 包,与 adapter 包分离 (`src/main/java/com/xxx/domain/`)
- OrderRepository 接口定义在 `domain/port/` 中,归属正确

### ✗ 扣分项

#### 🔴 严重 - Spring 注解出现在业务核心
- 位置:`src/main/java/com/xxx/domain/order/Order.java:12`
- 现象层:`Order` 类上有 `@Entity`、`@Table`,字段上有 `@Column`
- 机制层:这些注解让 `Order` 类编译期依赖 JPA;一旦换持久化方式,Order 必须改
- 原理层:违反 Uncle Bob 第 32 章"框架是细节" —— 业务核心不应该感知持久化技术
- 改造建议:把 JPA 注解抽到 `OrderJpaEntity`(adapter 层),用 `OrderMapper` 在两者间转换

#### 🟡 中等 - JSON 注解出现在 domain 对象  
- 位置:`src/main/java/com/xxx/domain/order/Order.java:25`
- ...

### 🔧 改造建议(按 ROI 排序)
- [P0] 把 JPA 注解从 Order 抽到独立 OrderJpaEntity(成本:小,收益:大,可独立测试)
- [P1] 把 RestTemplate 调用包装成 PaymentGateway 接口...
```
