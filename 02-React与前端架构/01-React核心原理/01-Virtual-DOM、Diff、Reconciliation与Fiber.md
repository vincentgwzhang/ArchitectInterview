# 01. Virtual DOM、Diff、Reconciliation 与 Fiber

> 本章以 React DOM 客户端函数组件为主，解释一次更新怎样形成界面、状态为什么保留或重置，以及调度能解决什么问题。公共行为按 React 官方文档核验；Fiber 字段、优先级编码和具体遍历方式属于内部实现，不能当稳定 API 使用。示例为教学代码，不是用户项目或性能基准。

## 一、先分清四个概念

| 概念 | 是什么 | 不是什么 |
| --- | --- | --- |
| React Element | 描述某个类型、属性与子节点的 UI 描述值 | 已挂载的组件实例或浏览器节点 |
| Virtual DOM | 对内存中 UI 描述及其比较机制的常用概括 | 浏览器 DOM 的完整复制或自动性能保证 |
| Reconciliation 协调 | 根据新描述与已有树的身份关系决定保留、更新或替换 | 任意树之间全局最少编辑操作算法 |
| Fiber | React 内部保存工作与组件关联信息的节点架构 | 多线程、Web Worker 或公开业务接口 |

JSX 是描述 UI 的语法，经构建工具转换为创建 Element 的调用。小写 div 等是 Host Type（宿主类型），由 React DOM 映射到浏览器节点；大写组件类型是函数等，由 React 调用计算子描述。其他渲染器可有不同宿主，因此不能把 React 的所有工作等同操作网页 DOM。

Element 通常是普通 JavaScript 对象，但应用应把它看成不可变、不可随意读取内部字段的描述。相同函数组件可在两个位置出现，各自有独立状态；函数每次返回新的对象，也不意味着旧实例必须重建。身份匹配才决定状态归属。

## 二、一次更新：Trigger、Render、Commit 与 Paint

### 2.1 触发与计算不等于页面已经改变

Trigger（触发）包括首次根渲染和状态更新；祖先更新也可能带来子组件重新计算，Context（上下文）变化等同样会影响使用它的组件。Props 通常是父组件渲染后传来的输入，并非存在一个自动监听任意对象字段的机制。直接修改普通变量或 props 对象不等于请求 React 更新。

Render（渲染计算）时，React 调用组件，用本次 props、state 与 context 计算下一份 UI 描述，并继续协调后代。一次 render 可以没有 DOM 变化：父组件重新执行但最终文字和结构没变，提交时无需重复改相同 DOM。也不能由 console.log 次数推断提交次数。

Commit（提交）把完成的结果落实到宿主环境，包括必要 DOM 修改、对应 ref 和 layout 工作。Paint（浏览器绘制）由浏览器完成，还涉及样式、布局、合成等成本。React render 很快不代表浏览器绘制快，大量复杂 DOM 和样式仍可能卡顿。

### 2.2 State 是本次渲染的快照

setState 请求后续更新，不会把当前事件处理器闭包中的变量立即改成新值。多次更新可能被批处理；如果后一次值依赖队列中的前一次，用函数更新表达关系。

~~~jsx
import { useState } from 'react';

export default function Counter() {
  const [count, setCount] = useState(0);
  return (
    <button onClick={() => {
      setCount(c => c + 1);
      setCount(c => c + 1);
    }}>
      {count}
    </button>
  );
}
~~~

点击一次最终增加二，函数更新按队列依次计算。若在同一次闭包里写两次 setCount(count + 1)，两次请求使用同一个快照值，通常只得到增加一。批处理降低重复工作，但不把网络、数据库或不同服务操作变成事务。

React 18 配合现代根 API 引入更广泛的自动批处理；具体何时 flush 不应作为业务时钟。读取更新后 DOM 的场景需使用合适生命周期或工具，不通过猜测固定毫秒等待来同步。

## 三、Diff：以可预测身份换取可接受计算量

### 3.1 启发式不是全局最优

React 用类型和同级身份等线索协调树。旧版官方协调文档用不同类型产生不同树、开发者给稳定 key 两个假设解释启发式线性比较；这是理解设计的模型，不承诺任意更新、所有组件计算和浏览器布局总共严格 O(n)。

原文“计算最小 Host Update”应理解为尽量复用并只落实必要变化，而非数学证明全局最小 DOM 编辑。比较本身、组件计算、分配和调度都有成本，直接精准改一个已知 DOM 属性可能更便宜；React 的价值还包括声明式组合和状态一致的编程模型。

### 3.2 同类型、不同类型与状态

