# TypeScript 入门知识点总结

## TypeScript 介绍

### 什么是 TypeScript

是 JavaScript 的一个超集，它可以编译成纯 JavaScript。编译出来的 JavaScript 可以运行在任何浏览器上，主要提供了**类型系统**和**对 ES6 的支持**。

### 为什么选择 TypeScript

- 增加了代码的可维护性
- 包容性强，支持 ES6 语法，`.js`文件直接重命名为`.ts`即可
- 兼容第三方库，即使第三方库不是 TypeScript 写的，也可以通过单独编写类型文件供识别读取
- 社区活跃，目前三大框架还有越来越多的库都支持 TypeScript 了

TypeScript 只会在编译的时候对类型进行静态检查，如果发现有错误，编译的时候就会报错。

### 安装 TypeScript

1. 全局安装 TS

```
npm i -g typescript
```

2. 查看 TypeScript 版本号

```
tsc -v
```

我当前版本为 Version 4.0.2

3. 初始化生成 tsconfig.json 文件

```
tsc --init
```

4. 在 tsconfig.json 中设置源代码目录和编译生成 js 文件的目录

```json
"outDir": "./dist",
"rootDir": "./src"
```

5. 监听 ts 文件的变化，每当文件发生改变就自动编译

```
tsc -w
```

之后你写的 ts 文件编译错误都会直接提示，如果想运行文件，就到 `/dist`目录下找相应的 js 文件，使用 node 运行即可

## ts-node 安装

当然这样其实也挺麻烦，我们想直接运行 TS 文件, 这时可以借助`ts-node`插件

全局安装

```
npm install -g ts-node
```

找到文件路径，运行即可

```
ts-node demo.ts
```

## 基础类型

### Number 类型

```typescript
let num: number = 2
```

### Boolean 类型

```typescript
let isShow: boolean = true
```

### String 类型

```typescript
let str: string = 'hello'
```

### Array 类型

```typescript
let arr1: number[] = [1, 2, 3]
let arr2: Array<number> = [2, 3, 4]
```

### Any 类型

```typescript
let foo: any = 'hello'
foo = 12
foo = false
```

### Null 和 Undefined 类型

null 和 undefined 可以赋值给任意类型的变量

```typescript
let test1: undefined = undefined
let test2: null = null
let test3: number
let test4: string
test3 = null
test4 = undefined
```

### Void 类型

void 类型像是与 any 类型相反，它表示没有任何类型。当一个函数没有返回值时，其返回值类型是 void

```typescript
let test5: void = undefined // 声明一个 void 类型的变量没有什么用，因为它的值只能为 undefined 或 null
function testFunc(): void {} // 函数没有返回值
```

### Never 类型

never 类型表示的是那些永不存在的值的类型。

```typescript
function bar(): never {
  throw new Error('never reach')
}
```

### Unknown 类型

所有类型都可以赋值给 any，所有类型也都可以赋值给 unknown。

```typescript
let value: unknown
value = 123
value = 'Hello'
value = true

let value1: unknown = value
let value2: any = value
let value3: boolean = value // Error
let value4: number = value // Error
let value5: string = value // Error
let value6: object = value // Error
let value7: any[] = value // Error
let value8: Function = value // Error
```

unknown 类型只能被赋值给 any 类型和 unknown 类型本身。

### Tuple 类型

数组合并了相同类型的对象，而元组（Tuple）合并了不同类型的对象。元组表示一个数量和类型都已知的数组

```typescript
let tupleArr1: [string, number] = ['hello', 10]
// let tupleArr2: [string, number] = [10, 'hello'] // Error
```

### Enum 类型

使用枚举可以定义一些带名字的常量。 TypeScript 支持数字的和基于字符串的枚举。

```typescript
  enum Season {
    Spring,
    Summer,
    Autumn,
    Winter,
  }
  let a: Season = Season.Spring
  let b: Season = Season.Summer
  let c: Season = Season.Autumn
  let d: Season = Season.Winter
  console.log(a, b, c, d) // 0 1 2 3
}
```

## 函数类型

### 函数声明

```typescript
// JS
function func1(x, y) {
  return x + y
}
// TS
function func2(x: number, y: number): number {
  return x + y
}
```

### 函数表达式

在 TypeScript 的类型定义中，`=>` 用来表示函数的定义，左边是输入类型，需要用括号括起来，右边是输出类型。

