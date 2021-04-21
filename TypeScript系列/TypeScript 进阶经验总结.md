# TypeScript 进阶经验总结

## 前言

使用 TypeScript 也快一年了，本文主要分享一些工作常用的知识点技巧和注意点。

本文适合了解 TypeScript 或有实际使用过一段时间的小伙伴。

如果对 TypeScript 基础知识不熟悉的可以看下我这篇文章：[TypeScript 入门知识点总结](https://juejin.cn/post/6886059772451012616)

## 操作符

### 类型别名

用来给一个类型起个新名字

```ts
type Props = TextProps
```

### keyof

用于获取某种类型的所有键，其返回类型是联合类型。

```ts
interface Person {
  name: string
  age: number
}

type PersonKey = keyof Person // "name" | "age"
```

### 联合类型

和逻辑 "||" 一样都是表示其类型为多个类型中的任意一个

比如当你在写[Button 组件](https://github.com/Jacky-Summer/monki-ui/blob/679c4033269714e18cfe095faf4a2b135f4d5fe2/components/button/button.tsx#L21)的时候：

```ts
export type ButtonSize = 'lg' | 'md' | 'sm'
```

### 交叉类型

将多个类型合并成一个类型

比如当你给一个新的 React 组件定义 Props 类型，需要用到其它若干属性集合

```ts
type Props = TypographyProps & ColorProps & SpaceProps
```

### extends

主要作用是添加泛型约束

```ts
interface WithLength {
  length: number
}
// extends 来继承
function logger<T extends WithLength>(val: T) {
  console.log(val.length)
}
logger('hello')
logger([1, 2, 3])
// logger(true) // error 没有length属性
```

### typeof

typeof 是获取一个对象/实例的类型

```ts
interface Person {
  name: string
  age: number
}
const person1: Person = { name: 'monkey', age: 18 }
const person2: typeof person1 = { name: 'jacky', age: 24 } // 通过编译
```

## 泛型工具类型

### Partial

将某个类型里的属性全部变为可选项。

```ts
// 实现：全部变可选
type Partial<T> = {
  [P in keyof T]?: T[P]
}
```

例子

```ts
interface Animal {
  canFly: boolean
  canSwim: boolean
}

// 变可选，可以只赋值部分属性
let animal: Partial<Animal> = {
  canFly: false,
}
```

### ReadOnly

它接收一个泛型 T，用来把它的所有属性标记为只读类型

```ts
// 实现：全部变只读
type Readonly<T> = {
  readonly [P in keyof T]: T[P]
}
```

```ts
interface Person {
  name: string
  age: number
}

let person: Readonly<Person> = {
  name: 'jacky',
  age: 24,
}
person.name = 'jack' // Cannot assign to 'name' because it is a read-only property.
```

### Required

将某个类型里的属性全部变为必选项。

```ts
// 实现：全部变必选
type Required<T> = {
  [P in keyof T]-?: T[P]
}
```

```ts
interface Person {
  name?: string
  age?: number
}

// Property 'age' is missing in type '{ name: string; }' but required in type 'Required<Person>'.
let person: Required<Person> = {
  name: 'jacky',
  // 没写 age 属性会提示错误
}
```

### Record

```ts
// 实现：K 中所有属性值转化为 T 类型
type Record<K extends keyof any, T> = {
  [P in K]: T
}
```

Record 生成的类型具有类型 K 中存在的属性，值为类型 T

```ts
interface DatabaseInfo {
  id: string
}

type DataSource = 'user' | 'detail' | 'list'

const x: Record<DataSource, DatabaseInfo> = {
  user: { id: '1' },
  detail: { id: '2' },
  list: { id: '3' },
}
```

### Pick

```ts
// 实现：通过从Type中选择属性Keys的集合来构造类型
type Pick<T, K extends keyof T> = {
  [P in K]: T[P]
}
```

用于提取接口的某几个属性

```ts
interface Animal {
  canFly: boolean
  canSwim: boolean
}

let person: Pick<Animal, 'canSwim'> = {
  canSwim: true,
}
```

### Exclude

```ts
// 实现：如果 T 中的类型在 U 不存在，则返回，否则不返回
type Exclude<T, U> = T extends U ? never : T
```

将某个类型中属于另一个的类型移除掉

```ts
interface Programmer {
  name: string
  age: number
  isWork: boolean
  isStudy: boolean
}

interface Student {
  name: string
  age: number
  isStudy: boolean
}

type ExcludeKeys = Exclude<keyof Programmer, keyof Student>
// type ExcludeKeys = "isWork"
```

### Omit

```ts
// 实现：去除类型 T 中包含 K 的键值对。
type Omit<T, K extends keyof any> = Pick<T, Exclude<keyof T, K>>
```

作用于 Pick 相反，还是直接看代码容易理解

```ts
interface Animal {
  canFly: boolean
  canSwim: boolean
}

let person1: Pick<Animal, 'canSwim'> = {
  canSwim: true,
}

let person2: Omit<Animal, 'canFly'> = {
  canSwim: true,
}
```

### ReturnType

```ts
// 实现：获取 T 类型(函数)对应的返回值类型
type ReturnType<T extends (...args: any) => any> = T extends (...args: any) => infer R ? R : any
```

获取函数返回值类型

```ts
function bar(x: string | number): string | number {
  return 'hello'
}
type FooType = ReturnType<typeof bar> // string | number
```

## 运算符

### 可选链运算符

可选链操作符(`?.`)允许读取位于连接对象链深处的属性的值，而不必明确验证链中的每个引用是否有效。如果遇到 null 或 undefined 就可以立即停止某些表达式的运行

这个在项目中比较常用，也很好用。

比如我们数据都是从 api 返回的，假设有这么一组数据

```js
// 请求 api 后才返回的数据
{
  data: {
    name: 'jacky',
    age: 24,
    address: {
      city: 'shenzhen',
    },
  },
}
```

前端接收后拿来取值，`apiRequestResult.data.address.city`，正常情况取得到值，如果后台 GG 没有返回 address 这个对象，那么就会报错：

```
TypeError: Cannot read property 'city' of undefined
```

这个时候需要前端处理了，比如最直接的就是

```js
apiRequestResult && apiRequestResult.data && data.address && data.address.city
```

或者 lodash 库方法也能解决

```js
import * as _ from 'lodash'
const city = _.get(apiRequestResult, 'apiRequestResult.data.address.city', undefined)
```

项目中如果都是这么处理，未免太过于繁琐，特别对象很多的情况下，这时就可以使用 TS 的可选链操作符，

```ts
apiRequestResult?.data?.address?.city
```

上述的代码会自动检查对象是否为 null 或 undefined，如果是的话就立即返回 undefined，这样就可以立即停止某些表达式的运行，从而不会报错。

当你在 React 中使用 ref 封装判断元素是否包含 XX 元素时，也会用到这个操作符

```ts
const isClickAway = !childRef.current?.contains?.(e.target)
```

### 空值合并运算符

空值合并操作符（`??`）是一个逻辑操作符，当左侧的操作数为 null 或者 undefined 时，返回其右侧操作数，否则返回左侧操作数。

与 `||`操作符类似，但是 `||` 是一个布尔逻辑运算符，左侧的操作数会被强制转换成布尔值用于求值。如`0, '', NaN, null, undefined`都不会被返回。

```ts
const foo1 = null
const foo2 = 12
const foo3 = 'hello'

const res1 = foo1 ?? 'value' // 'value'
const res2 = foo2 ?? 'value' // 12
const res3 = foo3 ?? 'value' // 'hello'
```

实际开发也可以混用`?.`和`??`

```
const title = document.getElementById("title")?.textContent ?? "";
```

### 非空断言运算符

在上下文中当类型检查器无法断定类型时，这个运算符(`!`)可以用在变量名或者函数名之后，用于断言操作对象是非 null 和非 undefined 类型。不过一般是不推荐使用的

这个适用于我们已经很确定知道不会返回空值

```ts
function print(name: string | undefined) {
  let myName: string = 'jacky'
  // Type 'string | undefined' is not assignable to type 'string'.
  // Type 'undefined' is not assignable to type 'string'
  myName = name
}
```

要解决这个报错，可以加入判断

```ts
function print(name: string | undefined) {
  let myName: string = 'jacky'
  if (name) {
    myName = name
  }
}
```

但这样写代码就显得冗长一点，这时你非常确定不空，那就用非空断言运算符

```ts
function print(name: string | undefined) {
  let myName: string = 'jacky'
  myName = name!
}
```

从而可以减少冗余的代码判断

## 项目中的细节应用

### 尽量避免 any

避免 any，但检查过不了怎么搞？可以用 unknown（不可预先定义的类型）代替

给个不是很好的例子，因为下面例子是本身不对还强行修正错误。大概情况是有些未知数据的情况，需要强定义类型才能过，可以用`(xx as unknown) as SomeType`的形式，当然这里只是例子，项目中类型检查逼不得已的情况下可能会使用

```ts
const metaData = {
  description: 'xxxx',
}
interface MetaType {
  desc: string
}

function handleMeta(data: MetaType): void {}

// 这两种写法都会报错
// handleMeta(metaData)
// handleMeta(metaData as MetaType)

// 这里可以通过类型检查不报错，替代 any 的功能同时保留静态检查的能力，
handleMeta((metaData as unknown) as MetaType)
```

### 可索引的类型

一般用来约束对象和数组

```ts
interface StringObject {
  // key 的类型为 string, 代表对象
  // 限制 value 的类型为 string
  [key: string]: string
}
let obj: StringObject = {
  name: 'jacky',
  age: 24, // 此行报错：Type 'number' is not assignable to type 'string'.
}
```

```ts
interface StringArr {
  // key 的类型为 number, 代表数组
  // 限制 value 的类型为 string
  [key: number]: string
}
let arr: StringArr = ['name', 'jacky']
```

### 注释

通过 `/** */` 形式的注释让 TS 类型做标记提示，编辑器会有更好的提示

```ts
/** Animal common props */
interface Animal {
  /** like animal special skill */
  feature: string
}

const cat: Animal = {
  feature: 'running',
}
```

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/53755f93b68d4cc8926499cecdb1410f~tplv-k3u1fbpfcp-watermark.image)

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fd63e1517a614326a6e55efacf304270~tplv-k3u1fbpfcp-watermark.image)

### ts-ignore

还是那样，尽可能用 unknown 来代替使用 any

如果你们项目禁用 any，那么 ts-ignore 是一个比 any 更有潜在影响代码质量的因素了。

禁用了但有些地方确实需要使用忽略的话，最好注释补多句理由。

比如有些函数单元测试文件，测试用例故意传入不正确的数据类型等，有时是逼不得已的报错。

```js
// @ts-ignore intentionally for test
```

<br>

- [个人技术博文 Github 仓库](https://github.com/Jacky-Summer/personal-blog)
  觉得不错的话欢迎 star，给我一点鼓励继续写作吧~
