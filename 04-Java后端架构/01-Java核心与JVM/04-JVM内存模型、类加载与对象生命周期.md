# 04. JVM内存模型、类加载与对象生命周期

## 学习目标与版本范围

本章把“对象存在哪里、为什么仍活着、类何时初始化、进程为什么被杀”连成一套可以用于设计与排障的推理。需要区分三个层次：JVMS定义的抽象运行时区域、HotSpot的具体实现、操作系统与容器的内存计量。Java Memory Model（JMM）另行规定并发读写可观察行为，不能用Heap/Stack示意替代。

规范与命令以Java SE 21及HotSpot JDK 21为基线；具体收集器、对象布局、压缩指针、默认参数和诊断命令按目标JVM核实。案例和预算均为教学假设，不代表用户实际项目。目标是能解释机制、辨认证据、提出可验证修复，而不是背一串内存区域名。

## 一、先建立两张地图：规范区域与进程资源

### 1.1 运行时数据区不是固定物理分区图

JVMS定义每线程的程序计数器、JVM栈，以及共享Heap和Method Area等概念。虚拟机实现可选择不同物理布局、垃圾收集和优化方式，因此“Java规定对象头固定多少字节”“方法区一定是某块连续内存”都过于绝对。

程序计数器（pc）用于标识当前线程正在执行的JVM指令位置；执行native方法时其值在规范中未定义。JVM栈保存调用帧（Frame），帧内含局部变量表、操作数栈和动态链接等信息；方法调用入帧、正常或异常返回出帧。异常退出不等于堆对象立即释放。

Native Method Stack（本地方法栈）服务于实现所需的native调用，具体是否以及怎样与平台线程栈结合由实现决定；它不等同于存储所有native对象的内存池。

局部变量表可以保存基本值或对象引用。引用位于某帧，不代表它指向的对象也必定位于栈。HotSpot可以优化掉部分对象分配，规范上的对象语义与最终机器码布局应分开理解。

### 1.2 规范区域和HotSpot实现如何对应

| 区域/资源 | 主要内容与寿命 | 关键边界 |
|---|---|---|
| Heap（堆） | 对象、数组的运行时存储；由GC管理 | -Xmx只限制堆上限，不限制整个进程 |
| JVM Stack（栈） | 平台线程调用状态、帧和局部值 | 递归深度、线程数及-Xss共同影响资源 |
| Method Area（方法区） | 规范中的类级运行信息、运行时常量池等 | 不应简单等同全部物理元空间 |
| Metaspace（元空间） | HotSpot保存大量类元数据的native内存 | 类加载器生命周期会影响回收 |
| Code Cache（代码缓存） | HotSpot编译后的机器码等 | 不属于Java堆，影响JIT与性能 |
| Native/Direct | 直接缓冲、JNI库及其他native资源 | 不都在堆dump里，也不都由NMT完整统计 |

运行时常量池是每类/接口的符号和常量信息，不等于把所有String对象都放进Metaspace。Class镜像对象、静态字段引用所指对象等与类有关的数据也不意味着全部物理位于元空间。面试应先讲规范职责，再说明实现映射，而非把方法区、元空间、常量池混成同一个词。

### 1.3 reserved、committed、used与RSS

Reserved是保留虚拟地址范围；Committed是JVM已提交可供使用的内存口径；Used是某区域已用量。RSS（Resident Set Size）反映进程当前驻留物理页，并不等于所有reserved的总和。不同工具对共享页、文件映射、堆内空闲页的统计也有差异。

因此看到预留32GB不等于容器消耗32GB物理内存，堆used降了也不保证RSS立即降。GC回收对象、收集器归还内存、操作系统回收页是不同过程；是否及何时归还依实现、配置和负载而定。

容器cgroup的内存计量又不等于单个JVM RSS：可能含同容器其他进程、文件缓存与内核相关计费项。应按实际cgroup版本与监控口径对齐，不能用“容器减堆used”就精确推断native泄漏。

## 二、堆与栈：生命周期和故障方式怎样不同

### 2.1 方法返回为何不一定让对象消失

方法创建List并放入static缓存，方法返回只结束调用帧，缓存仍引用List。反过来，一个方法尚未返回，某局部对象以后不再使用，优化后的可达性判断可能不要求它活到源代码作用域结束。源码大括号不是严格GC时间表。

递归时每次调用都需帧，过深可能StackOverflowError。增大-Xss可以容纳更深调用，但会提高大量平台线程下的栈预算，也不能修复无终止递归。线程数量过多导致无法创建native线程，可能涉及内存、OS线程或进程数限制，并不是“堆再加大就能解决”。

