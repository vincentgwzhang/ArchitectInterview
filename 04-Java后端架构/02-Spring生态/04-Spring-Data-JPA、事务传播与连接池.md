# 04. Spring Data JPA、事务传播与连接池

> 基线：Java 17+、Spring Boot 3.5.x、Spring Framework 6.2.x、Spring Data JPA 3.5.x、Hibernate ORM 6.6.x。JPA 指 Jakarta Persistence，本文使用 jakarta.persistence 包。数据库隔离、锁和超时必须结合实际数据库/驱动验证；Hibernate 的优化不是 JPA 标准保证。所有场景、参数与数字都是教学示例。

## 1. 先分清五个角色与业务边界

JPA 是对象持久化规范；Hibernate 是其常见实现；Spring Data JPA 提供 Repository 抽象和查询方法；Spring 事务管理协调提交/回滚；连接池复用数据库物理连接。它们解决不同问题，不能用“Repository.save 成功”推断整个业务已经持久提交，也不能用“连接池没满”证明 SQL 没有锁等待。

贯穿案例：库存服务扣减某商品库存，并记录一条库存变更。业务不变量是库存不得为负、重复业务请求不得重复扣减、扣减与变更记录要么同时提交要么同时回滚。单个数据库事务可以保护后两张表的原子写入，但不能自动保护发到外部支付系统的 HTTP 操作。选用 JPA 的理由是对象状态和关系处理方便；代价是 SQL 时机、抓取与刷新行为更隐式，需要开发者理解并观测。

事务边界应围绕应用用例，例如“预留库存”，而不是让每个 Repository 方法各自完成一段业务事务。若查询库存和保存变更分别提交，中间出现竞争或失败就可能破坏不变量。另一方面，也不能把整个 Web 请求、用户思考和外部 API 等待都圈进事务，因为持有的连接、锁和快照会持续占资源。

对复杂报表、批量更新和强 SQL 控制需求，JdbcTemplate、原生 SQL 或 SQL 映射工具可能更直接。一个服务可按用途组合它们，但要明确是否由同一个事务管理器、同一个 DataSource 和兼容的资源访问方式协调；引入两种访问库不自动获得跨库原子性。

## 2. Entity、Persistence Context 与 SQL 的真实时机

### 2.1 Entity 生命周期不是 Java 对象生命周期

Entity 是带持久化身份的对象。新建但尚未关联上下文的对象是 transient/new；由持久化上下文管理的是 managed；脱离上下文后是 detached；被标记删除的是 removed。对象仍在堆中不代表 ORM 仍追踪它，更不代表修改一定写数据库。

Persistence Context（持久化上下文）维护身份映射和托管状态：在同一个上下文中，同一实体类型与主键通常对应同一个托管实例。它帮助保持对象身份一致，常被称为一级缓存；它不是跨请求共享缓存，也不保证查询列表不会访问数据库。

EntityManager 是操作上下文的接口。Spring 常注入一个按当前事务等上下文委托的代理，不应因此把实际 EntityManager 或实体图当作线程安全对象。不要把托管实体交给多个并行任务共同修改；异步线程也不能假设自动继承原线程的事务。

### 2.2 Dirty Checking、flush、commit 各做什么

脏检查（Dirty Checking）检测托管对象相对于已知状态的变化，决定要同步哪些更新。Hibernate 可用状态快照或增强机制辅助检测，具体方式依实现与配置。业务在写事务里加载 managed Entity 并调用业务方法修改后，通常无需再次 save 才能触发更新。

flush 把上下文待同步变更转成 SQL 发送给数据库。它不等于 commit：SQL 已执行但事务仍可能回滚，别的事务能看见什么取决于隔离规则。commit 是结束事务并请求提交；某些约束、网络失败或并发检查可能在 flush/commit 才暴露，因此“方法最后一行没报错”仍不等于调用代理已完成提交。

默认自动刷新模式下，某些查询前也可能 flush，以避免查询读不到自身待写变更。不要假设 SQL 严格按 save 调用顺序立即执行；主键生成策略等也会改变插入时机。saveAndFlush 只是尽早同步，不是“更强的保存”，不能用它替代正确事务边界。

回滚数据库后，内存对象不一定自动恢复成原字段值，持久化上下文也可能不适合继续使用。失败后应结束当前事务，重新加载并在新事务中重新判断业务，不要拿旧 Entity 继续循环保存。

