# 01. SOLID、DRY、KISS、YAGNI 与关注点分离

## 学习目标

学完本节后，你应该能够：

1. 用自己的语言解释 SOLID、DRY、KISS、YAGNI 和 Separation of Concerns。
2. 识别代码及系统设计中违反这些原则的信号。
3. 理解这些原则之间可能发生的冲突，而不是机械套用。
4. 在架构面试中结合真实场景说明设计权衡。

## 一、为什么需要设计原则

设计原则不是语法规则，也不是必须遵守的法律。它们更像一组帮助团队控制复杂度的经验。

一个系统刚开始时，任何设计似乎都能工作。真正的差异通常在需求变化、团队扩大、故障发生和系统运行几年之后才出现。良好的设计原则主要改善以下质量属性：

- Changeability：修改一处需求时，影响范围可控。
- Testability：核心逻辑可以被快速、独立地验证。
- Maintainability：新成员能够理解并安全修改系统。
- Extensibility：增加能力时不需要反复破坏现有逻辑。
- Operability：部署、排障和恢复不依赖少数人的个人经验。

架构师需要避免两个极端：完全没有设计约束，以及为了“完美架构”进行过度抽象。

## 二、SOLID

SOLID 最初用于 Object-Oriented Design，但它同样适用于模块、Service 和系统边界。

### 2.1 Single Responsibility Principle（SRP）

> 一个模块应该只有一个引起它变化的原因。

这里的“责任”不是指一个 Class 只能有一个 method，而是指变化是否来自同一个业务角色或同一类原因。

例如，一个 `OrderService` 同时负责价格计算、数据库存储、发送邮件和生成 PDF，它会因为财务规则、数据库结构、通知模板和报表格式四种原因发生变化。

更合理的拆分是：

```text
OrderPricingService   -> 价格计算
OrderRepository       -> 订单持久化
NotificationService   -> 通知
InvoiceGenerator      -> 发票生成
```

SRP 的风险是过度拆分。如果每一个 method 都成为单独的 Class，代码跳转和理解成本反而会上升。判断标准应是“变化原因”，而不是代码行数。

### 2.2 Open/Closed Principle（OCP）

> 软件实体应该对扩展开放，对修改关闭。

这并不意味着现有代码永远不能修改，而是希望增加新行为时，尽量通过稳定的扩展点完成。

下面的支付逻辑需要在每次增加支付方式时修改：

```java
public void pay(String type, Money amount) {
    if (type.equals("CARD")) {
        // card payment
    } else if (type.equals("PAYPAL")) {
        // PayPal payment
    }
}
```

可以使用 Strategy：

```java
public interface PaymentProcessor {
    PaymentType supports();
    PaymentResult pay(Money amount);
}
```

新增支付方式时实现新的 `PaymentProcessor`，核心流程不需要增加更多 `if/else`。

但是，不要为了一个尚未出现的变化提前创建复杂 Plugin Framework。OCP 应该围绕已经识别出的稳定点和变化点设计。

### 2.3 Liskov Substitution Principle（LSP）

> 子类型必须能够替换父类型，并保持调用者所依赖的行为契约。

LSP 关注的不只是类型兼容，还包括行为兼容。例如父类型承诺“读取数据不会修改状态”，子类型就不应该在读取时删除数据。

常见违反信号：

- 子类抛出父类契约中不存在的异常。
- 子类加强输入条件或削弱输出保证。
- 调用者必须用 `instanceof` 判断具体子类。
- 子类覆盖 method 后只抛出 `UnsupportedOperationException`。

在 Service API 中，LSP 也意味着新版本不能悄悄破坏旧客户端依赖的语义。

### 2.4 Interface Segregation Principle（ISP）

> 客户端不应该依赖它不需要的接口。

不要设计一个包含几十个 method 的万能接口。与其提供 `UserOperations`，通常更适合按照使用方拆成：

```text
UserReader
UserWriter
UserAuthenticator
UserExporter
```

在 Microservice 中，ISP 也可以理解为：不要强迫消费者依赖一个巨大 API、巨大 SDK 或包含所有事件的统一 Schema。

### 2.5 Dependency Inversion Principle（DIP）

> 高层策略不应依赖低层实现；两者都应依赖抽象。

核心业务不应该直接绑定某个数据库、Message Broker 或外部供应商。

```text
OrderUseCase
    ↓ depends on
OrderRepository (port)
    ↑ implemented by
PostgresOrderRepository (adapter)
```

这正是 Hexagonal Architecture 中 Port and Adapter 的核心思想。

DIP 不等于每个 Class 都必须有 Interface。如果一个实现没有替换需求、没有外部副作用，也没有独立测试价值，额外接口可能只是噪音。

## 三、DRY

DRY 是 Don't Repeat Yourself，重点是避免“知识重复”，而不只是避免相同代码。

