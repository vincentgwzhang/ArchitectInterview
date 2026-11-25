# 01 TypeScript 类型系统、泛型与严格模式

## 一、类型在什么阶段提供什么保证

TypeScript通过静态类型检查（Static Type Checking）在执行前发现不符合已声明约束的代码，帮助重构、接口设计和IDE反馈。它是JavaScript的类型扩展，不会因为给变量写了User就自动验证服务器JSON。多数类型注解和接口在输出JavaScript时被擦除；类型断言也不生成检查逻辑。

例如把响应断言成余额number，服务器实际返回字符串时，程序仍可能做字符串拼接。正确的边界是：外部值先按unknown接收，运行时验证后转换为内部类型，再由编译器约束内部使用。静态检查与运行时验证承担不同责任，不能互相替代。

TypeScript主要采用结构类型（Structural Typing）：形状兼容即可赋值，不要求显式声明implements。因此两个都只有id:string的对象容易互换。它也保留与JavaScript生态兼容的非完全健全（Sound）边界；any、断言、可变别名和第三方声明错误都可能突破表面保证。面试应解释保证成立的前提，不说“编译通过所以绝不会出错”。

本章基础示例适用于TypeScript 4.9及以上常用语法，satisfies自4.9提供；严格选项需按项目锁定版本验证。React部分依赖匹配的React与类型包；Zod示例按Zod 4基础API核验。核验日期2026-11-25，不要求读者追逐最新版本才能理解原理。

## 二、严格模式：不要把所有安全开关当作一个开关

推荐新项目启用strict，并显式配置另外两个常用检查：

```json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "noEmit": true
  }
}
```

这是类型检查配置片段，不是完整React构建配置；module、target、lib、jsx和模块解析等由框架及运行环境决定。noEmit适用于编译器只检查、由其他工具产出代码的流程。构建器能转译TS不等于已经执行类型检查，应在CI明确运行项目对应的tsc检查。

strict启用一组严格检查，包含strictNullChecks、noImplicitAny等；noUncheckedIndexedAccess和exactOptionalPropertyTypes并不因strict自动开启，必须单独评估。未来编译器可能扩展strict检查，升级会带来新诊断，应锁版本并在升级PR中处理，不随意全局关闭。

### 2.1 空值与隐式any

strictNullChecks把null、undefined与普通类型区分。读取可能不存在的用户前先判断，而不是大量使用非空断言!。可选链只在左侧空值时短路，不证明最终结果一定存在；默认值??仅处理null/undefined，和把空字符串、0也当作缺失的||不同。

noImplicitAny阻止某些无法推断而隐式变成any的情况，但不禁止开发者显式写any，也不保证第三方库不会传播any。catch中的值也应按unknown处理；不能假设throw只会抛Error对象。通过instanceof Error或其他验证后再读取message。

### 2.2 索引访问与属性缺失

noUncheckedIndexedAccess使未保证存在的数组/字典索引读取包含undefined。长度为0的数组没有首元素，字符串键的Record也不表示运行时已拥有每个字符串键。for...of等直接遍历通常更清楚，或对索引结果显式处理缺失；不要把所有读取都加!来“恢复原来体验”。

exactOptionalPropertyTypes区分“属性不存在”与“属性存在且值为undefined”。例如theme?: "dark" | "light"允许省略，但不允许显式写theme: undefined，除非类型明确包含undefined。读取仍可能是undefined，因为属性可能不存在。

这在补丁接口尤其重要：省略表示不修改，null可能表示清空，undefined在JSON序列化时又可能被省略。必须由传输协议定义，不能仅靠TS语法猜语义。Object.assign、in检查和对象展开也会区分存在与否，严格选项有助于提前暴露这些差别。

### 2.3 遗留迁移的边界

迁移先找到外部输入与高风险业务，建立可验证适配层，再按模块开启严格检查和缩小any。必要的临时例外要有Owner、理由及清理计划。@ts-expect-error适用于明确预期诊断的测试或过渡点，原诊断消失会提示清理；不能把它变成长期绕过检查的习惯。

新代码不应因旧模块存在而继续扩散any。skipLibCheck会跳过声明文件的部分检查，不等于修复依赖冲突；升级类型包、编译器与框架版本时仍要验证。迁移成本应与风险和交付约束平衡，但“先全部as any再逐步处理”常把债务隐蔽化。