### 2.3 persist、merge 与 Spring Data save

persist 用于使新实体成为托管对象；merge 将传入对象状态合并到托管实例并返回那个实例，传入的 detached 对象本身不因此变成 managed。忽略 merge/save 返回值后继续修改旧实例，可能产生“为什么后续修改没有保存”的问题。

Spring Data save 根据新实体判断走 persist 或 merge；默认判断与版本、主键等有关，手工分配主键时尤其要核对，可通过 Persistable 明确新对象语义。不要把“id 不为空”当作所有实体都已存在的通用证明。

更安全的更新流程是按授权范围加载当前实体、显式应用允许修改的字段、执行业务校验，再交由事务刷新。直接把客户端 JSON 反序列化为 Entity 然后 save，可能把未传字段覆盖为空、修改版本/归属、覆盖并发写入，甚至通过级联扩大影响范围。Public API DTO 应独立于持久化模型。

## 3. 关系映射、抓取计划与 N+1

### 3.1 关系的拥有方、级联与删除要分别推理

关系拥有方决定关联列或关联表的更新，mappedBy 表示反向映射位置，不意味着“这边对象集合改了数据库一定跟着改”。双向关系应在领域方法中同时维护两边内存引用，并修改正确拥有方；否则内存看似正确，刷新后关系仍丢失。

Cascade 是将 persist、merge、remove 等操作传播到关联实体，不是数据库外键的同义词。订单与其独有明细可考虑生命周期级联，订单与共享商品一般不能级联删除商品。orphanRemoval 表达某些父子关系中，孩子从关系移除后应删除；解除关联与删除实体必须符合业务所有权，不能为了方便统一使用 CascadeType.ALL。

equals/hashCode 也影响 Set 中的实体。若 hashCode 随生成主键变化，对象插入 HashSet 后就可能查找失败；若 equals 遍历关联图，可能触发懒加载、递归和高成本。可选择稳定且不可变的自然键，或经过代理/主键生命周期验证的身份策略。不要对 Entity 直接生成包含所有字段的 equals、toString。

### 3.2 LAZY 与 EAGER 描述时机，不承诺 SQL 形状

懒加载（Lazy Loading）在真正访问关联时才获取数据；上下文关闭后访问未初始化关联，Hibernate 可能抛 LazyInitializationException。EAGER 要求及时获取数据，但不保证一定用单条 JOIN，仍可能通过多次查询完成，所以改成 EAGER 不必然修复 N+1。

教学场景：先查 50 个订单，循环中读取每个订单的客户名称；若每次访问都再发一个 SQL，就接近 1+50 次查询。实际次数受重复客户、缓存和抓取策略影响，不能靠一个固定公式断言全部情况。应通过 SQL 计数、请求级查询时间和数据规模变化验证。

N+1 的成本不只是数据库 CPU，还包括多次往返、连接持有时间、对象构造和尾延迟。开发环境只有一条数据时几乎看不出问题，因此测试必须构造多个不同关联对象，避免一级缓存恰好掩盖。

### 3.3 Fetch Join、Entity Graph、Projection 如何选

Fetch Join 在本次查询中一起获取明确需要的关联；Entity Graph 用声明的属性路径表达抓取需求；Projection 直接返回所需字段或 DTO，避免加载完整实体图。它们都应按用例选择，而不是把全系统的关联永久设为 EAGER。

订单列表只展示订单号和客户名，可使用 DTO 查询，减少跟踪成本和敏感字段暴露。订单编辑需要修改托管聚合，可按限定的抓取图加载。Entity Graph 并不承诺任何实现下一定只发一条 SQL，最终仍看实际 SQL 与执行计划。

集合 JOIN 会把父行重复展开，两个集合可能形成行数乘积；ORM 去重父对象也不能免除数据库和网络的重复工作。分页时直接 fetch 多值集合容易导致分页语义和性能问题，不能用 distinct 当作万能修复。可先分页查询主键，再按这些主键批量加载必要关联，并恢复原排序。两次查询之间可能发生变化，若业务要求一致快照，还需选择事务/隔离方案。

批量抓取可减少 N+1，但会增加每次读取的数据量；DTO 则牺牲一些通用实体便利。选择应考虑页面字段、数据规模、更新需求和数据库计划，而不是追求“查询数最少”这一个指标。

