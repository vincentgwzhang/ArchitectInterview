# 03. CompletableFuture、虚拟线程与响应式编程

## 学习目标、案例与版本

本章回答同一个问题：一个请求需要访问商品、库存与推荐服务，怎样组织等待、组合结果、限制资源，并在超时和故障后保持可解释行为？CompletableFuture表达有限异步步骤的依赖；虚拟线程让大量阻塞任务使用同步写法；响应式编程表达持续数据流与需求控制。它们可组合使用，但都不自动提供无限容量、事务或恰好一次。

Java示例以Java SE 21为基线；虚拟线程正式API要求JDK 21及以上。响应式协议按Reactive Streams 1.0.4与JDK Flow解释，Reactor片段仅用其3.x稳定的常见API，核查文档显示版本3.8.8。JDK 25只用于说明Pinning版本边界。所有业务、时延和并发数字都是教学假设，不是用户经历或benchmark。

读完应能解释“谁执行代码、谁等待、队列在哪里、失败怎么传播、谁取消工作、谁限制下游”，并能给出替代方案及工程验证。

## 一、先分清异步、非阻塞、并发与流

异步（Asynchronous）表示调用者可以在结果未就绪时继续推进，稍后通过回调、Future或事件得到结果。非阻塞（Non-blocking）在本章I/O语境中表示等待数据时不长期占住执行线程；它不等于并发算法中的Lock-free。把阻塞JDBC放进线程池可以对调用者异步，但池中线程仍在阻塞。

并发是多个工作交叠，CPU并行才是同时执行计算。两个独立远端调用分别耗时100ms和150ms，顺序等待约250ms，并发后理想约150ms加调度开销；如果第二步依赖第一步结果，不能靠换模型消除依赖。并发会增加在途请求与下游压力，降低单请求延迟不等于提高系统吞吐。

有限聚合要关注完成条件；无限事件流还要关注生产速度、消费速度与存储边界。CompletableFuture<T>只表达一次完成状态与一个结果，T可以是列表，但它不提供逐元素需求协议。响应式Publisher则可表达零到多个元素及终止，能在消费端能力不足时通过请求数量协调生产。

先画逻辑依赖，再选执行模型。不能因为方法返回Future就假定它延迟启动，也不能因为返回Mono就假定一定换线程或一定不会阻塞。

## 二、CompletableFuture：把控制流变成依赖图

### 2.1 创建、完成与执行位置

CompletableFuture同时是Future和CompletionStage：前者提供结果等待与状态，后者提供后续转换和组合。它可由异步工作完成，也可由调用者显式complete或completeExceptionally；多个完成者竞争时只会有一个成功设置结果，不能把后续再次complete当作覆盖。

supplyAsync返回值，runAsync无业务返回值；任务提交后即可运行，不是等join才启动。completedFuture适合已有结果，failedFuture适合把已知失败放入同一组合协议。

不带Async的后续操作可能由完成上游的线程执行，也可能在注册时由调用者执行；不要依赖它总在某个业务池。带Async通常把工作调度给Executor；默认异步方法一般使用公共ForkJoinPool，公共池并行度不足时有API规定的替代行为，子类也可改变部分默认执行策略。因此需要资源隔离时显式传Executor。

“传过一次Executor，后面所有Async都继承”是错误推断。每个异步边界要检查其重载与执行策略。短小纯转换可直接thenApply；昂贵CPU或阻塞调用需要明确执行预算，避免占住网络回调或公共池。

### 2.2 thenApply、thenCompose和thenCombine为什么不同

thenApply是T→U：把已有结果同步转换为另一个值。thenCompose是T→CompletionStage<U>：前一步结果决定下一次异步工作，并把嵌套阶段展开。若用thenApply调用返回Future的方法，会得到Future<Future<U>>，外层完成时内层不一定完成。

下面为业务类型片段，client方法必须由应用提供：

~~~java
CompletableFuture<Product> product = productClient.get(id);
CompletableFuture<Stock> stock = inventoryClient.get(id);

CompletableFuture<ProductView> view =
    product.thenCombine(stock, ProductView::new);

CompletableFuture<Recommendation> recommendation =
    product.thenCompose(p -> recommendationClient.get(p.categoryId()));
~~~

商品与库存先分别发起，因此可并发；thenCombine只在两者正常完成后使用结果，不负责“替你启动两个调用”。推荐依赖商品分类，所以用thenCompose。若get本身在返回Future之前就阻塞，这段表面异步代码仍会先卡在调用点，需要先核实客户端契约。