### 2.2 虚拟线程不要套平台线程栈乘法

在HotSpot虚拟线程实现中，挂起状态的栈可由堆中的栈块承载，载体平台线程仍有自己的栈。不能把每个虚拟线程都机械按完整-Xss承诺量乘一遍，也不能说虚拟线程没有栈或没有堆成本。

大量等待请求会保留参数、上下文和缓冲，虚拟线程降低OS线程成本，却可能让更多对象同时存活。内存预算仍须看在途任务与真实对象，而不只看平台线程数。

## 三、对象从new到可回收：机制、优化与观察边界

### 3.1 创建顺序与构造期间逸出

对象创建涉及确定类、所需初始化、分配空间、字段默认值及实例初始化/构造器等语义。字段先有默认零值、null、false，再按规定执行初始化和构造逻辑；不能假设构造器一开始所有业务字段都已赋值完成。

构造器调用可重写方法，子类字段可能尚未初始化；构造期间把this注册到其他线程可使外部观察到未完整状态。构造完成后的安全发布与final字段初始化语义都需要尊重边界，不能把new返回的引用随意无同步共享。

### 3.2 分配快路径、TLAB与大对象

HotSpot常用指针推进等快速分配方式；TLAB（Thread-Local Allocation Buffer）是从堆中划给线程的分配缓冲，减少多个线程争同一分配指针。TLAB仍是堆，不是线程私有的GC免管区，也不使对象只能被该线程访问。

对象布局通常含头部、字段/数组内容与对齐填充，但具体字节数受JDK、压缩指针、对齐和类结构影响。估算容量应测目标配置下实际大小及保留路径，不能只加字段声明大小。

“对象通常在Eden分配”是对某些分代收集器正常路径的概括。以JDK21 G1为例，堆按Region组织，年轻代和老年代是角色集合；较大Humongous对象有特殊分配路径，不能承诺每个对象都走Eden→Survivor→Old。其他收集器的布局与晋升语义也可能不同。

### 3.3 存活、晋升与分配压力

分代思想利用许多对象寿命短这一常见分布。年轻对象经过收集后仍可达，可能复制到Survivor或晋升老年代；年龄、空间与收集器策略影响路径，晋升阈值不应当语言规范背诵。

大量短命对象可能让GC频繁、CPU升高但存活基线稳定；少量长寿命对象持续被保留可能造成存活基线增长。两者都表现“分配很多”，修复方向却不同：前者看不必要中间对象和缓冲复制，后者看持有者与生命周期。换收集器不能让仍被强引用的无用业务对象自动消失。

### 3.4 Escape Analysis与标量替换

逃逸分析（Escape Analysis）判断对象是否逃出某个优化范围。如果对象无需作为可观察独立实体存在，JIT可能通过标量替换（Scalar Replacement）把字段拆成计算值，甚至消除整个分配和相关锁。

~~~java
record Point(int x, int y) {}
static int distanceLike(int x, int y) {
    Point p = new Point(x, y);
    return p.x() * p.x() + p.y() * p.y();
}
~~~

这是方法片段；足够优化后可能不再真正分配Point，但是否发生取决于编译、内联和实际使用。不能把“不逃逸”直接说成“JVM必定把对象分配到线程栈”，也不能从一段微基准推导业务中所有new都免费。

若把Point存进共享集合、作为身份对象同步或通过复杂调用暴露，优化条件会改变。性能测试应防常量折叠和无用代码消除，用目标负载的分配事件/Profiler验证，不为凑基准捏造分配次数。

## 四、GC Roots与泄漏：业务不再需要不等于不可达

### 4.1 引用图比引用计数更重要

追踪GC从根出发标记可达对象。根的具体集合依实现和时点，常见有线程栈中的活引用、活跃类静态状态关联引用、JNI全局引用等。两个对象互相引用但不从任何根可达，仍可回收；“循环引用必泄漏”是把引用计数直觉错误套用。

通常强可达对象不能作为垃圾回收。这里不能泛化成“任何Reference语义的可达都不会回收”，软/弱/虚引用有专门处理规则。可达性还可能在并发标记与程序运行中变化，不要把一次dump视为整个历史。

### 4.2 Strong、Soft、Weak、Phantom与资源关闭

强引用维持普通对象生存；SoftReference可在内存压力相关策略下被清理，但不能据此获得可预测容量和延迟；WeakReference不以普通强保留方式阻止对象回收，适合某些关联表；PhantomReference配合ReferenceQueue用于回收相关通知，其get不返回目标。

