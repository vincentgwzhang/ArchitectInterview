# 02. 组件分层、Feature-based 目录与领域边界

> 目标是让一个业务能力容易定位、理解、修改、测试和删除。目录只是边界的可见形式，真正边界由依赖、数据所有权、公共契约和团队责任共同定义。本文采用React与TypeScript项目举例；工具行为于2026-11-23核验官方资料，不把一套目录命名当框架强制标准。

## 一、从变化与删除验证架构

如果修改结算必须同时理解全站components、services、utils和一个全局Store，文件虽然按类型整齐，业务变化仍分散。Feature-based（按业务功能组织）把共同变化的UI、模型、请求适配与测试放近，使开发者从“我要改结算”找到入口，而非先猜文件类型。

不要把“每个业务名一个目录”误当低耦合。checkout目录若导入catalog内部缓存、payment内部状态和app路由单例，移动目录并未切断依赖。判断边界可问：删除此功能需要改哪些外部调用？实现重写时调用者是否只需遵守原契约？测试能否只装配必要依赖？

小应用先简单组织，避免为空想的未来写复杂层级。新增抽象应解决实际变化模式，例如同一结算规则被多个UI重复实现、生成DTO变化波及大量组件或团队同时修改共享目录；没有这些问题时，局部模块与清楚props可能已经足够。

## 二、组件分层：按责任而不是按尺寸

| 层次 | 责任与示例 | 不应承担 |
| --- | --- | --- |
| UI Primitive | Button、Input、Modal的通用交互与可访问性 | 订单查询、特定支付权限 |
| Composite Component | AddressForm、ProductCard组合字段与布局 | 默认拥有所有页面数据 |
| Feature Component | Checkout、SearchFilters的业务交互与流程 | 随意读另一Feature内部Store |
| Route/Page | 页面参数、能力组合、页面级数据与错误布局 | 复制各领域的核心规则 |
| Application Shell | Router、Provider装配、导航壳与最后错误防线 | 变成承载全部业务逻辑的God Component |

Composite不必全进shared：ProductCard若表达商品促销规则，属于catalog内部；AddressForm若跨域语义相同且契约稳定才值得共享。Primitive可包含自身焦点和展开状态，但不应知道“付费用户才能提交订单”。

层次不要求每次调用都经过五层。一页只有一个按钮和小量状态，无需制造空转容器。重点是上层组织业务、下层稳定能力不反向认识具体业务；弹窗通过onConfirm等契约表达意图，调用方决定是否发交易。

App中的全局Error Boundary是最后兜底，不排斥Feature/Route局部边界。功能失败应在适当故障域降级，别让可选推荐错误抹掉结算草稿。Provider也应按作用域放置：结算草稿Provider放结算区域，主题Provider才适合较高位置。

## 三、Feature-based目录与公共入口

推荐一种可调整布局，以下表格是文件位置映射，不是必须创建全部目录：

| 位置 | 内容 |
| --- | --- |
| src/app | 启动、路由和依赖装配 |
| src/pages | 路由入口及跨Feature页面组合 |
| src/features/checkout/api | 请求、DTO校验与适配 |
| src/features/checkout/model | 结算状态、规则、用例与公开类型 |
| src/features/checkout/components | 结算内部UI |
| src/features/checkout/index.ts | 少量公共导出 |
| src/features/catalog | 商品领域对应内部结构 |
| src/shared/ui | 经过治理的通用UI |
| src/shared/infrastructure | 无业务归属的HTTP/日志等基础能力 |

测试可与对应实现并置，便于改动时一起发现；文档、fixture与样式也应有明确归属。不要为了表格完整给每个Feature建空api/model目录，纯展示功能可以更扁平。

公共index.ts可只导出CheckoutPanel、CheckoutInput类型和必要用例，不export *所有内部。Public API（公共契约）应包含输入含义、输出、失败和生命周期，而不是仅列类型。外部从@/features/checkout入口导入；内部文件之间用局部路径，避免内部再绕自身index造成循环。

Barrel（聚合导出）不会自动建立封装或自动保证tree shaking。运行时有副作用的模块、复杂重导出和打包器配置会影响结果；巨型入口还可能把服务器专属代码和客户端组件混进同一依赖图。必要时设少量明确子入口，例如ui与server，保持其使用环境清楚。

## 四、Domain Boundary：业务语言与状态所有权

Ubiquitous Language（统一语言）是同一业务语境中对概念的共同定义。例如Cart是用户选购意图，Order是已提交交易记录，Payment是付款过程；三者可引用ID但不应混成一个全能Global Model。购物车改数量不能等价修改已确认订单，支付失败也不必删掉用户选购意图。

