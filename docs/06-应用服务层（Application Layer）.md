# 应用服务层（Application Layer）

## 应用服务层的概念

应用服务层是DDD架构中的重要组成部分，它负责协调领域对象执行用例，管理事务边界，并处理横切关注点。应用服务不包含业务逻辑，只负责编排。

### 应用服务层的职责

1. **用例编排**：协调领域对象完成业务用例
2. **事务管理**：定义事务边界
3. **输入验证**：验证外部输入
4. **安全控制**：执行权限检查
5. **日志记录**：记录用例执行过程
6. **外部系统集成**：调用外部API

### 应用服务层 vs 领域层

| 特性 | 应用服务层 | 领域层 |
|------|-----------|--------|
| 职责 | 用例编排 | 业务逻辑 |
| 事务 | 管理事务边界 | 不关心事务 |
| 状态 | 无状态 | 有状态 |
| 依赖 | 可以依赖基础设施层 | 纯领域逻辑 |
| 复用性 | 针对特定用例 | 可复用的业务规则 |

## 项目中的应用服务

### 1. AddBookToCatalogUseCase（添加书籍到目录）

```java
package library.catalog.application;

import library.UseCase;
import library.catalog.domain.Book;
import library.catalog.domain.BookRepository;
import library.catalog.domain.Isbn;

@UseCase
public class AddBookToCatalogUseCase {
    private final BookSearchService bookSearchService;
    private final BookRepository bookRepository;

    public AddBookToCatalogUseCase(BookSearchService bookSearchService, BookRepository bookRepository) {
        this.bookSearchService = bookSearchService;
        this.bookRepository = bookRepository;
    }

    public void execute(Isbn isbn) {
        // 1. 调用外部服务获取书籍信息
        BookInformation result = bookSearchService.search(isbn);

        // 2. 创建领域对象
        Book book = new Book(result.title(), isbn);

        // 3. 持久化
        bookRepository.save(book);
    }
}
```

**用例说明：**
- 根据ISBN获取书籍信息并添加到目录
- 调用外部API获取书籍元数据
- 创建Book聚合并保存

**执行流程：**
1. 接收ISBN作为输入
2. 调用`BookSearchService`从OpenLibrary获取书籍信息
3. 创建`Book`领域对象
4. 通过`BookRepository`保存书籍

### 2. RegisterBookCopyUseCase（注册图书副本）

```java
package library.catalog.application;

import jakarta.validation.constraints.NotNull;
import library.UseCase;
import library.catalog.domain.BarCode;
import library.catalog.domain.BookId;
import library.catalog.domain.Copy;
import library.catalog.domain.CopyRepository;

@UseCase
public class RegisterBookCopyUseCase {
    private final CopyRepository copyRepository;

    public RegisterBookCopyUseCase(CopyRepository copyRepository) {
        this.copyRepository = copyRepository;
    }

    public void execute(@NotNull BookId bookId, @NotNull BarCode barCode) {
        copyRepository.save(new Copy(bookId, barCode));
    }
}
```

**用例说明：**
- 为指定的书籍注册一个物理副本
- 验证输入参数不为null
- 创建Copy聚合并保存

**验证注解：**
- `@NotNull`：使用Spring Validation验证参数

### 3. RentBookUseCase（借书）

```java
package library.lending.application;

import library.UseCase;
import library.lending.domain.CopyId;
import library.lending.domain.Loan;
import library.lending.domain.LoanRepository;
import library.lending.domain.UserId;

@UseCase
public class RentBookUseCase {
    private final LoanRepository loanRepository;

    public RentBookUseCase(LoanRepository loanRepository) {
        this.loanRepository = loanRepository;
    }

    public void execute(CopyId copyId, UserId userId) {
        // TODO: ensure rented copy is not rented again
        loanRepository.save(new Loan(copyId, userId, loanRepository));
    }
}
```

**用例说明：**
- 用户借阅指定的图书副本
- 创建Loan聚合
- 聚合内部验证副本是否可借
- 保存时自动发布`LoanCreated`事件

