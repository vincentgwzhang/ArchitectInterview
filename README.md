# Architect 面试准备大纲

> 目标岗位：Solution Architect / Software Architect / Full-Stack Architect  
> 技术范围：React、AI 全栈、Java Backend、Microservices、CI/CD、AWS

## 使用说明

- 一级标题（章）：一个完整的能力领域。
- 二级标题（节）：该领域中的核心主题。
- 三级标题（点）：需要掌握、能够解释并可结合项目经验回答的知识点。
- 每个知识点建议按照“定义 → 原理 → 方案 → 权衡 → 实践案例”准备。
- 架构师面试不只关注“怎么做”，还需要说明“为什么这样做”“替代方案是什么”“风险与成本是什么”。

---

# 第一章：架构师基础能力

## 第一节：架构设计原则

### 1. SOLID、DRY、KISS、YAGNI 与关注点分离
### 2. 高内聚、低耦合、模块化与分层设计
### 3. 可扩展性、可维护性、可测试性与可演进性
### 4. CAP、PACELC、一致性模型与可用性权衡
### 5. 架构决策记录（ADR）与技术债管理

## 第二节：系统设计方法

### 1. 需求澄清：功能需求、非功能需求与约束条件
### 2. 容量估算：QPS、并发量、存储量、带宽与增长率
### 3. 高层设计：上下文图、容器图、组件图与部署图
### 4. API、数据模型、核心流程与异常流程设计
### 5. 性能、可靠性、安全性、成本与交付速度之间的权衡

## 第三节：非功能性需求

### 1. SLA、SLI、SLO、错误预算与服务等级管理
### 2. 延迟、吞吐量、并发性与容量规划
### 3. 可用性、容错、灾难恢复、RTO 与 RPO
### 4. 安全、合规、隐私、审计与数据治理
### 5. 可观测性、可运营性与成本优化

## 第四节：架构沟通与领导力

### 1. 面向技术团队和业务方表达架构方案
### 2. RFC、ADR、架构评审与技术路线图
### 3. 跨团队协作、冲突处理与共识建立
### 4. Build vs Buy、技术选型与供应商评估
### 5. 项目复盘、风险管理与架构治理

---

# 第二章：React 与前端架构

## 第一节：React 核心原理

### 1. Virtual DOM、Diff、Reconciliation 与 Fiber
### 2. 函数组件、Hooks、闭包与组件生命周期
### 3. useState、useEffect、useMemo、useCallback 与 useRef
### 4. Context、受控组件、非受控组件与状态提升
### 5. Error Boundary、Suspense、并发渲染与流式渲染

## 第二节：前端应用架构

### 1. SPA、MPA、SSR、SSG、ISR 与 React Server Components
### 2. 组件分层、Feature-based 目录与领域边界
### 3. 状态管理：Redux Toolkit、Zustand、Context 与服务端状态
### 4. 数据请求：React Query、缓存、失效、重试与乐观更新
### 5. 微前端、Module Federation、独立部署与团队边界

## 第三节：性能与用户体验

### 1. Core Web Vitals：LCP、INP 与 CLS
### 2. Code Splitting、Lazy Loading、Tree Shaking 与 Bundle 分析
### 3. 缓存策略、CDN、资源压缩与图片优化
### 4. 大列表虚拟化、防抖、节流与渲染优化
### 5. 性能监控、前端埋点、错误追踪与用户体验指标

## 第四节：前端工程化与质量

### 1. TypeScript 类型系统、泛型与严格模式
### 2. Vite、Webpack、Babel、ESLint 与代码规范
### 3. 单元测试、组件测试、集成测试与 E2E 测试
### 4. Design System、Storybook、主题与可访问性（WCAG）
### 5. 前端安全：XSS、CSRF、CSP、Token 存储与依赖安全

## 第五节：React 架构面试题

### 1. 设计一个大型多团队 React 应用
### 2. 设计支持 SSR、SEO 和高并发的电商前端
### 3. 如何选择全局状态、服务端状态与 URL 状态
### 4. 如何逐步迁移遗留前端而不中断业务
### 5. 如何定义前端平台的质量门禁和可观测性

