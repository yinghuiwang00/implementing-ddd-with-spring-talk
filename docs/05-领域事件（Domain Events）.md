# 领域事件（Domain Events）

## 领域事件的概念

领域事件是DDD中用于表示领域中发生的事情。当领域中的某个重要事情发生时，系统会发布一个领域事件，其他部分可以订阅并响应这个事件。

### 领域事件的特点

1. **过去时态**：事件描述已经发生的事情（如`LoanCreated`而不是`CreateLoan`）
2. **不可变**：一旦发布就不能修改
3. **携带上下文**：包含足够的信息让订阅者理解发生了什么
4. **最终一致性**：事件处理是异步的，不需要立即完成

### 领域事件的作用

1. **解耦限界上下文**：不同上下文通过事件通信
2. **实现最终一致性**：避免分布式事务
3. **触发副作用**：事件发生时执行相关操作
4. **记录历史**：事件可以作为审计日志

## 项目中的领域事件

### 1. LoanCreated（借阅创建事件）

```java
package library.lending.domain;

public record LoanCreated(CopyId copyId) {
}
```

**事件含义：**
- 表示一个新的借阅记录被创建
- 发生时间：在`Loan`聚合构造时
- 携带信息：被借阅的副本ID

**发布位置：**
```java
public class Loan extends AbstractAggregateRoot<Loan> {
    public Loan(CopyId copyId, UserId userId, LoanRepository loanRepository) {
        // ... 验证和初始化
        this.registerEvent(new LoanCreated(this.copyId));
    }
}
```

### 2. LoanClosed（借阅关闭事件）

```java
package library.lending.domain;

public record LoanClosed(CopyId copyId) {
}
```

**事件含义：**
- 表示一个借阅记录被关闭（图书归还）
- 发生时间：在`Loan.returned()`方法中
- 携带信息：归还的副本ID

**发布位置：**
```java
public class Loan extends AbstractAggregateRoot<Loan> {
    public void returned() {
        this.returnedAt = LocalDateTime.now();

        if (this.returnedAt.isAfter(expectedReturnDate.atStartOfDay())) {
            // 计算逾期费用
        }

        this.registerEvent(new LoanClosed(this.copyId));
    }
}
```

## 领域事件的发布

### 使用AbstractAggregateRoot

```java
@Entity
public class Loan extends AbstractAggregateRoot<Loan> {
    @EmbeddedId
    private LoanId loanId;
    // ... 其他字段

    public Loan(CopyId copyId, UserId userId, LoanRepository loanRepository) {
        // ... 验证和初始化
        this.registerEvent(new LoanCreated(this.copyId));
    }

    public void returned() {
        // ... 更新状态
        this.registerEvent(new LoanClosed(this.copyId));
    }
}
```

**AbstractAggregateRoot的作用：**
- 提供事件注册机制
- 自动收集聚合中的所有事件
- 与Spring Data JPA集成，在保存时自动发布事件

### 事件发布时机

```java
@UseCase
public class RentBookUseCase {
    private final LoanRepository loanRepository;

    public void execute(CopyId copyId, UserId userId) {
        // 创建Loan聚合（内部注册事件）
        Loan loan = new Loan(copyId, userId, loanRepository);

        // 保存聚合时自动发布事件
        loanRepository.save(loan);
    }
}
```

**发布流程：**
1. 聚合构造或状态变更时调用`registerEvent()`
2. 事件被收集到聚合内部
3. 调用`repository.save()`时自动发布所有事件
4. 事件被路由到对应的监听器

## 领域事件的监听

### ApplicationModuleListener注解

```java
@Component
public class DomainEventListener {

    private final CopyRepository copyRepository;

    public DomainEventListener(CopyRepository copyRepository) {
        this.copyRepository = copyRepository;
    }

    @ApplicationModuleListener
    public void handle(LoanCreated event) {
        Copy copy = copyRepository.findById(
            new CopyId(event.copyId().id())
        ).orElseThrow();
        copy.makeUnavailable();
        copyRepository.save(copy);
    }

    @ApplicationModuleListener
    public void handle(LoanClosed event) {
        Copy copy = copyRepository.findById(
            new CopyId(event.copyId().id())
        ).orElseThrow();
        copy.makeAvailable();
        copyRepository.save(copy);
    }
}
```