Ownership（所有权）明确谁能修改规则与事实、谁审查公共契约、谁处理运行故障。前端不必照搬后端微服务目录，一张页面可以组合多个后端能力，一个用户任务也可能跨服务；按用户任务与变化关系决定前端边界，再用适配隔离后端差异。

全局Store是存储位置，不会自动消除领域边界。若checkout直接修改payment内部状态字段，两个目录仍紧耦合。通过公开动作和结果协调，如付款完成后让上层触发订单刷新；共享事实由明确权威持有，其他领域通过ID/摘要或资源订阅读取，不复制一份再用Effect追平。

权限显示可以在Feature判断以改善体验，但服务器必须逐请求验证身份、租户和对象权限。前端分层、package或隐藏导出都不是防攻击沙箱，不能保护已发往浏览器的秘密。

## 五、API Adapter：阻止DTO把UI拖进后端实现

DTO（Data Transfer Object）是接口传输形状，Frontend Model是UI真正需要的语义。Adapter（适配器）在入口完成校验和转换，避免每个组件都知道status_code、可空字段和金额单位。并非每个DTO都必须机械复制一套：结构与语义完全一致且稳定时，可在边界验证后复用，避免无意义映射成本。

~~~typescript
type OrderSummary = {
  id: string;
  totalMinor: number;
  currency: string;
  status: 'pending' | 'paid' | 'unknown';
};

export function toOrderSummary(input: unknown): OrderSummary {
  if (typeof input !== 'object' || input === null) {
    throw new Error('Invalid order');
  }
  const x = input as Record<string, unknown>;
  if (typeof x.order_id !== 'string' || x.order_id.length === 0 ||
      typeof x.total_minor !== 'number' ||
      !Number.isSafeInteger(x.total_minor) || x.total_minor < 0 ||
      typeof x.currency !== 'string' || !/^[A-Z]{3}$/.test(x.currency) ||
      typeof x.status_code !== 'string') {
    throw new Error('Invalid order fields');
  }
  return {
    id: x.order_id,
    totalMinor: x.total_minor,
    currency: x.currency,
    status: x.status_code === 'PENDING' ? 'pending'
      : x.status_code === 'PAID' ? 'paid' : 'unknown'
  };
}
~~~

示例约定金额为非负最小货币单位整数；三位字母仅校验格式，不保证币种受业务支持。退款/负金额不是此摘要契约，应另建模型。unknown保留新增状态的显式未知语义，不能默默映射paid；关键交易遇未知需禁用相关动作、显示待确认并上报兼容问题。

TypeScript类型在运行时不会替你验证网络JSON，故入口从unknown开始。请求失败还需分为超时、鉴权、业务拒绝与结构错误，不把所有失败转换为空订单。UI显示安全消息，遥测关联版本和请求标识，不输出整个敏感DTO。Adapter不会让过期报价自动有效，提交仍需服务端版本与并发校验。

## 六、单向依赖与跨Feature协作

基础规则为app/pages组合features，features使用shared，shared不导入具体Feature。架构依赖图应无不必要循环；两个Feature互调常提示职责混合或共同编排应提升，而不只是需要换成别名路径。

例如页面要展示商品并创建结算，Page从catalog公共入口获得选择结果，将明确的商品ID与数量交checkout。checkout不读取catalog私有全局状态，价格由结算服务重新报价。若必须保持同一事务，浏览器两个动作不构成分布式事务，应交服务端用例或有明确失败补偿的流程。

可选协作方式有三种：上层编排适合页面任务；单向公开调用适合稳定能力但要检查依赖方向；事件适合松耦合通知但必须定义类型、订阅清理、顺序和失败。一个全局Event Bus并非免费解耦，它会把可见import依赖变成隐式运行依赖，故障追踪更难。

不要为消除循环把所有业务类型搬shared。先判断类型是否真同义、规则谁拥有；两个“User”可能分别是账户资料与配送收件人，应允许独立模型。仅共享稳定的技术原语或经过协商的契约。

## 七、让规则可执行：别名、Lint与Package

### 7.1 路径别名不是封装

TypeScript paths帮助编译器解析模块，不会自动重写输出import或让运行时理解别名。打包器、测试runner与Node环境需一致配置；编辑器不报错不证明生产能加载。相对路径变短也不会阻止深导入。

下例为ESLint flat config里的规则片段，用于禁止checkout内部实现被入口外的文件导入。项目需安装适当TS parser并组合其原配置，不是完整可直接运行的ESLint配置：

