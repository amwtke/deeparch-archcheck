# 维度 5:代码坏味道 (Code Smells)

> 这一维度收纳前四维之外的、影响架构质量的坏味道。来自 Martin Fowler《重构》、Uncle Bob《Clean Code》、Vernon《IDDD》。

注意:**这是架构维度的坏味道,不是代码风格的坏味道**。命名拼写、缩进、变量命名这种交给 lint。

## 检查项

### 检查项 5.1:SOLID 五原则违反

#### S - 单一职责违反

**信号**:类的方法服务于不同的"actor"。

```java
// ❌ 违反 SRP - 一个类服务多个 actor
public class Employee {
    public Money calculatePay() {  // 服务 CFO
        return regularHours.times(payRate).plus(...);
    }
    
    public String reportHours() {  // 服务 COO
        return formatTimesheet();
    }
    
    public void save() {  // 服务 CTO
        database.save(this);
    }
}
```

类规模信号:

- 类 > 500 行 → 警告
- 类 > 1000 行 → 严重
- 类有 > 20 个公开方法 → 警告

#### O - 开闭原则违反

**信号**:每加一种新业务类型都要改同一个类的 `if-else` / `switch`。

```java
// ❌ 违反 OCP
public class DiscountCalculator {
    public Money calculate(Order order, String discountType) {
        if ("FULL_REDUCTION".equals(discountType)) { ... }
        else if ("PERCENTAGE".equals(discountType)) { ... }
        else if ("SPECIAL_PRICE".equals(discountType)) { ... }
        // 加新折扣?改这里
    }
}
```

正确做法:策略模式 + 依赖注入。

#### L - 里氏替换违反

**信号**:子类覆写父类方法,但行为不一致(抛异常、改变契约)。

```java
// ❌ 违反 LSP
class Rectangle {
    public void setWidth(int w) { this.width = w; }
    public void setHeight(int h) { this.height = h; }
}

class Square extends Rectangle {
    @Override
    public void setWidth(int w) {  // 改变了父类语义
        this.width = w;
        this.height = w;
    }
}
```

#### I - 接口隔离违反

**信号**:实现类被迫实现自己不需要的方法(经常是空实现或抛 UnsupportedException)。

```java
// ❌ 违反 ISP
interface Repository<T> {
    T findById(Long id);
    List<T> findAll();
    void save(T entity);
    void delete(Long id);
    void batchInsert(List<T> entities);
    Page<T> findByPage(Pageable p);
    long count();
    boolean exists(Long id);
    // 12 个方法
}

class OrderRepository implements Repository<Order> {
    @Override
    public void batchInsert(List<Order> orders) {
        throw new UnsupportedOperationException();  // 不需要但被迫实现
    }
}
```

#### D - 依赖反转违反(已在维度 1 详细讲)

接口归属错误,业务依赖具体实现。

### 检查项 5.2:巨型类(God Class)

**度量**:

- 类行数 > 1000 → 警告
- 类公开方法 > 30 → 警告
- 类字段 > 20 → 警告
- 类导入(import)数 > 30 → 警告

**典型坏名字**:

- `XxxManager`
- `XxxHelper`
- `XxxUtil` / `XxxUtils`
- `BusinessService`
- `OrderProcessor`

这些名字本身就是坏味道,因为它们描述的是"做事"而不是"是什么"。Manager 在"管"什么?Helper 在"帮"谁?

### 检查项 5.3:DTO / Domain / Entity 三态混淆

正常情况下应该有三种不同的"对象":

| 角色 | 用途 | 例子 | 注解 |
|------|------|------|------|
| **DTO** | Web 边界传输 | `OrderDto`, `OrderRequest`, `OrderResponse` | `@JsonProperty` |
| **Domain Object** | 业务核心 | `Order`, `OrderItem` | 无任何框架注解 |
| **JPA Entity** | 持久化 | `OrderJpaEntity` | `@Entity`, `@Column` |

**反模式 - 一个类承担三角色**:

```java
// ❌ 三态合一
@Entity
@Table(name = "orders")
public class Order {  // 既是 JPA 实体,又是业务对象,又被序列化为 JSON
    @Id @GeneratedValue
    @JsonProperty("order_id")
    private Long id;
    
    // 业务方法
    public Money calculateTotal() { ... }
    
    // ... 同时出现 Spring Data 方法签名匹配
}
```

这种类一改,数据库迁移、API 兼容、业务逻辑全受影响。

### 检查项 5.4:过深继承层级

继承超过 3 层通常是设计问题。考虑组合代替继承。

```
class A
  └── class B extends A
        └── class C extends B
              └── class D extends C  ← 已经过深
```

### 检查项 5.5:循环依赖