thenAccept消费结果而不产生业务返回值，thenRun只关心正常完成、不读取上游结果。这些操作的名称说明依赖形状，不是线程数或并行度承诺。

### 2.3 allOf、anyOf与结果收集

allOf返回CompletableFuture<Void>，表达全部阶段完成，不直接返回结果列表。正常完成后可读取各输入结果，此时join通常不会再等待；任一失败会使聚合异常完成，但不能把allOf当成自动取消其他任务或保证立即失败的工具。

anyOf返回最先完成阶段的结果，可能是失败；“先完成”不等于“第一个成功”。若需要两个供应商竞速取首个有效结果，应显式区分失败与成功，并在所有候选失败时结束，避免等一个永远不存在的成功。赢家出现后也要处理输家的资源与副作用，不能只丢掉引用。

循环提交十万个Future后再allOf，即使线程池线程数有限，Future、任务和队列也可能占大量内存。应该按批次、信号量或有界接纳限制在途工作；allOf只是组合器，不是流量控制器。

### 2.4 get、join与同池等待陷阱

get可以响应中断，失败通常以ExecutionException包装；join通常用CompletionException表达失败，不提供同样的受检中断接口。两者遇到取消还需按CancellationException处理。保留cause和业务上下文，避免把包装异常误判为唯一错误类别。

在请求线程最外层等待结果可以是明确选择，但不要在事件循环里join。固定池内父任务提交子任务到同池又阻塞join/get，可能所有线程都在等队列中没有执行机会的子任务。改用依赖组合、不同资源域或同步的每任务虚拟线程模型，不能只无限扩大池。

## 三、异常、降级与重试：放在哪一层改变业务含义

### 3.1 三个异常操作符的职责

exceptionally在异常时返回替代值，正常时保留原值；handle同时接收结果与异常，可把二者转换成新值；whenComplete用于观察成功或失败，通常保留原结果，但回调自己抛异常会影响返回阶段，不能假定它绝不改变完成状态。

如果库存查询失败，返回0可能把“未知”伪装成“售罄”；对可选推荐服务失败返回空推荐则可能符合契约。降级（Fallback）必须是业务允许的结果，最好显式表达来源或降级状态，而不是把所有异常吞成默认对象。

对单个可选分支做降级后再合并，与对最终聚合统一降级不同。前者可以保留正常商品和库存；后者可能掩盖必需商品失败。应在能理解失败后果的层处理，底层保留真实原因，上层转换为稳定错误契约。

### 3.2 Deadline与Timeout不是同一层概念

Timeout是某次等待或操作的时长限制；Deadline是整个请求不应超过的截止点。入口预算500ms，已排队200ms后，下游不能仍各拿全新的500ms。剩余预算应覆盖连接、读取、重试和结果组装。

本地剩余时间宜用单调时钟System.nanoTime计算；不要把一个进程的nanoTime数值原封不动发给另一个进程比较。跨服务传播要采用约定的剩余预算或时间语义，并处理传输与时钟误差。预算传播需要客户端与服务都参与，Future链条不会自动完成。

orTimeout会在期限到来且未完成时使该CompletableFuture异常完成；completeOnTimeout给默认完成值。它们返回并修改同一个Future，若该对象被多个消费者共享，会影响所有观察者。需要独立消费者预算时，可评估copy等独立阶段，并明确该副本超时仍不停止原工作。

### 3.3 超时完成不等于停止执行

get(timeout)限制等待者等待；orTimeout限制Future完成状态。它们都不能保证底层线程、套接字或服务器停止。CompletableFuture.cancel(true)也不承诺中断正在工作的线程，它主要改变完成状态；取消依赖阶段不自动给所有源头建立一套端到端取消树。

资源控制要连接实际客户端的取消句柄、连接/读取超时、任务Future和远端协议。尽力取消仍可能有竞态：付款已提交但响应丢失，取消本地等待不会回滚。重试必须使用业务幂等键、结果查询或对账恢复，不能凭TimeoutException认定“未执行”。

原例view.orTimeout(500ms)适合表达聚合结果的期限，但仍需客户端级超时和剩余预算。不能在文档里只加一行orTimeout就宣称Deadline、Cancellation和资源释放已完整实现。

### 3.4 重试需要预算与副作用边界

