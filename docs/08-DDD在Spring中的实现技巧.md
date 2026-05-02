# DDD在Spring中的实现技巧

## Spring框架与DDD的结合

Spring框架为DDD的实现提供了强大的支持，包括依赖注入、AOP、事务管理、数据访问等特性。本章将介绍如何在Spring框架中优雅地实现DDD的各个核心概念。

## 1. Spring Modulith：模块化架构验证

### 什么是Spring Modulith？

Spring Modulith是Spring生态系统中的一个项目，用于支持模块化单体应用程序的开发。它提供了模块边界验证、依赖分析、文档生成等功能。

### 项目配置

```gradle
dependencies {
    implementation 'org.springframework.modulith:spring-modulith-starter-core'
    implementation 'org.springframework.modulith:spring-modulith-starter-jpa'
    testImplementation 'org.springframework.modulith:spring-modulith-starter-test'
}
```

### 模块验证

```java
@SpringBootTest
class LibraryApplicationTests {

    @Test
    void verifyModules() {
        // 验证模块间的依赖关系
        ApplicationModules.of(LibraryApplication.class).verify();
    }
}
```

**验证内容：**
1. 模块边界是否清晰
2. 依赖方向是否正确
3. 是否存在非法的跨模块访问
4. 循环依赖检测

### 模块结构

```
library/
├── catalog/              # catalog模块
│   ├── application/      # 应用层
│   ├── domain/           # 领域层
│   └── infrastructure/   # 基础设施层
└── lending/              # lending模块
    ├── application/
    └── domain/
```

**自动模块发现：**
- Spring Modulith自动根据包结构识别模块
- 默认规则：每个顶级包作为一个模块
- 可通过注解自定义模块

### 事件发布与监听

```java
// 发布事件
@Entity
public class Loan extends AbstractAggregateRoot<Loan> {
    public Loan(CopyId copyId, UserId userId, LoanRepository loanRepository) {
        // ...
        this.registerEvent(new LoanCreated(this.copyId));
    }
}

// 监听事件
@Component
public class DomainEventListener {
    @ApplicationModuleListener
    public void handle(LoanCreated event) {
        Copy copy = copyRepository.findById(new CopyId(event.copyId().id())).orElseThrow();
        copy.makeUnavailable();
        copyRepository.save(copy);
    }
}
```

**事件特性：**
1. 自动发现`@ApplicationModuleListener`注解的方法
2. 支持跨模块事件传递
3. 事务感知的事件发布
4. 事件文档生成

## 2. Spring Data JPA：仓储模式实现

### 仓储接口定义

```java
package library.catalog.domain;

import org.springframework.data.repository.CrudRepository;

public interface BookRepository extends CrudRepository<Book, BookId> {
}
```

**Spring Data JPA的优势：**
1. 自动生成实现类
2. 支持方法名查询
3. 支持自定义查询
4. 支持分页、排序

### 自定义查询方法

```java
public interface LoanRepository extends CrudRepository<Loan, LoanId> {
    @Query("select count(*) = 0 from Loan where copyId = :id and returnedAt is null")
    boolean isAvailable(CopyId id);

    List<Loan> findByUserIdAndReturnedAtIsNull(UserId userId);

    Page<Loan> findByUserId(UserId userId, Pageable pageable);

    @Query("SELECT l FROM Loan l WHERE l.returnedAt > :date")
    List<Loan> findOverdueLoans(@Param("date") LocalDateTime date);
}
```

**查询方法命名规则：**
- `findBy` + 属性名：精确查询
- `countBy` + 属性名：计数查询
- `existsBy` + 属性名：存在性查询
- `deleteBy` + 属性名：删除查询

### 值对象作为主键

```java
@Entity
public class Book {
    @EmbeddedId
    private BookId id;
    // ...
}

@Embeddable
public class BookId {
    @Column(name = "id")
    private UUID id;

    public BookId() {
        this.id = UUID.randomUUID();
    }

    public BookId(UUID id) {
        this.id = id;
    }

    public UUID id() {
        return id;
    }
}
```

