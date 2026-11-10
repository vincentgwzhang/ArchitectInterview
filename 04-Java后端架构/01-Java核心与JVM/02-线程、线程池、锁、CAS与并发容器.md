# 02. 线程、线程池、锁、CAS与并发容器

## 学习目标、版本与阅读方法

并发（Concurrency）是多个任务在时间上交叠推进；并行（Parallelism）是多个任务同时占用计算资源。单核也能并发，多线程也不保证更快。本章目标是从业务不变量推导共享状态、同步边界和资源预算，并能解释超时、重试、取消及故障之后的结果。

核心API以Java SE 21为基线，多数基础机制适用于较早版本；虚拟线程正式API属于JDK 21，后文单列版本差异。库存、调用配额和线程池参数均为教学案例，不代表用户项目。代码中的“不超卖”仅指单JVM内存对象，不能直接替代数据库事务或跨节点协调。

回答统一采用：结论 → 共享状态和不变量 → 机制 → 失败边界 → 工程选择与验证。先减少共享可变状态，再选择原语（Primitive，构建并发控制的基础操作），通常比遇到竞争就加更多线程有效。

## 一、线程的执行与生命周期：任务不是线程

### 1.1 start、run与状态

Runnable表示无返回值任务，Callable表示能返回结果并抛异常的任务；Thread是执行任务的线程对象。调用start启动新线程，直接调用run只是当前线程中的普通方法调用。同一Thread对象只能启动一次，任务可以反复执行不意味着线程对象可以反复start。

Java线程状态包括NEW、RUNNABLE、BLOCKED、WAITING、TIMED_WAITING、TERMINATED。RUNNABLE包括正在执行及等待CPU等情况，不等于一直在消耗CPU；BLOCKED特指等待进入对象监视器，显式锁等待常表现为WAITING或TIMED_WAITING。状态只是某个时刻的观察，不能仅凭一张快照判断泄漏或死锁。

sleep让当前线程暂停一段时间但不释放已持有监视器。Object.wait要求当前线程持有该对象监视器，等待时释放它，返回前重新获得；被唤醒不代表条件必然成立。join等待另一线程结束，可带超时；调用带超时join返回后仍需判断目标是否结束。

### 1.2 平台线程与成本

平台线程通常对应操作系统线程，栈、调度和上下文切换都需资源。线程数上升可能增加并发等待能力，也可能增加CPU切换、内存占用及下游竞争。某个请求在等数据库时，增加线程不一定减少数据库等待。

线程池复用执行线程；线程安全的任务对象仍可能访问不安全依赖。把任务提交给池，既不自动赋予事务边界，也不自动传播登录身份、日志上下文和请求截止时间。

## 二、Java Memory Model：先证明读到的状态为什么正确

### 2.1 原子性、可见性、有序性

原子性（Atomicity）指某个操作在并发观察下不可被分割；可见性（Visibility）指线程何时可以观察其他线程写入；有序性（Ordering）约束允许观察到的操作顺序。JMM规定合法执行行为，不是“每个线程有一个缓存然后定时刷内存”的硬件流程图。

count++包含读取、计算、写回。两个线程都读到0再各写1，结果丢一次更新。把count声明volatile使访问具备相应可见性和顺序语义，但不能把这三个步骤合成原子操作。原子变量、同一把锁或单线程归属可以解决这个例子，选哪一种取决于是否还有其他状态要一起维护。

数据竞争（Data Race）涉及未由happens-before排序的冲突访问；竞态条件（Race Condition）更宽泛，例如两个线程分别用线程安全的containsKey和put，组合后仍能重复创建。单个API安全不代表业务操作安全。

### 2.2 Happens-before不是墙钟先后

Happens-before是保证可见性与顺序的关系。常用边包括同一线程的程序顺序、同一监视器的解锁到随后加锁、volatile变量的写到后续读、start之前的操作到新线程内操作，以及线程全部操作到另一个线程成功检测它结束。关系可传递，因此可跨多次同步把数据安全传递。

并发库也提供内存一致性契约：提交任务前的操作可见于任务执行；通过Future.get取得完成结果后，可见相应任务操作；将对象放入并发队列前的操作可见于随后取得该对象的线程。这些边不保证提交后继续乱改同一对象是安全的。

“线程A先打印完、线程B过一秒读取”不是同步证明。sleep、日志输出或调试器偶然改变时序，可能掩盖错误；不能把机器上暂未复现当作JMM保证。

### 2.3 volatile发布与不可变快照

下面是可嵌入类中的示意，语法以JDK 21为基线：

~~~java
record Config(int limit, String region) {}
final class ConfigHolder {
    private volatile Config current = new Config(10, "eu");
    Config get() { return current; }
    void replace(int limit, String region) {
        current = new Config(limit, region);
    }
}
~~~

构造完成的不可变Config通过volatile引用发布，读者读取一次引用即可使用一致的两个字段。若改成volatile引用指向可变Map，再无同步地修改Map内容，volatile引用不会让所有内部操作自动线程安全。

