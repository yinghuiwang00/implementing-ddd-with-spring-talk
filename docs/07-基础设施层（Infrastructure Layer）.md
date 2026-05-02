# 基础设施层（Infrastructure Layer）

## 基础设施层的概念

基础设施层是DDD架构中负责技术实现细节的层，它提供领域层和应用层所需的技术能力，如持久化、外部API调用、消息传递等。

### 基础设施层的职责

1. **持久化实现**：数据库访问、ORM映射
2. **外部服务集成**：调用第三方API
3. **消息传递**：消息队列、事件总线
4. **文件存储**：文件上传、下载
5. **缓存**：Redis、Memcached等
6. **配置管理**：读取配置文件
7. **日志记录**：日志框架集成

### 基础设施层的设计原则

1. **依赖倒置**：高层模块（领域层、应用层）不依赖低层模块（基础设施层）
2. **接口隔离**：通过接口隔离具体实现
3. **可替换性**：可以轻松替换具体实现
4. **关注点分离**：技术细节不污染业务逻辑

## 项目中的基础设施层

### 1. OpenLibraryBookSearchService（OpenLibrary图书搜索服务）

```java
package library.catalog.infrastructure;

import library.catalog.application.BookInformation;
import library.catalog.application.BookSearchService;
import library.catalog.domain.Isbn;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestClient;

@Service
class OpenLibraryBookSearchService implements BookSearchService {
    private final RestClient restClient;

    public OpenLibraryBookSearchService(RestClient.Builder builder) {
        this.restClient = builder
                .baseUrl("https://openlibrary.org/")
                .build();
    }

    public BookInformation search(Isbn isbn) {
        OpenLibraryIsbnSearchResult result = restClient.get()
                .uri("isbn/{isbn}.json", isbn.value())
                .retrieve()
                .body(OpenLibraryIsbnSearchResult.class);
        return new BookInformation(result.title());
    }
}
```

**服务说明：**
- 实现`BookSearchService`接口
- 调用OpenLibrary API获取书籍信息
- 使用Spring的`RestClient`进行HTTP调用
- 将外部API响应转换为领域对象

### 2. OpenLibraryIsbnSearchResult（API响应DTO）

```java
package library.catalog.infrastructure;

import java.util.List;

record OpenLibraryIsbnSearchResult(List<String> publishers,
                                   String title,
                                   List<String> isbn_13,
                                   int revisions) {
}
```

**DTO说明：**
- 数据传输对象（Data Transfer Object）
- 映射外部API的响应格式
- 使用Java record，自动不可变
- 包含API返回的所有字段

## 依赖倒置原则的应用

### 接口定义在应用层

```java
// 接口定义在应用层
package library.catalog.application;

import library.catalog.domain.Isbn;

public interface BookSearchService {
    BookInformation search(Isbn isbn);
}
```

**原因：**
- 接口属于业务抽象
- 应用层依赖接口，不依赖具体实现
- 便于替换不同的实现

### 实现在基础设施层

```java
// 实现在基础设施层
package library.catalog.infrastructure;

@Service
class OpenLibraryBookSearchService implements BookSearchService {
    // 具体实现
}
```

**好处：**
- 具体实现细节与业务逻辑分离
- 可以轻松切换不同的API提供商
- 便于测试时使用Mock实现

## 外部服务集成

### REST API集成

```java
@Service
class OpenLibraryBookSearchService implements BookSearchService {
    private final RestClient restClient;

    public OpenLibraryBookSearchService(RestClient.Builder builder) {
        this.restClient = builder
                .baseUrl("https://openlibrary.org/")
                .build();
    }

    public BookInformation search(Isbn isbn) {
        OpenLibraryIsbnSearchResult result = restClient.get()
                .uri("isbn/{isbn}.json", isbn.value())
                .retrieve()
                .body(OpenLibraryIsbnSearchResult.class);
        return new BookInformation(result.title());
    }
}
```

**技术要点：**
1. **RestClient.Builder**：使用Spring 6的RestClient
2. **baseUrl**：设置基础URL
3. **uri**：构建请求URI，支持路径变量
4. **retrieve()**：执行请求并获取响应
5. **body()**：将响应转换为指定类型

