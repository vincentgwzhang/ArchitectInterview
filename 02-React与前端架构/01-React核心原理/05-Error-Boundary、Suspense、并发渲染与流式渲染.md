# 05. Error Boundary、Suspense、并发渲染与流式渲染

> 本章以 React18+ 现代根和客户端组件为基础，SSR 示例针对 Node.js；2026-11-22 核验官方文档标19.3。React19异步Action能力与18的同步Transition示例区分说明，不把框架数据协议或内部调度结构当通用标准。目标是让“失败隔离、等待反馈、输入响应与渐进交付”形成可解释的架构。

## 一、先区分四个问题

| 用户问题 | 机制 | 不能替代的责任 |
| --- | --- | --- |
| 局部组件渲染崩溃 | Error Boundary（错误边界） | 业务错误处理、服务端恢复、错误监控 |
| 子内容尚未准备好 | Suspense（等待边界） | 通用请求缓存、超时、权限、重试策略 |
| 重更新让输入迟钝 | Concurrent Rendering 与 Transition | 减少CPU工作、并行执行任意JavaScript |
| 等慢区域才有首屏 | Streaming SSR（流式服务端渲染） | 代理透传、客户端代码加载和水合正确性 |

Failure Domain（故障域）是可独立失效和恢复的区域。推荐组件失败可以保留购物车，订单确认失败则不能显示“下单成功”。边界不只是JSX包装，而是产品决定哪些区域一起出现、哪些区域可降级、失败后如何恢复。

Render计算候选UI，Commit把选定结果应用到界面。候选计算可能重做或丢弃，所以不能把render次数等同请求次数、埋点次数或交易次数。等待和失败也是不同状态：一直旋转的spinner不是错误处理。

## 二、Error Boundary：捕获范围与可恢复性

### 2.1 什么会被捕获

错误边界处理后代组件在渲染、构造及React管理的相关生命周期中抛出的错误，切换到Fallback UI（后备界面）。边界自身render或fallback再次抛错，需要更外层边界；同一边界不能保护自己。

普通事件处理器、setTimeout回调、独立Promise rejection通常不经这条渲染错误传播路径，应在相应异步/事件代码捕获，转换成业务错误状态并记录。React19文档对Transition Action中的错误另有边界集成，不能把“任何异步错误都不捕获”或“任何异步错误都能捕获”当无版本条件的规则。

客户端Error Boundary也不是服务器渲染try/catch。SSR需要服务端渲染API和框架错误处理；流式Suspense可让部分服务器失败转交客户端重试，后文解释。预期的余额不足、无权限、查询空结果属于明确业务状态，通常不必抛成渲染崩溃。

### 2.2 一个有明确恢复路径的实现

React当前仍支持用class定义原生错误边界；一般业务组件可继续函数式编写。getDerivedStateFromError只计算错误状态，componentDidCatch处理上报，不在render中发送日志。

~~~jsx
import { Component } from 'react';

export class PanelBoundary extends Component {
  state = { failed: false };

  static getDerivedStateFromError() {
    return { failed: true };
  }

  componentDidCatch(error, info) {
    this.props.onError?.(error, info.componentStack);
  }

  render() {
    if (this.state.failed) {
      return <section role="alert">
        <p>此区域暂时无法显示，其他功能仍可使用。</p>
        <button type="button" onClick={this.props.onRetry}>
          重新加载此区域
        </button>
      </section>;
    }
    return this.props.children;
  }
}
~~~

调用者必须提供onRetry。一个明确的契约是外层先执行资源层invalidate/reload，再增加attempt，并以 key={attempt} 重新挂载边界；仅清failed不能修复仍会抛错的数据。onError应由应用提供不向UI抛错的上报适配器，避免上报失败再破坏fallback；不要把上报队列的可靠性误当此例已实现。

key重建会丢掉被替换子树的局部状态，编辑器草稿应放在适当的更长生命周期所有者，或事先持久化。边界切fallback本身也会移除失败子树，关键草稿不能等失败后再抢救。重试有上限，持续配置错误应提供离开页面、联系客服或复制非敏感故障编号的路径，不能无限自动重挂造成风暴。

### 2.3 如何放置边界与上报

