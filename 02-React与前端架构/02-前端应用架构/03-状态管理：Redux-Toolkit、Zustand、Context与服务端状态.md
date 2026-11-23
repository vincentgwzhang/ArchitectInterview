# 03. 状态管理：Redux Toolkit、Zustand、Context 与服务端状态

> 选型先问事实属于谁、活多久、谁共享、更新多频繁、如何调试和恢复。本文以React18+客户端、Redux Toolkit 2.x、Zustand 5.x常用模式与TanStack Query v5为讨论范围；2026-11-23核验官方资料。不同运行时、库版本及SSR行为需以项目锁定版本验证，不能把默认值当业务保证。

## 一、先分类：一个Store不是所有事实的归宿

| 类别 | 例子与权威来源 | 合适位置与边界 |
| --- | --- | --- |
| Local UI State | 弹窗、Tab、当前输入 | 最近组件/区域；离开是否重置需明确 |
| URL State | 搜索条件、页码、选中ID | 路由/URL；支持分享刷新和前进后退 |
| Server State | 后端订单、列表、请求与过期状态 | 查询缓存；权威记录仍在服务器 |
| Global Client State | 跨页向导、未提交草稿 | 有限作用域Store；不必应用根永驻 |
| Form State | 字段值、Touched、校验与提交 | 表单局部或字段订阅库 |

Derived State（派生状态）能由现有事实计算，如购物车总件数，优先直接计算或用selector，不再复制存储。两份数据如果必须靠Effect反复追平，通常说明权威来源不清；但“服务器记录”和“用户编辑草稿”有不同含义，可同时存在，需基线版本与冲突策略。

决策顺序不是机械互斥条件链。先问能否派生、是否局部；再分别确定服务器权威与URL表示：URL中的orderId指明看哪份订单，Query Cache保存其远端快照，局部state保存展开项，Store可保存跨页草稿。几种工具组合完全正常，禁止的是同一语义多份无主副本。

## 二、Ownership、Lifetime与更新契约

Ownership（所有权）指谁决定事实变化及其合法规则；Lifetime（生命周期）指组件、路由、会话、实体编辑过程还是持久化记录。Shared（共享）与Global（全局）不同：兄弟共享只需提升父级；跨页面但仅属于一个向导的草稿可使用区域Provider，不必让全站订阅。

为每份状态记录读取者、允许动作、重置条件、敏感范围和恢复规则。例如结算草稿只能经changeAddress/confirm等动作变化，切租户必须重置，提交失败保留输入，成功后以服务端订单结果为准。不要让任意组件直接写内部字段来绕过规则。

前端单个Store更新不等于数据库事务。多个API调用失败后，Redux回放或Zustand reset不会撤销付款；需服务端幂等、版本校验与补偿。多浏览器/多标签页也不因使用同一个库自动同步，需明确跨上下文协调与冲突解决。

## 三、Context：依赖分发与订阅，不是完整治理套件

Context提供跨层依赖，如主题、语言、用户展示信息或服务实例；state可以由Provider内的useState/useReducer持有。useContext读取最近上方Provider，value按Object.is变化时通知相应消费者。读取对象的某字段不等于原生获得该字段selector订阅，memo也不能挡住消费者接收新的Context值。

低频且职责稳定的值适合Context，高频大对象会牵动许多无关消费者。按领域和频率拆分，必要时分开state与稳定dispatch Context，再稳定无关父渲染产生的value引用。useMemo不能阻止真实依赖变化传播；漏依赖只是产生陈旧值。

Context没有内建请求缓存、持久化迁移、事件回放与字段级selector，并不意味着它不能承载复杂逻辑，而是这些能力需要自己建设。少量设置用Context可降低依赖成本；若团队已开始手写细粒度外部订阅、调试历史和复杂异步编排，应评估成熟Store。Auth Context只影响UI显示，服务器仍需授权。

## 四、Redux Toolkit：显式动作与可预测更新

### 4.1 核心执行链