**业务逻辑在领域层：**
```java
public class Loan extends AbstractAggregateRoot<Loan> {
    public Loan(CopyId copyId, UserId userId, LoanRepository loanRepository) {
        Assert.notNull(copyId, "copyId must not be null");
        Assert.notNull(userId, "userId must not be null");

        // 业务规则验证：检查副本是否可借
        Assert.isTrue(loanRepository.isAvailable(copyId),
            "copy with id = " + copyId + " is not available");

        this.loanId = new LoanId();
        this.copyId = copyId;
        this.userId = userId;
        this.createdAt = LocalDateTime.now();
        this.expectedReturnDate = LocalDate.now().plusDays(30);

        // 发布领域事件
        this.registerEvent(new LoanCreated(this.copyId));
    }
}
```

### 4. ReturnBookUseCase（还书）

```java
package library.lending.application;

import library.UseCase;
import library.lending.domain.Loan;
import library.lending.domain.LoanId;
import library.lending.domain.LoanRepository;

@UseCase
public class ReturnBookUseCase {
    private final LoanRepository loanRepository;

    public ReturnBookUseCase(LoanRepository loanRepository) {
        this.loanRepository = loanRepository;
    }

    public void execute(LoanId loanId) {
        Loan loan = loanRepository.findByIdOrThrow(loanId);
        loan.returned();
    }
}
```

**用例说明：**
- 用户归还借阅的图书
- 根据LoanId查找借阅记录
- 调用领域对象的`returned()`方法
- 方法内部发布`LoanClosed`事件

**业务逻辑在领域层：**
```java
public class Loan extends AbstractAggregateRoot<Loan> {
    public void returned() {
        this.returnedAt = LocalDateTime.now();

        // 检查是否逾期
        if (this.returnedAt.isAfter(expectedReturnDate.atStartOfDay())) {
            // 计算逾期费用（业务逻辑）
        }

        // 发布领域事件
        this.registerEvent(new LoanClosed(this.copyId));
    }
}
```

## @UseCase注解

### 注解定义

```java
package library;

import org.springframework.stereotype.Service;
import org.springframework.validation.annotation.Validated;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Documented
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Service
@Validated
public @interface UseCase {
}
```

**注解作用：**
1. `@Service`：标记为Spring服务组件
2. `@Validated`：启用方法参数验证
3. 组合注解，简化配置

### 使用效果

```java
@UseCase
public class RentBookUseCase {
    public void execute(CopyId copyId, UserId userId) {
        // 方法参数自动验证
    }
}
```

**验证注解：**
```java
@UseCase
public class RegisterBookCopyUseCase {
    public void execute(@NotNull BookId bookId, @NotNull BarCode barCode) {
        // @NotNull验证生效
    }
}
```

## 横切关注点：AOP日志切面

### UseCaseLoggingAdvice

```java
package library;

import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Pointcut;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;
import org.springframework.util.StopWatch;

import java.util.Arrays;

@Component
@Aspect
@Order(1)
public class UseCaseLoggingAdvice {
    private static final Logger LOGGER = LoggerFactory.getLogger(UseCaseLoggingAdvice.class);

    @Pointcut("within(@library.UseCase *)")
    public void useCase() {
    }

    @Pointcut("execution(public * *(..))")
    public void publicMethod() {
    }

    @Pointcut("publicMethod() && useCase()")
    public void publicMethodInsideAUseCase() {
    }

    @Around("publicMethodInsideAUseCase()")
    public Object aroundServiceMethodAdvice(final ProceedingJoinPoint pjp) throws Throwable {
        StopWatch stopWatch = new StopWatch();
        try {
            LOGGER.info("Executing use case: {}#{} with parameters: {}",
                pjp.getTarget().getClass(),
                pjp.getSignature().getName(),
                Arrays.toString(pjp.getArgs()));
            stopWatch.start();
            return pjp.proceed();
        } finally {
            stopWatch.stop();
            LOGGER.info("Finished executing use case {}#{} in {}ms",
                pjp.getTarget().getClass(),
                pjp.getSignature().getName(),
                stopWatch.getTotalTimeMillis());
        }
    }
}
```

