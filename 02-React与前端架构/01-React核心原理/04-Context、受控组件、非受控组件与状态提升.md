# 04. Context、受控组件、非受控组件与状态提升

> 状态架构先确定谁拥有事实、谁提出修改、谁需要订阅，再选择传递方式。本章示例使用兼容 React 18+ 的客户端写法；2026-11-22 核验的官方文档标19.3，React19 Provider 简写另说明。案例均为教学场景，不代表个人项目。

## 一、状态所有权与 Single Source of Truth

Single Source of Truth（单一事实来源）意味着同一作用域中的某项事实有明确权威持有者。其他组件读取它或请求修改，不各自保存一份并用 Effect 互相追平。它不要求应用所有状态集中到一个根组件，也不禁止有不同含义的草稿和服务器记录。

例如两个面板要求最多展开一个。每个面板各有 isOpen，允许出现两个 true，已经无法自然表达约束。公共父级保存 activeId，子级只接收是否展开与打开请求，约束由表示方式保证。Lifting State Up（状态提升）就是把需要协调的事实移至最近共同所有者。

先问状态属于哪个页面、实体和会话。一个弹窗的输入通常不属于应用根；两个兄弟共享筛选条件才提升到共同父级。提升太高扩大依赖范围、延长生命周期，页面离开后草稿可能意外保留；放太低又需跨组件同步。最近公共父级是起点，业务生命周期与路由边界可进一步决定位置。

~~~jsx
import { useState } from 'react';

function Panel({ title, open, onOpen, children }) {
  return <section>
    <h2>{title}</h2>
    <button type="button" aria-expanded={open} onClick={onOpen}>
      展开
    </button>
    {open && <div>{children}</div>}
  </section>;
}

export function Details() {
  const [activeId, setActiveId] = useState('profile');
  return <>
    <Panel title="资料" open={activeId === 'profile'}
      onOpen={() => setActiveId('profile')}>资料内容</Panel>
    <Panel title="账单" open={activeId === 'billing'}
      onOpen={() => setActiveId('billing')}>账单内容</Panel>
  </>;
}
~~~

此例始终有一个逻辑选中项，不支持全部关闭；若需求允许，使用 null 表示无展开。子内容因条件渲染会卸载，里面的局部 state 也会丢失；若关闭时要保存草稿，应把草稿放在生命周期更长的所有者，或保留挂载并正确处理隐藏内容的可访问性。不能只讨论“开关状态正确”而忽略内容状态。

## 二、受控组件：值与修改请求分离

### 2.1 数据流与输入契约

Controlled Component（受控组件）的重要行为由外部 props 决定。对原生文本输入，value 来自 React state，onChange 同步保存用户输入；下一次渲染再次用 value 定义显示值。自定义日期选择器也可用 value/onChange 表达相同契约，不限于 input。

~~~jsx
import { useState } from 'react';

export function EmailEditor() {
  const [email, setEmail] = useState('');
  const [touched, setTouched] = useState(false);
  const invalid = touched && email.trim() === '';
  return <div>
    <label htmlFor="email">邮箱</label>
    <input id="email" name="email" type="email"
      value={email}
      onChange={e => setEmail(e.target.value)}
      onBlur={() => setTouched(true)}
      aria-invalid={invalid}
      aria-describedby={invalid ? 'email-error' : undefined} />
    {invalid && <p id="email-error">请输入邮箱</p>}
  </div>;
}
~~~

例子只演示必填提示，type=email 可提供浏览器格式检查，但既不能证明邮箱属于用户，也不能代替服务端校验。touched 表示已离开输入，用来避免刚打开页面就报错；错误可从当前值计算，不必另存一份 error 并用 Effect 追平。多个实例需生成唯一id，不能原样复用固定id导致标签关联错误。

文本值从 '' 开始，不能先 undefined 再变字符串，否则可能从非受控切为受控。复选框使用 checked 布尔值和 e.target.checked，不能读 value 判断选中。只传 value 而不更新会让输入看起来无法编辑；真正只读应明确 readOnly。

### 2.2 验证、输入体验与性能

受控适合字段互相影响、实时预览、条件 UI、统一重置与提交状态。用户每次输入触发 state 更新本身是正常机制，不等于必然性能差；问题通常是把输入状态放得太高，连带昂贵树或计算。先拆分字段与消费者，再测量是否需表单库的字段订阅。

保持输入原始字符串，尤其是数字字段：空字符串、负号和小数点可能是合法中间编辑态，过早转换 Number 会把空值误变0。输入时频繁大写化、trim 或异步回填还可能造成光标跳动与中文输入法组合中断。优先在提交/失焦时规范化，输入期间让用户完成表达。