弱键容器也有边界：若值通过别的强路径又指回键，可能仍保留；弱引用不是一键修复缓存策略。业务缓存应优先明确上限、过期和失效，不把SoftReference当可靠淘汰算法。

GC释放内存不等于及时关闭文件、套接字、事务或native句柄。优先AutoCloseable和try-with-resources，按协议显式释放。Cleaner等清理机制只适合作兜底，时机无确定保证；finalize已被弃用并朝移除方向演进，不应用来承担业务资源正确性。

### 4.3 常见泄漏链如何成立

无界缓存：存活类的static Map持有每个键和值，访问结束也不删除；监听器：长寿命事件总线持有短寿命页面/会话；ThreadLocal：池线程复用而任务不remove，值可能跨请求保留并串身份；排队任务：队列持有请求体、Future和上下文，过载把临时对象变成长寿命对象。

每条链都要回答谁是owner、何时应释放、上限是什么。把值设null但其他集合仍有引用无效；强制System.gc也无效。修复可能是有界缓存、取消注册、任务finally清理、有限排队或缩小捕获对象，而不是简单调大-Xmx。

## 五、类加载：同名不一定是同一个类型

### 5.1 Loading、Linking、Initialization分开理解

Loading找到类的二进制表示并创建对应运行时类型；不要求来源一定是磁盘.class，也可由网络、生成字节码或其他受支持来源提供。Linking包含验证、准备和解析：验证检查合法性与类型安全等；准备分配静态状态并建立初始值；解析把符号引用连接到实际运行时实体。

普通静态字段准备时先有默认值，显式初始化通常在Initialization执行；常量变量的处理有特殊规则，不能说所有static final都等到clinit才赋值。解析可以延迟，规范约束必须保持，但不要把三阶段内部实现说成一次不可拆开的固定时间线。

Initialization执行类初始化方法<clinit>，源自静态字段初始化器和静态代码块，顺序受声明与继承规则约束。构造器<init>初始化实例，不是<clinit>；同类可以有很多实例，但每个运行时类的初始化只会成功完成一次。

### 5.2 什么操作触发初始化

new实例、调用该类声明的静态方法、读写该类声明的非编译期常量静态字段等主动使用会触发相应初始化；反射某些操作也会触发。引用子类名读取实际由父类声明的静态字段，不能据此断言子类也初始化。

类字面量Foo.class不等于初始化Foo；Class.forName(name)通常触发初始化，带initialize=false与明确加载器的重载可只加载/链接相关处理而不主动初始化；ClassLoader.loadClass通常不主动初始化。数组类型创建也不能简单说成初始化元素类。

常量内联是陷阱：static final int VERSION=1可作为常量变量被调用方内联，读它未必初始化声明类；static final Integer BOXED=1不是同样的编译期常量。更新提供方常量但不重编译消费者，可能仍读取旧内联值。

### 5.3 Parent Delegation的收益与边界

JDK常见加载器包括Bootstrap、Platform、Application及自定义加载器。通常的父优先委派先检查已加载类，再委托父级，父级找不到再由自身查找，帮助避免核心类重复定义和版本混乱；Bootstrap在Java API中常用null表示，不能据此说不存在加载器机制。

父优先是常用ClassLoader策略，不是所有加载场景不可改变的语言铁律。插件或容器可对子包采用子优先隔离，但Java平台命名、安全与模块约束依然存在；不能通过随便写个java.lang.String就替换核心类。

类型身份由二进制名称和定义加载器等运行时规则决定。两个加载器各定义com.example.Plugin，哪怕字节完全相同，也可能不能互相强转；ClassCastException里两边类名一样时，要查加载器而不只查包名。

### 5.4 插件边界如何落地

稳定SPI接口由共同父级加载器定义，各插件加载自己的实现与私有依赖；跨边界传父级可见接口或稳定DTO，而非泄露插件私有类。SPI即Service Provider Interface，用来约定可插拔实现的调用契约。

线程上下文类加载器（TCCL）允许某些框架/服务发现从调用上下文查找实现，解决父级代码需要发现子级实现的问题，但不能把它当所有加载都自动倒置。临时切换TCCL应finally恢复，避免线程池任务影响下一任务及阻止插件卸载。

热加载不同版本需要管理版本隔离、线程、资源与引用图；“新建ClassLoader就完成热部署”忽略了旧版本的生命周期，也可能让类数量与Metaspace不断上涨。

## 六、类初始化的同步、失败与死锁

### 6.1 初始化为何能提供安全发布