**切面功能：**
1. **切入点定义**：拦截所有`@UseCase`注解的类的public方法
2. **前置通知**：记录用例开始执行，包含类名、方法名、参数
3. **性能监控**：使用`StopWatch`测量执行时间
4. **后置通知**：记录用例执行完成和耗时

**执行顺序：**
- `@Order(1)`：设置切面执行顺序
- 确保在验证之前或之后执行

**日志输出示例：**
```
Executing use case: class library.lending.application.RentBookUseCase#execute with parameters: [CopyId[id=...], UserId[id=...]]
Finished executing use case class library.lending.application.RentBookUseCase#execute in 45ms
```

## 应用服务设计原则

### 1. 应用服务应该是无状态的

```java
// 好的设计：无状态服务
@UseCase
public class RentBookUseCase {
    private final LoanRepository loanRepository;

    public RentBookUseCase(LoanRepository loanRepository) {
        this.loanRepository = loanRepository;
    }

    public void execute(CopyId copyId, UserId userId) {
        // 不保存状态
        loanRepository.save(new Loan(copyId, userId, loanRepository));
    }
}

// 不好的设计：有状态服务
@UseCase
public class RentBookUseCase {
    private Loan currentLoan;  // 不要这样做

    public void execute(CopyId copyId, UserId userId) {
        this.currentLoan = new Loan(copyId, userId, loanRepository);
    }
}
```

### 2. 应用服务不包含业务逻辑

```java
// 好的设计：业务逻辑在领域层
@UseCase
public class RentBookUseCase {
    public void execute(CopyId copyId, UserId userId) {
        // 只负责编排
        loanRepository.save(new Loan(copyId, userId, loanRepository));
    }
}

// 不好的设计：业务逻辑在应用层
@UseCase
public class RentBookUseCase {
    public void execute(CopyId copyId, UserId userId) {
        // 业务逻辑应该在领域层
        if (!loanRepository.isAvailable(copyId)) {
            throw new IllegalStateException("Copy not available");
        }
        Loan loan = new Loan(copyId, userId);
        loanRepository.save(loan);
    }
}
```

### 3. 应用服务管理事务边界

```java
@UseCase
@Transactional
public class RentBookUseCase {
    public void execute(CopyId copyId, UserId userId) {
        // 整个方法在一个事务中执行
        loanRepository.save(new Loan(copyId, userId, loanRepository));
        // 其他数据库操作
    }
}
```

### 4. 应用服务处理输入验证

```java
@UseCase
public class RegisterBookCopyUseCase {
    public void execute(@NotNull BookId bookId, @NotNull BarCode barCode) {
        // 参数自动验证
        copyRepository.save(new Copy(bookId, barCode));
    }
}
```

### 5. 应用服务处理异常

```java
@UseCase
public class AddBookToCatalogUseCase {
    public void execute(Isbn isbn) {
        try {
            BookInformation result = bookSearchService.search(isbn);
            Book book = new Book(result.title(), isbn);
            bookRepository.save(book);
        } catch (BookNotFoundException e) {
            throw new BusinessException("Book not found", e);
        }
    }
}
```

## 应用服务与领域事件

### 应用服务不直接处理事件

```java
@UseCase
public class RentBookUseCase {
    public void execute(CopyId copyId, UserId userId) {
        // 应用服务不处理事件
        // 事件由领域对象发布，由事件监听器处理
        loanRepository.save(new Loan(copyId, userId, loanRepository));
    }
}

// 领域对象发布事件
public class Loan extends AbstractAggregateRoot<Loan> {
    public Loan(CopyId copyId, UserId userId, LoanRepository loanRepository) {
        // ...
        this.registerEvent(new LoanCreated(this.copyId));
    }
}

// 事件监听器处理事件
@Component
public class DomainEventListener {
    @ApplicationModuleListener
    public void handle(LoanCreated event) {
        // 处理事件
    }
}
```