**JPA映射技巧：**
1. 使用`@Embeddable`标记值对象
2. 使用`@EmbeddedId`嵌入主键
3. 使用`@AttributeOverride`覆盖字段映射
4. 提供无参构造函数供JPA使用

## 3. Spring AOP：横切关注点处理

### UseCase日志切面

```java
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

**AOP概念：**
1. **Aspect**：切面，横切关注点的模块化
2. **Join Point**：连接点，程序执行的点
3. **Pointcut**：切点，匹配连接点的表达式
4. **Advice**：通知，在切点执行的动作
5. **Weaving**：织入，将切面应用到目标对象

### 事务管理切面

```java
@Aspect
@Component
public class TransactionalAspect {

    @Around("@annotation(transactional)")
    public Object manageTransaction(ProceedingJoinPoint joinPoint,
                                   Transactional transactional) throws Throwable {
        TransactionStatus status = transactionManager.getTransaction(
            new DefaultTransactionDefinition(transactional));

        try {
            Object result = joinPoint.proceed();
            transactionManager.commit(status);
            return result;
        } catch (Exception e) {
            transactionManager.rollback(status);
            throw e;
        }
    }
}
```

### 性能监控切面

```java
@Aspect
@Component
public class PerformanceMonitoringAspect {

    @Around("@annotation(MonitorPerformance)")
    public Object monitorPerformance(ProceedingJoinPoint joinPoint) throws Throwable {
        long startTime = System.currentTimeMillis();
        String className = joinPoint.getTarget().getClass().getSimpleName();
        String methodName = joinPoint.getSignature().getName();

        try {
            return joinPoint.proceed();
        } finally {
            long duration = System.currentTimeMillis() - startTime;
            log.info("{}.{} executed in {} ms", className, methodName, duration);

            if (duration > 1000) {
                log.warn("Slow execution detected: {}.{} took {} ms",
                    className, methodName, duration);
            }
        }
    }
}
```

## 4. 乐观锁：@Version注解

### 版本字段配置

```java
@Entity
public class Loan extends AbstractAggregateRoot<Loan> {
    @EmbeddedId
    private LoanId loanId;
    // ... 其他字段

    @Version
    private Long version;
}
```

**乐观锁机制：**
1. 每次更新时自动增加版本号
2. 检查版本号是否匹配
3. 版本不匹配时抛出`OptimisticLockException`
4. 避免并发修改冲突

### 处理并发冲突

```java
@Service
public class LoanService {

    @Retryable(value = OptimisticLockingFailureException.class, maxAttempts = 3)
    public void updateLoan(Loan loan) {
        loanRepository.save(loan);
    }

    @Recover
    public void recover(OptimisticLockingFailureException e, Loan loan) {
        log.error("Failed to update loan after retries: {}", loan.getLoanId(), e);
        throw new ConcurrentModificationException("Loan was modified by another user");
    }
}
```

## 5. 验证：Spring Validation

### 方法参数验证

```java
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

### 自定义验证注解

```java
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = IsbnValidator.class)
public @interface ValidIsbn {
    String message() default "Invalid ISBN format";

    Class<?>[] groups() default {};

    Class<? extends Payload>[] payload() default {};
}

public class IsbnValidator implements ConstraintValidator<ValidIsbn, String> {
    private static final ISBNValidator VALIDATOR = new ISBNValidator();

    @Override
    public boolean isValid(String value, ConstraintValidatorContext context) {
        return value != null && VALIDATOR.isValid(value);
    }
}

// 使用
public void addBook(@ValidIsbn String isbn) {
    // ...
}
```

### 验证组

```java
public interface CreateValidationGroup {}
public interface UpdateValidationGroup {}

public class BookDto {
    @Null(groups = CreateValidationGroup.class)
    @NotNull(groups = UpdateValidationGroup.class)
    private BookId id;

    @NotNull(groups = CreateValidationGroup.class)
    private String title;

    @NotNull(groups = CreateValidationGroup.class)
    private String isbn;
}

// 使用
public void createBook(@Validated(CreateValidationGroup.class) BookDto dto) {
    // ...
}

public void updateBook(@Validated(UpdateValidationGroup.class) BookDto dto) {
    // ...
}
```