### 错误处理

```java
@Service
class OpenLibraryBookSearchService implements BookSearchService {
    public BookInformation search(Isbn isbn) {
        try {
            OpenLibraryIsbnSearchResult result = restClient.get()
                    .uri("isbn/{isbn}.json", isbn.value())
                    .retrieve()
                    .body(OpenLibraryIsbnSearchResult.class);

            if (result == null || result.title() == null) {
                throw new BookNotFoundException("Book not found for ISBN: " + isbn.value());
            }

            return new BookInformation(result.title());
        } catch (WebClientResponseException e) {
            if (e.getStatusCode() == HttpStatus.NOT_FOUND) {
                throw new BookNotFoundException("Book not found for ISBN: " + isbn.value());
            }
            throw new ExternalServiceException("Failed to search book", e);
        }
    }
}
```

**错误处理策略：**
1. **空值检查**：验证响应不为空
2. **HTTP状态码**：根据状态码抛出不同的异常
3. **异常转换**：将技术异常转换为业务异常
4. **日志记录**：记录详细的错误信息

### 超时和重试

```java
@Service
class OpenLibraryBookSearchService implements BookSearchService {
    private final RestClient restClient;

    public OpenLibraryBookSearchService(RestClient.Builder builder) {
        this.restClient = builder
                .baseUrl("https://openlibrary.org/")
                .defaultHeader("Accept", "application/json")
                .build();
    }

    @Retryable(maxAttempts = 3, backoff = @Backoff(delay = 1000))
    @CircuitBreaker(name = "openLibrary", fallbackMethod = "searchFallback")
    public BookInformation search(Isbn isbn) {
        OpenLibraryIsbnSearchResult result = restClient.get()
                .uri("isbn/{isbn}.json", isbn.value())
                .retrieve()
                .body(OpenLibraryIsbnSearchResult.class);
        return new BookInformation(result.title());
    }

    public BookInformation searchFallback(Isbn isbn, Exception e) {
        log.warn("Fallback activated for ISBN: {}", isbn.value());
        return new BookInformation("Unknown Title");
    }
}
```

**容错机制：**
1. **重试**：使用`@Retryable`实现自动重试
2. **熔断器**：使用Resilience4j的`@CircuitBreaker`
3. **降级**：提供fallback方法
4. **超时配置**：在RestClient配置超时

## 持久化基础设施

### Spring Data JPA配置

```java
@Configuration
@EnableJpaRepositories(basePackages = "library.catalog.domain")
@EnableTransactionManagement
class JpaConfiguration {

    @Bean
    public DataSource dataSource() {
        return DataSourceBuilder.create()
                .url("jdbc:postgresql://localhost:5432/library")
                .username("library")
                .password("password")
                .build();
    }

    @Bean
    public LocalContainerEntityManagerFactoryBean entityManagerFactory(
            DataSource dataSource,
            JpaProperties jpaProperties) {

        LocalContainerEntityManagerFactoryBean em = new LocalContainerEntityManagerFactoryBean();
        em.setDataSource(dataSource);
        em.setPackagesToScan("library.catalog.domain", "library.lending.domain");
        em.setJpaVendorAdapter(new HibernateJpaVendorAdapter());
        em.setJpaProperties(jpaProperties.getProperties());

        return em;
    }
}
```

**配置说明：**
1. **@EnableJpaRepositories**：启用JPA仓储
2. **DataSource配置**：配置数据库连接
3. **EntityManagerFactory**：配置JPA实体管理器
4. **包扫描**：指定扫描领域对象的包

### JPA实体映射

```java
@Entity
public class Book {
    @EmbeddedId
    private BookId id;

    private String title;

    @Embedded
    @AttributeOverride(name = "value", column = @Column(name = "isbn"))
    private Isbn isbn;

    // JPA需要无参构造函数
    Book() {
    }

    // 业务构造函数
    public Book(String title, Isbn isbn) {
        Assert.notNull(title, "title must not be null");
        Assert.notNull(isbn, "isbn must not be null");

        this.id = new BookId();
        this.title = title;
        this.isbn = isbn;
    }
}
```