## 复杂用例的编排

### 多步骤用例

```java
@UseCase
public class PurchaseBookUseCase {
    private final BookSearchService bookSearchService;
    private final BookRepository bookRepository;
    private final CopyRepository copyRepository;

    public BookId execute(Isbn isbn, int copyCount) {
        // 步骤1：获取书籍信息
        BookInformation info = bookSearchService.search(isbn);

        // 步骤2：创建书籍
        Book book = new Book(info.title(), isbn);
        bookRepository.save(book);

        // 步骤3：创建多个副本
        for (int i = 0; i < copyCount; i++) {
            BarCode barCode = generateBarCode(book.getId(), i);
            Copy copy = new Copy(book.getId(), barCode);
            copyRepository.save(copy);
        }

        return book.getId();
    }
}
```

### 条件执行

```java
@UseCase
public class RenewLoanUseCase {
    public void execute(LoanId loanId) {
        Loan loan = loanRepository.findByIdOrThrow(loanId);

        // 业务条件检查
        if (loan.isOverdue()) {
            throw new BusinessException("Cannot renew overdue loan");
        }

        if (loan.getRenewalCount() >= MAX_RENEWALS) {
            throw new BusinessException("Max renewals reached");
        }

        // 执行续借
        loan.renew();
    }
}
```

### 调用多个聚合

```java
@UseCase
public class ProcessBookReturnUseCase {
    private final LoanRepository loanRepository;
    private final UserRepository userRepository;
    private final NotificationService notificationService;

    public void execute(LoanId loanId) {
        // 操作Loan聚合
        Loan loan = loanRepository.findByIdOrThrow(loanId);
        loan.returned();

        // 操作User聚合（如果需要）
        User user = userRepository.findById(loan.getUserId()).orElseThrow();
        user.updateReturnHistory(loan);

        // 发送通知（基础设施层）
        notificationService.sendReturnConfirmation(user.getEmail());
    }
}
```

## 应用服务层的测试

### 集成测试

```java
@SpringBootTest
@Transactional
class RentBookUseCaseTest {

    @Autowired
    private RentBookUseCase rentBookUseCase;

    @Autowired
    private LoanRepository loanRepository;

    @Test
    void shouldRentBook() {
        CopyId copyId = new CopyId();
        UserId userId = new UserId();

        // 执行用例
        rentBookUseCase.execute(copyId, userId);

        // 验证结果
        Loan loan = loanRepository.findAll().iterator().next();
        assertEquals(copyId, loan.getCopyId());
        assertEquals(userId, loan.getUserId());
        assertNotNull(loan.getCreatedAt());
    }
}
```

### 单元测试

```java
@ExtendWith(MockitoExtension.class)
class RentBookUseCaseTest {

    @Mock
    private LoanRepository loanRepository;

    @InjectMocks
    private RentBookUseCase rentBookUseCase;

    @Test
    void shouldRentBook() {
        CopyId copyId = new CopyId();
        UserId userId = new UserId();

        when(loanRepository.isAvailable(copyId)).thenReturn(true);

        rentBookUseCase.execute(copyId, userId);

        verify(loanRepository).save(any(Loan.class));
    }
}
```

## 应用服务层在项目中的应用总结

1. **用例实现**：每个用例对应一个应用服务类
2. **编排协调**：协调领域对象完成业务用例
3. **事务管理**：定义事务边界
4. **输入验证**：使用Spring Validation
5. **横切关注点**：通过AOP实现日志等功能
6. **无状态设计**：服务不保存状态，便于扩展
7. **不包含业务逻辑**：业务逻辑封装在领域层

应用服务层是连接外部世界和领域模型的桥梁，本项目展示了如何设计清晰、可维护的应用服务。