final字段有专门初始化安全语义，但对象构造期间不能让this提前逸出，例如在构造器里注册监听器并被其他线程调用。final List只是引用不能重新赋值，列表本身仍可变。工程上用不可变值、拷贝与明确发布边界，通常比维护多个零散volatile字段更容易证明正确性。

## 三、锁保护的是不变量，不是某一行代码

### 3.1 synchronized锁谁，保护多久

实例同步方法锁this，静态同步方法锁声明类的Class对象，同步块锁括号中求得的对象。两把不同锁不会互斥；每次new Object再加锁也无法保护跨调用共享状态。不要用公开对象、字符串常量或外部可控制对象作为内部锁，以免无意共享锁。

synchronized同时提供互斥与内存语义，可重入，即持有者可再次进入同一监视器。正常或异常退出同步块会自动释放锁，但异常之前的字段修改不会自动回滚。转账若先扣款后因校验异常退出，锁释放了，业务仍可能不一致。

临界区（Critical Section）应覆盖检查与修改这一完整不变量，同时尽量短。下面是完整的单对象库存实现：

~~~java
final class Stock {
    private int remaining;
    Stock(int initial) {
        if (initial < 0) throw new IllegalArgumentException("initial");
        remaining = initial;
    }
    synchronized boolean reserve(int quantity) {
        if (quantity <= 0) throw new IllegalArgumentException("quantity");
        if (remaining < quantity) return false;
        remaining -= quantity;
        return true;
    }
    synchronized int remaining() { return remaining; }
}
~~~

所有访问通过同一实例监视器，检查与扣减不被其他reserve插入，因此remaining不小于零。若只给减法加锁，检查在锁外仍会超卖；若多个JVM各有一个Stock，仍各自出售同一库存，需要权威存储的条件更新或事务。进程崩溃也会丢失这里的内存状态。

### 3.2 ReentrantLock何时值得用

ReentrantLock提供显式加解锁、可中断获取、限时尝试和多个条件队列。普通lock等待并不因中断直接抛InterruptedException；需要取消等待时使用lockInterruptibly或相应限时方法。获取后必须finally解锁，不能把“可能未拿到锁”的路径也执行unlock。

~~~java
boolean updateWithDeadline() throws InterruptedException {
    if (!lock.tryLock(50, TimeUnit.MILLISECONDS)) return false;
    try {
        updateState(); // 示意：短小的本地状态操作
        return true;
    } finally {
        lock.unlock();
    }
}
~~~

这里假定lock是共享的ReentrantLock字段；updateState是待实现业务方法，片段不能独立运行。超时拿不到锁与业务操作失败应区分。锁获取有50ms限制，也不意味着整个updateState只运行50ms，操作本身仍需预算。

公平锁倾向让等待更久者先获得锁，但可能牺牲吞吐，不保证操作系统调度公平；无参数tryLock可插队，不能靠公平构造参数推导所有获取方式都公平。若只是简单互斥，synchronized通常更少出错；不要背“显式锁永远更快”。

### 3.3 wait/notify与Condition为什么必须循环

等待条件必须在持锁时检查，条件不满足就await或wait，醒来后再次检查。原因包括虚假唤醒，以及唤醒到重新拿锁之间条件又被其他线程消费。signal/notify只让等待者有机会竞争锁，并不把条件和执行权直接交过去。

有界缓冲区通常有“非空”和“未满”两个Condition：生产者满时等待未满，消费者空时等待非空，修改状态后通知对应一类。await释放关联锁并在返回前重新获得；sleep不释放，因此拿sleep模拟等待会阻止别人改变条件。

优先使用成熟BlockingQueue，而不是自己实现缓冲区。若确需自定义，应测试容量1、多生产多消费、中断和关闭；关闭状态也要进入条件判断，否则等待者可能永远醒不过来。

### 3.4 AQS及读写锁：知道机制，不把实现当保证

AbstractQueuedSynchronizer（AQS）是许多同步器使用的实现框架：以状态值表达占用，通过原子状态竞争、等待队列及park/unpark等机制协调阻塞与唤醒。ReentrantLock用状态表达持有次数；具体排队和优化随JDK实现演进，业务应依赖API契约而非内部节点结构。

ReentrantReadWriteLock允许多个读者或一个写者，但读锁期间共享状态必须真的不变。读多并不自动更快，锁开销与写者等待也要测；持读锁直接升级写锁容易卡住，应重新设计获取流程。StampedLock的乐观读先取戳、读取字段，再validate，失败则在读锁下重新读；它不是可重入锁，也不能在验证前执行不可回滚副作用。没有明确收益时不应为了复杂度引入它。

## 四、CAS与原子类：把一个状态改变做成不可分割

### 4.1 比较并交换和重试循环

CAS（Compare-And-Set）把当前值与期望值比较，相同则原子写入新值并报告成功，不同则失败。调用者通常重新读值、计算和重试，避免先读再写被插入。底层可用硬件原子指令实现，但Java保证以所用API内存模式为准。

