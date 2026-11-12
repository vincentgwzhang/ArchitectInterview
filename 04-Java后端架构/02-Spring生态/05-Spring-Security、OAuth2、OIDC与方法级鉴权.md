# 05. Spring Security、OAuth 2.0、OIDC 与方法级鉴权

> 适用基线：Java 17+、Spring Boot 3.5.x、Spring Security 6.5.x，Servlet 应用。WebFlux 使用响应式安全 API，不能直接复制本章 SecurityFilterChain 配置。协议部分依据 OAuth 2.0、PKCE、OIDC Core 与 RFC 9700 安全最佳实践。所有组织、域名、权限和场景均为教学示例，不代表读者真实项目。

## 1. 从业务目标出发：身份、委托与资源政策分别回答什么

认证（Authentication）回答“你是谁、用什么证据证明”；授权（Authorization）回答“这个主体能否对这个资源执行这个动作”。用户成功登录只证明身份成立，不证明能读任意订单。React 隐藏按钮只能改善界面，攻击者仍可直接发送 HTTP 请求，后端必须强制执行规则。

贯穿例子是多租户订单服务：租户 A 的普通用户只能读自己的订单，租户管理员可读本租户订单，财务角色只能执行允许的财务动作。这里至少需要可信主体、租户归属、动作权限、订单所有者和当前业务状态。用一个 ADMIN 角色替代全部判断，会让跨租户和越业务边界的权限混在一起。

OAuth 2.0 是委托授权框架，不是统一“登录协议”，也不要求 Access Token 必须是 JWT。OIDC（OpenID Connect）在 OAuth 上增加身份层，使客户端按标准验证最终用户认证结果。Spring Security 实现其中的认证和授权机制，业务仍需定义“谁拥有订单”“离职后多久失效”等政策。

架构设计先确定信任边界：浏览器属于不可信客户端，身份提供方负责认证与发令牌，API 负责验证面向自己的令牌，业务服务负责对象政策。网关检查可以提前挡住无效请求，但内部 API 若可被绕过、消息消费者或定时任务直接调用服务，就不能依赖唯一入口永远安全。

## 2. OAuth 角色、令牌与完整授权码流程

### 2.1 四个角色与三类令牌

资源所有者（Resource Owner）通常是最终用户；客户端（Client）代表用户请求访问；授权服务器（Authorization Server）认证用户/客户端并发放令牌；资源服务器（Resource Server）是验证令牌并提供 API 的服务。同一个产品可以承担多个角色，但它们的逻辑责任仍需分开。

Access Token 用于向资源服务器出示访问授权；Refresh Token 用于在授权服务器换新 Access Token，不应发给业务 API；ID Token 用于 OIDC 客户端理解认证结果，面向该客户端而不是通用 API。Bearer Token 表示持有者可使用，泄露后攻击者可能无需再次证明身份即可重放，因此 TLS、存储和日志控制很重要。

Scope 是授权范围，例如 order:read。它通常表达“这类能力获准使用”，不必包含每个订单的归属规则。主体标识 sub 也不能脱离发行方解释：多身份提供方场景中应考虑 issuer 与 subject 的组合，不能把不同 issuer 的相同 sub 当作同一人。

### 2.2 Authorization Code + PKCE 的实际步骤

教学场景：浏览器访问订单网站，网站需要让用户通过企业身份平台登录。推荐使用成熟客户端库实现流程，不手写状态校验。

1. 客户端为这次登录创建高熵一次性 code_verifier，并计算 S256 code_challenge；同时建立与浏览器登录事务关联的 state，OIDC 场景可创建 nonce。
2. 浏览器跳转授权端点，携带 client_id、准确注册的 redirect_uri、response_type=code、scope、challenge 及相应状态参数。用户在身份平台完成认证，可能还需同意委托。
3. 身份平台把一次性授权码返回预先允许的回调。客户端验证这是否属于自己发起的登录事务，而不是接受任意浏览器提交的 code。
4. 客户端向令牌端点提交 code、原始 verifier 与正确回调信息；保密客户端还按约定证明自身身份。授权服务器验证 verifier 与原 challenge 的关系，发放令牌。
5. OIDC 客户端验证 ID Token 及与登录事务关联的数据，建立自己的登录状态。调用 API 时使用面向该 API 的 Access Token，API 独立验证并执行业务授权。

