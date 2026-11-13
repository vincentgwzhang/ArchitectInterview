# 04. OpenAPI、Contract First 与消费者驱动契约测试

> 本章示例固定采用 OpenAPI 3.1.1，Schema 按其 JSON Schema 2020-12 体系理解，不把 3.0 的 nullable 写法混入。Pact 等工具的 DSL/CLI 随版本变化，本文讲清工作流程和验证对象，不将伪代码冒称可直接运行的某版 SDK。所有订单场景均为教学设计。

## 1. API Contract 是跨团队的交付边界

契约（Contract）描述调用方可以发送什么、服务承诺返回什么，以及失败后可以怎样恢复。OpenAPI 是机器可读的 HTTP 接口描述，能够表达路径、参数、请求/响应结构、媒体类型、安全方案和示例，但不能单独证明业务正确。

贯穿例子：客服前端查询订单，订单服务负责真实状态与租户隔离。前端需要 id、state 和金额单位；服务还必须保证无权访问时不泄露对象。两队若等后端全部完成再沟通，就很晚才发现字段、错误与权限假设不一致。先评审契约可以让双方并行开发，但错误的共同假设也会使两边测试一起通过，因此契约不是免思考文件。

一个完整契约应包含结构和文字语义。例如 totalMinor 的整数 Schema 不能说明是“最小货币单位”还是“元”；200 响应结构不能说明数据来自一致快照；security 声明不能说明同角色是否能看别人的订单。业务不变量、恢复窗口、授权范围和兼容政策必须在正文/description 及相应测试里补全。

## 2. Contract First 与 Code First 如何选择

### 2.1 Contract First 的实际协作流程

Contract First（契约优先）先提出接口描述和交互案例，邀请消费者、提供方和必要的安全/平台负责人评审；冻结可实现的第一版，再分别开发客户端、服务与 Mock。不是先写一份巨大 YAML，未经需求验证便强迫所有人照做。

例如查询订单评审要确认资源 ID、当前主体可见范围、state 的未知值行为、金额单位、404 是否隐藏无权限对象、错误 code 和 traceId。讨论能提前暴露“不存在和无权是否区分”“前端能否处理新的枚举”等依赖，减少联调才改接口的返工。

好处是设计不被 ORM Entity 或 Spring Controller 参数形状牵着走，适合多团队/外部 API；成本是规范、实现和测试需要持续同步，早期需求不清时也可能反复修改。可先做小范围纵向切片，验证一个真实用例后扩展，不用一次写完所有潜在能力。

### 2.2 Code First 的边界与漂移控制

Code First（代码优先）从 Controller、DTO 和注解导出描述，上手快，适合现有服务补文档或代码驱动的小团队。但代码可见不代表外部契约合理：持久化内部字段、错误状态漏注解、默认分页等可能进入公开接口。

采用 Code First 时，应将生成后的规范作为被审查、可比较的制品，与已发布基线做兼容检查；对真实运行接口做符合性测试，确保注解不是空承诺。采用 Contract First 时，则将设计规范设为权威输入，验证实现符合它。两种方式都不能允许“手写文档一份、运行导出一份、生成客户端又用另一份”。

Source of Truth（事实来源）不是仅有一个文件名，而是一条可追溯流程：谁允许改、什么版本发布、制品摘要是什么、部署实现如何关联、发现差异时以何种门禁阻止漂移。可明确采用设计文件为权威或生成规范为权威，避免双向手工维护彼此覆盖。

## 3. OpenAPI 文档各部分做什么

openapi 声明规范版本；info.version 是本 API 描述的版本，不是 OpenAPI 语法版本，也不必等于 URL Major；servers 给出目标服务器模板；paths 定义资源操作；components 复用 Schema、响应和安全方案。operationId 应稳定且唯一，生成客户端常用它命名方法，随便改它可能造成源码层兼容问题。

参数要定义位置（path/query/header/cookie）、类型和编码。path 参数必须 required；数组 Query 的 style/explode 决定是重复键还是分隔列表，不能只写 array 就假设所有客户端相同。Content-Type 描述实际内容，Accept 表示期望响应类型；请求格式不支持与响应协商失败应分开。

