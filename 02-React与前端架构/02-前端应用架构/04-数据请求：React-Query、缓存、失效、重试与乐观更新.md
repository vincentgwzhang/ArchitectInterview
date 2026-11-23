# 04. 数据请求：React Query、缓存、失效、重试与乐观更新

> React Query现称TanStack Query。本文以React18+与TanStack Query v5对象参数API为范围，2026-11-23核验官方v5文档。服务器是持久事实的权威，查询缓存是带身份、新鲜度和生命周期的客户端快照；它不是数据库事务或授权系统。

## 一、先建立请求与缓存的心智模型

Query（查询）描述某个身份的数据读取；QueryClient持有缓存并协调订阅、请求与更新；组件观察结果，不应各自再复制到Redux形成双权威。Mutation（写操作）表示外部状态改变，需要明确成功、失败、结果不确定以及对缓存的影响。

组件挂载不一定需要网络：可复用新鲜缓存；组件卸载也不等于服务器记录消失。相同QueryClient下相同key可共享结果和进行中的读取；这不是任意相似URL永久去重，也不保证不同标签页、SSR请求或不同Client实例共享。

简单独立请求可手写Effect，但随着缓存、分页、取消、重试和错误恢复组合，重复实现的维护成本会上升。使用库能统一机制，业务仍需决定允许多旧、何时失效、哪些失败可重试与哪些操作能乐观展示。

## 二、Query Key：缓存身份是正确性边界

Key应为稳定、可序列化的数组，包含影响结果的参数。不是只要URL相同就代表相同结果：租户、用户可见范围、币种、过滤、排序、分页都可能改变语义。令牌不宜直接放key，会进入调试/日志；使用非秘密作用域标识并在身份改变时清除旧缓存，服务器仍重新授权。

~~~javascript
export const orderKeys = {
  root: tenantId => ['orders', tenantId],
  lists: tenantId => ['orders', tenantId, 'list'],
  list: (tenantId, filters) => ['orders', tenantId, 'list', filters],
  detail: (tenantId, id) => ['orders', tenantId, 'detail', id]
};

export function normalizeFilters({ status, page }) {
  return {
    status: status === 'paid' ? 'paid' : 'all',
    page: Number.isInteger(page) && page > 0 ? page : 1
  };
}
~~~

Factory统一key层级，便于按域失效；filters先规范化，避免同义输入产生无意义缓存碎片。此例只支持paid/all，真实应用需显式允许集合，不能把未知关键业务筛选悄悄变all导致用户误解。

v5默认key哈希会稳定处理对象属性顺序，而数组元素位置有意义。不要依赖函数、DOM对象或可变非序列化结构作为身份；查询参数变化时应改变key，而非只修改queryFn闭包导致同key缓存装不同内容。分页普通查询与无限查询数据形状不同，应使用不同key命名，不能混用。

## 三、Freshness、GC与刷新是三条独立时间线

staleTime是从数据更新时间起判断“新鲜多久”；gcTime是无活跃观察者后保留多久。Fresh不等于服务器绝不会改变，只是客户端策略允许暂时不因普通staleness触发刷新。Inactive缓存可留待返回复用，GC回收不是删除服务器记录。

v5普通默认读取很快被视作stale，无活跃查询通常5分钟后回收；挂载、聚焦和网络恢复等条件可触发后台刷新。stale本身不表示马上清空，也不是到了某秒就自动发网络定时器；周期轮询由refetchInterval等另配置。

例：数据t=0成功，staleTime=30秒，t=10卸载，gcTime=5分钟。t=20返回可能直接读新鲜缓存；t=40返回可先展示旧缓存并后台刷新；若持续无订阅直到回收后再进，则没有旧结果可用。主动失效可提前改变新鲜策略，不必等30秒。

Infinity不是一律错误：稳定字典或由可靠事件失效的数据可用，前提是有更新协议。权限和库存往往随外部主体变化，不能仅为省请求永久新鲜。当前v5还存在更严格的static选项，其失效行为不同，本文示例不使用，勿和Infinity互换。