PKCE 防护的重点是授权码被截获后，仅拿到 code 的一方缺少 verifier，不能成功兑换。它不加密 Access Token，也不能防止浏览器已被 XSS 完全控制后的所有操作。state 用于关联请求/响应、防止请求伪造等风险，nonce 将 OIDC 认证结果绑定到预期事务；成熟库可能组合使用标准保护，不能把它们混成“随便有一个随机数就行”。

redirect_uri 必须按注册策略严格匹配，不接受任意回跳地址；否则 code 或结果可能被送往攻击者控制处。SPA 和原生应用属于难以保密的 public client，打包进 JavaScript 的 client_secret 不是秘密。BFF 等服务器端客户端可保护凭证，但仍需按最佳实践使用 PKCE 等保护，而不是认为有 secret 就没有重定向风险。

### 2.3 哪些流程不应继续用，机器身份怎样选

新设计避免 Implicit Flow 在前通道直接交付令牌带来的暴露风险；不要采用让应用收集用户密码再换令牌的 Resource Owner Password Credentials 模式。RFC 9700 已明确收紧这些旧方式。迁移应盘点现有客户端、回调与令牌受众，不能只改授权类型而漏掉存储和验证。

Client Credentials 适合服务以自身身份访问服务，不代表某个用户。若后台作业调用订单管理 API，审计应记录机器主体，不能虚构一个用户身份。不同服务不应共享一个万能 client secret，否则一个服务泄露可冒充全部调用者。

Workload Identity（工作负载身份）通过平台证明运行实例身份，换取短期凭证或令牌，减少静态 secret 分发。它仍需信任平台证明链、限制 audience/权限并管理过期；不是“部署到集群就自动授权”。需要代表用户跨服务调用时，应采用明确的委托/令牌交换等受支持协议，不能把任何收到的 Token 转发到所有下游。

## 3. Token 验证：解码、验签与授权是三层

### 3.1 JWT 可读不等于可信，也通常不保密

常见签名 JWT 由 header、payload、signature 构成，前两部分通常只是 Base64URL 编码。任何拿到它的人都可能读出 claims，因此不要存放密码、医疗细节或不必要个人数据。签名保护完整性和来源，不自动提供加密。

资源服务器验证时应明确接受的发行方、算法和密钥来源，验证签名、受众 audience、有效期 exp、尚未生效时间 nbf 等，再映射权限。需要容忍有限时钟偏差，但不能用巨大容忍窗口掩盖时钟错误。某个令牌即使签名正确，也可能是发给别的 API 的，必须拒绝。

kid 只是用于选择受信任密钥集合中某把钥匙的标识，不是让应用按令牌头里的任意 URL 下载密钥。不要接受未受信任 issuer 自动变成新租户，也不要让算法由外部 Token 任意决定。多租户发行方需要允许列表与明确验证器，防止扩大信任边界。

### 3.2 ID Token、Scope、Role 与本地权限不可混用

ID Token 的 audience 通常是登录客户端，API 不应用它充当 Access Token。它证明的认证事件与业务访问授权不同；即使都由同一平台签发、都是 JWT，也不能因为验签成功就互换。

Spring Resource Server 常把 scope/scp 映射成带 SCOPE_ 前缀的 authority。例如 scope=order:read 常得到 SCOPE_order:read；hasAuthority 做精确匹配，hasRole 默认则应用 ROLE_ 前缀约定。若配置 hasAuthority('order:read')，却没有自定义映射，可能永远不匹配。反过来，随意把全部 claim 映射为管理员角色会过度授权。

Role 是组织角色，Scope 是委托范围，资源政策是对象关系，三者可以组合但不相等。即使已有 SCOPE_order:read，也必须检查订单是否属于当前用户/租户；令牌可能只承诺具备访问订单 API 的资格，不承诺所有订单可读。

### 3.3 JWT 与 Opaque Token 的取舍