required 数组表示对象成员必须出现，不自动表示该值不允许 null。3.1 可用 type: [string, "null"] 表达可空；允许省略又不可 null 与必有但可 null 是不同集合。default 通常是描述性注解，不保证服务器或生成器自动填值；需要在业务与测试中执行默认策略。

example/examples 帮助理解，不取代 Schema。枚举、范围、格式和对象附加成员规则都要按请求/响应方向评估：响应过严可能阻碍兼容扩展；请求过宽可能让拼错字段被静默忽略。format 的验证支持随工具和配置不同，不能以写了 uuid 就推断运行时一定拒绝任意字符串。

## 4. 一份可评审的 OpenAPI 3.1.1 查询示例

下面是完整的单操作描述，使用匿名占位域名。它描述 GET 的结构与外部语义，不是可直接部署的订单服务。为了避免误导，以 HTTP Bearer 安全方案声明 Access Token；scope 映射、issuer/audience 和对象授权仍由实现负责。

~~~yaml
openapi: 3.1.1
info:
  title: Order Query API
  version: 1.0.0
servers:
  - url: https://orders.example.invalid
security:
  - bearerAuth: []
paths:
  /orders/{orderId}:
    get:
      operationId: getOrder
      summary: Read a visible order
      description: >
        Returns only an order visible to the authenticated subject in its trusted
        tenant scope. Missing and non-visible orders use the same 404 contract.
        Amount is an integer number of minor units of the indicated currency.
      parameters:
        - name: orderId
          in: path
          required: true
          schema:
            type: string
            minLength: 1
            maxLength: 80
      responses:
        '200':
          description: Visible order
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Order'
              example:
                id: ord-42
                state: PAID
                totalMinor: 1234
                currency: EUR
        '401':
          description: Missing or invalid credentials
          headers:
            WWW-Authenticate:
              schema:
                type: string
          content:
            application/problem+json:
              schema:
                $ref: '#/components/schemas/Problem'
        '404':
          description: Order not found or not visible
          content:
            application/problem+json:
              schema:
                $ref: '#/components/schemas/Problem'
        '503':
          description: Temporarily unable to serve the query
          content:
            application/problem+json:
              schema:
                $ref: '#/components/schemas/Problem'
components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
  schemas:
    Order:
      type: object
      required: [id, state, totalMinor, currency]
      properties:
        id:
          type: string
          minLength: 1
          maxLength: 80
        state:
          type: string
          description: >
            Known values include PENDING, PAID and CANCELLED.
            Consumers must preserve unknown values and must not treat them as success.
        totalMinor:
          type: integer
          minimum: 0
          maximum: 9007199254740991
          description: Exact minor units; never a floating point major-unit amount.
        currency:
          type: string
          pattern: '^[A-Z]{3}$'
          description: Valid supported currency code; pattern alone is not a code registry.
        note:
          type: [string, 'null']
          maxLength: 200
    Problem:
      type: object
      required: [type, title, status, code, traceId]
      properties:
        type:
          type: string
          format: uri
        title:
          type: string
        status:
          type: integer
          minimum: 400
          maximum: 599
        detail:
          type: string
        code:
          type: string
        traceId:
          type: string
~~~

state 刻意用开放字符串并描述已知值，以支持客户端 UNKNOWN 回退；若业务必须封闭枚举，也可声明 enum，但新增值是否破坏生成 SDK 必须验证。开放字符串不等于任何状态都可由客户端写入，本例只有读取操作，写模型应独立限制。

totalMinor 的上限是教学接口为 JavaScript 安全整数边界所作的契约选择，仍需客户端避免错误浮点运算；业务若要更大值，可选择十进制字符串与明确范围。currency 的正则只证明三个大写字母，不证明代码有效或该币种支持，领域校验不可省略。

note 可省略，也可出现并为 null；这两个输入在此读取模型都合法，但含义仍应由业务约定。没有 additionalProperties:false 意味此处允许额外属性，便于响应扩展；不能从此反推所有请求也要允许任意字段。

