# 领域建模与Bounded Context

## Bounded Context（限界上下文）的概念

限界上下文是DDD中最核心的战略性设计概念。它将一个复杂的软件系统分解为多个独立的、内聚的领域模型，每个上下文都有自己明确的边界和语言。

### 为什么需要限界上下文？

在大型系统中，不同的业务领域对相同的概念可能有不同的理解：

- **Customer**：在销售部门，Customer是"购买者"；在客服部门，Customer是"需要帮助的人"
- **Order**：在物流部门，Order是"配送任务"；在财务部门，Order是"收款记录"

如果强行统一这些概念，会导致模型复杂度急剧上升，难以维护。

## 本项目的限界上下文划分

### 1. Catalog Context（编目上下文）

**职责：** 管理图书馆的书籍信息和物理副本

**核心概念：**
- Book（书籍）：书籍的基本信息
- Copy（副本）：书籍的物理实例
- ISBN（国际标准书号）：书籍的唯一标识
- BarCode（条形码）：物理副本的标识

**业务规则：**
- 书籍可以有多个副本
- 每个副本有唯一的条形码
- 副本状态：可借阅/不可借阅

```java
// 编目上下文的领域模型
package library.catalog.domain;

@Entity
public class Book {
    @EmbeddedId
    private BookId id;
    private String title;
    @Embedded
    @AttributeOverride(name = "value", column = @Column(name = "isbn"))
    private Isbn isbn;
    // ...
}

@Entity
public class Copy {
    @EmbeddedId
    private CopyId id;
    @Embedded
    @AttributeOverride(name = "id", column = @Column(name = "book_id"))
    private BookId bookId;
    @Embedded
    private BarCode barCode;
    private boolean available;
    // ...
}
```

### 2. Lending Context（借阅上下文）

**职责：** 管理借阅流程和借阅记录

**核心概念：**
- Loan（借阅）：借阅记录
- User（用户）：借阅者
- CopyId（副本ID）：指向编目上下文的副本

**业务规则：**
- 一个副本在同一时间只能被一个人借阅
- 借阅期限为30天
- 逾期归还需要计算费用

```java
// 借阅上下文的领域模型
package library.lending.domain;

@Entity
public class Loan extends AbstractAggregateRoot<Loan> {
    @EmbeddedId
    private LoanId loanId;
    @Embedded
    @AttributeOverride(name = "id", column = @Column(name = "copy_id"))
    private CopyId copyId;
    @Embedded
    @AttributeOverride(name = "id", column = @Column(name = "user_id"))
    private UserId userId;
    private LocalDateTime createdAt;
    private LocalDate expectedReturnDate;
    private LocalDateTime returnedAt;

    public Loan(CopyId copyId, UserId userId, LoanRepository loanRepository) {
        // 验证副本是否可借
        Assert.isTrue(loanRepository.isAvailable(copyId),
            "copy with id = " + copyId + " is not available");
        this.loanId = new LoanId();
        this.copyId = copyId;
        this.userId = userId;
        this.createdAt = LocalDateTime.now();
        this.expectedReturnDate = LocalDate.now().plusDays(30);
        this.registerEvent(new LoanCreated(this.copyId));
    }

    public void returned() {
        this.returnedAt = LocalDateTime.now();
        if (this.returnedAt.isAfter(expectedReturnDate.atStartOfDay())) {
            // 计算逾期费用
        }
        this.registerEvent(new LoanClosed(this.copyId));
    }
}
```

## 上下文映射（Context Mapping）

### 上下文之间的关系

在本项目中，两个上下文存在**下游-上游关系**：

```
┌──────────────────┐         ┌──────────────────┐
│   Lending        │────────>│   Catalog        │
│   (下游)         │ 事件驱动 │   (上游)         │
│                  │         │                  │
│  - LoanCreated   │────────>│  - 更新Copy状态  │
│  - LoanClosed    │────────>│  - 更新Copy状态  │
└──────────────────┘         └──────────────────┘
```

**关系类型：** Customer/Supplier（客户/供应商）
- Lending是Catalog的客户
- Catalog是Lending的供应商
- 通信方式：领域事件

### 事件驱动的上下文通信

