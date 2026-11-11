# 03. Spring MVC、WebFlux 与请求处理模型

> 基线：Spring Framework 6.2.x、Spring Boot 3.5.x；虚拟线程部分要求 Java 21+。Reactor 示例使用这些版本所支持的常见 API，不依赖某个调度器默认线程数。场景与容量数字均为教学假设，不代表真实项目或基准测试。

## 1. 先区分阻塞、异步与响应式

阻塞（Blocking）指调用线程必须等待结果，例如 JDBC 查询未返回时，线程暂时无法处理其他任务。非阻塞（Non-blocking）指没有数据时不占住线程等待，由就绪事件通知后继续工作。异步（Asynchronous）指结果在之后交付：把阻塞查询交给线程池也是异步，但查询仍占用池中的线程。因此“返回 Future”不等于底层非阻塞。

响应式流（Reactive Streams）另外定义了生产者与消费者之间的协议：消费者订阅后通过 request(n) 表达还能接收多少元素，生产者按需求发出数据，并最终完成或报错。Reactor 的 Mono 表达零或一个结果，Flux 表达零到多个结果。它们是计算描述，不是结果已经完成的证明，更不意味着每个操作自动创建线程。

面试判断的核心是：等待发生在哪里，哪个资源在等待期间被占用，超过容量后在哪里排队，以及失败如何终止。MVC 与 WebFlux 都可以构建正确系统，也都可能因无限排队、阻塞依赖和错误重试而过载。框架选择无法取消下游数据库的容量边界。

例如商品详情请求并行读取库存和价格：同步写法可能顺序等待两次；异步可以让等待重叠，但总调用数仍是两次。若每个请求额外扇出十个子请求，即使使用非阻塞，连接、下游 QPS 和响应缓存仍可能先耗尽。应先减少不必要调用，再讨论承载等待的线程模型。

## 2. MVC：一次请求如何走过 Servlet 栈

### 2.1 正常同步请求的执行链

典型 MVC JSON 请求由容器接收，经过 Servlet Filter 链进入 DispatcherServlet。Filter 可以在进入 Spring MVC 之前处理认证、安全头、日志等；DispatcherServlet 是统一分派入口，不负责实现所有业务逻辑。

HandlerMapping 根据路径、HTTP 方法和媒体类型寻找处理器；HandlerAdapter 负责按约定调用它。调用前，参数解析器把路径、查询参数或请求体转换为方法参数，数据绑定和校验检查其合法性。Controller 调用业务服务，返回值由返回值处理器和消息转换器序列化为响应，例如 JSON。异常则由异常解析机制寻找适当映射，而不是任意异常都直接把堆栈发给客户端。

这解释几个常见现象：请求没有进入 Controller 也能被拒绝，因为认证、请求体解析或参数校验可能先失败；没有匹配的 HTTP 方法不是业务 Service 出错；JSON 序列化异常可能发生在业务成功之后。排障应按阶段定位，不能把所有 4xx/5xx 都归因于 Controller。

HandlerInterceptor 位于 MVC 处理器周围，可用于业务无关的处理器级观察，但不能替代完整安全过滤链。异步分派时还要考虑再次进入拦截器和 Filter 的条件，日志不得重复统计一次请求为两次。

### 2.2 每请求一线程的含义与边界

传统同步 Servlet 请求通常在一个容器工作线程中执行；调用 JDBC 或阻塞 HTTP SDK 时，该线程也在等待。线程池有上限，过多并发会进入队列或被拒绝。平台线程还占栈等内存，因此不是把线程数调到几万就一定提高吞吐。

不过 MVC 不等于“永远只能同步”。Callable、DeferredResult 等异步返回方式允许容器线程先释放，完成后再分派输出响应；执行 Callable 的其他线程仍可能阻塞。MVC 也可以使用 WebClient 调用外部服务，不能由客户端类名判断整个服务器变成 WebFlux。

响应写入仍有自己的容量和容器行为，尤其慢客户端和流式输出不能只看 Controller 已经返回。异步超时应设置，异步执行器必须有界并监控队列；只把等待从容器池搬到一个无界业务池，会把线程耗尽变成内存积压。

### 2.3 虚拟线程改善什么