**映射技巧：**
1. **@EmbeddedId**：使用嵌入的主键
2. **@Embedded**：嵌入值对象
3. **@AttributeOverride**：覆盖值对象字段的列名
4. **无参构造函数**：JPA反射需要
5. **业务构造函数**：业务代码使用

### 复杂类型映射

```java
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

**映射说明：**
1. **@Embeddable**：标记为可嵌入的类型
2. **@Column**：指定列名
3. **构造函数**：支持无参和有参构造函数
4. **访问方法**：提供业务语义的方法

## 缓存基础设施

### Spring Cache配置

```java
@Configuration
@EnableCaching
class CacheConfiguration {

    @Bean
    public CacheManager cacheManager() {
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(Duration.ofHours(1))
                .disableCachingNullValues();

        return RedisCacheManager.builder(redisConnectionFactory())
                .cacheDefaults(config)
                .transactionAware()
                .build();
    }

    @Bean
    public RedisConnectionFactory redisConnectionFactory() {
        return new LettuceConnectionFactory("localhost", 6379);
    }
}
```

**缓存策略：**
1. **@EnableCaching**：启用缓存
2. **Redis配置**：使用Redis作为缓存
3. **TTL配置**：设置过期时间
4. **事务感知**：缓存与事务同步

### 缓存应用

```java
@Service
class OpenLibraryBookSearchService implements BookSearchService {

    @Cacheable(value = "book-search", key = "#isbn.value()")
    public BookInformation search(Isbn isbn) {
        // 调用外部API
        OpenLibraryIsbnSearchResult result = restClient.get()
                .uri("isbn/{isbn}.json", isbn.value())
                .retrieve()
                .body(OpenLibraryIsbnSearchResult.class);
        return new BookInformation(result.title());
    }

    @CacheEvict(value = "book-search", key = "#isbn.value()")
    public void evictCache(Isbn isbn) {
        // 清除缓存
    }
}
```

**缓存注解：**
1. **@Cacheable**：缓存方法结果
2. **@CacheEvict**：清除缓存
3. **@CachePut**：更新缓存
4. **@Caching**：组合多个缓存操作

## 消息传递基础设施

### Spring Modulith事件配置

```java
@Configuration
class EventConfiguration {

    @Bean
    public ApplicationModuleListenerApplicationListener applicationModuleListener() {
        return new ApplicationModuleListenerApplicationListener();
    }
}
```

**事件处理：**
1. **事件监听**：自动注册`@ApplicationModuleListener`注解的方法
2. **事件路由**：根据事件类型路由到对应的监听器
3. **事务管理**：与Spring事务集成

## 配置管理

### application.yml配置

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/library
    username: library
    password: ${DB_PASSWORD}
    hikari:
      maximum-pool-size: 10
      minimum-idle: 5

  jpa:
    hibernate:
      ddl-auto: validate
    show-sql: false
    properties:
      hibernate:
        dialect: org.hibernate.dialect.PostgreSQLDialect

  data:
    redis:
      host: localhost
      port: 6379
      timeout: 2000ms

openlibrary:
  base-url: https://openlibrary.org/
  timeout: 5000
  max-retries: 3
```

### 配置属性类

```java
@ConfigurationProperties(prefix = "openlibrary")
@Configuration
class OpenLibraryProperties {
    private String baseUrl = "https://openlibrary.org/";
    private int timeout = 5000;
    private int maxRetries = 3;

    // getters and setters
}
```

**配置管理：**
1. **@ConfigurationProperties**：绑定配置属性
2. **默认值**：提供合理的默认值
3. **类型安全**：强类型配置访问
4. **环境变量**：支持环境变量覆盖

## 基础设施层的测试

### 外部服务Mock测试

