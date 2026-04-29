# 维度 4:可测试性 (Testability)

> 可测试性是架构质量的"硬指标"。**架构错了,测试一定难写**。这一维度是用来反向验证前三维的。

## 核心原则

> 业务核心应该可以**不启动 Spring、不连数据库、不开 HTTP 服务**就完成单元测试。

如果做不到,说明业务核心还没有从外部世界中独立出来 —— 所有维度 1、2、3 的问题最终都会在这里暴露。

## 检查项

### 检查项 4.1:业务核心测试是否依赖 Spring 容器?

#### 信号扫描

在 `src/test/` 下查找业务核心的测试:

```
对每个测试类,检查类上的注解:
  @SpringBootTest          → 启动整个 Spring 容器(慢,单测里不应该有)
  @WebMvcTest              → 启动 Web 层(集成测试用,单测里不该有)
  @DataJpaTest             → 启动 JPA(集成测试用)
  @ContextConfiguration    → 加载 Spring 上下文
  @ExtendWith(SpringExtension.class)  → Spring 扩展
```

理想状态:**业务核心(Use Case 和 Entity)的测试零 Spring 注解**,只用 JUnit + Mockito 或者干脆手写假实现。

```java
// ❌ Bad - 单元测试启动 Spring
@SpringBootTest
public class PlaceOrderUseCaseTest {
    @Autowired private PlaceOrderUseCase useCase;
    @MockBean private OrderRepository repository;
    
    @Test
    void should_place_order() {
        // 启动 Spring 才能跑,几秒级
    }
}

// ✓ Good - 纯 Java 测试
public class PlaceOrderUseCaseTest {
    @Test
    void should_place_order_with_discount() {
        OrderRepository fakeRepo = new InMemoryOrderRepository();
        DiscountQuery fakeDiscount = m -> List.of(new FullReduction(50, 5));
        OrderNumberGenerator fakeGen = () -> OrderNumber.of("20260429120000123456");
        
        PlaceOrderUseCase useCase = new PlaceOrderUseCaseImpl(
            fakeRepo, fakeDiscount, fakeGen
        );
        
        OrderResult result = useCase.placeOrder(buildCommand("60.00"));
        
        assertThat(result.getFinalAmount()).isEqualTo(money("55.00"));
        // 毫秒级
    }
}
```

#### 量化指标

```
统计:
  total_tests = 业务核心的测试类总数
  spring_tests = 用了 @SpringBootTest 等 Spring 注解的测试数

spring_test_ratio = spring_tests / total_tests
```

| 比例 | 判定 |
|------|------|
| < 10% | A 级,业务核心独立 |
| 10%-30% | B 级,大部分业务可独立测 |
| 30%-60% | C 级,一半依赖 Spring |
| 60%-90% | D 级,基本测不动 |
| > 90% | F 级,所有测试都是集成测试 |

### 检查项 4.2:Mock 密度

业务核心一旦依赖正确(端口接口),测试时**不需要大量 mock**,直接写假实现(Fake)即可。

```java
// 正常情况:依赖 2-4 个端口,写 2-4 个 Fake 即可
public class PlaceOrderUseCase {
    private final OrderRepository repository;
    private final DiscountQuery discountQuery;
    private final OrderNumberGenerator generator;
}

// ❌ Bad - 大量 mock 是依赖混乱的信号
public class OrderServiceTest {
    @Mock private OrderJpaRepository jpaRepo;
    @Mock private UserRepository userRepo;
    @Mock private MerchantRepository merchantRepo;
    @Mock private ProductRepository productRepo;
    @Mock private CouponRepository couponRepo;
    @Mock private InventoryClient inventoryClient;
    @Mock private PaymentClient paymentClient;
    @Mock private NotificationService notification;
    @Mock private RedisTemplate redisTemplate;
    @Mock private KafkaTemplate kafkaTemplate;
    @Mock private ApplicationEventPublisher eventPublisher;
    @Mock private TransactionTemplate transactionTemplate;
    // 12 个 mock - 这个 Service 已经失控
}
```

**信号**:一个测试类里 `@Mock` 数量 > 5,通常说明:

- 被测类职责过多(违反 SRP)
- 依赖方向混乱(没有用端口收敛)
- 业务核心被外部细节淹没

### 检查项 4.3:测试是否依赖真实外部资源?

业务核心测试应该是**确定性的、毫秒级的、无副作用的**。

