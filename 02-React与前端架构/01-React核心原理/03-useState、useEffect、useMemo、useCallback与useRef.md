# 03. useState、useEffect、useMemo、useCallback 与 useRef

> 本章讨论客户端函数组件中的五种工具。普通示例适用 React 18+；核验日期为2026-11-22，官方当前文档标19.3。示例不依赖 React Compiler。目标是先保证状态与同步正确，再选择有证据支持的性能优化。

## 一、先区分事实、计算、同步与句柄

Render（渲染）根据 props、state、context 计算界面；Commit（提交）把选定结果应用到界面。React 可以重新执行或放弃渲染计算，所以渲染必须保持纯，不能在其中发付款请求或修改共享 Store。Hook 必须在组件或自定义 Hook 顶层按稳定顺序调用，不能放条件或普通循环，否则状态与调用记录无法稳定对应。

| 需求 | 首选 | 判断依据与反例 |
| --- | --- | --- |
| 用户输入等需要改变画面的本地事实 | useState | 改变后需要重新渲染；计时器句柄不必放入 |
| 可以由当前输入推导的值 | 普通计算 | 不能因“值会变化”就重复存 state |
| 与连接、浏览器监听等外部系统同步 | useEffect | 纯字符串拼接不属于同步 |
| 测量证实昂贵的派生计算 | useMemo | 不能靠缓存保住订单业务事实 |
| 消费者真正关心函数身份 | useCallback | 给普通按钮包一层通常无可见收益 |
| DOM、timer 等非画面可变引用 | useRef | 用户看到的计数不应只改 current |

例如购物车的商品数量是事实，总价可由数量与价格计算，窗口监听是外部同步，定时器 ID 是句柄。四者混放 state 或 Effect 会增加冗余渲染、循环更新和清理负担。服务器记录还涉及过期、权限和多组件共享，不能直接等同本地 state。

## 二、useState：下一次渲染的状态与更新队列

### 2.1 快照、替换和函数更新

useState 返回当前快照与 setter。调用 setter 是请求下一次更新，不会改变当前函数中的变量；它也不返回可等待的“DOM完成”Promise。日志紧跟 setCount 后仍读旧 count 是正常现象。

同一 count=0 的事件里，三次 setCount(count + 1) 都是在提交替换值1；三次 setCount(c => c + 1) 则依次从队列前态计算，得到3。Functional Update（函数式更新）适合下一值依赖前值，但仍可能捕获旧 step 等其他变量；它不是所有闭包问题的自动解法。

~~~jsx
import { useState } from 'react';

export function Basket() {
  const [items, setItems] = useState([]);
  function add(item) {
    setItems(current => [...current, item]);
  }
  return <button onClick={() => add({ id: 'demo', price: 10 })}>
    商品数：{items.length}
  </button>;
}
~~~

示例允许重复添加同一演示商品；生产应明确按行添加还是按商品合并数量。数组展开只复制外层，若修改某行对象，还需复制该对象以及变化路径，不能把旧 state 的嵌套对象原地改掉。

批处理（Batching）让多个更新合并处理以减少中间界面，并不赋予业务事务性；网络付款、库存和日志不会随 React 更新自动回滚。具体批处理边界与根模式和版本有关，React18现代根扩展了自动批处理范围，不能据此承诺任意异步任务只渲染一次。

### 2.2 初始化、相等与最小状态

useState(() => expensiveInitialValue()) 延迟到初始化时计算；写 useState(expensiveInitialValue()) 会先在每次组件执行中计算参数，即使 React 后续忽略它。初始化函数和更新函数必须纯，开发检查可能额外调用，不能在里面写入持久化或发送交易。

新旧值按 Object.is 判断相等时，React 通常可跳过相关更新工作；不要把这一优化理解为绝不调用组件。原地修改对象再传同一引用破坏快照，也可能看不到期待的变化。setter 身份稳定，依赖 lint 允许省略时可省略。

fullName 可由 firstName 与 lastName 拼出，不必独立存储再用 Effect 追平。重复事实会出现某字段已新、另一字段仍旧的瞬间。多个字段总要一起合法变化时，可合并一个对象或用 reducer：reducer 根据动作计算下一整体状态，便于表达“提交中不能再次开始”等规则，仍不能代替服务端事务。

传入新的 initial prop 不会重新初始化已有 state。编辑另一条记录时，若产品需要丢弃旧草稿，可用不同 key 明确重建；若需保留用户未保存输入，则应设计切换确认，而不是无条件 Effect 覆盖。存函数本身要用 setFn(() => actualFunction)，否则 setter 会把函数当更新器执行。

## 三、useEffect：管理一项外部关系

### 3.1 依赖与清理的时序