模块/包/类之间的循环依赖是**严重**的架构腐烂信号。

Java 圈检测工具:

- IntelliJ IDEA 自带"Cyclic dependency"分析
- ArchUnit 的 `slices().should().beFreeOfCycles()`
- jdeps 命令行工具

报告里如果发现循环依赖,**应该明确指出**:

```
❌ 检测到包间循环依赖
  com.xxx.order → com.xxx.user → com.xxx.order
  
影响:
  - 编译耦合,改一处影响多处
  - 无法单独打包/部署
  - 测试时必须一起加载
```

### 检查项 5.6:命名失真

业务概念的名字应该来自业务领域(DDD 统一语言),不该是技术词汇。

| 失真命名 | 业务命名 |
|---------|---------|
| `OrderProcessor` | `PlaceOrderUseCase` |
| `OrderHandler` | `OrderService`(领域服务)或具体 UseCase |
| `OrderManager` | 拆成多个具体职责 |
| `dataMap` | `customerProfile` |
| `obj1`, `tempVar` | 表达意图的名字 |

特别警告的名字(强坏味道):

- `BaseService` / `AbstractService`(继承 vs 组合权衡)
- `CommonUtils` / `CommonService`(垃圾桶名字)
- `CoreXxx` / `BaseXxx`(没说清楚是什么)

### 检查项 5.7:数据库化的领域对象

#### 信号:Entity 字段名带技术后缀

```java
// ❌ Bad
public class Order {
    private Long orderId;          // _id 后缀是表设计思维
    private String orderNo;
    private Date createTime;       // create_time 是 SQL 字段名
    private Date updateTime;
    private Integer isDeleted;     // is_deleted 是软删除标记
    private Integer status;        // 用 int 表示业务状态
}

// ✓ Good
public class Order {
    private OrderId id;
    private OrderNumber number;
    private OrderStatus status;    // 枚举,不用 int
    // 无 createTime/updateTime/isDeleted - 这些是持久化关心的,放 JPA 实体上
}
```

#### 信号:用 `Map<String, Object>` 当业务模型

```java
// ❌ Bad - 失去类型安全
public Map<String, Object> getOrder(Long id) { ... }
```

### 检查项 5.8:异常处理反模式

```java
// ❌ Bad - 吞异常
try { ... } catch (Exception e) { /* nothing */ }

// ❌ Bad - 异常即流程控制
try { 
    return parseInt(s);
} catch (NumberFormatException e) {
    return 0;
}

// ❌ Bad - 业务异常用 RuntimeException
throw new RuntimeException("订单不存在");

// ✓ Good - 业务异常有明确语义
throw new OrderNotFoundException(orderId);
```

业务异常应该是**领域概念**,定义在 domain 层。

### 检查项 5.9:静态工具类滥用

业务逻辑放在 `static` 方法里,无法注入、无法 mock、无法替换。

```java
// ❌ Bad - 业务在静态方法里
public class OrderUtils {
    public static Money calculateTotal(List<Item> items) { ... }
    public static boolean canCancel(Order order) { ... }
}
```

工具类只应该承载**纯函数式的、无业务的**逻辑(比如字符串工具)。业务逻辑必须有归属的对象。

## 评分标准

| 字母 | 标准 |
|------|------|
| **A** | SOLID 普遍遵守,无巨型类,DTO/Domain/Entity 分离,无循环依赖 |
| **B** | 个别地方违反 SOLID,有少量巨型类,但整体健康 |
| **C** | 一些经典坏味道(贫血 + Service 巨型 + DTO/Entity 混淆),典型 Spring 项目 |
| **D** | 多种坏味道并存,有循环依赖,巨型类多,命名严重失真 |
| **F** | 烂泥坑,几乎所有坏味道都能找到例子 |

## 输出格式

```markdown
## 维度 5:代码坏味道

### 评分:D

### ✓ 做对了的
- ...

### ✗ 扣分项

#### 🔴 严重 - 巨型类
- `OrderService.java:1` - 1842 行,42 个公开方法,28 个字段
- `BusinessHelper.java:1` - 970 行
- ...

#### 🔴 严重 - 三态混淆
- `Order.java` 同时是:
  - JPA 实体(`@Entity`)
  - JSON 响应(`@JsonProperty`)
  - 业务对象(有 `calculateTotal()`)

#### 🟡 中等 - SOLID 违反
##### OCP - DiscountCalculator
- 位置:`DiscountCalculator.java:45-89`
- 6 种折扣类型用 if-else 串联
- 改造建议:策略模式 + DiscountStrategy 接口

##### SRP - OrderService
- ...

#### 🟡 中等 - 命名失真
- `OrderProcessor`、`OrderHandler`、`OrderManager` 同时存在,职责不清
```
