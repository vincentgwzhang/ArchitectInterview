# 01. IoC、DI、AOP、Bean生命周期与自动配置

## 学习目标、版本与贯穿案例

Spring的价值不仅是少写new，而是把对象依赖、创建/销毁、横切行为和默认配置放到可治理的边界。面试要能从业务代码解释容器做了什么、没做什么，以及“注解存在但行为没有发生”如何定位。

本章以Java17及以上、Spring Framework 6.2、Spring Boot 3.5为解释基线，示例不要求使用最新大版本。Boot3使用jakarta.annotation的生命周期注解，与旧教程javax.annotation不能混用；AOP、生命周期和自动配置内部细节也不可不加版本地泛化。所有通知、订单、指标均为教学案例，不是用户项目。

贯穿案例是订单应用：OrderService依赖通知接口Notifier，默认日志通知可被企业实现替换；容器管理资源，AOP统一观察耗时，业务失败不能因日志切面被吞掉。涉及事务时只解释代理边界，不宣称内存示例已实现真实数据库事务。

## 一、IoC与DI：控制什么，依赖怎样明确

### 1.1 IoC不是接口，也不是注解

IoC（Inversion of Control，控制反转）表示对象创建、协作装配或流程控制由框架/外部机制接管；DI（Dependency Injection，依赖注入）是其中一种方式，调用者不自行new或查找依赖，而在构造器、工厂参数或属性中声明需要什么。

例如OrderService只接收Notifier，不在业务方法里判断环境再new某个短信客户端。这样测试可注入内存替身，生产可注入企业适配器，依赖选择集中在组合根（Composition Root，组装对象图的边界）。DI不要求一定用Spring，手动构造也能实现。

它不能自动消除业务耦合。若接口暴露供应商专有请求结构，换实现仍会影响调用者；若接口粒度过大，只是把实现依赖换成同样不稳定的抽象。架构师要先确定变化边界，再用容器装配，而不是为每个类机械造接口。

### 1.2 Constructor、Setter与Field Injection

构造器注入适合必要依赖：对象创建时必须具备所有协作对象，可用final字段固定引用，普通单元测试直接new即可。final引用不保证被引用对象不可变，也不让整个Service自动线程安全。

Setter适合真正可选或需要后续配置的依赖，但对象可能出现未配置阶段，要定义默认行为和校验。字段注入省代码，却隐藏构造前提，实例化后依赖才被反射填入，手动new会留下null，测试常被迫启动容器或使用反射。它不是语言上禁止，但通常降低可读性与可测试性。

构造器参数很多也不是改字段注入的理由，可能表示职责过多或依赖层次混乱。应检查是否能拆领域能力，而不是用一个大Context对象把所有依赖再藏起来。

### 1.3 Service Locator为何难维护

在业务代码中调用applicationContext.getBean按名称取依赖，是Service Locator风格：方法签名看不出依赖，错误可延后到运行时，核心逻辑与容器耦合。对稳定业务协作优先显式注入。

插件选择、按需多实例或框架基础设施可能确需运行时定位。可把定位限制在适配层，使用明确接口、ObjectProvider或策略注册表，并写清缺失/多候选行为。不是所有getBean都错误，问题是它是否把业务依赖和生命周期隐藏到无法推理。

## 二、BeanDefinition到对象图：容器的工作单位是什么

### 2.1 定义、实例和容器边界

Bean是由容器管理生命周期的对象；BeanDefinition是创建配方，描述类型、依赖、作用域、初始化和销毁等元数据。一个类可以被注册成多个定义，一个定义也会因Scope产生多个实例，所以“Bean就是类”不准确。

@Component及派生注解通过扫描发现候选；@Bean明确用工厂方法创建对象，适合第三方类或需要精确配置的对象。ApplicationContext在BeanFactory能力上整合事件、资源、环境等服务，但容器本身不该成为领域对象的万能依赖。

父子Context可能各有定义和实例，类型与名称查找还受层级可见性规则影响。Spring singleton是每容器、每Bean定义的共享实例，不是整个JVM或所有微服务节点全局唯一。

### 2.2 BeanFactoryPostProcessor与BeanPostProcessor

BeanFactoryPostProcessor在实例化普通Bean前修改定义等配置元数据，例如调整属性配方；BeanPostProcessor围绕对象创建/初始化处理实例，可实现注入、生命周期注解处理或代理包装。两者操作对象不同，不能把“后置处理器”全部当成new完以后执行的回调。