```typescript
// JS
let func3 = function (x, y) {
  return x + y
}
// TS 第一种方式
let func4 = function (x: number, y: number): number {
  return x + y
}
// TS 第二种方式
// => 用来表示函数的定义，左边是输入类型，需要用括号括起来，右边是输出类型。
let func5: (x: number, y: number) => number = function (x: number, y: number): number {
  return x + y
}
```

### 接口定义函数的形状

采用函数表达式|接口定义函数的方式时，对等号左侧进行类型限制，可以保证以后对函数名赋值时保证参数个数、参数类型、返回值类型不变。

```typescript
interface SearchFunc {
  (source: string, subString: string): boolean
}

let mySearch: SearchFunc
mySearch = function (source: string, subString: string) {
  return source.search(subString) !== -1
}
```

### 可选参数

对于 TS 的函数，输入多余的（或者少于要求的）参数，是不允许的，那么该怎么设置可选参数呢？，我们可以在声明之后加一个问号

```typescript
function getName1(firstName: string, lastName?: string) {
  // 可选参数必须接在必需参数后面
  if (lastName) {
    return `${firstName} ${lastName}`
  } else {
    return firstName
  }
}
let name1 = getName1('jacky')
let name2 = getName1('jacky', 'lin')
console.log(name1, name2)
```

### 参数默认值

```typescript
function getName2(firstName: string = 'monkey', lastName: string) {
  return `${firstName} ${lastName}`
}
console.log(getName2('jacky', 'Lin'))
console.log(getName2(undefined, 'Lin')) // monkey Lin
```

### void 和 never 类型

当函数没有返回值时，可以用 void 来表示。

当一个函数永远不会返回时，我们可以声明返回值类型为 never

```typescript
function func6(): void {
  // return null
}

function func7(): never {
  throw new Error('never reach')
}
```

### 剩余参数

```typescript
function push1(arr, ...items) {
  items.forEach(function (item) {
    arr.push(item)
  })
}
let a: any[] = []
push1(a, 1, 2, 3)

function push2(arr: any[], ...items: any[]): void {
  items.forEach(function (item) {
    arr.push(item)
  })
}
let b: any[] = []
push1(b, 1, 2, 3, '5')
```

### 函数参数为对象（解构）时

```typescript
// js写法
function add({ one, two }) {
  return one + two
}

const total = add({ one: 1, two: 2 })

// ts 写法
function add1({ one, two }: { one: number; two: number }): number {
  return one + two
}

const three = add1({ one: 1, two: 2 })
```

### 函数重载

重载允许一个函数接受不同数量或类型的参数时，作出不同的处理。

```typescript
function reverse(x: number): number // 函数定义
function reverse(x: string): string // 函数定义
function reverse(x: number | string): number | string {
  // 函数实现
  if (typeof x === 'number') {
    return Number(x.toString().split('').reverse().join(''))
  } else if (typeof x === 'string') {
    return x.split('').reverse().join('')
  }
}
```

## 类型断言

TypeScript 允许你覆盖它的推断，并且能以你任何你想要的方式分析它，这种机制被称为「类型断言」。

TypeScript 类型断言用来告诉编译器你比它更了解这个类型，你知道你自己在干什么，并且它不应该再发出错误。

```typescript
interface Cat {
  name: string
  run(): void
}

interface Fish {
  name: string
  swim(): void
}

// 只能访问共有的属性
function getName(animal: Cat | Fish): string {
  return animal.name
}

// 将 animal 断言成 Fish 就可以解决访问 animal.swim 时报错的问题
function isFish(animal: Cat | Fish): boolean {
  if (typeof (animal as Fish).swim === 'function') {
    return true
  }
  return false
}

// 任何类型都可以被断言为 any
;(window as any).randomFoo = 1
```

```typescript
// 类型断言第一种方式："尖括号"语法
let value1: any = 'hello'
let value1Length: number = (<string>value1).length

// 类型断言第二种方式：as
let value2: any = 'world'
let value2Length: number = (value2 as string).length
```

## 类

### 存取器

```typescript
class Animal {
  constructor(name: string) {
    this.name = name
  }
  // getter
  get name() {
    return '名字'
  }
  // setter
  set name(value: string) {
    console.log('setter: ' + value)
  }
}

let animal = new Animal('monkey')
console.log(animal.name) // monkey
animal.name = 'mk' // setter: mk
```

### 访问修饰符

