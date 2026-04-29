# 维度 3:Entity vs UseCase 区分

> Uncle Bob 第 20 章核心:**企业级业务规则**(Entity)与**应用级业务规则**(Use Case)必须分开。

这是整洁架构对端口适配器的精化。区分这两层,你才能让"订单的不变量"和"下单流程"分别演化。

## 核心区分

| 维度 | Entity(企业级业务规则) | Use Case(应用级业务规则) |
|------|------------------------|-----------------------|
| 本质 | 业务概念本身 | 业务流程 |
| 时间 | 即使没有这套软件也存在 | 只存在于这套软件里 |
| 例子 | 订单、用户、商品 | 下单、注册、支付 |
| 行为 | 状态合法性、不变量 | 多对象编排、调端口 |
| 是否依赖端口 | **不依赖** | 依赖 |
| 是否可替换 | 不可替换 | 较少替换 |

## 检查项

### 检查项 3.1:充血还是贫血?

#### 贫血模型(Anemic Domain Model)的特征

- 类只有字段 + getter/setter,没有业务方法
- 业务规则全在 Service 里
- 类和数据库表一一对应

```java
// ❌ 贫血模型
public class Order {
    private OrderId id;
    private List<OrderItem> items;
    private Money totalAmount;
    private OrderStatus status;
    
    // 只有 getter/setter
    public OrderId getId() { return id; }
    public void setId(OrderId id) { this.id = id; }
    // ...
}

@Service
public class OrderService {
    public void placeOrder(Order order) {
        if (order.getItems().isEmpty()) throw new ...;
        BigDecimal total = ...; // 在 Service 里计算
        order.setTotalAmount(total);
        order.setStatus(OrderStatus.PENDING);
        repository.save(order);
    }
}
```

**信号**:Service 类的方法体里有大量 `xxx.setYyy(...)` → 贫血模型

#### 充血模型(Rich Domain Model)

- 业务方法在领域对象上
- 状态变更通过方法,不通过 setter
- 不变量在构造方法/工厂方法中校验

```java
// ✓ 充血模型
public class Order {
    private final OrderId id;
    private final List<OrderItem> items;
    private Money totalAmount;
    private OrderStatus status;
    
    // 工厂方法:校验不变量
    public static Order create(MerchantId merchantId, List<OrderItem> items, ...) {
        if (items.isEmpty()) throw new BusinessException("订单至少有一个商品");
        if (!sameMerchant(items)) throw new BusinessException("订单只能属于一个商家");
        Order order = new Order(...);
        order.calculateTotal();
        return order;
    }
    
    // 业务方法:封装状态变更
    public void applyDiscount(Discount discount) {
        DiscountResult result = discount.apply(this);
        this.totalAmount = this.totalAmount.subtract(result.getDiscountAmount());
    }
    
    public void cancel() {
        if (this.status != OrderStatus.PENDING_PAYMENT) {
            throw new BusinessException("已支付订单不能取消");
        }
        this.status = OrderStatus.CANCELLED;
    }
    
    // 没有 setStatus(), setTotalAmount() 这种暴露的 setter
}
```

#### 量化指标

写个简单脚本统计:

```
对每个 domain 包下的类,统计:
  方法总数 = 公开方法
  纯 setter 方法数 = 名字以 set 开头且只有一行赋值
  纯 getter 方法数 = 名字以 get 开头且只 return
  业务方法数 = 方法总数 - getter 数 - setter 数 - constructor

业务方法占比 = 业务方法数 / 方法总数
```

| 比例 | 判定 |
|------|------|
| > 50% | 充血,A 级 |
| 30%-50% | 中等,B 级 |
| 10%-30% | 偏贫血,C 级 |
| < 10% | 完全贫血,D-F 级 |

### 检查项 3.2:业务规则散落在哪里?

理想分布:

```
Entity:
  - 状态合法性校验(订单只能属于一个商家)
  - 数学计算(订单总价 = sum(items))
  - 状态机(待支付 → 已支付 → 已发货)

Use Case:
  - 流程编排(查折扣 → 创建订单 → 应用折扣 → 持久化 → 发事件)
  - 跨对象协调(检查库存 + 创建订单 + 扣减库存)
  - 端口调用(调 Repository、调 Gateway)

Service(如果有):
  - 跨聚合的领域服务
```

**反模式 - "上帝 Service"**:

```java
// ❌ 一个 Service 包打所有业务
@Service
public class OrderService {
    public void placeOrder(...) {
        // 校验:在 Service 里
        if (items.isEmpty()) ...
        
        // 计算:在 Service 里
        BigDecimal total = ...;
        
        // 状态机:在 Service 里
        if (currentStatus == ... ) ...
        
        // 编排:在 Service 里
        repository.save(...);
        eventPublisher.publish(...);
    }
}

// 这种 Service 通常 500-2000 行
// Order 类只有 10 行 getter/setter
```