## 4. 一段完整的乐观并发更新示例

以下类分别放在 example.stock 包下的同名文件，依赖 spring-boot-starter-data-jpa 和对应数据库驱动，应用需扫描该包。建表通过迁移脚本完成；运行时唯一约束与非负约束应由数据库再次保护。为聚焦版本冲突，本例未实现请求去重表，不能当作完整幂等库存 API。

~~~java
package example.stock;

import jakarta.persistence.Entity;
import jakarta.persistence.Id;
import jakarta.persistence.Version;

@Entity
public class Stock {
    @Id
    private Long id;
    @Version
    private Long version;
    private long available;

    protected Stock() {}

    public Stock(Long id, long available) {
        if (id == null || available < 0) {
            throw new IllegalArgumentException("invalid initial stock");
        }
        this.id = id;
        this.available = available;
    }

    public Long getId() { return id; }
    public Long getVersion() { return version; }
    public long getAvailable() { return available; }

    public void reserve(long quantity) {
        if (quantity <= 0 || quantity > available) {
            throw new IllegalArgumentException("invalid reservation quantity");
        }
        available -= quantity;
    }
}
~~~

~~~java
package example.stock;

import org.springframework.data.jpa.repository.JpaRepository;

public interface StockRepository extends JpaRepository<Stock, Long> {}
~~~

~~~java
package example.stock;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class StockService {
    private final StockRepository stocks;

    public StockService(StockRepository stocks) {
        this.stocks = stocks;
    }

    public record StockView(Long id, long available) {}

    @Transactional
    public StockView reserve(Long id, long quantity) {
        Stock stock = stocks.findById(id)
            .orElseThrow(() -> new IllegalArgumentException("stock not found"));
        stock.reserve(quantity);
        return new StockView(stock.getId(), stock.getAvailable());
    }
}
~~~

调用必须从另一个 Bean 经 Spring 代理进入 reserve。方法内查询得到 managed 实体，reserve 改变字段，事务刷新时生成带版本条件的更新。概念上接近下面的 SQL，实际列名、参数顺序与生成方式依 Hibernate 和数据库：

~~~sql
UPDATE stock
SET available = :newAvailable, version = :newVersion
WHERE id = :id AND version = :oldVersion;
~~~

若两个事务读到相同版本，先成功更新并提交的一方推进版本；另一方的旧版本条件无法完成预期更新，触发乐观锁冲突。异常可能发生在刷新或代理提交阶段，Spring 常转换为相应数据访问异常；不能只在方法内部 stock.reserve 周围 catch，就保证接住提交错误。

StockView 刻意不返回 version，因为业务方法返回时最终刷新时点尚未完成，不能假设此时内存版本就是客户端应使用的提交后版本。本例用 IllegalArgumentException 简化错误类型，生产需区分不存在、库存不足和输入错误，并在 API 边界映射稳定状态，不泄露内部异常。

若客户端重试相同预留请求，即使版本锁工作正常，第一次提交后的第二次请求仍可能再次扣减。版本锁防并发覆盖，幂等键防重复业务执行，它们不是同一保证。完整方案应在同一数据库事务内记录唯一业务请求键与结果；遇到重复键读取既有结果，并校验同键是否对应相同请求内容。

## 5. @Transactional 的代理、回滚与用例边界

### 5.1 注解生效取决于调用经过什么

常见代理模式下，调用进入代理，事务拦截器按属性获取事务，再执行目标方法，最终提交或回滚。同一对象中 this.reserve 绕过代理，不会为内部方法重新应用传播规则；若外层已有事务，内部操作可能仍在外层事务中，不能笼统说“完全没有事务”。

方法可见性需要版本边界：Spring 6 的类代理默认可支持 protected/包可见事务方法，接口代理则要求 public 接口方法；private 方法不能通过普通代理覆盖，final 方法也限制类代理增强。因此“所有非 public 事务都无效”并不准确。工程上使用明确的 public 应用服务入口、跨 Bean 调用，更容易审查和测试。

构造器与初始化阶段、直接 new 服务对象、切换到自行启动的线程，均不能照搬正常代理入口假设。需要短小动态事务边界时，可使用 TransactionTemplate 显式包住工作；但它也不让跨线程、跨数据库自动加入同一事务。