Route边界用于阻止一页崩溃扩散到导航壳；Widget边界隔离可选推荐、图表；Editor边界结合草稿策略。全站一层可作最后防线，但若任何可选模块失败都整站白屏，隔离粒度过粗。反之每个小按钮一层会让恢复与日志难协调，也可能割裂业务完整性。

记录error、component stack、route、发布版本、时间和关联trace/request标识，并按指纹去重。component stack表示组件层级，JavaScript stack表示调用链，两者不能互相替代。避免把令牌、完整用户输入和内部堆栈放进用户可见错误页面；服务端日志与前端发布版本共同定位问题。

## 三、Suspense：协调“未准备好”的显示

### 3.1 哪些加载会激活它

Suspense在后代渲染遇到受支持的挂起来源时显示fallback，例如lazy代码拆分、集成Suspense的数据框架；当前React也有读取缓存Promise的use能力，但它不是React18通用fetch模板。不要自行把不稳定的内部抛Promise协议当长期公共接口。

在useEffect里fetch再setState，不会自动触发Suspense；把任意组件包起来也不会让React知道它所有网络请求。简单Effect查询仍需自己显示loading/error并防止旧响应覆盖。框架数据层负责缓存键、生命周期和挂起协议，Suspense负责UI协调，两者职责不同。

~~~jsx
import { lazy, Suspense } from 'react';

const Reports = lazy(() => import('./Reports.js'));

export function ReportsArea() {
  return <PanelBoundary
    onRetry={() => window.location.reload()}
    onError={(error, stack) => {
      console.error('Reports boundary', error, stack);
    }}>
    <Suspense fallback={<div role="status"
      style={{ minHeight: 180 }}>报表加载中</div>}>
      <Reports />
    </Suspense>
  </PanelBoundary>;
}
~~~

需从上节模块导入PanelBoundary，Reports.js默认导出一个组件。示例用于浏览器环境，console仅为开发演示，生产换成脱敏上报；整页reload是明确但粗粒度的代码资源恢复，会丢失未持久化页面状态。不要把它作为编辑器默认重试。lazy加载被拒绝通常交错误边界；等待期间交Suspense。lazy声明在模块顶层，避免每次render创建新组件身份。

### 3.2 边界粒度与揭示顺序

一个边界内多个子内容通常一起揭示；把推荐放内层边界，可以先展示商品主体再补推荐。边界应对应用户理解的一组内容与布局，不等于每个请求或每个组件。骨架（Skeleton）应尽量预留稳定空间，减少内容出现时的Layout Shift（布局位移）。

边界过大，慢的次要模块拖住整页；过细则多spinner、跳动和难以理解的部分状态。不要因为“技术上可独立加载”就把价格和结算资格拆得互相矛盾；必须同时理解的业务信息可作为一组揭示，或明确标示未确认状态。

第一次挂起且未成功挂载的树没有可保留的已提交局部state；React重试时会重新渲染。已显示内容再次挂起可能隐藏并显示fallback；Transition或deferred value可在合适更新中保留旧内容。隐藏已显示内容时，布局Effect可能清理并在重新显示时建立，资源逻辑须能应对，而不是假定“可见一次就永远挂着”。

## 四、并发渲染：优先级而不是任意代码并行

Concurrent Rendering允许React在准备UI时中断、重做或放弃某些工作，先处理紧急输入。用户看到的仍是已提交的一致树，而不是渲染一半直接暴露的半成品。具体调度优先级与内部Fiber结构不属于业务可依赖保证。

并发不等于多个CPU线程同时跑所有组件，也不保证总耗时更短；额外重算甚至增加CPU消耗，收益是响应性。组件函数里一个不可切分的巨大同步循环依旧阻塞主线程，startTransition不会把该循环变成Worker。应先减少数据规模、使用索引或窗口化；确需CPU隔离再使用Web Worker，明确数据复制与通信成本。

Pure Render（纯渲染）意味着相同输入不造成可观察的外部写入。render里给全局数组push、发交易或写Store会在候选被丢弃时留下副作用；正确做法是事件处理业务动作、Effect管理已提交UI的外部同步，并按生命周期清理。StrictMode额外开发检查帮助发现问题，但不等于生产固定执行两次。