不要防抖保存受控 input 的即时值：React仍拿旧 value 回写可能妨碍输入。应立即存草稿，防抖的是搜索、自动保存等昂贵外部操作。迟到校验结果需关联当前值或请求序号，不能把旧邮箱的“可用”提示套到新邮箱。前端禁用提交是交互保护，不能保证服务器不收到重复请求。

## 三、非受控组件：DOM 持有当前编辑值

### 3.1 defaultValue、ref 与 FormData

Uncontrolled Component（非受控组件）把输入当前值留在 DOM，React给初始值，在提交等边界读取。defaultValue 表达初始内容，不是后续每次渲染强制覆盖；换了 defaultValue 不应被当作可靠的“切换记录并重置草稿”协议。

~~~jsx
import { useState } from 'react';

export function SimpleContact() {
  const [submitted, setSubmitted] = useState('');
  function handleSubmit(event) {
    event.preventDefault();
    const data = new FormData(event.currentTarget);
    const email = String(data.get('email') ?? '').trim();
    setSubmitted(email);
  }
  return <form onSubmit={handleSubmit}>
    <label>邮箱<input name="email" type="email"
      defaultValue="" required /></label>
    <button type="submit">读取表单</button>
    <output>{submitted ? '已读取：' + submitted : ''}</output>
  </form>;
}
~~~

此例只读取并显示，没有发网络请求。FormData 依赖控件的 name；未勾选复选框、禁用字段等不会按普通文本方式出现，同名多值需 getAll，不能盲目 Object.fromEntries 后丢掉重复项。ref.current.value 可读单个 DOM输入，但修改ref不通知React；要实时展示联动值，需订阅输入或选择受控。

初次值来自服务器时，要区分尚未加载与空值，必要时加载完成后再挂载编辑器。切换记录用 key 重建会清掉未提交内容，须符合产品预期；form.reset 可恢复原生控件默认值，但不会同步重置独立React状态或外部widget。受控表单应由其状态所有者执行重置。

### 3.2 适用场景与边界

简单提交型表单、浏览器原生行为、文件输入、第三方编辑器可适合非受控。文件选择通过 FileList/FormData 读取，不能把任意本地路径写入 value 让浏览器选择文件。accept 只是选择提示，服务器仍需校验类型、大小与内容。

非受控减少每次输入都驱动React的需要，但实时跨字段校验、远程重置、撤销和外部同步更难协调。表单库可以结合非受控注册与细粒度订阅，不宜仅凭“用ref”认定快；复杂widget也必须处理注册、清理、初始值与外部更新冲突。

同一字段不要同时把 value 和 defaultValue 当两套权威来源。确需同时支持受控与非受控的公共组件，应规定挂载时模式、受控优先级和onChange语义，禁止中途静默切换。不同字段分别选择模式可以成立，关键是每个字段的所有权明确。

## 四、Context：跨层传递依赖，不自动提供状态治理

### 4.1 Provider 查找与默认值

Context 是跨组件层级传值与订阅的机制。createContext 在组件外创建同一个上下文对象；Provider 给一个子树提供value，useContext读取调用组件上方最近的匹配Provider。它按React组件树工作，不是根据DOM外观距离查找。

没有Provider时返回创建时的默认值；默认值不是动态全局变量。显式Provider value=undefined 也不会回退默认值。组件自己返回的Provider不能反过来影响该组件顶部已经调用的useContext，需要把消费者放在它下面。

~~~jsx
import { createContext, useContext, useState } from 'react';

const LocaleContext = createContext(null);

function useLocale() {
  const value = useContext(LocaleContext);
  if (value === null) throw new Error('缺少 Locale Provider');
  return value;
}

function Greeting() {
  const locale = useLocale();
  return <p>{locale === 'zh' ? '你好' : 'Hello'}</p>;
}

export function LocalePage() {
  const [locale, setLocale] = useState('zh');
  return <LocaleContext.Provider value={locale}>
    <button type="button"
      onClick={() => setLocale(x => x === 'zh' ? 'en' : 'zh')}>
      切换语言
    </button>
    <Greeting />
  </LocaleContext.Provider>;
}
~~~

null 加守卫让缺Provider及时暴露；如果确实有安全合理默认主题，也可用'light'。React19支持 <LocaleContext value={locale}> 简写；本例保留React18可用的.Provider。TypeScript可将权限上下文声明为 createContext<AuthContextValue | null>(null)，其中AuthContextValue必须是项目定义的用户/动作结构，类型声明不会替代运行时Provider与权限检查。

嵌套Provider可局部覆盖语言或主题，适合预览与隔离子区域。模块重复打包导致Provider与Consumer使用不同Context对象时，即使命名相同也不能匹配，应检查导入和依赖去重，而不是不断加Provider。