Java 虚拟线程（Virtual Thread）让大量等待型任务以较低线程管理成本保留顺序代码。Boot 3.5 在 Java 21+ 可通过 spring.threads.virtual.enabled=true 启用其支持的虚拟线程集成，但具体组件是否采用它仍需按目标服务器和执行器核实。自建线程池、第三方 SDK 和连接池不会因此自动改变。

关键资源仍需配额：数据库最大连接、支付接口并发、文件描述符和请求内存。若数据库只允许 50 个连接，让 5000 个虚拟线程同时抢连接，只会得到大量等待者。需要入口并发限制、池等待超时和请求截止时间，不能通过廉价线程掩盖过载。

CPU 密集计算没有因为等待线程便宜而变快，应限制并行度以匹配 CPU。Java 21 下还需评估某些同步/本地调用导致载体线程无法释放的情况；后续 JDK 对锁相关固定问题有变化，不应把 Java 21 的结论泛化到所有版本。用目标 JDK、实际依赖和负载验证，比只凭“已开虚拟线程”可靠。

## 3. WebFlux：事件循环与订阅链怎样协作

### 3.1 事件循环不是为每个请求开一个线程

WebFlux 可以运行在支持非阻塞的服务器上，Reactor Netty 是常见组合，但 WebFlux 并不等于 Netty。事件循环（Event Loop）用少量线程处理许多连接的就绪事件，每次回调应快速结束，将等待交还网络机制。

Controller 返回 Mono 后，框架负责订阅并把结果写到响应。业务链通过 map 转换值，通过 flatMap 组合异步结果；等待网络响应时不要求某线程一直停在那个调用点。因此线程节省主要来自等待复用，而不是消灭 CPU 工作。

反例是事件循环回调里执行 JDBC、Thread.sleep、文件读取或阻塞 SDK。一个回调占住线程时，与该线程关联的其他连接也无法及时推进。表现可能是很多不相关接口一起出现尾延迟，而不是只有那条慢 SQL 的请求慢。线程名只是线索，应结合线程栈和实际耗时验证。

### 3.2 哪些写法只是把阻塞藏起来

~~~java
// 错误示意：调用方法时已经执行查询，Mono 并没有把它变成异步。
Mono<OrderView> result = Mono.just(repository.load(id));
~~~

Java 会先计算方法参数，再调用 just。因此 repository.load 发生在创建 Mono 时。即便换成 map，若其执行线程是事件循环，阻塞仍然在那里。惰性执行与正确调度缺一不可。

桥接遗留依赖时可采用以下局部写法，假设 repository.load 是同步方法，返回非 null 的 OrderView；业务“不存在”用明确领域异常表达：

~~~java
Mono<OrderView> result =
    Mono.fromCallable(() -> repository.load(id))
        .subscribeOn(blockingScheduler)
        .timeout(Duration.ofMillis(800));
~~~

fromCallable 延迟查询直到订阅；subscribeOn 把源的订阅/执行安排到专门调度器；timeout 限制这条流等待结果的时间。这里 blockingScheduler 必须是应用统一管理、容量受控的 Scheduler，而不是每次请求临时创建。一个可用于教学的 Bean 是：

~~~java
@Bean(destroyMethod = "dispose")
Scheduler blockingScheduler() {
    return Schedulers.newBoundedElastic(16, 64, "legacy-db");
}
~~~

所需导入分别为 reactor.core.publisher.Mono、java.time.Duration、org.springframework.context.annotation.Bean、reactor.core.scheduler.Scheduler、reactor.core.scheduler.Schedulers。上面的 16 是线程上限，64 是该工厂队列参数，并非“全服务只能排 64 个请求”的承诺；具体排队语义要按 Reactor 版本验证。因此仍需独立入口准入控制和拒绝策略。容量数字只是示意，应与数据库连接预算配合。

timeout 发出错误并取消上游订阅，不保证 JDBC 驱动立即停止查询。还需要 SQL/驱动超时、连接池等待上限，且必须观察取消后真实资源是否释放。桥接的代价是多了调度、排队和上下文传播；若大部分业务本来就是阻塞，MVC 往往更容易维护。

### 3.3 subscribeOn、publishOn 与并行度

subscribeOn 主要影响向源订阅的执行位置，适合把阻塞源移走；publishOn 从其位置开始把后续信号处理切换到指定 Scheduler。不能把 publishOn 随意放在链尾，就期待前面已经执行的阻塞查询被移动。

