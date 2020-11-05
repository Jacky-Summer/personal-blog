## 前言

前端测试普及度虽然不算高，但对项目代码质量以及后期维护项目上却有很大的作用。本文会抛砖引玉的讲下 Jest 单元测试，因为很多 API 具体看官网就可以了。

## 什么是 Jest

Jest 是用来创建、执行和构建测试用例的一个 JavaScript 测试
库。可以在任何项目中安装使用它，如 Vue/React/Angular/Node/TypeScript 等。

## 使用 Jest 的好处

目前前端测试在开发应用场景主要是 UI 组件库，但其实不止是组件库，日常项目开发也可以加入单元测试，保证项目的稳定。

它的好处有哪些呢？

- 速度快，使用简单和容易配置
- 通过编写测试，让代码出现 bug 的概率更小
- 有利于写出健壮性更好的代码，项目的可维护性增强
- 好的测试，就具有文档解释的作用
- 减少回归测试中的 bug

## 举个栗子

比如写一个加法和减法函数

```
function add(a, b) {
  return a + b
}

function minus(a, b) {
  return a - b
}
```

我们从最简单的函数开始入手讲起来，假设要对改函数进行测试，我们测试的到底是什么，即是他的功能，调用函数时传入两个参数，返回结果为他们相加的值。比如传入`add(1, 1)`，我们期望结果是返回相加结果`2`
写成代码可以这样`expect(add(1, 1)) === 2`，再封装得语义化一点的话，`expect(add(1, 1)).toBe(2)`为我们期望的测试函数，来实现这个函数：

```javascript
function expect(result) {
  return {
    toBe(actual) {
      if (result !== actual) {
        throw new Error(`与预期结果不相等。实际结果：${actual}，期望结果：${result}`)
      }
    },
  }
}
```

使用：

```javascript
expect(add(1, 1)).toBe(2)
expect(minus(2, 1)).toBe(1)
expect(add(2, 2)).toBe(3) // Error: 与预期结果不相等。实际结果：3，期望结果：4
```

测试不通过我们就让它抛出错误

这样写的话实际有点混乱，因为一眼看下去不知道在测试啥，于是，我们多封装一个函数，代表我们正在做什么测试

```
// desc 用来描述测试行为， fn则为我们执行测试的函数
function test(desc, fn) {
  try {
    fn()
    console.log(`${desc}通过测试`)
  } catch (e) {
    console.log(`${desc}测试不通过 ${e}`)
  }
}

test('加法测试', () => {
  expect(add(1, 1)).toBe(2)
})

test('减法测试', () => {
  expect(minus(2, 1)).toBe(1)
})
```

执行代码，输出

```
加法测试通过测试
减法测试通过测试
```

这就是我们动手的最简单的测试了，因为我们函数比较简单，所以你可能会觉得在做着啰嗦的工作。

从流程上看，一个简单的测试流程（输入 —— 预期输出 —— 验证结果）如下：

1. 引入要测试的函数
2. 传入测试函数执行结果
3. 定义预期输出
4. 检查函数是否返回了预期的输出结果

## 初始化环境

创建一个空文件夹进入初始化

```
npm init -y
```

安装 jest

```
yarn add --dev jest
```

在`package.json`配置脚本命令

```
{
  "scripts": {
    "test": "jest"
  }
}
```

新建文件`babel.config.js`，安装依赖并配置

```
yarn add --dev babel-jest @babel/core @babel/preset-env
```

```javascript
// babel.config.js
module.exports = {
  presets: [
    [
      '@babel/preset-env',
      {
        targets: {
          node: 'current',
        },
      },
    ],
  ],
}
```

接下来我们每段介绍都会采取`example.js`里面写代码，再到`*.test.js`文件引入`example.js`的函数并测试。

## 匹配器 Matchers

使用 Matchers 让你可以用各种方式测试你的代码

跟我们上面写的代码一样，jest 提供了`test`,`expect`,`toBe`方法

`example.js`

```javascript
export const multiply = (a, b) => {
  return a * b
}
```

新建文件`matcher.test.js`

```javascript
import { multiply } from './example'

test('测试乘法', () => {
  expect(multiply(4, 2)).toBe(8)
})
```

`toBe` 使用 `Object.is` 来测试精确相等。 如果想要检查对象的值，使用 `toEqual` 代替。