## 6. 依赖注入：构造函数注入

### 推荐的依赖注入方式

```java
@UseCase
public class AddBookToCatalogUseCase {
    private final BookSearchService bookSearchService;
    private final BookRepository bookRepository;

    // 使用构造函数注入
    public AddBookToCatalogUseCase(BookSearchService bookSearchService,
                                   BookRepository bookRepository) {
        this.bookSearchService = bookSearchService;
        this.bookRepository = bookRepository;
    }

    public void execute(Isbn isbn) {
        BookInformation result = bookSearchService.search(isbn);
        Book book = new Book(result.title(), isbn);
        bookRepository.save(book);
    }
}
```

**构造函数注入的优势：**
1. 保证依赖不可变
2. 便于单元测试
3. 避免循环依赖
4. 明确表达依赖关系

### Lombok简化

```java
@UseCase
@RequiredArgsConstructor  // Lombok注解
public class AddBookToCatalogUseCase {
    private final BookSearchService bookSearchService;
    private final BookRepository bookRepository;

    public void execute(Isbn isbn) {
        BookInformation result = bookSearchService.search(isbn);
        Book book = new Book(result.title(), isbn);
        bookRepository.save(book);
    }
}
```

## 7. 事件处理：Spring Events

### 自定义事件

```java
public class BookAddedEvent extends ApplicationEvent {
    private final BookId bookId;
    private final String title;

    public BookAddedEvent(Object source, BookId bookId, String title) {
        super(source);
        this.bookId = bookId;
        this.title = title;
    }

    // getters
}
```

### 事件发布

```java
@Service
public class BookService {
    private final ApplicationEventPublisher eventPublisher;

    public BookService(ApplicationEventPublisher eventPublisher) {
        this.eventPublisher = eventPublisher;
    }

    public void addBook(Book book) {
        bookRepository.save(book);
        eventPublisher.publishEvent(new BookAddedEvent(this, book.getId(), book.getTitle()));
    }
}
```

### 事件监听

```java
@Component
public class BookEventListener {

    @EventListener
    public void handleBookAdded(BookAddedEvent event) {
        log.info("Book added: {} - {}", event.getBookId(), event.getTitle());
        // 处理事件
    }

    @Async
    @EventListener
    public void handleBookAddedAsync(BookAddedEvent event) {
        // 异步处理事件
    }

    @EventListener(condition = "#event.title.length() > 10")
    public void handleLongTitleBook(BookAddedEvent event) {
        // 条件监听：只处理标题长度大于10的书
    }
}
```

## 8. 配置管理：@ConfigurationProperties

### 配置属性类

```java
@ConfigurationProperties(prefix = "library.openlibrary")
@Configuration
public class OpenLibraryProperties {
    private String baseUrl = "https://openlibrary.org/";
    private int timeout = 5000;
    private int maxRetries = 3;
    private boolean enabled = true;

    // getters and setters
}
```

### application.yml配置

```yaml
library:
  openlibrary:
    base-url: https://openlibrary.org/
    timeout: 5000
    max-retries: 3
    enabled: true
```

### 使用配置

```java
@Service
class OpenLibraryBookSearchService implements BookSearchService {
    private final OpenLibraryProperties properties;
    private final RestClient restClient;

    public OpenLibraryBookSearchService(OpenLibraryProperties properties,
                                       RestClient.Builder builder) {
        this.properties = properties;
        this.restClient = builder
                .baseUrl(properties.getBaseUrl())
                .build();
    }

    public BookInformation search(Isbn isbn) {
        if (!properties.isEnabled()) {
            throw new ServiceUnavailableException("OpenLibrary service is disabled");
        }
        // ...
    }
}
```

## 9. 条件装配：@Conditional