自定义扩展应避免过早getBean导致普通对象提前实例化，绕开尚未注册的处理器。基础设施Bean的依赖要克制；某个Bean“not eligible for all BeanPostProcessors”等提示需要追创建时机，而非只加一个注解。

普通业务多数不需写后置处理器。能用显式@Bean、配置属性或切面表达时更易理解；使用扩展点需解释运行阶段、顺序与对其他Bean的影响，不把容器当业务流程引擎。

### 2.3 @Configuration方法调用的特殊边界

完整@Configuration默认可对@Bean方法调用做增强，使同配置类中直接调用@Bean方法时获取受管理语义。proxyBeanMethods=false取消这种方法拦截，直接Java调用可能创建额外对象；普通@Component中的@Bean方法也不应按完整配置增强理解。

更清晰方式是把依赖写成@Bean方法参数，由容器解析。自动配置通常采用不依赖配置类自调用的方式，避免为配置类子类代理付出不必要复杂度。配置类增强与Spring AOP业务代理是不同用途，不要混为“所有方法有代理”。

## 三、依赖解析：歧义、可选项、集合与循环

### 3.1 多候选不是随便取一个

按类型有多个Notifier时，注入单个依赖会需要消歧。@Primary表达默认优先候选，@Qualifier缩小语义候选范围；不能假设扫描顺序或类名恰巧决定稳定结果。参数名回退等规则有版本/编译参数边界，显式表达选择更可靠。

例如客户通知与审计通知都是Notifier，但语义不同，可用限定符或更明确接口，不能只为了消启动错误随意把任一个设Primary。注入List<Notifier>或Map<String,Notifier>可构建策略注册表，但需要定义顺序、键稳定性与空集合行为。

### 3.2 Optional与Provider怎样体现真实语义

Optional<Notifier>适合缺失时有明确定义的可选能力，不适合把必须的付款网关隐藏为可选。ObjectProvider可延迟解析、按需取得实例或在缺失时选择行为；它把获取时机变成运行时决策，也把部分错误延后，需要测试。

构造器注入prototype对象只在singleton创建时解析一次，不会每次业务调用自动new。确需每次获取新prototype，可注入Provider并在任务边界取得，且自行处理需要关闭的资源。Provider解决生命周期获取问题，不能当规避所有循环依赖的魔法。

### 3.3 循环依赖为什么优先改边界

A构造时必须有B，B构造时必须有A，容器无法先造出完整任一方，构造器循环通常直接失败。某些setter/字段循环在特定容器配置下可能借提前引用解决，但容易暴露未初始化对象和代理一致性问题；Boot是否允许循环也需看配置，不依赖历史默认。

真实原因可能是双向业务职责混合：订单服务调用通知，通知又同步回调订单服务。可提取查询接口、把协调移到第三个应用服务或改成符合业务语义的事件边界。异步事件同时改变事务、失败和一致性，不能只为启动成功机械“解环”。

@Lazy或Provider有时可延迟获取，但循环仍存在，只是首次调用时才显现；初始化期间再访问还可能失败。面试应说明何时延迟合理、失败怎样暴露，而不是回答“加Lazy就行”。

## 四、Bean生命周期：概念顺序与真实扩展点

### 4.1 从创建到可用的典型流程

常见流程可理解为：解析定义和依赖 → 实例化 → 属性/依赖填充 → 相关Aware回调 → 初始化前处理 → 初始化回调 → 初始化后处理/可能包装代理 → 对外取得可用Bean。销毁发生在容器关闭或相应Scope结束的受管理边界。

这是概念模型，不是所有扩展点无条件只执行一次的固定列表：实例化感知处理器可在更早阶段介入，部分处理器可返回替代对象；@PostConstruct本身由特定后置处理器在初始化前处理阶段调用。不能把“所有before回调全部结束后才执行PostConstruct”作为通用保证。

普通初始化回调的常见顺序是@PostConstruct、InitializingBean.afterPropertiesSet、自定义initMethod；同方法重复配置的处理与去重受机制约束，不应故意把同一副作用配置三遍。优先选择一种适合的回调方式，并测试真实Bean行为。

### 4.2 初始化不等于完整业务代理已生效

初始化回调作用在目标对象生命周期，通常不能依赖该对象最终对外AOP代理拦截自身调用。@PostConstruct里this.saveTransactional()即使方法有@Transactional，也不能据此认定事务已开启。

初始化阶段适合验证本地必填配置、建立轻量内部状态；不可控网络、全量数据扫描、无限重试或等待其他服务会延长整个依赖图启动。若必须预热，要有超时、失败策略与可观测进度，并明确应用是否可接流。

