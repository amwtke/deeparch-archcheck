# 语言特定模式:Python(扩展点)

> Python 项目的架构评估。Python 生态有 Django、Flask、FastAPI 等不同流派,统一以"业务核心独立"为目标。

## Python 的架构光谱

### 风格 A:Django MVT(C 级常见)

```
project/
├── apps/
│   ├── orders/
│   │   ├── models.py         ← Django ORM,业务也在这
│   │   ├── views.py          ← 视图(类似 Controller)
│   │   ├── serializers.py    ← DRF 序列化
│   │   └── services.py       ← 有时有,业务集中
└── ...
```

**特征**:Models 既是 ORM 又是业务对象,典型的 ActiveRecord 模式。Django 的设计就是这样。

### 风格 B:Hexagonal Python

```
project/
├── domain/                ← 纯 Python 业务对象
│   └── order.py
├── application/
│   ├── ports/
│   └── use_cases/
├── adapter/
│   ├── api/               ← FastAPI/Flask handlers
│   └── repository/        ← SQLAlchemy 实现
└── infrastructure/
```

参考:Cosmic Python 一书(《Architecture Patterns with Python》)

## Python 特定的扫描规则

### 规则 P.1:domain 包的 import 纯净度

**禁止在 domain 出现**:

```python
from django.db import models           # Django ORM
from sqlalchemy import Column, ...     # SQLAlchemy ORM
from fastapi import ...                # FastAPI
from flask import ...                  # Flask
from rest_framework import ...         # DRF
from pydantic import BaseModel         # Pydantic(有争议,见下)
```

**Pydantic 的特殊性**:

Pydantic 是 Python 生态里类似"无害"的库,但严格说它仍然是一个外部依赖。社区有两派:

- **激进派**:domain 对象用 `dataclass` 或纯 class,不用 Pydantic
- **务实派**:domain 对象用 Pydantic,反正几乎所有现代 Python 项目都用,等于"事实标准库"

archcheck 的态度:**用 Pydantic 不扣分,但要标注"domain 对外部库 Pydantic 有依赖"**,让用户知情。

### 规则 P.2:Django Models 的双重身份

Django Models 天生混合了 ORM 和 ActiveRecord 模式:

```python
# Django 典型 Model
class Order(models.Model):
    user = models.ForeignKey(User, ...)
    total = models.DecimalField(...)
    
    def calculate_discount(self):  # 业务方法
        ...
    
    def save(self, *args, **kwargs):  # ActiveRecord:对象自己存自己
        ...
        super().save(*args, **kwargs)
```

这种设计**不符合端口适配器**,但**是 Django 的设计哲学**。

archcheck 的态度:

- **如果项目是纯 Django** → 不强求拆分,但建议把"重业务逻辑"提到 service 层或独立的 domain 类
- **如果项目想做整洁架构** → 拆成 `Order` (domain) + `OrderModel` (Django ORM) + `OrderRepository` (mapper),即"Repository Pattern over Django ORM"

### 规则 P.3:类型注解的使用

Python 是动态语言,但类型注解(Type Hints)对架构清晰度至关重要。

**好信号**:

```python
def place_order(cmd: PlaceOrderCommand) -> OrderResult:
    ...
```

**坏信号**:

```python
def place_order(data):  # 没有类型注解
    ...

def get_order(id: Any) -> dict:  # Any 和 dict 失去类型信息
    ...
```

类型注解覆盖率高 → 加分;`Any` / `dict` / `list` 滥用 → 减分。

### 规则 P.4:DI 实现

Python 没有 Spring 那种 DI 框架(虽然有 `dependency-injector` 等库),但应该用**构造函数注入**而不是模块级全局:

```python
# ❌ Bad - 全局依赖
db = create_engine(...)

class OrderService:
    def place_order(self, ...):
        with Session(db) as session:  # 依赖全局
            ...

# ✓ Good - 构造注入
class PlaceOrderUseCase:
    def __init__(self, repo: OrderRepository, ...):
        self._repo = repo
    
    def place_order(self, cmd: PlaceOrderCommand) -> OrderResult:
        order = Order.create(...)
        self._repo.save(order)
```

### 规则 P.5:协议(Protocol)的使用

Python 3.8+ 引入了 `typing.Protocol`,实现"结构化子类型"(类似 Go 的隐式接口):

```python
from typing import Protocol

class OrderRepository(Protocol):
    def save(self, order: Order) -> None: ...
    def find_by_id(self, id: OrderID) -> Order | None: ...

# adapter 不需要 import 这个 Protocol,只要方法签名匹配就行
class SqlAlchemyOrderRepository:
    def save(self, order: Order) -> None: ...
    def find_by_id(self, id: OrderID) -> Order | None: ...
```

使用 Protocol 是 Python 项目实现端口适配器的最佳方式。

## Python 特有的反模式

### 反模式 P.A:`*args, **kwargs` 滥用

业务方法签名用 `*args, **kwargs` 通常是设计偷懒,失去类型信息:

```python
# ❌ Bad
def place_order(*args, **kwargs):
    ...

# ✓ Good
def place_order(cmd: PlaceOrderCommand) -> OrderResult:
    ...
```

### 反模式 P.B:循环 import

Python 的循环 import 经常通过"延迟 import"绕开,但这是架构问题:

```python
# order/service.py
def place_order():
    from inventory.service import check_stock  # 延迟 import,通常掩盖循环依赖
    ...
```

延迟 import 是循环依赖的强信号。

### 反模式 P.C:全局可变状态

```python
# ❌ Bad
_cache = {}

def get_order(id):
    if id in _cache:
        return _cache[id]
    ...
```

### 反模式 P.D:Django 视图里写业务

```python
# ❌ Bad
def create_order_view(request):
    if request.POST['items']:  # 视图里做业务校验
        total = ...  # 视图里做业务计算
        Order.objects.create(...)  # 视图直接 ORM
```

应该提取到 service / use case 层。

## Python 评分要点

- 类型注解覆盖率高 → 加分(说明设计严谨)
- 用了 Protocol/抽象基类做端口 → 加分
- 纯 Django 项目不强求端口适配器(尊重框架哲学)
- 但 Django 项目应该有"业务集中点"(service.py 或独立 domain 模块)

## 输出适配

Python 报告里 Python 特有问题单独标注:

```markdown
## 维度 1:依赖方向

### Python 特有问题
- `domain/order.py:5` - 直接继承 `models.Model`,domain 与 Django ORM 不可分
- 类型注解覆盖率 23%,大量函数无类型标注
```