过短新鲜度增加请求、能耗与后端压力，过长则旧数据损害正确性。先设业务允许旧窗口，再估活跃用户、聚焦与重连触发频率，测端到端行为；gcTime更长主要扩大缓存内存，不自动延长fresh。

## 四、一个明确处理HTTP、取消与错误的查询

~~~jsx
import { useQuery } from '@tanstack/react-query';

class ApiError extends Error {
  constructor(status) {
    super('HTTP ' + status);
    this.status = status;
  }
}

export function Orders({ tenantId, customerId, status, page }) {
  const filters = { customerId, status, page };
  const query = useQuery({
    queryKey: ['orders', tenantId, 'list', filters],
    enabled: Boolean(tenantId && customerId),
    staleTime: 30_000,
    retry: (failureCount, error) =>
      failureCount < 2 &&
      (error instanceof ApiError
        ? [502, 503, 504].includes(error.status)
        : error instanceof TypeError),
    retryDelay: attempt => Math.min(1000 * 2 ** attempt, 8000),
    queryFn: async ({ signal }) => {
      const params = new URLSearchParams({
        customerId, status, page: String(page)
      });
      const response = await fetch(
        '/api/tenants/' + encodeURIComponent(tenantId) +
        '/orders?' + params, { signal }
      );
      if (!response.ok) throw new ApiError(response.status);
      const data = await response.json();
      if (!Array.isArray(data.items) ||
          !data.items.every(x => x && typeof x.id === 'string' &&
            typeof x.label === 'string')) {
        throw new Error('Invalid orders response');
      }
      return data.items;
    }
  });

  if (!tenantId || !customerId) return <p>请选择客户</p>;
  if (query.isPending) return <p role="status">加载中</p>;
  if (query.isError && !query.data) return <section role="alert">
    <p>订单暂时无法加载</p>
    <button onClick={() => query.refetch()}>重试</button>
  </section>;
  return <section aria-busy={query.isFetching}>
    {query.isError && <p role="status">更新失败，当前显示已有快照</p>}
    {query.data.length === 0 ? <p>暂无符合条件的订单</p> :
      <ul>{query.data.map(x => <li key={x.id}>{x.label}</li>)}</ul>}
  </section>;
}
~~~

需在上层用稳定QueryClientProvider装配；浏览器应用可在适当作用域保持一个Client，不能每次render创建。参数约定为已规范化字符串与正整数，响应id需业务保证唯一。示例未处理完整分页元数据和按钮重复点击节流，生产应按需求补齐。

fetch收到HTTP失败也可能resolve，故必须检查ok并throw。普通结构错误不重试；TypeError仅作为简化网络分类，现实中也可能来自配置/编程问题，应由请求适配层分类，避免把所有TypeError无限当瞬时故障。指数等待示例没有jitter，生产多客户端需加入抖动并受总预算约束。

初次无数据、空数据与后台刷新失败分开呈现。query.isFetching包含后台活动，isPending表达无成功数据的状态；disabled查询不能仅凭pending一直转圈，所以先展示“请选择客户”。对不允许旧的金额与权限，即使有data也应限制依赖新鲜性的动作，不能照搬此快照展示策略。

## 五、失效：把业务事件映射为查询范围

Mutation成功后有三种策略：用可信服务器返回值精确更新详情；对相关query失效重取；或将返回的完整对象替换相应缓存。setQueryData需不可变更新，不能改getQueryData拿到的对象，否则订阅与快照假设被破坏。

例如订单支付成功，详情状态、按状态筛选的列表、客户汇总可能都变化。建立事件→key映射：详情exact key更新，相关租户列表前缀失效，汇总按自己的域key处理。太宽会触发大量活跃重取，太窄会留下旧列表与错误计数。