---

# 第三章：AI 全栈架构

## 第一节：AI 与大模型基础

### 1. Transformer、Token、Embedding 与 Attention 基础
### 2. Prompt、上下文窗口、温度与结构化输出
### 3. 推理模型、多模态模型与模型能力边界
### 4. Fine-tuning、Prompt Engineering 与 RAG 的选择
### 5. 准确率、延迟、吞吐量、Token 成本与模型选型

## 第二节：RAG 系统设计

### 1. 文档采集、清洗、切分、元数据与版本管理
### 2. Embedding 模型、向量数据库与索引策略
### 3. 语义检索、关键词检索、混合检索与重排序
### 4. Query Rewrite、Context Compression 与引用溯源
### 5. RAG 评估：召回率、相关性、忠实度与答案质量

## 第三节：Agent 与工具调用

### 1. Function Calling、Tool Use 与结构化参数校验
### 2. Workflow、Agent、Planner 与状态机的区别
### 3. Memory、会话状态、长期记忆与上下文管理
### 4. 多 Agent 协作、任务分解与失败恢复
### 5. Human-in-the-loop、审批机制与高风险操作控制

## 第四节：AI 应用全栈实现

### 1. React 聊天界面、流式响应、取消与重试
### 2. Java/Python AI Gateway、模型适配层与 API 设计
### 3. SSE、WebSocket、异步任务与长耗时工作流
### 4. Prompt 模板、版本管理、缓存与灰度发布
### 5. 用户反馈、对话记录、追踪信息与分析平台

## 第五节：AI 安全与治理

### 1. Prompt Injection、越权工具调用与数据泄露
### 2. PII 脱敏、内容过滤、租户隔离与访问控制
### 3. 幻觉检测、引用验证、Guardrails 与输出校验
### 4. 模型评测、红队测试、偏见与合规审查
### 5. 审计日志、模型卡、数据血缘与责任边界

## 第六节：LLMOps 与生产化

### 1. Prompt、模型、数据集和评测集的版本管理
### 2. Offline Evaluation、Online Evaluation 与 A/B Test
### 3. Token、延迟、错误率、质量与成本监控
### 4. 限流、降级、回退模型、语义缓存与故障隔离
### 5. 自托管模型、托管 API 与混合模型策略

## 第七节：AI 架构面试题

### 1. 设计一个企业知识库问答系统
### 2. 设计一个可调用内部系统的智能 Agent
### 3. 如何在准确率、延迟与成本之间进行权衡
### 4. 如何防止模型访问或泄露未授权数据
### 5. 如何评估并持续改进一个生成式 AI 产品

---

# 第四章：Java 后端架构

## 第一节：Java 核心与 JVM

### 1. 集合、泛型、反射、注解与异常体系
### 2. 线程、线程池、锁、CAS 与并发容器
### 3. CompletableFuture、虚拟线程与响应式编程
### 4. JVM 内存模型、类加载与对象生命周期
### 5. GC 算法、内存泄漏、性能分析与 JVM 调优

## 第二节：Spring 生态

### 1. IoC、DI、AOP、Bean 生命周期与自动配置
### 2. Spring Boot 配置、Starter、Profile 与 Actuator
### 3. Spring MVC、WebFlux 与请求处理模型
### 4. Spring Data JPA、事务传播与连接池
### 5. Spring Security、OAuth 2.0、OIDC 与方法级鉴权

## 第三节：API 设计

### 1. REST 资源建模、HTTP 语义与状态码
### 2. 分页、过滤、排序、批处理与部分更新
### 3. API 版本、向后兼容、幂等性与错误模型
### 4. OpenAPI、Contract First 与消费者驱动契约测试
### 5. REST、GraphQL、gRPC 与异步 API 的选择

## 第四节：数据与持久化

### 1. 关系型数据库建模、范式与反范式
### 2. 索引、执行计划、锁、隔离级别与死锁
### 3. ACID、事务边界、乐观锁与悲观锁
### 4. Redis 缓存模式、缓存穿透、击穿与雪崩
### 5. SQL、NoSQL、搜索引擎与向量数据库选型