Effect 的 setup 在相应提交后运行。依赖改变，React 先执行旧 setup 返回的 cleanup，再建立新同步；卸载清理最后一次资源。旧清理闭包读取旧 ID，正是为了释放旧资源。

依赖包括 setup 使用的响应式值，即 props、state 和组件体内定义的相关变量、函数。React 逐项按 Object.is 比较，数组长度与位置应固定。省略数组通常每次提交后运行；[] 声明没有响应式依赖，不保证全生命周期宇宙范围只执行一次；[userId] 则跟随该输入变化。

开发 StrictMode 可进行额外 setup/cleanup 检查，真实卸载再挂载也会重建。用 didRun ref 屏蔽第二次运行可能在清理后失去资源，应保证建立和释放对称。Effect 不在服务端渲染中执行，不能依赖它填充服务端首屏数据。

useEffect 不保证所有情况总在 paint 后执行。必须在绘制前完成布局测量与修正时才考虑 useLayoutEffect，它会阻塞绘制，应保持短小。普通网络请求不因此需要布局 Effect。

### 3.2 完整查询示例：错误、取消和结果归属

原来的 loadUser(...).then(setUser) 只有取消动作，未处理 rejection，且取消未必阻止旧结果写入。下面教学组件假设 userId 是有效字符串，接口为同源只读 JSON 查询，浏览器支持 fetch 与 AbortController。

~~~jsx
import { useEffect, useState } from 'react';

export function UserPanel({ userId }) {
  const [result, setResult] = useState(null);

  useEffect(() => {
    let active = true;
    const controller = new AbortController();

    setResult({ id: userId, status: 'loading' });
    async function run() {
      try {
        const response = await fetch(
          '/api/users/' + encodeURIComponent(userId),
          { signal: controller.signal }
        );
        if (!response.ok) throw new Error('HTTP ' + response.status);
        const data = await response.json();
        if (active) setResult({ id: userId, status: 'success', data });
      } catch (error) {
        if (active) setResult({ id: userId, status: 'error' });
      }
    }
    run();
    return () => {
      active = false;
      controller.abort();
    };
  }, [userId]);

  if (!result || result.id !== userId || result.status === 'loading') {
    return <p role="status">加载中</p>;
  }
  if (result.status === 'error') return <p role="alert">加载失败</p>;
  return <p>{String(result.data.name ?? '')}</p>;
}
~~~

A 请求慢，切到 B 后旧清理把 active 置 false，A 迟到也不写回；id 校验避免新输入已经渲染但新 Effect 尚未运行时显示旧人的数据。AbortController 尝试停止客户端请求/读取，active 控制逻辑归属，两者作用不同。生产还需响应结构校验、错误分类、可观测和重试交互，示例将错误统一为可见失败而不泄露服务器细节。

Effect 回调不能直接 async，因为 async 返回 Promise，而 React 预期返回清理函数或无返回值。内部启动 async 并同步返回 cleanup 才有明确资源释放路径。对付款请求，abort 不保证服务器未执行，幂等键、查询与对账必须放业务层实现。

### 3.3 避免循环与不必要同步

Effect 设 state，state 又改变依赖，可能循环。不要立刻删依赖，应先确认是否可以直接派生。只供 Effect 用的 options 对象可在 setup 内创建，从真实基本输入声明依赖，避免每次 render 新对象引起重复连接。

监听要用对应回调和必要选项移除，timer 要 clear，连接要 disconnect；共享连接的所有权交资源层管理，单个组件清理不应断开所有订阅者。

服务器状态（Server State）通常适合框架加载器或查询库：以稳定键共享请求、缓存、失效和重试。库不会自动知道租户与权限边界，键必须覆盖相关作用域，退出登录需清除敏感缓存，重试须受次数与业务语义限制。简单隔离页面可用 Effect，规模扩大后应避免每个组件各自实现一套不兼容策略。

## 四、useMemo：缓存纯计算，不承载业务事实

useMemo(() => calculate(inputs), [inputs]) 缓存结果；依赖未变时复用，变化时重新计算。Memoization（记忆化）只改变重复工作的成本，去掉后程序结果仍应正确。依赖必须包含计算读取的响应式输入，否则得到陈旧结果。

~~~jsx
import { useMemo } from 'react';

export function Results({ products, query }) {
  const visible = useMemo(() => {
    const normalized = query.trim().toLowerCase();
    return products.filter(p =>
      p.name.toLowerCase().includes(normalized)
    );
  }, [products, query]);

  return <ul>{visible.map(p => <li key={p.id}>{p.name}</li>)}</ul>;
}
~~~