类初始化有同步过程，其他线程主动使用时需要等待相应初始化完成，完成后的静态状态具备规定的可见性。因此按需持有者（Holder）模式可用于懒加载单例，但它只保障实例的初始化发布，不使实例以后任意可变操作都线程安全。

~~~java
final class ClientRegistry {
    private static class Holder {
        static final ClientRegistry INSTANCE = new ClientRegistry();
    }
    static ClientRegistry instance() { return Holder.INSTANCE; }
}
~~~

第一次访问Holder.INSTANCE触发Holder初始化。若构造器连接远端并长期阻塞，所有等待者都受影响；懒加载也把首次成本转嫁到请求，不一定优于受控启动与预热。

### 6.2 失败为什么不能靠重复调用静态方法重试

静态初始化抛出某些异常时，首次使用可能看到ExceptionInInitializerError，类进入错误状态；后续同一运行时类的主动使用可能收到NoClassDefFoundError。这不一定是jar缺失，而可能是类早前初始化失败，必须找第一次cause。

异常本身属于Error时包装规则有差异，不能说所有失败一律包装。错误状态不意味着下一次调用自动重跑<clinit>。需要可重试连接的业务，应把可失败外部初始化设计成明确状态机或受控组件生命周期，不塞进不可恢复的静态初始化器。

### 6.3 互相等待如何卡住启动

线程甲初始化A并等待B，线程乙初始化B并等待A，可能形成类初始化等待环。静态代码块还可能启动新线程并join，而新线程回调该类静态方法，导致新线程等初始化、初始化线程等join。

避免在<clinit>中执行网络调用、复杂锁协议、线程启动与等待。启动依赖用可观察的显式阶段、超时和失败处理管理；加载日志与线程栈可帮助看出谁在初始化谁。类初始化同步不是业务事务，也不替外部副作用提供回滚。

## 七、卸载与ClassLoader Leak：为什么重部署后内存不回落

普通类的卸载与其定义类加载器是否可回收紧密相关，具体卸载由收集器与运行时决定。类没有实例并不等于类可卸载；Class对象、加载器及相关引用仍可能被外部持有。也不是每次GC都保证立即卸载全部合格类。

典型链：应用存活池线程的TCCL指向旧插件加载器；父级静态注册表持有旧插件监听器；定时任务捕获插件对象；ThreadLocal值是插件类实例。加载器持有其类及元数据相关结构，外部根使整批旧版本无法释放。

卸载流程需要停止接纳、取消并等待任务、注销监听器/驱动/指标、清理线程本地与TCCL、关闭加载器相关资源并去掉外部引用。URLClassLoader.close释放其打开的文件资源，不等于强制卸载所有类。关闭文件、删除引用和实际GC是不同动作。

诊断要对比重复部署后的加载器数量、类数量、元空间基线及到根路径；类不断新增但同一加载器仍业务存活，可能是动态代理/脚本类无界生成，而不一定是旧加载器泄漏。修复需分清“旧版本留住”还是“现版本无限造类”。

## 八、JMM与内存区域：存放位置不能证明并发正确

JMM规定不同线程能观察到哪些读写及顺序。Heap/Stack模型描述运行时数据职责，不能证明一个线程读取共享对象会及时看见最新字段。TLAB分配或“对象在堆上”都不提供业务同步协议。

~~~java
final class Config {
    final int limit;
    Config(int limit) { this.limit = limit; }
}
final class ConfigStore {
    private volatile Config current = new Config(10);
    Config get() { return current; }
    void replace(int limit) { current = new Config(limit); }
}
~~~

完整构造的不可变Config通过volatile引用发布，读者读取引用获得相应可见性；把current改成普通字段后仅靠“在堆中共享”不足。volatile引用也不使它指向的可变集合内部操作原子。

happens-before包括同一锁释放到后续获取、volatile写到后续读、线程启动及完成检测等关系，并可传递。它不是墙钟时间先后；sleep一秒不等于安全发布。final有构造相关保证，但this逃逸或后续可变字段仍要正确同步。容量排障与并发排障可互相影响，却不能用一种图替代另一种证明。

## 九、容器资源预算：-Xmx只是其中一项

### 9.1 预算从峰值和故障场景建立

假设容器limit为2 GiB，教学预算为堆上限1 GiB、Metaspace峰值150 MiB、Code Cache实际提交100 MiB、平台线程栈预计实际占用120 MiB、Direct/其他native峰值350 MiB、其他计费与余量304 MiB，合计2048 MiB。数字只是分类算术，不能直接复制成生产参数，尤其部分项需要用高水位实测。

