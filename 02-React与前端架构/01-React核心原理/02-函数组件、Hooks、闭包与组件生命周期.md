# 02. 函数组件、Hooks、闭包与组件生命周期

> 本章以客户端函数组件为主，建立“每次渲染的快照”和“外部同步的生命周期”两条线。常规示例兼容 React 18+；React 19 的特殊 use API 单独说明，不将内部 Hook 节点结构当公共保证。代码为教学示例，验证范围见末尾。

## 一、函数组件每次 Render 都是一次独立执行

Function Component（函数组件）由 React 调用，根据 props、state 和 context 计算 UI。一次 Render（渲染计算）会建立新的局部变量和函数闭包，但 React 根据组件身份保留关联的状态；不是每调用一次函数就把 state 重置，也不是一个永远变化的 this 对象。

Snapshot（快照）表示本次执行看到的输入与状态值。setState 请求后续渲染，不会修改当前执行里的 state 绑定。Event Handler（事件处理器）也是本次 UI 输出的一部分，旧处理器和异步回调仍可保留旧执行的环境。这是 JavaScript Closure（闭包）的自然结果，不是 React 将所有变量深拷贝后冻结。

对象快照尤其要小心：闭包可持有同一对象引用，直接修改对象会破坏不可变输入假设。更新数组、对象时创建新结构，渲染只计算结果；不要在组件体里改外部 Store、props 或发送交易。候选渲染可能重做或丢弃，外部写入不会随之自动回滚。

状态是否保留还与类型、父位置和 key 有关。切换 key 会重建相应状态；组件函数内定义新的组件类型也可能造成意外重置。本章重点是状态保留后的快照与同步，不能把所有闭包问题归咎于渲染次数。

## 二、Closure 与 Stale Closure：先确定想读哪一次值

### 2.1 什么是闭包

函数能够访问创建它时所在作用域的变量，即使外层执行已经结束。每次调用外层函数会创建不同环境；重新渲染不会改写旧环境里的局部绑定。

~~~javascript
function makeHandlers(count) {
  return {
    read: () => count,
    incrementFromSnapshot: () => count + 1,
    incrementFromCurrent: current => current + 1
  };
}
~~~

makeHandlers(0) 与 makeHandlers(5) 创建不同闭包，旧 read 仍返回0。incrementFromCurrent 的参数由调用者在执行时传入，它只解决“下一值依赖当前值”，不会让所有其他捕获变量自动变新。

Stale Closure（陈旧闭包）是程序需要最新值，却仍使用旧环境的错误。旧值有时正是业务要求，例如延迟发送用户点击时已确认的订单金额；不能无差别地把所有回调改成读“最新”，否则点击后换了收件人可能发给错误对象。

### 2.2 延迟计数与 Functional Update

原例可以写为：

~~~jsx
import { useState } from 'react';

export default function Counter() {
  const [count, setCount] = useState(0);

  function incrementLater() {
    setTimeout(() => {
      setCount(current => current + 1);
    }, 1000);
  }

  return <button onClick={incrementLater}>{count}</button>;
}
~~~

若同一快照 count=0 下排了三个 setCount(count + 1)，回调都请求替换为1，最终可能只有1；三个函数更新分别基于更新队列的前值计算，得到3。这里延时1000ms不是准确执行时刻，繁忙事件循环可能更晚。

Functional Update（函数式更新）适合下一状态依赖上一状态，更新函数必须纯，不能在里面付款、改全局变量或埋点，因为 React 可为检查或重新计算调用它。若还捕获 step 或 recipient，函数式更新并不自动修正那些旧值；要分别决定快照语义或同步方式。

此最小计数例为清楚展示更新语义，未跟踪卸载取消。生产若延迟动作只在当前组件存在时有意义，应保存 timer 句柄并清理，或把任务生命周期交给更合适层；不要让用户离开后仍执行意外业务动作。

## 三、Hooks 怎样关联状态与调用规则

### 3.1 稳定顺序的原因

useState、useReducer、useRef 和 useEffect 等常规 Hooks 依赖每次渲染中稳定的调用顺序关联相应状态与同步记录。可用“每个组件有有序槽位”理解：第一次依次调用 name 与 count，下一次若跳过 name，原 count 的位置就不再对应。具体内部可能用链式记录等，不是公开可操作的数组。

因此不能放条件、循环、事件处理器、普通嵌套函数、useMemo 回调或条件提前 return 后。修复时把 Hook 放顶层，把条件放入 Effect 内，或拆出按条件挂载的组件。后者的卸载会清理并丢弃局部状态，需确认产品是否允许。