## 三、基础建模：对象、联合与不可变约定

type可表达联合、交叉、别名等；interface适合对象契约并支持声明合并。二者都不能自动验证对象。团队可按用途形成习惯，不必为“永远选哪一个”争论。公共库使用interface合并能力时要考虑扩展与冲突，应用内部封闭联合通常用type更直接。

联合类型A | B表示值属于某个分支，未收窄前只能安全使用共同能力；交叉类型A & B要求同时满足两侧，不是任选。相冲突字段交叉可能成为never或产生不可满足约束，不能把交叉当成运行时对象合并。

对象字面量会有额外属性检查，但结构赋值并不保证对象只有声明字段。多一个字段的变量可能仍能赋给较窄类型；安全边界需运行时Schema或显式字段投影，不能靠interface当作精确字段白名单。

readonly限制通过当前类型路径的写入，不自动冻结运行时对象，也不深度冻结嵌套结构。另一可变别名仍可能修改同一对象。对并发请求、缓存和状态共享，需采用实际的不可变更新与清晰所有权，不能仅加readonly就宣布没有竞态。

as const保留更窄字面量与只读推断；satisfies检查表达式满足契约，同时通常保留其自身推断信息。两者均不进行运行时验证，不能把未知JSON通过satisfies“认证”。当错误提示过于复杂时，先引入清楚的命名类型或显式公共函数签名，而不是堆更多类型运算。

## 四、联合收窄与状态机

### 4.1 用判别字段排除非法组合

多个Boolean如loading、hasError、hasData可出现“加载失败却同时成功”等矛盾。判别联合（Discriminated Union）让每个状态具有自己的必要字段：

```ts
type LoadState<T> =
  | { status: "idle" }
  | { status: "loading"; requestId: number }
  | { status: "success"; data: T }
  | { status: "error"; error: Error };

function assertNever(value: never): never {
  throw new Error("Unexpected load state");
}

function describe<T>(state: LoadState<T>): string {
  switch (state.status) {
    case "idle": return "尚未请求";
    case "loading": return "正在加载";
    case "success": return "加载成功";
    case "error": return state.error.message;
    default: return assertNever(state);
  }
}
```

switch按status收窄后，error分支可读取error，idle分支不能读取data。所有分支处理完，剩余类型为never；新增cancelled状态而没处理时，传给assertNever会触发诊断。运行时异常是防御兜底，不应输出包含敏感原始对象的消息。

联合约束状态形状，不自动保证合法转换。它不会阻止业务直接从idle跳success，也不会让旧请求结果自动失效。并发场景需用requestId、取消信号和reducer等逻辑核对当前请求；授权和事务状态仍要在服务端处理。

### 4.2 收窄必须是真实证据

typeof、instanceof、in、相等比较和控制流可帮助收窄。typeof null是"object"，数组也是对象，因此验证unknown对象通常需要排除null和Array。真值判断会排除0、空字符串等合法值，数值或可空字段应按真实契约判断。

自定义类型谓词如value is User是开发者对检查的承诺，编译器不会证明函数实现真的覆盖User全部字段。写一个总返回true的谓词只会制造虚假安全。断言函数asserts value is User也一样，需要正确的运行时逻辑。

收窄还受可变引用与闭包影响：判断对象字段后，异步回调执行前对象可能被别处修改。可在合适位置提取稳定值，或者在使用时重新验证；不要只靠断言压掉编译器对生命周期的提醒。

## 五、泛型：表达关系，不是把类型变模糊

泛型（Generics）让函数、接口或类表达多个位置之间的类型关系。T不是any：实现必须对所有允许的T成立，不能随便读取T没有承诺的属性。

```ts
function first<T>(items: readonly T[]): T | undefined {
  return items[0];
}

function getProperty<T, K extends keyof T>(value: T, key: K): T[K] {
  return value[key];
}

const name = getProperty({ name: "Ana", age: 30 }, "name");
// name为string；传不存在的key应报错。
```

first保留元素类型，并明确空数组返回undefined；readonly表示函数不通过参数修改数组，不证明数组一定非空。若业务要求至少一项，可接受readonly [T, ...T[]]元组，但外部数组仍需运行时检查后才能获得这个保证。