### 5.2 默认回滚规则与吞异常的后果

通常默认 RuntimeException 和 Error 导致回滚，受检异常（Checked Exception）默认不触发回滚；可用 rollbackFor 指定类型。Spring 6.2 还支持全局默认回滚策略调整，所以实际系统应查配置，不能只背注解默认值。尽量按异常类型配置，不用容易误匹配的字符串规则。

回滚取决于拦截器看到的结果和事务状态。业务 catch 异常然后返回正常值，可能使本应失败的操作被提交；但若底层已将事务标记为 rollback-only，即使 catch 后返回，提交仍会失败。吞异常既不保证提交，也不保证已恢复，必须说明底层状态。

例如订单用例调用另一个 REQUIRED 服务，内部因数据库错误标记同一事务回滚；外层捕获错误并继续保存“成功记录”，最后试图提交时可能收到 UnexpectedRollbackException。它是提醒调用者事务实际没有提交，不能用 noRollbackFor 试图修复一个数据库已经判定失败的事务。

### 5.3 readOnly、timeout 与远程调用

readOnly 是事务意图和优化提示，具体如何影响刷新、连接与数据库取决于事务管理器/实现。它不等于数据库用户只读权限，也不是业务授权。不能依赖 readOnly=true 防止恶意写操作，应以数据库权限和应用规则保护。

事务 timeout、连接获取 timeout、SQL 执行 timeout、HTTP 总 deadline 属于不同层。事务超时不是强制终止任意 Java 代码的闹钟；驱动与管理器如何执行要验证。把 HTTP 请求超时改短，也不能证明数据库已撤销 SQL。

不要在持有写事务/锁时等待支付商。可先短事务保存 PENDING 状态及 Outbox 事件，提交后异步调用外部系统，再以幂等处理结果推进状态。Outbox 是与业务数据同事务写入的待发送记录，由独立发布者重试发送；它解决提交与可靠发布窗口，但不保证消息恰好一次，消费者仍需去重。

## 6. 事务传播：逻辑范围与物理资源

### 6.1 各传播模式真正约束什么

| 模式 | 无现有事务 | 有现有事务 | 典型用途/风险 |
|---|---|---|---|
| REQUIRED | 新建 | 加入 | 同一用例原子执行，内层失败可影响外层 |
| REQUIRES_NEW | 新建 | 挂起外层，新建独立事务 | 独立审计，额外连接与锁等待风险 |
| MANDATORY | 报错 | 加入 | 明确要求调用方提供事务 |
| SUPPORTS | 无事务执行 | 加入 | 允许两种语义的读取，不能假定永远无事务 |
| NOT_SUPPORTED | 无事务执行 | 挂起后无事务执行 | 明确隔离某段非事务操作，注意一致性 |
| NEVER | 无事务执行 | 报错 | 强制禁止调用时存在事务 |
| NESTED | 通常新建 | 保存点式嵌套，依管理器支持 | 局部回滚，不是独立提交 |

传播回答的是“已有事务时如何参与”，不是隔离级别。REQUIRED 内层声明更强隔离不一定重建已有物理事务，通常继承外层；可通过验证现有事务属性的配置拒绝不兼容加入。不要在内层贴一个 SERIALIZABLE 就推断整个调用链得到可串行化。

### 6.2 REQUIRES_NEW 为什么可能卡死连接池

挂起不是提交外层事务：外层已经借出的连接、持有的锁仍可能保留。内层独立事务通常还要借另一连接。教学场景有 10 个请求，各持一个外层连接，池最多 10 个；它们同时调用独立审计事务，全部等待新连接，而外层只有等内层返回才释放。最终可能直到借连接超时才打破等待。

简单增加一个连接可能让某些模式逐个推进，但不等于通用容量公式，更不能解决数据库锁依赖。如果内层更新外层尚未提交锁住的同一行，独立连接可能等待外层释放，而外层又等待内层结束。应优先避免互相依赖的独立事务，减少嵌套并评估容量，而不是无限增池。

独立审计还有业务含义：外层回滚后审计仍可能提交，这是需要的“尝试记录”，不能称为“成功记录”。内层看不到外层未提交的新行时，还可能外键失败。设计审计字段应保存稳定请求标识与尝试结果，不能不加区分引用未提交业务状态。