~~~java
static boolean reserveOne(java.util.concurrent.atomic.AtomicInteger stock) {
    for (;;) {
        int observed = stock.get();
        if (observed <= 0) return false;
        if (stock.compareAndSet(observed, observed - 1)) return true;
    }
}
~~~

此算法要求所有写入都保持同一库存语义。成功CAS是操作的线性化点，即可以把该操作看作在这一瞬间生效；失败必须重读，不能拿旧值永久重试。它不支持任意数量扣减的其他业务规则，也没有持久化和分布式保障。

AtomicInteger.incrementAndGet返回新值，getAndIncrement返回旧值。默认get/set和compareAndSet具备相应volatile语义；plain、opaque、acquire/release等变体有不同保证，不应看名字像就替换。updateAndGet的函数可能被重复执行，不能在函数中付款、发消息或累加外部审计副作用。

### 4.2 Lock-free不等于每个线程都不会等待

CAS原语可用于无锁（Lock-free）算法，整体进展不依赖某个持锁线程释放；但某个线程可能持续失败。Wait-free要求每次操作在有界步骤内完成，更强。使用CAS的算法若还有锁或无界依赖，不能仅凭一个CAS就宣称整体Lock-free。

高竞争时多个核反复争同一缓存行，失败重试和缓存一致性流量可能比阻塞锁更贵。自旋适合预期极短等待；长时间CPU忙转会挤压真正能推进的线程。优先使用JDK成熟原子类，不自行在业务中编写复杂无锁链表。

### 4.3 ABA：值相同不代表没有变化

线程甲读到A后暂停；乙把A改B再改回A，甲CAS看到A成功，却不知道中间发生过变化。若算法只关心当前数值，ABA未必是问题；若依赖节点从未移除或资源生命周期，它可能破坏正确性。

AtomicStampedReference把引用与版本戳一起比较，可让A/版本1与A/版本3不同，但要求所有更新正确推进版本，还需考虑版本回绕与生命周期。Java垃圾收集减少手工内存释放风险，不意味着引用复用或逻辑状态ABA自动不存在。

### 4.4 AtomicLong与LongAdder怎样选择

AtomicLong对同一个数值提供原子更新，适合需要立即得到唯一序列或做条件状态转移。LongAdder在竞争下将更新分散到多个内部单元，读取sum再聚合，通常适合高竞争统计；额外单元增加空间，低竞争也未必更优，吞吐需实测。

sum不是并发更新时的单一原子快照。不能先看sum<限额再increment来实现严格限流，也不能用它发唯一ID。统计请求量可接受近似瞬时值时适用；无并发更新时汇总可得到准确总数。AtomicLong两个方法组合“get检查后increment”也不是完整的条件原子操作，需CAS或锁。

## 五、线程池：定义接收、执行、拒绝和退出四条边界

### 5.1 参数与接收顺序

ThreadPoolExecutor管理corePoolSize、maximumPoolSize、keepAliveTime、工作队列、ThreadFactory与RejectedExecutionHandler。核心线程通常按需创建，可预启动；空闲回收默认作用于超核心线程，也可显式允许核心超时。

持续提交时：工作线程少于core先创建线程；达到core后尝试入队；入队失败再尝试扩到maximum；仍无法接收则拒绝。并发关闭也可能导致拒绝。execute使用非阻塞入队，不会因为队列叫BlockingQueue就自动put等待。

例：core=2、max=4、队列容量3，且已有任务一直占用。前2个执行，第3至5个排队，第6至7个扩线程执行，第8个拒绝。若任务提前结束，编号结果会改变，因此实验应使用门闩（Latch）稳定时序，不能只sleep猜测。

### 5.2 队列与拒绝怎样影响系统

| 选择 | 行为与适用边界 |
|---|---|
| 有界ArrayBlockingQueue或显式容量LinkedBlockingQueue | 限制积压；满后扩线程/拒绝，需要定义调用方响应 |
| 默认LinkedBlockingQueue | 容量极大，通常持续积压而不扩到max；风险是高排队延迟与内存 |
| SynchronousQueue | 不存任务，直接交接；没有空闲接收者时触发扩线程，达到上限拒绝 |
| AbortPolicy | 抛RejectedExecutionException，便于明确失败和限流响应 |
| CallerRunsPolicy | 未关闭时提交者自己执行以减慢生产；可能阻塞事件循环或持锁线程 |
| DiscardPolicy | 静默丢当前任务，只适合明确允许丢失且可观测的工作 |
| DiscardOldestPolicy | 丢队头再重试；会改变等待任务结果，不应当通用可靠方案 |

无界队列不是永不拒绝：关闭等状态仍可拒绝。CachedThreadPool允许线程大量增长，FixedThreadPool默认排队边界宽松；工厂方法不是禁用清单，问题是配置是否符合资源上限。