- public 修饰的属性或方法是公有的，可以在任何地方被访问到，默认所有的属性和方法都是 public 的
- private 修饰的属性或方法是私有的，不能在声明它的类的外部访问
- protected 修饰的属性或方法是受保护的，它和 private 类似，区别是它在子类中也是允许被访问的

```typescript
class Person {
  public name
  private age
  protected sex
  public constructor(name: string, age: number, sex: string) {
    this.name = name
    this.age = age
    this.sex = sex
  }
}

let person1 = new Person('jacky', 22, 'man')
person1.name = 'monkey' // name值可以访问且修改
// person1.age // Property 'age' is private and only accessible within class 'Person'.
// person1.sex // Property 'sex' is private and only accessible within class 'Person'.

class Person1 extends Person {
  constructor(name: string, age: number, sex: string) {
    super(name, age, sex)
    // console.log(this.name, this.age, this.sex) // Property 'age' is private and only accessible within class 'Person'.
  }
}
```

### 参数属性

同时给类中定义属性的同时赋值

```typescript
class Animal3 {
  public name
  constructor(name: string) {
    this.name = name
  }
}
console.log(new Animal3('animal3').name) // animal3

class Animal2 {
  constructor(public name: string) {} // 简洁形式
}
console.log(new Animal2('animal2').name) // animal2
```

- readonly 只读属性

```typescript
class Animal4 {
  readonly name
  constructor(name: string) {
    this.name = name
  }
}
let animal4 = new Animal4('animal4')
// animal4.name = '5' // Cannot assign to 'name' because it is a read-only property
```

### 抽象类

```typescript
// 抽象类是行为的抽象，一般来封装公共属性的方法的，不能被实例化
abstract class CommonAnimal {
  name: string
  abstract speak(): void
}
```

### static

```typescript
class Animal5 {
  static sayHi() {
    console.log('Hello Animal5')
  }
}
Animal5.sayHi()
```

## 联合类型

联合类型（Union Types）表示取值可以为多种类型中的一种。使用一个`|`分割符来分割多种类型

```typescript
let foo: string | number | boolean
foo = 'test'
foo = 3
foo = true
```

## 交叉类型

```typescript
interface IPerson {
  id: string
  age: number
}

interface IWorker {
  companyId: string
}

type IStaff = IPerson & IWorker

const staff: IStaff = { id: '007', age: 24, companyId: '1' }
```

## 类型别名

```typescript
type Message = string | string[]

let getMsg = (message: Message) => {
  return message
}

type Weather = 'SPRING' | 'SUMMER' | 'AUTUMN' | 'WINTER'
let weather1: Weather = 'SPRING'
let weather2: Weather = 'AUTUMN'
```

## 接口

### 对象的形状

```typescript
interface Person1 {
  name: string
  age: number
}
let person1: Person1 = {
  name: 'jacky',
  age: 23,
}
```

### 描述行为的抽象

```typescript
interface AnimalLike {
  eat(): void
  move(): void
}

interface PersonLike extends AnimalLike {
  speak(): void
}

class Human implements PersonLike {
  speak() {}
  eat() {}
  move() {}
}
```

### 含构建函数作参数的写法

```typescript
class Animal1 {
  constructor(public name: string) {}
  age: number
}

class Animal2 {
  constructor(public age: number) {}
}

interface WithNameClass {
  new (name: string): Animal1
}

function createClass(classname: WithNameClass, name: string) {
  return new classname(name)
}

let instance1 = createClass(Animal1, 'monkey')
// let instance2 = createClass(Animal2, 'monkey') // 没有name属性则报错
```

### 其它任意属性

```typescript
interface Person2 {
  readonly id: number
  [propName: string]: any //任意属性
}
```

## 泛型

泛型是指定义函数、接口或类的时候，不预先指定具体的类型，而在使用的时候再指定类型的一种特性；通常用 T 表示，但不是必须使用改字母，只是常规，通常还有其他常用字母：

- T（Type）：表示一个 TypeScript 类型
- K（Key）：表示对象中的键类型
- V（Value）：表示对象中的值类型
- E（Element）：表示元素类型

### 泛型类

```typescript
class GenericNumber<T> {
  name: T
  add: (x: T, y: T) => T
}

let generic = new GenericNumber<number>()
generic.name = 123
```

### 泛型数组

- 写法一

```typescript
function func<T>(params: T[]) {
  return params
}
func<string>(['1', '2'])
func<number>([1, 2])
```

- 写法二

