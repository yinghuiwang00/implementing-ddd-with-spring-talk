# 仓储模式（Repository Pattern）

## 仓储模式的概念

仓储模式是DDD中用于封装数据访问的设计模式。它提供了一种类似集合的接口来访问领域对象，将领域逻辑与持久化细节分离。

### 仓储模式的作用

1. **抽象持久化细节**：领域层不需要知道数据如何存储
2. **提供类似集合的接口**：模拟内存集合的操作方式
3. **集中查询逻辑**：将复杂的查询逻辑封装在仓储中
4. **提高可测试性**：便于在测试中使用模拟对象

### 仓储 vs DAO

| 特性 | 仓储（Repository） | DAO（Data Access Object） |
|------|-------------------|---------------------------|
| 关注点 | 领域对象的持久化 | 数据表的CRUD操作 |
| 抽象级别 | 领域层抽象 | 数据访问层抽象 |
| 操作语义 | 业务语义（如保存、删除） | 技术语义（如增删改查） |
| 返回类型 | 领域对象 | 数据传输对象或实体 |

## 项目中的仓储实现

### 1. BookRepository（编目上下文）

```java
package library.catalog.domain;

import org.springframework.data.repository.CrudRepository;

public interface BookRepository extends CrudRepository<Book, BookId> {
}
```

**特点：**
- 继承Spring Data JPA的`CrudRepository`
- 直接操作领域对象`Book`
- 使用值对象`BookId`作为主键类型
- 默认提供基本的CRUD操作

### 2. CopyRepository（编目上下文）

```java
package library.catalog.domain;

import org.springframework.data.repository.CrudRepository;

public interface CopyRepository extends CrudRepository<Copy, CopyId> {
}
```

**特点：**
- 简洁的仓储接口
- 只继承基础CRUD操作
- 针对特定领域对象`Copy`

### 3. LoanRepository（借阅上下文）

```java
package library.lending.domain;

import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.CrudRepository;

public interface LoanRepository extends CrudRepository<Loan, LoanId> {
    @Query("select count(*) = 0 from Loan where copyId = :id and returnedAt is null")
    boolean isAvailable(CopyId id);

    default Loan findByIdOrThrow(LoanId loanId) {
        return findById(loanId).orElseThrow();
    }
}
```

**特点：**
- 继承基础CRUD操作
- 自定义查询方法`isAvailable()`
- 使用`@Query`注解定义查询逻辑
- 提供默认方法`findByIdOrThrow()`处理可选值

## 仓储设计原则

### 1. 仓储接口属于领域层

```java
// 正确：仓储接口定义在领域层
package library.catalog.domain;

public interface BookRepository extends CrudRepository<Book, BookId> {
}
```

**原因：**
- 仓储是领域模型的组成部分
- 领域层需要通过仓储来访问持久化的领域对象
- 保持领域层的完整性

### 2. 仓储实现属于基础设施层

```java
// Spring Data JPA自动提供实现
// 开发者不需要编写实现类
```

**Spring Data JPA的优势：**
- 自动生成实现类
- 支持方法名查询
- 支持自定义查询
- 支持分页、排序等功能

### 3. 仓储操作聚合根

```java
// 仓储只操作聚合根，不操作聚合内部的实体
public interface CopyRepository extends CrudRepository<Copy, CopyId> {
}

// 不要这样做：
// public interface BookCopyRepository extends CrudRepository<BookCopy, BookCopyId> {
// }
```

**原因：**
- 聚合根是唯一的外部访问点
- 保证聚合的一致性
- 简化事务管理

## 仓储中的查询逻辑

### 1. 自定义查询方法

```java
public interface LoanRepository extends CrudRepository<Loan, LoanId> {
    @Query("select count(*) = 0 from Loan where copyId = :id and returnedAt is null")
    boolean isAvailable(CopyId id);
}
```

**业务逻辑：**
- 检查指定副本当前是否可借
- 查询条件：副本ID相同且未归还的借阅记录
- 返回布尔值：true表示可借，false表示不可借

### 2. 默认方法

```java
default Loan findByIdOrThrow(LoanId loanId) {
    return findById(loanId).orElseThrow();
}
```

**作用：**
- 简化调用代码
- 将常见的错误处理逻辑封装在仓储中
- 提供更友好的API

### 3. 使用示例

```java
@UseCase
public class RentBookUseCase {
    private final LoanRepository loanRepository;

    public void execute(CopyId copyId, UserId userId) {
        // 在聚合构造函数中使用仓储验证业务规则
        loanRepository.save(new Loan(copyId, userId, loanRepository));
    }
}

// Loan聚合根中使用仓储
public class Loan extends AbstractAggregateRoot<Loan> {
    public Loan(CopyId copyId, UserId userId, LoanRepository loanRepository) {
        Assert.notNull(copyId, "copyId must not be null");
        Assert.notNull(userId, "userId must not be null");

        // 使用仓储验证业务规则
        Assert.isTrue(loanRepository.isAvailable(copyId),
            "copy with id = " + copyId + " is not available");

        // ... 其他初始化逻辑
    }
}
```

## 仓储与聚合的关系

### 一次事务一个聚合