| 方式 | 验证路径 | 好处 | 主要成本 |
|---|---|---|---|
| JWT 本地验证 | 用缓存的受信任公钥和验证规则检查 | 请求路径减少中心查询，便于扩展 | 权限与撤销状态可能滞后，密钥缓存需治理 |
| Opaque + Introspection | API 向授权服务查询 active 与授权信息 | 便于集中控制状态 | 网络延迟、中心容量与故障依赖 |
| JWT 加在线策略/撤销检查 | 本地验证后追加状态判断 | 可缩短高风险变更窗口 | 又引入在线依赖、缓存和一致性成本 |

Opaque 表示调用方不应理解内部结构，不意味着服务端绝对不能用 JWT 实现其内部令牌。Introspection 是在线询问令牌是否仍有效的标准接口；它本身也要认证调用者，避免泄露令牌状态。

“JWT 不能撤销”过于绝对：可以通过短有效期、拒绝列表、主体版本或在线策略补充撤销。但每种都增加状态和成本。删除浏览器 Token 或撤销 Refresh Token 不会自动令所有已签发 Access Token 立即无效。Introspection 若缓存 5 分钟，也有最多相应滞后窗口，不能声称天然实时。

选择应从撤权目标推导：普通读取容忍几分钟延迟，短期 JWT 可能合理；紧急禁用高风险转账权限要求更快失效，则需在线状态或更强策略，并说明中心故障时拒绝还是保留哪类缓存授权。高风险操作通常不能因验证服务故障就直接放行。

## 4. Spring Security 的请求链与最小 API 配置

### 4.1 请求怎样变成可信 Authentication

Servlet 容器通过过滤器接入 Spring，FilterChainProxy 按匹配规则选择 SecurityFilterChain。链中认证过滤器提取凭证并交给相应认证机制；成功后创建 Authentication 并放入 SecurityContext，后续授权使用其中主体与 authorities。SecurityContext 不是客户端可以自行声明的“已登录”字符串。

Bearer 认证可由 JWT 解码验证器或 Opaque introspector 完成。若认证失败，应以合适的认证入口处理；已认证但缺少权限，由拒绝处理器处理。对 API 一般表现为 401 与 403，浏览器登录路径则可能重定向登录页。不要把 302 HTML 登录页面当作机器 API 的正常失败契约。

多条链按顺序选第一条匹配链，而不是把所有链的规则自动叠加。链级 securityMatcher 决定是否使用这条链，链内 requestMatchers 决定其中授权规则；必须有覆盖其余请求的策略，避免遗漏。自定义 Filter 还需防止被容器和安全链重复注册。

### 4.2 一个明确限定的 Bearer-only API 示例

下面是 Boot 3.5 / Security 6.5 Servlet 应用示例，依赖 spring-boot-starter-oauth2-resource-server 和 Web Starter。假设本应用只提供显式 Authorization: Bearer 认证的 API，不接受浏览器自动附带 Cookie/Session/Basic 等替代认证；此条件是关闭 CSRF 的前提。BFF/Session 应用不能照抄。

~~~java
package example.security;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.HttpMethod;
import org.springframework.security.config.Customizer;
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.web.SecurityFilterChain;

@Configuration(proxyBeanMethods = false)
@EnableMethodSecurity
public class ApiSecurity {
    @Bean
    SecurityFilterChain api(HttpSecurity http) throws Exception {
        return http
            .sessionManagement(s -> s.sessionCreationPolicy(
                SessionCreationPolicy.STATELESS))
            .csrf(csrf -> csrf.disable())
            .authorizeHttpRequests(r -> r
                .requestMatchers(HttpMethod.GET, "/public/status").permitAll()
                .requestMatchers("/api/orders/**").authenticated()
                .anyRequest().denyAll())
            .oauth2ResourceServer(o -> o.jwt(Customizer.withDefaults()))
            .build();
    }
}
~~~

~~~properties
spring.security.oauth2.resourceserver.jwt.issuer-uri=https://id.example.invalid
spring.security.oauth2.resourceserver.jwt.audiences=orders-api
~~~

域名为占位值，必须替换为受信任身份平台及其准确发行方标识；依赖平台的发现文档、公钥发布、TLS 和时钟等配置。audiences 显式要求 orders-api，不能因只配置 issuer 就假设 audience 已符合业务要求。自定义 JwtDecoder 时应重新确认默认时间/发行方校验与自定义 audience 校验组合完整，避免覆盖掉框架默认验证。