~~~jsx
function Room({ roomId, enabled }) {
  useEffect(() => {
    if (!enabled) return;
    const connection = connect(roomId);
    return () => connection.disconnect();
  }, [roomId, enabled]);
  return null;
}
~~~

此片段需从 React 导入 useEffect，并由应用提供 connect；假设 connect 同步返回拥有 disconnect 的连接对象。Hook 每次都调用，但 enabled 决定是否建立外部同步。若真实连接是异步，清理还需处理建立完成晚于卸载的情况，不能直接套此接口假设。

### 3.2 只能从 React 函数调用，工具帮助守规则

常规 Hook 只在函数组件或 Custom Hook（自定义 Hook）顶层调用。自定义 Hook 命名以 use 开头，组件由 React 以 JSX 方式调用，不把 Component() 当普通工具函数混入调用顺序。使用官方推荐的 Hooks lint 规则检查调用和依赖，不能随手关闭警告。

React 19 的特殊 use(resource) API 是需单独知道的例外：可在条件和循环中读取相应资源，但仍有自身约束，不能据此允许 useState/useEffect 条件调用；use 也不能放 try/catch 捕获挂起。本章普通 Hook 代码不依赖此API，React 18 项目不可照搬19能力。

## 四、选择 state、ref、派生值与共享状态

| 需要 | 选择及原因 | 陷阱 |
| --- | --- | --- |
| 值变化需驱动 UI | state/reducer，通知 React 重新计算 | 直接改对象后不请求更新 |
| 可从当前输入直接得到 | Render 派生计算，必要时缓存昂贵计算 | Effect 再设 state 导致额外渲染与不同步 |
| 句柄等不参与 UI 的可变值 | ref，跨渲染保留对象但改 current 不触发渲染 | 用 ref 当屏幕状态导致页面不更新 |
| 多组件共享同一事实 | 提升状态、Context 或合适外部 Store | 误以为调用相同 Hook 就共享状态 |
| 同步 React 外的可变 Store | 合适订阅协议，如 useSyncExternalStore | render 随意读写造成不一致 |

例如 fullName = firstName + ' ' + lastName 直接在 render 得到，不需要 Effect 再 setFullName。重复存储派生值引入两个事实来源，可能短暂显示旧值。useMemo 是计算缓存优化，不应成为必须永久保存的业务状态。

useRef 返回在同一组件身份下稳定的容器，current 可保存定时器或非 UI 对象。读“最新值”时要明确何时更新和何时读取，不能每次 render 随意写外部可变对象以绕过并发模型。事件或提交后的同步常是更清楚的位置；依赖需要响应变化时仍应声明，而不是藏进 ref。

外部 Store 的快照读取和订阅必须一致，useSyncExternalStore 提供相应接入协议；getSnapshot 的结果需遵守稳定性/缓存要求，不能每次无变化也返回新对象导致反复更新。这里不展开完整 Store 实现，但不能把“render 中改 Store”当同步方案。

## 五、Effect 的生命周期：同步一项外部关系

### 5.1 Setup、Cleanup 与依赖变化

Effect 不应机械等同 componentDidMount、componentDidUpdate、componentWillUnmount 的语法替换。组件有挂载、更新和卸载，某条外部关系则可能多次启动和停止。例如房间连接只应随 roomId 等真实输入变，而不是随所有重渲染重连。

Setup（建立同步）在相应提交后运行；依赖改变时先用旧闭包执行 Cleanup（清理），再用新输入 setup；卸载也清理最后一次同步。旧清理读取旧 roomId 正是为了断开旧连接，不是需要修掉的 stale bug。

被暂停或丢弃且未提交的 render 不应擅自建立外部连接。Effect 让同步与已提交 UI 关联，但不能自动赋予外部操作“只执行一次”的交易保证。订阅清理和幂等需要应用自己实现。

### 5.2 Dependency Array 的精确定义

依赖数组声明 setup 中使用的 Reactive Values（响应式值），包括组件内 props、state 与本次定义的变量/函数。React 逐项使用 Object.is 比较；数组长度和顺序应固定，不能运行时随意拼装。

省略数组通常每次提交后同步；空数组表示没有响应式依赖，不等于宇宙范围只运行一次；[roomId] 表示随其身份变化重同步。StrictMode、卸载再挂载等仍会建立与清理。依赖是正确性描述，不是为了减少运行次数随便选的性能开关。

如果对象和函数每次 render 新建，即使内容相同，引用改变也会触发依赖变化。优先把只供 Effect 使用的对象/函数放其内部，或在模块层定义真正不依赖输入的值；必要时才用 useMemo/useCallback 稳定身份。不能删除依赖来维持一次连接，同时偷偷读取会变的 roomId。