重试要限定可恢复错误、最大次数、总预算及退避抖动；权限错误、参数错误不应盲目重试。多个分支各重试再上层整体重试会放大流量，应计算最坏尝试数，并只重试所需部分。

例如两个必需分支中商品成功、库存暂时失败，重新请求整个聚合会重复商品读取；若其中有写操作，风险更高。先确认业务能否复用已成功结果及数据新鲜度，再选择重试层。重试属于架构与业务语义，不是任意exceptionally里再调一次自身。

## 四、完整CompletableFuture实验：验证形状与异常，而非靠sleep猜并行

下面是可保存为FutureCompositionDemo.java的完整JDK 21类。使用手工完成Future，确定性验证嵌套展开、独立合并和可选失败降级。命令为javac FutureCompositionDemo.java，然后java FutureCompositionDemo。

~~~java
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.atomic.AtomicInteger;

public class FutureCompositionDemo {
    record Product(String id, String category) {}
    record View(Product product, int stock, String recommendation) {}

    public static void main(String[] args) {
        CompletableFuture<Product> product = new CompletableFuture<>();
        CompletableFuture<Integer> stock = new CompletableFuture<>();
        CompletableFuture<String> recommendation = new CompletableFuture<>();
        AtomicInteger invoked = new AtomicInteger();

        CompletableFuture<String> optional = product.thenCompose(p -> {
            invoked.incrementAndGet();
            return recommendation;
        }).exceptionally(error -> "unavailable");

        CompletableFuture<View> view = product
            .thenCombine(stock, (p, count) -> new View(p, count, "pending"))
            .thenCombine(optional,
                (v, rec) -> new View(v.product(), v.stock(), rec));

        stock.complete(7);
        if (view.isDone() || invoked.get() != 0) {
            throw new AssertionError("dependency violated");
        }
        product.complete(new Product("p1", "books"));
        if (view.isDone() || invoked.get() != 1) {
            throw new AssertionError("inner stage not awaited");
        }
        recommendation.completeExceptionally(new IllegalStateException("demo"));
        View result = view.join();
        if (result.stock() != 7 || !result.recommendation().equals("unavailable")) {
            throw new AssertionError(result);
        }
        System.out.println(result);
    }
}
~~~

stock先完成不能让view完成，因为商品未到；商品完成触发一次推荐阶段创建，但推荐尚未完成，最终结果仍未完成；推荐失败被该可选分支转换成unavailable，库存7保留。它没有掩盖必需商品或库存失败。

本实验不创建线程，所以能检验依赖语义，却不能测并行速度、线程池饱和或网络取消。不要把输出的确定性理解成所有业务回调线程都确定。生产还需故障注入、有限Executor及真实客户端取消测试。

## 五、虚拟线程：同步控制流中的廉价等待

### 5.1 载体、挂载与吞吐

平台线程通常绑定OS线程；虚拟线程由运行时调度到载体线程（Carrier Thread）上执行。兼容阻塞I/O等待时，虚拟线程可挂起并释放载体去执行其他任务；恢复后未必是同一个载体。虚拟线程仍是Thread，局部变量、异常栈和大部分同步代码模型保持熟悉。

价值来自大量等待任务不必一对一长期占OS线程，不是让单次I/O响应更快。计算10ms再等90ms的任务可能受益；纯计算工作仍受CPU预算制约。虚拟线程也消耗栈与任务状态内存，并非免费或无限。

通常每任务一个虚拟线程，不为了复用而建固定虚拟线程池。限制的是稀缺依赖、在途请求和内存，可用Semaphore、连接池与入口接纳控制。只把阻塞线程池换虚拟线程执行器而取消所有上限，会让数据库面对更猛烈并发。

### 5.2 同一商品聚合如何表达

以下是JDK 21方法片段，需导入java.util.concurrent.Future和TimeUnit，Product、Stock、ProductView及两个阻塞客户端是待提供业务类型。io是应用生命周期管理的newVirtualThreadPerTaskExecutor返回的ExecutorService，不在每次请求里关闭共享执行器。

~~~java
ProductView load(String id, long budgetNanos) throws Exception {
    long started = System.nanoTime();
    Future<Product> product = io.submit(() -> productClient.getBlocking(id));
    Future<Stock> stock;
    try {
        stock = io.submit(() -> inventoryClient.getBlocking(id));
    } catch (RuntimeException rejected) {
        product.cancel(true);
        throw rejected;
    }
    try {
        long left = budgetNanos - (System.nanoTime() - started);
        Product p = product.get(Math.max(0L, left), TimeUnit.NANOSECONDS);
        left = budgetNanos - (System.nanoTime() - started);
        Stock s = stock.get(Math.max(0L, left), TimeUnit.NANOSECONDS);
        return new ProductView(p, s);
    } finally {
        product.cancel(true);
        stock.cancel(true);
    }
}
~~~