Boot ApplicationRunner/CommandLineRunner可用于容器刷新后的启动工作，但它们也会影响启动与ready时机，不是天然后台。Lifecycle/SmartLifecycle适合管理可启动/停止的长期组件，需定义phase、停止协议与超时；这些机制不能承诺在强制杀进程时仍有回调。

### 4.3 销毁顺序与清理责任

受管理singleton关闭时可执行@PreDestroy、DisposableBean.destroy、自定义destroyMethod等销毁逻辑，依赖关系影响销毁顺序。正常关闭不是业务事务，回调抛错或超时可能留下未完成工作，需要设计幂等关闭与日志。

prototype通常只负责创建、装配与初始化，不自动追踪所有实例以执行销毁；谁取得、谁管理后续资源责任。应用手动new的对象也不因类上有@PreDestroy就获得容器销毁待遇。

关闭网络客户端或线程池之前，应停止新接纳、等待/取消在途并按业务保存未完成状态。只写close而不控制顺序，可能让消费者仍在用已关闭依赖。容器/OS强制终止和崩溃不保证钩子运行，关键数据必须在正常处理路径持久化。

## 五、Scope与并发：共享实例不等于安全实例

| Scope | 大致实例边界 | 常见陷阱 |
|---|---|---|
| singleton | 每容器每定义共享 | 请求特有字段被多线程覆盖 |
| prototype | 每次容器获取创建 | 注入singleton后并不会每调用刷新；销毁需调用者管 |
| request | Web请求范围 | 请求结束后异步任务继续使用，Scope可能已失效 |
| session | HTTP会话范围 | 同会话多个请求可能并发；不是天然线程隔离 |
| application | ServletContext范围 | 不等同任意Context内singleton的全部语义 |
| websocket | WebSocket会话范围 | 连接结束后的资源与订阅清理 |

把tenantId、当前用户或订单临时变量写进singleton字段，会串请求甚至串租户；优先方法局部值、显式参数和不可变依赖。构造器注入只固定依赖引用，不保证某个Map、客户端或缓存本身的并发行为。

短Scope注入长Scope可通过Scoped Proxy或Provider在使用时解析当前目标；这需要活跃作用域上下文。把request代理捕获到后台线程，并不自动延长HTTP请求或传播授权身份。异步场景应复制必要不可变数据并明确生命周期，而不是抓住整个请求对象。

request与session属于Web作用域，普通非Web Context可能不支持。测试若用简化容器，要明确没有模拟真实HTTP Scope，不把单次getBean成功当成跨请求隔离已验证。

## 六、AOP：代理边界、匹配规则和副作用

### 6.1 Aspect、Pointcut、Advice、Join Point

AOP（Aspect-Oriented Programming，面向切面）把多个模块共享的横切关注点集中表达。Aspect是切面模块；Pointcut选择哪些连接点；Advice定义连接点周围执行的行为；Spring代理式AOP主要拦截方法执行，不是任意字段读写和构造器事件。

Before在调用前，After Returning仅正常返回，After Throwing匹配异常，After类似最终观察，Around可包围调用并决定是否/何时proceed。Around若忘proceed，目标业务就不执行；若调用两次，可能重复副作用。切面不是注解本身在执行，仍有处理器、匹配和代理调用链。

日志、事务、安全、指标适合集中治理，但核心审批、付款顺序和补偿决策应显式出现在应用流程。切面过多时同一方法发生什么难以发现，必须控制范围、文档和测试。

### 6.2 JDK代理与类代理：final的结论要带条件

JDK动态代理通过接口实现调用转发，消费者应按暴露接口使用，不可随意强转为目标实现类。类代理（Spring中的CGLIB机制）通过生成目标子类实现拦截，final类无法被继承，final方法不能被覆盖，private方法也不能通过这种覆盖被拦截。

不能把“final方法永远不能被任何Spring AOP拦截”作为无条件结论：JDK接口代理接收接口调用后转发到目标，目标实现方法是否final与子类覆盖是不同问题。另一方面，不在接口暴露范围内的方法也不能借JDK代理随意调用。

框架通常依据接口和配置决定代理方式，Boot及显式设置可能改变选择，应该检查运行时代理而非只背默认。即使类代理可覆盖某些非public方法，具体@Transactional或方法安全功能的可见性约束仍按目标版本与机制验证，不把代理能力等同每个注解的全部规则。

### 6.3 自调用为什么绕过