~~~javascript
export default [
  {
    files: ['src/**/*.{ts,tsx}'],
    ignores: ['src/features/checkout/**'],
    rules: {
      'no-restricted-imports': ['error', {
        patterns: [{
          group: ['@/features/checkout/*'],
          message: '请从 checkout 公共入口导入'
        }]
      }]
    }
  }
];
~~~

这只限制匹配的静态import字符串，不会解析所有相对路径、别名变体、动态加载或循环。生产边界检查应结合解析后的依赖图/边界插件，明确测试与生产例外，并在CI运行。先为正常公共导入、深导入、相对绕过和循环建立正负样例，防止“配置存在”但实际不拦截。

### 7.2 什么时候拆Package

Monorepo（单仓库多项目）适合多个应用共享设计系统、明确Ownership和稳定Public Contract的独立包。package可提供独立构建、测试、发布与依赖声明，但也增加版本、兼容矩阵、构建图和调试成本；每个Feature一个包往往过早。

package.json exports可声明包名允许的入口，例如只公开根入口和tokens；内部文件不暴露为普通包子路径。这约束支持该解析机制的消费者，不是绝对安全封装，直接文件路径等仍需工程规则。新增exports可能破坏以前的深导入，是兼容性变更，应先盘点调用者再迁移。

共享React组件包还需治理React等peer依赖与构建产物，避免重复运行时导致Hooks/Context问题；SSR、RSC与浏览器环境入口分离需按实际框架验证。Package拆分不等于运行时独立部署，Micro-frontend还增加跨应用路由、版本、认证、样式与遥测协调，不应仅为目录整洁引入。

## 八、Smart与Presentational：分离有价值的变化

Smart组件或Hook负责数据获取、权限展示、状态转换和业务编排；Presentational组件接收明确props、渲染UI与发回语义事件。比如OrderSummaryView接收summary、loading、error、onRetry，不自己导入账户Token或读取router单例，便于Storybook覆盖不同视觉状态。

不必给每个小组件都制造Container/View双文件。业务耦合小、不会复用的组件可保持内聚；出现独立测试、多个数据源或复杂状态时再分离。一个名为View却内部fetch与写全局Store的组件仍不是纯展示，命名不能代替职责。

Hook也有所有权：useCheckout可封装结算流程，却不意味着多次调用自动共享同一state或请求。若流程跨多个组件需要共享，就由区域Provider或资源层明确持有，Hook只作接入。页面卸载、用户换租户、旧请求迟到和重复提交都需在用例层处理，不能只把代码挪到model目录就认为失败语义解决。

## 九、渐进迁移、测试与治理

虚构遗留系统迁移可先选checkout一条完整用户路径。先画出实际import与状态读写关系，定义当前公共入口；再把文件移近并保留临时兼容转发；抽出DTO adapter与状态Owner；迁移调用者并启用新增违规禁止；确认旧入口无人使用后删除。不要一次改路径、接口、状态库与渲染框架，否则回归难归因。

测试围绕边界价值安排：纯规则/Adapter测输入和未知状态；展示组件测加载空错与交互语义；Feature集成测试测真实编排和请求失败；少量E2E覆盖结算等关键路径。Storybook验证可见状态，不能证明服务端授权、接口兼容和交易幂等。

架构治理可记录每个Feature的Owner、公共入口、允许依赖、外部请求和敏感数据。CI检查新增循环、非法深导入与构建结果，运行时监测错误与性能回归。衡量一次业务改动跨多少领域、删除功能需改多少外部文件、测试反馈时间，比追求文件夹数量有用；不编造固定阈值。

shared必须有责任人和准入条件。某函数只被两个地方使用不代表业务语义相同，先允许小量重复，确认共同变化规律再提取；过早公共化会要求多团队协商每次变更。废弃shared能力要有调用者清单与迁移期，不能无限保留“兼容旧接口”。

## 十、常见反模式与具体修复

| 反模式 | 为什么有害 | 修复与边界 |
| --- | --- | --- |
| 巨型components/utils/services | 按文件类型隐藏共同变化关系 | 先按业务路径归位，真正通用技术能力才共享 |
| shared无Owner | 任何需求都加条件，没人敢删除 | 明确准入/责任/弃用，不靠新命名解决 |
| 深层相对路径跨Feature | 依赖私有结构，重构牵连调用者 | 通过小公共API并加解析规则 |
| UI遍用生成DTO所有字段 | 后端小改动扩散，未知值处理不一致 | 边界验证与有意义适配 |
| 全局state重新耦合领域 | 任意位置修改他域事实 | 按Owner公开动作，跨域由上层编排 |
| 每Feature一个包/微前端 | 构建与协调成本压过自治收益 | 先模块化，证实共享/发布需求再拆 |