两个任务先提交，随后依次get不等于远程调用顺序执行。第二次等待使用剩余预算，避免两个get各耗完整预算。finally向未完成任务发协作中断；已完成Future的cancel不会回滚结果。

这不是完整生产接口：还需验证预算为正、在提交前做接纳控制、为客户端设置超时、对ExecutionException映射、对中断传播，以及处理业务鉴权。按这个等待顺序，如果库存先失败而商品一直慢，可能直到商品结束或超时才观察到库存失败；若要求立即发现任一失败，需要完成顺序协调或其他明确协议。

不要用try-with-resources在每次聚合方法内新建执行器后，假定方法Future等待超时就立刻返回。ExecutorService.close会等待任务终止，可能继续阻塞；失控任务仍需取消与客户端边界。本片段选择应用共享生命周期，就是为了把执行器关闭与请求期限分开。

### 5.3 Semaphore限制依赖，但也要限制等待者

假设供应商最多允许本服务100个并发调用，可先按剩余预算tryAcquire，成功后在finally release，获取失败则明确拒绝或降级。连接池限制连接并发，但不一定限制等待借连接的请求数；入口需要另外控制内存与排队时间。

许可证应覆盖真正占用下游的时段。若请求线程超时就释放许可证，而后台网络请求仍在运行，100许可可能对应超过100个真实远端调用；释放时机要和实际操作终止协议匹配。过载控制的计数对象必须说清，不能只在图上画Semaphore。

### 5.4 Pinning与版本边界

Pinning指虚拟线程不能在相关阻塞期间从载体卸载。JDK 21的synchronized内长阻塞及native/外部函数相关路径需要注意；频繁长时间固定会减少可用载体，而非使程序本身必然错误。

后续版本已改善监视器相关行为，JDK 25官方指南不再将synchronized列为固定原因，仍说明native或外部函数相关情形。因此不能把“虚拟线程必须全部改ReentrantLock”当跨版本规则。先用目标JDK的JFR事件与实际调用栈确认，再调整热点同步边界。

即使没有Pinning，长时间持锁I/O仍会串行化等待该锁的其他任务，这是锁竞争问题，不能靠卸载载体解决。版本升级不替代合理临界区设计。

### 5.5 上下文、事务和观测

ThreadLocal在虚拟线程中仍可用，但每任务初始化巨大缓存会复制很多对象；过去给少量池线程缓存连接或大对象的做法需要重审。请求身份应显式传递或使用框架支持的上下文，不依赖载体线程身份。

事务绑定当前执行上下文时，另起子线程不会自动加入父线程事务。跨任务共享JDBC Connection或EntityManager通常违反其使用约束；按目标驱动/框架契约设计，不以“虚拟线程也是Thread”推导所有上下文自动传播。

观测仍看每依赖等待、连接池、在途与完成率，而非只看线程数量。线程dump要用能展示虚拟线程的目标JDK工具，JFR用于检查固定及阻塞；迁移前后在同样负载、超时和失败条件下比较吞吐、p99和内存。

## 六、Reactive Streams：背压是一套需求协议

### 6.1 四个角色与信号顺序

Publisher产生数据，Subscriber接收，Subscription连接二者并提供request(n)与cancel，Processor同时扮演订阅者和发布者以处理流水线。JDK 9起Flow提供对应语义接口，但类型与第三方Reactive Streams接口不是同一个Java类型，互通通常需适配。

订阅者先收到onSubscribe，再在需求许可内接收onNext，最后收到onComplete或onError之一；终止后不再正常发送元素。同一订阅信号必须序列化，序列化不要求始终同一个线程。订阅者若还访问链外共享可变状态，仍需正确同步。

request(n)表达增量需求，不是要求立即发送n个，也不是每秒n个。没有未消费需求，发布者不能继续无限推送onNext；Long.MAX_VALUE通常表示无界需求。取消要求停止后续工作并清理资源，但与已经在途信号可能竞态，不等于回滚已消费的数据。

### 6.2 背压怎样保护资源，又保护不了什么