flatMap 可以让多个内部异步操作同时在途，合并结果时不保证原顺序。限制 concurrency 参数是限制同时订阅数量的一种手段；需要按序输出可考虑 concatMap，但它减少并发，慢项会拖住后续项。若业务要求保序又希望并行，还要评估结果缓冲成本，不能把“保序”和“无限并发”同时免费获得。

在 Controller 内手动 subscribe 通常是错误信号：它创建与 HTTP 生命周期脱离的订阅，框架不知道何时完成、怎样传播错误和取消。一般返回组合好的 Publisher，由框架订阅。真正后台任务应交给有持久化、重试和停机语义的任务机制，不能把一次裸 subscribe 当作可靠消息队列。

## 4. 一个端到端非阻塞聚合示例

假设以下 Controller 属于 WebFlux 应用，依赖 spring-boot-starter-webflux，包内有一个配置好的 WebClient Bean：baseUrl 指向受控测试服务。Inventory 与 Price 响应非空且 JSON 结构与 record 匹配。本例两条下游请求均为查询，不承担扣款副作用。

~~~java
import java.time.Duration;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.reactive.function.client.WebClient;
import reactor.core.publisher.Mono;

@RestController
public class ProductController {
    private final WebClient client;

    public ProductController(WebClient client) {
        this.client = client;
    }

    public record Inventory(int available) {}
    public record Price(long cents) {}
    public record ProductView(String id, int available, long cents) {}

    @GetMapping("/products/{id}")
    public Mono<ProductView> product(@PathVariable("id") String id) {
        Mono<Inventory> stock = client.get()
            .uri("/inventory/{id}", id)
            .retrieve()
            .bodyToMono(Inventory.class)
            .switchIfEmpty(Mono.error(
                new IllegalStateException("inventory returned empty body")));

        Mono<Price> price = client.get()
            .uri("/prices/{id}", id)
            .retrieve()
            .bodyToMono(Price.class)
            .switchIfEmpty(Mono.error(
                new IllegalStateException("price returned empty body")));

        return Mono.zip(stock, price)
            .map(t -> new ProductView(
                id, t.getT1().available(), t.getT2().cents()))
            .timeout(Duration.ofMillis(800));
    }
}
~~~

zip 需要两边各有一个值才能形成产品视图，空完成并不等于错误；显式 switchIfEmpty 防止“下游返回空体，聚合接口却静默空完成”。retrieve 默认对错误 HTTP 状态产生异常，但业务还应按约定区分找不到商品、下游拒绝和协议错误；不能把所有错误都转换成空对象或 200。

两边可以在订阅后同时在途，因此整体等待常由较慢者支配，但连接获取、DNS、序列化和调度仍有成本，不能保证恰好等于最大耗时。总流超时不是完整网络配置：WebClient 所用连接器仍应设置连接获取、建立连接、读取等相应上限，并限制连接数量和等待队列。

若库存失败可以展示“库存暂不可用”，降级应该在库存子流中产生带状态的领域结果，不能直接填 0。0 表示确定无库存，与不知道库存不同。价格不可用可能必须整体失败，这由业务决定。对聚合整体 onErrorReturn 一个“正常商品”会掩盖来源和错误范围。

一边失败或总超时时，其他订阅会被取消，但取消是否抵达远端、远端是否停止工作不受这里保证。对写操作必须有幂等键、状态查询与补偿设计。非阻塞只改变等待方式，不提供分布式事务。

## 5. 请求生命周期中的工程边界

### 5.1 限制输入要覆盖字节、结构和展开成本

请求大小限制保护内存与解析成本。Content-Length 可能不存在，也不能单独代表解压后的大小，因此入口代理、服务器与应用解析层要协调。JSON 深度、数组长度、批处理条数、分页上限和上传文件大小都要有边界。

WebFlux codec 的内存聚合上限限制某些解码/聚合操作，不是全服务器总内存预算；流式读取也可能被后面的 collectList 重新全部收集。MVC 则要避免把大文件一口气读成 byte[]。两种模型都应在读取过程中执行上限，超出后明确返回合适错误并释放资源。

安全校验与业务校验分层：认证（Authentication）确认身份；授权（Authorization）检查该身份能否访问这一个资源；参数校验确认输入结构和范围。合法 UUID 不代表用户拥有这个订单，反之授权也不能代替数量范围检查。不要先查询所有数据再在返回端过滤越权记录。

