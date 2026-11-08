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

- 子类引入父类契约不允许的失败行为（是否违规必须结合契约判断）。
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


## 十一、从原则到可执行设计：如何识别真正的变化边界

前面给出概念地图，下面深入到代码、失败语义和架构决策。以下订单、支付和税务场景均为教学假设，不代表读者的真实项目经历。判断设计的顺序应是：**业务不变量 → 变化来源 → 调用契约 → 实现边界 → 验证证据**。

### 11.1 SRP：用变化来源和共同变化验证拆分

假设订单类同时有计算折扣、写数据库、发送确认邮件三个方法。不能仅凭“有三个方法”判定违反 SRP：先问三类变更是否由不同业务角色决定，是否必须同一次发布，以及变更测试范围是否不同。如果促销人员改折扣，数据库管理员改存储，运营人员改邮件模板，放在一个模块会让无关变更共享风险。

可以查看一段时间内的 Git 变更：若改邮件总要触碰价格规则，并且不是正常业务关联，可能存在边界泄漏。共同变化只是证据，不是数学证明；大规模格式化、统一升级也会造成共同修改，不能据此合并模块。

拆分之后仍需有人编排用例。例如 CreateOrderUseCase 可以协调报价、保存和通知安排，因为“完成创建订单用例”本身是责任。它不应重新实现税率公式、拼 SQL 或处理邮件服务商响应。**编排依赖几个协作者，不等于承载几份业务知识。**

可操作的重构步骤：

1. 用特征测试（Characterization Test）固定现有价格结果和异常行为。
2. 先提取无外部副作用的报价函数，显式接收订单项、价格版本和促销规则。
3. 将存储与外部发送移到适配器，保留原事务边界，暂不同时拆部署。
4. 检查依赖方向，禁止领域层反向导入 Controller 或供应商 SDK。
5. 观察一次新促销规则修改是否局限于报价模块，而不是用类数量衡量成功。

**反例：**把所有校验提到 ValidationService 并不一定改善 SRP。邮箱格式、订单能否取消、信用额度和数据库唯一性属于不同语义；仅因都叫“校验”而合并，会形成新的万能服务。领域不变量应归属相应领域模型，用例前置条件归应用层，输入格式归边界；数据库约束作为并发下的最终防线。

### 11.2 OCP：关闭的是某一变化轴，不是整个系统

支付方式已多次增加时，可把供应商处理作为扩展点。先定义业务语义：金额的货币与精度、请求标识、结果状态、超时后的处理。若接口只有 pay(double amount) 返回 boolean，即使实现了 Strategy，也无法表达“供应商可能已经扣款但响应丢失”。

新增供应商时通常仍需修改装配、配置、凭证和监控。这不自动违反 OCP：目标是让稳定结账流程无需了解各供应商细节，不是承诺零修改。相反，如果每次都扩充参数对象，加入仅某一家需要的字段，并让其他实现到处判断 null，说明变化轴没有真正隔离。

选择扩展机制时，比较实际约束：

| 机制 | 适用情况 | 付出的成本 |
|---|---|---|
| 明确的 if/switch | 分支少、集合稳定、逻辑短 | 新类型可能需修改多处 |
| Strategy + 显式注册 | 行为差异真实存在，公共契约稳定 | 接口、装配、契约测试 |
| 配置驱动规则 | 参数变化多于算法变化 | 配置校验、版本和回滚 |
| 独立插件运行环境 | 第三方扩展且需要隔离 | 生命周期、兼容、安全和诊断 |

不要把普通配置参数变成几十个策略类，也不要把安全敏感业务脚本化仅为追求“不改代码”。可执行配置同样需要审批、测试、审计和故障回退。

### 11.3 LSP：把“能替换”写成可测试的行为契约

一个简化的账户扣款契约可以规定：输入为正数且不超过可用余额；成功时余额准确减少该值；失败时余额不变。替代实现不得要求“扣款额至少100”，因为这会拒绝父契约允许的输入；也不得成功返回后只扣一半，因为这削弱输出保证。

检查 LSP 不止看方法签名，还要明确：

