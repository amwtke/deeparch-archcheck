# 语言特定模式:Go(扩展点)

> Go 项目的架构评估。规模较小,主要给出关键差异点和扫描清单。

## Go 的架构光谱

Go 社区两种主流风格:

### 风格 A:Standard Project Layout

参考 [golang-standards/project-layout](https://github.com/golang-standards/project-layout)

```
project/
├── cmd/                  ← main 入口
├── internal/             ← 内部包,外部不可 import
│   ├── domain/           ← 领域对象
│   ├── usecase/          ← Use Case
│   └── infrastructure/   ← 实现
├── pkg/                  ← 可被外部 import 的包
└── api/                  ← API 定义(protobuf 等)
```

### 风格 B:Hexagonal/Clean Go

```
project/
├── cmd/
├── internal/
│   ├── domain/                ← 领域核心
│   ├── application/           ← Use Case + 端口
│   │   └── port/
│   └── adapter/
│       ├── handler/           ← HTTP 适配器
│       └── repository/        ← DB 适配器
```

## Go 特定的扫描规则

### 规则 G.1:domain 包的 import 纯净度

Go 没有 Spring 的注解污染,但有其他污染源:

**禁止在 domain 包出现**:

```go
import "github.com/gin-gonic/gin"            // HTTP 框架
import "gorm.io/gorm"                        // ORM
import "go.mongodb.org/mongo-driver/..."     // MongoDB
import "github.com/go-redis/redis/..."       // Redis
import "github.com/Shopify/sarama"           // Kafka
import "github.com/streadway/amqp"           // RabbitMQ
import "net/http"                            // HTTP 标准库,在 domain 里出现是问题
import "database/sql"                        // 标准 SQL,domain 里不该有
```

**允许出现**:

```go
import "context"          // Go 惯例,普遍接受
import "errors"
import "fmt"
import "time"
import "github.com/google/uuid"  // 纯类型库
```

### 规则 G.2:Tag 污染

Go 用 struct tag 处理序列化、ORM 映射:

```go
// ❌ Bad - Domain 对象上有所有 tag
type Order struct {
    ID     int64  `json:"id" gorm:"primarykey" db:"id"`
    Amount string `json:"amount" gorm:"column:amount" validate:"required"`
}
```

理想:domain 对象**没有 tag**,持久化和序列化用单独的结构体。

### 规则 G.3:接口归属

Go 的接口由"使用方"定义,这是 Go 的惯用法,**天然就符合端口适配器**。

```go
// usecase/place_order.go
package usecase

// 接口在 usecase 包定义(使用方)
type OrderRepository interface {
    Save(ctx context.Context, order *domain.Order) error
    FindByID(ctx context.Context, id domain.OrderID) (*domain.Order, error)
}

type PlaceOrderService struct {
    repo OrderRepository
}
```

```go
// adapter/repository/postgres_order_repo.go
package repository

import "myapp/internal/usecase"

type PostgresOrderRepository struct { ... }

// 实现 usecase.OrderRepository(隐式实现,不需要 implements 关键字)
func (r *PostgresOrderRepository) Save(ctx context.Context, o *domain.Order) error { ... }
```

**Go 的优势**:接口隐式实现,adapter 包不需要 import usecase 包就能实现接口(只要方法签名匹配)。这让端口适配器在 Go 里非常自然。

### 规则 G.4:`internal/` 目录的使用

Go 有语言级特性 `internal/`:`internal/` 下的包只能被同级或下级包 import。

合理使用 `internal/` 是架构边界的硬性保证。如果项目把所有业务都放在 `pkg/` 而不是 `internal/`,容易暴露内部细节。

### 规则 G.5:context 传递

`context.Context` 应该作为第一参数传递,但**不应该用 context 传业务参数**(反模式)。

```go
// ❌ Bad - 用 context 传业务参数
func PlaceOrder(ctx context.Context) error {
    userID := ctx.Value("userID").(int64)  // 反模式
}

// ✓ Good - context 只传请求级元数据
func PlaceOrder(ctx context.Context, cmd PlaceOrderCommand) error {
    // userID 在 cmd 里
}
```

### 规则 G.6:errors 处理

```go
// ❌ Bad - 吞错误
result, _ := someFunc()

// ❌ Bad - 包装无信息
if err != nil {
    return err
}

// ✓ Good - 包装有上下文
if err != nil {
    return fmt.Errorf("place order: %w", err)
}

// ✓ Better - 业务错误用 sentinel 或 typed error
var ErrOrderNotFound = errors.New("order not found")

type ValidationError struct {
    Field string
    Msg   string
}
```

业务错误应该是 domain 层的概念,有类型,可被上层判断。

## Go 特有的反模式

### 反模式 G.A:大 Service 包

```
service/
├── user_service.go      ← 1500 行
├── order_service.go     ← 2000 行
└── ...
```

Go 文件可以很大,但单文件 > 1000 行通常意味着应该拆。

### 反模式 G.B:全局变量

```go
var DB *gorm.DB
var Redis *redis.Client

func GetOrder(id int64) *Order {
    return DB.First(&Order{}, id)  // 全局依赖
}
```

应该通过依赖注入,接受接口参数。

### 反模式 G.C:`interface{}` / `any` 滥用

业务签名里出现大量 `any` 通常是设计问题。Go 是强类型语言,应该用具体类型。

## Go 评分要点

Go 项目评估时:

- 不必苛求"没有任何框架 import",Go 标准库的 `net/http` 在 handler 包里很正常
- 接口归属在 Go 里更容易做对(隐式实现)
- 关注 `internal/` 的合理使用
- 关注 struct tag 是否污染了 domain 对象

## 输出适配

Go 项目的报告维度和 Java 一样,但语言特定的扣分项放在每个维度的"Go 特有"小节中,例如:

```markdown
## 维度 1:依赖方向

### Go 特有问题
- `internal/domain/order.go:1` - import "gorm.io/gorm",domain 被 ORM 污染
- `internal/domain/order.go:8` - struct tag 包含 `gorm:"primarykey"`,违反纯净
```