队列容量应结合请求截止时间和处理能力。若实际完成1000任务/秒，积压10000个就可能带来约10秒清空尺度，不能因内存装得下就接受。平均等待不能保证p99，需测突发和长任务；过期任务应尽量在开始前检查，避免用户已超时还耗尽下游。

### 5.3 一个可复用的有界池配置片段

~~~java
AtomicInteger sequence = new AtomicInteger();
ThreadFactory factory = task -> {
    Thread t = new Thread(task, "quote-io-" + sequence.incrementAndGet());
    t.setUncaughtExceptionHandler((thread, error) ->
        System.err.println(thread.getName() + ": " + error));
    return t;
};
ThreadPoolExecutor pool = new ThreadPoolExecutor(
    8, 16, 30, TimeUnit.SECONDS,
    new ArrayBlockingQueue<>(64), factory,
    new ThreadPoolExecutor.AbortPolicy());
~~~

需要import java.util.concurrent.*及java.util.concurrent.atomic.AtomicInteger，片段放在初始化方法内。8/16/64是教学参数，应由CPU、等待、内存、延迟与下游配额校准。线程命名使dump能关联业务；示例stderr仅用于演示，生产日志应控制敏感数据并关联任务标识。

ThreadFactory设置的异常处理器能观察线程未捕获异常，却不能替代任务结果处理。对submit，异常通常被Future保存，若没人get或检查就可能沉默；这是下一节的重要边界。

### 5.4 线程数从预算推导，再用压测校准

CPU密集任务的并行执行数从可用CPU预算附近起步，过多线程增加切换并不创造算力。I/O密集任务可使用“线程数≈核数×目标CPU利用率×(1+等待时间/计算时间)”作初步假设，但前提是任务组合稳定且没有更紧下游限制，不能当自动调参公式。

假设8核、每任务计算10ms等待90ms，取满CPU时公式给80线程；若数据库只允许该服务20个并发事务，80个任务可能主要排队，应另用信号量或连接池限制下游。容器CPU配额也不同于物理主机全部核数。

按依赖或业务重要性隔离池称为舱壁（Bulkhead）：报表慢不能耗尽交易池。但隔离太细会闲置资源并增加运维复杂度，需要独立指标、容量和总预算；把所有任务扔公共池的简单方式则容易互相拖累。

## 六、Future、异常、超时和中断：取消是一种协作协议

### 6.1 execute与submit的失败出口

execute直接运行Runnable，未捕获异常走线程异常处理路径；submit包装任务并返回Future，get抛ExecutionException时应检查cause。Future成功完成证明任务正常返回，不证明它调用的外部系统没有其他异步失败。

get(timeout)只限制调用者等待，不自动停止任务。超时后可发起cancel(true)，但这只是请求中断，任务可能忽略，外部写也可能已完成。取消成功不等于回滚付款；重试外部副作用需业务幂等键或查询结果，不能仅凭超时判定未执行。

CompletableFuture的组合可把依赖表达清楚，但默认异步执行常涉及公共ForkJoinPool，阻塞工作应评估专用Executor。orTimeout使Future超时完成，不保证底层工作停止；CompletableFuture.cancel的mayInterruptIfRunning也不能当作中断执行线程的承诺。不要把所有Future类型的取消语义混为一谈。

### 6.2 正确处理中断

interrupt设置协作取消信号；sleep、wait、BlockingQueue.take等响应中断抛InterruptedException时通常清除标记。当前层能向上抛则传播；不能传播时恢复标记并退出，不能恢复后继续无限工作。

~~~java
try {
    while (!Thread.currentThread().isInterrupted()) {
        Task task = queue.take(); // Task和queue为业务类型与共享队列
        process(task);
    }
} catch (InterruptedException cancelled) {
    Thread.currentThread().interrupt();
    // 退出工作循环；清理资源放finally或try-with-resources
}
~~~

这里是流程片段，process必须自己遵守取消与资源清理约定。Thread.interrupted读取并清除当前线程标记，isInterrupted只读取；误用前者会把取消信号消费掉。监视器获取本身不提供可中断失败出口，显式锁可选择lockInterruptibly。

### 6.3 停机与嵌套任务饥饿

shutdown拒绝新任务并处理已提交任务；shutdownNow尽力中断执行者并返回尚未开始任务，不能保证强制终止。不响应中断的I/O需要客户端超时或资源关闭机制。应用必须等待有限时间、记录未完成状态并决定重投，不能只调用shutdown就声称安全下线。

~~~java
pool.shutdown();
try {
    if (!pool.awaitTermination(5, TimeUnit.SECONDS)) {
        java.util.List<Runnable> notStarted = pool.shutdownNow();
        // 按业务要求持久记录/取消notStarted，而不是静默丢弃
        if (!pool.awaitTermination(5, TimeUnit.SECONDS)) {
            System.err.println("pool still running");
        }
    }
} catch (InterruptedException cancelled) {
    pool.shutdownNow();
    Thread.currentThread().interrupt();
}
~~~

ExecutorService停机不自动把shutdownNow返回的Future任务全部标为取消，使用者应处理对应句柄与业务状态；示例不提供持久化实现。