`matcher.test.js`

```javascript
test('测试对象是否相等', () => {
  const data = { name: 'jacky' }
  data['age'] = 23
  expect(data).toEqual({ name: 'jacky', age: 23 })
})
```

运行`yarn test`，可看到控制台显示两个测试通过；如不通过控制台也会提示哪个测试用例出错。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/804c59a4689b4b2bb980115f77e7d29f~tplv-k3u1fbpfcp-watermark.image)

当运行该命令的时候，jest 会自动搜寻项目中有`.test.js`后缀的文件，并运行测试。

当然，也有其他匹配器

```javascript
test('匹配器', () => {
  const n = null
  const isTrue = true
  const value = 3
  expect(n).toBeNull() // 只匹配 null
  expect(n).toBeDefined() // 只匹配 undefined
  expect(n).not.toBeUndefined() // 与 toBeUndefined 相反
  expect(isTrue).toBeTruthy() // 匹配任何 if 语句为真

  expect(value).toBeGreaterThan(2)
  expect(value).toBeLessThan(5)
  expect(value).toBeLessThanOrEqual(3)
})
```

## 测试异步代码

`example.js`

```javascript
import axios from 'axios'

export const fetchData = () => {
  return axios.get('https://jsonplaceholder.typicode.com/todos/1')
}
```

`async.test.js`

```javascript
import { fetchData } from './example'

test('测试 async await', async () => {
  const data = await fetchData()
  expect(data.data.id).toBe(1) // 请求api返回的id值是否为1
})

test('测试 Promise', () => {
  return fetchData().then(data => {
    expect(data.data.id).toBe(1)
  })
})
```

## mock 函数

如果我们有很多异步请求的函数需要测试，会使测试变得缓慢，那么上述用真实接口请求的方式就不太适合了；或者想要捕获对函数的调用，这时就需要 Mock，它可以做到：

1. 捕获对函数的调用，以及 this 和调用顺序
2. 它允许测试时配置返回值
3. 改变函数的内部实现

mock.test.js

```javascript
import axios from 'axios'
import { fetchData } from './example'
jest.mock('axios') // 自动模拟 axios 模块。

test('测试请求接口', async () => {
  axios.get.mockResolvedValue({ data: 'hello' }) // 返回假数据 { data: 'hello' } 用于测试
  await fetchData().then(data => {
    expect(data).toEqual({ data: 'hello' })
  })
})
```

可能你会想，这样测试有什么用，都是假数据，作为前端这边，我们只要确保函数被调用和正常返回结果就够了；当然这个例子是比较简单，涉及复杂接口测试的话还有其他 API 可用。其他更深入具体的接口请求测试是后端要做的。

在`example.js`增加一个函数

```javascript
// 执行该函数需要传入一个回调函数，然后回调函数传递一个参数并执行
export function runFn(fn) {
  fn('函数执行了')
}
```

如果这是一个复杂的函数，我们并不想测试就让它执行，就可以用 mock 来测试

```javascript
test('函数被调用了', () => {
  const fn = jest.fn()
  runFn(fn) // 调用函数

  expect(fn).toHaveBeenCalled() // 期望fn被调用了
  expect(fn).toHaveBeenLastCalledWith('函数执行了') // 函数fn被调用时的参数是否符合
})
```

此外，还可以自定义设置返回参数，测试被执行次数等等，具体查官方文档。

## Snapshot

每当我们想要确保 UI（指 html css 等）或者配置文件不会改变时，可以使用快照测试。

这里只举例配置文件

`example.test.js`

```javascript
export function getConfig() {
  return {
    port: 8080,
    url: 'http://localhost:3000',
  }
}
```

`snapshot.test.js`

```javascript
import { getConfig } from './example'

test('测试配置文件有没有更改', () => {
  expect(getConfig()).toMatchSnapshot()
})
```

运行`npm run test`，第一次运行，会发现文件所在位置生成一个文件夹`__snapshots__`,里面有个`snapshot.test.js.snap`文件，里面内容是：

```
// Jest Snapshot v1, https://goo.gl/fbAQLP

exports[`测试配置文件有没有更改 1`] = `
Object {
  "port": 8080,
  "url": "http://localhost:3000",
}
`;
```

