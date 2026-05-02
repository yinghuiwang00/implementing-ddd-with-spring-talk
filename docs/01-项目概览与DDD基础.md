# 项目概览与DDD基础

## 项目背景

本项目是Spring IO 2024演讲"Implementing Domain Driven Design with Spring"的示例代码，展示如何在Spring框架中实现领域驱动设计（Domain-Driven Design, DDD）。

### 业务场景

这是一个简单的图书馆管理系统，包含两个核心业务领域：
- **图书编目（Catalog）**：管理书籍信息、图书副本
- **图书借阅（Lending）**：管理借阅流程、借阅记录

### 技术栈
- Spring Boot 3.3.0
- Spring Modulith 1.2.0-RC1
- Spring Data JPA
- PostgreSQL数据库
- Java 21

## DDD核心概念

### 1. Ubiquitous Language（通用语言）

通用语言是DDD的核心思想，它要求开发团队和业务专家使用一致的术语来沟通和建模。

**项目中的体现：**

```java
// 领域对象名称直接反映业务术语
public class Book { }           // 书籍
public class Copy { }           // 副本
public class Loan { }           // 借阅
public record Isbn(String value) { }    // ISBN
public record BarCode(String code) { }  // 条形码
```

**业务术语映射：**
- 业务人员说"借一本书" → 代码中`RentBookUseCase`
- 业务人员说"还书" → 代码中`ReturnBookUseCase`
- 业务人员说"图书编号" → 代码中`BookId`、`CopyId`

### 2. Bounded Context（限界上下文）

限界上下文是DDD中最重要的概念之一。它将大型系统分解为多个独立的小型领域，每个领域有自己的模型和规则。

**项目中的限界上下文：**

```
library/
├── catalog/          # 编目上下文
│   ├── domain/
│   ├── application/
│   └── infrastructure/
└── lending/          # 借阅上下文
    ├── domain/
    └── application/
```

### 3. Domain Model（领域模型）

领域模型是对业务逻辑和规则的软件表达，它不是简单的数据结构，而是包含业务行为的对象。

**传统贫血模型 vs DDD充血模型：**

```java
// 贫血模型（传统方式）
public class Loan {
    private LocalDate expectedReturnDate;
    private LocalDateTime returnedAt;
    // 只有getter/setter，没有业务逻辑
}

// DDD充血模型（本项目方式）
public class Loan extends AbstractAggregateRoot<Loan> {
    private LocalDate expectedReturnDate;
    private LocalDateTime returnedAt;

    public void returned() {
        this.returnedAt = LocalDateTime.now();
        if (this.returnedAt.isAfter(expectedReturnDate.atStartOfDay())) {
            // 计算逾期费用（业务逻辑）
        }
        this.registerEvent(new LoanClosed(this.copyId));
    }
}
```

## 传统架构 vs DDD架构

### 传统分层架构

```
┌─────────────────────────────────────┐
│         Presentation Layer          │  ← Controller
├─────────────────────────────────────┤
│         Business Logic Layer        │  ← Service
├─────────────────────────────────────┤
│         Data Access Layer           │  ← DAO/Repository
├─────────────────────────────────────┤
│         Database Layer              │  ← Database
└─────────────────────────────────────┘
```

**问题：**
- 业务逻辑散落在各层
- 领域概念不清晰
- 难以维护复杂的业务规则
- 数据库设计主导系统设计

### DDD架构（本项目）

```
┌─────────────────────────────────────┐
│         Presentation Layer          │  ← Controller（未实现）
├─────────────────────────────────────┤
│         Application Layer           │  ← Use Cases
│         (Use Cases, Event Listeners)│
├─────────────────────────────────────┤
│         Domain Layer                │  ← Aggregates, Value Objects
│         (Domain Model, Events)      │
├─────────────────────────────────────┤
│         Infrastructure Layer        │  ← Repositories, External Services
├─────────────────────────────────────┤
│         Database, External APIs     │
└─────────────────────────────────────┘
```

**优势：**
- 领域逻辑集中在领域层
- 清晰的限界上下文边界
- 技术关注点与业务关注点分离
- 易于测试和维护

## 项目结构说明

```
library/
├── catalog/                          # 编目限界上下文
│   ├── domain/                       # 领域层
│   │   ├── Book.java                # 聚合根
│   │   ├── Copy.java                # 实体
│   │   ├── BookId.java              # 值对象
│   │   ├── CopyId.java              # 值对象
│   │   ├── Isbn.java                # 值对象
│   │   ├── BarCode.java             # 值对象
│   │   ├── BookRepository.java      # 仓储接口
│   │   └── CopyRepository.java      # 仓储接口
│   ├── application/                 # 应用层
│   │   ├── AddBookToCatalogUseCase.java
│   │   ├── RegisterBookCopyUseCase.java
│   │   ├── BookSearchService.java   # 应用服务
│   │   ├── DomainEventListener.java # 事件监听器
│   │   └── BookInformation.java     # DTO
│   └── infrastructure/              # 基础设施层
│       ├── OpenLibraryBookSearchService.java
│       └── OpenLibraryIsbnSearchResult.java
└── lending/                         # 借阅限界上下文
    ├── domain/
    │   ├── Loan.java                # 聚合根
    │   ├── LoanCreated.java         # 领域事件
    │   ├── LoanClosed.java          # 领域事件
    │   ├── LoanId.java              # 值对象
    │   ├── CopyId.java              # 值对象
    │   ├── UserId.java              # 值对象
    │   └── LoanRepository.java      # 仓储接口
    └── application/
        ├── RentBookUseCase.java
        └── ReturnBookUseCase.java
```

## DDD的战略性与战术性设计

### 战略性设计（Strategic Design）
- **限界上下文划分**：确定系统的边界
- **通用语言建立**：统一团队沟通语言
- **上下文映射**：定义上下文之间的关系

### 战术性设计（Tactical Design）
- **领域模型**：具体的编程实现
- **聚合与聚合根**：数据一致性边界
- **值对象**：不可变的业务概念
- **领域事件**：领域状态变化的表达
- **仓储**：持久化抽象

## 本项目的DDD实践总结

1. **清晰的限界上下文**：catalog和lending各自独立
2. **通用语言应用**：代码命名直接反映业务概念
3. **充血领域模型**：业务逻辑封装在领域对象中
4. **值对象使用**：ISBN、条形码等作为值对象
5. **领域事件**：跨上下文通信
6. **仓储模式**：抽象持久化细节
7. **Spring Modulith**：模块化架构验证

这个项目虽然简单，但完整展示了DDD的核心概念在Spring框架中的实现方式，是学习DDD实践的绝佳示例。