若固定池两个工作线程都运行父任务，父任务又向同池提交子任务并阻塞get，子任务只能在队列等空闲线程，形成线程饥饿死锁。它不一定表现为传统监视器死锁。避免同池阻塞依赖，使用组合、不同资源域或改变任务结构；不能只把池从2调成20掩盖无限嵌套。

## 七、并发容器：选择的是操作语义与读取一致性

### 7.1 ConcurrentHashMap的原子边界

ConcurrentHashMap支持并发读写，不允许null键值，便于区分缺失。它不是给整个Map一把全局锁；现代实现通过不同并发机制协调桶和更新，具体布局属于实现细节。

putIfAbsent、compute、merge可表达单键复合操作。用containsKey再put仍有竞态。computeIfAbsent回调应短小，不能随意递归更新或进行昂贵、不可回滚的外部调用；发生异常、返回null或映射删除后，将来调用仍可能再次计算，不是业务“永远只执行一次”。

Map线程安全不使值对象自动安全：Map<K, ArrayList<V>>中的ArrayList仍需同步或改用不可变值/并发结构。遍历通常弱一致，可见部分并发更新，不是全局快照；size也不适合在并发写时决定严格配额。跨两个键转账需要更高层协议，单键compute不是跨键事务。

### 7.2 BlockingQueue与生产消费

有界队列限制积压，但背压取决于操作：offer立即失败，限时offer有限等待，put等待空间或中断；消费poll返回缺失，take等待。生产者应把满队列反馈为限流、等待或明确失败，而不是无限重试占CPU。

入队前对元素的写可见于取得元素的消费者；入队后生产者继续修改同一可变对象，仍可能竞争。可传递不可变任务或移交所有权。队列线程安全也不等于任务可靠：消费者取出后崩溃会丢失内存任务，需持久消息系统与确认机制满足相应可靠性要求。

### 7.3 CopyOnWriteArrayList与其他结构

CopyOnWriteArrayList写时复制并发布新数组，迭代者持有旧快照，适合数量有限、读远多于写的监听器名单。它避免遍历与修改互相阻塞，但写入O(n)、瞬时双份数组与旧快照保留会增加内存和GC；修改后已有迭代器不会立即看见新监听器，迭代器也不支持remove。

ConcurrentLinkedQueue适合非阻塞FIFO，容量不受其自身上限保护；不需要等待不等于不需要过载控制。ConcurrentSkipListMap支持并发排序和范围查询，通常比纯哈希查找付出更多结构成本。容器应按顺序、容量、等待、快照与复合更新要求选，而非都换成名字带Concurrent的类。

## 八、同步工具与上下文：限制资源、等待完成和隔离身份

### 8.1 Semaphore、Latch、Barrier

Semaphore是许可证计数，可限制最多N个任务同时访问某依赖。先成功获取才可在finally释放；获取失败或被中断时不能多release。它不是自动与线程绑定的锁，释放次数错误会扩大配额。只限制执行并发而不限制等待者数量，仍可能堆积大量请求。

CountDownLatch让等待者等计数到零，通常用于启动门或一次完成汇聚，不能重置；任务失败也应在finally countDown，并把失败结果独立保存，否则主线程只知道“已结束”不知道“成功”。CyclicBarrier适合一组参与者重复阶段对齐，某参与者失败或超时会影响这一轮。生产服务处理独立请求通常不需要人为凑齐一批才能前进。

### 8.2 ThreadLocal泄漏与跨请求污染

ThreadLocal给每个线程独立值，不是让同一对象自动线程安全。线程池线程活得久，请求结束不remove会让下个任务沿用前个用户/租户身份，这是正确性和安全问题。典型实现中键弱引用不代表值能立刻回收：存活线程内的过期条目可能继续持有值，直到后续清理。

~~~java
try {
    tenantContext.set(tenantId);
    handleRequest();
} finally {
    tenantContext.remove();
}
~~~

以上变量为业务示意。若是可嵌套上下文，内层退出可能需要恢复旧值再由最外层清理，而非一律删除。异步任务可能换线程，ThreadLocal不会自动传过去；应显式捕获必要上下文、在任务边界设置和恢复，且避免把整个请求对象长期挂住。

不要把InheritableThreadLocal当线程池请求传播方案：线程创建时的继承与每次任务提交不是一回事。日志MDC、租户身份和追踪需要共同遵守清理规则。代码只清日志却残留授权身份仍可能串租户。

## 九、死锁、饥饿、活锁与竞态：从现象找到等待关系

死锁（Deadlock）是互相等待使参与者无法前进，例如甲持A等B、乙持B等A。对多资源使用统一、稳定的锁顺序；在获取第二把锁失败时释放第一把，必要时限时重试；避免持锁调用网络、数据库或未知回调。缩小临界区不能替代正确锁顺序。