### 5.2 Deadline 比零散 Timeout 更容易保持预算

Timeout 是某一步允许等待多久；Deadline 是整个操作最晚完成的时刻。若入口预算 1 秒，三个调用每个都设 1 秒并顺序执行，整体可能超过 3 秒。应根据已消耗时间计算剩余预算，预留响应编码和网络时间，向下游传播受控截止信息。

外部传入的 deadline 不能无限信任，应裁剪到服务策略范围。已无剩余时间就不要再排队或重试。连接池等待、业务线程池等待和下游读取都要计入，而不是只有 socket 超时。

超时后的服务端处理可能继续运行，尤其不可中断 SDK 和远程写操作。因此需要把响应超时率与“响应后仍在工作的任务数”关联观察；否则客户端早已收到错误，服务还在后台耗资源。

### 5.3 响应提交前后，错误映射能力不同

Problem Details 是结构化错误表示，常见字段有 type、title、status、detail、instance，可附稳定业务码和 traceId。对外消息帮助调用方采取动作，对内日志保留原始原因；不能暴露 SQL、访问令牌或堆栈。

MVC 可以用 @RestControllerAdvice/@ExceptionHandler，WebFlux 注解控制器也支持相应异常处理机制。但安全过滤链和更早的解析失败可能走另一条错误通道，需要统一契约测试。不能只给 Controller 写一个异常处理器便宣称所有错误格式统一。

响应头和部分响应体一旦发送，通常不能再把状态从 200 改成 500，也不能在已发送的 JSON 后追加另一份 Problem Details。流式协议应预先定义中途失败信号，或关闭连接并让客户端按游标重连。日志要区分“完整成功交付”和“开始输出后失败”，否则成功率可能虚高。

### 5.4 客户端断开不是业务回滚

客户端断开后，WebFlux 链可能收到取消；MVC 中常在后续写入或容器通知时发现，不应假设瞬间检测。释放订阅、定时器和流式资源很重要，不能指望一个 finally 自动覆盖所有异步生命周期。

对于读取流，可以终止无用查询并关闭游标；对于已提交的订单事务，取消不能撤销数据库提交。事务提交后通知客户端失败，重试必须通过幂等键返回已有结果。若业务要求异步处理，请先可靠接受任务、持久化状态并返回可查询标识，而不是把长事务绑定到浏览器始终在线。

## 6. 背压、流式传输与内存边界

### 6.1 request(n) 不等于网络世界里的全局流量开关

背压（Backpressure）在支持相同协议的生产消费链上控制需求。HTTP 对端、数据库驱动、消息系统、操作系统 socket 与浏览器的缓冲方式不完全相同。Reactive Streams 的元素需求不能直接等同于 TCP 字节窗口，更不能保证数据库只读取 n 行。

因此要画清边界：来源是否支持按需读取，驱动是否预取，操作符是否缓冲，编码是否聚合，出口是否慢。上游不可减速时，只能在业务允许范围内选择有界缓冲、丢弃、保留最新值、拒绝或持久化转存；没有一种策略能同时保证无限输入、零丢失、有限内存和任意慢消费。

例如实时仪表盘可只保留最新状态；支付事件不能任意丢弃，应通过持久化日志/消息系统保存并控制消费；批量导出可限制并发与导出大小。这些选择由业务数据语义决定，而不是默认加 onBackpressureBuffer。

### 6.2 collectList、flatMap 与慢客户端如何隐藏积压

Flux 不代表恒定内存。collectList 在完成前保留全部元素；flatMap 高并发会让许多请求和结果在途；缓存、预取与按序合并也会保存尚未交付的数据。若每条数据很大，按元素计数的容量还可能低估字节占用。

教学预算：1000 条慢连接，每条排队 100 个结果，每个结果约 10 KiB，仅有效负载就约 977 MiB，尚未包含对象、编码和连接缓冲。要同时监控连接数、队列长度、字节和最老元素等待时间，不能只看堆总量暂时没有 OOM。

Server-Sent Events（SSE）适合服务器向客户端单向持续发送事件；WebSocket 适合双向消息交互。两者都要设计心跳、空闲超时、最大连接数和重连。SSE 的事件 ID 可辅助恢复，但客户端重连不自动得到 exactly-once：服务端需要保留可重放数据，客户端也可能重复处理，应按事件 ID 去重。

### 6.3 资源释放与数据缓冲