keyof T表示T的属性键联合，K extends keyof T限制key必须来自它；T[K]保持返回值与具体key的关系。若把返回值粗暴写成any，调用者就失去关系；若把key写成string，任意字符串又可能不是合法属性。

泛型约束T extends { id: string }表示至少有id能力，不意味着能返回任意新造的{id:"x"}作为T，因为T可能还有其他必要字段。一个只出现在返回位置的fetchJson<T>()若没有Schema或构造器，通常只是让调用方自选一个未经验证的类型，不证明服务器值符合T。

工具类型可减少重复：Pick选择字段，Omit排除字段，Partial把字段可选，Readonly限制写入，Record描述键值映射。它们只做类型变换。Partial<User>不自动等于合法更新命令：ID、角色等字段可能不应允许修改，嵌套也不是默认深度可选。应为写操作定义明确DTO。

条件类型按约束分支推导；映射类型遍历键构造类型；索引访问取属性类型。这些工具适合稳定、重复的关系，不适合为公共API堆叠难诊断的递归“类型体操”。复杂类型增加编译器和IDE负担，接口可读性、错误信息与维护成本也是架构质量。

## 六、unknown、any、断言与运行时验证

unknown能接收未知值，但使用其属性或调用前必须收窄；any允许不受检查的操作并继续传播。受控遗留边界可以短期使用any，但应尽早归一化为unknown或具体已验证模型。as和非空断言!只是改变检查视角，不转换值，不检查范围，也不安装异常处理。

以下Zod 4教学片段以Schema作为运行时与静态结构的同一来源：

```ts
import * as z from "zod";

const UserDtoSchema = z.object({
  id: z.string().min(1),
  display_name: z.string().min(1),
  status: z.enum(["active", "disabled"])
});

type UserDto = z.infer<typeof UserDtoSchema>;

type User = {
  id: string;
  displayName: string;
  active: boolean;
};

function decodeUser(input: unknown): User {
  const result = UserDtoSchema.safeParse(input);
  if (!result.success) throw new Error("Invalid user response");
  const dto: UserDto = result.data;
  return {
    id: dto.id,
    displayName: dto.display_name,
    active: dto.status === "active"
  };
}
```

safeParse返回成功/失败联合；成功后才使用data，失败给出受控错误。parse会在失败时抛验证异常，两种风格按调用边界选择；有异步校验或转换时需相应异步API。示例不记录整个原始输入，避免把个人信息写入错误日志。

Schema验证结构与已声明规则，不证明用户有权限，也不证明数据在下一秒仍有效。字符串非空不证明它是真实用户ID；金额与日期还需单位、范围、精度及业务规则。转换可能改变输出类型，涉及transform时要区分Schema输入与输出类型，不假定两者永远相同。

对未知字段要显式确定拒绝、移除或保留策略，按Schema库版本核验；不要依赖模糊默认值。大输入需限制字节、数组长度和验证工作量，客户端验证不是服务端安全边界。请求失败、JSON解析失败、Schema失败和领域拒绝也应分类，便于恢复与监控。

## 七、React类型：让组件契约能表达意图

Props应区分必需、可选、互斥与回调。不应为方便调用把所有字段设Optional，否则组件内部会遍布默认值和非法状态。children也不默认属于每个组件，确实接受任意可渲染子内容时才声明ReactNode；只接受单个元素等更窄契约则按需要选择类型。

下面示例把导航链接和操作按钮区分；真实项目需匹配React与@types/react：

```tsx
import type { MouseEventHandler } from "react";

type ActionProps =
  | { kind: "link"; href: string; onClick?: never }
  | { kind: "button"; onClick: MouseEventHandler<HTMLButtonElement>; href?: never };

function Action(props: ActionProps) {
  if (props.kind === "link") {
    return <a href={props.href}>打开</a>;
  }
  return <button type="button" onClick={props.onClick}>执行</button>;
}
```

判别字段让每个分支拥有必要参数，never限制互斥字段。exactOptionalPropertyTypes打开时，onClick?: never也不接受显式undefined；类型仍不能阻止JavaScript调用者传坏数据，公共运行时边界需另外验证。