## 五、Transition：让输入与昂贵结果分开更新

~~~jsx
import { memo, useState, useTransition } from 'react';

const ResultList = memo(function ResultList({ query, products }) {
  const matches = products.filter(p =>
    p.name.toLowerCase().includes(query.toLowerCase())
  );
  return <ul>{matches.map(p => <li key={p.id}>{p.name}</li>)}</ul>;
});

export function Search({ products }) {
  const [input, setInput] = useState('');
  const [query, setQuery] = useState('');
  const [isPending, startTransition] = useTransition();

  function handleChange(event) {
    const next = event.target.value;
    setInput(next);
    startTransition(() => setQuery(next));
  }

  return <>
    <label>搜索<input value={input} onChange={handleChange} /></label>
    <p role="status">{isPending ? '正在更新结果' : ''}</p>
    <div aria-busy={isPending}>
      <ResultList query={query} products={products} />
    </div>
  </>;
}
~~~

假设products是不可变更新且身份稳定的数组，id唯一、name字符串。受控输入input立即更新，query低优先级更新；memo使紧急输入更新时在query/products未变的情况下有机会跳过结果列表。仅创建两个state却让昂贵子树无条件随父渲染，可能削弱效果。

startTransition回调立即执行，标记其中相应state更新，并不延时调用它。此例filter本身仍为同步计算，大到阻塞时应拆解或改变算法，不能宣称列表过滤可在每一次循环中被抢占。Transition不减少网络请求，也不是防抖；发起外部请求仍需去重、取消和结果身份保护。

useTransition返回pending提示，独立startTransition函数适合组件外标记更新但本身不给pending。useDeferredValue则让一个已有值的消费者暂用旧值，适合你无法直接控制更新源时；它也不提供固定等待时长，不会自动减少请求。旧结果应清楚标注“更新中”，避免用户以为旧价格或权限是当前确认事实。

本章同步写法兼容18+。React19支持异步Action等扩展，但await之后的更新如何标记、错误如何进入边界以及多个请求的完成顺序，必须按具体版本API设计；不能把任意async回调包一次就宣称全部异步结果有序。交易仍由服务端保证幂等与并发控制。

## 六、Streaming SSR：Shell先送，慢区域后补

### 6.1 从HTML到可交互

SSR（Server-Side Rendering）在服务器生成HTML；Streaming先输出Shell，即不必等待慢Suspense区域的页面结构及fallback，后续再输出准备好的内容与所需指令。它可缩短用户等待主体内容的时间，但不能保证TTFB在所有环境一定下降：Shell自己很慢、代理缓冲或网络拥塞仍会延迟。

Hydration（水合）让客户端React接管已有HTML并建立交互。HTML已出现不代表按钮已能执行客户端业务；还依赖代码下载、执行与对应区域水合。Selective Hydration（选择性水合）结合边界使React能够优先处理需要交互的区域，不必等全页全部准备完，但不能在所需JavaScript缺失时凭空交互。

Server Component在服务端执行组件逻辑，Client Component负责客户端能力，这是执行边界；SSR是生成初始HTML，二者不是互斥概念，客户端组件也可由框架预渲染HTML。Server Components通常通过框架提供协议，不能仅换renderToPipeableStream就获得整套RSC能力。

### 6.2 Node流式入口与失败路径

以下为接入模式，需Node HTTP请求response、已授权且隔离的requestData，以及输出完整html文档的Document组件。框架项目优先采用其既有服务器入口，避免重复水合/流协议。

~~~jsx
import { renderToPipeableStream } from 'react-dom/server';

export function respond(response, Document, requestData) {
  let failed = false;
  let timer;
  const { pipe, abort } = renderToPipeableStream(
    <Document data={requestData} />,
    {
      bootstrapScripts: ['/client.js'],
      onShellReady() {
        response.statusCode = failed ? 500 : 200;
        response.setHeader('Content-Type', 'text/html; charset=utf-8');
        pipe(response);
      },
      onShellError() {
        response.statusCode = 500;
        response.setHeader('Content-Type', 'text/html; charset=utf-8');
        response.end('<!doctype html><p>暂时无法加载，请稍后重试</p>');
      },
      onError(error) {
        failed = true;
        console.error('SSR failed', error);
      }
    }
  );
  timer = setTimeout(abort, 10000);
  response.once('finish', () => clearTimeout(timer));
  response.once('close', () => {
    clearTimeout(timer);
    abort();
  });
}
~~~