线程栈reserved与实际驻留不能混加；预算也不能重复统计NMT中已归入其他类别的数据。Xms、Xmx、收集器和页提交策略影响实际峰值；把Xmx设为limit减当前非堆used，会漏掉启动、重建和突发空间。

容器可能在Java堆未到Xmx时被内核杀死。自动按容器大小计算堆的参数也不保证剩余native足够；CPU配额还会影响GC与业务竞争，内存压力和延迟需一起验证。不给所有环境统一“堆必须占75%”的魔法比例。

### 9.2 直接内存、代码缓存与元空间限制

ByteBuffer.allocateDirect在堆外分配主要内容，堆里仍有管理对象；不能只看堆dump中的wrapper浅大小判断真实占用。MaxDirectMemorySize约束相关直接缓冲分配，不是整个native进程内存硬上限，也不限制所有JNI库分配。

MaxMetaspaceSize可限制相关元空间增长，但调大只可能延后类增长故障。Code Cache受编译代码与运行特征影响，满了可能影响编译与性能，不必表现为普通Java heap space错误。限制哪一层，要对应实际增长来源而非同名“内存”都调Xmx。

native分配器碎片、库缓存或映射页也会影响RSS。需要OS证据和目标库统计；NMT与BufferPool指标是局部视角，不应硬凑成一张永远相等的总账。

## 十、OOM故障分类：先找发生在哪个资源边界

| 表现 | 优先证据 | 不能直接推导的结论 |
|---|---|---|
| Java heap space | 错误堆栈、GC前后堆占用、heap dump | 不等于一定泄漏，可能请求或存量超预算 |
| Metaspace | 类/加载器数量、元空间趋势、加载路径 | 不一定是堆小，可能动态类或旧加载器保留 |
| Direct buffer memory相关错误 | BufferPool、直接分配栈、native与RSS趋势 | 堆used低不代表进程资源健康 |
| unable to create native thread | 平台线程数、OS/cgroup限制、栈和进程资源 | 不一定Xmx不足，增堆还可能挤占native |
| StackOverflowError | 重复递归栈、调用深度与-Xss | 不应先当成Heap OOM |
| 容器OOMKill/进程被杀 | 容器终止原因、cgroup事件、内核/平台记录 | exit 137单独不能证明OOM，SIGKILL也可人为触发 |

Java抛OutOfMemoryError意味着某分配或资源请求无法满足，并不保证应用仍可安全继续。不要在全局catch(Throwable)后返回伪成功，内存紧张时连日志与恢复动作也可能失败。应按服务故障策略保留最小证据、隔离流量并安全重启，修复原因后验证。

OOMKill可能没有Java异常和堆dump，因为进程被外部强制结束。预先配置HeapDumpOnOutOfMemoryError只覆盖适用的JVM OOM路径，不能当容器被杀后必有证据的承诺；也要有可写目录、足够磁盘和安全访问控制。

## 十一、排障工作流：证据、影响和解释连起来

### 11.1 先留低成本趋势和运行环境

记录JDK构建、收集器、启动参数、容器限制、发布时间、流量与错误起点；对照堆used/committed、GC后存活量、元空间、类数量、线程数、BufferPool、RSS和容器计量。先统一时间与单位，再谈哪个区域增长。

部署后启动预热导致类与代码增长，随后稳定，可能是正常生命周期；相同负载下每次GC后基线仍阶梯上涨则需调查。但“GC后低水位上涨”也不自动证明泄漏，业务活跃集合和缓存预热可能合理扩大，应与业务量、保留策略核对。

### 11.2 Heap Dump、Dominator与到根路径

Heap dump记录对象及引用等快照。Shallow Size是对象自身占用；Retained Size是若移除该对象，按引用支配关系可随之释放的那批对象大小。支配（Dominator）表示从根到目标对象的所有路径都经过某个对象；它适合找集中持有者，但不等于直接“该对象分配了全部内存”。

先看大支配者，再查看Path to GC Roots确认是谁保留；多个独立持有者共享同一对象时，删除其中一个可能无法释放，因此浅大小、保留大小和引用路径要一起看。类直方图只有数量/大小，不能独立回答为什么活着。

例如HashMap节点很多只是症状，需继续查到缓存owner、键基数、TTL是否真的执行、值是否捕获请求上下文。修复后在相同负载/周期重测基线与功能，不以一次手工清空缓存当完成。

dump可能停顿、占磁盘，并含用户数据、密钥和令牌；采集前评估副本与容量、尽量隔离实例，文件按敏感生产数据管理。不要为了诊断把全部副本同时dump导致额外故障。