```typescript
function func1<T>(params: Array<T>) {
  return params
}
func1<string>(['1', '2'])
func1<number>([1, 2])
```

### 泛型接口

可以用来约束函数

```typescript
interface Cart<T> {
  list: T[]
}
let cart: Cart<number> = { list: [1, 2, 3] }
```

### 泛型别名

```typescript
type Cart2<T> = { list: T[] } | T[]
let c1: Cart2<number> = { list: [1, 2, 3] }
let c2: Cart2<number> = [1, 2, 3]
```

泛型接口 VS 泛型别名

- 接口创建了一个新的名字，它可以在其他任意地方被调用。而类型别名并不创建新的名字
- 类型别名不能被 extends 和 implements，这时我们应该尽量使用接口代替类型别名
- 当我们需要使用联合类型或者元组类型的时候，类型别名会更合适

### 多个泛型

```typescript
// 不借助中间变量交换两个变量的值
function swap<T, P>(tuple: [T, P]): [P, T] {
  return [tuple[1], tuple[0]]
}

let ret = swap([1, 'a'])
ret[0].toLowerCase()
ret[1].toFixed(2)
```

### 默认泛型

```typescript
function createArray<T = number>(length: number, value: T): T[] {
  let arr: T[] = []
  for (let i = 0; i < length; i++) {
    arr[i] = value
  }
  return arr
}
let arr = createArray(3, 9)
```

### 泛型约束(继承)

```typescript
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

### 泛型工具类型

为了方便开发者 TypeScript 内置了一些常用的工具类型，比如 Partial、Required、Readonly、Pick 等。它们都是使用 keyof 实现。

keyof 操作符可以用来一个对象中的所有 key 值。

```typescript
interface Person1 {
  name: string
  age: number
  sex?: string
}
type PersonKey = keyof Person1 // 限制 key 值的取值
function getValueByKey(p: Person1, key: PersonKey) {
  return p[key]
}
```

### 内置的工具类型

```typescript
// type Partial<T> = {[P in keyof T]?: T[P]}
type PersonSearch = Partial<Person> // 全部变可选

// type Required<T> = { [P in keyof T]-?: T[P] }
type PersonRequired = Required<Person> // 全部变必选

// type ReadOnly<T> = { readonly [P in keyof T]: T[P] }
type PersonReadOnly = Readonly<Person> // 全部变只读

// type Pick<T, K extends keyof T>  = {[P in K]: T[P]}
type PersonSub = Pick<Person, 'name'> // 通过从Type中选择属性Keys的集合来构造类型。
```

### 类数组

```typescript
let root = document.getElementById('root')
let children: HTMLCollection = root!.children // ! 代表非空断言操作符
let childNodes: NodeListOf<ChildNode> = root!.childNodes
```

## 类型保护

更明确的判断某个分支作用域中的类型，主要尝试检测属性、方法或原型，以确定如何处理值。

### typeof

```typescript
function double(input: string | number | boolean): number {
  // 基本数据类型的类型保护
  if (typeof input === 'string') {
    return input.length
  } else if (typeof input === 'number') {
    return input
  } else {
    return 0
  }
}
```

### instanceof

```typescript
class Monkey {
  climb: string
}
class Person {
  sports: string
}
function getAnimalName(animal: Monkey | Person) {
  if (animal instanceof Monkey) {
    console.log(animal.climb)
  } else {
    console.log(animal.sports)
  }
}
```

### in

```typescript
class Student {
  name: string
  play: string[]
}
class Teacher {
  name: string
  teach: string
}
type SchoolRole = Student | Teacher
function getRoleInformation(role: SchoolRole) {
  if ('play' in role) {
    console.log(role.play)
  }
  if ('teach' in role) {
    console.log(role.teach)
  }
}
```

### 自定义类型保护

比如有时两个类型有不同的取值，也没有其他可以区分的属性

```typescript
interface Bird {
  leg: number
}
interface Dog {
  leg: number
}
function isBird(x: Bird | Dog): x is Bird {
  return x.leg === 2
}
function getAnimal(x: Bird | Dog): string {
  if (isBird(x)) {
    return 'bird'
  } else {
    return 'dog'
  }
}
```

上述完整代码示例：[typescript-tutorial](https://github.com/Jacky-Summer/typescript-tutorial)

<br>

- ps： [个人技术博文 Github 仓库](https://github.com/Jacky-Summer/personal-blog)，觉得不错的话欢迎 star，给我一点鼓励继续写作吧~