401 的 Header 在文档声明并不自动强制实现必有，测试应另断言 WWW-Authenticate。Problem 中 code/traceId 是本应用要求的扩展，非 RFC 9457 强制成员；status 与实际 HTTP 状态的一致性也需跨报文测试，不由单个 JSON Schema 单独保证。

## 5. OpenAPI 治理：每一道检查提供什么证据

### 5.1 版本控制、Lint 与 Compatibility Diff

把设计文件、复用组件、文字语义、示例和工具配置纳入版本控制，PR 显示契约变化而不只显示生成代码。发布不可变规范制品及摘要，API Catalog（接口目录）关联负责人、支持版本、访问方式、弃用状态与部署，不能只堆 Swagger UI 链接却找不到责任人。

Lint（静态规范检查）检查可执行的团队规则，例如 operationId 唯一、path 参数完整、错误响应存在、分页参数上限和安全声明。规则应解释目标与例外：健康端点可匿名，但必须显式审查；不能为过 Lint 给所有接口机械添加一个从不实现的错误响应。

Compatibility Diff 比较候选与已发布规范，识别字段删除、类型/必填/范围和状态变化。基线要选真正支持中的版本，而非随便比较主分支最新；工具应固定版本，误报/漏报通过可追溯例外处理，不能把退出码永久忽略。

结构差异无法读懂“分改成元”“默认排序反转”“原来同步完成现只接受”。这些需文字变更审查、业务案例和消费者测试。工具门禁越容易制造绿色假象，越要写清它不验证什么。

### 5.2 生成 Client/Server Stub 的收益和陷阱

生成客户端减少手写序列化、路径和错误类型；生成服务接口/Stub 提供方法边界，业务实现依旧需要自己完成。生成模型是传输类型，不应直接作为 ORM Entity 或领域聚合，避免外部字段变化迫使数据库结构和内部规则一起变化。

固定生成器版本、模板、参数和输入摘要，保证可复现；生成后编译与关键序列化测试，不因“生成成功”就认定行为正确。直接改生成目录会在下次覆盖丢失；扩展应放适配器、受控模板或生成器支持的定制点，且模板本身要版本化。

尤其验证可空/缺失、日期时区、大整数、未知枚举、oneOf 等复杂模型与错误 Body。SDK 默认超时和重试也属于交付行为：生成代码对 POST 自动重试可能制造副作用，需应用层配置幂等和恢复，不把生成器的默认策略当业务真理。

### 5.3 实现符合性与文档发布顺序

请求校验可检查路径/参数/Body 是否满足规范，响应校验发现真实状态码、字段和媒体类型偏离。但生产强制响应校验会有性能与可用性成本，可在 CI、预生产、采样观测等位置部署，按风险选择，不能未经预算直接让验证器成为线上单点。

发布时记录规范制品与服务构建关联，Catalog 不能提前把尚未部署能力标成已可用；设计预览与稳定发布分开。Code First 导出应来自明确构建，Contract First 则验证该构建符合已批准规范；两者都要防止手工发布错文件。

发现漂移应判断是实现 Bug 还是规范需要合法演进。不能为让测试绿而无条件把规范更新成线上错误行为，尤其金额、权限与幂等承诺的退化必须经过设计决策。

## 6. CDC 的两个半场：消费者测试与提供方验证

### 6.1 消费者测试必须调用真实客户端代码

Consumer-driven Contract（消费者驱动契约，CDC；此处不是数据库 Change Data Capture）表达消费者真正依赖的交互。消费者使用契约工具启动 Mock Provider，注册预期请求与最低响应，再让自己的实际 API 客户端调用它，断言业务代码正确理解结果。

教学伪代码，不是某个 Pact SDK 的可运行语法：

~~~text
Given provider state: tenant A owns order ord-42 in PAID state
Expected request: GET /orders/ord-42 with required authentication mechanism
Example response: 200 {id:"ord-42", state:"PAID", totalMinor:1234, currency:"EUR"}
Run real consumer OrderClient.getOrder("ord-42")
Assert consumer model keeps totalMinor=1234 and currency="EUR"
Assert presentation formats it according to currency rules, not as 1234 major units
Publish contract with immutable consumer build identity
~~~