外部调用进入代理，再经过拦截器到目标；目标内部this.other()直接调用自身，没有回到代理，相关Advice不会因此再次触发。把代理改为CGLIB不自动解决自调用，因为目标内部调用边界仍未经过外部代理链。

典型例：public submit调用同类@Transactional persist，事务未按预期生效；修复可以把persist移到另一个明确事务协作者，由注入的代理调用，或使用显式事务API。切面织入（例如AspectJ方式）是不同模型，有构建/运行配置代价，不是“换个注解”即可。

自注入或AopContext取当前代理会把业务耦合到代理配置，可能涉及循环/线程上下文并降低测试性，作为特殊选择要说明代价。new Service绕过容器获得的是普通目标对象，注解不会自动注入拦截行为。

### 6.4 顺序、异常与异步边界

多个切面顺序会改变行为：重试包事务表示每次尝试可能新事务，事务包重试可能在一个已失败事务中继续。安全校验应在副作用之前，指标需说明包含排队、重试还是单次方法耗时。用@Order等明确相对顺序后仍要以集成测试验证，不靠扫描偶然顺序。

Around应在finally记录结束和耗时，异常按原契约传播；捕获Exception后返回null会掩盖失败、破坏事务判断和监控。日志避免写密码、令牌和完整请求体。指标Tag若把每个用户ID当维度，会形成高基数成本与内存问题。

若方法返回CompletableFuture或Publisher，普通Around计时到方法返回只测到提交/组装，不是异步业务完成。要通过返回类型对应的完成/终止信号记录，区分成功、异常和取消；也要避免装饰时重复订阅或改变线程和上下文。

## 七、自动配置：默认装配是一套有条件的规则

### 7.1 Component Scan与Auto-configuration不同

组件扫描按包发现应用组件；自动配置按候选清单导入配置类，再依类路径、配置属性和已有Bean等条件提供默认能力。SpringBootApplication把常见入口能力组合起来，但不会无条件扫描所有依赖jar的所有包。

Boot3.5的自动配置类使用@AutoConfiguration，通过META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports列出候选，每行一个全限定类名。不要把Boot2时代某些spring.factories注册教程直接当Boot3.5自动配置注册方式，也不要笼统说spring.factories在所有用途消失。

自动配置应放独立包且通过导入发现，不依赖宽泛组件扫描；不要让库擅自扫描消费者整个应用。依赖传递是Starter的职责之一，默认Bean及条件是Auto-configuration的职责，两者可以分模块发布，也可在简单库内组织。

### 7.2 条件判断的输入与时机

ConditionalOnClass检查相关类可用，ConditionalOnProperty检查配置值，ConditionalOnMissingBean在条件评估可见定义范围内决定是否补默认Bean。条件匹配不是随运行流量每次重算，改配置也不等于已存在对象自动热切换。

MissingBean与配置处理顺序有关；自动配置应遵守相关顺序机制，声明足够明确的返回类型，让类型条件能判断。before/after主要安排配置定义处理，不保证任意Bean按该顺序实例化；真正初始化依赖应表达在依赖图，必要时用合适的DependsOn等机制。

类路径条件包围的方法若在签名中直接引用不存在的可选类，类加载可能在预期条件生效前遇到问题。可隔离到受条件保护的嵌套配置，并使用适当的name形式等方式，按Boot文档验证。不能只加一个OnClass就认为任意引用位置安全。

### 7.3 Back-off不等于同名覆盖

自动配置提供默认实现，应用显式声明同类型Bean时默认配置退让（Back-off），是可预期的扩展点。不要混成“允许同名Bean定义后者覆盖前者”；同名冲突和缺失条件判断是两回事，覆盖设置还可能受Boot配置限制。

例如企业提供Notifier实现，默认LoggingNotifier不注册；若同时给两个企业实现，应让应用明确Qualifier/Primary或设计策略集合，而不是自动配置偷偷选择最后一个。扩展契约越清楚，升级和排障成本越低。

## 八、一个完整的最小自动配置设计

以下分文件片段按Boot3.5、Framework6.2、Java17+组织，不是可直接复制到一个Java文件的完整工程。包为demo.notify，需Boot自动配置、配置属性相关依赖；验证使用spring-boot-test及测试依赖。省略构建文件不代表依赖自动存在。

### 8.1 契约与默认实现

~~~java
// Notifier.java
package demo.notify;
public interface Notifier {
    void send(String message);
}
~~~