### 6.3 NESTED 不是 REQUIRES_NEW 的廉价替代品

NESTED 常依赖 JDBC 保存点，在同一物理事务里回滚一段；外层整体回滚，内层结果也会消失。REQUIRES_NEW 独立提交，外层回滚不自动撤销它。两者故障语义不同，不能只比较连接数。

JpaTransactionManager 下的保存点支持不等于 JPA 实体状态能正确回退。JPA 上下文可能仍保存保存点回滚前的修改，且具体支持受管理器/驱动限制。若需求主要是 JDBC 局部回滚，应明确选用兼容的事务访问路径并测试；不要凭注解名称承诺实体级嵌套事务。

## 7. 隔离、乐观锁与悲观锁

### 7.1 先把并发异常讲成时间线

丢失更新（Lost Update）：两个事务读相同旧值，各自计算并覆盖，后写吞掉前写。不可重复读（Non-repeatable Read）：同一事务再次读某行，发现已被别的提交修改。幻读（Phantom）：按条件再次查询，匹配的行集合变化。写偏差（Write Skew）：两个事务分别检查跨行约束，各自写不同的行，单行更新无冲突但共同破坏约束。

隔离级别给出的保证与数据库实现有关。例如可重复读下是否通过快照避免某类幻象、是否仍可能写偏差，不能把数据库 A 的行为当作所有 JDBC 数据库的承诺。串行化可能通过阻塞或检测冲突中止实现，应用仍要准备事务级有限重试。

应先定义不变量：库存非负可用单行版本/条件更新；“至少一名医生值班”跨多行，单行 @Version 不自动保护所有相关行。可选择锁住共同协调行、数据库约束/可串行化事务等，按并发与成本取舍。

### 7.2 乐观锁如何处理失败

@Version 让更新验证读到的版本，适合冲突相对少、业务可重新判断的场景。冲突后应在新事务重新加载、重新校验，不是在已失败的事务里反复 save 同一个实体。

重试必须有上限和退避，并确认没有重复外部副作用。若用户编辑基于旧页面，常应返回冲突让用户确认，自动覆盖重试反而违反用户意图。对库存热点，大量乐观冲突可能浪费查询与计算，应比较原子条件更新或短时悲观锁。

原子条件更新可表达：

~~~sql
UPDATE stock
SET available = available - :quantity, version = version + 1
WHERE id = :id AND available >= :quantity;
~~~

必须先验证 quantity > 0，再检查受影响行数。0 行需要区分不存在与不足；批量/原生更新可能绕开上下文快照，之后要清理/刷新相关实体，且所有写路径需保持版本策略一致。这是 SQL 语义示意，非跨数据库通用的完整 Repository 实现。

### 7.3 悲观锁适合短临界区，不适合长业务流程

悲观写锁通常让数据库协调冲突访问，如基于 SELECT ... FOR UPDATE 的机制，实际 SQL 依方言。适合冲突较高且临界区短的资源扣减，但锁持有到事务结束，慢外部调用会延长所有竞争者等待。

死锁可能来自两个事务按不同顺序锁 A/B。统一访问顺序、缩短事务、为过滤条件建立合适索引，可减少锁范围和冲突。数据库检测死锁后可能中止某一事务；应用要识别可重试错误，在新事务中重新执行，不把任意 SQL 异常都当可重试。

锁只保护共同遵循数据库协议的访问。多实例 Java synchronized 不能代替数据库并发控制；悲观锁也不保证 HTTP 请求不重复，更不会让外部支付参加本地事务。

## 8. 连接池：容量、等待与超时预算

### 8.1 连接是有限工作许可

连接池维护可借用连接，复用建连、认证与会话成本。借连接不等于执行 SQL，持有连接也可能在等锁、传输结果或等待应用。close 对池包装连接通常表示归还池，不是每次断开物理连接；手写 JDBC 仍需 try-with-resources 正确归还。

总预算要计算所有实例、每实例多个 DataSource、批处理、迁移工具、监控和滚动发布时新旧实例重叠。教学假设数据库安全应用预算 160，10 个实例各最多 12 条连接，滚动发布额外 2 个实例则可到 144，剩余空间还要留给其他使用者。数据库配置最大连接 200 不代表同时运行 200 条重查询仍安全。