Redux通过dispatch Action（动作）描述事件，Reducer根据旧状态与动作计算新状态，订阅者读取所需片段。Redux Toolkit的createSlice生成slice reducer及action creators，configureStore装配store和常用开发检查。Slice是按业务责任切分状态更新，不是每个组件一个文件的强制规则。

Immer允许在RTK reducer中以修改draft的语法产出不可变结果，并通过结构共享保留未改分支引用。这个便利只在相应draft上下文成立，不允许在组件或异步回调里直接改store对象；reducer仍须纯，不能fetch、读当前时间决定结果或发送交易。

~~~javascript
import { configureStore, createSlice, createSelector } from '@reduxjs/toolkit';

const draftSlice = createSlice({
  name: 'checkoutDraft',
  initialState: { delivery: 'standard', note: '' },
  reducers: {
    deliveryChanged(state, action) {
      if (action.payload === 'standard' || action.payload === 'express') {
        state.delivery = action.payload;
      }
    },
    noteChanged(state, action) {
      state.note = action.payload;
    },
    draftCleared() {
      return { delivery: 'standard', note: '' };
    }
  }
});

export const { deliveryChanged, noteChanged, draftCleared } = draftSlice.actions;
export const store = configureStore({
  reducer: { checkoutDraft: draftSlice.reducer }
});
export const selectDelivery = state => state.checkoutDraft.delivery;
export const selectPreview = createSelector(
  [selectDelivery, state => state.checkoutDraft.note],
  (delivery, note) => ({ delivery, note })
);
~~~

示例在浏览器单会话使用；React接入需react-redux Provider及useSelector/useDispatch，不是导出store后到处store.getState就自动订阅。noteChanged假设调用方提供字符串，生产需输入校验与长度限制；Action可从任何入口传入，关键不变量不能只由某个按钮保证。

Memoized Selector（记忆化选择器）仅当输入未变时复用派生结果引用，降低重复计算/对象变化。此例selectPreview是轻量展示，主要演示语义，不宣称性能收益。普通selector若每次返回新对象，默认引用比较可能造成额外更新；不要为每个简单字段机械memo。

### 4.2 Middleware、调试与适用边界

Middleware（中间件）包裹dispatch链，可记录、处理异步任务或响应动作，但外部副作用要有显式失败与取消策略。复杂工作流可用thunk/listener等组织，不能把异步混入reducer。通常保持state/action可序列化，避免DOM、Promise和连接对象破坏回放与持久化。

多Feature共享、动作可追踪、更新规则复杂且团队需统一约定时，RTK有价值。DevTools和time travel帮助重现客户端状态，但不是防篡改审计：日志可被关闭、客户端不可信、外部交易不可逆；真正审计应由服务器持久记录主体、动作和结果。

不要手写所有API response复制进slice。RTK Query已提供Redux生态的服务器状态能力，TanStack Query也可承担这一职责；同一资源选择一个权威查询层，避免无理由同时维护两个缓存和手工slice副本。大型团队规范收益可能值得较多概念成本，小局部功能则不必引入全套全局流程。

## 五、Zustand：简洁接入仍需要明确架构

Zustand的create把状态与动作装成可订阅Hook，组件用selector选择片段。它减少样板，但不强制你按业务切分、记录动作或处理持久化；“API少”不等于复杂系统治理成本自动消失。

~~~javascript
import { create } from 'zustand';

export const useDraft = create(set => ({
  delivery: 'standard',
  note: '',
  setDelivery: delivery => {
    if (delivery !== 'standard' && delivery !== 'express') return;
    set({ delivery });
  },
  setNote: note => set({ note }),
  reset: () => set({ delivery: 'standard', note: '' })
}));

// 组件内分别调用，避免无意订阅整个Store：
/*
const delivery = useDraft(s => s.delivery);
const setDelivery = useDraft(s => s.setDelivery);
*/
~~~

