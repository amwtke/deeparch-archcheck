# 维度 2:边界完整性 (Boundary Integrity)

> Uncle Bob 第 17-18 章核心:**架构的本质是画对边界**。

边界画错的项目,会出现"改一处崩一片"的现象。这一维度专门检查边界。

## 四类边界

### 边界 A:Web 边界

**Controller(主动适配器)和 Use Case(业务核心)之间的边界**。

#### 检查项 A.1:Controller 是否瘦?

判定标准:Controller 的方法体应该 ≤ 10 行,只做三件事:

1. 把 HTTP 请求翻译成业务命令(Command)
2. 调用 Use Case
3. 把 Use Case 返回的结果翻译成 HTTP 响应(Response)

**信号**:Controller 里有业务判断、循环、计算 → 业务逻辑泄漏到边界

```java
// ❌ Bad - Controller 在做业务
@PostMapping("/orders")
public ResponseEntity<?> create(@RequestBody OrderDto dto) {
    if (dto.getItems().isEmpty()) {  // 业务校验
        return badRequest();
    }
    BigDecimal total = BigDecimal.ZERO;
    for (Item item : dto.getItems()) {       // 业务计算
        total = total.add(item.getPrice().multiply(...));
    }
    ...
}

// ✓ Good - Controller 只做翻译
@PostMapping("/orders")
public ResponseEntity<OrderResponse> create(@RequestBody OrderDto dto) {
    PlaceOrderCommand cmd = orderDtoMapper.toCommand(dto);
    OrderResult result = placeOrderUseCase.placeOrder(cmd);
    return ResponseEntity.ok(orderResponseMapper.toResponse(result));
}
```

#### 检查项 A.2:Controller 是否泄漏 ORM 实体?

```java
// ❌ Bad - JPA Entity 直接被序列化为 HTTP 响应
@GetMapping("/orders/{id}")
public OrderJpaEntity get(@PathVariable Long id) { ... }
```

#### 检查项 A.3:Controller 是否直接调 Repository?

跳过 Service/UseCase 直接调 Repository 是反架构信号:

```java
// ❌ Bad
@RestController
public class OrderController {
    @Autowired private OrderRepository repository;
    
    @GetMapping("/orders/{id}")
    public Order get(@PathVariable Long id) {
        return repository.findById(id);  // 跳过业务层
    }
}
```

### 边界 B:持久化边界

**Use Case 和数据库之间的边界**。

#### 检查项 B.1:Domain 对象 ≠ ORM 实体?

理想状态有两个类:`Order`(领域对象)和 `OrderJpaEntity`(持久化实体),通过 `OrderMapper` 转换。

```
Domain 对象:                ORM 实体:
- 反映业务概念                - 反映表结构  
- 有行为方法                  - 全是 getter/setter
- 强类型 ID(OrderId)         - 弱类型 ID(Long)
- 不可变值对象                - mutable
- 无注解                      - @Entity, @Column
```

**信号**:只有一个 Order 类,既有 `@Entity` 又有业务方法 → 两态混合,必扣分

#### 检查项 B.2:Repository 接口签名是业务语言还是 ORM 语言?

```java
// ❌ ORM 语言
interface OrderRepository {
    OrderJpaEntity findById(Long id);
    Page<OrderJpaEntity> findAll(Pageable pageable);
    void persist(OrderJpaEntity entity);
}

// ✓ 业务语言
interface OrderRepository {
    Optional<Order> findById(OrderId id);
    List<Order> findActiveOrdersOf(Merchant merchant);
    void save(Order order);
}
```

#### 检查项 B.3:业务代码里出现事务管理细节?

```java
// ❌ Bad - 事务管理细节泄漏
public class OrderService {
    public void placeOrder(...) {
        EntityManager em = ...;
        em.getTransaction().begin();  // ORM 细节
        ...
        em.getTransaction().commit();
    }
}
```

`@Transactional` 注解放在 Use Case 上是工程妥协(可接受,但要标注)。直接操作事务对象是严重违反。

### 边界 C:外部系统边界

**Use Case 和外部 API/MQ/Cache 之间的边界**。

#### 检查项 C.1:外部调用是否有端口包装?

```java
// ❌ Bad - 业务直接 new HttpClient
public class PaymentService {
    public void pay(Order order) {
        RestTemplate rest = new RestTemplate();
        rest.postForObject("https://pay.api/charge", ...);
    }
}

// ✓ Good - 通过端口
public class PaymentUseCase {
    private final PaymentGateway gateway;  // 端口接口
    
    public void pay(Order order) {
        gateway.charge(order.getId(), order.getAmount());
    }
}
```

#### 检查项 C.2:Redis/Kafka/RabbitMQ 客户端是否泄漏到业务?

业务代码里出现 `RedisTemplate`、`KafkaTemplate`、`RabbitTemplate` → 严重违反。

应该有 `CacheGateway`、`EventPublisher`、`MessageQueue` 这种端口接口。

#### 检查项 C.3:外部数据格式是否泄漏?

```java
// ❌ Bad - 第三方支付的字段名进入业务
public class Order {
    private String wxpayTransactionId;     // 微信特定字段
    private String alipayBuyerLogonId;     // 支付宝特定字段
}
```

应该用业务通用的 `paymentId` + `paymentMetadata`。

### 边界 D:模块/上下文边界

**不同业务领域之间的边界**(Context Boundary)。

#### 检查项 D.1:不同业务模块是否共享 Entity?

```java
// ❌ Bad - 订单模块和库存模块共享同一个 Product 类
package com.xxx.shared;
public class Product {
    private BigDecimal price;        // 订单模块关心
    private int stockCount;          // 库存模块关心
    private Date expireDate;         // 仓储模块关心
    private List<String> tags;       // 营销模块关心
}
```

DDD 的限界上下文(Bounded Context)告诉我们:**同一个"产品"在不同上下文里应该是不同的类**。

#### 检查项 D.2:模块间是否通过端口通信?

跨模块调用应该通过明确的接口,不能直接 import 别人的内部类。

```java
// ❌ Bad - 订单模块直接 import 库存模块的内部类
import com.xxx.inventory.internal.StockRepository;

// ✓ Good - 通过库存模块对外暴露的 API
import com.xxx.inventory.api.InventoryService;
```

#### 检查项 D.3:循环依赖检测

模块 A → 模块 B → 模块 A 是严重的架构腐烂信号。

## 评分标准

| 字母 | 标准 |
|------|------|
| **A** | 四类边界都清晰,Controller 瘦,Domain ≠ ORM,外部调用全包装,模块独立 |
| **B** | 主要边界清晰,有 1-2 处局部漏洞(如某个外部调用没包装) |
| **C** | Web 边界基本有(Controller + Service 分层),但 Domain ≈ ORM,外部调用混杂 |
| **D** | Controller 里有业务,Service 里有 SQL,模块间互相 import 内部类 |
| **F** | 没有边界概念,一个 Service 类里 HTTP/SQL/MQ/业务全混 |

## 输出格式

参考维度 1 的格式,每个边界单独一节:

```markdown
## 维度 2:边界完整性

### 评分:C

### 边界 A:Web 边界 - 良好
✓ Controller 与 Service 分层清晰
✗ OrderController.list() 方法返回了 OrderJpaEntity (位置:...)

### 边界 B:持久化边界 - 不及格
✗ Order 类同时是 Domain 对象和 JPA 实体...

### 边界 C:外部系统边界 - 一般
...

### 边界 D:模块边界 - 良好
...
```