若测试仅检查 mock 自己吐出的 JSON，没有调用真实客户端序列化/解析路径，它没有证明消费者能使用接口。若客户端真会读取 currency，契约也必须包含它；只声明 id 一个字段会错过关键依赖。

匹配器应表达真实保证而不是偶然样本：订单 ID 常检查类型/格式，固定业务状态的前置条件下可检查 PAID；traceId 通常无需等于某个固定字符串。过度精确会让无害变化失败，过度宽松会让错单位、错状态通过。对未知枚举回退另建消费者行为测试，不靠一个正常样本推断。

### 6.2 提供方验证真实实现与前置状态

Provider Verification 读取已发布契约，为每个交互建立 Provider State（提供方前置状态），向真实待测实现发送请求，并核对响应匹配。订单查询需准备租户/主体/订单及业务状态，不应直接 Stub Controller 返回契约 JSON，否则只证明复制文档成功。

状态设置必须可重复、交互独立并清理数据，不依赖测试执行顺序或共享环境恰好已有 ord-42。可以替代下游来隔离测试，但要记录证据范围：Repository 全被 mock 时，无法证明真实 SQL 的租户限制和数据映射。

认证处理要谨慎。用测试专用凭证和身份平台/验证器配置，不能关闭安全链后宣称认证契约通过。若为某类交互绕开认证，应将其限定为响应结构验证，另用安全集成测试补真实 issuer/audience、scope 和对象授权。契约制品中不要保存生产 Token、个人数据或客户端 secret。

验证完成需发布“哪个 provider build 验证了哪个 consumer contract”的结果。只在日志写 PASS，没有可追溯版本，就无法判断当前待部署构建与生产客户端是否兼容。

## 7. Broker、版本矩阵与独立发布

Broker 是保存契约、验证结果与部署/发布关系的协作服务，不是业务请求运行时必须调用的代理。它帮助回答“这个候选版本是否验证过当前环境中真实存在的对端版本”。

教学版本矩阵：

| 消费者版本 | 提供方 P10 | 提供方 P11 |
|---|---|---|
| C7，当前生产使用 | 通过 | 通过 |
| C8，候选，需要新增字段 | 失败 | 通过 |

若生产仍是 P10，C8 不能只因为与 P11 通过就先上线；可先部署同时支持 C7/C8 的 P11，再发布 C8。若 C8 已在用，回滚 P11 到 P10 也要重新检查矩阵，不能默认旧版本永远安全。矩阵只反映已验证交互，不证明全部业务能力。

发布门禁（例如 Pact 的 can-i-deploy）要结合可靠的部署/发布记录与消费者选择策略。仅取 latest 契约可能漏掉仍在线的旧版本；移动端或外部应用常有多版长期共存，应登记仍受支持的发布版本，不可把某个 Git 分支头误当全部线上用户。

缺结果是“不知道”，不是通过。新分支契约可使用工具支持的 pending/WIP 等流程避免立刻阻断所有无关提供方开发，但新能力消费者上线前仍要获得必要验证；不能把 pending 永久当免验证标签。例外须有负责人、范围、期限和替代证据。

Broker 权限要保护契约和验证结果的可信性，构建身份尽量不可变。不要复用一个可变版本号覆盖不同内容，也不要随意删除“看起来旧”的消费者契约；退役前用实际部署记录和负责人确认。否则矩阵会展示虚假的安全。

## 8. Mock、Sandbox 与各种测试怎样互补

| 机制 | 主要证明 | 不直接证明 |
|---|---|---|
| OpenAPI 解析/Lint | 描述结构、约定及静态规则 | 服务实际履约 |
| Schema/兼容 Diff | 可表达结构变化风险 | 单位、默认行为和所有消费者依赖 |
| 消费者 Mock 测试 | 客户端构造/理解约定交互 | Provider 实现正确 |
| Provider 验证 | 指定构建满足选中契约 | 未记录交互、真实外部故障 |
| 功能/数据库集成测试 | 领域规则、持久化与隔离边界 | 所有消费者都能解析 |
| Sandbox/E2E | 部署链路的代表性真实交互 | 无限负载和所有故障均安全 |