无限循环常由 Effect 设置状态，该状态又使依赖改变。例如每次创建 options 对象且 Effect 因它重新设状态，应先问是否根本不需要外部同步，再修依赖身份。加空数组压住循环可能只是变成错误旧值。

### 5.3 useEffect、useLayoutEffect 与服务端

useEffect 用于一般外部同步，不应假定所有情形永远在浏览器 paint 后；交互等时机会有不同。useLayoutEffect 在重新绘制前执行布局相关同步，适合必须测量后修正以避免闪动的场景，但阻塞绘制，不能塞大计算或长网络等待。

两类 Effect 都不在服务端渲染时建立客户端连接或测 DOM。需要首屏数据时，框架数据加载或合适资源层往往比挂载后请求更好，可以减少瀑布并处理缓存与服务端输出。Effect 获取数据是可用机制，但不是所有应用最合适的数据架构。

## 六、Cleanup：资源、异步结果和失败语义

### 6.1 资源必须对称管理

Listener、Timer、连接和 Observer 的 setup 有相应清理：用同一回调/必要选项移除监听，clearTimeout/clearInterval 清定时器，断开旧连接。清理只处理本次拥有的资源，不要笼统删除全局共享资源影响别的组件。

依赖变更也是清理时机，不能仅写“退出页面才清”。一个漏掉 roomId 依赖的订阅会继续收旧房间信息，即使组件仍挂载，也构成正确性甚至隐私风险。共享连接由专门资源层管理引用或订阅生命周期，单组件不要擅自关闭所有人的共享通道。

Effect 回调不能直接写 async 函数返回 Promise，因为 React 需要的是清理函数或不返回；在内部定义 async 流程并显式处理取消、错误与结果归属。异步建立连接尤其要处理“卸载后才连接成功”，成功后发现已失效应立即释放。

### 6.2 查询竞态：取消与忽略旧结果配合

以下自定义 Hook 为最小教学查询器，假设浏览器 fetch、相对同源只读接口返回 JSON；生产还需要校验响应模式、权限和统一请求策略。这里用 ignore 标记阻止旧查询结果写回，并用 AbortController 尝试取消网络。

~~~jsx
import { useEffect, useState } from 'react';

export function useUser(userId) {
  const [result, setResult] = useState({
    id: null, status: 'idle', data: null, error: null
  });

  useEffect(() => {
    let ignore = false;
    const controller = new AbortController();

    setResult({ id: userId, status: 'loading', data: null, error: null });

    async function load() {
      try {
        const response = await fetch(
          '/api/users/' + encodeURIComponent(userId),
          { signal: controller.signal }
        );
        if (!response.ok) throw new Error('HTTP ' + response.status);
        const data = await response.json();
        if (!ignore) {
          setResult({ id: userId, status: 'success', data, error: null });
        }
      } catch (error) {
        if (!ignore) {
          setResult({ id: userId, status: 'error', data: null, error });
        }
      }
    }

    load();
    return () => {
      ignore = true;
      controller.abort();
    };
  }, [userId]);

  // 新输入已 render、旧 Effect 尚未清理时，不显示别人的旧结果。
  return result.id === userId
    ? result
    : { id: userId, status: 'loading', data: null, error: null };
}
~~~

A 请求慢、用户切到 B 后 B 先完成，A 迟到时旧 Effect 的 ignore 已置 true，因此不覆盖 B。即使 abort 未能阻止服务器或某层缓存继续处理，ignore 仍限制旧结果写入；id 校验还避免新输入首次 render 立即显示旧人的数据。userId 应是有效稳定字符串，组件调用规则仍需遵守。

清理取消客户端等待不保证服务器没执行。查询可忽略结果，付款等业务动作则应在事件/业务层用幂等键、查询与对账，不因组件卸载就假定事务回滚。请求缓存如果跨租户、权限或环境复用，缓存键和清理还需包含这些边界；这个 Hook 没有提供缓存、去重、重试或共享数据能力。

## 七、Stale Closure 常见成因与正确修法

| 场景 | 为什么旧 | 优先处理 |
| --- | --- | --- |
| 空依赖 interval 读 count | 回调一直是首次环境 | 下一状态依赖前值可用函数更新，仍清理 timer |
| 订阅回调读旧 roomId | Effect 漏依赖未重同步 | 声明依赖并正确断开旧连接 |
| 旧搜索结果覆盖新查询 | 异步完成顺序不等于发起顺序 | 取消/忽略旧结果、关联请求身份 |
| useCallback 依赖漏字段 | 稳定引用把旧环境一直留下 | 补真实依赖，先看是否需要缓存 |
| 延迟提交读错收件人 | 未区分点击快照和最新业务意图 | 明确语义、保存确认输入并服务端验证 |