普通set对顶层合并，嵌套对象仍应不可变更新；不能直接state.address.city=...后期待订阅可靠通知。选择稳定primitive/引用有助于减少无关更新；Zustand5尤其需关注selector输出稳定性，返回新数组/对象时按官方支持的shallow辅助或拆分订阅，不能照搬旧版自定义比较参数签名。

中等客户端共享、原型和已有明确领域约束的应用可选择Zustand；更复杂系统也能用，但要自行定义动作命名、Owner、异步结果归属、调试和测试约定。禁止跨Feature任意setState操作内部字段，否则简洁Store最终变成巨大共享可变模型。

示例是浏览器模块级store，不可直接用于SSR保存所有用户状态。SSR应按请求/应用实例创建、提供初始快照且保持客户端首渲染一致；服务端组件不能把全局客户端Store当跨请求身份容器。库是否需要Provider与是否需要作用域是不同问题，避免“无需Provider”被误解为无需隔离。

## 六、Server State：缓存的是远端快照，不是远端权威

### 6.1 查询键、新鲜度和回收

Server State由远端持有并可能被他人改变，客户端只掌握某时刻快照。Query Key标识“哪一份结果”，需包含影响请求结果的ID、筛选、分页及租户等作用域；查询函数参数与key不一致会把A的数据当B复用。key不应包含原始令牌或个人敏感信息，因为可进入调试与日志。

TanStack Query（原称React Query）v5中，staleTime表示何时认为数据需要重新验证，gcTime表示无活跃订阅后缓存保留多久，两者不等同TTL删除。默认缓存会被视作stale，无活跃查询通常保留5分钟；挂载、聚焦、网络恢复等可触发相应后台刷新，具体按配置。stale并不等于立刻清空界面，GC也不意味着服务器数据被删。

Deduplication（请求去重）通常围绕同key共享进行中的查询，而不是任意相似URL永久只请求一次。不要把mutation交易假定自动去重或安全重试。缓存仍受内存、会话和生命周期限制，跨标签页/设备的同步不由普通内存QueryClient自动保证。

### 6.2 最小查询接口与失败语义

~~~jsx
import { useQuery } from '@tanstack/react-query';

export function OrderTitle({ tenantId, orderId }) {
  const query = useQuery({
    queryKey: ['order', tenantId, orderId],
    enabled: Boolean(tenantId && orderId),
    staleTime: 30_000,
    retry: false,
    queryFn: async ({ signal }) => {
      const response = await fetch(
        '/api/tenants/' + encodeURIComponent(tenantId) +
        '/orders/' + encodeURIComponent(orderId),
        { signal }
      );
      if (!response.ok) throw new Error('HTTP ' + response.status);
      const data = await response.json();
      if (typeof data?.title !== 'string') throw new Error('Invalid order');
      return { title: data.title };
    }
  });
  if (!tenantId || !orderId) return <p>请选择订单</p>;
  if (query.isPending) return <p role="status">加载中</p>;
  if (query.isError) return <p role="alert">加载失败，请重试</p>;
  return <h2>{query.data.title}</h2>;
}
~~~

需在上层提供稳定QueryClientProvider；tenantId/orderId约定字符串，接口按当前登录者重新鉴权，URL含租户并不自动授权。示例不配置previous-data占位，避免切订单时将旧内容伪装为新；真实列表可展示旧内容但应标识更新中。retry:false是教学中明确禁用自动重试，不代表所有查询都不应重试。

queryFn显式检查HTTP状态，否则fetch收到404/500仍可能正常resolve。传递signal支持库取消相关等待，但不能保证服务器没执行。示例只显示初次成功或错误，后台刷新失败时产品可选择保留已有快照并提示更新时间，而不是一律抹掉可用信息；关键业务若不允许旧则应限制操作。

### 6.3 Invalidation与乐观更新

Mutation成功后，可将服务器返回结果写回相应key，或invalidate相关列表/详情触发重新验证。Invalidation（失效）是让缓存需要更新，不等于所有浏览器瞬间变新；必须明确过滤范围、活跃订阅和界面等待策略。