流式数据库游标、文件句柄和网络响应体应随正常完成、异常和取消都释放。Reactor 可用 using/usingWhen 表达资源生命周期，doFinally 观察完成/错误/取消；不要把观察回调当成自动事务补偿。若清理本身异步，需要把清理结果正确组合，不能仅在 doFinally 中启动一个无人管理的异步任务。

使用 Netty 等引用计数缓冲时，自行截断、丢弃或变换底层 DataBuffer 需要遵守释放约定，否则可能产生直接内存泄漏；反过来过早释放会导致访问已释放内存。普通 JSON DTO 场景优先让 codec 管理缓冲，只有确需操作原始数据时才承担这层复杂性。

## 7. Context、日志与事务：请求不等于线程

传统 MVC 同步路径中，MDC/安全身份常借助 ThreadLocal 关联当前请求，必须在结束时清理，以免线程复用把甲的身份带到乙。进入业务线程池、异步回调或新的虚拟线程后，普通 ThreadLocal 不会自动成为完整的跨任务传播协议。

Reactor Context 是随订阅关联的上下文，可在切线程后保持逻辑请求信息。contextWrite 向订阅上下文写入，deferContextual 在执行时读取；它不是可随意修改的全局 Map。下面片段说明写入与读取关系：

~~~java
Mono<String> message = Mono.deferContextual(context ->
    Mono.just("trace=" + context.get("traceId"))
).contextWrite(context -> context.put("traceId", "demo-123"));
~~~

这里读取在订阅时发生。不要把线程名作为 traceId，也不要把不受信任的外部追踪头直接视为可信身份。日志桥接应使用与当前 Micrometer/Reactor 集成兼容的机制，验证跨调度、取消和错误路径后信息仍正确且能清理。只在入口 MDC.put 一次不足以覆盖异步链。

事务也有同样问题：JDBC 的传统声明式事务通常依赖线程绑定，不能在其内部随意切线程并假设仍属同一事务。响应式事务需要响应式事务管理器和兼容驱动，其上下文随响应式链传播。R2DBC 是响应式关系数据库访问 API，不意味着 SQL 更快，也不与任意 JPA 代码互换。迁移必须重新验证事务边界、连接生命周期和库支持。

## 8. 怎样选择：先找约束，再比较方案

| 工作负载与约束 | 倾向方案 | 必须验证的代价 |
|---|---|---|
| 以 JPA/JDBC 和阻塞 SDK 为主的 CRUD | MVC，有需要再评估虚拟线程 | 池等待、下游连接与准入 |
| 大量并发慢 I/O，依赖均支持非阻塞 | WebFlux 或其他非阻塞方案 | 全链阻塞点、调试与上下文成本 |
| SSE、聚合网关、长连接 | WebFlux 是合理候选，MVC 异步也需比较 | 慢消费、缓冲、取消、代理超时 |
| CPU 密集计算 | 两者都须限并行或移交计算服务 | CPU 饱和、调度和排队 |
| 少量异步外部调用，团队主要使用 MVC | MVC 配合合适客户端 | 避免为局部异步整体重写 |

选择时先问瓶颈是否真是线程：如果 SQL 缺索引、下游限额太低或序列化很重，换 WebFlux 可能增加复杂度而没有收益。端到端非阻塞需要驱动、认证、日志输出与 SDK 等路径配合，不只是 Controller 返回 Mono。

团队成本也属于架构成本。响应式链需要理解订阅、调度、上下文和取消，事故定位工具与测试方式都会变化。若团队无法稳定维护，节省少量线程可能抵不过交付与恢复时间成本。另一方面，面对大量慢连接时，只因熟悉 MVC 而拒绝测量非阻塞方案也不合理。

迁移可从边缘聚合/流式接口试点，使用明确 API 契约与相同指标比较；不要一开始替换整个持久层。生产验证要观察尾延迟、错误率、CPU、堆/直接内存、连接等待、队列与取消后剩余工作。压测至少包含慢下游、慢客户端、下游错误、突发流量和长时间稳态，平均延迟和单次吞吐峰值不足以决定选型。

## 9. 排障推演：上线 WebFlux 后所有接口一起变慢

教学假设：流量未显著上涨，但 p99 延迟上升，数据库活跃连接未满，CPU 也不高。不要立即增加 Netty 事件线程；先取线程栈/采样，检查事件循环是否停在 JDBC、锁或阻塞日志输出中。