平均并发可粗估为到达率乘平均持有时间：每秒 100 次事务、每次平均持连接 0.1 秒，平均约需 10 条在用连接。这不是建议池大小等于 10，因为峰值、长尾和故障会改变等待；它帮助定位“为什么慢查询让连接需求突然上升”。加池可能把应用排队变成数据库排队。

### 8.2 HikariCP 参数各管哪一段

~~~properties
spring.jpa.open-in-view=false
spring.datasource.hikari.maximum-pool-size=12
spring.datasource.hikari.connection-timeout=1000
spring.datasource.hikari.validation-timeout=500
spring.datasource.hikari.max-lifetime=1500000
~~~

这里只演示 Boot 3.5 使用 HikariCP 时的属性，时间数值单位按对应属性文档为毫秒，不是通用生产推荐。connection-timeout 限制等待从池借连接的时间，不是 SQL 超时；validation-timeout 限制存活校验；max-lifetime 影响物理连接生命周期，不是强制取消长事务。应让生命周期与数据库/代理关闭空闲或老连接的策略协调，且不要期待它自动中断正在使用的连接。

还需分别配置驱动连接/网络超时、SQL/锁等待上限和整体请求预算，确保连接等待不会吃掉全部 deadline。minimum-idle 影响预备空闲连接与扩容行为；是否使用固定规模应看负载和数据库总预算，不必为省几条空闲连接引入突发建连延迟。

泄漏检测阈值用于报告长时间未归还的连接线索，不会替业务修复泄漏，也不能把每条告警都认定为忘了 close。合法长事务、锁等待和慢结果消费也会触发，应沿借出堆栈与归还时间调查。

### 8.3 连接池耗尽的排查顺序

先看 active、idle、pending/wait、获取超时、借出持续时间与事务时长；再关联数据库会话状态、运行 SQL、锁等待、CPU 和 I/O。active 高但数据库 CPU 低，可能在等锁、等应用或等网络，不应立即增连接。

检查发布是否增加实例或新建了额外池，排除全局连接预算被突破。再查慢 SQL、长事务、REQUIRES_NEW 嵌套、流式结果未关闭与手写 JDBC 泄漏。如果查询本身很快而事务持续很久，重点检查事务内 HTTP、文件处理和循环。

恢复时先限制流量/并发并缩短无效占用，修复慢查询或锁依赖；单纯把池翻倍可能压垮数据库。修复后用同样故障条件复测，确认等待和事务持有时间下降，不能只看超时暂时少了。

## 9. Open Session in View、DTO 与批处理

Open Session/EntityManager in View 让持久化上下文延伸到 Web 渲染/序列化阶段，可能在业务事务结束后继续懒加载。它方便简单页面，却把查询藏进输出过程，造成 N+1、不明确的读取快照和额外连接需求。开放上下文不等于整个 HTTP 请求都在同一个数据库事务里。

显式关闭后，在服务事务内根据用例抓取、映射为 DTO，再返回 Web 层。这样边界更清楚，但必须补齐抓取设计；不能只关闭后把 LazyInitializationException 当成框架不稳定。API DTO 只包含允许对外展示字段，避免序列化意外访问巨大图、双向循环或敏感字段。

批处理也要避免一个事务加载百万实体：上下文会保留实体和跟踪状态。可按可恢复批次处理，周期性 flush/clear，并注意 clear 会使实体 detached；不要在 clear 后继续假设此前对象受跟踪。分批提交改变原子性，需记录批次游标与幂等进度，失败后能够重启。

批量 JPQL/SQL 更新直接作用数据库，可能绕过逐实体回调、版本检查和上下文同步。它适合明确的大规模操作，但要核对审计、并发策略与缓存失效，不能把批量方法视作逐个 save 的等价加速版本。

## 10. 完整面试回答与追问

### 10.1 “JPA 的性能问题怎样排查？”

可表达的回答：“我按用例采集 SQL 次数、耗时、结果行数和事务持连接时间，先确认是 N+1、单条慢查询还是对象图过大。列表读取可用 DTO Projection，编辑用例按需 Fetch Join 或 Entity Graph；集合抓取与分页分开设计，避免行乘积。关闭隐式 Web 懒加载后在服务边界映射 DTO，并通过真实数据规模验证执行计划和连接等待，而不是把全部关系改为 EAGER。”