Optimistic Update（乐观更新）先显示预计成功的结果，再按响应确认或回滚。点赞等可逆操作适合；付款成功、权限授予不应仅凭乐观UI当事实。基本流程是识别操作、保存基线/撤销信息、避免旧查询覆盖、应用临时变化、失败恢复或重新拉取、最终与服务器对齐。

并发A/B修改时，A失败后盲目恢复整个旧快照可能覆盖B已成功结果。可用每操作补丁/版本与局部回滚，或并发复杂时直接失效重取，必要时串行化同实体写入。服务器仍需幂等与版本条件，查询库不会解决多客户端数据库并发。

## 七、URL State与Form State的边界

可分享与刷新重现的过滤、排序和页码优先URL。读取时校验缺省值、允许集合和范围，URL属于不可信输入；改变筛选通常需重置页码，避免停留无结果的旧页。浏览器Back/Forward必须重新派生界面，不要只在首次挂载时复制URL到Store后再失联。

临时输入可先局部保存，用户确认搜索再写URL；replace适合不想每次打字增加历史，push适合用户期待返回的明确导航。分享URL不要放令牌、敏感草稿或个人信息。URL改变可触发相应queryKey改变，查询结果不用再复制进全局Store。

Form State区分value、touched（是否触碰/离开过字段）、dirty（相对基线是否修改）和提交状态。后台刷新不应无条件覆盖dirty草稿；保留服务器版本基线，提交时检测冲突，成功后按确认结果重设基线。大量字段可用表单库细粒度订阅，Redux保存每个按键通常不是默认好选择，但跨页长期草稿可有明确持久化需求。

## 八、持久化、隐私与恢复

Persist（持久化）只保存白名单业务字段，版本化schema并提供迁移、验证与清除策略。不能把整个Store，包括loading、error、连接、QueryCache和过期权限全部序列化；页面重开后“提交中”残留可能让流程卡死，旧权限可能造成误导。

本地存储可被脚本读取，不作为秘密保险箱。共享设备退出登录、切租户、权限撤销和账号切换时清除或隔离草稿/缓存，并使旧请求失效。持久化恢复可能晚于首次渲染，SSR水合阶段需避免用不一致快照生成首屏，恢复中给明确状态。

迁移失败宁可提示恢复或丢弃不兼容草稿，也不把坏数据当合法交易输入。跨标签页同时编辑需版本冲突或明确最后写入策略；浏览器存储事件/广播只是传递机制，不提供数据库级事务。敏感数据保存期限和用户主动删除路径也属于状态架构。

## 九、选型、演进与排障

| 选择 | 更适合 | 主要成本 |
| --- | --- | --- |
| Local/Reducer | 有限组件/流程内状态 | 共享扩大时需调整Owner |
| Context | 跨层低频依赖与作用域注入 | 高频订阅与治理能力需自建 |
| Redux Toolkit | 明确动作、复杂共享规则、团队调试规范 | 概念与约定成本，需避免过度全局化 |
| Zustand | 简洁外部Store、细粒度订阅 | 领域结构、动作追踪和治理更多靠团队 |
| RTK Query/TanStack Query | 远端缓存、请求与失效 | key、权限、并发写和产品旧数据策略仍需设计 |

虚构后台：列表筛选放URL，订单数据放Query Cache，行展开放局部，跨页批处理向导用区域Store，主题用Context。已有RTK团队可用RTK Query减少两套机制，已有TanStack Query也无需为“统一”复制到Redux。库数量不是目标，同一事实只有清楚权威才是。

迁移从一类重复副本开始：列出所有读写者，选定权威缓存/Store，提供临时只读适配，逐个迁移，删去双写与旧副本，再启用约束。不要一次同时换路由、Store和数据协议，避免无法归因回归。

排障先区分：UI没更新查订阅与原地修改；重复请求查key与Client实例生命周期；旧结果查作用域与并发；输入卡顿查状态位置与selector输出；重登录串数据查持久化、缓存和迟到请求。测试覆盖未知动作、并发写失败、租户切换、离线恢复、刷新与前进后退、SSR请求隔离及迁移失败，不只测正常setter。

