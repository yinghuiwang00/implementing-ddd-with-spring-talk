# DDD思想与Spring实践 - 完整文档

基于Spring IO 2024演讲示例项目的领域驱动设计完整指南。

## 📚 文档导航

本项目文档系统地介绍了领域驱动设计（DDD）的核心思想和在Spring框架中的具体实践，共分为10个模块：

### 基础篇

1. **[项目概览与DDD基础](./01-项目概览与DDD基础.md)**
   - 项目背景和业务场景
   - DDD核心概念：Ubiquitous Language、Bounded Context、Domain Model
   - 传统架构vs DDD架构对比
   - 项目结构说明

2. **[领域建模与Bounded Context](./02-领域建模与Bounded Context.md)**
   - 限界上下文的概念和重要性
   - 图书馆系统的两个核心领域：Catalog和Lending
   - 上下文映射和事件驱动通信
   - Spring Modulith的模块化应用

### 核心概念篇

3. **[领域对象设计](./03-领域对象设计.md)**
   - 聚合根（Aggregate Root）设计
   - 实体（Entity）实现
   - 值对象（Value Object）应用
   - 领域对象与JPA映射

4. **[仓储模式（Repository Pattern）](./04-仓储模式（Repository Pattern）.md)**
   - 仓储模式的作用和职责
   - Spring Data JPA实现
   - 自定义查询方法
   - 仓储与聚合的关系

5. **[领域事件（Domain Events）](./05-领域事件（Domain Events）.md)**
   - 领域事件的概念和特点
   - 事件发布和监听
   - 跨上下文通信
   - 事件驱动架构

### 实现篇

6. **[应用服务层（Application Layer）](./06-应用服务层（Application Layer）.md)**
   - 应用服务层的职责
   - 用例实现：RentBook、ReturnBook等
   - @UseCase注解和AOP切面
   - 事务管理

7. **[基础设施层（Infrastructure Layer）](./07-基础设施层（Infrastructure Layer）.md)**
   - 基础设施层的职责
   - 外部服务集成（OpenLibrary API）
   - 依赖倒置原则的应用
   - 错误处理和容错机制

### 技术篇

8. **[DDD在Spring中的实现技巧](./08-DDD在Spring中的实现技巧.md)**
   - Spring Modulith模块化验证
   - Spring Data JPA仓储实现
   - Spring AOP横切关注点
   - 乐观锁、验证、配置管理等

### 实践篇

9. **[测试策略](./09-测试策略.md)**
   - 测试金字塔：单元测试、集成测试、E2E测试
   - 领域对象测试
   - Testcontainers集成测试
   - Spring Modulith测试

10. **[架构演进与最佳实践](./10-架构演进与最佳实践.md)**
    - 从贫血模型到充血模型
    - 事务边界管理
    - 性能考虑和优化
    - 扩展性设计和架构演进

## 🎯 学习路径

### 初学者路径
1. 阅读[项目概览与DDD基础](./01-项目概览与DDD基础.md)了解DDD基本概念
2. 学习[领域建模与Bounded Context](./02-领域建模与Bounded Context.md)理解上下文划分
3. 研究[领域对象设计](./03-领域对象设计.md)掌握核心建模技术

### 进阶开发者路径
1. 深入[仓储模式](./04-仓储模式（Repository Pattern）.md)和[领域事件](./05-领域事件（Domain Events）.md)
2. 学习[应用服务层](./06-应用服务层（Application Layer）)和[基础设施层](./07-基础设施层（Infrastructure Layer）)
3. 掌握[DDD在Spring中的实现技巧](./08-DDD在Spring中的实现技巧.md)

### 架构师路径
1. 研究[测试策略](./09-测试策略.md)建立完整的测试体系
2. 学习[架构演进与最佳实践](./10-架构演进与最佳实践.md)进行架构设计
3. 结合实际项目进行DDD实践

## 📖 关键概念速查

### DDD核心概念
- **Ubiquitous Language（通用语言）**：开发团队和业务专家使用一致的术语
- **Bounded Context（限界上下文）**：系统的边界，每个上下文有自己的模型
- **Aggregate（聚合）**：数据一致性边界，由聚合根管理
- **Entity（实体）**：有唯一标识的对象
- **Value Object（值对象）**：通过属性值标识的不可变对象
- **Domain Event（领域事件）**：表示领域中发生的事情
- **Repository（仓储）**：抽象持久化细节的接口

### Spring框架特性
- **Spring Modulith**：模块化架构验证和事件支持
- **Spring Data JPA**：仓储模式自动实现
- **Spring AOP**：横切关注点处理
- **@Version**：乐观锁支持
- **Spring Validation**：声明式验证

## 🔗 相关资源

### 官方资源
- [Spring IO 2024演讲视频](https://www.youtube.com/watch?v=VGhg6Tfxb60)
- [演讲幻灯片](https://speakerdeck.com/maciejwalkowiak/implementing-domain-driven-desing-with-spring)
- [Spring Modulith文档](https://docs.spring.io/spring-modulith/reference/)
- [Spring Data JPA文档](https://docs.spring.io/spring-data/jpa/reference/)

### DDD经典著作
- 《领域驱动设计》- Eric Evans
- 《实现领域驱动设计》- Vaughn Vernon
- 《领域驱动设计精粹》- Vaughn Vernon

## 💡 项目特点

- ✅ 清晰的限界上下文划分
- ✅ 充血领域模型实现
- ✅ 完整的领域事件机制
- ✅ Spring框架深度集成
- ✅ 完善的测试体系
- ✅ 实际可运行的代码

## 🚀 快速开始

```bash
# 克隆项目
git clone <repository-url>

# 进入项目目录
cd implementing-ddd-with-spring-talk

# 运行测试
./gradlew test

# 启动应用
./gradlew bootRun
```

## 📝 代码示例

本项目包含了大量可直接运行的代码示例，涵盖：

- 领域对象：Book、Copy、Loan等聚合根
- 值对象：Isbn、BookId、CopyId、UserId等
- 仓储接口：BookRepository、CopyRepository、LoanRepository
- 应用服务：RentBookUseCase、ReturnBookUseCase等
- 事件处理：LoanCreated、LoanClosed等事件
- 外部集成：OpenLibrary API集成

## 🤝 贡献

欢迎提交问题和改进建议！

## 📄 许可证

本项目基于原演讲示例代码，遵循相应的开源许可证。

---

**开始学习**：建议从[项目概览与DDD基础](./01-项目概览与DDD基础.md)开始，逐步深入各个模块。