函数式更新不是通用最新值读取器，ref 也不是依赖逃生门。若同步关系确实应随值改变，就重同步；若仅回调某部分需要读最新值而不重连，使用明确设计的方式，并检查项目版本支持的 API。当前 React 官方文档中的 useEffectEvent 可表达某些非响应式 Effect 逻辑，但不能拿来隐藏真正依赖或在事件处理器任意调用；本章代码不依赖它。

闭包本身并非泄漏。只有它被 timer、listener、Promise 等继续引用时，环境和相关对象才可能继续保留；清理外部注册通常比无差别加 memo 更有效。内存问题用工具看实际保留链，不能只凭函数每次新建认定泄漏。

## 八、Custom Hook：共享逻辑不等于共享状态

useUser 被两个组件调用，会各自拥有状态与 Effect，可能发两次请求。相同函数名不使 React 自动合并资源。要共享同一事实，可提升到共同父级/Provider，或使用明确缓存与订阅层，再让 Hook 接入它；共享层需处理过期、权限、租户隔离和取消责任。

Custom Hook 接口应表达业务同步含义，例如 useRoomConnection(roomId)，比 useMount(callback) 更能保留依赖和重同步语义。隐藏所有依赖的通用生命周期封装，容易迫使调用者关闭 lint 或保留旧值。只做纯计算的 helper 不需要命名 use；真正调用 Hooks 的函数应守相同规则。

不要一次塞进连接、标题、分析事件等互不相关资源，使一个变化重建全部同步。按独立外部关系拆 Effect，既容易清理也便于测试；需要原子一致的业务操作则不靠多个 Effect 的执行先后猜测来协调，放到明确应用状态机或事务层。

## 九、StrictMode 与生命周期迁移

开发 StrictMode 会额外检查渲染、Effect 和 ref 清理，帮助暴露不纯和资源未释放。完整根边界下常见 setup→cleanup→setup 压力检查，正确逻辑应等价于一份有效订阅，而不是两份泄漏。具体检查受版本和边界影响，不承诺固定次数，也不代表生产每次都重复同样次数。

不要用 didRun ref 强行跳过第二次 setup：清理后跳过重建可能留下没有连接的组件，也会掩盖真实卸载再挂载问题。若外部动作本就不可重复，先判断是不是应该由用户事件或服务端幂等控制，而不是关闭诊断。

迁移 class 时，按外部关系重组：同一个连接原来分散在 mount/update/unmount 的逻辑可合并为一个随依赖同步的 Effect；与连接无关的页面标题另一个 Effect。不要把一切原 mount 动作塞进 []，否则新 props 变化时不正确。

React 内部能够重新准备 UI，组件函数保持纯，Effect 与提交后的状态同步，两者职责不同。Hooks 不要求把 UI 模型改写成“每次变化后执行脚本”；能直接由状态派生的 UI 仍应声明式计算。

## 十、架构落地、排障与验证

先分类工作：纯派生在 render，用户动作在事件/应用层，外部订阅在 Effect，需要共享与缓存的服务器数据在合适资源层。这样减少重复请求、冗余 state 与循环同步，降低维护成本；不能为了“抽象统一”把所有业务都放一个大 Hook。

排障按因果顺序：确认组件是否被 key/type 重建，确认本次输入和提交，列回调捕获值与存活来源，再看依赖与清理，最后验证异步完成顺序。暂时日志可带请求 ID 和资源 ID，但避免泄露个人内容或令牌；闭包导致的旧租户订阅不是只影响性能的小事。

测试应覆盖依赖 A→B、旧结果迟到、错误、卸载、重复 setup/cleanup、两个独立调用与快速连续更新。假 timer 能验证排队语义但不代表真实浏览器网络和调度；测试环境也可能不复现并发准备，生产兼容性需在实际根与构建中验收。

本章用纯 JavaScript 模型实际验证闭包快照与函数更新，其余 JSX 通过人工路径检查，未浏览器执行。不要由模型测试通过宣称 React DOM 生命周期已实测。

## 十一、面试完整回答与追问

### 11.1 每次 render 都有闭包是什么意思？

“组件每次执行看到一份 props/state，生成的回调捕获该执行环境。setState 请求下一次计算，不改变旧变量，所以延迟回调可能读旧值。先确定业务要点击时快照还是最新值，计数依赖上一状态用函数更新，订阅随依赖重同步，异步结果用取消或身份校验防乱序。”