消费者每批处理完成再请求下一批，可以限制生产者越过异步边界的速度。若源头可控，如按需数据库游标或文件读取，需求可以减少读取；若源是不可暂停外部事件，必须选择有限缓冲、丢弃、合并最新值、拒绝或持久化转存，并说明丢失与延迟语义。

协议不是自动内存上限。操作符可能预取（Prefetch，提前申请若干元素）、缓存、并发映射或聚合全部结果；collectList面对无界流会永不正常收尾并积累内存。flatMap并发数、内层预取、重试和每元素字节都要预算。

request(10)只限制元素数，10个10 MB对象与10个100 B对象的内存差异很大。TCP流控也不等于业务背压：网络能少发字节，不保证上游业务不先在内存生成百万对象。跨系统是否传播需求取决于协议与实现，不能凭一个Publisher接口就推导端到端受控。

### 6.3 Hot、Cold与重复订阅

Cold（冷源）通常为每次订阅建立独立产生过程；例如一个延迟执行的HTTP调用每次订阅都可能重新发请求。Hot（热源）可能独立于订阅者持续产生数据，不同订阅者看到不同时间段。具体行为由创建、共享和重放操作符决定，不能只看Mono/Flux类型判断。

两个地方各subscribe一次可能触发两次数据库写；retry也可能通过重新订阅重复源操作。share/cache/replay会改变共享、重放与生命周期，又可能带来过期、敏感数据跨用户共享和内存成本，不能把缓存操作符当通用去重或幂等机制。

## 七、Reactor中的流水线：语法背后的执行与失败

### 7.1 Mono、Flux与惰性执行

Mono表示零或一个元素的序列，Flux表示零到多个；完成但没有元素是独立于错误的状态。典型冷流水线需要订阅才推进，但部分创建方式会在组装时就求值，例如Mono.just(blockingCall())已经调用了阻塞函数。

需要延迟阻塞调用可用fromCallable，并把它调度到受控阻塞工作域。下面是Reactor 3.x片段，非独立完整程序：

~~~java
Mono<Product> product =
    Mono.fromCallable(() -> legacyProductClient.getBlocking(id))
        .subscribeOn(blockingScheduler);
~~~

blockingScheduler应由应用管理并有容量预算，例如按版本与配置选择有界弹性调度器；它仍可能排队，不能当无限资源。若客户端已有真正非阻塞API，通常直接组合它返回的Publisher，而不是再包一层阻塞等待。

服务框架通常负责最终订阅。业务服务里手动subscribe再立即返回，会让错误、取消、上下文与事务脱离请求生命周期。应返回组合后的Mono/Flux，让边界统一接管。

### 7.2 map、flatMap、zip与顺序

map做每元素同步转换；flatMap把元素映射为内部发布者再合并，可能交错结果；concatMap通常按顺序逐个推进，更容易保持顺序但减少并发。需要并发又保留输出顺序时可评估相应有序合并操作，但可能为等待慢元素而缓冲。

对商品与库存两个单值异步结果，可使用Mono.zip，等必要值齐全再构造视图；任一源为空时，组合可能没有值，不能把“空”当成“库存为0”。若业务要求必需数据，应明确switchIfEmpty转业务错误或约定替代值。错误与取消也会影响其他分支，具体操作符契约需要测试。

~~~java
Mono<ProductView> view = Mono.zip(
        productClient.get(id),
        inventoryClient.get(id))
    .map(tuple -> new ProductView(tuple.getT1(), tuple.getT2()))
    .timeout(Duration.ofMillis(500));
~~~

这里假设两个client返回必需单值Mono，ProductView为教学业务类型。timeout不能保证远端请求已经停止；实际客户端和驱动必须支持取消与资源释放。若库存失败可降级，应在库存分支明确处理，而非最终把商品错误也吞掉。

### 7.3 subscribeOn与publishOn不是多写几个就并行

subscribeOn影响订阅和源端执行安排，常放在延迟阻塞源旁以便读者看清；publishOn在链中引入下游执行上下文切换。Scheduler负责安排任务，不等于每个操作符自动开新线程；多次切换增加调度、队列和上下文成本。

事件循环（Event Loop）用少量线程处理大量I/O就绪事件，要求回调迅速完成。在其上调用Thread.sleep、阻塞JDBC、join或同步远程接口，会阻塞同线程其他连接。即使外层是WebFlux或返回Mono，也不能改变内部阻塞事实。