invalidateQueries按匹配范围标记并按配置重取，不等于所有历史缓存、其他浏览器与服务端CDN同时强制刷新。默认活跃查询的行为与inactive需区分；如果后续导航必须立即看新值，应设计等待重取或直接写入可信结果，不能把调用失效函数等同一致性已完成。

不要把单实体详情对象直接塞进包含排序、分页、聚合的全部列表。一个记录状态变化可能移出当前筛选页，补位记录和总数也变化，无法仅改一个字段准确维护；不确定时宁可重取相关列表。服务端事件推动失效时还需处理重复、断线补偿与跨租户范围。

## 六、Retry：分类、预算与多层放大

Retry适合确认可安全重放的瞬时故障，例如只读查询遇部分网络/临时服务不可用。参数错误、权限拒绝和业务冲突通常需修改输入或重新授权，不应原样反复发。429可能在服务端指定等待条件后重试，408等也应按上下文判断，所以“所有4xx都绝不重试”过于机械。

v5普通客户端查询默认有自动重试，而Mutation默认不应被理解为自动可靠重放。业务应显式配置，不依赖默认隐藏等待时间。重试次数外还有总体deadline、单次timeout、退避与jitter；浏览器断网/CORS与真正服务器超时也需可诊断，不能盲重试掩盖配置问题。

假设前端、网关、后端各最多尝试3次，最坏下游可能承受3×3×3=27次尝试，而不是9次。统一责任和总预算，尽量让最了解幂等与错误类别的一层负责。UI重试按钮也可能与自动重试叠加，应显示状态并防止无意重复操作。

写操作超时可能“服务器已成功但响应丢失”。取消、组件卸载或本地回滚都不证明交易未发生；付款需幂等键、状态查询和对账，不能生成新业务ID盲目重发。服务器提供ETag/版本时，客户端带If-Match等条件更新；冲突后重新读取并让用户确认，具体HTTP契约按后端实现落实。

## 七、乐观更新：先显示预测，再与权威对齐

### 7.1 五步流程与必要前提

经典流程为取消相关进行中读取、保存旧快照、写入预计结果、失败回滚、最终与服务器Reconcile（对齐）。取消读取减少旧快照覆盖预测的机会，保存快照用于恢复，不代表可以回滚服务器。只有明确可逆、低冲突且失败可解释的交互适合，例如独立收藏标记。

下面为v5缓存乐观更新Hook示意，必须由应用提供toggleFavoriteApi，它按输入设置明确目标值而非“再翻转一次”，返回服务器确认数据。仅适合同实体写入已由调用方串行化/禁止重叠，且示例不覆盖其他共享列表副本。

~~~jsx
import { useMutation, useQueryClient } from '@tanstack/react-query';

export function useFavorite(tenantId, itemId, toggleFavoriteApi) {
  const client = useQueryClient();
  const key = ['favorite', tenantId, itemId];

  return useMutation({
    mutationFn: favorite => toggleFavoriteApi({
      tenantId, itemId, favorite
    }),
    retry: false,
    onMutate: async favorite => {
      await client.cancelQueries({ queryKey: key, exact: true });
      const previous = client.getQueryData(key);
      if (previous !== undefined) {
        client.setQueryData(key, { ...previous, favorite });
      }
      return { previous };
    },
    onError: (error, favorite, context) => {
      if (context?.previous !== undefined) {
        client.setQueryData(key, context.previous);
      }
    },
    onSettled: () => client.invalidateQueries({
      queryKey: key, exact: true
    })
  });
}
~~~

返回Promise的onSettled把相应失效任务纳入回调完成过程，但不保证服务端最终一致副本已经追平。previous不存在时不创造不完整实体；业务可在无基线时禁止操作。Hook示例不包含UI，调用方应展示pending/error、限制同实体所有入口并处理身份切换，不能仅禁用一个按钮就假定多组件/多设备串行。

### 7.2 并发为什么破坏朴素回滚

