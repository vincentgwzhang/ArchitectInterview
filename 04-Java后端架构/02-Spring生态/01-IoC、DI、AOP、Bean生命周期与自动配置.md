# 01. IoC、DI、AOP、Bean 生命周期与自动配置

## IoC 与 DI

IoC Container 负责创建对象和管理依赖，DI 是实现方式。优先 Constructor Injection：依赖明确、对象可保持 Immutable、测试容易，也能更早发现循环依赖。

避免 Field Injection 和从 ApplicationContext 主动查找 Bean，这会隐藏依赖并产生 Service Locator。

## Bean 生命周期

概念流程：实例化 → 属性/依赖注入 → BeanPostProcessor Before → 初始化回调 → After → 可用 → 销毁回调。`@PostConstruct` 不适合执行不可控长时间网络任务，否则拖慢启动。

Scope 包括 Singleton、Prototype、Request、Session。Spring Singleton 不等于线程安全；其共享可变字段仍需同步或移除。

## AOP

AOP 适合 Logging、Transaction、Security、Metric 等 Cross-cutting Concern。Spring Proxy 通常只拦截通过 Proxy 的调用，自调用可能绕过 Advice；Final/Private Method 与 Proxy 类型也会影响行为。

不要用 AOP 隐藏核心业务流程。隐式 Side Effect 越多，调试越困难。

## 自动配置

Spring Boot 根据 Classpath、Property 和 Conditional 创建默认 Bean。使用 Actuator Condition Report 理解为什么配置生效。自定义 Starter 应提供明确 Property、默认值、条件和 Metadata。

## 常见问题

- 循环依赖揭示 Boundary 不清。
- 多个 Bean 候选导致注入歧义。
- Proxy 导致 Annotation 未生效。
- Bean 初始化访问外部系统，启动脆弱。

## 总结

Spring 的价值是显式依赖与生命周期治理。Constructor DI、克制 AOP 和可解释自动配置能保持系统可测试、可诊断。