### 条件装配Bean

```java
@Configuration
public class BookSearchServiceConfiguration {

    @Bean
    @ConditionalOnProperty(name = "library.book-search.provider", havingValue = "openlibrary")
    public BookSearchService openLibraryBookSearchService(RestClient.Builder builder) {
        return new OpenLibraryBookSearchService(builder);
    }

    @Bean
    @ConditionalOnProperty(name = "library.book-search.provider", havingValue = "google")
    public BookSearchService googleBooksBookSearchService(RestClient.Builder builder) {
        return new GoogleBooksBookSearchService(builder);
    }

    @Bean
    @ConditionalOnMissingBean(BookSearchService.class)
    public BookSearchService defaultBookSearchService() {
        return new InMemoryBookSearchService();
    }
}
```

### 条件注解类型

```java
// 条件装配注解
@ConditionalOnClass          // 类路径中存在指定类时装配
@ConditionalOnMissingClass   // 类路径中不存在指定类时装配
@ConditionalOnBean           // 容器中存在指定Bean时装配
@ConditionalOnMissingBean    // 容器中不存在指定Bean时装配
@ConditionalOnProperty       // 配置属性满足条件时装配
@ConditionalOnExpression     // SpEL表达式为true时装配
@ConditionalOnWebApplication // Web应用时装配
```

## 10. Profile配置：环境隔离

### Profile特定配置

```yaml
# application-dev.yml
library:
  datasource:
    url: jdbc:postgresql://localhost:5432/library_dev
  book-search:
    provider: openlibrary

# application-prod.yml
library:
  datasource:
    url: jdbc:postgresql://prod-db:5432/library_prod
  book-search:
    provider: google

# application-test.yml
library:
  datasource:
    url: jdbc:h2:mem:testdb
  book-search:
    provider: mock
```

### 激活Profile

```java
@SpringBootApplication
public class LibraryApplication {
    public static void main(String[] args) {
        SpringApplication app = new SpringApplication(LibraryApplication.class);
        app.setAdditionalProfiles("dev");  // 设置默认profile
        app.run(args);
    }
}
```

### Profile特定Bean

```java
@Configuration
@Profile("dev")
public class DevConfiguration {
    @Bean
    public BookSearchService mockBookSearchService() {
        return new MockBookSearchService();
    }
}

@Configuration
@Profile("prod")
public class ProdConfiguration {
    @Bean
    public BookSearchService openLibraryBookSearchService(RestClient.Builder builder) {
        return new OpenLibraryBookSearchService(builder);
    }
}
```

## 11. 测试支持：Spring Boot Test

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

        rentBookUseCase.execute(copyId, userId);

        List<Loan> loans = (List<Loan>) loanRepository.findAll();
        assertEquals(1, loans.size());
        assertEquals(copyId, loans.get(0).getCopyId());
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

        rentBookUseCase.execute(copyId, userId);

        verify(loanRepository).save(any(Loan.class));
    }
}
```

### Testcontainers

```java
@SpringBootTest
@Testcontainers
class LoanRepositoryTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15")
            .withDatabaseName("library")
            .withUsername("library")
            .withPassword("library");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    private LoanRepository loanRepository;

    @Test
    void shouldSaveAndFindLoan() {
        // 测试逻辑
    }
}
```

## DDD在Spring中的实现总结

1. **Spring Modulith**：模块化架构验证和事件支持
2. **Spring Data JPA**：仓储模式自动实现
3. **Spring AOP**：横切关注点处理
4. **@Version**：乐观锁支持
5. **Spring Validation**：声明式验证
6. **构造函数注入**：推荐依赖注入方式
7. **Spring Events**：事件发布和监听
8. **@ConfigurationProperties**：类型安全配置
9. **@Conditional**：条件装配
10. **Profile**：环境隔离
11. **Spring Boot Test**：完善的测试支持

Spring框架为DDD的实现提供了全面的支持，通过合理使用这些特性，可以构建出清晰、可维护的DDD应用。