A保存v0并改A，B在其后保存状态并成功改B，随后A失败恢复v0，会抹掉B。每个Mutation都有自己的快照并不让整体恢复正确。可按操作ID维护补丁队列、基于服务器版本应用结果、串行同实体写入，或冲突复杂时只显示pending预测层并最终重取，不直接覆盖共享缓存。

响应乱序也有相同风险：旧请求迟到成功，不应覆盖更高版本结果。服务端版本/ETag及明确更新顺序比“最后收到的就是最新”可靠。若重取读到暂时落后的副本，应结合业务一致性要求等待确认版本或提示同步中，不能承诺调用一次invalidate就实现强一致。

付款、库存扣减和权限授予不可仅凭乐观界面宣布事实。可以即时显示“正在处理”提升反馈，但最终成功须来自服务端确认；失败与结果未知要有不同文案和恢复路径。

## 八、Race Condition与取消的三层含义

快速从客户A切B，key应区分身份，旧A结果归A缓存，不应被放到B页。queryFn消费AbortSignal可让底层fetch停止相关等待；没有消费signal时，卸载后的读取可能继续并填充缓存，这有时有利于返回复用，不应笼统称为内存泄漏。

取消库查询、取消客户端网络、撤销服务器操作是三件事。cancelQueries会影响库状态，signal控制支持取消的请求，服务器可能仍已处理。避免在每个then中额外setLocalCopy制造失效副本；若确需局部异步状态，按当前请求ID/生命周期检查归属。

防抖只减少发起频率，不保证已发请求按顺序完成。重复Mutation的时间戳也不是跨设备全局顺序，关键写入需要服务端并发契约。切租户/退出登录时取消或废弃旧作用域请求并清除敏感缓存，不能只改页面标题。

## 九、SSR、Dehydrate与Hydration

SSR预取后Dehydrate（提取可传输缓存状态），客户端在对应QueryClient与HydrationBoundary中恢复，可减少首屏重复获取。服务器QueryClient必须按请求隔离，不能模块级单例携带所有用户私有结果；客户端初始key、参数和权限作用域必须一致。

只传客户端有权看到的字段，安全序列化，避免原样插进script引入XSS。错误对象、内部日志、令牌和不必要敏感数据不应脱水发送。页面HTTP/CDN缓存也要正确隔离，QueryClient隔离并不能阻止代理把私有HTML共享。

dataUpdatedAt和staleTime影响客户端是否立即认为预取结果旧；服务器到客户端耗时、时钟差及缓存层可能影响体验，应测试生产构建。请求完成后的资源释放与内存预算也重要，不把gcTime设置Infinity当服务器无限缓存方案。RSC/Streaming框架集成有独立约束，优先按所用框架版本接入，而不是手写通用HTML拼接。

## 十、Observability、容量与失败UX

跟踪API延迟、错误类别、尝试次数、查询缓存复用、活跃请求、失效触发量、关键Mutation失败及结果未知数量。区分“客户端读了缓存”和“HTTP/CDN命中”，它们不是同一个Cache Hit；监控标签用低基数域/路由，不把每个客户ID或完整key塞进指标。

用户可见错误给重试、调整筛选、重新登录或状态查询等具体动作，不暴露堆栈。后台刷新失败可保留带更新时间的已有数据；初次无数据与空结果单独显示。交易超时应提示查询结果，避免“失败，再付一次”诱发重复交易。

验收场景包括同key多订阅、参数快速切换、429与网络断开、重试耗尽、后台失败、并发乐观A失败B成功、版本冲突、SSR两用户隔离和退出后的迟到响应。压测还要检查聚焦/重连集体触发和过宽失效造成的流量尖峰，库默认值无法替业务做容量规划。

## 十一、完整面试回答与深入追问

### 11.1 Query Key和staleTime怎样设计？