同一父位置的同类型宿主元素通常复用节点并更新变化属性，再协调子节点。组件类型保持且身份相同，React 通常保留它关联的 state，再按新输入计算输出。属性变了不自动重置局部状态，useState 初始值也不是每次 props 改变都会重新应用。

类型改变通常替换对应子树并重置其中状态。例如外层 div 换成 section，内部同样写 Counter 也可能因父结构身份被替换而重新挂载。条件分支写在代码不同位置，不等于 UI 树位置不同；两个分支最终在同一位置都渲染 Counter 时可以保留状态。

组件函数定义应稳定。若在父组件函数体里重新定义子组件函数，每次父渲染都会创建新类型引用，子组件可能不断重置。把定义放模块顶层，按 props 传数据；不要用每次新建组件类型来模拟参数。

## 四、key：数据身份决定状态属于谁

### 4.1 正确作用域与选择

key 帮 React 在同一父节点的子列表中识别元素，通常用稳定数据 ID。它只需在对应兄弟集合唯一，不是整个应用全局唯一，也不是传给组件的普通 prop；组件需要 ID 时另传 item.id。相同 key 若类型不同仍不能视为同一组件，相同 key 移到另一个父节点也不保证保留。

~~~jsx
items.map(item => (
  <Row key={item.id} item={item} />
))
~~~

ID 应在创建数据时分配并保存，不在 render 时用 Math.random 或随机 UUID 生成。每次 key 变会使旧状态、焦点和编辑上下文丢失，并导致额外挂载清理。包装一层时 key 放在 map 直接返回的那层，不能藏在 Row 内部的 div 上。

### 4.2 Index key 怎样造成错位

旧列表为 A、B，索引 key 是0、1。用户在 A 的 Row 中把局部草稿改为 A-edited。头部插入 X 后，新列表 X、A、B 的 key 是0、1、2。React 按位置复用旧0给 X，旧1给 A，于是 X 可能拿到 A 的草稿；这不是数据本身被排序错，而是状态归属错。

受控输入即使文字来自 props 看起来正确，仍可能存在焦点、Effect 订阅或其他局部状态错配。不要以“页面文字暂时没错”证明 index 安全。

Index 在内容固定、不插删、不重排、不筛选且身份就是位置时相对可接受，例如静态说明条目。可编辑、排序、分页重用行或过滤列表优先真实 ID。没有稳定 ID 先改数据模型，不把警告清除当正确性完成。

### 4.3 主动重置也是合法设计

~~~jsx
function ChatPanel({ recipient }) {
  return <DraftEditor key={recipient.id} recipient={recipient} />;
}
~~~

切换收件人时主动重建编辑器，可避免上一个人的草稿误发；代价是未保存草稿丢失。若产品要求切回时恢复每个草稿，应把草稿按 recipient.id 存在父层或持久存储，再选择重置视图。key 不能代替业务数据保存或授权隔离。

列表虚拟化可能按窗口复用视图，需结合其 item key 和状态设计；稳定 React key 也不能保证滚出窗口被卸载的组件状态永远保存。组件生命周期与业务数据生命周期是两个问题。

## 五、可运行的身份模型：验证理解，不冒充 React 实现

下面纯 JavaScript 只演示在同一个父列表中按 key 与 type 保留草稿，故意省略 DOM、Hooks、优先级和完整协调规则。它不能用作生产 reconciler。

~~~javascript
function reconcileModel(previous, items, keyOf, type = 'Row') {
  const oldByKey = new Map(previous.map(row => [row.key, row]));
  return items.map((item, index) => {
    const key = String(keyOf(item, index));
    const old = oldByKey.get(key);
    return {
      key,
      type,
      id: item.id,
      draft: old && old.type === type ? old.draft : item.name
    };
  });
}
~~~

previous 的 draft 模拟 React 已保存的局部状态，items 是新输入。稳定 ID 插入后 A 保留 A-edited，X 初始化；索引 key 则把旧0草稿给 X。改变 type 或 key 初始化新草稿。模型假设 key 唯一；重复 key 不在其保证内，真实应用同样不应依赖重复 key 的结果。

本章实际运行此模型的稳定 ID、索引错配、类型改变、key 改变和重排检查；它只验证例子推理，不证明 React DOM 或 Fiber 行为。上面的 JSX 与后文 transition 示例未在浏览器运行。

## 六、Fiber：让工作可管理，而不是自动多线程

### 6.1 Fiber 保存什么，为什么有用

Element 描述想要什么，Fiber 节点关联当前组件身份、输入与状态、待处理更新和子树工作。以常见实现模型理解，节点之间有 child、sibling、return 等关系，便于把递归树工作组织成可以逐单元推进的过程。