- **前置条件（Precondition）**：新实现不能要求调用方额外满足未约定条件。
- **后置条件（Postcondition）**：成功或失败后的状态保证不能变弱。
- **不变量（Invariant）**：例如余额不为负、订单状态转换合法。
- **状态历史**：例如只读视图不能突然提供绕过限制的可变状态入口。
- **异常与副作用**：失败是否可能已执行外部操作，是否改变共享状态。

原文中的 UnsupportedOperationException、instanceof 和新增异常都是警示信号，**不是仅凭出现就能判定违规**。如果接口本来允许可选操作，或者明确约定可能抛出某类失败，按契约拒绝操作可能合法。反之，即使没有抛异常，也可能因静默丢数据违反契约。Java 对重写方法的 checked exception 有语言限制，但编译通过不证明 RuntimeException、幂等性和副作用语义兼容。

本地内存实现返回立即可见结果，远程副本实现只保证最终一致，二者不能未经说明便互换。若接口只承诺最终一致，则测试不能偷加 read-your-writes 假设；若业务依赖写后读，应该明确一致性需求并选择实现。这是将行为契约思路用于服务边界的工程推演，不是把所有 API 兼容问题都简化为面向对象继承。

### 11.4 ISP：从消费者需要的能力设计接口

假设报表只需查询订单，后台管理需要修改，支付适配器需要授权、扣款和退款。把它们都放入 EnterpriseService 会让报表模块依赖写操作和无关类型；一次退款签名修改还可能迫使报表重新构建。

按使用方提取 OrderReader、OrderWriter 能缩小依赖面，但“一个接口一个方法”不是目标。一次用例经常一起使用的查询与分页能力可保留在同一个窄接口中。拆分时要追踪实际消费者、发布节奏和权限，而不是按数据库表机械切接口。

**接口隔离不是访问控制。**某个对象只注入 OrderReader，并不能替代服务端授权；用户仍可能直接调用写接口。HTTP API、SDK 和事件 Schema 也要明确兼容与消费者迁移策略。为每个消费者复制完整领域模型，会把 ISP 变成维护大量副本的成本；应先判断是否真的存在不同稳定契约。

### 11.5 DIP、DI、IoC 与依赖方向

| 名称 | 核心问题 | 简例 |
|---|---|---|
| DIP | 策略在源码上依赖谁 | 结账依赖业务定义的 PaymentGateway |
| Dependency Injection | 协作者怎样交给对象 | 构造器传入网关实现 |
| Inversion of Control | 谁控制创建或执行流程 | 框架调用处理器或管理对象生命周期 |

构造器注入 ConcreteStripeClient 是 DI，但高层策略仍绑定具体 SDK，不因此满足 DIP。不使用框架，在 main 中手工装配业务端口和适配器，也可以满足 DIP。

业务端口应该以用例语言描述所需能力，而不是原样复制供应商 SDK。若 PaymentGateway 返回 StripeResponse，或者核心逻辑仍需判断供应商错误码，依赖仍然泄漏。适配器负责把供应商“处理中/拒绝/未知”等状态翻译到业务契约；翻译不能抹掉关键差异。

运行时调用方向可以是“业务 → 适配器”，而源码依赖为“适配器 → 业务定义的接口”。两者不冲突。依赖反转也没有消除网络超时、凭证管理、交易一致性或替换成本；更换数据库仍需迁移数据和核验查询语义，不能靠接口宣称已经做到数据库无关。

## 十二、可运行示例：让扩展与替换接受行为验证

下面完整程序使用 Java 17，无第三方依赖。它只验证进程内的策略选择、消费方定义的端口及同键重复调用；生产支付还必须处理跨进程幂等、故障恢复和供应商契约。示例使用整数最小货币单位，避免二进制浮点金额；这里只支持 EUR，不代表所有货币都具有两位小数。

保存为 DesignPrinciplesDemo.java：