**追问：函数更新能解决全部旧值吗？** 不能，只让该更新拿到队列前态，仍可能捕获旧 step、收件人或权限，需分别设计。权限最终由服务端验证。

### 11.2 Effect 与 class 生命周期有什么区别？

“Effect 更适合同步一项外部关系：提交后建立，依赖变时清旧再建新，卸载清理。它不是三个 class 方法逐个换语法。依赖声明同步使用的响应式输入，纯派生不必 Effect，用户交易不应依赖挂载次数。”

**追问：[] 是否就是只执行一次？** 不等同，开发检查和真正重新挂载仍会执行。应证明 setup/cleanup 对称与重复安全，而不是用 ref 屏蔽再次运行。

### 11.3 自定义 Hook 复用了什么？

“它复用 stateful logic，即状态与外部同步的组织方式；每次调用的状态默认独立。共享数据要有共同持有者或缓存订阅层，Hook 只提供接入接口，同时处理权限、资源释放与错误。”

**追问：两个 useUser 会自动去重吗？** 本章简单实现不会，会各自发请求。要去重需明确缓存键、请求共享、过期与取消策略，不能把同名函数当全局单例。

## 十二、扫雷自测与完整答案

1. **常规 Hooks 为何不能条件调用？** React 按稳定顺序关联状态与同步记录，跳过一项会改变后续对应。把条件移入 Effect 或拆组件；React19特殊 use 的例外不适用于 useState。
2. **Stale Closure 怎样产生？** 旧回调仍被外部资源保留，而逻辑需要新值。漏依赖、稳定缓存或异步乱序都可能造成，修复要匹配语义，不一律用 ref。
3. **StrictMode 重复 Effect 有何价值？** 暴露资源未清理、setup不可重建等缺陷。正确设计面对清理再启动仍有一份有效关系，不靠关闭检查隐藏问题。
4. **三个延迟 count+1 为什么可能只有1？** 它们都捕获 count=0，均请求替换为1；三个 current=>current+1 按队列前态分别增加，得到3。更新函数仍须纯。
5. **Cleanup 读旧 roomId 是 bug 吗？** 通常不是，它要释放旧连接；新 setup 读新值。把清理强行改为最新 ID 可能关错资源。
6. **能在 Effect 里保存 fullName 吗？** 可以写但通常多余，它能由当前输入派生，另存会增加渲染和一致性风险。只有真实外部同步才需要 Effect。
7. **AbortController 保证服务端没执行吗？** 不保证，只控制客户端相关请求/读取。交易需幂等和业务确认，查询仍需防旧结果写回。
8. **ref.current 变化会更新画面吗？** 不会自动触发渲染。UI事实用state，句柄等非UI可变值用ref，不能藏依赖绕过同步。
9. **async Effect 为什么不对？** async 函数返回Promise而不是清理函数，React不能把它当预期资源清理。内部启动异步流程并同步返回cleanup，处理错误与迟到结果。
10. **自定义 Hook 是否共享 state？** 默认不共享，各调用关联各自组件状态。共同数据持有者或显式共享资源层才带来共享，需另设计权限和生命周期。

## 十三、速记、版本与核验来源

速记：**每次执行一份快照 → 闭包保留环境 → 常规 Hooks 顺序稳定 → 纯派生/事件/同步分工 → 依赖变先清旧再建新 → 迟到结果不越界 → 共享逻辑不等于共享状态。**

核验日期：2026-11-21，官方当前文档页标19.3。普通 useState/useEffect 例适用 React18+，use 属于 React19 API；useEffectEvent 的说明以本次核验的19.3文档为准，不能假定所有19.x版本均支持，应按具体版本与 lint 支持核对，本章未采用它们实现代码。模型测试不等于 React runtime/浏览器集成验证；未执行真实网络、DOM、StrictMode 或性能测试。

- [React：State as a Snapshot](https://react.dev/learn/state-as-a-snapshot)
- [React：Rules of Hooks](https://react.dev/reference/rules/rules-of-hooks)
- [React：useEffect](https://react.dev/reference/react/useEffect)
- [React：Reusing Logic with Custom Hooks](https://react.dev/learn/reusing-logic-with-custom-hooks)

- [React：use（特殊调用规则）](https://react.dev/reference/react/use)
- [React：useEffectEvent（本次核验19.3文档）](https://react.dev/reference/react/useEffectEvent)
- [React：useSyncExternalStore](https://react.dev/reference/react/useSyncExternalStore)