饥饿（Starvation）是部分任务长期得不到资源，可能来自长任务占满池、优先级或不公平竞争；公平锁只影响某种锁竞争，无法修复数据库或线程池饥饿。活锁（Livelock）是线程一直忙着让步或重试却没有业务进展，例如双方同时释放再同时抢；可用随机退避、集中协调或稳定顺序减少同步碰撞。

竞态条件是结果依赖未控制时序。常见反例为“余额检查与扣款分开锁”“并发Map里存普通List”“任务超时后无幂等重试”。修复先写不变量和线性化边界，而不是盲目给每个getter加synchronized。

本地锁只能协调共享同一锁对象的线程。多个服务节点、数据库事务与外部付款要在权威资源上保证条件更新、隔离与幂等；本地锁对进程崩溃也没有恢复协议。把分布式问题错当线程问题，是架构面试的重要扣分点。

## 十、虚拟线程：释放等待成本，不放大下游能力

JDK 21正式提供虚拟线程（Virtual Thread），大量任务可在等待兼容阻塞I/O时让出载体线程（Carrier Thread，实际执行Java代码的平台线程）。它减少每任务绑定OS线程的成本，不让CPU代码更快，也不创造数据库连接或第三方配额。

适合大量独立阻塞I/O任务；CPU密集工作仍受核数制约。通常每任务一个虚拟线程，不应为了复用虚拟线程再造固定虚拟线程池；限制稀缺资源应用Semaphore、连接池与入口接纳控制。仅限制数据库执行而允许百万等待任务，仍可能耗尽请求状态内存。

版本边界很重要：JDK 21中，虚拟线程在某些synchronized和native相关阻塞路径可能被固定在载体上（Pinning）；后续JDK已改变监视器相关行为，不能把21的所有限制套在25。Oracle JDK 25指南已不把synchronized列为固定原因，仍说明native方法或外部函数相关固定情形。选择锁仍先看语义、实际JFR证据与目标JDK，不为“虚拟线程必定不能用synchronized”做机械替换。

迁移时核查线程本地大对象、驱动与客户端兼容性、取消超时、连接限制及观测能力；JDK 21传统线程dump对虚拟线程展示与专用dump能力不同，应使用目标JDK支持的工具。结构化并发等API有版本与预览边界，本章不依赖它们实现生产方案。

## 十一、工程案例：给慢下游加并发，怎样避免拖垮交易

假设报价聚合需要访问三个供应商，每请求截止时间300ms，一个供应商有时阻塞2秒。顺序调用可能超预算，盲目并发可以降低正常端到端等待，却把入口每秒1000请求变成最多3000外部调用，供应商与本地连接可能先耗尽。

先定义结果语义：是否必须三家全到、允许部分报价吗、过期报价能否用、失败是否收费。若允许部分报价，可独立隔离每供应商并发、设置连接/读取超时，按剩余截止时间等待，并在返回后请求取消不再有价值的任务；取消仍不保证远端停止或不收费。

线程池或虚拟线程只是执行模型，入口还需有限接纳、每供应商许可与重试预算。假设某供应商每次平均占用100ms、允许200并发，理想稳定吞吐上界约2000调用/秒，但长尾和失败会降低有效能力；这不是容量承诺，应按真实分布压测。

拒绝应映射为可观察的业务降级或明确失败，不能静默返回全空报价伪成功。监控每供应商排队时间、调用时间、许可占用、超时/拒绝/取消和有效结果比例。资源隔离保护其他供应商，但会增加配置与常驻预算，需要按依赖风险维护。

## 十二、排障与性能验证：dump、JFR和指标如何互证

### 12.1 先看症状，再找资源链

先分清CPU高、延迟高、吞吐降还是内存涨。记录池active/pool size、队列长度与最老等待时间、完成率、拒绝、超时、下游连接等待。active只是线程在执行任务，不证明在有效工作，可能都阻塞在一个外部调用。

Thread Dump（线程转储）看多次快照中的线程栈和锁拥有者：大量线程等同一锁，查持有者；大量线程等Future，追子任务是否还在同池队列；都在套接字读取，核查客户端超时和下游。一次WAITING可能完全正常，不应直接杀进程。

JFR（Java Flight Recorder）以事件方式观察执行、锁竞争、分配和阻塞等，结合目标JDK事件及采样配置分析。不要凭“没看到事件”断言无竞争，阈值和采样可能遗漏。CPU忙的CAS自旋可能表现为热点代码而非锁等待，需结合栈与CPU采样。

### 12.2 验证正确性与吞吐是两类工作

正确性先列不变量，如库存非负、同意图最多一次、关闭后无新接纳、上下文不跨租户。用门闩/屏障构造关键交错，验证失败、中断、拒绝与停机，不靠sleep作为唯一同步。

压力测试未发现错误不证明所有交错正确；JMM专门测试可用jcstress，微基准可用JMH，但它们需要正确的场景、预热和结果解释。实际服务压测还要固定请求组合、CPU预算、下游延迟与错误率；不能用空任务基准证明某锁在业务中永远更快。