```java
// Lending上下文发布事件
public class Loan extends AbstractAggregateRoot<Loan> {
    public Loan(CopyId copyId, UserId userId, LoanRepository loanRepository) {
        // ... 验证和初始化
        this.registerEvent(new LoanCreated(this.copyId));  // 发布事件
    }

    public void returned() {
        // ... 更新状态
        this.registerEvent(new LoanClosed(this.copyId));   // 发布事件
    }
}

// Catalog上下文监听事件
@Component
public class DomainEventListener {
    @ApplicationModuleListener
    public void handle(LoanCreated event) {
        Copy copy = copyRepository.findById(
            new CopyId(event.copyId().id())
        ).orElseThrow();
        copy.makeUnavailable();  // 标记副本不可借
        copyRepository.save(copy);
    }

    @ApplicationModuleListener
    public void handle(LoanClosed event) {
        Copy copy = copyRepository.findById(
            new CopyId(event.copyId().id())
        ).orElseThrow();
        copy.makeAvailable();  // 标记副本可借
        copyRepository.save(copy);
    }
}
```

## 模块化：Spring Modulith的应用

### 依赖倒置原则

两个上下文之间的依赖关系遵循依赖倒置原则：

```
Lending Domain
    ↓ 依赖
Catalog Domain (只有ID的引用)
```

```java
// Lending上下文只引用CopyId，不引用Copy实体
package library.lending.domain;

public record CopyId(UUID id) {
    public CopyId {
        Assert.notNull(id, "id must not be null");
    }

    public CopyId() {
        this(UUID.randomUUID());
    }
}

// Lending聚合根只使用CopyId
public class Loan {
    @Embedded
    @AttributeOverride(name = "id", column = @Column(name = "copy_id"))
    private CopyId copyId;  // 只引用ID，不引用完整对象
}
```

### Spring Modulith模块验证

```java
@SpringBootTest
class LibraryApplicationTests {

    @Test
    void verifyModules() {
        // 验证模块间的依赖关系是否符合预期
        ApplicationModules.of(LibraryApplication.class).verify();
    }
}
```

**验证内容：**
- 模块边界是否清晰
- 依赖方向是否正确
- 是否存在非法的跨模块访问

## 通用语言在上下文中的体现

### Catalog上下文的通用语言

| 业务术语 | 代码表示 |
|---------|---------|
| 书籍 | `Book` |
| 副本 | `Copy` |
| ISBN编号 | `Isbn` |
| 条形码 | `BarCode` |
| 添加书籍到目录 | `AddBookToCatalogUseCase` |
| 注册图书副本 | `RegisterBookCopyUseCase` |

### Lending上下文的通用语言

| 业务术语 | 代码表示 |
|---------|---------|
| 借阅 | `Loan` |
| 借书 | `RentBookUseCase` |
| 还书 | `ReturnBookUseCase` |
| 借阅记录ID | `LoanId` |
| 用户ID | `UserId` |
| 借阅创建事件 | `LoanCreated` |
| 借阅关闭事件 | `LoanClosed` |

## 限界上下文的好处

### 1. 降低复杂度
每个上下文可以独立理解和修改，不需要考虑全局影响。

### 2. 提高可维护性
修改一个上下文的逻辑不会影响其他上下文。

### 3. 支持团队协作
不同团队可以负责不同的上下文，减少协调成本。

### 4. 便于演进
上下文可以独立演进，采用不同的技术栈和架构模式。

### 5. 测试友好
每个上下文可以独立测试，测试边界清晰。

## 实践建议

### 1. 从业务边界开始识别上下文
- 与业务专家沟通
- 识别业务流程中的不同领域
- 分析组织结构（不同部门可能对应不同上下文）

### 2. 保持上下文的小而专注
- 避免创建过于庞大的上下文
- 每个上下文应该有明确的职责

### 3. 定义清晰的上下文边界
- 通过包结构体现边界
- 使用模块化框架进行验证
- 明确上下文间的通信方式

### 4. 选择合适的上下文映射关系
- Customer/Supplier：事件驱动
- Anticorruption Layer：防腐层
- Shared Kernel：共享内核
- Conformist：遵奉者

### 5. 持续演进
- 上下文边界不是一成不变的
- 随着业务发展，可能需要重新划分
- 保持灵活性，允许调整

这个项目展示了如何在一个简单的图书馆管理系统中应用限界上下文的概念，为理解DDD的战略性设计提供了清晰的示例。