追问：“一条 SQL 是否一定比十条好？”答案：不是。一条连接多个集合可能返回巨大笛卡尔展开，网络和组装成本更高。目标是满足延迟、资源与正确性预算，不是单独最小化查询条数。

### 10.2 “为什么 catch 异常后事务仍回滚？”

可表达的回答：“外层捕获 Java 异常不会清除物理事务的 rollback-only 状态。内层 REQUIRED 与外层共享资源，内层失败可能已经使事务不能提交；最终代理提交时报告 UnexpectedRollbackException，避免调用者误以为成功。我会终止失败用例，在新事务中执行允许独立的记录或重试，同时确保独立记录不被误称为业务成功。”

追问：“换 REQUIRES_NEW 就行吗？”答案：只有独立提交符合业务意图时才合理。它可能多借连接、等待外层锁，也看不到外层未提交数据；必须重新设计数据依赖与容量，不能作为通用异常补丁。

### 10.3 “怎样保证库存不超卖？”

可表达的回答：“我先定义单商品库存非负与重复请求不重复扣减。可用 @Version 在更新时检测竞争，并在新事务有限重试；热点可比较原子条件更新或短悲观锁。扣减与预留记录同事务提交，业务请求键有数据库唯一约束。远端通知通过 Outbox 可靠发布，重复消息按业务键去重。单行版本不自动保护跨商品总额度，也不提供远端原子提交。”

追问：“查询后在 Java if 检查数量为什么不够？”答案：两个事务可以同时读到相同余量，检查都通过。必须使检查与写入由数据库并发协议协调，例如带版本或余量条件的更新，或持锁后检查。

### 10.4 “连接池越大越好吗？”

可表达的回答：“池是数据库工作许可，大小要按数据库安全并发和所有实例总预算分配。我会先测持有时间与等待，排除长事务、锁、泄漏和嵌套新事务，再调整池。扩容实例和滚动发布都要算进去；过大的池可能把应用排队转移到数据库，反而拉高尾延迟。”

追问：“CPU 不高为什么连接满？”答案：连接可以在等锁、网络或事务内慢 API，不一定在计算。应看会话等待与借出堆栈，不能用 CPU 低推断数据库有无限余量。

## 11. 扫雷自测与推理答案

### 11.1 saveAndFlush 返回后是否可以告诉第三方业务已提交？

不能。flush 发送 SQL，但事务仍可回滚，提交还可能失败。可靠外部通知宜用同事务 Outbox 并在提交后发布；只靠提交后内存回调也有进程崩溃丢通知的窗口。

### 11.2 merge 后修改传入 detached 对象，为什么没有写入？

merge 返回托管副本，原对象仍 detached。后续修改原对象不必然受脏检查跟踪。应使用返回实例或更清楚地加载当前托管实体、应用允许字段，避免整图合并的覆盖与级联风险。

### 11.3 全部 EAGER 能否避免 N+1 与懒加载异常？

它可能减少某些延迟访问，却不保证单 SQL，还可能扩大图与额外查询。正确方法是用例级抓取/投影和事务内 DTO 映射，实际测查询数、结果规模及执行计划，而不是全局提前加载。

### 11.4 外层 REQUIRED 内层 REQUIRED 设置更高隔离，为何未实现目标？

两者通常加入同一物理事务，隔离已由外层决定；内层声明不自动重建事务。应在真正创建事务的边界设置合适属性，或启用不兼容属性验证；不能为“升级隔离”随意独立提交破坏原子性。

### 11.5 池有 10 条连接，10 个外层事务都调用 REQUIRES_NEW，会怎样？

若外层均已占连接且内层也需要连接，就没有可借连接；所有请求可能等到超时。外层挂起不会释放已有锁/资源。要减少嵌套独立事务、核对业务依赖并重新预算，不能只无限加池。

### 11.6 @Version 是否同时解决请求重复和跨行约束？

不能。它主要检测参与版本协议的实体竞争。相同业务请求两次顺序执行仍可两次成功；跨行条件也可能写偏差。分别需要幂等请求键/结果记录以及覆盖完整不变量的锁、约束或隔离策略。

### 11.7 关闭 OSIV 后序列化报错，应重新打开还是改服务？

应先补服务层抓取与 DTO 映射，使输出不再依赖已关闭上下文。是否保留 OSIV 是明确的边界决策，不能仅以消除异常为由保留隐式 SQL；也不能关闭后不做配套测试。