## 第五节：后端性能与可靠性

### 1. 性能基线、压测、Profiling 与瓶颈定位
### 2. 连接池、线程池、批处理与背压
### 3. 超时、重试、熔断、隔离与限流
### 4. 幂等、去重、补偿与最终一致性
### 5. 日志、指标、链路追踪与线上问题诊断

## 第六节：Java 架构面试题

### 1. 设计一个高并发订单服务
### 2. 如何排查 Java 服务 CPU 或内存异常
### 3. 如何保证接口幂等与数据一致性
### 4. 如何从同步阻塞系统演进到异步架构
### 5. 如何设计兼容旧客户端的 API 演进策略

---

# 第五章：微服务架构

## 第一节：服务拆分与领域建模

### 1. 单体、模块化单体与微服务的适用场景
### 2. DDD、统一语言、限界上下文与聚合
### 3. 按业务能力拆分服务与识别服务边界
### 4. 数据所有权、团队自治与康威定律
### 5. 服务粒度、分布式复杂度与演进式拆分

## 第二节：服务通信

### 1. 同步 REST/gRPC 与异步消息的选择
### 2. API Gateway、BFF、服务发现与负载均衡
### 3. Kafka/RabbitMQ、Topic、Partition 与 Consumer Group
### 4. 消息顺序、重复消费、丢失处理与死信队列
### 5. Schema Registry、事件版本与兼容性管理

## 第三节：分布式数据一致性

### 1. 分布式事务、2PC 与最终一致性
### 2. Saga 编排、事件协同与补偿事务
### 3. Transactional Outbox、CDC 与可靠事件发布
### 4. CQRS、Event Sourcing 与读写模型分离
### 5. 幂等键、去重表与一致性校验

## 第四节：弹性与稳定性

### 1. Timeout、Retry、Circuit Breaker 与 Bulkhead
### 2. Rate Limiting、Load Shedding 与 Backpressure
### 3. 健康检查、优雅启动、优雅关闭与滚动发布
### 4. 故障注入、Chaos Engineering 与恢复演练
### 5. 多可用区、多区域与灾难恢复架构

## 第五节：服务治理与可观测性

### 1. 集中日志、结构化日志与关联 ID
### 2. Metrics、Tracing、OpenTelemetry 与告警设计
### 3. Service Mesh、Sidecar 与零信任通信
### 4. 服务目录、API 治理与依赖关系管理
### 5. Golden Signals、SLO Dashboard 与故障复盘

## 第六节：微服务架构面试题

### 1. 什么时候不应该使用微服务
### 2. 如何把大型单体逐步拆分为微服务
### 3. 如何处理跨服务事务和数据一致性
### 4. 如何设计可承受下游故障的服务调用链
### 5. 如何治理数百个服务的版本、依赖与可观测性

---

# 第六章：CI/CD 与 DevOps

## 第一节：持续集成

### 1. Git Flow、Trunk-based Development 与分支策略
### 2. Build、Lint、Test、Scan 与质量门禁
### 3. 单元、集成、契约、E2E 与性能测试金字塔
### 4. 制品版本、Artifact Repository 与可重复构建
### 5. Pipeline as Code、并行执行、缓存与反馈速度

## 第二节：持续交付与部署

### 1. Continuous Delivery 与 Continuous Deployment 的区别
### 2. Rolling、Blue-Green、Canary 与 Feature Flag
### 3. 数据库迁移、Expand-and-Contract 与向后兼容
### 4. 自动回滚、发布验证与渐进式交付
### 5. 多环境配置、环境一致性与审批策略

## 第三节：容器与 Kubernetes

### 1. Dockerfile、镜像分层、多阶段构建与镜像安全
### 2. Pod、Deployment、Service、Ingress 与 ConfigMap
### 3. Requests、Limits、HPA、PDB 与调度策略
### 4. Helm、Kustomize、Operator 与集群配置管理
### 5. Kubernetes 网络、安全、存储与故障排查

## 第四节：Infrastructure as Code 与 GitOps