current 树代表已提交结果，work-in-progress 表示准备中的候选工作，两者可通过内部关联复用信息。Render 中计算变化并标记待提交工作，完成后 Commit 发布结果。不要直接读取 Fiber 私有字段、复制 flags/lanes 数值或把某版字段布局写进业务逻辑；升级可能改变这些细节。

Fiber 的意义不只是“用链表代替递归”，而是把身份、状态、更新和可调度工作显式组织起来。它为优先级、暂停与重做提供基础，但不代表每次更新都会暂停，也不代表所有 React 功能都自动异步。

### 6.2 优先级与可中断的边界

Concurrent Rendering（并发渲染）允许 React 在准备较低优先级 UI 时让位于更紧急工作，暂停、重启或放弃尚未提交的 render。常见内部实现以 lanes 等表示工作类别，它们是内部调度机制，不是开发者指定任意优先级数字的公共协议。

浏览器主线程通常仍一次执行一段 JavaScript。React 可以在调度检查点让出执行权，但不能在你的单次同步长循环中任意抢占；一个组件计算连续堵塞几百毫秒，Fiber 不会自动把它分到其他 CPU 核心。拆组件、减少算法成本或将合适纯计算放 Web Worker，才可能处理这种瓶颈。

紧急输入可先更新，低优先级结果稍后完成，用户看到的是上次已提交 UI 与最终新 UI，而不是 React 任意展示半棵未完成候选树。但这不是跨根、外部 store、网络请求或浏览器绘制的通用事务保证。

## 七、Render 纯度与 Commit、Effect 的时序

### 7.1 被丢弃的计算不能留下业务副作用

Pure Render（纯渲染）要求在给定 props、state、context 下计算输出，不修改外部状态、输入对象或执行交易。React 可能重试或放弃 render，若组件体里发付款、推日志统计或订阅，就可能发生未显示却已执行或重复执行的行为。

局部创建并修改本次计算的新数组通常可以，因为没有改变外部可见输入；直接 props.items.sort 则修改调用方数据，应先复制再排序。时间、随机数等若影响输出，要按业务使用稳定输入或在合适的事件/生命周期获取，避免渲染不可预测和服务端客户端差异。

事件引发的付款放事件处理器或应用请求层，并采用服务端幂等；需要与外部系统保持同步的订阅放 Effect 并清理。Effect 也可能重新执行，不能把移到 Effect 理解为交易只会发生一次。框架支持的资源读取/Suspense 数据方案可能在 render 中读取缓存并挂起，不等于可在普通组件体任意发无管理请求。

### 7.2 Commit 的同步工作与绘制

React DOM 的提交中的 DOM 修改、ref 附着及 layout 工作不是可任意中断的后台 render。layout effect 在浏览器重新绘制前执行，可读布局并必要时同步修正；大量计算或连续布局读写会延迟绘制。使用 useLayoutEffect 应限于确实需要避免视觉闪动的测量等场景。

useEffect 的被动同步工作与 layout effect 不同，不应笼统说“所有 Effect 都在不可中断的同步 Commit 里”，也不能保证 useEffect 永远在 paint 后；交互等情况可能影响时机。两者都只在客户端运行，不在服务端产生 DOM 布局测量。

更新依赖变化时先清理旧 Effect 再设置新同步，卸载也清理；旧订阅、计时器或请求结果未处理可能污染新界面。网络请求清理可取消或忽略旧结果，但客户端取消不保证服务端业务操作撤销。资源和外部副作用仍需自己的失败语义。

### 7.3 StrictMode 是开发诊断，不是生产重复次数契约

StrictMode 在开发中增加渲染与 Effect/ref 清理等检查，以暴露不纯计算和清理缺失。不要承诺“所有组件任何情况永远只调用两次”，也不要据开发日志次数估生产性能。具体检查取决于边界与版本，目标是代码能正确面对重复准备和资源清理。

关闭 StrictMode 隐藏重复请求往往掩盖真实生命周期缺陷。先判断请求属于用户事件、外部同步还是框架数据资源，再解决幂等与清理。即便生产少见，也不能依赖 render 只执行一次。

## 八、Transition：优先保交互，而非减少总计算

下面为 React 18+ 可使用的客户端模式：输入值紧急更新，昂贵列表的筛选条件作为 Transition（过渡更新）。items 在真实应用应有稳定数据来源，SlowResults 定义在模块顶层。

~~~jsx
import { useState, useTransition } from 'react';