### 4.2 共享依赖与安全边界

Theme、Locale、当前用户展示信息和Service Instance常适合Context：跨多层需要、职责清楚、作用域可限定。Service Instance是封装API调用等能力的服务对象，Context可为测试替换实例，但对象内部变化不会自动触发React订阅。

Context本身不保存业务state、不提供缓存、撤销、持久化或事务。状态可由Provider中的state/reducer或外部资源持有。把用户信息传下去便于UI显示，不代表客户端隐藏按钮就完成授权；服务器必须校验请求者对对象和租户的权限。

服务端渲染若使用共享模块可变单例保存当前用户，可能跨请求泄露。身份相关值应按请求与Provider作用域隔离。退出登录或切租户时，除了Context展示信息，还需清理查询缓存和草稿，旧请求也需失效，不能仅把头像置空。

## 五、Context 更新范围与性能取舍

### 5.1 value 身份与消费者订阅

Provider新旧value按Object.is比较。value对象每次重新创建，即使字段看起来相同，也可能触发读取它的消费者更新。读取整个Context后只取theme，并不等于React自动订阅theme字段；同一value的其他部分改变也可影响该消费者。

memo 对普通props的跳过不能阻止组件接收它订阅到的新Context。反过来，不能说所有Provider后代都因Context强制更新：直接机制针对读取该Context的消费者；父级正常重新渲染仍有另一条传播路径，需要分别诊断。

虚构例：一个Context同时放theme、鼠标坐标和用户档案。坐标频繁变化会牵动本来只需用户名字的消费者。拆分职责与变化频率，使鼠标位置保持局部或独立订阅，通常比对整个value堆useMemo更有效。

### 5.2 四种改进怎样选择

| 手段 | 能解决什么 | 不能保证什么 |
| --- | --- | --- |
| 按领域/频率拆Context | 减少无关订阅耦合 | 不自动消除父级普通渲染 |
| useMemo稳定value、useCallback稳定动作 | 避免无关父更新造成新引用 | 真实数据变化仍传播，不能删依赖 |
| 查询缓存管理服务器状态 | 去重、失效、共享请求生命周期 | 租户键、重试业务语义仍需设计 |
| Store与Selector | 按所需片段订阅 | 库与selector比较规则必须核对 |

例如 useMemo(() => ({ user, logout }), [user, logout]) 只在两者身份未变时复用对象，不能防user真实更新；logout若每次新建又会破坏效果。稳定身份应是性能优化，不是依赖它保证授权正确。

Selector（选择器）是从Store状态取所需片段的函数，例如s => s.cart.total；配合订阅与相等比较可减少无关通知。若selector每次返回全新对象，也可能失去收益。复杂全局状态才值得引入专门Store的学习、调试和依赖成本；少量低频设置并不必先装库。

## 六、按状态类别设计作用域

| 状态类别 | 典型所有者 | 失败方式与处理 |
| --- | --- | --- |
| 输入草稿、展开项 | 最近页面/局部组件 | 提升过高串页；以实体key与明确重置策略隔离 |
| 页面筛选、页码、排序 | URL或路由层 | 只存本地则刷新/分享丢失；解析URL并校验范围 |
| 后端记录与列表 | 查询缓存/加载层 | 旧请求覆盖或跨租户复用；作用域键与失效 |
| 会话用户、主题、语言 | 有限作用域Provider | 大Context高频传播；拆职责并限制敏感信息 |
| 高频跨区域交互 | 必要时专门Store | 多份副本漂移；明确动作与细粒度订阅 |

URL State（地址状态）适合可分享、可刷新恢复的视图条件；秘密、令牌和敏感表单内容不应放URL。Form State（表单状态）是未提交草稿，Server State是已持久化记录，两者不能在后台刷新时无条件互相覆盖。

编辑资料时，可保存草稿及“基于哪个服务器版本编辑”的基线。后台数据变化，未改动字段可按策略合并，已改动字段提示冲突；提交携带版本，由服务端拒绝过期修改或执行明确合并。React单一事实来源不解决并发写数据库：它只澄清客户端权威范围。

## 七、从需求到实现的决策流程与排障

先列读取者、修改者、生命周期和业务不变量。仅一组件需要则local；兄弟协作提升；跨层依赖先考虑props或组件组合，再Context；高频跨区域且需片段订阅再Store；服务器数据交查询层。Prop Drilling（逐层转发props）两层并非必然坏事，显式依赖反而易追踪；不应只为少写几个props把状态挪到root。

虚构注册表单：简单联系方式只需提交可非受控；密码强度和条件字段需要实时UI，可受控或采用字段订阅库；文件字段由浏览器选择；提交状态由表单所有者统一管理。失败保留草稿并显示可理解错误，成功后按产品规则重置，不能网络开始就先清空所有输入。