若发现某认证 SDK 在事件循环等待远端响应，它会拖住共享事件线程，表现为多个接口延迟一起上升。短期可把该阻塞调用以惰性源隔离到专用有界调度器，同时限制并发并设置底层超时；长期评估非阻塞替代或回到更匹配依赖的 MVC。隔离后要测专用队列是否持续增长，否则只是转移瓶颈。

若线程不阻塞，但内存持续增长，则追踪 collectList、缓存、无界缓冲、响应消费速度和数据缓冲释放。若取消量高且数据库仍持续执行，则检查驱动是否响应取消以及实际查询超时。一次请求超时变快并不意味着资源恢复变快。

修复后重放原故障条件，而不是只跑健康下游；验证限制生效时明确拒绝且错误契约稳定。增加告警应聚焦最老排队时间、池等待、事件循环延迟与取消后工作，而不仅是线程数量。

## 10. 面试回答与追问

### 10.1 “MVC 和 WebFlux 怎么选？”

完整回答：“我先看依赖和等待模型。以 JDBC/JPA 为主的 CRUD 通常选择 MVC，控制连接池和入口并发；Java 21 后可评估虚拟线程降低等待成本。需要大量长连接或端到端非阻塞聚合时，我会考虑 WebFlux，但要核对 SDK、驱动和日志路径，限制扇出与缓冲。最终用相同业务和故障负载比较尾延迟、资源、成本与团队维护能力，而不是用框架名称推断性能。”

追问：“WebFlux 一定更快吗？”答案：不一定。它主要改善等待型高并发的资源使用，不消除 CPU、网络和数据库瓶颈。低并发请求的操作符和调度成本也可能抵消收益，必须测量。

追问：“已有 WebFlux 却只有阻塞库怎么办？”答案：少量阻塞可隔离并限流，且配置真实底层超时；若绝大多数路径阻塞，就应比较 MVC 的整体复杂度与收益，不能为保留返回类型不断叠线程池。

### 10.2 “背压可以解决所有过载吗？”

完整回答：“背压只在支持需求协议的链路内控制元素发送。网络、数据库预取和消息系统可能有自己的缓冲，应用中 collectList 或高并发 flatMap 也可能积压。我会定义每个边界的元素与字节上限，以及满时选择拒绝、丢弃、最新值或持久化。支付事件不能像仪表盘刷新那样丢弃，业务语义决定策略。”

追问：“无限缓冲是否能保证不丢？”答案：只能把过载延后到内存耗尽，进程崩溃还可能丢失更多；要保留数据需要持久化且控制写入速率、磁盘容量和恢复消费速度。

### 10.3 “接口 timeout 后是否可以安全重试写入？”

完整回答：“不能只看接口超时。远端可能已提交，取消也不保证终止服务器执行。写操作应携带稳定幂等键，服务器对相同请求返回同一业务结果；结果未知时先查状态，必要时补偿。时间预算与有限重试保护资源，一致性靠业务协议，不由 WebFlux 或虚拟线程提供。”

追问：“客户端断开后事务回滚吗？”答案：取决于操作是否尚在可回滚事务中及取消怎样传播，已提交事务不能因连接断开撤回。需要明确提交与响应交付之间的窗口。

## 11. 扫雷自测与完整答案

### 11.1 返回 Mono.just(jdbc.query(...)) 是否非阻塞？

不是。参数先求值，查询在构造 Mono 之前已执行。应使用延迟源并调度到合适的有界执行资源，或使用真正非阻塞驱动；即便隔离成功，JDBC 仍然是阻塞调用，容量由线程和连接预算限制。

### 11.2 在链尾 publishOn 能挪走前面的慢代码吗？

不能。它影响其后的信号处理，不能倒转已执行的代码。对阻塞源用 fromCallable 等延迟包装并在源附近指定 subscribeOn，同时核对底层超时、队列和拒绝行为。

### 11.3 zip 两个 Mono，其中一个没有值地完成，会返回缺字段对象吗？

不会自动填空。组合需要两边值，空源可能导致整体无组合结果。若空体违反协议，显式转成异常；若业务允许缺失，先转为表达“未知/缺失”的领域值，而不是含糊吞掉。

### 11.4 多了虚拟线程能否把数据库连接池从 50 放大为 5000？