把阻塞工作转移到有界池是一种桥接，不是令阻塞I/O变成非阻塞。必须限制桥接池、等待队列、调用时长和下游连接；当大量步骤都必须桥接，虚拟线程同步方案可能更易维护。

### 7.4 错误、重试、释放和上下文

onError是终止信号；onErrorResume切换到替代序列，不是让已失败的原序列继续。doOnError适合观察但不等于恢复，onErrorReturn提供替代值需要业务许可。retry会重新订阅相关源，因此副作用可能重做；限定错误、次数与预算仍是必要条件。

同步try/finally可能只覆盖流水线组装，而执行在稍后发生；资源应跟订阅生命周期绑定。可按资源类型使用using/usingWhen等机制，doFinally适合终止观察，但不能把异步关闭放进一个不等待的回调就认定关闭已完成。正常、失败、取消三条路径都要验收资源归还。

Reactor Context是随订阅传播的上下文机制，不是线程局部变量的直接别名；读写位置与订阅方向有关。租户、追踪与安全身份需通过支持的机制传递或显式参数，不能假定线程切换后MDC自动保持。响应式事务也需与框架上下文集成，不能混用线程绑定事务的直觉。

## 八、同一个聚合需求，三种模型怎样作选择

| 维度 | CompletableFuture | 虚拟线程同步写法 | Reactive流水线 |
|---|---|---|---|
| 主要表达 | 有限异步依赖与一次结果 | 每任务同步控制流 | 零到多元素流、需求和信号 |
| 等待执行成本 | 取决于客户端及Executor | 兼容阻塞等待可让出载体 | 非阻塞客户端与短回调共享执行资源 |
| 限流手段 | 有界执行器、接纳与许可 | 接纳、许可与连接预算 | 需求、并发/预取/缓冲限制及入口配额 |
| 调试重点 | 回调执行者、异常和依赖图 | 任务栈、下游与版本行为 | 订阅链、信号、调度与上下文 |
| 不适合直接套用 | 无界流及无界Future扇出 | 期望提高CPU算力或无限放开下游 | 团队全是阻塞依赖却无桥接治理能力 |

普通低并发接口顺序同步代码可能已经足够；引入并发需要收益大于取消、资源和调试复杂度。有限的独立异步客户端调用，Future组合自然；团队主要用阻塞库且任务等待多，虚拟线程有吸引力；需要长期流、端到端非阻塞与需求管理，响应式更贴近问题。

CPU密集计算无论套Future、虚拟线程还是Flux都应限制并行度，考虑拆分、算法与CPU预算。混合系统也合理，例如响应式HTTP入口桥接有限阻塞遗留库；但每个边界要知道在哪里排队、上下文怎样传、取消怎样释放。

不能宣称虚拟线程取代所有Reactive，或Reactive必然性能最好。它们优化和表达的对象不同，比较需固定相同业务语义、可靠性、峰值和资源成本。

## 九、架构案例：Fan-out、配额与部分成功

假设入口1000 RPS，每请求并发查询商品、库存、推荐三个服务。没有缓存和重试时最多产生3000次下游请求/秒；若每个分支最多重试一次，最坏可能6000次。Fan-out（扇出）降低独立等待的关键路径，却扩大下游工作。

先区分必需与可选：商品不存在是业务失败；库存未知不能冒充0；推荐失败可省略且标注降级。设计截止时间内的结果契约，再对每依赖分配并发/连接/重试预算，避免推荐慢占尽商品资源。

若每请求平均在途0.2秒，1000 RPS稳态平均约200个入口请求；它们最多可对应更多下游在途。Little's Law中的平均量不能拿p99替换，也不能在积压不断增长时把完成吞吐当稳定系统容量。模型改变线程等待成本，不改变守恒的业务工作量。

隔离（Bulkhead）应按依赖故障域和业务重要性设置，过细会增加闲置资源与配置维护。停止等待、取消任务、关闭网络资源、远端终止是不同层，必须逐一说明。观测入口成功率之外，还要看每依赖尝试次数、有效完成、超时后仍在运行的工作及降级比例。

## 十、工程验收与排障：测清资源从哪里泄漏

### 10.1 必测的失败场景

对Future组合，构造上游失败、内层未完成、可选降级、聚合超时、源头持续执行和同池等待；不要只测正常结果。手工complete可以稳定依赖测试，真实网络取消还要用可控慢服务验证。

对虚拟线程，测试目标JDK下客户端阻塞、许可等待中断、关闭期间任务不退出、线程本地大对象和下游饱和。不要只看平台线程减少就认定性能通过，关注目标p99、内存、连接与完成率。