表单事件使用ChangeEvent<HTMLInputElement>等具体类型，或从JSX上下文推断；事件对象不等于字符串值，应取currentTarget.value等实际数据。不要用any把事件与业务回调混在一起。组件内部状态可用LoadState<User>等联合表达，不用无关Boolean拼凑状态。

受控组件值与onChange应一同定义，必要时用联合区分受控与非受控模式。仅把value可选、defaultValue可选、onChange可选会允许含糊组合；默认值与缺失语义需说明。类型约束还应配可访问性与真实交互验证，它不能证明焦点或屏幕阅读器行为正确。

## 八、领域ID与传输边界

结构类型下UserId和OrderId若都只是string，就可能混用。品牌类型（Branded Type）通过一个仅编译期存在的标记区分：

```ts
declare const userIdBrand: unique symbol;
type UserId = string & { readonly [userIdBrand]: true };

function parseUserId(input: unknown): UserId {
  if (typeof input !== "string" || !/^usr_[A-Za-z0-9]+$/.test(input)) {
    throw new Error("Invalid user id");
  }
  return input as UserId;
}
```

断言集中在已验证构造边界，内部函数接受UserId可减少与普通字符串混用。这里usr_格式完全是教学协议，不代表任何真实系统规则。品牌不产生运行时秘密或权限检查，序列化后标记消失，重新读取需验证；有人任意as UserId仍能绕过，因此团队需约束断言出口。

前端不应直接依赖后端数据库Entity。Entity可能带内部字段、懒加载关系或敏感信息，数据库变更也不应强迫UI同时改。可从OpenAPI/GraphQL生成Transport Type作为线格式契约，再运行时验证并映射为前端领域模型。

生成类型描述Schema，不保证实际部署响应符合Schema；服务端旧版本、网关错误页和未知枚举都可能出现。需要明确新字段、缺失、null与未知枚举兼容策略。领域映射层有少量维护成本，但能隔离传输命名、单位和演进，避免UI到处依赖display_name等线格式细节。

## 九、工程落地与错误定位

CI应明确类型检查、构建和运行测试三个阶段：tsc检查约束，构建器验证打包和模块解析，测试验证运行行为。只看到页面能打开，不说明所有类型通过；只tsc通过，也不能证明接口数据或业务逻辑正确。

对公共类型API保留正例与负例：正确调用应通过，互斥Props、错误ID和漏状态应产生预期诊断。可用@ts-expect-error测试拒绝能力，但应让测试文件真实进入类型检查。运行时Schema还需输入null、数组、缺字段、错误枚举、边界长度和超大输入等用例，不能只测一份成功响应。

虚构排障：开启exactOptionalPropertyTypes后，补丁对象name: undefined报错。先确认接口中“省略不修改”“null清空”的语义，按条件构造字段，而不是统一扩大成string | undefined。这个诊断暴露了存在性契约，不只是编译器变严格。

另一个虚构案例：静态类型正确却出现toFixed不是函数。追踪到泛型请求函数直接把JSON断言为T，应把返回边界改成unknown并接Schema，分类处理不兼容响应。修复重点在建立真实证据，不是在每个使用点再加断言。

类型复杂导致构建慢时，分析公共泛型与大联合的使用，优先拆命名类型、减少不必要分布与递归、明确模块边界。不要为了省几行重复，让错误信息长到无人能维护。类型系统的性能与学习成本也是工程取舍。

## 十、面试完整回答与深入追问

### 1. TypeScript能保证什么，不能保证什么？

答：它根据声明和推断在编译期检查约束，帮助重构和API使用；类型大多被擦除，不能自动验证外部JSON，也不能证明权限、事务或运行时状态。我的做法是外部unknown、Schema验证、领域映射，再在内部使用严格类型，让非法状态难以表示。

追问：as能把字符串变成number吗？答：不能，断言不做转换。需要显式解析、范围与业务验证，再取得真实数字；双重断言只绕过检查，不创造事实。

### 2. strict打开后是否安全检查全开？

答：不是。strict是一组检查，strictNullChecks和noImplicitAny是重要部分，noUncheckedIndexedAccess和exactOptionalPropertyTypes还需显式配置。升级可能新增诊断，CI要锁版本并验证。any、断言和错误声明仍能突破约束。