## 十、完整面试回答与追问

### 10.1 Redux还是Zustand？

“先分类状态。服务器记录进查询缓存，可分享视图进URL，输入局部保存，只有真正跨Feature的客户端事实才评估Store。复杂动作与调试治理要求高且团队熟悉Redux时选RTK；较轻共享可用Zustand，但同样要定义Owner、动作、重置和持久化边界。”

追问：Zustand不能做复杂业务吗？可以，复杂度不会因换库消失，区别是哪些约定由工具引导、哪些由团队自己建立，不应仅按代码行数决定。

### 10.2 为什么不要双存API结果？

“QueryCache已有请求身份、新鲜度与失效生命周期，再复制进slice会产生两套更新顺序和错误恢复。需要派生就selector，编辑就建立明确草稿与基线，不能把拷贝伪装成另一份权威。”

追问：草稿算重复吗？含义不同，可存在，但要明确服务器刷新如何与dirty字段冲突，提交用版本校验，成功重新对齐。

### 10.3 Context能替代Redux吗？

“小应用可以用Context加reducer持有状态，但Context原生负责跨层传值，不提供Redux整套动作中间件与调试，也不原生按字段selector订阅。低频依赖用它简单，高频复杂状态需衡量自建治理成本。”

追问：DevTools历史等于审计吗？不是，客户端不可信且历史不覆盖外部交易，正式审计应服务器持久记录，time travel不会撤销付款。

## 十一、扫雷自测与完整答案

1. **为什么先分类？** 远端权威、URL重现与输入草稿的生命周期不同，一套Store硬装会重复缓存与错误持久化。
2. **能派生的总数为何不再存？** 避免第二事实来源及同步遗漏；昂贵时memo selector即可，不改变权威。
3. **Context消费者只取一个字段会怎样？** 仍订阅整个Context值的变化，不自动变字段订阅，拆Context或外部Store可改善。
4. **Immer允许任意改store吗？** 不，只在相应draft reducer上下文便利表达不可变更新，外部原地修改仍错误。
5. **Zustand set会深合并吗？** 普通set顶层合并，嵌套结构需显式不可变更新，不能假定自动递归合并。
6. **staleTime与gcTime差别？** 一个控制新鲜判断，一个控制无活跃订阅后的缓存回收，过期不等于删除。
7. **QueryKey能放Token吗？** 不宜，会进入日志/调试，使用非秘密作用域标识并由服务器鉴权，身份变化清缓存。
8. **乐观更新失败直接恢复全部旧值安全吗？** 并发时可能覆盖后来成功修改，需操作级回滚或版本/重新拉取。
9. **URL仅初始化一次够吗？** 不，Back/Forward与外部导航会变URL，应以路由状态持续驱动视图。
10. **Persist整个Store有什么风险？** 保存敏感过期状态、恢复错误流程与schema不兼容；需白名单、版本、迁移和退出清除。

## 十二、速记、来源与验证范围

速记：**先派生再局部，URL管可重现视图，Cache管远端快照，Store管共享客户端事实，Context管作用域依赖；持久化只白名单，权限与交易仍在服务器。**

适用范围：React18+、RTK2.x、Zustand5.x常见API与TanStack Query v5。示例为浏览器模式；SSR需按请求隔离实例。Zustand文档站本次部分入口404，改用维护者官方仓库README核对，不以第三方文章代替一手资料。

- [React useContext](https://react.dev/reference/react/useContext)
- [Redux Toolkit createSlice](https://redux.js.org/toolkit/api/createSlice)
- [Zustand 官方README](https://github.com/pmndrs/zustand/blob/main/README.md)
- [TanStack Query v5 Important Defaults](https://tanstack.com/query/latest/docs/framework/react/guides/important-defaults)

示例仅人工核对状态、依赖和失败路径，未安装/运行Redux、Zustand、Query、React/SSR或浏览器集成测试，未做性能实验；不要将模型推理宣称为实际框架验证结果。