没有依据。线程变便宜不代表数据库容量增加。过大连接池可能增加锁竞争、数据库内存和上下文切换，应按数据库承载与请求预算测量；先限制并发和等待时间，而不是让大量任务无限等待。

### 11.5 为什么 Flux 接口仍然 OOM？

它可能内部聚合全部数据、无限缓冲、高并发扇出，或遇到慢客户端导致队列增加。还可能是底层缓冲未释放。需要检查源、操作符、编码和出口每个边界，不能凭 Flux 返回类型排除积压。

### 11.6 流式响应已发送一半，抛异常后能否统一返回 500 Problem Details？

通常不能，状态和部分响应体已提交。协议应允许错误事件或断流重连，客户端按游标恢复，并明确重复/缺失处理。服务端日志记录中途失败，不要将开始响应计为完整成功。

### 11.7 为什么入口设置 MDC 后，异步日志没 traceId？

MDC 常基于线程本地值，而异步操作可能在其他线程上执行。使用与订阅关联的 Context 和受支持桥接，测试调度、错误及取消后的传播/清理。不能通过全局共享可变 Map 修复，否则可能串用户信息。

### 11.8 timeout 已生效而 SQL 仍运行，是否框架 Bug？

不一定。取消是协作信号，驱动和远端未必响应。应用超时只限制等待结果，需要查询/驱动超时与资源释放策略；验证取消后的实际连接和任务，而不是只验证响应在期限内返回。

## 12. 验证矩阵与本次执行范围

对示例聚合控制器，应验证两边正常、任一 4xx/5xx、任一空体、慢响应、连接获取超时和客户端取消。正常结果必须组合正确字段；错误映射应保留内部 cause 并输出稳定契约；空体不能误报完整成功；超时后观察连接与下游是否仍工作。

用 MockMvc 验证 MVC 分派、绑定和错误映射，用 WebTestClient 验证响应式 HTTP 契约，用 Reactor StepVerifier 验证流的值、错误和完成；它们不自动证明真实网络断连和池资源释放。还需受控测试服务器、真实连接器与慢消费者实验，分别记录响应交付时间与后台资源回收时间。

对隔离调度器，超过线程/队列预算时要看到受控拒绝或限流，而不是无限内存增长。对上下文，同时发不同 traceId 请求并制造切线程，确认日志不串请求。对流式接口，在发送若干事件后断开，确认游标/文件/缓冲释放，同时验证已提交写入仍可查询。

本次正文示例未实际编译、运行 Spring 应用或执行网络故障实验，不把上述验证计划宣称为已通过测试。已人工核对原主题覆盖、执行顺序、异常/取消语义、API使用与 Markdown 结构。生产参数需要目标 JDK、服务器、驱动和团队负载下的实测。

## 13. 面试前速记

- 异步不等于非阻塞，Mono/Flux 不等于另起线程。
- MVC 有同步与异步路径，虚拟线程降低等待成本但不扩大下游容量。
- WebFlux 事件回调要短，阻塞源必须隔离且有界；大面积阻塞依赖应重新评估模型。
- 限输入、限扇出、限缓冲、限等待；背压只在可传播边界内成立。
- Deadline 管总预算，timeout 与取消不能撤回已发生副作用。
- Context 绑定逻辑订阅，ThreadLocal 绑定线程；跨异步边界要验证传播。
- 流式响应提交后无法随意改状态，必须设计中途失败与恢复协议。
- 选型依据业务负载、依赖、测量和维护成本，修复必须在故障条件下复测。

## 14. 官方来源与版本范围

来源用于核验，不替代本章正文。框架细节固定以 Spring 6.2 / Boot 3.5 为基线；Reactor 文档 release 链接会滚动，落地时应对齐 Boot 实际管理版本，不机械套用新版默认值。

- [Spring Framework 6.2：DispatcherServlet](https://docs.spring.io/spring-framework/reference/6.2/web/webmvc/mvc-servlet.html)
- [Spring Framework 6.2：WebFlux Overview](https://docs.spring.io/spring-framework/reference/6.2/web/webflux/new-framework.html)
- [Reactor：Threading and Schedulers](https://projectreactor.io/docs/core/release/reference/coreFeatures/schedulers.html)
- [Spring Boot 3.5：SpringApplication](https://docs.spring.io/spring-boot/3.5/reference/features/spring-application.html)