### 11.3 NMT如何使用，为什么不能等同RSS

HotSpot NMT需启动时启用summary或detail，默认不启用；不能靠jcmd在未启用的进程中随时开始完整跟踪。先建立baseline，稍后对比summary.diff，以定位HotSpot分类增长。NMT有开销，级别与开启策略需评估。

~~~text
# 启动配置片段，合并到真实应用命令中
java -XX:NativeMemoryTracking=summary -jar app.jar

# PID替换为目标进程，先确认目标支持哪些命令
jcmd PID help
jcmd PID VM.native_memory baseline
jcmd PID VM.native_memory summary.diff scale=MB
~~~

NMT主要跟踪HotSpot自身相关内存，不覆盖所有第三方native及类库分配，也不能精确代表全部驻留页或CDS映射。若RSS涨而NMT稳定，应继续查native库、映射、分配器和OS计量，而不是宣布“没有native泄漏”。

summary.diff是相对基线差值，不是峰值历史；建立基线过晚可能看不到启动阶段异常，基线期间工作负载变化也会影响解释。

### 11.4 jcmd、JFR和GC日志怎样分工

以下为HotSpot JDK21诊断命令示例，先运行目标jcmd help COMMAND核对可用参数与影响。attach需满足进程、权限和命名空间等条件；容器里工具缺失或PID不一致不是JVM必然损坏。

~~~text
jcmd PID VM.flags
jcmd PID GC.heap_info
jcmd PID GC.class_histogram
jcmd PID GC.heap_dump /secure/path/heap.hprof
jcmd PID Thread.print
jcmd PID JFR.start name=memory settings=profile duration=60s filename=/secure/path/memory.jfr
~~~

直方图与dump可能产生明显停顿/GC影响，不能当低成本每秒监控。JFR分配采样、线程、类及GC相关事件帮助还原时间关系，但采样和阈值不提供每个对象的全历史。Thread.print用于平台线程与等待关系，虚拟线程需采用目标版本支持的相应转储工具。

GC日志示例为-Xlog:gc*:file=gc.log:time,uptime,level,tags:filecount=5,filesize=20M，属于HotSpot JDK21统一日志语法，实际路径与轮转容量按环境配置。类加载/卸载可按需用-Xlog:class+load=info,class+unload=info，注意体量；不同时无差别打开所有detail。

GC日志回答何时分配/回收压力高、暂停和回收后基线怎样；heap dump回答当时谁保留对象；NMT和OS指标回答非堆/进程边界。三者相互补充，任何一个都不是万能证据。

## 十二、完整类初始化实验：读出“加载”与“初始化”的区别

保存为InitLifecycleDemo.java，在完整JDK21执行javac InitLifecycleDemo.java，再java InitLifecycleDemo。实验不依赖sleep或GC时机：

~~~java
public class InitLifecycleDemo {
    static class Target {
        static final int CONSTANT = 7;
        static int value = init();
        static int init() {
            System.out.println("Target initialized");
            return 9;
        }
    }
    static class Broken {
        static int value = fail();
        static int fail() {
            throw new IllegalStateException("teaching failure");
        }
    }
    public static void main(String[] args) throws Exception {
        System.out.println("constant=" + Target.CONSTANT);
        Class<?> type = Target.class;
        Class.forName(type.getName(), false, type.getClassLoader());
        System.out.println("loaded without initialization");
        System.out.println("value=" + Target.value);

        try {
            System.out.println(Broken.value);
            throw new AssertionError("expected initialization failure");
        } catch (ExceptionInInitializerError first) {
            System.out.println("first=" + first.getCause().getClass().getSimpleName());
        }
        try {
            System.out.println(Broken.value);
            throw new AssertionError("expected erroneous class state");
        } catch (NoClassDefFoundError later) {
            System.out.println("later=NoClassDefFoundError");
        }
    }
}
~~~

期望先输出constant=7和loaded without initialization，再输出Target initialized、value=9；Broken第一次访问包装原IllegalStateException，第二次看到NoClassDefFoundError。原因是常量内联和类字面量不触发Target初始化，value访问触发；Broken进入错误状态后不自动重试。

实验不证明任何特定GC回收时间。不要写“对象设null后System.gc，再断言弱引用一定清空”的不稳定测试，System.gc只是请求而非普遍时限保证。类卸载实验也要结合目标收集器、加载器引用及日志验证，不能用一次sleep当证据。