~~~java
// LoggingNotifier.java
package demo.notify;
public final class LoggingNotifier implements Notifier {
    private final String prefix;
    public LoggingNotifier(String prefix) { this.prefix = prefix; }
    @Override public void send(String message) {
        System.out.println(prefix + ": notification accepted");
    }
}
~~~

示例只输出受控固定内容，不记录业务message，避免把敏感通知正文写日志。它不是可靠投递系统，也不保证远端收到；send返回语义应在真实适配器契约中定义。final类没问题，因为本例未要求对它使用类代理；若将来增加AOP，应重新检查代理类型。

### 8.2 类型化属性与条件配置

~~~java
// NotifyProperties.java
package demo.notify;
import org.springframework.boot.context.properties.ConfigurationProperties;

@ConfigurationProperties("demo.notify")
public class NotifyProperties {
    private String prefix = "demo";
    public String getPrefix() { return prefix; }
    public void setPrefix(String prefix) { this.prefix = prefix; }
}
~~~

~~~java
// NotifyAutoConfiguration.java
package demo.notify;
import org.springframework.boot.autoconfigure.AutoConfiguration;
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.context.annotation.Bean;

@AutoConfiguration
@EnableConfigurationProperties(NotifyProperties.class)
@ConditionalOnProperty(prefix = "demo.notify", name = "enabled",
        havingValue = "true", matchIfMissing = true)
public class NotifyAutoConfiguration {
    @Bean
    @ConditionalOnMissingBean(Notifier.class)
    Notifier notifier(NotifyProperties properties) {
        if (properties.getPrefix() == null || properties.getPrefix().isBlank()) {
            throw new IllegalArgumentException("demo.notify.prefix must not be blank");
        }
        return new LoggingNotifier(properties.getPrefix());
    }
}
~~~

消费者可用demo.notify.enabled=false关闭这一默认能力；这意味着依赖必需Notifier的应用仍需自定义实现，否则启动应失败，不能“关闭就默认业务不需要”。prefix在默认Bean创建时校验，不是所有可能场景的全局配置验证；若要求不论是否back-off都校验，可用合适的配置属性校验并测试。

@EnableConfigurationProperties注册属性绑定。属性类与使用它的Bean分开，便于文档和测试；复杂属性应有单位、默认值、验证、敏感标记和升级兼容策略，不能用几十个随意的@Value字符串散落业务中。

### 8.3 发现清单与元数据

在jar资源路径META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports中写一行：

~~~text
demo.notify.NotifyAutoConfiguration
~~~

自动配置候选清单负责“被发现”；spring-configuration-metadata.json负责属性工具提示等元数据，可用配置处理器生成，二者不是同一个文件。元数据不负责创建Bean，也不能代替运行时属性验证。

Starter应声明明确依赖、支持版本与配置说明，避免拉入不必要Web/数据库组件。默认值要保守，例如不自动启用外部付款或强制开放管理端口；库提供的隐式行为会影响所有消费方，升级应有兼容与退让测试。

### 8.4 ApplicationContextRunner验证什么

测试片段需导入Boot测试工具、JUnit5及AssertJ，并按上述文件包结构编译：

~~~java
package demo.notify;
import org.junit.jupiter.api.Test;
import org.springframework.boot.autoconfigure.AutoConfigurations;
import org.springframework.boot.test.context.runner.ApplicationContextRunner;
import static org.assertj.core.api.Assertions.assertThat;

class NotifyAutoConfigurationTest {
    private final ApplicationContextRunner runner =
        new ApplicationContextRunner().withConfiguration(
            AutoConfigurations.of(NotifyAutoConfiguration.class));

    @Test void providesDefault() {
        runner.run(context ->
            assertThat(context).hasSingleBean(Notifier.class));
    }
    @Test void backsOff() {
        Notifier custom = message -> {};
        runner.withBean("customNotifier", Notifier.class, () -> custom)
            .run(context -> {
                assertThat(context).hasSingleBean(Notifier.class);
                assertThat(context.getBean(Notifier.class)).isSameAs(custom);
            });
    }
    @Test void disabledMeansNoDefault() {
        runner.withPropertyValues("demo.notify.enabled=false")
            .run(context ->
                assertThat(context).doesNotHaveBean(Notifier.class));
    }
    @Test void rejectsBlankPrefix() {
        runner.withPropertyValues("demo.notify.prefix=")
            .run(context -> assertThat(context).hasFailed());
    }
}
~~~

这些测试检验默认存在、显式替换、关闭和错误配置四条边界，不能只测“容器启动成功”。因为直接指定AutoConfigurations，它不证明jar候选清单打包和消费者真实发现已正确；还应有使用实际Starter的独立消费应用烟测，检查资源清单、类路径缺失、升级和管理端暴露。