追问：为什么可选字段不能直接写undefined？答：精确可选模式区分缺失与存在的undefined，对in、展开和补丁接口有意义；若业务真的允许显式undefined，应在类型中表达，但JSON传输语义仍需单独定义。

### 3. 泛型如何设计才有价值？

答：它应保持输入、输出或多个参数间的类型关系，例如first保留元素类型且返回undefined，getProperty保持键与对应值的关系。约束只赋予能力，不允许随意构造任意T。若类型参数只让调用者宣称响应是什么却不验证，就可能制造虚假安全。

追问：为何不能在T extends {id:string}里直接返回{id:"x"}？答：T可能还要求name等字段，满足约束只代表它属于允许集合，不代表一个基础对象满足所有具体T。

### 4. 为什么用判别联合代替多个Boolean？

答：联合让每个状态的必要字段跟随status，避免loading和success等矛盾组合；switch收窄后安全访问，never穷尽检查让新增状态暴露漏分支。但状态转换与并发响应仍需业务逻辑和请求版本校验。

追问：never检查能处理服务器新增未知状态吗？答：不能自动处理。外部值先验证并制定未知值策略，穷尽检查只针对当前静态类型声明的集合，错误断言会破坏前提。

## 十一、扫雷自测与完整答案

1. **interface会在运行时验证JSON吗？** 不会。它是静态契约，需运行时检查后才可信。
2. **strict是否自动开启两个额外选项？** noUncheckedIndexedAccess和exactOptionalPropertyTypes需分别配置，不能混为一组。
3. **noImplicitAny能禁止全部any吗？** 不能。显式any和依赖传播仍可能存在，需边界治理与代码审查。
4. **readonly是否深度冻结？** 不是。它限制类型路径上的写入，运行时对象与其他别名仍可能可变。
5. **unknown等于any吗？** 不等于。unknown使用前要求收窄，any会绕开检查并传播风险。
6. **类型谓词总返回true会被编译器识破吗？** 不能依赖它识破。谓词是作者承诺，逻辑错误会制造假安全，必须验证实现。
7. **first返回T是否足够？** 普通数组可能为空，应返回T | undefined或要求并验证非空结构。
8. **Partial<User>是否就是合法更新命令？** 不一定。它会允许本不该更新的字段且只做类型变换，应定义具体命令。
9. **Branded ID能当授权证明吗？** 不可以。品牌多为编译期区分，序列化后消失，权限必须在服务端验证。
10. **生成Transport Type后是否可以省Schema和映射？** 不能自动省。实际响应可能偏离声明，映射也承担命名、单位和演进隔离，应按风险决定验证深度。

## 十二、速记、来源与验证范围

速记：类型管内部约束，Schema管边界证据；strict不是所有开关；可选是存在性，索引须考虑缺失；Union表达状态，never检查遗漏；泛型表达关系，断言不创造事实；Props表达意图，品牌区分ID，Transport与领域模型分离。

核验日期2026-11-25。示例使用TS 4.9及以上常用语法与严格选项；Zod示例按4基础API，React类型需匹配项目版本。编译器strict检查集合可能演进，升级需重新检查。

- [TypeScript：基础类型与断言](https://www.typescriptlang.org/docs/handbook/2/everyday-types.html)
- [TypeScript：收窄与never](https://www.typescriptlang.org/docs/handbook/2/narrowing.html)
- [TypeScript：泛型](https://www.typescriptlang.org/docs/handbook/2/generics.html)
- [TSConfig：strict](https://www.typescriptlang.org/tsconfig/strict.html)
- [TSConfig：noUncheckedIndexedAccess](https://www.typescriptlang.org/tsconfig/noUncheckedIndexedAccess.html)
- [TSConfig：exactOptionalPropertyTypes](https://www.typescriptlang.org/tsconfig/exactOptionalPropertyTypes.html)
- [Zod：safeParse与类型推导](https://zod.dev/basics)
- [React：TypeScript与Props](https://react.dev/learn/typescript)

验证范围：原知识点覆盖、全节自足性、示例类型关系与失败语义、Markdown结构已人工检查。示例为独立教学片段，未实际运行tsc、React或Zod测试；案例为虚构推演，不是用户项目结论。