### 1. Terraform 模块、State、Provider 与环境隔离
### 2. AWS CDK、CloudFormation 与 Terraform 的选择
### 3. GitOps、Argo CD、期望状态与漂移检测
### 4. Secret 管理、密钥轮换与最小权限
### 5. Policy as Code、合规检查与基础设施测试

## 第五节：DevSecOps 与供应链安全

### 1. SAST、DAST、SCA、IaC Scan 与 Secret Scan
### 2. SBOM、依赖签名、镜像签名与来源证明
### 3. 漏洞等级、风险接受与修复 SLA
### 4. CI/CD 身份认证、短期凭证与权限隔离
### 5. 审计、合规证据与发布责任分离

## 第六节：CI/CD 架构面试题

### 1. 设计支持数百个微服务的交付平台
### 2. 如何实现零停机部署与安全数据库变更
### 3. 如何提高慢速 Pipeline 的反馈效率
### 4. 如何在速度、质量、安全与审批之间平衡
### 5. 如何设计跨账户、跨区域的发布和回滚流程

---

# 第七章：AWS 云架构

## 第一节：AWS 基础与账户治理

### 1. Region、Availability Zone 与 Edge Location
### 2. AWS Organizations、Control Tower 与多账户策略
### 3. IAM User、Role、Policy、STS 与最小权限
### 4. Tagging、配额、资源治理与责任分配
### 5. Shared Responsibility Model 与 Well-Architected Framework

## 第二节：网络架构

### 1. VPC、Subnet、Route Table、NAT Gateway 与 Internet Gateway
### 2. Security Group、NACL、PrivateLink 与 VPC Endpoint
### 3. VPC Peering、Transit Gateway 与混合网络
### 4. Route 53、CloudFront、Global Accelerator 与 DNS 策略
### 5. WAF、Shield、Network Firewall 与分层防护

## 第三节：计算与容器

### 1. EC2、Auto Scaling、Launch Template 与负载均衡
### 2. ECS、EKS、Fargate 与 App Runner 的选择
### 3. Lambda、API Gateway、EventBridge 与 Serverless
### 4. Graviton、Spot、Reserved Instances 与 Savings Plans
### 5. 多 AZ 部署、无状态服务与弹性伸缩

## 第四节：存储与数据库

### 1. S3 存储类型、生命周期、版本控制与复制
### 2. EBS、EFS、FSx 与不同存储模型
### 3. RDS、Aurora、Read Replica 与 Multi-AZ
### 4. DynamoDB 分区键、GSI、容量模式与 Global Tables
### 5. ElastiCache、OpenSearch、Redshift 与数据服务选型

## 第五节：集成与消息服务

### 1. SQS Standard/FIFO、可见性超时与死信队列
### 2. SNS、EventBridge 与事件路由
### 3. MSK/Kinesis 与流式数据处理
### 4. Step Functions 与分布式工作流编排
### 5. API Gateway、AppSync 与服务集成模式

## 第六节：安全与合规

### 1. KMS、Secrets Manager、Parameter Store 与证书管理
### 2. CloudTrail、Config、GuardDuty 与 Security Hub
### 3. Cognito、IAM Identity Center 与身份联邦
### 4. 数据加密、密钥轮换、备份与数据驻留
### 5. 零信任、纵深防御与安全事件响应

## 第七节：可观测性与运维

### 1. CloudWatch Metrics、Logs、Alarms 与 Dashboard
### 2. X-Ray、OpenTelemetry 与分布式追踪
### 3. Systems Manager、自动化运维与补丁管理
### 4. AWS Backup、DR 演练与跨区域恢复
### 5. Incident Management、Runbook 与自动修复

## 第八节：成本优化与 FinOps

### 1. Cost Explorer、Budgets、CUR 与成本分摊
### 2. Rightsizing、自动伸缩与闲置资源治理
### 3. 存储生命周期、数据传输与 NAT 成本优化
### 4. Spot、Savings Plans、Reserved Instances 的组合策略
### 5. 单位经济成本、成本异常检测与 FinOps 文化

## 第九节：AWS 架构面试题