假设商品拥有稳定唯一 id 与字符串 name，products 由上游不可变更新。若原地改数组却不换身份，缓存无法据此发现内容变化。先把过滤当普通计算写正确，数据多且重复渲染中计算昂贵时再测量是否值得缓存；小数组可能维护成本超过收益。

useMemo 不会加速第一次计算，也不会把长计算移到后台。长过滤依旧占主线程，可考虑索引、服务端搜索、拆分工作或 Worker（独立执行线程）；列表 DOM 太多则需窗口化，只绘制可见行，单靠缓存过滤不解决布局成本。

不要在计算函数创建连接、改原数组排序或付款。sort 会改变输入，应复制后排序。缓存可能因开发编辑、初次挂起等原因被丢弃，不适合唯一业务 ID、资源所有权和必须持续存在的存储。缓存大对象也会延长保留时间，需权衡内存与 CPU，而非认为缓存永远省资源。

## 五、useCallback：函数身份必须有消费者

useCallback(fn, deps) 返回可复用的函数引用，不会执行 fn，也不是禁止 JavaScript 创建本次传入的函数表达式。其价值通常来自下游 memo 组件比较 props，或确实按函数身份管理的订阅接口。

~~~jsx
import { memo, useCallback, useState } from 'react';

const AddButton = memo(function AddButton({ onAdd }) {
  return <button onClick={onAdd}>增加</button>;
});

export function Counter() {
  const [count, setCount] = useState(0);
  const [theme, setTheme] = useState('light');
  const onAdd = useCallback(() => setCount(c => c + 1), []);
  return <section data-theme={theme}>
    <p>{count}</p>
    <AddButton onAdd={onAdd} />
    <button onClick={() => setTheme(t => t === 'light' ? 'dark' : 'light')}>
      换主题
    </button>
  </section>;
}
~~~

memo 尝试在 props 未变时跳过父级造成的子组件渲染；子自身 state、context 变化仍可更新。本例按钮很轻，主要用于展示身份链，不能宣称生产有显著收益。onAdd 用函数更新而不读取 count，故不需 count 依赖；若回调读取 productId，就必须声明它，不能追求稳定而提交旧商品。

若给 AddButton 又传每次新建的 options={{...}}，其他 prop 已变化，单独稳定 onAdd 可能无效。把函数传给普通 DOM 按钮、没有依赖身份的消费者或依赖每次都变时，通常没有跳过工作的收益。错误自定义比较器若忽略函数变化，还可能保留旧闭包。

与 useMemo(() => fn, deps) 可作概念对比，但优先用表达意图的 useCallback。二者都不是业务正确性的持久化机制；当前官方文档说明 React Compiler 可减少手动缓存需求，是否启用需看项目构建配置，不能假定升级 React 自动获得所有编译优化。

## 六、useRef：稳定容器、可变 current 与命令式边界

ref 对象在同一组件身份的多次渲染中保持，改 current 不请求渲染。适合保存 timer ID、第三方对象句柄和 DOM 引用。状态是显示事实，ref 是非 UI 的可变存储；在 ref 中存放画面计数会让视图与事实脱离。

~~~jsx
import { useRef } from 'react';

export function SearchInput() {
  const inputRef = useRef(null);
  return <>
    <label>搜索<input ref={inputRef} /></label>
    <button onClick={() => inputRef.current?.focus()}>聚焦搜索</button>
  </>;
}
~~~

React 在提交时设置 DOM 引用，节点移除时清空。不要在 render 中调用 focus，节点可能尚未存在且会产生副作用；用户动作的事件处理器是此例正确位置。TypeScript 可写 useRef<HTMLInputElement>(null)，需 JSX/DOM 类型支持；空值检查适应尚未挂载或已移除的状态。

ref 不会自动清理 timer 或第三方对象。Effect 建立 interval 时保存句柄并在 cleanup 清除；反复启动前也需明确是否先取消旧 timer。DOM引用被清空不等于外部订阅自动释放。

一般不要在 render 中读写 current 来表达业务逻辑，候选渲染可能丢弃，写入却污染共享可变容器。仅可预测的一次性初始化有特殊适用方式，也不能创建需要副作用清理的外部资源。想让回调读最新值时，先确认哪些变化应触发重同步，不能把真实依赖全部藏进 ref；权限最终必须由服务器判断。

## 七、工程决策与排障

从正确性开始：本地事实保持最小，派生在 render，事件触发业务操作，Effect 管外部关系，ref 保存句柄。随后用 React Profiler 检查哪些组件及计算耗时；Profiler 是测量渲染工作耗时与更新来源的工具，不等于完整网络与布局性能分析，还需浏览器性能工具定位主线程阻塞。