```java
// 正确：在一个事务中操作一个聚合
@Transactional
public void execute(CopyId copyId, UserId userId) {
    Loan loan = new Loan(copyId, userId, loanRepository);
    loanRepository.save(loan);  // 保存Loan聚合
}

// 错误：在一个事务中操作多个聚合
@Transactional
public void execute(CopyId copyId, UserId userId) {
    Loan loan = new Loan(copyId, userId, loanRepository);
    loanRepository.save(loan);

    Copy copy = copyRepository.findById(copyId).orElseThrow();
    copy.makeUnavailable();  // 不应该直接操作其他聚合
    copyRepository.save(copy);
}
```

### 通过事件协调聚合

```java
// 正确：通过领域事件协调聚合
public class Loan extends AbstractAggregateRoot<Loan> {
    public Loan(CopyId copyId, UserId userId, LoanRepository loanRepository) {
        // ...
        this.registerEvent(new LoanCreated(this.copyId));
    }
}

// 另一个上下文监听事件并更新自己的聚合
@Component
public class DomainEventListener {
    @ApplicationModuleListener
    public void handle(LoanCreated event) {
        Copy copy = copyRepository.findById(new CopyId(event.copyId().id())).orElseThrow();
        copy.makeUnavailable();
        copyRepository.save(copy);  // 在另一个事务中执行
    }
}
```

## 仓储模式的实现技巧

### 1. 使用值对象作为ID

```java
public interface BookRepository extends CrudRepository<Book, BookId> {
}

// 好处：
// - 类型安全
// - 避免ID混淆
// - 表达业务语义
```

### 2. 仓储返回领域对象

```java
public interface BookRepository extends CrudRepository<Book, BookId> {
    Book findById(BookId id);  // 返回领域对象
}

// 不要返回DTO或实体
```

### 3. 封装复杂查询

```java
public interface LoanRepository extends CrudRepository<Loan, LoanId> {
    @Query("select count(*) = 0 from Loan where copyId = :id and returnedAt is null")
    boolean isAvailable(CopyId id);

    // 可以添加更多业务查询方法
    List<Loan> findByUserIdAndReturnedAtIsNull(UserId userId);
    List<Loan> findByReturnedAtAfter(LocalDateTime date);
}
```

### 4. 避免在领域层直接使用JPA

```java
// 错误：在领域对象中直接使用JPA注解和API
@Entity
public class Book {
    @PersistenceContext
    private EntityManager entityManager;  // 不要这样做

    public void someMethod() {
        entityManager.flush();  // 不要在领域对象中使用JPA
    }
}

// 正确：在应用服务层处理持久化
@UseCase
public class AddBookToCatalogUseCase {
    public void execute(Isbn isbn) {
        BookInformation result = bookSearchService.search(isbn);
        Book book = new Book(result.title(), isbn);
        bookRepository.save(book);  // 持久化在应用层处理
    }
}
```

## 仓储的测试策略

### 1. 集成测试

```java
@SpringBootTest
@Transactional
class LoanRepositoryTest {

    @Autowired
    private LoanRepository loanRepository;

    @Test
    void shouldCheckAvailability() {
        CopyId copyId = new CopyId();
        UserId userId = new UserId();

        // 创建借阅记录
        Loan loan = new Loan(copyId, userId, loanRepository);
        loanRepository.save(loan);

        // 验证不可借
        assertFalse(loanRepository.isAvailable(copyId));
    }
}
```

### 2. 使用测试数据

```java
@SpringBootTest
@Transactional
class CopyRepositoryTest {

    @Autowired
    private CopyRepository copyRepository;

    @Test
    void shouldFindCopyById() {
        BookId bookId = new BookId();
        BarCode barCode = new BarCode("123456");
        Copy copy = new Copy(bookId, barCode);

        copyRepository.save(copy);

        Copy found = copyRepository.findById(copy.getId()).orElseThrow();
        assertEquals(bookId, found.getBookId());
        assertEquals(barCode, found.getBarCode());
    }
}
```

## 仓储模式的常见问题

### 1. 仓储应该返回集合还是流？

```java
// 返回集合：适用于小数据量
List<Loan> findByUserId(UserId userId);

// 返回流：适用于大数据量，避免内存溢出
Stream<Loan> streamByUserId(UserId userId);
```

### 2. 仓储应该处理分页吗？

```java
// 是的，Spring Data JPA支持分页
Page<Loan> findByUserId(UserId userId, Pageable pageable);

// 使用示例
Page<Loan> loans = loanRepository.findByUserId(userId, PageRequest.of(0, 10));
```

### 3. 仓储应该处理缓存吗？

```java
// 可以使用Spring Cache抽象
@Cacheable("books")
Book findById(BookId id);

@CacheEvict(value = "books", key = "#book.id")
Book save(Book book);
```

## 仓储模式在项目中的应用总结

1. **接口定义在领域层**：保持领域层的纯净性
2. **实现由Spring Data JPA提供**：减少样板代码
3. **操作聚合根**：保证聚合的一致性
4. **封装查询逻辑**：将业务查询集中管理
5. **使用值对象**：提高类型安全性和表达力
6. **支持自定义查询**：满足复杂业务需求

仓储模式是DDD中连接领域模型和持久化层的重要桥梁，本项目展示了如何在Spring框架中优雅地实现这一模式。