**@ApplicationModuleListener的作用：**
- Spring Modulith提供的注解
- 自动处理跨模块的事件传递
- 提供事件路由和分发
- 支持事务边界管理

### 事件处理逻辑

#### 处理LoanCreated事件

```java
@ApplicationModuleListener
public void handle(LoanCreated event) {
    // 1. 根据事件中的副本ID查找副本
    Copy copy = copyRepository.findById(
        new CopyId(event.copyId().id())
    ).orElseThrow();

    // 2. 更新副本状态为不可借
    copy.makeUnavailable();

    // 3. 保存副本
    copyRepository.save(copy);
}
```

**业务含义：**
- 当创建借阅记录时，对应的副本变为不可借状态
- 确保一个副本同时只能被一个人借阅

#### 处理LoanClosed事件

```java
@ApplicationModuleListener
public void handle(LoanClosed event) {
    Copy copy = copyRepository.findById(
        new CopyId(event.copyId().id())
    ).orElseThrow();
    copy.makeAvailable();
    copyRepository.save(copy);
}
```

**业务含义：**
- 当借阅关闭（归还图书）时，对应的副本变为可借状态
- 允许其他用户借阅该副本

## 领域事件的优势

### 1. 解耦限界上下文

```
Lending Context                    Catalog Context
     │                                    │
     ├─ 发布 LoanCreated ─────────────────┤
     │                                    ├─ 更新Copy状态
     │                                    │
     ├─ 发布 LoanClosed ──────────────────┤
                                          ├─ 更新Copy状态
```

**好处：**
- Lending上下文不需要直接调用Catalog上下文的API
- 两个上下文可以独立开发和部署
- 降低系统耦合度

### 2. 实现最终一致性

```java
// Lending上下文：创建借阅记录
@Transactional
public void execute(CopyId copyId, UserId userId) {
    Loan loan = new Loan(copyId, userId, loanRepository);
    loanRepository.save(loan);
    // 事务提交后，事件被异步处理
}

// Catalog上下文：更新副本状态（可能稍后执行）
@Transactional
@ApplicationModuleListener
public void handle(LoanCreated event) {
    Copy copy = copyRepository.findById(new CopyId(event.copyId().id())).orElseThrow();
    copy.makeUnavailable();
    copyRepository.save(copy);
}
```

**好处：**
- 避免跨上下文的分布式事务
- 提高系统性能和可用性
- 允许异步处理

### 3. 记录业务历史

```java
// 可以保存所有事件用于审计
public class EventStore {
    public void save(DomainEvent event) {
        // 保存事件到事件存储
    }
}

// 可以重放事件来重建状态
public class LoanRebuilder {
    public Loan rebuildFromEvents(List<DomainEvent> events) {
        // 根据事件重建Loan聚合
    }
}
```

## 事件驱动架构

### 完整的事件流程

```
1. 用户发起请求
   ↓
2. 应用服务调用领域对象
   ↓
3. 领域对象执行业务逻辑
   ↓
4. 领域对象注册领域事件
   ↓
5. 聚合保存时自动发布事件
   ↓
6. 事件监听器接收事件
   ↓
7. 监听器执行相应操作
```

### 时序图

```
用户   应用服务   Loan聚合   事件总线   监听器   Copy聚合
 │        │          │          │         │         │
 ├───────>│          │          │         │         │
 │        ├───────>  │          │         │         │
 │        │          ├───────>  │         │         │  registerEvent
 │        │          │          ├───────> │         │  publish event
 │        │          │          │         ├───────> │
 │        │          │          │         │         ├─ makeUnavailable
 │        │          │          │         │         ├─ save
 │        │          │          │         │<───────┤
 │        │          │          │<───────┤         │
 │        │<───────  │          │         │         │
 │<───────┤          │          │         │         │
```