修复时保留正确业务行为，先列回归场景。模块化不承诺性能自动提升：入口导入、CSS副作用、代码拆分和状态订阅仍需独立测量；边界主要优化变更和认知成本，不是用目录结构替代性能治理。

## 十一、面试完整回答与深入追问

### 11.1 怎样设计大型React项目目录？

“先识别用户任务和业务变化边界，把结算、目录等能力内部的组件、模型、请求适配与测试放近。app/pages负责装配，Feature通过少量公共入口提供能力，shared只放稳定通用能力。用依赖规则和Owner保证边界，按实际共享和发布需要再拆package。”

追问：为什么不按components/hooks/services全局分类？规模小时可行，但业务变化会跨多个大目录；按Feature可降低定位和删除成本。不是目录越深越好，应以一次真实业务变更检验。

### 11.2 两个Feature互相依赖怎么处理？

“先看是不是共同用例应由页面或应用层编排，还是稳定单向能力调用。不能为去循环把所有模型丢shared，也不能全换事件掩盖运行依赖。用明确输入、结果与失败契约协调，事务一致性交服务端负责。”

追问：商品与订单都用Product类型要共享吗？若订单保存购买时快照、商品表示当前目录，两者含义不同，应按语境分别建模，可共享稳定ID或明确摘要契约。

### 11.3 何时拆Package或微前端？

“多应用共享、独立Owner、稳定接口与明确发布收益支持package；先计算构建、版本和兼容成本。微前端还要求运行时自治与独立部署收益，目录分层本身不需要它。”

追问：index.ts就能封装吗？不能，它只是约定入口，仍能被深导入；结合Lint依赖图与package exports，并测试绕过路径，且这些都不是安全沙箱。

## 十二、扫雷自测与完整答案

1. **Feature目录本身能隔离吗？** 不能，import、全局state、事件和数据访问可绕开目录，需公共契约、Owner与执行规则。
2. **Primitive为何不查订单权限？** 权限属于业务语义，会让所有通用按钮耦合具体域；调用方决定行为，服务器最终授权。
3. **Composite是否都放shared？** 不，ProductCard可能承载catalog规则，应先按业务归属，语义稳定通用才共享。
4. **DTO为何不能只信TS类型？** JSON运行时可违反声明，类型擦除不验证外部输入；入口应校验并处理未知状态。
5. **index.ts是否应导出所有内部？** 不，应只公开调用者必要能力，否则内部结构成为兼容负担，还可能引入循环与环境混用。
6. **路径别名是否自动运行可用？** 不，TS paths不自动重写输出，打包器/测试/运行时需要匹配解析。
7. **全局Store是否破坏边界？** 不必然，但任意模块直接改他域字段会破坏；仍要Owner与公开动作。
8. **事件总线是否总比import解耦？** 不，事件形成隐式时序、订阅与失败依赖；没有明确契约时排障更难。
9. **所有Feature拆包好不好？** 过早拆包增加构建版本成本，先证明共享、发布与治理收益，不按文件数机械判断。
10. **怎样证明重构有价值？** 用真实变更/删除路径、非法依赖拦截、关键行为回归与反馈时间观察，不以目录漂亮或层数增多作证据。

## 十三、速记、来源与验证范围

速记：**业务变化聚在Feature，公共入口控制暴露，单向依赖限制耦合，Adapter隔离传输，状态各有Owner，shared有准入，拆包有收益。**

资料于2026-11-23核验：React当前19.3、TypeScript模块参考（5.x/6.x相关paths语义）、ESLint当前no-restricted-imports文档、Node当前26.13.1包入口文档。本文TS示例使用普通现代语法；ESLint片段为flat config规则示意，应结合项目锁定版本与解析器实测，不声称覆盖所有导入方式。

- [React Thinking in React](https://react.dev/learn/thinking-in-react)
- [TypeScript Modules Reference：paths](https://www.typescriptlang.org/docs/handbook/modules/reference.html#paths)
- [ESLint no-restricted-imports](https://eslint.org/docs/latest/rules/no-restricted-imports)
- [Node.js Package entry points](https://nodejs.org/api/packages.html#package-entry-points)

验证范围：Adapter、ESLint片段和架构依赖路径经人工核对，未实际编译TypeScript、运行ESLint、React/Storybook/E2E或构建包。本文未提供真实项目性能benchmark，不将结构推演宣称为工具执行结果。