对Reactive，用StepVerifier等测试工具验证值、终止、错误与取消，使用虚拟时间验证适用的时间操作符，并测试有限需求、慢消费者和溢出策略。虚拟时间不模拟真实网络与CPU，也不能证明驱动取消正确。相同源重复订阅要检查是否重复副作用。

### 10.2 如何从现象定位

Future链条不完成，检查是否有源从未complete、thenApply意外嵌套或任务在同池互等；异常“消失”，检查是否忽略返回阶段或统一降级吞掉失败。

虚拟线程很多且数据库等待长，先查连接预算、事务与查询，不忙着增加载体；JDK 21的固定事件需与目标版本对照。Reactive延迟普遍变差且少数事件循环栈在阻塞调用，定位阻塞源并治理桥接；队列/内存上涨则核查prefetch、flatMap并发、collectList与无限缓冲。

三种模型都应关联请求与下游traceId，记录排队与执行时间，避免在日志写完整令牌、敏感URL或巨大payload。线程ID可辅助排查，但异步链不能只靠一个线程ID串联全程。

### 10.3 演进与成本

迁移先选非关键但有代表性的接口，保持超时、错误、鉴权与返回契约，做相同负载对照及故障测试。回滚应能切回旧执行路径，而不是只比某个空任务微基准。

计算成本、内存、连接、网络、观测与团队维护共同构成成本。虚拟线程可能减少平台线程，却增加可同时运行的下游请求；响应式可能降低等待占用，却提高调试和上下文治理要求。成本结论应由可重复测量支持，不编造“节省百分之多少”。

## 十一、完整面试回答与追问

**问：CompletableFuture、虚拟线程和Reactive怎样选？**

“我先看任务形状、依赖生态和资源限制。有限的异步依赖用CompletableFuture组合很自然，thenCompose处理依赖异步，thenCombine合并独立结果。已有大量阻塞库且等待多，我倾向评估JDK 21以上虚拟线程，让每任务同步编写，但用许可和入口配额保护数据库。长期流、慢消费者和端到端非阻塞链路则评估Reactive，明确需求、预取和溢出策略。CPU密集都要受控并行，不能用模型名称代替容量验证。”

**追问：thenApplyAsync就能避免阻塞问题吗？**

“它把回调调度出去，但默认可能用公共池，且阻塞本身仍占工作线程。需要显式执行域、队列/并发预算与超时，不能因为Async后缀就认为是非阻塞I/O。”

**追问：thenCompose与thenCombine的核心区别？**

“compose的第二步需要第一步结果才能创建，函数返回一个阶段并被展开；combine接收另一独立阶段并合并两个正常结果。是否并发还看独立操作是否已发起，combine自身不启动它们。”

**追问：超时后如何防止重复扣款？**

“先承认结果可能未知，超时不是未执行。传业务幂等键、查询权威状态，并按协议恢复；本地cancel和Reactive取消都不能回滚远端已提交付款。需要观测取消后剩余工作和资源释放。”

**追问：Reactive有背压，为什么还会OOM？**

“需求只是一部分，操作符可预取、缓冲、并发展开或收集全部元素，源头也可能不可暂停。要限制每层队列、在途和字节，并对不可控源选择拒绝、持久化或允许的丢弃，不能只看接口合规。”

**追问：换虚拟线程后还需要Reactor吗？**

“虚拟线程改善阻塞等待成本，Reactor还表达流式信号与需求管理；是否保留取决于任务形状和现有端到端生态。同步聚合可能更简单，持续事件流可能仍适合Reactive，不做全仓库机械替换。”

## 十二、扫雷自测与完整答案

### 12.1 为什么thenApply调用返回Future的方法会多一层？

thenApply把函数返回值原样当U，因此U本身若为Future就形成Future<Future<U>>，外层只表示函数已返回内层句柄。thenCompose专门等待并展开该阶段，最终结果随内层完成；不是简单语法糖，而是不同完成条件。

### 12.2 thenCombine后orTimeout是否完整实现500ms请求期限？

没有。它限制相应Future的完成，不证明底层停止，也不自动扣减此前排队与组装耗时。应在入口建立剩余预算、传到客户端和许可等待，并连接实际取消与清理；共享Future超时还会影响其他消费者。

### 12.3 allOf有一个失败会自动取消其他任务吗？