本次环境无javac/Maven和Spring依赖，以上Java及测试未实际编译运行，已人工检查导入、Bean类型、条件与预期。不能把测试代码存在宣称为测试通过，也不把日志默认实现说成真实消息投递。

## 九、最小代理实验：理解自调用而不是只背结论

下面单文件依赖spring-aop及spring-core等匹配版本依赖，演示ProxyFactory/JDK接口代理，不需要完整Boot应用。public类放ProxyBoundaryDemo.java，JDK17+。它验证调用路径，不验证数据库事务或注解扫描。

~~~java
import java.util.concurrent.atomic.AtomicInteger;
import org.aopalliance.intercept.MethodInterceptor;
import org.springframework.aop.framework.ProxyFactory;

public class ProxyBoundaryDemo {
    public interface Work {
        void outer();
        void inner();
    }
    static class Target implements Work {
        public void outer() { inner(); }
        public void inner() { System.out.println("business"); }
    }
    public static void main(String[] args) {
        AtomicInteger intercepted = new AtomicInteger();
        ProxyFactory factory = new ProxyFactory();
        factory.setTarget(new Target());
        factory.setInterfaces(Work.class);
        factory.addAdvice((MethodInterceptor) invocation -> {
            intercepted.incrementAndGet();
            return invocation.proceed();
        });
        Work proxy = (Work) factory.getProxy();
        proxy.outer();
        if (intercepted.get() != 1) throw new AssertionError(intercepted.get());
        proxy.inner();
        if (intercepted.get() != 2) throw new AssertionError(intercepted.get());
        System.out.println("intercepted=2");
    }
}
~~~

outer经代理计一次，目标内部inner直接调用不经过代理；随后从外部调用proxy.inner再计一次。业务输出两次，拦截总数2，不是3。此实验未运行，结果是按代理语义推导；真实@Transactional还需启用基础设施、真实事务管理器并用数据库回滚结果验收。

类上/方法上写注解只是元数据；代理工厂、切面识别、匹配条件和调用边界缺一不可。排查不能只截图注解证明“Spring坏了”。

## 十、启动、代理与条件装配的排障流程

### 10.1 依赖缺失或多候选

从第一条根因异常与注入点开始，确认Bean定义是否被扫描/导入、条件是否匹配、包/模块是否正确。检查缺失类型是否实际由代理暴露，或@Bean返回类型过宽使类型推断不充分。多候选先查语义，不能直接开Bean覆盖让启动蒙混过去。

循环异常画出最小依赖环，区分业务结构、配置类自调用与基础设施过早创建；构造器循环应优先改职责。测试替身应在配置边界替换，而不是让生产代码主动查一个特殊“testBean”。

### 10.2 代理行为未生效

依次确认对象是否由容器管理、调用者拿到的是代理还是手动new目标、切面/事务/安全基础设施是否启用、Pointcut与注解是否匹配、方法是否对当前代理可拦截、是否自调用。最后再看异常传播、顺序与具体功能限制。

初始化中调用、异步回调换线程和直接对象引用都可能绕开预期边界。集成测试应验证实际业务效果，如失败后的数据库状态、安全拒绝时无副作用、指标成功/失败分类，而非只断言对象类名包含Proxy。

### 10.3 自动配置为何生效或未生效

Boot可通过debug启动日志查看条件评估信息；Actuator的conditions端点展示条件匹配原因，beans和configprops可辅助核对实例与属性。它们有依赖、访问许可及HTTP/JMX暴露条件，不是只装Starter就都向外开放。

先看候选是否发现，再看每条正/负条件、实际属性来源、用户定义的Bean与配置顺序。某个类路径条件未满足时，不要只问“依赖pom写了没有”，应查运行时最终jar和版本冲突。配置文件内容也不等于生效值，环境变量/命令行等可能改变属性。

conditions、beans、env/configprops含内部结构或配置相关信息，需认证授权、网络边界与脱敏；不要为了排查把全部Actuator端点公开互联网。Condition Report证明的是装配决策，不证明外部连接、业务授权或数据正确性。

### 10.4 启动脆弱怎样演进

若@PostConstruct全量拉远端数据导致每次重启失败，先确定无该数据是否允许服务：必需则有界重试、明确失败和可观测启动；可选则允许降级并把状态放进readiness/业务响应。不能静默吞错又宣布ready。