| 反模式 | 改造方向 |
|--------|---------|
| 测试连真实 MySQL | 用 InMemory Repository / H2 内存数据库 |
| 测试调真实 Redis | 用 InMemoryCache 假实现 |
| 测试发真实 HTTP | 用 Stub Gateway |
| 测试用 `Thread.sleep(1000)` 等异步 | 注入 Clock,可控时间;或同步执行的假实现 |
| 测试在不同时间跑结果不同 | 注入 Clock 而不是用 `LocalDateTime.now()` |

### 检查项 4.4:Fixture 构造的便利性

写测试时构造领域对象的难易程度,反映了领域模型的设计质量。

#### 信号:Builder/Factory 难用

```java
// ❌ 难写 - 必须设置所有字段
Order order = new Order();
order.setId(...);
order.setUserId(...);
order.setMerchantId(...);
order.setItems(...);
order.setStatus(...);
order.setCreatedAt(...);
// 20 行才能 new 一个 Order

// ✓ 易写 - 工厂方法 + Builder
Order order = OrderTestBuilder.newOrder()
    .withItems(item("商品 A", "10.00", 2))
    .withDelivery(delivery("张三", "13800000000", "上海"))
    .build();
```

如果项目里有大量 `XxxTestBuilder` / `XxxFixture` 工具类,通常是好信号(有人在认真写测试)。

如果没有,且测试代码里到处是 `new XXX()` + 一堆 `setXxx()`,通常是坏信号。

### 检查项 4.5:测试覆盖率的"形状"

不是看覆盖率数字,而是看**覆盖率分布**:

```
理想:
  Domain Entity 测试覆盖率 > 80%       ← 业务核心,必须高
  Use Case 测试覆盖率 > 70%            ← 流程,必须高
  Adapter 集成测试覆盖率 > 60%         ← 适配器,中等
  Controller 集成测试覆盖率 > 50%      ← 边界,基本

反模式:
  Controller 测试覆盖率 95%             ← 在测错位置
  Domain 测试覆盖率 5%                  ← 业务核心没测试
  全部都是 @SpringBootTest              ← 测试金字塔倒挂
```

### 检查项 4.6:架构守护测试

成熟项目应该有**自动化的架构合规测试**,防止架构腐烂。

Java 生态:**ArchUnit**

```java
@AnalyzeClasses(packages = "com.xxx")
public class ArchitectureTest {
    
    @ArchTest
    static final ArchRule domain_should_not_depend_on_infrastructure =
        noClasses().that().resideInAPackage("..domain..")
            .should().dependOnClassesThat().resideInAPackage("..infrastructure..");
    
    @ArchTest
    static final ArchRule domain_should_not_depend_on_spring =
        noClasses().that().resideInAPackage("..domain..")
            .should().dependOnClassesThat().resideInAPackage("org.springframework..");
    
    @ArchTest
    static final ArchRule controllers_should_only_be_called_by_spring =
        classes().that().resideInAPackage("..controller..")
            .should().beAnnotatedWith(RestController.class);
}
```

如果项目里有这种测试 → 加分(架构有人守护)
如果没有 → 视项目阶段决定(早期可不要,成熟项目应有)

## 评分标准

| 字母 | 标准 |
|------|------|
| **A** | 业务核心 100% 独立测试,Mock 密度低,有 ArchUnit 守护 |
| **B** | 大部分业务可独立测,少量依赖 Spring,有合理 Fixture |
| **C** | 一半测试是 `@SpringBootTest`,Mock 密度中等 |
| **D** | 几乎所有测试都启动 Spring,Mock 数量爆炸,测试运行 5+ 分钟 |
| **F** | 没有单元测试,只有人肉点击或集成测试 |

## 输出格式

```markdown
## 维度 4:可测试性

### 评分:D

### 量化数据
- 业务核心测试总数:23
- 启动 Spring 的测试数:21 (91%)
- 平均每个测试类的 @Mock 数量:8.4
- 单测套件运行耗时:4 分 30 秒

### ✓ 做对了的
- 有 OrderTestBuilder 帮助构造测试数据

### ✗ 扣分项

#### 🔴 严重 - 业务核心无法独立测试
- 位置:`src/test/java/com/xxx/order/`
- 现象层:91% 测试用 @SpringBootTest,启动整个容器
- 机制层:Service 类直接 @Autowired 多个依赖,无法单独构造
- 原理层:这是维度 1 依赖方向问题在测试侧的暴露
- 改造建议:对核心 Use Case 提取 InMemory Repository,改写 5-10 个高频测试为纯 JUnit 测试,体验差异;之后逐步迁移
```