```java
@ExtendWith(MockitoExtension.class)
class OpenLibraryBookSearchServiceTest {

    @Mock
    private RestClient restClient;

    @Mock
    private RestClient.RequestHeadersUriSpec requestHeadersUriSpec;

    @Mock
    private RestClient.ResponseSpec responseSpec;

    @InjectMocks
    private OpenLibraryBookSearchService service;

    @Test
    void shouldSearchBook() {
        Isbn isbn = new Isbn("978-3-16-148410-0");
        OpenLibraryIsbnSearchResult result = new OpenLibraryIsbnSearchResult(
            List.of("Publisher"),
            "Test Book",
            List.of("978-3-16-148410-0"),
            1
        );

        when(restClient.get()).thenReturn(requestHeadersUriSpec);
        when(requestHeadersUriSpec.uri(anyString(), any())).thenReturn(requestHeadersUriSpec);
        when(requestHeadersUriSpec.retrieve()).thenReturn(responseSpec);
        when(responseSpec.body(OpenLibraryIsbnSearchResult.class)).thenReturn(result);

        BookInformation info = service.search(isbn);

        assertEquals("Test Book", info.title());
    }
}
```

### 集成测试

```java
@SpringBootTest
@Testcontainers
class OpenLibraryBookSearchServiceIntegrationTest {

    @Container
    static GenericContainer<?> wiremock = new GenericContainer<>("wiremock/wiremock:latest")
            .withExposedPorts(8080);

    @Autowired
    private BookSearchService bookSearchService;

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("openlibrary.base-url",
            () -> "http://" + wiremock.getHost() + ":" + wiremock.getFirstMappedPort());
    }

    @Test
    void shouldSearchBookFromWireMock() {
        // 配置WireMock stub
        stubFor(get(urlPathEqualTo("/isbn/978-3-16-148410-0.json"))
                .willReturn(aResponse()
                        .withStatus(200)
                        .withBody("{\"title\":\"Test Book\"}")));

        Isbn isbn = new Isbn("978-3-16-148410-0");
        BookInformation info = bookSearchService.search(isbn);

        assertEquals("Test Book", info.title());
    }
}
```

## 基础设施层的最佳实践

### 1. 接口与实现分离

```java
// 接口在应用层
public interface BookSearchService {
    BookInformation search(Isbn isbn);
}

// 实现在基础设施层
@Service
class OpenLibraryBookSearchService implements BookSearchService {
    // 实现细节
}
```

### 2. 使用适配器模式

```java
// 适配器接口
public interface BookSearchAdapter {
    BookInformation searchByIsbn(String isbn);
}

// 具体适配器
@Service
class OpenLibraryAdapter implements BookSearchAdapter {
    public BookInformation searchByIsbn(String isbn) {
        // 适配OpenLibrary API
    }
}

// 另一个适配器
@Service
class GoogleBooksAdapter implements BookSearchAdapter {
    public BookInformation searchByIsbn(String isbn) {
        // 适配Google Books API
    }
}
```

### 3. 错误处理和转换

```java
@Service
class ExternalServiceAdapter {
    public BookInformation search(Isbn isbn) {
        try {
            return callExternalApi(isbn);
        } catch (HttpClientException e) {
            throw new BusinessException("External service error", e);
        }
    }
}
```

### 4. 配置外部化

```yaml
external:
  services:
    openlibrary:
      url: https://openlibrary.org/
      timeout: 5000
      retries: 3
```

### 5. 健康检查

```java
@Component
class ExternalServiceHealthIndicator implements HealthIndicator {

    @Autowired
    private BookSearchService bookSearchService;

    @Override
    public Health health() {
        try {
            bookSearchService.search(new Isbn("978-3-16-148410-0"));
            return Health.up().build();
        } catch (Exception e) {
            return Health.down().withException(e).build();
        }
    }
}
```

## 基础设施层在项目中的应用总结

1. **依赖倒置**：接口在应用层，实现在基础设施层
2. **外部服务集成**：使用RestClient调用OpenLibrary API
3. **持久化**：Spring Data JPA实现仓储
4. **配置管理**：外部化配置，便于环境切换
5. **错误处理**：统一异常处理和转换
6. **可测试性**：支持Mock和集成测试
7. **可替换性**：可以轻松替换具体实现

基础设施层是DDD架构的技术基础，本项目展示了如何优雅地处理技术关注点，保持领域层的纯净。