将预热移到后台会改变接流时数据可用性；拆分初始化任务还要管理取消、重复启动、并发刷新和停机顺序。Spring提供生命周期工具，最终责任仍在应用契约。架构价值是降低启动级联故障，同时保留真实健康信号。

## 十一、Solution Architect视角：哪些机制带来收益，何时过度

构造器DI让依赖与替换明确，适合分层和模块演进；但为极小内部实现增加过多抽象，会增加导航成本。把供应商依赖隔离在适配器有意义，给每个无变化的数据类造工厂通常没有同等收益。

克制AOP可统一安全与观测，减少重复；过宽Pointcut、过多隐式重试、吞异常与异步计时误差会破坏可靠性。核心业务决策保持显式，把横切契约变成可测试标准，并对性能和高基数指标做预算。

自动配置降低多团队接入成本，但默认值、依赖冲突和升级会扩大影响面。可替换默认、关闭开关、类型化验证、条件报告和消费方测试构成治理基础。Starter不是把公司所有平台依赖一次拉进每个服务的理由。

AOT/Native Image等部署方式会对动态代理、反射与运行期注册有额外要求，需按目标版本工具链核实并测试；本章不把JVM运行通过直接当原生镜像也通过。选型还要考虑团队调试能力与发布成本，而非只比较启动毫秒数。

## 十二、完整面试回答与深入追问

**问：IoC、DI和Spring容器有什么关系？**

“IoC是控制反转的思想，DI是让对象声明协作者、由外部组装的一种方式。Spring容器按BeanDefinition创建、注入并管理对象生命周期。我倾向构造器注入必需依赖，让对象构造即有完整前提，测试可直接new替身；但final引用不代表整个对象不可变或线程安全。动态选择会限制在明确适配边界，不让领域代码到处getBean。”

**追问：构造器循环依赖怎么修？**

“先画最小依赖环，判断是否双向职责混在一起，提取查询端口或独立协调者。Lazy/Provider可延迟特定依赖，但会延后错误且不消除结构；异步事件还改变事务和失败语义，不为启动成功随意替换。”

**问：Bean生命周期和AOP有什么容易踩坑？**

“普通流程是构造、注入、初始化相关处理与回调、初始化后处理，再对外使用可能包装的代理。PostConstruct也由后置处理器参与，不应背成所有before都结束后才调用。初始化中不能依赖本对象最终代理，自调用事务方法可能没生效。销毁还受Scope和正常关闭边界限制，prototype和手动new资源要另管。”

**追问：CGLIB能解决this调用事务方法吗？**

“不能自动解决。外部代理转目标后，目标this调用仍没回到拦截链。我更倾向明确事务协作者或显式事务边界；AspectJ织入是另一种模型，需付配置和验证成本。”

**问：自动配置和Bean覆盖有什么区别？**

“自动配置根据类路径、属性和已有定义提供默认能力，MissingBean使默认退让；同名覆盖则是替换注册定义，两者不同。Boot3.5自动配置候选通过AutoConfiguration.imports发现，条件顺序也不等于实例创建顺序。我会测试默认、替换、关闭和错误配置，并用条件报告定位。”

**追问：一个计时Around是否能测Future任务总耗时？**

“普通finally只测到方法返回，可能只是提交任务。要在Future完成或Publisher终止时记录成功、异常、取消，同时避免重复订阅和高基数标签。指标边界要写清，不把异步提交时间当业务延迟。”

## 十三、扫雷自测与完整答案

### 13.1 构造器注入是否让Service线程安全？

不自动。它明确必要依赖并可固定引用，依赖对象、缓存和后续可变字段仍可能竞争。请求特有状态用局部值和参数，真正共享状态另定义同步与一致性，不依赖singleton或final口号。

### 13.2 singleton注入prototype，是否每次调用都有新对象？

不是，依赖在singleton装配时通常只解析一次。每次业务获取新对象需Provider或其他明确机制，并管理prototype后续资源销毁。Scope规定容器获取实例的边界，不改写任意Java方法调用。

### 13.3 @PostConstruct中调用自己的@Transactional方法安全吗？

不能据此保证事务Advice执行。初始化阶段目标与最终代理边界不同，this自调用本来也可能绕过代理。需要把工作放到明确事务协作者并验证真实事务效果，同时限定外部初始化时长与失败策略。

### 13.4 final方法是否一律不能使用Spring AOP？

需区分代理方式。类代理无法覆盖final方法；JDK接口代理拦截接口调用并转发到目标，与目标实现是否final不是同一约束。仍需方法暴露、Pointcut与具体功能规则匹配，不能凭一个final关键字作跨机制结论。