验证限制：本次环境只有OpenJDK21运行时，无javac与jcmd；Java示例与attach诊断命令未实际执行，期望输出是按规范推导而非运行记录。结构、触发顺序、异常路径已人工核对；真实故障采集仍需在目标环境演练。

## 十三、架构场景：堆稳定但容器反复被杀

教学场景：2 GiB容器、Xmx约1.6 GiB；业务堆GC后稳定，上传并发增加时Pod被杀，没有Java OOM日志。不能因为堆稳定就判监控误报，也不能先把堆加到1.9 GiB。

先核对终止reason与内核/cgroup事件，确认是否OOM而非人工kill；再对照上传在途、Direct Buffer、线程、native库和容器计量。上传请求可能同时保留网络缓冲、解压结果和临时文件缓存，Java堆只是其中一部分。NMT未覆盖的库分配要找对应证据。

若原因是无界上传并发，采用入口接纳、单请求大小限制、流式处理和有界缓冲；若Direct对象迟迟未释放，核查客户端/库资源生命周期。适当降低Xmx给native留空间或提高容器limit只能作为有依据的预算调整，不能替代修复泄漏或无界增长。

提高副本数也要考虑流量分摊是否均匀、热点、数据库和成本；长远应建立按每在途请求字节与固定JVM开销计算的容量模型，并演练峰值、恢复及dump时的额外空间。可靠性目标、租户隔离和数据安全与内存调优是同一个交付问题。

## 十四、完整面试回答与深入追问

**问：JVM内存有哪些区域，和容器limit有什么关系？**

“我先区分规范与HotSpot实现。Heap保存对象数组，JVM栈保存调用帧，方法区是类级运行信息的抽象；HotSpot还有元空间、编译代码缓存和各类native/直接内存。Xmx只管堆，不等于进程RSS或cgroup计量。容器预算需加入线程、类元数据、直接缓冲及库分配的峰值和余量，再用目标JDK指标、NMT与OS数据校准。”

**追问：方法里的new是否一定进堆，是否一定产生GC压力？**

“规范按对象语义描述堆，但最终实现可做逃逸分析和标量替换，消除一些分配。不能说所有不逃逸对象必定栈分配，也不能承诺优化总发生。要看内联和实际使用，并用目标负载分配证据验证。”

**问：Java有GC为什么还泄漏？**

“GC按可达性，不按业务需要回收。static缓存、监听器、线程本地、任务队列或旧类加载器可能让不再需要的数据继续强可达。我会用dump的支配树和到根路径找持有者，确认释放时机与上限，而不是只看哪个类数量最多或强制GC。”

**追问：NoClassDefFoundError是否就是jar没放进去？**

“不一定。它也可能是同一个运行时类早先初始化失败后进入错误状态。要找首次ExceptionInInitializerError及cause、类加载日志和加载器身份；不能看到后续错误就反复重启或补jar。”

**问：NMT和RSS为什么对不上？**

“NMT跟踪HotSpot分类，RSS反映驻留页，reserved/committed又是不同口径；第三方native、类库和映射等也不全被NMT覆盖。我会比较同时间趋势与差值，再结合OS及具体库证据，不硬把一份统计当总账。”

**追问：是否应把Metaspace限制取消来解决故障？**

“先看类和加载器是否合理增长。若是预期动态类量大且稳定，可重新预算；若每次重部署保留旧加载器或无限生成类，放开限制只把故障推向容器。要修生命周期、容量上限，并验证卸载和基线。”

## 十五、扫雷自测与完整参考答案

### 15.1 Heap/Stack图为什么不能替代JMM？

运行时区域说明数据职责，JMM说明线程能观察哪些读写。共享堆不保证可见性，栈中引用也不意味着对象不共享。需要volatile、锁或并发库等明确发布关系，并解释happens-before，而非靠sleep和“刷新内存”口号。

### 15.2 对象互相引用就不能回收吗？

可以回收，只要整个循环不从GC Roots可达。追踪式GC不靠简单引用数为零判断；应找外部保留链。相反只有一个static引用也足以让巨大对象图存活。

### 15.3 Eden→Survivor→Old是否每个对象必经？

不是。它是某些分代收集器的常见概括，大对象、晋升空间、年龄策略及收集器布局都会改变路径，JIT还可能消除分配。必须说明目标JDK和收集器，不把HotSpot某路径当Java标准。

### 15.4 Class.forName与Foo.class是否都初始化类？

不一样。Foo.class取得类镜像不主动初始化Foo；Class.forName(name)通常会初始化，显式false重载可抑制主动初始化。访问编译期常量也未必触发初始化；测试要用非常量静态字段区分。