此例把所有请求都纳入一条链，只有健康状态路径匿名开放，订单路径先要求认证，其余默认拒绝；对象权限在后面的服务层落实。STATELESS 表示不以 Session 保存此认证流程，不是“服务内部完全没有状态”，也不证明其他自定义过滤器不会读取 Cookie。

对 /public/status 携带无效 Bearer Token，认证过滤器仍可能先拒绝；permitAll 只放开授权规则，不等于跳过全部认证过滤器。不要为匿名端点测试只看“没 Token 成功”，也应明确带坏凭证时预期。

CORS（跨来源资源共享）控制浏览器是否允许跨来源脚本读取/发送某些请求，不是服务器认证机制。若前端跨域，应按精确允许来源和预检需求配置，不能把 allow-all CORS 当成登录修复。后端工具和恶意服务器不受浏览器 CORS 限制，仍需认证与对象授权。

## 5. 方法级鉴权与防 IDOR：权限必须落到这一条数据

### 5.1 原注解的完整语义

原有规则“hasAuthority('order:read') and @access.canRead(#orderId)”只有在 authority 映射、access Bean、参数名发现和方法安全均正确时才成立。@EnableMethodSecurity 需要显式启用；仅引入 Security Starter 不保证它已经启用。

IDOR（不安全的直接对象引用）是攻击者替换订单 ID 等引用，访问无权访问的对象。UUID 难猜可降低枚举效率，但知道别人的 UUID 后仍不能访问，真正保护是授权查询。

下面以 JWT 已通过请求链验证为前提，展示“动作权限 + 数据范围”实现。代码按类拆到 example.order 包；权限注解使用 SCOPE_ 约定。为避免编译器参数名影响 SpEL，不依赖 #orderId 表达式，直接在查询时绑定可信身份。

~~~java
package example.order;

import java.util.Optional;
public interface OrderReader {
    Optional<OrderView> findVisible(
        String issuer, String tenant, String subject, String orderId);
}
~~~

~~~java
package example.order;

public record OrderView(String id, String state) {}
~~~

~~~java
package example.order;

import java.util.Objects;
import org.springframework.security.access.AccessDeniedException;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.oauth2.jwt.Jwt;
import org.springframework.stereotype.Service;

@Service
public class OrderQueryService {
    private final OrderReader orders;

    public OrderQueryService(OrderReader orders) {
        this.orders = orders;
    }

    @PreAuthorize("hasAuthority('SCOPE_order:read')")
    public OrderView read(String orderId) {
        Authentication auth = SecurityContextHolder.getContext()
            .getAuthentication();
        if (auth == null || !(auth.getPrincipal() instanceof Jwt jwt)) {
            throw new AccessDeniedException("unsupported principal");
        }
        String tenant = jwt.getClaimAsString("tenant_id");
        String subject = jwt.getSubject();
        if (tenant == null || tenant.isBlank()
                || subject == null || subject.isBlank()
                || jwt.getIssuer() == null) {
            throw new AccessDeniedException("identity context missing");
        }
        Objects.requireNonNull(orderId, "orderId");
        return orders.findVisible(jwt.getIssuer().toString(),
                tenant, subject, orderId)
            .orElseThrow(() -> new AccessDeniedException("order not accessible"));
    }
}
~~~

OrderReader 是必须实现的持久化契约，不是假装已有数据库实现。其查询必须同时约束订单 ID、可信发行方/租户和所有者映射，例如把 tenant_id 与 owner_subject 条件放入 WHERE，不能先按 ID 取任意订单再忽略这些参数。数据库 tenant/owner 数据与令牌标识的映射须有可信创建/同步流程；签名通过也不表示任意 issuer 自定义的 tenant_id 都是本系统授权命名空间。

代码从已认证 SecurityContext 获取主体，不让调用方通过方法参数任意冒充 Jwt。缺失租户/主体拒绝，不默认给一个“公共租户”。读不到与无权读使用统一外部行为可以减少存在性泄露；生产可选择经设计的 403 或隐藏存在性的 404，需统一映射，不把此处异常消息原样当公众诊断。

若使用 @access.canRead，必须让它真正查询归属、租户、授权记录；写操作还需处理检查与修改之间的竞态。例如检查后订单已被转移给别人，后续更新不能仍按过时判断写入。可在同事务通过条件更新、合适锁/版本和不变量约束把授权条件与动作协调，必要时重新检查。单独一个布尔权限函数不提供数据库原子性。

### 5.2 代理绕过、批量操作与默认拒绝

方法安全通常通过 Spring AOP 代理执行。同类内部 this.read、直接 new 对象、未受管理入口可能绕开拦截。调用方应经受保护服务 Bean，关键数据条件也在持久化契约中约束；不能仅靠 Controller 没暴露那个方法就认定没有绕路。

批量查询、导出、搜索、关联加载与计数接口都要按租户/权限范围执行。只保护 GET /orders/{id} 却让导出接口输出全部订单，等于侧门仍开着。先读所有数据再 @PostFilter 过滤，可能泄露计数、消耗大量资源并破坏分页；优先数据库限定范围。

跨租户管理员不应因 role 名称含 admin 就绕过所有租户。必须显式定义其可管理范围、是否需要提升认证强度、哪些动作记录审计。服务账号、最终用户和紧急账号应分开策略，权限政策应能解释每次允许的原因。

## 6. Session、CSRF、XSS 与 BFF 的实际取舍

### 6.1 CSRF 利用的是浏览器自动携带凭证

Cookie Session 模式下，浏览器会按 Cookie 规则自动附带会话凭证。攻击者可诱导浏览器向受害网站发起改变状态的请求，即使攻击者读不到响应也可能造成修改，这就是 CSRF 的关键。GET 应保持安全语义，不用 GET 执行删除或转账；不安全方法应验证与会话/请求关联的 CSRF 保护，并配合来源检查等措施。

SameSite 限制某些跨站 Cookie 发送，但跨站不等于跨来源，业务跳转和子域等也有边界，不能只设置一个属性就删掉 CSRF 设计。HttpOnly 防脚本直接读 Cookie，Secure 限制 HTTPS 传输；它们不阻止恶意脚本在同站上下文发出合法认证请求。

显式 Bearer Header-only API 因浏览器不会自动替攻击页面填该应用持有的 Authorization 值，经典 CSRF 风险不同。但如果 Bearer 放进自动发送的 Cookie，或仍接受 Session/Basic 认证，就必须重新分析。STATELESS 和 JSON 请求体都不是自动免疫证明。

### 6.2 XSS 与 Token 存储没有万能解

XSS（跨站脚本）让攻击代码在站点权限中运行。localStorage 中 Token 可被脚本读走，长期保存尤其扩大泄露窗口；内存保存降低持久暴露，但页面刷新、多个标签和刷新令牌管理更复杂，活动 XSS 仍可调用 API。

BFF（Backend for Frontend）让服务器持有 OAuth Token，浏览器只持不透明会话 Cookie。它减少 Token 暴露给 JavaScript 的机会，代价是服务器会话存储、横向扩展、CSRF 防护和代理容量。BFF 必须限制代理目标和可调用操作，不能成为拿着用户 Token 的任意 URL 转发器。

不论选哪种，都需要输出编码、限制危险 HTML、CSP 等减少脚本注入途径，保护依赖链和登录回调。HttpOnly 不是 XSS 修复工具。选择依据客户端类型、撤销目标、团队运维能力和风险，而不是声称某个存储方式绝对安全。

### 6.3 登录、刷新与退出的失败语义

登录成功后需要会话固定攻击防护，避免攻击者预设会话 ID 被沿用为已认证会话；优先使用框架受支持的 Session 管理。退出应使本地会话无效并清 Cookie，还要定义是否撤销 Refresh Token、是否结束身份平台 SSO 会话。三者不是同一动作。

Refresh Token Rotation 在刷新时替换令牌，配合重用检测识别被盗凭证；并发刷新可能让两个合法请求互相竞争，所以客户端/BFF 应协调刷新，服务端按平台明确的重用策略处理。不能因刷新失败就无限重试并扩大攻击或登录风暴。

Access Token 过期可按策略刷新后有限重试，403 通常不应通过反复登录修复权限不足。对于写请求，自动重放还要有幂等协议，不能让刷新后的重试重复执行支付。

## 7. 运营、故障与密钥轮换

### 7.1 最小权限、短期凭证与撤权目标

权限先按工作任务分配，避免全服务共用万能 scope。短期凭证减少长期泄露影响，但过短会增加刷新、时钟和身份平台依赖，应衡量安全窗口与可用性。业务权限变化若只存在 Token 中，会直到过期或在线状态检查才体现，离职停权时必须明确这段延迟。

Workload Identity、密钥托管和自动轮换减少静态凭证复制。权限变更要有 Owner、审批、期限和回收机制；临时访问到期自动收回比依赖人记住更可靠。不要为便于排障授予所有工程师长期生产管理员。

### 7.2 正常轮换与密钥泄露是不同事件

正常公钥轮换通常先发布新公钥，再用新私钥签发，保留旧公钥覆盖仍有效旧 Token 与合理传播延迟，最后移除旧钥。资源服务器需要受控缓存与刷新，未知 kid 不应触发无上限网络请求，否则攻击者可放大发现/公钥服务负载。

私钥泄露时，继续接受旧钥意味着攻击者可能继续伪造。需按应急流程停止信任泄露钥、处理缓存、撤销受影响凭证并准备重新认证，不能为避免少量 401 一直保留旧钥。安全与可用性的冲突要事前演练。

身份平台不可用时，已有缓存公钥可能允许部分 JWT 继续本地验证；新密钥、首次发现或在线 introspection 仍可能失败。不能假设“JWT 永远不依赖身份平台”，也不能在公钥获取失败后跳过验签。对于未能验证的凭证，应拒绝并返回受控错误，避免泄露内部连接细节。

### 7.3 审计、指标与 Break-glass

审计记录主体标识、可信租户、动作、目标、允许/拒绝、政策版本、时间和关联 ID，保护写入完整性并限制查看权限。不要记录完整 Token、Authorization Header、Refresh Token、密码或客户端 secret；排障使用不泄密的标识和受控采样。用户 ID/订单号也不应作为无限基数监控标签。

监控区分签名失败、过期、错误 audience、缺 scope、对象政策拒绝、身份平台延迟和密钥刷新失败；公众响应保持克制，内部诊断保留原因。大量 403 可能是正常缺权、配置错误或枚举攻击，需要结合行为和部署变更，不立即把全部失败当攻击。

Break-glass 是紧急情况下受控提升权限的通道，应独立身份、强认证、短时授权、范围限制、即时告警与事后复核。它不是藏一个永久超级密码。紧急恢复后必须撤销提升并检查审计，避免应急路径成为长期后门。

## 8. 事故推演：验签成功，为何仍然泄露订单

教学场景：API 接受企业发行方签名的所有 JWT，只判断 ROLE_USER。攻击者拿到面向报表服务的 Token，并替换订单 ID，成功读到其他租户订单。

这里至少有两处独立缺口：缺 audience 使发给别的资源的授权被错误使用；缺订单/租户政策使一般用户能读任意对象。仅换签名密钥不能修复它们。立即限制受影响入口与权限路径，修复受众验证和带数据范围的查询，再核查访问记录确定影响范围。

回归不能只测“正常用户登录成功”。应准备相同 issuer、不同 audience 的有效签名 Token，确认拒绝；准备同 audience 与 scope、不同租户/所有者，确认不能读取；检查批量、导出、缓存键和内部服务入口。缓存也必须包含授权范围或采用安全的数据缓存策略，不能把 A 的响应按订单 ID 缓存后直接发给 B。

如果权限先允许、随后账号被禁用，本地 JWT 仍通过，则是撤权窗口设计问题，不应谎称验签故障。应按风险缩短有效期或引入在线状态，明确缓存失效和故障策略。事故复盘要把认证、令牌有效性和业务授权三层分别说明。

## 9. 面试完整回答与追问

### 9.1 “OAuth2 与 OIDC 有什么区别？”

可表达回答：“OAuth2 让客户端获得访问资源的委托授权，Access Token 面向资源服务器。OIDC 在其上增加最终用户身份层和 ID Token，客户端按协议验证认证结果。我会按角色区分客户端、授权服务器和 API，Web 登录采用授权码与 PKCE，API 验证自己的 audience 和 scope，再做对象授权。不能把登录 ID Token 当作 API 通行证。”

追问：“有 PKCE 还需要 secret 吗？”答案：public client 不能有效保密 secret；保密客户端仍按约定认证自己。PKCE 绑定这次授权码与 verifier，客户端认证证明客户端身份，防护目标不同，不能简单互相替代。

### 9.2 “JWT 验签成功还要检查什么？”

可表达回答：“验签只证明令牌未被未授权篡改且来自对应密钥。还要限制发行方、受众、算法、有效期和使用类型，映射 scope/authority，并执行资源归属与租户政策。公钥必须来自受信任配置，不能跟着 Token 中任意地址取。撤销与权限变化另有窗口，需要在安全目标中明确。”

追问：“为什么不全部 introspection？”答案：它提供集中状态检查，但增加延迟、身份服务容量和故障依赖，缓存又带来滞后。可按操作风险选择，不是绝对优于本地验证。

### 9.3 “怎样避免 IDOR？”

可表达回答：“我把动作授权和对象范围结合，身份与租户从可信认证上下文获取，查询/更新按资源 ID、租户和主体范围约束；角色管理员也只在明确范围内生效。方法安全保护服务入口，数据访问保护批量、导出和绕路场景。写操作还要协调授权检查与更新之间的竞态，测试同租户他人和跨租户访问。”

追问：“把 ID 换成 UUID 就安全吗？”答案：只能降低猜测效率，泄露链接、日志或正常分享后 ID 仍可获得。授权不应依赖资源引用不可知。

### 9.4 “BFF 是否解决所有前端安全问题？”

可表达回答：“BFF 减少 OAuth Token 暴露给脚本，但浏览器仍持会话 Cookie，需要 CSRF、Session 生命周期和严格代理目标。XSS 仍可能以用户身份发动作，所以还需输出处理、CSP 与依赖治理。它是在客户端暴露面与服务器状态成本间做取舍，不是安全免责层。”

追问：“REST API 为什么会有 CSRF？”答案：是否 REST 与是否自动携带凭证不同。Cookie/Session/某些浏览器认证即使不保存应用业务状态，也可能被跨站请求利用；要分析真实凭证传输方式。

## 10. 扫雷自测与完整推理答案

### 10.1 Token 能 Base64 解出用户名，是否证明有效？

不能。解码不验证签名、受众、发行方和时效，更不验证访问哪张订单。未经验证 claims 只能当不可信输入，不能用于选择任意密钥源或建立主体。

### 10.2 ID Token 与 Access Token 都有企业签名，能互换吗？

不能。它们的接收方、语义和验证规则不同。API 必须要求面向自身的 Access Token，登录客户端验证 ID Token；同样的签名只解决部分来源问题。

### 10.3 hasAuthority('order:read') 为什么拿着正确 scope 仍拒绝？

Spring 默认转换常生成 SCOPE_order:read，应核对 authority 映射，避免混淆 ROLE_ 与 SCOPE_。修正表达式或明确配置转换器，不能用放开全部路径掩盖映射错误。

### 10.4 关闭 Session 是否足以关闭 CSRF？

不是。必须确认端点不会接受浏览器自动携带的凭证，且没有 Cookie Bearer/Basic 等其他认证路径。本章关闭示例仅适用于限定的 Bearer Header-only API；BFF 与 Session 路径保持相应防护。

### 10.5 租户 ID 从 Header 取，与 Token 中相同就安全吗？

还要验证 Token 来源与该 claim 的可信契约，并验证主体确实属于该租户、资源也在该租户。Header 可作为用户选择，但不能单独授予身份；签名方也可能不具备任意本系统租户的授权权力。

### 10.6 在服务方法上写 PreAuthorize 为什么仍可能被绕过？

可能没启用方法安全、对象未由 Spring 管理或同类内部调用绕过代理。测试必须从实际 Bean 入口调用；关键查询和更新也要带可信数据范围，减少遗漏的破坏面。

### 10.7 删除 Refresh Token 后为何旧 Access Token 还能用？

已签发令牌若本地验证，只要签名、时效等仍有效便可能继续被接受。撤销 Refresh Token 主要阻止以后刷新；立即撤权需要额外状态机制或更短有效期，并处理缓存窗口。

### 10.8 身份平台宕机时可以临时忽略签名吗？

不能。这样把无法验证变成任意人可冒充。可以按事先政策使用仍可信的缓存公钥验证既有 JWT，但缺少必要证据时拒绝，并通过隔离、容量和受控错误提升可用性。

### 10.9 允许用户读订单后，后台转移了所有者，写操作还能直接继续吗？

不能依赖旧判断。授权与操作之间可能存在竞态；写入条件需与可信租户、所有权/版本及状态协调，必要时在事务内重新验证。一次布尔检查不能保证后续数据库状态未变化。

## 11. 验证矩阵与本次执行范围

验证分层进行。令牌层测试缺凭证、坏签名、未知/错误发行方、错误 audience、过期、尚未生效、缺 scope 和有效凭证；权限层测试同租户自己、同租户他人、跨租户、缺 tenant claim、管理员有限范围和机器主体。对拒绝场景同时断言响应体不包含目标数据及敏感信息，而不只检查状态码。

Spring 的 mock JWT 测试可验证 authority 与方法政策，但常会绕过真实密码学解码，不能证明签名验证器正确。需另用受控测试密钥和真实解码器测试签名、算法、时间与 audience，并验证应用自定义 Decoder 没覆盖默认校验。不要用真实生产 Token 做测试夹具。

浏览器路径测试 OAuth 回调状态、错误 verifier/nonce、严格 redirect_uri、CSRF 缺失/正确、Session 退出与并发刷新；API 路径测试不意外接受 Cookie 认证。覆盖所有安全链与 catch-all，测试 public 端点带无效 Token 的预期，检查导出/搜索/缓存/内部入口。

运营演练正常轮换、公钥服务不可用、未知 kid 洪峰、权限撤销和紧急提升到期；验证故障时不误放行、审计无 Token、诊断指标能定位阶段。本章 Java 配置和服务代码**未实际编译或运行**，未执行真实身份平台与浏览器测试；已人工核对原主题覆盖、代码和权限约定、失败语义、Markdown 结构与官方版本资料。OrderReader 还需按项目实现与集成验证，示例不是可直接部署的完整身份系统。

## 12. 面试前速记

- 认证证明主体，OAuth 委托能力，OIDC 表达认证结果，对象政策决定这一条数据能否访问。
- Access Token 给 API，ID Token 给客户端，Refresh Token 给授权服务器；不能因都是 JWT 而互换。
- PKCE 绑定授权码与 verifier，不代替全部回调、客户端和浏览器保护。
- JWT 解码不等于验签，验签不等于受众正确，scope 不等于对象归属。
- 默认 authority 前缀要一致；方法安全需启用且经过代理，查询/更新要限制可信租户与主体。
- Cookie Session 与 BFF 需要 CSRF 设计，HttpOnly 不修复 XSS，CORS 不等于授权。
- JWT 本地验证与在线检查的取舍是延迟、可用性、撤销窗口和治理成本。
- 正常轮换保留兼容窗口，泄露密钥要应急撤信任；不得以可用性为由跳过验证。
- 所有拒绝路径、批量出口与内部调用都要测，审计永不输出 Token。

## 13. 一手来源与版本范围

来源用于核验与可选追溯，正文已解释必要概念与推理。Spring 示例固定在 Security 6.5 / Boot 3.5 系列，升级时重新验证默认行为与属性。

- [Spring Security 6.5：Servlet Architecture](https://docs.spring.io/spring-security/reference/6.5/servlet/architecture.html)
- [Spring Security 6.5：JWT Resource Server](https://docs.spring.io/spring-security/reference/6.5/servlet/oauth2/resource-server/jwt.html)
- [Spring Security 6.5：Method Security](https://docs.spring.io/spring-security/reference/6.5/servlet/authorization/method-security.html)
- [Spring Security 6.5：CSRF](https://docs.spring.io/spring-security/reference/6.5/servlet/exploits/csrf.html)
- [RFC 9700：OAuth 2.0 Security Best Current Practice，2025-01](https://www.rfc-editor.org/rfc/rfc9700.html)
- [RFC 7636：PKCE](https://www.rfc-editor.org/rfc/rfc7636.html)
- [RFC 7662：Token Introspection](https://www.rfc-editor.org/rfc/rfc7662.html)
- [OpenID Connect Core 1.0，Errata Set 2](https://openid.net/specs/openid-connect-core-1_0.html)