不会。allOf是聚合完成机制，不是任务生命周期监督器，也不能当首失败立即取消协议。需要显式失败策略与源任务句柄，取消后仍验证协作响应和外部副作用。正常allOf完成后读取输入可不再等，但异常完成要保留每分支诊断。

### 12.4 200个平台线程改成虚拟线程，数据库可以放开到十万并发吗？

不能。数据库CPU、锁、连接和供应商配额没增加，更多等待者还会耗内存。限制稀缺资源并限制接纳数量，核查目标驱动和JDK行为。虚拟线程提高等待任务可管理规模，不提高数据库物理能力。

### 12.5 request(100)是否每秒只消费100个？

不是。它增加最多100个元素的未满足需求，不设时间速率；生产者可以很快发送这些元素，也可能慢慢发送。限速需要时间策略，内存还需考虑每元素大小、预取和队列。累计无界请求则放弃了有限元素需求保护。

### 12.6 Mono.just(legacyCall())再subscribeOn能延迟调用吗？

不能。Java先求legacyCall()实参，阻塞已在组装线程发生。用fromCallable或适当defer将调用放进订阅过程，再用合适受控Scheduler安排；仍需下游与队列预算，不是把阻塞变非阻塞。

### 12.7 推荐服务失败返回空，与库存失败返回0为什么不同？

空推荐可能是业务允许的可选降级；库存0是“确认没有货”的断言，错误转0把未知变成事实，可能影响购买和监控。应显式表达未知/降级或失败，降级放在理解该业务意义的分支。

### 12.8 为什么同一响应式写入源retry会重复执行？

retry通过重新订阅相关源再次产生工作；上次写可能已成功但响应失败。需要业务幂等键与权威结果查询，并控制重试范围、次数和总时间。share/cache不等于持久幂等协议。

### 12.9 主请求事务会随Future或虚拟线程子任务自动传播吗？

不会自动成立。线程绑定事务与异步/订阅上下文的生命周期不同，JDBC连接等资源也未必允许跨线程共享。必须按框架明确的事务模型设计任务边界，不把创建子任务当加入父事务。

## 十三、复习连接与速记

三种模型都围绕“等待、组合与资源”工作，但抽象不同：Future表达完成依赖，虚拟线程保留同步控制流，Reactive表达流与需求。架构选择要把编程复杂度、生态兼容性、失败语义、观测和成本放在同一比较条件里。

速记：apply转值、compose展平、combine合并；Async不等于非阻塞；allOf不监督取消；超时不等于停止或回滚；虚拟线程不放大下游；request控制元素需求不控制每秒速率；冷源订阅可重做；阻塞桥接仍要预算；上下文与事务显式治理；先定义业务结果再谈模型。

## 十四、来源、适用范围与验证记录

核查日期：2026-11-10。正文已就地解释必要概念，以下只供核验与追溯；不存在厂商吞吐保证或真实项目指标。

- [Java SE 21 CompletableFuture](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/CompletableFuture.html)：完成、组合、执行与取消契约。
- [JDK 21 Virtual Threads](https://docs.oracle.com/en/java/javase/21/core/virtual-threads.html)与[JDK 25 Virtual Threads](https://docs.oracle.com/en/java/javase/25/core/virtual-threads.html)：执行模型、采用和版本差异。
- [Reactive Streams](https://www.reactive-streams.org/)与[Java SE 21 Flow](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/Flow.html)：需求与信号协议，正文按1.0.4/JDK21解释。
- [Reactor线程与Scheduler](https://projectreactor.io/docs/core/release/reference/coreFeatures/schedulers.html)及[错误处理](https://projectreactor.io/docs/core/release/reference/coreFeatures/error-handling.html)：核查页面显示3.8.8；具体调度器容量/默认参数按目标依赖与配置核实，本章不依赖固定默认数字。

验收覆盖原章：CompletableFuture三种核心组合与allOf、公共池隔离、商品聚合、异常/取消/Deadline、Fan-out压力；虚拟线程与载体、稀缺资源、Pinning；非阻塞流/背压、混入阻塞、上下文和调试代价；四类选型与CPU限制。原薄弱章节均原位整合，所有面试与自测附推理。

本次环境为OpenJDK 21运行时，无javac/jshell；完整Java实验、业务片段及Reactor片段均未实际编译运行。已人工检查类型关系、完成顺序、错误、取消与清理边界；期望结果不是实测输出，没有声称真实网络取消、吞吐或生产SLO得到验证。