### 13.5 @Primary能否代替所有@Qualifier？

不能。Primary表达默认单候选优先，Qualifier表达特定语义选择或筛选。业务需要“客户通知”而默认是“审计通知”时，随意Primary虽可能消歧却选错协作者；集合注入也有自己的候选语义。

### 13.6 为什么MissingBean正确，默认仍可能不按预期退让？

检查配置处理顺序、条件可见定义范围、类型返回信息与容器层级，以及消费方是否真的注册目标Bean。条件是装配期决策，不是运行时实时查询；查Condition Report和实际定义，不盲开覆盖。

### 13.7 @PreDestroy能否保证宕机前一定把内存任务发完？

不能。强制kill或崩溃不保证回调，正常关闭也可能超时或依赖已失效。需要正常处理路径持久化、停止接纳、在途协调与恢复协议，不能把销毁回调当可靠消息存储。

### 13.8 ApplicationContextRunner测试通过是否证明Starter交付无误？

只证明该受控Context中指定配置的行为。直接导入配置绕过jar候选发现，未覆盖真实依赖组合、扫描、资源打包、WebScope和生产管理端安全。需要实际消费应用烟测及必要集成验证。

### 13.9 为什么不在业务里手动new一个注解Service？

可以构造普通对象用于明确纯逻辑测试，但它不会自动得到容器注入、生命周期或AOP。若业务依赖事务/安全等代理行为，生产调用必须经过相应管理边界，测试也需分清纯逻辑与基础设施验收。

### 13.10 自动配置把Notifier关掉后，OrderService启动失败是否一定是bug？

不一定。如果通知是应用必需依赖，关闭默认而未提供替代就应暴露缺失。若能力真可选，应用必须显式定义缺失行为与业务契约。自动配置开关不应静默改变必要业务保证。

## 十四、复习连接与速记

DI描述对象需要谁，Scope决定实例边界，生命周期规定何时初始化和释放，AOP规定哪些调用经过横切行为，自动配置决定什么条件下提供默认实现。把五者连接起来，才能解释“对象存在但代理没生效”“定义有了但依赖选错”“关闭配置后业务不能启动”。

速记：IoC思想、DI装配；构造必需依赖；定义不是实例；singleton不是线程安全；prototype注入不自动刷新；初始化不依赖自身代理；this调用绕过；final看代理方式；MissingBean是退让不是覆盖；imports发现、metadata提示；条件报告解释装配；真实效果需集成验收。

## 十五、来源与验证记录

核查日期：2026-11-11。Framework内容限定6.2、Boot限定3.5；已核查固定版本文档，避免无版本URL跳到7.x后混用。正文足以理解和作答，来源仅供核验。

- [Framework6.2依赖注入](https://docs.spring.io/spring-framework/reference/6.2/core/beans/dependencies/factory-collaborators.html)、[容器扩展点](https://docs.spring.io/spring-framework/reference/6.2/core/beans/factory-extension.html)与[Java配置](https://docs.spring.io/spring-framework/reference/6.2/core/beans/java/basic-concepts.html)。
- [Framework6.2生命周期](https://docs.spring.io/spring-framework/reference/6.2/core/beans/factory-nature.html)、[Scope](https://docs.spring.io/spring-framework/reference/6.2/core/beans/factory-scopes.html)与[AOP代理](https://docs.spring.io/spring-framework/reference/6.2/core/aop/proxying.html)。
- [Boot3.5创建自动配置](https://docs.spring.io/spring-boot/3.5/reference/features/developing-auto-configuration.html)：候选、条件、属性与测试机制。
- [Boot3.5 Actuator端点](https://docs.spring.io/spring-boot/3.5/reference/actuator/endpoints.html)：conditions、beans及配置观测与暴露边界。

已原位覆盖原IoC/DI、构造器/字段/Service Locator、完整生命周期与PostConstruct、四类原Scope与共享状态、AOP横切/自调用/final/private、自动配置三类条件/条件报告/Starter属性默认与元数据、循环/歧义/代理/启动脆弱问题。新增完整实验与自测均有解释，未保留原提纲再追加详解。

验证范围：逐节自足性、原主题、代码与版本边界人工核对，Markdown标题/围栏/来源链接检查。当前只有Java运行时，无javac/Maven和Spring依赖，Java、代理实验及ContextRunner测试未编译运行；期望结果不是实测通过，真实事务、WebScope和Starter打包需按正文计划验证。
