# 语言特定模式:Java / Spring(主深)

> 这是 archcheck skill 的**主战场**。Java/Spring 项目占据了你将要审查的 80% 项目。

## Spring 应用的常见架构模式光谱

```
传统三层 ─────────── DDD-lite ─────────── 整洁架构 ────── 严格端口适配器
   C 级              B 级                    B+ 级           A 级

│                                                                    │
└── 大量 Spring 项目                            少数标杆项目 ─────┘
```

大部分企业 Spring 项目落在 **C 级到 B 级**之间。**直接拿 A 级标准苛求 C 级项目是不公平的**,要根据项目阶段和团队规模动态调整。

## 包结构识别

### 模式 A:传统三层(C 级常见)

```
com.xxx.app/
├── controller/       ← Spring MVC
├── service/          ← 业务逻辑全在这
├── dao/ 或 mapper/   ← MyBatis 或 JPA
├── entity/ 或 po/    ← 数据库实体
├── dto/              ← 传输对象
└── util/
```

**特征**:看包名就知道是 Spring 项目,看不出业务。

### 模式 B:DDD-lite(B 级常见)

```
com.xxx.app/
├── controller/
├── application/      ← 应用服务(类似 Use Case)
├── domain/
│   ├── model/        ← 领域对象
│   ├── service/      ← 领域服务
│   └── repository/   ← Repository 接口(归属正确!)
├── infrastructure/
│   ├── persistence/  ← Repository 实现 + JPA Entity
│   └── client/       ← 外部 API 客户端
└── interfaces/       ← 适配器统一目录
```

**特征**:有 domain 包,Repository 接口归属正确,但 Entity 上可能还有 JPA 注解。

### 模式 C:严格端口适配器/整洁架构(A 级)

```
com.xxx.app/
├── domain/                    ← Entity 层
│   ├── order/
│   │   ├── Order.java         (无任何框架注解)
│   │   ├── OrderId.java       (值对象)
│   │   └── OrderStatus.java
│   └── shared/
├── application/               ← Use Case 层
│   ├── port/
│   │   ├── in/                ← 入站端口
│   │   │   └── PlaceOrderUseCase.java
│   │   └── out/               ← 出站端口
│   │       ├── OrderRepository.java
│   │       └── DiscountQuery.java
│   └── service/
│       └── PlaceOrderService.java  (实现入站端口)
├── adapter/
│   ├── in/
│   │   └── web/
│   │       └── OrderController.java
│   └── out/
│       ├── persistence/
│       │   ├── OrderRepositoryImpl.java
│       │   ├── OrderJpaEntity.java   (JPA 注解都在这)
│       │   └── OrderMapper.java
│       └── client/
└── infrastructure/
    ├── config/                ← Spring 配置(@Configuration)
    └── BootApplication.java   ← Main
```

## Java/Spring 特定的扫描规则

### 规则 J.1:domain 包的纯净度扫描

对 `domain/` 包(或类似)下的所有 `.java` 文件,扫描 import 列表:

**禁止出现**(出现即扣分,严重度高):

```
import org.springframework.*
import javax.persistence.* / jakarta.persistence.*
import com.fasterxml.jackson.*
import javax.servlet.* / jakarta.servlet.*
import javax.validation.* / jakarta.validation.*  (Bean Validation 注解)
import org.hibernate.*
import com.alibaba.fastjson.*
import com.google.gson.*
import org.apache.commons.*  (除了 lang3.StringUtils 这种纯工具)
import java.sql.*
```

**警告出现**(中等扣分):

```
import lombok.*  (注解处理器,理论上编译期消失,可接受但不推荐 domain 用)
import org.slf4j.*  (日志,可接受)
```

**允许出现**:

```
import java.* / javax.* (JDK 标准库)
import 自家 com.xxx.domain.*
import 自家 com.xxx.application.port.*
```

### 规则 J.2:application 包的纯净度

`application/` (Use Case 层)允许少量 Spring 注解,但要警惕:

**可接受(工程妥协)**:

```
import org.springframework.stereotype.Service     (Spring 管理 bean)
import org.springframework.transaction.annotation.Transactional  (事务管理)
```

**应该警惕**:

```
import org.springframework.beans.factory.annotation.Autowired  (用构造注入代替)
import org.springframework.context.ApplicationEventPublisher  (用自家 EventPublisher 端口包装)
import javax.persistence.*  (绝不允许在 Use Case 层)
```

**不允许**:

```
任何 ORM、数据库、HTTP 相关 import
```

### 规则 J.3:Controller 厚度扫描

对 `controller/` 包下所有类:

```
对每个 @RequestMapping / @GetMapping / @PostMapping 方法:
  统计方法行数(去除空行和注释)
  if (行数 > 15) {
    🟡 警告:Controller 方法过长,可能业务逻辑泄漏
  }
  if (行数 > 30) {
    🔴 严重:Controller 严重违反单一职责
  }
  
  if (方法体里出现 if-else / for / while / switch 多于 1 处) {
    🔴 严重:Controller 在做业务判断
  }
  
  if (方法里直接调用 Repository 方法) {
    🔴 严重:跳过业务层
  }
```