虚构场景：商品搜索输入卡顿。若是过滤慢，缓存可能帮助无关主题更新，但每次 query 真正变化仍重算；若是数万行 DOM，窗口化更重要；若网络请求过多，可加防抖减少请求，但必须保留响应归属校验。防抖延迟发起，不能保证已发请求按顺序完成。为每个函数加 useCallback 无法解决所有这些瓶颈。

验证应覆盖快速 A→B 查询、A迟到、失败、卸载、StrictMode清理再建立、重复点击和不可变更新。日志带必要请求标识而不记录令牌、完整个人数据。比较优化前后应使用相同数据、设备和构建；开发模式额外检查不能当作生产 benchmark。本章不声称任何固定提速倍数。

## 八、面试完整回答与深入追问

### 8.1 如何选择五个 Hook？

“先判断数据是否影响界面。需要持有的本地事实放 state，可以推导的直接算；外部同步用 Effect，并为每次建立设计清理；非界面句柄用 ref。昂贵计算和函数身份有测量证据时再加 useMemo/useCallback，先证明消费者能省掉工作。”

追问：总价是否放 state？通常不，因为可由当前商品与数量计算，重复存储增加一致性风险。若总价是服务器确认的报价，包含折扣版本和到期时间，那是独立业务事实，需要明确来源与失效规则。

### 8.2 useCallback 为什么常常没有收益？

“它稳定的是返回引用，不会免除函数表达式的构建，也不直接让子组件不渲染。只有消费者关心身份、其余条件也允许跳过工作，且节省大于比较与维护成本时才有用。”

追问：能否删除所有依赖保持稳定？不能，回调会捕获旧输入。用函数更新消除不必要的 state 读取是合法设计，删除仍然读取的 productId 只是制造错误。

### 8.3 Effect 查询怎样保证正确？

“每次查询关联输入身份，失败转为显式状态，清理取消等待并使旧结果失效，新界面不展示旧身份数据。取消不等于服务器回滚，共享缓存还需租户、权限、失效和重试策略。”

追问：付款也放 Effect 吗？用户交易由事件/业务层发起并实施幂等与确认，不能依赖挂载次数或清理取消保证只执行一次。

## 九、扫雷自测：答案与推理

1. **哪些数据不应放 state？** 可由当前输入直接计算的派生值和不影响 UI 的句柄。前者另存会形成双事实，后者每次改变触发渲染没有必要；服务器确认报价等独立事实另论。
2. **State 与 ref 决定性差别？** setter 请求新渲染并保留快照语义，current 可变但不通知 React。界面要响应的事实不能只改 ref。
3. **为何 set 后日志还是旧值？** 当前执行使用原快照；setter 排队请求后续计算，不原地改当前绑定，也不是 Promise。
4. **函数更新能修复所有旧闭包吗？** 不能，只给更新器队列前态。它捕获的其他 props仍可能旧，需决定快照或重同步语义。
5. **useMemo 能保存唯一订单 ID 吗？** 不适合，缓存可被丢弃且计算可能重复。ID应由明确业务创建流程和持久化层负责。
6. **useCallback 什么时候无收益？** 没有身份敏感消费者、其余props总变或子组件很轻时，省不掉足够工作，反增加依赖和内存成本。
7. **空数组是否只执行一次？** 不是，没有响应式依赖不排除开发检查及重新挂载。应保证 setup/cleanup 对称。
8. **为什么 abort 后还检查 active？** 取消未必阻止所有后续处理，active 控制结果是否仍属于有效同步；也需避免显示旧身份结果。
9. **为什么不在 render 聚焦或改 Store？** 候选计算可重做或丢弃，外部写入不回滚，破坏纯度与可预测性。
10. **缓存过滤为何仍卡顿？** 每次查询变化仍计算，长任务仍占主线程，DOM布局也可能是真瓶颈。按测量选择窗口化、索引或异步工作，不能把缓存当并行化。

## 十、速记、来源与验证边界

速记：**State持有事实，Render推导结果，Effect同步外部，Memo减少重算，Callback服务身份，Ref保存句柄；依赖真实、清理对称、旧结果不越界。**

五份官方文档分别核对接口、依赖、清理、纯度、缓存与ref语义，来源供追溯，不替代正文。普通示例以React18+客户端为边界；Compiler说明来自当前19.3文档，不假定项目已启用。

- [React useState](https://react.dev/reference/react/useState)
- [React useEffect](https://react.dev/reference/react/useEffect)
- [React useMemo](https://react.dev/reference/react/useMemo)
- [React useCallback](https://react.dev/reference/react/useCallback)
- [React useRef](https://react.dev/reference/react/useRef)

验证范围：本章JSX逐段人工核对输入、依赖、更新、清理和失败路径，未在React runtime或浏览器运行；未执行实际网络、DOM聚焦、StrictMode、Profiler或性能实验，不将教学推演当实测性能结论。