10秒是演示预算，不是通用建议；需按上游截止时间与业务SLO决定。示例只展示流渲染取消，未自动取消Document底层数据请求，仍需请求层传播取消信号。console为开发演示，生产应记录脱敏错误、请求与发布标识。Web Streams运行时应使用相应renderToReadableStream，而非Node pipe API。

Shell失败时还未开始正常页面输出，可以设置500并返回安全兜底；Shell已送出后HTTP状态与响应头不能再随意改写，晚到区域错误需在UI和遥测表示，不能期待把既有200改500。对某些爬虫/静态生成，可选择等待全部就绪再发，代价是失去早送收益；这必须是明确产品策略。

## 七、流式架构的五个必查边界

### 7.1 服务器错误与客户端恢复

服务器在Suspense区域内失败时，流式机制可输出该边界fallback并让客户端尝试渲染；若客户端仍失败，再进入相应错误边界。一次服务器错误不必必然整页失败，也不能因客户端最终恢复就忽略服务器onError日志。客户端错误边界不等于服务器恢复策略。

重试要区分代码chunk下载失败、数据超时、权限拒绝与组件确定性bug。缓存坏结果一直重试无用，404旧chunk可能要求部署资源保留/重新加载；权限错误应转权限界面而非无限spinner。清理重试前的请求与订阅，保留未失败区域，避免重复交易。

### 7.2 缓存与用户隔离

公共Shell可以缓存，个性化内容必须按认证与租户边界隔离；不能只按URL缓存整条含用户数据的流。流中后发的片段同样受安全策略约束。服务端模块级可变单例缓存当前用户会跨请求污染，应使用请求作用域与明确缓存键。

把服务器数据交给客户端时使用框架安全序列化，避免把未转义用户内容直接拼进script造成XSS。私密凭据不能因“在服务端组件中读取”就随props/HTML送出。CSP（内容安全策略）可能限制流式内联指令，nonce等配置应沿框架/渲染API传递并与响应头一致。

### 7.3 Hydration Mismatch

服务端与客户端第一次渲染应匹配。Date.now、Math.random、不同语言时区格式、非法HTML嵌套和读取浏览器专属存储造成差异，都可能触发Hydration Mismatch（水合不一致）。传递确定的初始数据与格式条件，浏览器特有内容放明确客户端阶段或框架边界。

不要普遍加suppressHydrationWarning隐藏问题，它只适合有限不可避免差异，也不会保证所有错配自动修补。恢复为客户端渲染可能增加工作、丢失预期状态并延后交互，应连接hydrateRoot的适用错误上报接口，按项目版本核对配置。

### 7.4 代理、资源与真实交付

CDN、反向代理或压缩链路若缓冲整个响应，服务器即使逐段写也可能用户一次才收到。应从真实部署入口观察分块到达时间、HTML可见时间和交互时间，不只在localhost看response.write日志。部署需保证旧HTML引用的chunk仍可用，否则缓存页面遇到新版本404。

### 7.5 资源预算与失败UX

流式可让慢请求更早占用连接并保持更久，需限制并发、数据截止时间、客户端断开后的清理和服务器内存。不能用无限等待换取“最终完整”。Loading、Empty、Partial Error、Complete Error四种状态应分别设计：空结果可调整筛选，局部错误可单独重试，完全错误给稳定导航与退出路径。

骨架预留空间并提供适度状态提示，不用多个持续播报的alert骚扰屏幕阅读器。保持焦点与编辑内容，降级时不能把金额、授权或交易成功伪造成可用。Observability要把区域失败率、持续fallback、chunk错误与水合错误关联到发布，才能区分网络、部署和逻辑故障。

## 八、面试完整回答与深入追问

### 8.1 错误边界与等待边界如何放置？