## 领域事件的最佳实践

### 1. 使用过去时态命名

```java
// 好的命名
public record LoanCreated(CopyId copyId) { }
public record LoanClosed(CopyId copyId) { }

// 不好的命名
public record CreateLoan(CopyId copyId) { }
public record CloseLoan(CopyId copyId) { }
```

### 2. 事件应该携带必要信息

```java
// 好的设计：包含足够的信息
public record LoanCreated(CopyId copyId, UserId userId, LocalDateTime createdAt) { }

// 不好的设计：信息不足
public record LoanCreated() { }
```

### 3. 事件应该是不可变的

```java
// 使用Java record保证不可变性
public record LoanCreated(CopyId copyId) { }
```

### 4. 事件处理器应该是幂等的

```java
@ApplicationModuleListener
public void handle(LoanCreated event) {
    Copy copy = copyRepository.findById(new CopyId(event.copyId().id())).orElseThrow();

    // 检查状态避免重复处理
    if (copy.isAvailable()) {
        copy.makeUnavailable();
        copyRepository.save(copy);
    }
}
```

### 5. 事件处理应该有超时和重试机制

```java
// 可以配置事件处理的超时和重试
@ApplicationModuleListener
@RetryableTopic(attempts = "3")
public void handle(LoanCreated event) {
    // 处理逻辑
}
```

## 复杂的事件处理场景

### 1. 事件链

```java
// 一个事件触发另一个事件
@ApplicationModuleListener
public void handle(LoanCreated event) {
    copy.makeUnavailable();
    copyRepository.save(copy);

    // 触发通知事件
    eventPublisher.publishEvent(new NotificationEvent("Book rented"));
}
```

### 2. 条件事件处理

```java
@ApplicationModuleListener
public void handle(LoanClosed event) {
    Copy copy = copyRepository.findById(new CopyId(event.copyId().id())).orElseThrow();
    copy.makeAvailable();
    copyRepository.save(copy);

    // 如果有等待列表，通知下一个用户
    if (copy.hasWaitingList()) {
        eventPublisher.publishEvent(new NotifyNextUserEvent(copy.getId()));
    }
}
```

### 3. 事件聚合

```java
// 聚合多个事件后处理
@ApplicationModuleListener
public void handle(LoanReturnedEvent event) {
    // 收集返回的副本
    returnedCopies.add(event.copyId());

    // 达到一定数量后批量处理
    if (returnedCopies.size() >= BATCH_SIZE) {
        processBatch(returnedCopies);
        returnedCopies.clear();
    }
}
```

## 事件存储与审计

### 事件溯源（Event Sourcing）

```java
public class EventStore {
    public void save(LoanId aggregateId, List<DomainEvent> events) {
        for (DomainEvent event : events) {
            eventRepository.save(new EventRecord(
                aggregateId,
                event.getClass().getSimpleName(),
                event,
                LocalDateTime.now()
            ));
        }
    }
}
```

### 事件回放

```java
public Loan rebuildFromEvents(LoanId loanId) {
    List<DomainEvent> events = eventStore.findByAggregateId(loanId);
    Loan loan = null;

    for (DomainEvent event : events) {
        if (event instanceof LoanCreated created) {
            loan = new Loan(created.copyId(), created.userId(), loanRepository);
        } else if (event instanceof LoanClosed && loan != null) {
            loan.returned();
        }
    }

    return loan;
}
```

## 领域事件在项目中的应用总结

1. **事件定义**：使用Java record定义不可变事件
2. **事件发布**：通过AbstractAggregateRoot在聚合中注册
3. **事件监听**：使用@ApplicationModuleListener注解
4. **跨上下文通信**：Lending → Catalog的事件驱动集成
5. **最终一致性**：避免分布式事务，提高性能
6. **业务解耦**：限界上下文之间松耦合

领域事件是实现DDD中限界上下文协作的重要机制，本项目展示了如何使用Spring Modulith优雅地实现事件驱动架构。