```java
import java.util.HashMap;
import java.util.Map;
import java.util.Objects;

public class DesignPrinciplesDemo {
    enum Method { CARD, WALLET }
    enum Status { APPROVED, DECLINED, UNKNOWN }

    record Request(String key, long minorUnits, String currency) {
        Request {
            if (key == null || key.isBlank() || minorUnits <= 0
                    || !"EUR".equals(currency)) {
                throw new IllegalArgumentException("invalid payment request");
            }
        }
    }

    record Result(Status status, String reference) {
        Result {
            Objects.requireNonNull(status);
            Objects.requireNonNull(reference);
        }
    }

    interface Gateway {
        // Same key + same payload -> same result in this demo.
        // Same key + different payload -> reject; no second charge.
        Result charge(Request request);
    }

    static final class FakeGateway implements Gateway {
        private record Saved(Request request, Result result) {}
        private final Map<String, Saved> saved = new HashMap<>();
        private int executions;

        @Override
        public synchronized Result charge(Request request) {
            Objects.requireNonNull(request);
            Saved previous = saved.get(request.key());
            if (previous != null) {
                if (!previous.request().equals(request)) {
                    throw new IllegalArgumentException("idempotency conflict");
                }
                return previous.result();
            }
            Result result = new Result(Status.APPROVED, "demo-" + (++executions));
            saved.put(request.key(), new Saved(request, result));
            return result;
        }

        synchronized int executions() { return executions; }
    }

    static final class Checkout {
        private final Map<Method, Gateway> gateways;

        Checkout(Map<Method, Gateway> gateways) {
            this.gateways = Map.copyOf(gateways);
        }

        Result pay(Method method, Request request) {
            Gateway gateway = gateways.get(Objects.requireNonNull(method));
            if (gateway == null) {
                throw new IllegalArgumentException("unsupported method");
            }
            return gateway.charge(request);
        }
    }

    static void check(boolean condition, String message) {
        if (!condition) throw new AssertionError(message);
    }

    public static void main(String[] args) {
        FakeGateway card = new FakeGateway();
        FakeGateway wallet = new FakeGateway();
        Checkout checkout = new Checkout(
                Map.of(Method.CARD, card, Method.WALLET, wallet));
        Request request = new Request("order-123-attempt-1", 2500, "EUR");
        Result first = checkout.pay(Method.CARD, request);
        check(first.equals(checkout.pay(Method.CARD, request)), "stable result");
        check(card.executions() == 1, "no duplicate execution");
        try {
            checkout.pay(Method.CARD, new Request(request.key(), 2600, "EUR"));
            throw new AssertionError("conflicting payload accepted");
        } catch (IllegalArgumentException expected) {
            check(card.executions() == 1, "conflict must not charge");
        }
        checkout.pay(Method.WALLET, new Request("wallet-1", 100, "EUR"));
        check(wallet.executions() == 1, "independent strategy");
        System.out.println("PASS: routing, replay, payload conflict");
    }
}
```

运行：`javac --release 17 DesignPrinciplesDemo.java && java DesignPrinciplesDemo`。

逐项理解，而不是只记代码结构：

1. Checkout 持有不可修改的注册表，使用方不直接构造具体网关；main 是 Composition Root。
2. 新增同契约网关主要影响实现与装配，稳定支付编排不必了解 HTTP、签名或供应商异常。
3. FakeGateway 的 synchronized 保护“查重 → 检查请求 → 执行 → 保存”这段整体操作，不能仅用 ConcurrentHashMap 替换 HashMap 就宣称复合流程原子。
4. 锁只适用于本进程该对象。多副本部署、进程重启、外部扣款后本地保存失败都超出示例保证。
5. 同键不同金额必须拒绝，否则“幂等”可能错误重放旧订单的结果。真实契约还需约定幂等键作用域、持久期限、租户隔离和保留策略。
6. 注册表允许同一键在 CARD 和 WALLET 的不同网关分别执行，示例并未保证跨供应商全局去重；生产业务必须在应用层约束支付尝试和切换策略。

**为什么定义 UNKNOWN？**网络超时只说明没拿到响应，不证明没扣款。示例 FakeGateway 不模拟此状态，但生产适配器不得把这种不确定性伪装为 DECLINED 并立即切换另一家扣款。恢复可通过供应商查询、Webhook 和对账，按业务允许的流程推进。