里面保存了我们配置信息，如果你不小心改了配置信息的话，在执行测试就会报错，因为与这个文件的内容不符合；这时你就知道了你改到不应该改的位置了
比如把端口好改成`8081`，再运行`npm run test`

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d398cf43a34149d69a0868bbcd8833de~tplv-k3u1fbpfcp-watermark.image)

但如果真的要改的话呢，那就按提示里面的信息做，更新快照，执行：

```
npm test -- -u
```

执行后就会在终端看到：

```
Snapshots:   1 updated, 1 total
```

这次再执行测试就不会报错了，就把配置信息更新了。

## Timers Mocks

当要测试有定时器的函数时，比如过 5 秒执行什么任务，对于我们来说并不是很方便，要真正等待它经过一定时间执行，这是可以模拟定时器函数。

`example.js`

```javascript
export function timerGame(callback) {
  console.log('Ready....go!')
  setTimeout(() => {
    console.log("Time's up -- stop!")
    callback && callback()
  }, 3000)
}
```

```javascript
import { timerGame } from './example'

jest.useFakeTimers()

test('测试一定时间后执行函数', () => {
  const fn = jest.fn()
  timerGame(fn)
  jest.advanceTimersByTime(3000) // 把时间提前了三秒
  expect(fn).toBeCalled() // 函数被调用了
  expect(fn).toHaveBeenCalledTimes(1) // 函数被执行了1次
})
```

## 测试覆盖率

增加一个脚本命令，将测试覆盖率信息输出为报告

```
"coverage": "jest --coverage"
```

执行该命令
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/aa658fe9b80540b9b9e5e8dee53af03c~tplv-k3u1fbpfcp-watermark.image)

可以在终端看到生成了报告，上述例子的代码测试覆盖率是 100%，这是因为测试的函数比较简单，所以轻易 100%。

同时项目根目录也成功了一个文件夹`coverage`，里面的`lcov-report`目录里生成 HTML 文件，可以通过浏览器打开查看。

如下：
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6d07e062bd224e21819cbd3118fec6ed~tplv-k3u1fbpfcp-watermark.image)

## TDD 与 BDD

现在流行的单元测试风格主要有 TDD（测试驱动开发）和 BDD（行为驱动开发）两种。比较明显的区别是一个是先写测试用例再写代码，一个是先写代码再写测试用例。

### TDD

TDD 的开发流程一般是先编写测试用例，再编写代码。

比如写一个 Todolist，我们一开始先想好它的功能和注意点，每一个功能都具备对应的测试用例，编写测试用例，没写代码之前肯定是通不过的；然后写完开始编写代码，使测试用例通过测试，再继续重复步骤，完成开发。这样写出来的代码质量和维护性也会更好。

所以 TDD 一般就是作为单元测试，即是单一组件功能上测试得较为完善，但几个组件加起来测试这种却不一定能测试完善。

单元测试应用比较多的是函数库，可对每个函数进行单独细致测试。

单元测试比较独立，但过于独立也会隐藏一些潜在问题无法测到，这个时候就有集成测试辅助了。

### BDD

BDD 的开发流程一般是先编写代码，再写测试用例。

这个就比较符合我们以往的开发模式，不过 BDD 更关注的是整体的行为是否符合预期，一般结合集成测试，也就是说整体好多个组件整体业务测试这种。

集成测试重点关心的是结果，而不是代码，更适合业务开发。

## 结语

两种测试风格都有它自己的优缺点，这里只是举例一部分；整体简单理解下来，前端测试能让我们写出更好的代码，减少 bug 的产生；而且，在维护一些老项目方面，也是有优势，比如你在改动旧项目代码的时候，可能怕影响其他组件或功能但你无法知晓，但如果有测试，直接改完运行所有测试代码就知道有没有影响到其他的功能代码了；而如果老项目没有写单元/集成测试，为了更好的维护，你也可以加入测试代码。

如果要学习更多有关 Jest 测试的内容，当然是直接去 [Jest 官网](https://jestjs.io/) 看了，然后在实际开发运用中才会逐渐体会到它带来的好处。

[本文代码地址](https://github.com/Jacky-Summer/jest-tutorial)

- ps： [个人技术博文 Github 仓库](https://github.com/Jacky-Summer/personal-blog)，觉得不错的话欢迎 star，给我一点鼓励继续写作吧~