“Key首先是数据身份，应覆盖租户、参数和分页并用Factory统一；staleTime由允许旧窗口决定，gcTime由返回复用和内存预算决定。它们分别影响刷新判断与inactive回收，不能当一个TTL开关。”

追问：Infinity一定不可以吗？可以用于稳定内容或可靠事件失效，但必须有更新协议；权限、库存不能仅为省请求永远fresh。

### 11.2 Mutation成功如何更新缓存？

“可信完整返回可更新exact详情，列表与汇总按域事件映射精确失效。涉及排序筛选和分页时不盲目手工修所有副本，选择重取。等待失效任务不代表其他浏览器或后端副本立刻强一致。”

追问：invalidate全站最简单？会造成活跃查询风暴并损害成本和延迟，应该从影响集合出发设计key层级，监测触发量。

### 11.3 乐观回滚与重试有哪些风险？

“快照回滚只在没有覆盖其他操作的前提下可靠。并发A失败不能抹掉B成功，需操作补丁、版本或串行；交易超时还可能已在服务器成功。重试按幂等和错误分类，有预算与退避，不靠UI取消撤销交易。”

追问：三个层各3次最多几次？独立嵌套最坏27次，应统一责任和deadline，次数不是简单相加。

## 十二、扫雷自测与完整答案

1. **Key少一个过滤参数会怎样？** 不同查询共享错误身份，读到其他结果或覆盖缓存；参数与key需一致。
2. **对象属性顺序改变就新key吗？** v5默认稳定哈希会处理对象顺序，数组位置仍有意义；不要靠不稳定非序列化对象作身份。
3. **staleTime到点就删吗？** 不，stale是新鲜判断，gcTime是inactive回收，定时轮询另配置。
4. **fetch的500会自动reject吗？** 通常不会，要检查response.ok并转错误，否则库可能缓存错误响应为成功。
5. **所有4xx都不重试吗？** 业务输入/权限通常不原样重试，但429等按服务端等待与幂等条件处理，不能机械归类。
6. **取消查询等于服务器回滚吗？** 不，库状态、网络等待与服务器事务不同，交易需幂等与查询确认。
7. **快照回滚为何可覆盖别人？** 快照落后于后续成功操作，整对象恢复会抹掉新结果；需操作级策略或重取。
8. **invalidate是否所有端立即刷新？** 不，匹配范围、订阅活跃性和配置影响本Client行为，跨端和服务端副本需另设计。
9. **SSR一个全局QueryClient安全吗？** 私有状态可能跨请求复用泄漏，应按请求隔离并安全脱水，CDN也要隔离。
10. **错误界面只给重试够吗？** 不，权限需重新授权，冲突需重新读取确认，交易未知需查状态，动作应匹配失败语义。

## 十三、速记、来源与验证范围

速记：**Key定身份，fresh定允许旧，GC定无人订阅保留，失效定影响范围，Retry受幂等预算约束，乐观受并发约束，SSR按用户隔离。**

适用TanStack Query v5与React18+。以下官方v5链接本次转向当前文档且页标v5；示例采用兼容的回调前几个参数和闭包QueryClient，不依赖新版额外context参数。旧版cacheTime在v5对应gcTime命名，不能直接混用配置。

- [Query Keys](https://tanstack.com/query/v5/docs/framework/react/guides/query-keys)
- [Important Defaults](https://tanstack.com/query/v5/docs/framework/react/guides/important-defaults)
- [Optimistic Updates](https://tanstack.com/query/v5/docs/framework/react/guides/optimistic-updates)
- [Query Cancellation](https://tanstack.com/query/v5/docs/framework/react/guides/query-cancellation)
- [Server Rendering and Hydration](https://tanstack.com/query/v5/docs/framework/react/guides/ssr)

验证范围：JavaScript/JSX仅人工核对key、回调、错误、取消和恢复路径，未实际运行TanStack/React、网络、SSR隔离或浏览器测试，未做性能与故障实验。时序与27次为推理算例，不是实际后端测试结果。