排障从现象反推：输入卡住先查value/onChange；重置失效查默认值与模式；切记录串值查key和草稿作用域；Context读不到查Provider层级与对象一致性；渲染多查value身份及订阅范围。使用Profiler观察组件更新原因，不能只看console次数认定生产瓶颈。

验收应覆盖空值与初始异步加载、打字/粘贴/中文组合、复选框、多值FormData、切记录未保存、嵌套Provider、缺Provider、退出登录、迟到响应和键盘标签关联。前端格式验证不代替后端安全校验；缓存优化不代替正确的所有权。

## 八、面试完整回答与追问

### 8.1 什么是单一事实来源，如何提升状态？

“每个作用域的同一事实有一个明确持有者，其他组件读值并提出动作。两个面板互斥展开时，我在共同父级存activeId而非两个独立isOpen，减少非法组合。只提升需要协调的部分，输入草稿仍按实体和生命周期局部保留。”

追问：是不是所有state进root？不是，root延长生命周期与依赖范围；主题可跨应用，某弹窗草稿不该因此常驻。先找共享者和重置边界。

### 8.2 Context为什么会大范围更新？

“消费者订阅的是Context的value，按Object.is判断变化，新对象会触发传播；只读一个字段不自动获得字段订阅，memo也挡不住所订阅Context的新值。先按职责频率拆分，再稳定无关更新产生的引用，确有高频细粒度需求才引入Store。”

追问：value加useMemo就好了？只有依赖未变才复用，真实数据变化仍通知，不能通过漏依赖换取稳定，否则变成陈旧状态。

### 8.3 受控和非受控怎样选？

“受控由React定义当前值，适合实时校验和联动；非受控由DOM保存编辑值，适合简单提交和文件/widget集成。大型表单要测订阅范围与渲染成本，而不是把两者当快慢标签。每个字段应有稳定模式和明确重置协议。”

追问：接口返回后改defaultValue能当重置吗？不能当受控同步协议。可等数据就绪再挂载，或按实体key显式重建，但需先处理未保存草稿丢失。

## 九、扫雷自测与完整答案

1. **Single Source of Truth是什么？** 同一作用域的同一事实有权威持有者，修改经明确动作进入。不是整个应用只有一个state，也不是禁止含义不同的编辑草稿。
2. **两个独立isOpen有何风险？** 两者可同时true而违反互斥要求。公共父级activeId用状态表示直接限制组合，子级只发请求。
3. **Context为何可能大范围Render？** value对象身份变化通知其消费者；消费者只取字段仍订阅该Context。拆领域和频率比隐藏依赖更正确。
4. **memo能阻止Context更新吗？** 不能阻止组件读取到其订阅Context的新值；props优化与Context订阅是两条路径。
5. **哪些表单适合非受控？** 简单提交、原生文件选择和命令式widget等。复杂实时联动可能需要额外订阅，不能只用ref就保证一致性。
6. **defaultValue和value怎样区分？** 前者给非受控初始值，后者持续定义当前值。不要同时做两套权威来源或中途无意切换模式。
7. **受控复选框读什么？** checked布尔值与target.checked，value通常是提交值而非选中事实，读错会使UI与状态脱离。
8. **所有state提升可避免同步吗？** 会增加耦合、范围和意外保留。只提升需要协调的事实，并保持页面/实体的生命周期边界。
9. **Context用户信息能保证授权吗？** 不能，客户端可被绕过；服务器逐请求校验身份与对象权限，缓存还需隔离租户。
10. **服务器刷新应立即覆盖草稿吗？** 通常不应。草稿与已保存记录含义不同，需dirty标记、版本与冲突策略，否则用户未提交输入被静默丢弃。

## 十、速记、版本、来源与验证范围

速记：**先定事实所有者 → local优先 → 共享才提升 → 受控定义值/非受控读取DOM → Context跨层传递 → 高频订阅按片段治理 → 草稿、URL、服务器记录各有边界。**

本章普通示例兼容React18+；React19 Provider简写按当前19.3文档说明，未把Context当事务、安全或持久化机制。以下一手来源用于核验与可选追溯。

- [React：Sharing State Between Components](https://react.dev/learn/sharing-state-between-components)
- [React：input](https://react.dev/reference/react-dom/components/input)
- [React：createContext](https://react.dev/reference/react/createContext)
- [React：useContext](https://react.dev/reference/react/useContext)

代码验证：JSX逐段人工核对状态所有权、事件、输入模式、Provider作用域及失败路径，未在React runtime或浏览器实际运行；未执行DOM、FormData、中文输入法、SSR隔离或性能实验，不能由教学推演宣称真实环境验证通过。