### 规则 J.4:JPA Entity 与 Domain 是否分离?

```
查找所有带 @Entity 注解的类,记为 jpa_entities

查找所有 domain/ 包下的类,记为 domain_classes

if (jpa_entities ∩ domain_classes != ∅) {
  🔴 严重:Domain 对象与 JPA 实体混合
  对每一个混合类:
    标注扣分:位置 + 字段数 + 注解数
}
```

如果项目用 MyBatis 而不是 JPA,看 `@TableName`(MyBatis-Plus)、xml mapper 文件等价信号。

### 规则 J.5:Repository 接口归属

```
查找所有 interface XxxRepository / XxxDao / XxxMapper

对每一个接口,看:
  接口所在包路径
  
  if (路径包含 "infrastructure" / "dao" / "mapper" / "persistence") {
    if (有同名 Use Case 在 application/ 里调用它) {
      🔴 接口归属错误:应该把接口提到 domain 或 application/port/out 中
    }
  }
  
  if (接口里返回 Page<T> / Pageable / EntityManager 这种 Spring/JPA 类型) {
    🟡 接口签名是 ORM 语言而非业务语言
  }
```

### 规则 J.6:Spring 注解的位置

把整个项目的 Spring 注解使用做一次扫描:

| 注解 | 应出现的位置 | 不应出现的位置 |
|------|-------------|--------------|
| `@RestController` / `@Controller` | adapter/in/web | domain, application |
| `@Service` | application/service | domain |
| `@Repository` | adapter/out/persistence | domain, application |
| `@Component` | adapter, infrastructure | domain |
| `@Autowired` | (尽量用构造注入,不用注解) | domain |
| `@Configuration` | infrastructure/config | 其他任何地方 |
| `@Entity` / `@Table` | adapter/out/persistence | domain |
| `@Transactional` | application/service(妥协) | domain |
| `@EventListener` | adapter/out 的 handler | domain |

### 规则 J.7:常见反模式快速识别

#### "巨型 ServiceImpl"

```java
@Service
public class OrderServiceImpl implements OrderService {
    @Autowired private OrderMapper orderMapper;
    @Autowired private UserMapper userMapper;
    @Autowired private MerchantMapper merchantMapper;
    @Autowired private CouponMapper couponMapper;
    @Autowired private InventoryMapper inventoryMapper;
    @Autowired private PaymentMapper paymentMapper;
    @Autowired private RedisTemplate redis;
    @Autowired private KafkaTemplate kafka;
    @Autowired private RestTemplate rest;
    // 12+ 依赖
    
    // 方法 1:1500 行
    public void placeOrder(...) { ... }
    
    // 方法 2-50:每个 200-500 行
    ...
}

// 这个文件 8000 行
```

**信号**:`@Autowired` 数量 > 8,文件 > 2000 行

#### "RestTemplate 直接 new"

```java
public Order pay(Order order) {
    RestTemplate rest = new RestTemplate();  // 业务里 new 客户端
    rest.postForObject("https://pay.api/charge", ...);
}
```

#### "MyBatis 在 Service 里"

```java
@Service
public class OrderService {
    public Order get(Long id) {
        SqlSession session = sqlSessionFactory.openSession();  // ORM 直接出现
        return session.selectOne("Order.findById", id);
    }
}
```

## 工程妥协(可接受的"违反")

不是所有"违反整洁架构"都要扣分。以下是**工程实践中合理的妥协**,在评分时要考虑:

| 妥协 | 说明 |
|------|------|
| `@Service` 在 Use Case 类上 | 让 Spring 接管 bean 生命周期,工程上方便 |
| `@Transactional` 在 Use Case 方法上 | 事务管理是工程问题,放 Use Case 层是 Spring 项目的惯例 |
| 用 `ApplicationEventPublisher` 直接发事件 | 严格说违反,但实战常见。要标注"建议包装为 DomainEventPublisher 端口" |
| `@Component` 在 Adapter 类上 | adapter 本来就在框架层,无问题 |
| Controller 用 `@Valid` 做基础校验 | 这是 Web 层的边界检查,可接受 |

报告里遇到这些情况,**标注"已知工程妥协"**而不是扣分。

## Java/Spring 评分修正

对 Java/Spring 项目,以下情况评分**不应过苛**:

- 项目年龄 > 5 年 → 给基线 +1 个等级的宽容度
- 团队规模 > 30 人 → 给基线 +1 级宽容度(协作成本天然推向妥协)
- 用了 MyBatis(不是 JPA) → ORM 注解少,Domain 易于纯净,不应因此加分,但不应苛责"没用 JPA"
- 微服务架构 → 每个服务可单独评分,不强求大一统

## 推荐的标杆参考

如果想给项目作者展示"什么样是 A 级",推荐参考开源项目:

- **buckpal**(Tom Hombergs 的整洁架构示例项目,GitHub 上有)
- **leshalv/clean-architecture-spring-boot-example**
- **Spring 官方的 Spring Boot 例子**(注意:Spring 官方例子大多是 C 级,演示框架用法,不适合作为架构标杆)