由同一规范生成 Mock 可让前端并行开发，Sandbox 提供受控测试环境与凭证，帮助验证更完整集成。它们应标明模拟数据、支持范围和重置方式，避免被误当生产；真实支付只用沙箱，不在测试中发送真实扣款。

Mock 可能一直返回一个合法 200，掩盖 404、超时、授权拒绝与未知枚举。需要明确失败场景、延迟和数据状态，但模拟延迟不等于生产性能证明。Sandbox 也可能部署滞后、数据太少或安全配置放宽，必须核对它与生产的差异。

CDC 不替代 Provider Functional Test：例如库存扣减非负、幂等同键并发、事务回滚和退款状态机需要服务端测试。也不替代少量关键 E2E：证书、网关路径、CORS、身份平台、序列化与网络配置可能在真实链路出错。测试策略按风险组合，不用一个工具宣称删除其他所有测试。

## 9. Breaking Change：结构、行为与安全逐项判定

请求新增必填、缩小合法范围、禁止原来允许的 null，通常使旧请求失败；响应删除保证字段、改变类型或状态码会破坏读取。响应新增枚举、optional 字段也可能使严格 SDK 失败，不能把任何“添加”一概归为安全。

改变默认排序、金额单位、错误 code 或同步完成语义，可能在结构不变时破坏业务；新旧 Schema Diff 看不出这些变化。PR 应给具体旧请求/响应与消费者动作的前后对比，解释为什么仍兼容或如何迁移，而不是只写“非破坏性重构”。

安全修复有时必须收紧错误放行行为，不能为了兼容保留越权。应明确安全风险、受影响客户端、迁移/应急路径与审计，采用受控流程，而不是把所有授权变化当成无需说明的内部细节。

性能也影响契约：新增昂贵字段让响应超出消费者超时，或 totalCount 造成每页大扫描，可能在生产上失败。规范可以描述预算，但需要真实测量与压测验证，不能因为 response schema 正确就忽略延迟与容量。

## 10. 一个从设计 PR 到部署的完整流程

先用订单查询纵向切片：消费者列实际读取字段和恢复动作，提供方列授权与数据约束。设计 PR 提交 OAS、文字语义、成功和失败示例、版本影响及 Owner；共同评审金额单位、未知状态和 404 隐藏政策。

CI 首先固定工具版本并解析/解析引用，检查 Lint、示例和发布基线 Diff。随后生成客户端/服务接口并编译，验证缺失/null、错误解析等边界；实现映射到领域类型，不直接暴露 Entity。规范样例与 Schema 要互相校验，避免样例本身长期错误。

消费者测试调用真实客户端并发布不可变契约；提供方准备隔离状态、启动真实接口验证选中消费者版本并发布结果。数据库、授权与并发功能测试独立运行；关键 E2E 穿过网关/身份链路。所有结果关联同一个待发布构建，不能拿昨天构建的通过证明今天制品安全。

部署前查候选与目标环境/支持发布的矩阵，未知或失败按规则阻断。部署后记录版本，更新 Catalog 稳定制品与环境指向，观测错误/延迟/契约漂移。回滚也需矩阵和数据兼容判断，不能在紧急时刻默认向后随便退。

外部消费者无法直接提交 Pact 时，可提供公开 OAS、官方 SDK、代表性客户端回归套件和 Sandbox，并结合客户登记与使用观测。CDC 在已知团队间价值高，不能假装能穷举互联网所有客户端。

## 11. 故障推演：所有契约绿了，生产为何仍然失败

教学场景：提供方把金额从分改为元，字段仍是 integer；契约仅用“任意整数”匹配。生成 Mock 与真实接口都通过，消费者却将值再除以 100，显示错误。

问题是契约没有表达/断言实际依赖的单位。修复要恢复旧语义或新增明确字段，消费者测试断言真实模型/显示行为；提供方功能测试以业务金额验证序列化。不能为测试绿而把消费者断言也同步改错，必须先确认权威业务含义。

另一场景是只验证最新 C8，而生产仍有 C7。P11 删除 C7 依赖字段，latest 验证通过但旧用户报错。修复选取真实支持版本、记录部署关系，并把未验证视为未知。契约文件本身保存正确也不够，选择和门禁逻辑同样是架构的一部分。