如果加线程后吞吐不涨而数据库等待变长，应减少并发并修查询/事务或下游容量；若CPU已满，减少重复工作或扩CPU可能有效。优化目标是目标延迟下的有效结果与成本，不是线程数或总尝试次数最大。

## 十三、可复制的正确性实验与阅读结果的方法

下面完整类使用固定线程池、Future结果传播和有界退出验证CAS扣减。保存为CasStockDemo.java，在完整JDK 21执行javac CasStockDemo.java再java CasStockDemo。它只验证这次实验结果，不是对所有交错的形式证明。

~~~java
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;

public class CasStockDemo {
    static boolean reserveOne(AtomicInteger stock) {
        for (;;) {
            int old = stock.get();
            if (old <= 0) return false;
            if (stock.compareAndSet(old, old - 1)) return true;
        }
    }
    public static void main(String[] args) throws Exception {
        AtomicInteger stock = new AtomicInteger(1000);
        AtomicInteger accepted = new AtomicInteger();
        ExecutorService pool = Executors.newFixedThreadPool(4);
        List<Future<?>> futures = new ArrayList<>();
        try {
            for (int i = 0; i < 8; i++) {
                futures.add(pool.submit(() -> {
                    for (int attempt = 0; attempt < 500; attempt++) {
                        if (reserveOne(stock)) accepted.incrementAndGet();
                    }
                }));
            }
            for (Future<?> future : futures) future.get(10, TimeUnit.SECONDS);
            if (stock.get() != 0 || accepted.get() != 1000) {
                throw new AssertionError("stock=" + stock + ", accepted=" + accepted);
            }
            System.out.println("remaining=0, accepted=1000");
        } finally {
            pool.shutdownNow();
            if (!pool.awaitTermination(5, TimeUnit.SECONDS)) {
                throw new IllegalStateException("pool did not terminate");
            }
        }
    }
}
~~~

任务总共尝试4000次，库存初始1000，只有CAS成功者计数，因此期望1000次成功、库存0。Future.get让工作线程异常传到main，也让主线程等待完成后再观察；若仅提交就立即打印，看到部分结果完全可能是主线程没等。

实验只提交8个短任务，所以这里固定池的默认队列有明确实验负载上界，不意味着推荐生产入口无界排队。accepted与stock分别更新，运行中同时读取不能当它们是原子快照；所有任务结束后才验证配套不变量。若模拟任务抛异常，程序应失败而不是吞掉后打印通过。

验证限制：本次环境有OpenJDK 21运行时但没有javac/jshell，以上Java示例未编译运行，已人工检查导入、异常路径、等待与退出。不得将期望输出宣称为实测结果；生产锁性能、虚拟线程和故障行为仍需在目标环境验证。

## 十四、面试完整回答与深入追问

**问：你会怎样保证并发正确？**

“我先确定权威共享状态和必须同时保持的业务不变量。单个JVM库存检查加扣减，可以让所有访问经过同一把锁；单变量简单更新也可用CAS循环。volatile只解决相应可见性和顺序，不能保护count++。跨节点库存要靠数据库条件更新或事务等权威协议，本地锁不够。随后定义超时、取消、重试和崩溃结果，用可控交错测试，并用池指标、线程栈和JFR验证资源竞争。”

**追问：CAS是否总比锁快？**

“不是。短小低冲突时它可避免阻塞，但高冲突下失败重试会争同一缓存行。锁能让等待者休眠，多字段不变量也更容易表达。要比较相同正确性、竞争分布和目标JDK下的业务性能，不把无锁等同零成本。”

**问：如何设计线程池？**

“先区分CPU与阻塞任务，估计计算与等待，再以CPU、下游并发、连接、内存和截止时间限制参数。核心线程、最大线程与队列共同决定接纳顺序，必须说明满了怎么处理。比如有界队列加AbortPolicy，让入口明确降级；还要观测排队/完成/拒绝和安全退出。增加线程不是数据库过载的解法。”

**追问：CallerRunsPolicy不是天然背压吗？**

“它通过占用提交线程减慢生产，但如果提交者是事件循环，可能把所有请求卡住；持锁提交还会延长临界区。池关闭时该策略不运行任务。需要明确提交者身份与可靠性，不能当万能拒绝策略。”

**问：ConcurrentHashMap是否能保证只扣一次钱？**

“它提供单键并发操作，不提供外部付款事务。即便compute回调对当前映射有原子性，外部调用也可能在超时后成功，重试会再次付款。必须用业务幂等、权威状态及对账恢复，不能让内存回调的执行范围冒充端到端恰好一次。”

**追问：Future超时后重试安全吗？**

“get超时只是调用者不再等，任务或远端操作可能继续。先请求取消并记录未知结果，再按幂等键查结果或重试；无法保证幂等的副作用要有专门恢复，不直接当未执行。”

## 十五、扫雷自测与完整答案

### 15.1 volatile int count执行count++为什么丢更新？