测试分层也必须清楚：单元测试验证编排；针对每个真实适配器运行共用契约测试；集成测试验证请求转换、认证、限流和超时；故障测试验证响应丢失、重复回调和重启恢复。Fake 通过测试不证明真实网关遵守契约。

## 十三、DRY 深入：知识权威、重复验证与跨领域复用

### 13.1 一份知识权威不等于只能执行一次校验

前端校验必填与格式可以改善体验；后端必须重新验证，因为客户端可被绕过；数据库唯一约束处理并发竞态。这三处不是简单删除其中两处就更符合 DRY。应区分：

- 知识来源：订单状态何时允许取消，由领域规则定义。
- 边界防御：客户端提示、服务端校验和数据库约束的失败模型不同。
- 派生表示：API 类型从 Schema 生成，可减少手工漂移，但还需要兼容性检查。
- 实际执行：缓存规则、客户端展示和服务端最终裁决可以分布执行，但必须明确权威和版本。

税务计算案例中，客户端可显示后端提供的报价，并在提交时携带报价标识或版本。服务端重新确认有效期、货币和商品变化；不能信任前端提交的应付金额。具体选择保存报价快照还是重新报价，要看价格有效期与用户确认义务。这比“所有逻辑搬到一个公共库”更接近消除知识漂移。

### 13.2 共享库也会制造耦合

若两个领域都有 status 字段，订单状态和发票状态仍可能有不同含义。抽成 UniversalStatus 后，一个领域添加状态可能迫使所有消费者处理不相关分支。判别方法是问：**这两处未来是否必须因同一个业务决定同步变化？**

共享安全库适合复用稳定算法与协议处理，但权限规则仍可能属于各领域；共享库升级也需要消费者发布，不会自动同步。远程规则服务可以集中版本，却引入运行时依赖和可用性风险。只有必须实时一致的规则才值得付出同步调用成本；允许延迟的规则可版本化分发，并定义过期和回滚策略。

## 十四、KISS 与 YAGNI：把简单性放在完整生命周期中衡量

### 14.1 最短代码可能具有最高运营成本

把支付、邮件和订单保存写成一个20行方法，看起来简单，却隐藏部分成功。反过来，将每一步拆成独立微服务，会增加网络调用、部署协调、幂等、追踪和补偿负担。

例如前文每日5,000条记录的系统，平均吞吐量低不代表没有尖峰，也不代表延迟要求宽松。正确选择还需问批处理窗口、突发流量、数据大小、故障恢复和监管约束。**不能仅凭日总量就判定单体一定足够，也不能用未来可能增长为复杂架构背书。**

选型时列出核心场景：正常处理、重复请求、数据库不可用、通知不可用、重启恢复。逐一比较组件数量、故障传播路径、人工恢复步骤和团队掌握程度。为了减少日常运维，可以接受少量明确重复代码；为了满足审计与恢复，增加持久化状态可能是必要复杂度。

### 14.2 延迟实现未来功能，同时保留便宜的演进空间

假设业务仅在一个国家运营，未来可能进入其他国家。当前可明确金额货币字段、把税务规则隔离于用例、记录业务时间；无需立即实现全球税务引擎、跨区主动主动部署或通用插件语言。

但如果合同已要求三个月后支持第二国家，那已是已知约束，应纳入当前决策。YAGNI 不能用来拒绝已承诺需求、备份恢复、安全或必要测试。区分四种工作：

| 工作 | 判断 |
|---|---|
| 尚无消费者的完整功能 | 通常延迟 |
| 当前业务正确性与安全 | 当前必须满足 |
| 降低后续修改成本的局部整理 | 比较当下投入和修改收益 |
| 昂贵且不可逆的决定 | 先验证约束、保留退出路径 |

不要把所有未来风险都接口化。建立低成本选择权更实际：明确模块依赖、保持可迁移数据格式、提供导出能力、记录 ADR 的触发条件。例如“当两团队需要独立发布且模块边界稳定后重新评估拆服务”，比“将来业务大了再拆”可验证。

## 十五、关注点分离的难点：事务、外部副作用与安全

### 15.1 分层不自动提供一致性