如果接口结构正确但跨租户泄露，说明状态和权限测试不足；若一直 Mock Repository，则它从未验证 SQL 范围。补充不同主体/租户、不可见对象、批量导出与缓存测试。绿色结果必须解释“验证了什么”，不能等同所有风险归零。

## 12. 面试完整回答与深入追问

### 12.1 “Contract First 有什么实际价值？”

可表达回答：“它在实现前让消费者和提供方共同确认输入、响应、错误与业务语义，支持双方并行开发。我会把 OAS、示例、文字约束和工具配置版本化，生成边界代码但保持领域模型独立，再用实际接口符合性和消费者契约验证防漂移。成本是治理和同步，所以从真实纵向用例开始，不先写不可实现的大而全规范。”

追问：“Code First 就不合适吗？”答案：不是。已有服务可从代码导出，但生成制品必须评审、与稳定基线比较并验证运行行为。关键是明确权威来源与可验证流程，不是选择一个名字后自动获得一致性。

### 12.2 “CDC 到底验证谁？”

可表达回答：“消费者测试通过 Mock 检查自己的真实客户端能构造请求并理解响应，产生它依赖的契约；提供方验证用对应前置状态调用真实实现，证明某个提供方构建满足这份契约。两边证据与不可变版本进入 Broker，部署前检查目标环境中实际可能共存的版本组合。”

追问：“只跑消费者测试为什么不够？”答案：Mock 是按预期返回，生产者即使从没实现该接口也可能通过。必须验证真实提供方，反过来只有服务测试也不能证明客户端实际解析正确。

### 12.3 “为什么 CDC 不替代 E2E？”

可表达回答：“CDC 缩小了交互验证范围并提供快反馈，但不覆盖所有业务不变量和部署基础设施。数据库事务、真实认证、网关路由、TLS、故障恢复和性能需要其他层证据。我会用契约覆盖大量已知边界，保留少量关键 E2E 与专门功能/故障测试，而不是用一种测试证明所有东西。”

追问：“测试太多如何控制成本？”答案：按失败风险分层，快速结构检查最前，契约只表达真实依赖，昂贵 E2E 选择关键路径。删除重复且无独立证据的检查，不能把关键故障窗口一并删掉。

### 12.4 “Breaking Change 怎样识别？”

可表达回答：“先分别看请求接受范围与响应保证，再看单位、默认、状态码、错误和恢复语义。Diff 找结构变化，消费者测试找真实依赖，业务审查和负载测试补行为与性能。新增字段和枚举也要检查真实 SDK；安全修复允许必要收紧，但应有明确沟通和迁移。”

追问：“Schema 都没变呢？”答案：金额单位、排序、同步变异步、权限和幂等窗口都可能变。契约必须包含文字和行为测试，不能只比较 JSON 类型。

## 13. 扫雷自测与完整推理答案

### 13.1 OAS security 写了 Bearer，是否已保证订单只被所有者读取？

没有。它描述认证方案，不执行验签、受众检查或对象政策。实现需验证 Token 和可信租户/主体，数据查询限定范围；测试要包含同角色他人订单与跨租户。

### 13.2 required: [note] 是否代表 note 不能为 null？

不能单独推断。required 控制成员存在，允许类型控制 null。3.1 可用联合类型表达可空；缺失与 null 要分别测试，不依赖生成器恰好如何映射。

### 13.3 example 是 1234，匹配任意整数是否能保护金额单位？

不能。样例不是完整语义断言，宽匹配只证明数值类型。需明确最小单位并让实际消费者/提供方以业务金额验证解释，不只检查反序列化不报错。

### 13.4 C8 对 P11 通过，当前生产 P10，能直接部署 C8 吗？

不能据此判断。需要 C8 对当前 P10 或实际将共存版本的验证结果；若 P10 不满足，先部署兼容旧新消费者的 P11，再发布 C8，并记录部署关系。

### 13.5 Provider State 直接让 Controller 返回预设 JSON，有何问题？

只证明测试夹具能复现期望，没有验证真实路由、映射与业务读取。应准备前置数据/依赖并调用真实待测实现；必要隔离下游时明确范围，补数据库与权限集成验证。