### 检查项 3.3:Use Case 是否纯编排?

Use Case 应该**像导演,不像演员**。它调度 Entity 和 Repository,但不亲自做业务计算。

```java
// ❌ Bad - Use Case 在做业务计算
public class PlaceOrderUseCase {
    public OrderResult placeOrder(PlaceOrderCommand cmd) {
        // 业务计算应该在 Order 里做,不该在这
        BigDecimal total = cmd.getItems().stream()
            .map(i -> i.getPrice().multiply(BigDecimal.valueOf(i.getCount())))
            .reduce(BigDecimal.ZERO, BigDecimal::add);
        
        BigDecimal final = total.add(BigDecimal.valueOf(1)).add(BigDecimal.valueOf(3));
        
        Order order = new Order();
        order.setTotal(final);
        // ...
    }
}

// ✓ Good - Use Case 只编排
public class PlaceOrderUseCase {
    public OrderResult placeOrder(PlaceOrderCommand cmd) {
        // 1. 创建领域对象,业务规则在 Order.create() 里
        Order order = Order.create(cmd.getMerchantId(), cmd.getItems(), cmd.getDelivery());
        
        // 2. 应用折扣,折扣计算在 Discount.apply() 里
        List<Discount> discounts = discountQuery.findActive(cmd.getMerchantId());
        discounts.forEach(order::applyDiscount);
        
        // 3. 持久化
        orderRepository.save(order);
        
        // 4. 发事件
        eventPublisher.publish(new OrderPlacedEvent(order));
        
        return OrderResult.from(order);
    }
}
```

### 检查项 3.4:值对象的使用

业务概念应该用**值对象**(Value Object)封装,而不是用基础类型(Primitive Obsession 反模式)。

| 反模式 | 推荐 |
|--------|------|
| `String email` | `Email email` |
| `BigDecimal amount` | `Money amount` |
| `Long id` | `OrderId id` |
| `String phoneNumber` | `PhoneNumber phoneNumber` |
| `String address` | `Address address` |

值对象的好处:

- 不变量在构造时校验(创建出来就一定合法)
- 类型系统帮你抓 bug(`OrderId` 不能传给要 `UserId` 的方法)
- 业务方法集中(`Money.add()`、`Money.subtract()`)

### 检查项 3.5:聚合边界是否合理?

DDD 聚合设计的四条原则(Vernon《IDDD》第 10 章):

1. 在一致性边界内建模真正的不变量
2. 设计**小**聚合
3. 通过唯一标识引用其他聚合
4. 边界外使用最终一致性

**反模式**:把所有相关对象都塞进一个聚合(巨型聚合)

```java
// ❌ Bad - 巨型聚合
public class Order {
    private List<OrderItem> items;
    private User user;            // 应该用 UserId 引用
    private Merchant merchant;    // 应该用 MerchantId 引用
    private List<Coupon> coupons; // 应该独立聚合
    private Address address;
    private List<DeliveryStatus> deliveryHistory;
    // ... 50+ 字段
}
```

## 评分标准

| 字母 | 标准 |
|------|------|
| **A** | 充血模型,业务规则在 Entity,Use Case 纯编排,值对象普及,聚合合理 |
| **B** | 大部分业务在 Entity,但有少量泄漏到 Use Case;基础类型偶尔用 |
| **C** | 半充血半贫血,业务规则一半在 Entity 一半在 Service;基础类型为主 |
| **D** | 几乎完全贫血,Entity 是 POJO,业务全在巨型 Service 里 |
| **F** | 没有 Domain 概念,只有 Controller-Service-DAO,Entity 就是 ORM 表映射 |

## 输出格式

```markdown
## 维度 3:Entity vs UseCase 区分

### 评分:D

### ✓ 做对了的
- 有 Order, OrderItem, Money 等领域类
- Money 是值对象,有 add/subtract 方法

### ✗ 扣分项

#### 🔴 严重 - 贫血模型
- 位置:`src/main/java/com/xxx/domain/Order.java`
- 现象层:Order 类有 15 个字段,15 个 getter,15 个 setter,1 个无参构造,无任何业务方法
- 机制层:业务规则全在 OrderService.placeOrder() 里(560 行)
- 原理层:违反 Uncle Bob 第 20 章 - Entity 应该承载企业级业务规则
- 业务方法占比:0/30 = 0%
- 改造建议:把 OrderService 里的校验、计算、状态机迁移到 Order 类的工厂方法和实例方法里

#### 🟡 中等 - 基础类型滥用
- 位置:多处
- 现象层:订单 ID 用 `Long`,金额用 `BigDecimal`,邮箱用 `String`
- 改造建议:引入 OrderId, Money, Email 值对象
```