创建订单时，数据库提交成功而邮件失败，应不应该撤销订单？一般应由业务决定；技术分层无法代替这个决策。一个常见教学方案是：在同一数据库事务保存订单与待发送事件（Outbox），由后台处理器投递通知。这样可避免“保存成功但发送意图永久丢失”的窗口。

代价是事件可能重复投递、通知延迟、后台处理失败和积压治理。需要幂等消费者、可观察的重试状态与人工修复路径。Outbox 并不保证外部邮件只发送一次，尤其当邮件已被接收但本地状态未更新时；要利用供应商去重能力或接受并控制重复影响。

本段是模式级方案，不绑定某个框架版本。它也不意味着所有项目必须引入消息队列：数据库中的待处理任务加后台工作者有时已够用，具体取决于规模、隔离和可靠性要求。

### 15.2 跨切面能力不能吞掉业务语义

认证、日志与 Tracing 可通过中间件统一处理，但“当前用户能否取消这一租户的这一订单”需要业务上下文。只在网关确认登录，不代表服务内部已经完成对象级授权。

日志要有请求或业务关联标识，但避免完整支付卡数据、凭证和个人资料；错误转换保留诊断原因，同时屏蔽供应商内部信息。重试也不能作为无差别拦截器：只读查询、幂等扣款与非幂等外部操作具有不同风险。把重试代码集中起来，并不代表所有调用都应采用相同重试策略。

### 15.3 设计异味与可观测证据

| 现象 | 可能的问题 | 下一步验证 |
|---|---|---|
| 改展示模板导致价格测试失败 | 关注点混杂 | 检查共享可变对象及错误依赖 |
| 新供应商要求修改核心用例多处分支 | 扩展点或契约不稳 | 区分公共语义与供应商差异 |
| Fake正常、线上超时出现重复扣款 | 契约/故障模型缺失 | 检查幂等作用域和未知结果处理 |
| 一个通用模块修改迫使所有团队发布 | 错误复用或接口过宽 | 查看消费者实际需要的能力 |
| 类越来越多但一个需求仍跨十几个模块 | 拆分按技术动作而非变化原因 | 用真实变更重新划分边界 |
| 通知失败造成订单接口持续超时 | 副作用耦合与恢复不足 | 结合业务语义评估异步和持久状态 |

这些信号只是排查入口。不要仅凭指标认定根因，也不要把“接口数量增加”“测试覆盖率上升”直接等同于架构质量提升。

## 十六、架构面试追问：从口号进入约束与证据

### 16.1 “有 Spring 注入，就是 DIP 吗？”

**回答：**不是。注入解决对象如何取得依赖，DIP关注高层策略的源码依赖方向。我会检查核心用例是否依赖消费方定义的业务端口，以及供应商类型是否泄漏。构造器注入具体 SDK 客户端仍可能把业务绑定在供应商上。

**追问：接口是否越多越好？**不是。只在明确变化、外部副作用或隔离测试的边界抽象；稳定内部值对象没有必要一一配接口。比较的是替换与测试收益和导航、维护成本。

### 16.2 “支付超时后切换备用供应商，是高可用设计吗？”

**回答：**要先确定是否已产生扣款。超时可能留下未知结果，切换可能导致双扣。我会先定义支付尝试状态、幂等键作用域、查询和对账流程；备用供应商用于哪些失败类型必须明确，高可用不能以破坏资金正确性为代价。

**追问：同一个幂等键不就行了？**供应商A与B通常不会共享去重状态；即使本系统有唯一键，也需考虑扣款成功与本地记录失败的窗口。接口层的抽象不能消除跨系统不确定性。

### 16.3 “两段代码完全一样，为什么允许重复？”

**回答：**我先确认它们是否表达同一个业务决定。订单折扣和员工福利可能当前都乘0.9，但由不同角色、不同时间变化。强行提取 DiscountUtil 容易产生布尔开关和领域泄漏。短期保留独立实现，并在真实共同变化出现时再抽象，可能更便宜。

**追问：如何避免放任复制？**记录规则归属，观察变更，并将必须一致的规则集中或由同一源生成；不是以重复行数为唯一门槛。