### 13.6 生成代码为何不直接当 Domain Model？

它服务传输契约和工具，字段与可空策略会随外部 API 改变。直接绑定领域/持久化会把接口变化传到核心模型，暴露内部字段并增加迁移耦合。用适配层明确映射和校验。

### 13.7 Broker 没有失败结果，能否理解为可上线？

不能。缺验证是未知，可能选错消费者、未发布结果或从没测过。门禁需检查具体候选和目标环境/支持发布的完整证据，不能把没有红色当绿色。

### 13.8 Mock 能否证明 p99 延迟和取消释放资源？

不能。Mock 的固定响应和延迟没有真实线程、连接、数据库与取消行为。需要真实负载、慢依赖、断连与资源观测实验；模拟帮助准备客户端恢复，不替代承载证明。

### 13.9 规范与实现不一致，应该总是更新规范吗？

不应。先判断规范是否正确承诺、消费者是否依赖，以及实现是否是 Bug。金额/安全/恢复退化不能通过改文档合法化；合法演进则走兼容评审和迁移流程，保留审计。

## 14. 验收与执行边界

静态检查应覆盖规范版本、内部引用、operationId、路径参数、媒体类型、安全方案与示例。用正确 Schema 方言验证合法响应、字段缺失/null、金额范围和错误结构；验证器支持 format 的方式要明确，不把解析 YAML 成功当 OAS 完整有效。

契约流程验收需证明真实客户端被调用、提供方状态隔离、真实接口执行、结果绑定构建、部署/发布记录可靠、回滚也检查矩阵。负面用例包括选择 latest 漏掉生产旧版、缺验证、状态夹具失败和凭证错误。测试制品与 Broker 不含秘密或生产个人数据。

集成验证覆盖数据库映射、真实安全链、不可见对象、网关错误、故障恢复与关键 E2E。发布 Catalog 的制品摘要应与被验证描述一致，防止文档页面指向与服务实现不同版本。

本次已人工核对原主题覆盖、所有正文小节自足性、示例的 OpenAPI 3.1 语义和工具责任边界。已实际通过 YAML 解析、两个组件的 Draft 2020-12 Schema 自检；合法响应及可省略/可空 note、未知 state/额外字段通过，负金额、超出安全整数上限、错误 note 类型和缺 currency 被拒绝。这是组件级检查，未使用完整 OpenAPI 验证器，不证明所有 OAS 规则；未运行 Pact、生成 SDK 编译、真实 Spring 服务或部署门禁，不把流程伪代码宣称为已运行测试。

## 15. 面试前速记

- OAS 描述结构与交互，业务语义还需文字、不变量和行为测试。
- Contract First 与 Code First 都要明确权威来源、制品版本和实现符合性。
- required 是存在，null 是值类型，default 不自动执行业务默认。
- 生成代码是边界适配，不是 Entity；工具、模板、输入版本都要固定。
- 消费者测试验证真实客户端，提供方验证真实实现，两半不能互相替代。
- Broker 记录版本矩阵，部署看实际共存组合，不只 latest，未知不是通过。
- Mock 支持并行开发，不能证明授权、数据库、故障和性能。
- 结构 Diff 加消费者证据，再补行为、安全与性能；绿色结果有范围。
- API Catalog 关联负责人、稳定制品、环境与弃用，发布和回滚都可追溯。

## 16. 一手来源与版本范围

来源供核验和可选追溯，正文已解释必要概念和推理。OAS 固定在 3.1.1 以避免方言混用，不宣称它是最新版本；Pact/Generator 工具执行应按项目锁定版本验证。

- [OpenAPI Specification 3.1.1](https://spec.openapis.org/oas/v3.1.1.html)：对象、参数、Schema、安全声明及版本语义。
- [Pact 官方：How Pact Works](https://docs.pact.io/getting_started/how_pact_works)：消费者测试、前置状态和提供方验证。
- [Pact Broker：Can I Deploy](https://docs.pact.io/pact_broker/can_i_deploy)：版本矩阵与部署判断。
- [OpenAPI Generator：Customization](https://openapi-generator.tech/docs/customization/)：模板和生成定制；使用方式依工具版本。