读、加、写不是一个原子动作，两个线程可读到同一旧值再写同一新值。volatile不合并这三个步骤。改用原子增量、覆盖整个复合操作的同一把锁或单线程归属；若多个变量还需一起一致，单个原子计数也未必足够。

### 15.2 core=2、max=4、队列3，八个一直阻塞的任务会怎样？

前2个由核心线程执行，中间3个排队，第6、7个因入队失败扩到最大4线程，第8个拒绝。前提是之前任务未完成且池正常运行；并发完成和停机会改变结果。不能说先开4线程才排队。

### 15.3 一个ConcurrentHashMap里的ArrayList为什么仍会损坏？

Map只保护映射操作，取出的可变列表仍可能被多个线程同时add。选择每键锁、不可变替换或合适并发值，并明确遍历快照与顺序要求。把容器换Concurrent不会递归保护对象图。

### 15.4 AtomicLong和LongAdder能否用于严格余额？

LongAdder.sum不是并发原子快照，不适合严格条件扣款；AtomicLong也必须用CAS等把检查与扣减合起来。若余额还涉及账目、事务和持久化，两个类都不能直接替代数据库账本。

### 15.5 线程池任务超时为何仍可能产生副作用？

等待超时不等于停止执行，中断也只是协作请求；远端服务可能已提交。需要端到端截止时间、客户端超时、幂等键和结果查询。取消本地Future不意味着撤销已发生付款。

### 15.6 为什么等待Condition要while而非if？

允许虚假唤醒，而且其他线程可在等待者重新拿到锁前消费条件。循环在锁保护下重新验证不变量，不满足继续等待；同时应处理关闭与中断，避免无限等待。

### 15.7 ThreadLocal键是弱引用，为何还会泄漏和串租户？

实现中存活线程仍可能通过条目持有值，键回收不使值立即回收；线程复用还会让未清理值被下一任务读取。用任务边界finally清理或恢复旧上下文，异步传播显式处理，不能只依赖GC。

### 15.8 dump未报告死锁，为什么所有请求仍卡住？

可能是同池父任务等子任务、下游I/O无超时或连接耗尽，不是监视器环。结合队列、Future等待栈、连接池与完成率找到资源依赖；增线程只可能暂缓，不能消除无限依赖。

### 15.9 换虚拟线程能否取消数据库并发上限？

不能。等待线程便宜不代表数据库连接、锁与CPU无限。保留入口接纳和下游许可证，并按目标JDK验证驱动、取消与Pinning。否则只是更便宜地产生更多等待者和更猛烈的连接压力。

## 十六、复习连接与速记

并发正确性来自可证明的状态边界，稳定性来自可解释的资源边界。JMM保证发布与观察；锁和CAS决定状态转移；并发容器提供特定操作契约；线程池与许可证限制工作量；超时、取消和幂等决定失败后能否恢复。这些层次必须一起设计。

速记：volatile不让复合动作原子；同锁保护整个不变量；CAS重试无副作用；LongAdder用于统计；先core再队列再max再拒绝；get超时不停止工作；中断要传播或恢复退出；Map安全不等于值安全；ThreadLocal任务边界清理；虚拟线程不扩下游；dump看等待链，JFR和指标看证据。

## 十七、来源与验证范围

核查日期：2026-11-10。以下为规范与官方API/指南，供核验，不替代正文。基础语义按Java SE 21；JDK 25只用于说明虚拟线程版本边界，未将实现内部结构当成语言保证。

- [JLS 21，第17章 Threads and Locks](https://docs.oracle.com/javase/specs/jls/se21/html/jls-17.html)：监视器、内存模型与happens-before。
- [Java SE 21 java.util.concurrent](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/package-summary.html)：任务、队列和同步工具的内存一致性及用途。
- [Java SE 21 ThreadPoolExecutor](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/ThreadPoolExecutor.html)：参数、排队与拒绝。
- [Java SE 21 ReentrantLock](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/locks/ReentrantLock.html)：获取、释放、公平性与扩展能力。
- [Java SE 21 CompletableFuture](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/CompletableFuture.html)：异步执行与取消语义。
- [Java SE 21 LongAdder](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/atomic/LongAdder.html)：统计与sum边界。
- [Java SE 21 atomic包](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/atomic/package-summary.html)：单变量原子操作和相关类。
- [JDK 21 Virtual Threads](https://docs.oracle.com/en/java/javase/21/core/virtual-threads.html)及[JDK 25 Virtual Threads](https://docs.oracle.com/en/java/javase/25/core/virtual-threads.html)：执行模型、采用与观测边界。

原主题覆盖：JMM三性质、happens-before与volatile；CPU/I/O线程池、队列/工厂/拒绝/指标/退出；synchronized、ReentrantLock、CAS/ABA、自旋、AtomicInteger/LongAdder；三类原有并发容器；死锁/饥饿/活锁/竞态/ThreadLocal；锁顺序、临界区、外部调用和JFR/dump。所有原薄弱章节均已原位整合，面试与自测附解释；Java未执行验证的限制已在实验章节注明。