### 16.4 “如何证明 LSP，而不是只说子类可替换？”

**回答：**先写出输入条件、结果、不变量、异常和副作用，再让各实现运行同一契约测试。例如同一有效请求不能因具体类型不同突然被拒绝；重复支付请求的语义要一致。测试提供工程证据，不是对任意输入和所有并发调度的形式化证明。

**追问：缓存实现也能用相同测试吗？**只有缓存满足接口的一致性和失效契约时可以。如果缓存允许过期数据，而消费方要求写后读，不能靠改测试掩盖不兼容，应调整契约或实现。

### 16.5 “为了 YAGNI，现在可以先不做灾备吗？”

**回答：**先确认当前 RTO、RPO 和恢复责任。已要求的恢复能力属于当前需求，不是猜测性功能。可以选择成本合适的备份和恢复方案并演练，而不是提前上多地域复杂架构，也不是完全不做恢复。

### 16.6 “如何重构一个巨型 OrderService，又不打断交付？”

**回答：**用特征测试固定关键业务行为，选一个变化频繁且边界清晰的能力；先在同一部署内提取领域规则或适配器，保持数据库和事务语义，再验证真实需求的修改范围是否缩小。避免一边重划模块一边切消息系统和迁移数据库，降低故障归因难度。

**追问：如何衡量成效？**观察同类需求所触及的模块、测试耗时、发布协调和回滚复杂度；用真实变更与故障样本比较，而不是单凭类数或抽象层数评价。

## 十七、扫雷自测与面试前速记

先遮住答案独立作答，再对照本章机制解释。

| 自测 | 应包含的关键判断 |
|---|---|
| 一个用例协调五个对象，是否违反 SRP？ | 看编排与业务知识的职责，不数依赖 |
| 增加实现还需改配置，是否违反 OCP？ | 看关闭的变化轴和稳定流程是否受影响 |
| 子类抛 UnsupportedOperationException 必然违反 LSP？ | 先查父契约是否允许可选操作 |
| 将所有查询拆成单方法接口就满足 ISP？ | 从消费者能力和发布耦合判断 |
| 注入具体数据库客户端是否就是 DIP？ | 注入方式与源码依赖方向是两回事 |
| 前后端都校验金额就违反 DRY？ | 区分权威规则、展示提示与服务端裁决 |
| synchronized 能防止多副本重复扣款？ | 只能保护同对象进程内临界区 |
| 支付超时能否直接当作拒绝？ | 不能把未知外部结果当作明确失败 |
| Outbox是否使通知恰好发送一次？ | 仍需处理重复、丢响应与消费幂等 |
| 每日5,000条数据是否足以确定技术架构？ | 还需峰值、延迟、恢复、安全和团队约束 |

**速记：**SRP按变化来源聚合；OCP为明确变化轴保留扩展点；LSP检查行为契约；ISP按消费者收窄能力；DIP让策略依赖所需抽象；DRY消除知识漂移；KISS比较全生命周期复杂度；YAGNI延迟未需要功能并保留演进空间；关注点分离必须落到数据、事务、失败与权限边界。

## 十八、资料与验证范围

核查日期：2026-11-08。原则性内容不绑定框架版本；完整代码明确以 Java 17 为基线。支付、Outbox、测试与选型部分是本章教学推演，不是公司面试真题或厂商交付保证。

- [Robert C. Martin：The Single Responsibility Principle](https://blog.cleancoder.com/uncle-bob/2014/05/08/SingleReponsibilityPrinciple.html)：用于核对责任与变化原因的概念。
- [Liskov 与 Wing：A Behavioral Notion of Subtyping（1994）](https://www.cs.cmu.edu/~wing/publications/LiskovWing94.pdf)：用于核对行为子类型的契约基础。
- [Martin Fowler：Yagni](https://martinfowler.com/bliki/Yagni.html)：用于核对延迟猜测性功能的含义。

验证记录：已检查原知识点覆盖、示例调用链及 Markdown 围栏；环境有 Java 17 运行时但无 javac，编译执行尝试未能进行，因此本章完整代码尚未获得实际运行验证。其余短代码为局部示意，省略的业务类型需由项目定义。