function SlowResults({ items, query }) {
  const visible = items.filter(item => item.name.includes(query));
  return <ul>{visible.map(item => <li key={item.id}>{item.name}</li>)}</ul>;
}

export default function Search({ items }) {
  const [text, setText] = useState('');
  const [query, setQuery] = useState('');
  const [isPending, startTransition] = useTransition();

  return (
    <>
      <input value={text} onChange={event => {
        const value = event.target.value;
        setText(value);
        startTransition(() => setQuery(value));
      }} />
      <p aria-live="polite">{isPending ? '正在更新结果' : '结果已更新'}</p>
      <SlowResults items={items} query={query} />
    </>
  );
}
~~~

不能把控制输入的 text 更新放进 transition，否则不能保持即时受控输入。startTransition 的回调立即执行，它标记其中状态更新的调度性质，不是延迟运行任意函数的工具；把巨大的 filter 直接放回调里同步计算，并不会自动后台化。

示例把结果放子组件以形成边界，但 text 的紧急更新仍可能重新调用未 memo 的 SlowResults。实际测得必要时可在稳定 items/query 下给该子组件 memo，使紧急更新跳过相同输入；即使如此，单次巨大 filter 仍会同步阻塞，需算法、索引、虚拟列表或 Worker。不能把这个教学例子当无条件加速保证。

Transition 可改善响应和允许丢弃过期工作，也可能增加被重做的计算；不会自动取消网络、减少数据量或解决异步返回乱序。异步数据需要请求标识、取消/忽略旧结果与缓存策略；此处只展示同步状态标记，避免把 React 不同版本的异步 Action 行为混为一谈。

## 九、性能与架构选型：先定位工作在哪里

### 9.1 区分四类成本

| 层面 | 典型问题 | 对应方向 |
| --- | --- | --- |
| Render 计算 | 大排序、重复遍历、广泛 Context 更新 | 状态靠近使用处、算法、合适 memo 与边界 |
| 协调及对象变化 | 大树反复创建、不稳定输入打破跳过 | 缩更新范围，稳定必要输入，不迷信所有缓存 |
| Commit 与布局 | 大量节点增删、layout effect 重活 | 虚拟化、减少 DOM、控制布局测量 |
| 网络与数据 | 请求风暴、旧结果覆盖、缓存无界 | 请求层去重、生命周期、权限与失效 |

memo 是可跳过相同输入计算的性能手段，不是状态正确性保证；新对象和函数引用可能让默认浅比较无法跳过，但“引用不稳定”不意味着 DOM 一定重建。Context 或组件自身 state 变化仍可能需要更新。useMemo 的缓存也不应承载不可丢失业务状态。

列表虚拟化减少同时挂载节点，改善 DOM 与部分计算，但增加高度、滚动、焦点、键盘和读屏支持复杂度。需要完整页面可访问与打印时考虑分页或其他呈现；不能单靠 transition 优先级解决十万 DOM 节点的内存与提交成本。

### 9.2 可复现的排查顺序

先以真实操作复现卡顿，明确输入延迟、首次显示或滚动问题。用 React DevTools Profiler 看提交与组件工作，浏览器 Performance 看主线程长任务、布局和绘制；不能把组件日志数量当耗时。对比合适生产构建，考虑开发检查与设备差异，不发布未经测量的“快十倍”。

状态错位优先检查 key、类型、父位置与意外卸载；性能问题再看为何触发、哪部分计算和提交昂贵。一次只改可解释因素，重复同场景观察响应、总 CPU、内存与正确性，不以 FPS 一个数字代替所有体验。

React 适合复杂交互中可组合的声明式状态管理；简单静态内容不因加 Virtual DOM 自动更快，内容重的页面可结合服务器输出与适量交互。DOM 安全仍要正确转义和控制危险 HTML，Virtual DOM 不是授权或防注入的通用边界。本章不深入服务端组件或 hydration，只提醒首次服务端输出与客户端身份/内容不一致另需检查。

## 十、面试完整回答与追问

### 10.1 Virtual DOM、Diff、Fiber 有什么关系？

“Element 是 UI 描述，Virtual DOM 是相关内存描述和比较模型的常用说法。Reconciliation 按位置、类型和 key 识别哪些状态与宿主节点可复用，并非全局最优编辑。Fiber 是内部关联状态与工作的节点架构，让 React 可以调度候选 render；完成后提交必要宿主变化，浏览器再布局绘制。性能要分别看计算、协调、提交和浏览器成本。”

**追问：Virtual DOM 比直接 DOM 快吗？** 不一定。比较和分配也花时间，对一个已知属性直接修改可以更便宜；React 提供可组合模型与合理复用，不提供对任意页面更快的保证。