“我按用户任务和故障域分组。页面壳有最后防线，可选推荐有局部错误与等待边界，订单确认则保持一致的业务状态。Suspense处理受支持的未就绪来源，Error Boundary处理相应渲染失败；事件和独立异步错误另处理。失败记录版本与请求链路，重试同时处理资源与组件状态。”

追问：边界重挂就一定恢复？不一定，坏缓存或确定性bug仍会失败；key还会清局部草稿，所以恢复协议必须包含资源失效与草稿所有权。

### 8.2 为什么并发要求纯渲染？

“React可以准备后丢弃某次候选，若render写外部Store或发请求，丢弃UI不会回滚这些副作用。组件只计算UI，事件处理明确业务动作，Effect管理提交后的外部关系。Transition把紧急输入和可延后结果区分，改善响应性但不把任意同步代码变成并行。”

追问：长循环包Transition能不卡吗？不能，回调立即执行且单个长JS任务仍占线程。需算法优化、拆分或Worker，UI调度不能替代计算治理。

### 8.3 Streaming能解决首屏所有问题吗？

“它能先交Shell和逐步揭示慢区域，仍需客户端代码及水合才可交互。我会检查Shell依赖、缓存与用户隔离、晚错误状态语义、代理缓冲和真实交互指标。SSR不自动等于RSC，也不自动消除水合错配。”

追问：头发出后区域失败如何返回500？通常不能修改已发状态。通过局部错误UI、客户端恢复与遥测表示，Shell前失败才有机会选择整体响应状态。

## 九、扫雷自测与完整答案

1. **Error Boundary不捕获什么？** 自身错误、普通事件处理器和独立异步回调通常不在其渲染捕获范围，SSR另走服务器处理。React19 Transition Action集成是需注明版本的例外，不能扩大为任意Promise。
2. **为什么render必须纯？** 候选可重做或丢弃，外部写入无法随候选回滚，导致重复或状态污染。
3. **Suspense过细有何体验问题？** 多spinner、布局跳动和割裂信息揭示。按用户理解的一组内容与一致性要求分组，而非按每个fetch。
4. **Effect里fetch会自动挂起吗？** 不会，Suspense需受支持来源；普通查询仍要手动loading/error或接入有明确集成的数据层。
5. **Transition能控制文本输入值吗？** 受控输入应紧急更新，结果等可延后部分单独标Transition，否则输入反馈可能不正确。
6. **并发是否一定减少总CPU？** 不一定，可能重算；主要收益是调度响应性，长同步函数仍阻塞。
7. **lazy失败怎样处理？** 等待交Suspense，加载拒绝通常交错误边界；重新挂边界不保证清除lazy/模块缓存或修复chunk404，需要明确资源恢复策略。
8. **HTML显示为何还不能点击？** 代码与对应区域水合可能尚未完成。选择性水合能协调优先级，但不能省去所需代码。
9. **Streaming晚错误能改HTTP状态吗？** 头已送出一般不能，需局部UI恢复和遥测；Shell失败可在发出前设置状态与兜底。
10. **个性化流按URL缓存安全吗？** 不一定，同URL可有不同用户租户数据。需明确缓存作用域，后发片段也要隔离与安全序列化。

## 十、速记、来源与验证范围

速记：**错误按故障域隔离，等待按内容组协调，输入紧急结果可延后，候选渲染必须纯，Shell先送仍需水合，晚失败用局部恢复与遥测。**

本章React18+同步Transition、lazy/Suspense与Node流式接口为基础；异步Action等19能力单独标注，具体框架数据缓存和RSC协议须按所用框架版本落实，不自造通用fetch挂起接口。

- [React Component：错误边界](https://react.dev/reference/react/Component#catching-rendering-errors-with-an-error-boundary)
- [React Suspense](https://react.dev/reference/react/Suspense)
- [React useTransition](https://react.dev/reference/react/useTransition)
- [React renderToPipeableStream](https://react.dev/reference/react-dom/server/renderToPipeableStream)
- [React hydrateRoot](https://react.dev/reference/react-dom/client/hydrateRoot)

验证边界：全部JSX及Node接入模式仅人工核对，未运行React runtime、Node HTTP流式服务器、浏览器水合、代理/CDN、CSP或性能实验。示例说明前置依赖与恢复限制，未把理论时序宣称为真实部署性能或故障测试结果。