### 1. 设计高可用、可扩展的互联网应用
### 2. 设计跨区域容灾并定义 RTO/RPO
### 3. 设计安全的企业多账户 Landing Zone
### 4. 设计事件驱动的 Serverless 系统
### 5. 如何在可靠性、性能、安全与成本之间权衡

---

# 第八章：端到端系统设计案例

## 第一节：AI 驱动的企业知识平台

### 1. React 前端、身份认证与流式问答体验
### 2. Java API、AI Gateway、RAG 与向量检索
### 3. 文档接入、异步处理、权限过滤与引用溯源
### 4. AWS 部署、可观测性、安全与成本控制
### 5. 质量评测、反馈闭环、灰度发布与模型回退

## 第二节：高并发电商平台

### 1. 商品、购物车、订单、支付与库存领域拆分
### 2. 缓存、搜索、消息队列与流量削峰
### 3. 超卖防护、分布式事务、幂等与补偿
### 4. 多区域容灾、自动扩展与热点活动保障
### 5. React 性能、CI/CD、安全与业务监控

## 第三节：实时通知与消息平台

### 1. WebSocket/SSE、连接管理与消息路由
### 2. Kafka/Kinesis、顺序性、重试与投递语义
### 3. 用户偏好、模板、频率控制与多渠道发送
### 4. 峰值容量、背压、降级与可观测性
### 5. 多租户隔离、隐私合规与成本模型

## 第四节：遗留系统现代化

### 1. 现状评估、依赖分析与目标架构定义
### 2. Strangler Fig、模块化改造与渐进式迁移
### 3. 数据迁移、双写、CDC 与一致性验证
### 4. 云迁移策略：Rehost、Replatform 与 Refactor
### 5. 风险控制、业务连续性、团队组织与路线图

---

# 第九章：行为面试与项目经验

## 第一节：项目案例准备

### 1. 使用 STAR 方法描述背景、任务、行动与结果
### 2. 准备一个成功项目、一个失败项目和一个救火案例
### 3. 用性能、稳定性、成本或业务指标量化结果
### 4. 明确个人贡献、关键决策与团队协作方式
### 5. 说明复盘结论以及下一次会如何改进

## 第二节：架构决策案例

### 1. 一次重要技术选型及其替代方案
### 2. 一次性能、成本或可靠性的重大权衡
### 3. 一次跨团队架构分歧与共识建立
### 4. 一次技术债治理或遗留系统现代化
### 5. 一次生产事故处理与长期改进

## 第三节：领导力问题

### 1. 如何影响没有直接汇报关系的团队
### 2. 如何建立技术标准而不阻碍交付
### 3. 如何指导工程师并提升团队能力
### 4. 如何处理业务期限与架构质量的冲突
### 5. 如何向管理层解释复杂技术风险

---

# 第十章：面试冲刺与检查清单

## 第一节：知识检查

### 1. 每个核心主题能否在两分钟内解释清楚
### 2. 每项技术能否说明适用场景和不适用场景
### 3. 每个架构方案能否给出至少一个替代方案
### 4. 是否能解释性能、安全、可靠性与成本权衡
### 5. 是否能把技术方案映射到真实业务价值

## 第二节：系统设计演练

### 1. 练习需求澄清与非功能指标定义
### 2. 练习容量估算和核心数据模型设计
### 3. 练习高层架构图、关键链路与 API 设计
### 4. 练习识别瓶颈、单点故障与安全风险
### 5. 练习演进路线、成本分析与方案总结

## 第三节：个人材料

### 1. 一分钟、三分钟和十分钟自我介绍
### 2. 个人架构项目清单与关键数据指标
### 3. 架构图、ADR、复盘报告与技术文章示例
### 4. 针对职位描述准备能力匹配表
### 5. 准备向面试官询问的团队与架构问题

## 第四节：面试前最终检查

### 1. 熟悉目标公司的业务、产品和技术公开信息
### 2. 准备白板画图、在线画图与口头设计流程
### 3. 检查开发环境、网络、摄像头和共享屏幕
### 4. 准备简洁、量化且诚实的项目答案
### 5. 保留时间反问架构挑战、团队职责与成功标准