例如，税率规则同时存在于 React、Java Backend 和 SQL Stored Procedure 中。即使三份代码写法完全不同，它们仍然表达了同一份业务知识。一旦税率变化，就可能出现不一致。

### 适合消除的重复

- 同一业务规则在多个位置独立维护。
- 同一 API Schema 被手工复制到多个项目。
- 相同安全校验在不同入口实现不一致。
- 大量机械代码可以通过 Library、Template 或 Code Generation 统一。

### 不应该急于消除的重复

两段代码当前相似，不代表它们属于同一个概念。如果属于不同 Domain，未来变化方向不同，强行复用会制造错误耦合。

一个实用判断是：

> 它们是“碰巧长得一样”，还是“必须永远保持一致”？

只有后者更适合抽象。

## 四、KISS

KISS 是 Keep It Simple, Stupid。它要求选择满足需求的最简单方案。

简单不等于简陋，也不等于代码最少。真正的简单是系统行为容易理解、变化影响容易预测、故障容易定位。

例如，一个每日处理 5,000 条记录的内部应用，使用单体应用和 PostgreSQL 可能已经足够。引入 Kubernetes、Kafka、Event Sourcing 和十几个 Microservices，可能只增加运维成本，并没有解决实际问题。

评估一个方案是否简单，可以问：

1. 新成员多久能理解核心流程？
2. 发生故障时，需要检查多少个组件？
3. 数据是否存在多个 Source of Truth？
4. 是否可以用更少的技术满足相同的 NFR？
5. 删除某个抽象后，系统是否仍然清晰？

## 五、YAGNI

YAGNI 是 You Aren't Gonna Need It，意思是不要实现当前并不需要的功能。

常见的 YAGNI 违反包括：

- 业务只在一个国家运行，却提前实现全球税务引擎。
- 当前只有一种数据库，却开发通用 Database Abstraction Framework。
- 没有多租户需求，却提前让每张表和每个 API 支持多租户。
- 没有规模证据，却提前拆分大量 Microservices。

YAGNI 不等于不做长期规划。架构师仍然应该识别未来方向，但可以通过保留演进路径，而不是立即实现全部能力。

例如，可以先使用模块化单体并明确 Domain Boundary；当团队规模和部署独立性真正需要时，再拆成 Microservices。

## 六、Separation of Concerns

Separation of Concerns（关注点分离）是把不同性质的问题放在不同边界中处理。

典型关注点包括：

- Presentation：React 页面、交互和 View Model。
- Application：Use Case 编排和事务边界。
- Domain：业务规则和不变量。
- Infrastructure：数据库、Kafka、第三方 API。
- Cross-cutting Concerns：认证、日志、Tracing、Metrics。

以创建订单为例：

```text
React Form
  -> OrderController
  -> CreateOrderUseCase
  -> Order Domain Model
  -> OrderRepository / EventPublisher
```

如果价格计算散落在 React、Controller 和 SQL 中，关注点没有被清楚分离，规则就很难测试和复用。

## 七、这些原则如何共同工作

这些原则有时会产生张力：

- DRY 鼓励复用，但错误复用会破坏低耦合。
- SRP 鼓励拆分，但过度拆分会违反 KISS。
- OCP 鼓励扩展点，但没有真实需求的扩展点会违反 YAGNI。
- DIP 改善测试与替换能力，但过多抽象会增加认知负担。

因此，不应该问“是否完全符合 SOLID”，而应该问：

> 这个原则正在帮助我们控制哪一种变化或风险？为此付出的复杂度成本是否值得？

## 八、架构面试回答框架

遇到设计原则问题时，可以按以下方式回答：

1. 简洁定义原则。
2. 描述它解决的具体问题。
3. 给出代码级或架构级案例。
4. 说明过度使用的风险。
5. 结合项目说明最终权衡。

示例回答：

> 在支付系统中，我们使用 DIP 将核心 Checkout Flow 与支付供应商隔离。Application Layer 依赖 `PaymentGateway`，Stripe 和 Adyen 分别作为 Adapter。这样既能做单元测试，也能在供应商故障时切换。但我们没有为所有内部 Class 创建 Interface，只在外部依赖和明确变化点建立抽象，以控制复杂度。

## 九、自测题

1. SRP 中“一个责任”应该如何定义？
2. 为什么两段相同代码不一定应该被抽象？
3. OCP 和 YAGNI 发生冲突时应该如何选择？
4. DIP 与 Dependency Injection 有什么区别？
5. 举例说明一个为了 KISS 而主动放弃复杂技术的设计决策。

## 十、总结

- SOLID 帮助我们设计清晰、可替换和可演进的边界。
- DRY 的目标是消除重复知识，而不是追求零重复代码。
- KISS 强调满足需求的最小复杂度。
- YAGNI 防止团队为不确定的未来提前支付成本。
- Separation of Concerns 让不同变化原因落在不同边界。
- 架构师的价值不是机械应用原则，而是在收益与复杂度之间做出可解释的选择。