### 11.8 乐观锁失败后在原事务循环 save 是否可靠？

不可靠。当前事务/上下文可能已失败，旧实体状态也不能代表最新数据库。应结束失败事务，在外层重试协调器中新建事务、重读并重新判定，限制次数；用户编辑冲突有时应直接告知，不应自动覆盖意图。

### 11.9 readOnly=true 是否能保护数据库免受写入？

不能视作权限控制。它主要表达意图与可能的优化，具体限制取决于实现。需要数据库账号权限、应用授权和测试共同约束；不得通过只读标记绕过安全设计。

## 12. 验证方案与执行边界

对库存示例，验证合法扣减、非正数量、余量不足和不存在；再用两个独立事务与同步屏障确保读到相同版本，让竞争真实发生，断言至多一方按旧版本成功、失败方无部分写入，最终库存满足不变量。不能用一个测试事务里的顺序调用声称验证了并发。

事务测试覆盖外部代理调用与同类内部调用、受检/非受检异常、REQUIRED 的 rollback-only、独立事务外层回滚后记录仍保留。提交阶段异常要通过显式结束事务观察；测试框架默认回滚可能掩盖提交时约束。数据库约束和锁测试使用与生产相同数据库引擎/主要版本，内存替代库不能证明同样隔离语义。

查询测试准备多个不同客户和多条明细，清理一级上下文与相关缓存后统计 SQL；验证序列化不新增查询、分页稳定且大图不会爆炸。连接池实验用小池重现独立新事务等待，并设置足够短但明确的借连接超时，避免测试无限挂起。监控借出、等待、SQL 和事务时长，区分结果与推断。

本次 Java、SQL、Spring 事务及真实数据库并发实验**未实际运行**。已进行原主题覆盖、逐节自足性、版本边界、代码与失败语义、Markdown 围栏与来源链接的人工核对；这些检查不等同于编译/集成测试通过。落地应补齐建表迁移、错误类型、授权、幂等表与目标数据库实验。

## 13. 面试前速记

- Entity 与 DTO 分离，managed 才受跟踪，merge 返回对象与原 detached 对象不同。
- save、flush、commit 是不同阶段；SQL 发出不代表事务已提交。
- 抓取按用例，EAGER 不承诺单 SQL，集合 JOIN 要防行乘积与分页陷阱。
- 事务放应用用例，代理入口与实际管理器决定语义，失败后不要复用坏事务。
- REQUIRED 共享物理事务，REQUIRES_NEW 独立但占额外资源，NESTED 受保存点与实现限制。
- 隔离保护完整不变量，版本锁防覆盖，幂等键防重复，三者不能互相替代。
- 总连接预算包括实例扩容、多个池和滚动重叠；先治长占用，再评估池大小。
- OSIV 延长上下文而非自动延长原事务；服务内加载 DTO，让查询边界显式。
- 数据库原子性不跨 HTTP，可靠通知需要 Outbox 等协议与消费者去重。

## 14. 一手来源与适用版本

以下来源用于核验与可选追溯，正文已解释对应机制。滚动文档可能更新，落地按依赖锁定版本再次检查。

- [Spring Data JPA 3.5：Transactionality](https://docs.spring.io/spring-data/jpa/reference/3.5/jpa/transactions.html)：Repository 与应用用例事务边界。
- [Spring Framework 6.2：Transaction Propagation](https://docs.spring.io/spring-framework/reference/6.2/data-access/transaction/declarative/tx-propagation.html)：逻辑/物理事务、独立事务与保存点。
- [Spring Framework 6.2：Using @Transactional](https://docs.spring.io/spring-framework/reference/6.2/data-access/transaction/declarative/annotations.html)：代理调用及方法可见性。
- [Spring Framework 6.2：Rollback Rules](https://docs.spring.io/spring-framework/reference/6.2/data-access/transaction/declarative/rolling-back.html)：默认与自定义回滚规则。
- [Hibernate ORM 6.6 User Guide](https://docs.hibernate.org/orm/6.6/userguide/html_single/)：实体状态、刷新、抓取、版本与锁；实现优化不作为跨厂商保证。
- [HikariCP 官方项目与配置说明](https://github.com/brettwooldridge/HikariCP)：连接池参数与超时意义；本文不依赖滚动版本的默认池大小。
