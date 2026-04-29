# 语言特定模式:TypeScript / Node.js(扩展点)

> Node.js / TypeScript 后端项目。NestJS 走 Spring 路线,Express 灵活,各有评估方式。

## TypeScript 后端的架构光谱

### 风格 A:Express 自由派

```
src/
├── routes/
├── controllers/
├── services/
├── models/        ← 通常是 Mongoose / Sequelize 模型
└── middlewares/
```

**特征**:类似 Spring 三层,但更松散。

### 风格 B:NestJS(Spring 在 Node 的对应物)

```
src/
├── orders/
│   ├── orders.controller.ts
│   ├── orders.service.ts
│   ├── orders.module.ts
│   └── dto/
└── ...
```

**特征**:有 IoC、装饰器、Module 概念,和 Spring Boot 高度相似。同样面临注解污染问题。

### 风格 C:Hexagonal TypeScript

```
src/
├── domain/
│   ├── order/
│   │   ├── order.ts
│   │   └── order-id.ts
├── application/
│   ├── ports/
│   └── use-cases/
├── adapter/
│   ├── http/
│   └── persistence/
└── infrastructure/
```

## TypeScript 特定的扫描规则

### 规则 T.1:domain 包的 import 纯净度

**禁止在 domain 出现**:

```typescript
import { Injectable } from '@nestjs/common';        // NestJS 装饰器
import { Entity, Column } from 'typeorm';            // TypeORM
import { Schema, Prop } from '@nestjs/mongoose';     // Mongoose+NestJS
import express from 'express';                       // Express
import { Request, Response } from 'express';
import axios from 'axios';                           // HTTP 客户端
import Redis from 'ioredis';                         // Redis
import { Logger } from '@nestjs/common';             // 框架 Logger
```

**允许出现**:

```typescript
import { v4 as uuid } from 'uuid';   // 纯类型库
// 自家 domain / port import
```

### 规则 T.2:装饰器污染

NestJS 用装饰器实现 IoC,但装饰器 = 注解 = 编译期依赖:

```typescript
// ❌ Bad - 业务核心被 NestJS 装饰器污染
@Injectable()  // 来自 @nestjs/common
export class PlaceOrderService {
    constructor(
        @Inject('OrderRepository')  // NestJS 特定注入
        private readonly repo: OrderRepository,
    ) {}
}
```

**工程妥协**:用 NestJS 就接受 `@Injectable`,但应该把核心业务逻辑提到无装饰器的纯类中。

### 规则 T.3:TypeORM Entity 与 Domain 分离

```typescript
// ❌ Bad - TypeORM 实体直接是业务对象
@Entity('orders')
export class Order {
    @PrimaryGeneratedColumn()
    id: number;
    
    @Column()
    totalAmount: number;
    
    calculateDiscount(): void { ... }  // 业务方法混在 ORM 实体里
}

// ✓ Good - 分离
// domain/order.ts
export class Order {
    constructor(
        public readonly id: OrderId,
        public readonly items: OrderItem[],
        // ...
    ) {}
    
    static create(...): Order { ... }
    applyDiscount(d: Discount): void { ... }
}

// adapter/persistence/order.entity.ts
@Entity('orders')
export class OrderEntity {
    @PrimaryGeneratedColumn()
    id: number;
    
    @Column()
    totalAmount: number;
}

// adapter/persistence/order.mapper.ts
export class OrderMapper {
    static toDomain(entity: OrderEntity): Order { ... }
    static toEntity(domain: Order): OrderEntity { ... }
}
```

### 规则 T.4:类型严格度

TypeScript 的核心价值是类型,但很多项目用得很松散:

**坏信号**:

```typescript
function placeOrder(data: any): any { ... }  // any 滥用
function getOrder(id): Order { ... }          // 隐式 any
```

**好信号**:

```typescript
function placeOrder(cmd: PlaceOrderCommand): Promise<OrderResult> { ... }
```

`tsconfig.json` 里有没有 `strict: true`、`noImplicitAny: true`,是项目类型严格度的指标。

### 规则 T.5:接口归属(TypeScript 接口)

TypeScript 的 `interface` 完美适合做端口:

```typescript
// application/ports/order-repository.ts
export interface OrderRepository {
    save(order: Order): Promise<void>;
    findById(id: OrderId): Promise<Order | null>;
}

// adapter/persistence/typeorm-order-repository.ts
export class TypeOrmOrderRepository implements OrderRepository {
    async save(order: Order): Promise<void> { ... }
    async findById(id: OrderId): Promise<Order | null> { ... }
}
```

扫描:`Repository` / `Gateway` / `Client` 后缀的接口定义在哪个目录?

## TypeScript 特有的反模式

### 反模式 T.A:`as` 强制转换滥用

```typescript
// ❌ Bad - 用 as 绕过类型检查
const order = data as Order;
```

业务代码里大量 `as` 是类型设计有问题的信号。

### 反模式 T.B:Promise 链不断

```typescript
// ❌ Bad - 嵌套 then
function placeOrder(cmd) {
    return repo.findUser(cmd.userId)
        .then(user => repo.findMerchant(cmd.merchantId)
            .then(merchant => ...))
        .then(...)
        ...
}

// ✓ Good - async/await
async function placeOrder(cmd: PlaceOrderCommand): Promise<OrderResult> {
    const user = await repo.findUser(cmd.userId);
    const merchant = await repo.findMerchant(cmd.merchantId);
    // ...
}
```

### 反模式 T.C:过度依赖 NestJS Module

NestJS 的 `Module` 系统容易让人把架构层次"塌缩"成 Module 平铺,丢失业务边界。

### 反模式 T.D:DTO 失控

JavaScript/TypeScript 项目的 DTO 经常等于"任何长得像数据的对象",最终所有边界都模糊。

应该有清晰的:

- Request DTO(API 入参)
- Response DTO(API 出参)
- Command(应用层入参)
- Domain Object(领域对象)

## TypeScript 评分要点

- `tsconfig.json` 配置严格 → 加分
- 接口归属正确(端口在 application 层) → 加分
- TypeORM Entity 与 Domain 分离 → 加分
- 用了 Protocol-style 接口而不是 abstract class → 中性
- 大量 `any` → 减分
- NestJS 装饰器在 domain → 减分(工程妥协可商量)

## 输出适配

```markdown
## 维度 1:依赖方向

### TypeScript 特有问题
- `domain/order.ts:1` - import { Entity } from 'typeorm',domain 被 ORM 污染
- `tsconfig.json` 中 strict 为 false,类型保护弱
- 业务代码出现 23 处 `any` 类型
```