### 10.2 为什么列表不用索引 key？

“索引表达位置，重排或插入后位置可以属于另一个业务项。React 可能保留位置对应的草稿、焦点或订阅给新数据，导致状态错位。稳定 ID 帮同级身份保持，但类型或父边界改变、节点卸载后仍可能重置。”

**追问：静态列表可以吗？** 若不重排插删且没有需要按数据保持的身份，索引相对可用；一旦可编辑、过滤或加入动态行为，重新评估。随机 key 更糟，会持续重置。

### 10.3 Fiber 怎样保证输入不卡？

“它支持紧急与较低优先级 render 的调度和让位，transition 可将结果更新放到后台准备。但 JavaScript 通常仍在主线程，单个长循环不会被任意抢占，提交和 layout 重工作也会阻塞。需要测量并缩小更新、改善算法、虚拟化或迁移适合计算到 Worker。”

**追问：render 为什么不能请求付款？** 候选 render 可重复或放弃，副作用可能未显示已执行或重复执行。付款应从明确事件经应用请求层发起并由后端幂等，不能依赖渲染次数保证交易。

## 十一、扫雷自测与推理答案

1. **Render Phase 为什么必须无外部副作用？** 工作可重试、暂停或放弃，组件体的外部写入不会随候选树自动回滚。纯计算允许安全重做，交易和订阅需正确事件/同步边界。
2. **类型改变时如何处理旧子树？** 对应身份通常替换、卸载并重建，原状态丢失；内部写相同组件名不抵消父类型变化。不要把新 Element 对象与新组件类型混为一谈。
3. **Index key 什么场景相对安全？** 固定、不增删筛选重排、身份本就是位置的列表。动态业务项应有稳定 ID，否则局部状态可能移给别人。
4. **Render 次数等于 DOM 修改次数吗？** 不等于。计算可被丢弃或输出相同，提交才落实宿主变化；浏览器 paint 又是另一步。
5. **相同 key 能跨任意父节点保留状态吗？** 不能，key 在对应父级子集合内识别身份，还要结合类型和位置。移父或卸载后不能仅靠字符串相同保证保留。
6. **useState(props.name) 会随 name 自动重置吗？** 不会，初始值用于初始化，相同身份后续保留状态。要按产品需求设计受控状态、显式同步或 key 重置，不能以为改 props 自动重建。
7. **useEffect 永远 paint 后吗？** 不能这样承诺。它与 layout effect 不同，但交互及更新时机会影响执行；layout effect 确实可能阻塞绘制，应少做重活。
8. **Transition 会让 filter 多核执行吗？** 不会。它标记状态更新，单次同步 filter 仍会阻塞主线程。真正并行需合适 Worker 或其他执行环境并处理通信成本。
9. **memo 能修正错误 key 吗？** 不能，它优化计算跳过，身份错误仍可能保留错状态。先正确性，再按测量决定优化。
10. **StrictMode 多一次检查是否生产 Bug？** 不是生产次数保证，而是开发诊断。重复暴露不纯或清理缺失应修原因，不能靠关闭检查证明正确。

## 十二、速记、版本与验证范围

速记：**描述不等于实例 → 身份决定状态 → 协调不求全局最优 → render 可重做 → commit 落实 → 浏览器绘制 → 调度改善响应但不消除工作。**

适用 React DOM 函数组件及 React 18+ 现代根和同步 transition 用法，官方当前页面在核验时标为19.3；不依赖19专属异步 Action 语义。旧版18.2协调文档仅用于启发式背景，不采用其中过时类生命周期写新代码。Fiber 是内部模型，不保证字段/API稳定。核验日期：2026-11-21。

验证：实际运行纯 JavaScript 身份模型检查；JSX、真实 DOM 状态/焦点、StrictMode、transition 调度和性能未在浏览器执行，不宣称实测加速。来源供可选追溯，正文已解释答题所需机制：

- [React：Render and Commit](https://react.dev/learn/render-and-commit)
- [React：Preserving and Resetting State](https://react.dev/learn/preserving-and-resetting-state)
- [React 18：Concurrent Rendering](https://react.dev/blog/2022/03/29/react-v18)
- [React：useTransition](https://react.dev/reference/react/useTransition)
- [React：useLayoutEffect](https://react.dev/reference/react/useLayoutEffect)
- [React：StrictMode](https://react.dev/reference/react/StrictMode)
- [Legacy React 18.2：Reconciliation](https://legacy.reactjs.org/docs/reconciliation.html)