### 15.5 同名Plugin为何无法强转为Plugin？

可能由不同定义加载器加载，运行时类型身份不只是类名。检查加载器与模块信息，让跨插件SPI由共同父级定义，避免把插件私有版本泄露到共同接口。仅把jar文件内容比成一样不能证明类型相同。

### 15.6 旧ClassLoader.close了，Metaspace为什么仍涨？

close可释放加载器打开的资源，不强制卸载类；存活线程TCCL、父级注册表、ThreadLocal或任务仍可保留它。需停止任务、注销及移除引用，并通过目标GC日志和加载器趋势验证；也要排查现加载器无限生成新类。

### 15.7 堆dump没有巨大对象，RSS很高，下一步是什么？

先统一dump与RSS时间及单位，核查堆committed、直接缓冲、线程、元空间、代码、native库、映射和cgroup口径。启用NMT的进程可作差分，但NMT不覆盖所有native；不能凭heap dump否定进程内存问题。

### 15.8 exit137能否直接证明OOMKill？

不能。它常表示SIGKILL相关退出，外部人工或平台终止也可能如此。需要容器终止原因、cgroup与内核记录交叉确认；若真OOMKill，JVM可能来不及产生Java异常或dump。

### 15.9 Retained Size最大是否意味着分配最频繁？

不是。它反映引用支配下可随持有者移除而释放的规模，分配频率是时间维度。一个缓存可能只负责保留别处创建的对象；应结合分配采样、到根路径和业务owner判断原因。

### 15.10 把Xmx加大为什么可能让容器更容易死？

堆上限占更多预算，使直接内存、元空间、线程和库分配峰值没有空间，外部cgroup上限仍固定。若原故障不在堆，增加Xmx可能恶化。应根据各层证据做整体预算并限制在途量，不能只调一个参数。

## 十六、复习连接与速记

对象的生命周期受“分配优化、可达关系、收集器与资源关闭”共同影响；类的生命周期受“定义加载器、初始化状态和外部引用”影响；进程生存则受全部native及容器计费边界限制。三条链连起来，才能解释堆低却被杀、热部署元空间涨、类存在却初始化失败等现象。

速记：规范不等于HotSpot布局；Xmx不等于RSS；TLAB仍在堆；标量替换不等于承诺栈分配；GC看可达不是业务需要；资源要显式关；类身份看定义加载器；加载不等于初始化；初始化失败找第一次cause；dump找持有者、NMT看局部native趋势、OS看进程边界。

## 十七、来源与验证范围

核查日期：2026-11-10。规范部分按Java SE21；HotSpot参数、G1路径和诊断命令限定JDK21，目标环境需核实支持和影响，正文已给出理解与作答所需解释。

- [JVMS21第2章](https://docs.oracle.com/javase/specs/jvms/se21/html/jvms-2.html)：抽象运行时数据区及实现边界。
- [JVMS21第5章](https://docs.oracle.com/javase/specs/jvms/se21/html/jvms-5.html)与[JLS21第12章](https://docs.oracle.com/javase/specs/jls/se21/html/jls-12.html)：加载、链接、初始化、实例和卸载语义。
- [JLS21第17章](https://docs.oracle.com/javase/specs/jls/se21/html/jls-17.html)：JMM与发布关系。
- [Java SE21引用包](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/ref/package-summary.html)、[ClassLoader](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/ClassLoader.html)与[虚拟线程指南](https://docs.oracle.com/en/java/javase/21/core/virtual-threads.html)：引用语义、加载器边界与线程实现背景。
- [JDK21 G1指南](https://docs.oracle.com/en/java/javase/21/gctuning/garbage-first-g1-garbage-collector1.html)：Region与分代路径背景。
- [JDK21 NMT](https://docs.oracle.com/en/java/javase/21/vm/native-memory-tracking.html)：启用、差分与覆盖限制。
- [JDK21 jcmd](https://docs.oracle.com/en/java/javase/21/docs/specs/man/jcmd.html)和[java命令](https://docs.oracle.com/en/java/javase/21/docs/specs/man/java.html)：诊断命令、启动与日志参数。

已原位覆盖原章Heap/Stack/Metaspace/Code/Native、容器预算、分配与逃逸优化、GC Roots与四类泄漏、类加载与委派、初始化同步、JMM区分、dump支配树/Retained Size、NMT/jcmd/JFR/GC日志/RSS及OOM分类。所有面试与自测附完整解释。Java与诊断命令未实际运行的限制已在实验章说明；没有编造GC行为、容量或生产故障结果。
