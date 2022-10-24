# 一文解读 React 17 与 React 18 的更新变化

## 前言

项目目前 react17 和 react18 都有使用，但在开发者角度绝大部分场景还是感知不到多大变化，但也要详细了解清楚具体更新了什么。本文就来一次性梳理下 react17 与 react18 的变化。

## React 17 更新

首先，官方发布日志称 react17 最大的特点就是[无新特性](https://reactjs.org/blog/2020/10/20/react-v17.html#no-new-features)，这个版本主要目标是让 React 能渐进式升级，它允许多版本混用共存，可以说是为更远的未来版本做准备了。

### 去除事件池

在 React17 之前，如果使用异步的方式来获取事件 e 对象，会发现合成事件对象被销毁，如下：

```tsx
function App() {
  const handleClick = (e: React.MouseEvent) => {
    console.log('直接打印e', e.target) // <button>React事件池</button>
    // v17以下在异步方法拿不到事件e，必须先调用 e.persist()
    // e.persist()

    // 异步方式获取事件e
    setTimeout(() => {
      console.log('setTimeout打印e', e.target) // null
    })
  }
  return (
    <div className="App">
      <button onClick={handleClick}>React事件池</button>
    </div>
  )
}
```

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/58521a933adb4200b379c9b46769b072~tplv-k3u1fbpfcp-watermark.image?)
如果你需要在事件处理函数运行之后获取事件对象的属性，你需要调用  `e.persist()`，它会将当前的合成事件从事件池中删除，并允许保留对事件的引用。

> 事件池：合成事件对象会被放入池中统一管理。这意味着合成事件对象可以被复用，当所有事件处理函数被调用之后，其所有属性都会被回收释放置空。

事件池的好处是在较旧浏览器中重用了不同事件的事件对象以提高性能，但它对现代浏览器的性能优化微乎其微，反而给开发者带来困惑，因此去除了事件池，因此也没有了事件复用机制。

```tsx
function App() {
  // v17 去除了 React 事件池，异步方式使用e不再需要 e.persist()
  const handleClick = (e: React.MouseEvent) => {
    console.log('直接打印e', e.target) // <button>React事件池</button>

    setTimeout(() => {
      console.log('setTimeout打印e', e.target) // <button>React事件池</button>
    })
  }
  return (
    <div className="App">
      <button onClick={handleClick}>React事件池</button>
    </div>
  )
}
```

### 事件委托到根节点

reactv17 前，React 将事件委托到 document 上，在 react17 中，则委托到根节点

```tsx
const rootNode = document.getElementById('root')
ReactDOM.render(<App />, rootNode)
```

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6bcb3ca9b21b47f79c70883f6f264012~tplv-k3u1fbpfcp-watermark.image?)

```tsx
import { useState, useEffect } from 'react'

function App() {
  const [isShowText, setIsShowText] = useState(false)

  const handleShowText = (e: React.MouseEvent) => {
    // e.stopPropagation() // v16无效
    // e.nativeEvent.stopImmediatePropagation() // 阻止监听同一事件的其他事件监听器被调用
    setIsShowText(true)
  }

  useEffect(() => {
    document.addEventListener('click', () => {
      setIsShowText(false)
    })
  }, [])

  return (
    <div className="App">
      <button onClick={handleShowText}>事件委托变更</button>

      {isShowText && <div>展示文字</div>}
    </div>
  )
}
```

如上代码，在 react16 和 v17 版本，点击按钮时，都不会显示文字。这是因为 react 的合成事件是基于事件委托的，有事件冒泡，先执行 React 事件，再执行 document 上挂载的事件。

v16：出于对冒泡的了解，我们直接在按钮事件上加`e.stopPropagation()`，这样就不会冒泡到 document，`isShowText` 也不会被置为 false 了。但由于 v16 版本的事件委托是绑在`document`上的，它的事件源跟`document`就是同级了，而不是上下级，所以`e.stopPropagation()`并没有起作用。如果要阻止冒泡，可以使用原生的
`e.nativeEvent.stopImmediatePropagation()`阻止同级冒泡，这样文字就可以显示了。

v17：由于事件委托到根目录 root 节点，与`document`属于上下级关系，所以可以直接使用`e.stopPropagation()`阻止

> stopImmediatePropagation() 方法可以阻止监听同一事件的其他事件监听器被调用

这种更新不仅方便了局部使用 React 的项目，还可以用于项目的渐进式升级，解决不同版本的 React 组件嵌套使用时，e.stopPropagation()无法正常工作的问题

### 更贴近原生浏览器事件

对事件系统进行了一些较小的更改：

- `onScroll` 事件不再冒泡，以防止[出现常见的混淆](https://github.com/facebook/react/issues/15723)
- React 的 `onFocus` 和 `onBlur` 事件已在底层切换为原生的 `focusin` 和 `focusout` 事件。它们更接近 React 现有行为，有时还会提供额外的信息。

> blur、focus 和 focusin、focusout 的区别：blur、focus 不支持冒泡，focusin、focusout 支持冒泡

- 捕获事件（例如，`onClickCapture`）现在使用的是实际浏览器中的捕获监听器。

这些更改会使 React 与浏览器行为更接近，并提高了互操作性。

> 尽管 React 17 底层已将 onFocus 事件从 focus 切换为 focusin，但请注意，这并未影响冒泡行为。在 React 中，onFocus 事件总是冒泡的，在 React 17 中会继续保持，因为通常它是一个更有用的默认值

### 全新的 JSX 转换器

总结下来就是两点：

- 用 `jsx()` 函数替换 `React.createElement()`
- 运行时自动引入 `jsx()` 函数，无需手写引入`react`

在 v16 中，我们写一个 React 组件，总要引入

```tsx
import React from 'react'
```

这是因为在浏览器中无法直接使用 jsx，所以要借助工具如`@babel/preset-react`将 jsx 语法转换为 `React.createElement` 的 js 代码，所以需要显式引入 React，才能正常调用 createElement。

通过`React.createElement()` 创建元素是比较频繁的操作，本身也存在一些问题，无法做到性能优化，具体可见官方优化的 [动机](https://github.com/reactjs/rfcs/blob/createlement-rfc/text/0000-create-element-changes.md#motivation)

v17 之后，React 与 Babel 官方进行合作，直接通过将 `react/jsx-runtime` 对 jsx 语法进行了新的转换而不依赖 `React.createElement`，因此 v17 使用 jsx 语法可以不引入 React，应用程序依然能正常运行。

```tsx
function App() {
  return <h1>Hello World</h1>
}

// 新的 jsx 转换为
// 由编译器引入（禁止自己引入！）
import { jsx as _jsx } from 'react/jsx-runtime'

function App() {
  return _jsx('h1', { children: 'Hello world' })
}
```

**如何升级至新的 JSX 转换**

- 更新至支持新转换的 React 版本（v17）

> 如果你还在使用 v16，也可升级至 React [v16.14.0](https://github.com/facebook/react/releases/tag/v16.14.0) 的版本，官方在该版本也支持这个特性。

- 修改配置

1. `@babel/preset-react`编译增加 `runtime: 'automatic'`配置

```tsx
// 如果你使用的是 @babel/preset-react
{
  "presets": [
    ["@babel/preset-react", {
      "runtime": "automatic"
    }]
  ]
}

// 如果你使用的是 @babel/plugin-transform-react-jsx
{
  "plugins": [
    ["@babel/plugin-transform-react-jsx", {
      "runtime": "automatic"
    }]
  ]
}
```

2. 修改 `tsconfig.json` 配置，[具体配置可见 TS 官方文档](https://www.typescriptlang.org/tsconfig#jsx)

```json
{
  "compilerOptions": {
    // "jsx": "react",
    "jsx": "react-jsx"
  }
}
```

> 从 Babel 8 开始，"automatic" 会将两个插件默认集成在 rumtime 中

### 副作用清理时机

```tsx
useEffect(() => {
  // This is the effect itself.
  return () => {
    // This is its cleanup.
  }
})
```

- v17 前，组件被卸载时，`useEffect`的清理函数都是同步运行的；对于大型应用程序来说，同步会减缓屏幕的过渡（如切换标签）
- v17 后，`useEffect` 副作用清理函数是**异步执行**的，如果要卸载组件，则清理会在屏幕更新后运行

此外，v17 将在运行任何新副作用之前执行所有副作用的清理函数（针对所有组件），v16 只对组件内的副作用保证这种顺序。

不过需要注意

```tsx
useEffect(() => {
  someRef.current.someSetupMethod()
  return () => {
    someRef.current.someCleanupMethod()
  }
})
```

问题在于 someRef.current 是可变的且因为异步的，在运行清除函数时，它可能已经设置为 null。

```tsx
// 用一个变量量在 ref 每次变化时，将 someRef.current 保存起来，放到副作用清理回调函数的闭包中，来保证不可变性。
useEffect(() => {
  const instance = someRef.current
  instance.someSetupMethod()

  return () => {
    instance.someCleanupMethod()
  }
})
```

或者用 `useLayoutEffect`

```tsx
useLayoutEffect(() => {
  someRef.current.someSetupMethod()
  return () => {
    someRef.current.someCleanupMethod()
  }
})
```

useLayoutEffect 可以保证回调函数同步执行，这样就能确保 ref 此时还是最后的值。

### 返回一致的 undefined 错误

在 v17 以前，组件返回`undefined`始终是一个错误。但是有漏网之鱼，React 只对类组件和函数组件执行此操作，但并不会检查 `forwardRef` 和 `memo` 组件的返回值。

```tsx
function Button() {
  return // Error: Nothing was returned from render
}

function Button() {
  // We forgot to write return, so this component returns undefined.
  // React surfaces this as an error instead of ignoring it.
  ;<button />
}
```

在 v17 中修复了这个问题，forwardRef 和 memo 组件的行为会与常规函数组件和类组件保持一致，在返回 undefined 时会报错

```tsx
let Button = forwardRef(() => {
  // We forgot to write return, so this component returns undefined.
  // React 17 surfaces this as an error instead of ignoring it.
  ;<button />
})

let Button = memo(() => {
  // We forgot to write return, so this component returns undefined.
  // React 17 surfaces this as an error instead of ignoring it.
  ;<button />
})
```

### 原生组件栈

v16 中错误调用栈的缺点：

- 缺少源码位置追溯，在控制台无法点击跳转到到出错的地方
- 无法适用于生产环境

整体来说不如原生的 JavaScript 调用栈，不同于常规压缩后的 JavaScript 调用栈，它们可以通过 sourcemap 的形式自动恢复到原始函数的位置，而使用 React 组件栈，在生产环境下必须在调用栈信息和 bundle 大小间进行选择。

在 v17 使用了不同的机制生成组件调用栈，直接从 JavaScript 原生错误栈生成的，所以在生产环境也能按 sourcemap 还原回来，且支持点击跳到源码位置。

想详细了解的可见该 [PR](https://github.com/facebook/react/pull/18561)

### 移除私有导出 API

v17 删除了一些私有 API，主要是 [React Native for Web](https://github.com/necolas/react-native-web) 使用的

另外，还删除了`ReactTestUtils.SimulateNative`工具方法，因为其行为与语义不符，如果你想要一种简便的方式来触发测试中原生浏览器的事件，可直接使用 [React Testing Library](https://testing-library.com/docs/dom-testing-library/api-events)

### 启发式更新算法更新

引用 [React17 新特性：启发式更新算法](https://juejin.cn/post/6860275004597239815)

- React16 的`expirationTimes`模型只能区分是否>=`expirationTimes`决定节点是否更新。
- React17 的`lanes`模型可以选定一个更新区间，并且动态的向区间中增减优先级，可以处理更细粒度的更新。

这个我目前也不是太清楚具体算法，先不展开了有兴趣的可去查阅相关资料

## React 18 更新

### 并发模式

v18 的新特性是使用现代浏览器的特性构建的，彻底放弃对 IE 的支持。

v17 和 v18 的区别就是：从同步不可中断更新变成了异步可中断更新，v17 可以通过一些试验性的 API 开启并发模式，而 v18 则全面开启并发模式。

> 并发模式可帮助应用保持响应，并根据用户的设备性能和网速进行适当的调整，该模式通过使渲染可中断来修复阻塞渲染限制。在 Concurrent 模式中，React 可以同时更新多个状态。

这里参考下文区分几个概念：

- **并发模式**是实现**并发更新**的基本前提
- v18 中，以是否使用**并发特性**作为是否开启**并发更新**的依据。
- **并发特性**指开启**并发模式**后才能使用的特性，比如：`useDeferredValue`/`useTransition`

可阅读参考 [Concurrent Mode（并发模式）](https://juejin.cn/post/7094037148088664078#heading-23)

### 更新 render API

v18 使用 ReactDOM.createRoot() 创建一个新的根元素进行渲染，使用该 API，会自动启用并发模式。如果你升级到 v18，但没有使用`ReactDOM.createRoot()`代替`ReactDOM.render()`时，控制台会打印错误日志要提醒你使用 React，该警告也意味此项变更没有造成 breaking change，而可以并存，当然尽量是不建议。

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5fa61c46feb94c8b9ff7e71542763b94~tplv-k3u1fbpfcp-watermark.image?)

```tsx
// v17
import ReactDOM from 'react-dom'
import App from './App'

ReactDOM.render(<App />, document.getElementById('root'))

// v18
import ReactDOM from 'react-dom/client'
import App from './App'

ReactDOM.createRoot(document.getElementById('root') as HTMLElement).render(<App />)
```

### 自动批处理

批处理是指 React 将多个状态更新，聚合到一次 render 中执行，以提升性能

在 v17 的批处理只会在事件处理函数中实现，而在 Promise 链、异步代码、原生事件处理函数中失效。而 v18 则所有的更新都会自动进行批处理。

```tsx
// v17
const handleBatching = () => {
  // re-render 一次，这就是批处理的作用
  setCount((c) => c + 1)
  setFlag((f) => !f)
}

// re-render两次
setTimeout(() => {
  setCount((c) => c + 1)
  setFlag((f) => !f)
}, 0)

// v18
const handleBatching = () => {
  // re-render 一次
  setCount((c) => c + 1)
  setFlag((f) => !f)
}
// 自动批处理：re-render 一次
setTimeout(() => {
  setCount((c) => c + 1)
  setFlag((f) => !f)
}, 0)
```

如果在某些场景不想使用批处理，可以使用 `flushSync`退出批处理，强制同步执行更新。

> flushSync 会以函数为作用域，函数内部的多个 setState 仍然是批量更新

```tsx
const handleAutoBatching = () => {
  // 退出批处理
  flushSync(() => {
    setCount((c) => c + 1)
  })
  flushSync(() => {
    setFlag((f) => !f)
  })
}
```

### Suspense 支持 SSR

SSR 一次页面渲染的流程：

1. 服务器获取页面所需数据
2. 将组件渲染成 HTML 形式作为响应返回
3. 客户端加载资源
4. （hydrate）执行 JS，并生成页面最终内容

上述流程是串行执行的，v18 前的 SSR 有一个问题就是它不允许组件"等待数据"，必须收集好所有的数据，才能开始向客户端发送 HTML。如果其中有一步比较慢，都会影响整体的渲染速度。

v18 中使用并发渲染特性扩展了`Suspense`的功能，使其支持流式 SSR，将 React 组件分解成更小的块，允许服务端一点一点返回页面，尽早发送 HTML 和选择性的 hydrate， 从而可以使 SSR 更快的加载页面

```tsx
<Suspense fallback={<Spinner />}>
  <Comments />
</Suspense>
```

具体可参考文章 [React 18 中新的 Suspense SSR 架构](https://juejin.cn/post/6982010092258328583)

### startTransition

Transitions 是 React 18 引入的一个全新的并发特性。它允许你将标记更新作为一个 transitions（过渡），这会告诉 React 它们可以被中断执行，并避免回到已经可见内容的 Suspense 降级方案。本质上是用于一些不是很急迫的更新上，用来进行并发控制

在 v18 之前，所有的更新任务都被视为急迫的任务，而 Concurrent Mode 模式能将渲染中断，可以让高优先级的任务先更新渲染。

React 的状态更新可以分为两类：

- 紧急更新：比如点击按钮、搜索框打字是需要立即响应的行为，如果没有立即响应给用户的体验就是感觉卡顿延迟
- 过渡/非紧急更新：将 UI 从一个视图过渡到另一个视图。一些延迟可以接受的更新操作，不需要立即响应

`startTransition` API 允许将更新标记为非紧急事件处理，被`startTransition`包裹的会延迟更新的 state，期间可能被其他紧急渲染所抢占。因为 React 会在高优先级更新渲染完成之后，才会渲染低优先级任务的更新

React 无法自动识别哪些更新是优先级更高的。比如用户的键盘输入操作后，setInputValue 会立即更新用户的输入到界面上，是紧急更新。而 setSearchQuery 是根据用户输入，查询相应的内容，是非紧急的。

```tsx
const [inputValue, setInputValue] = useState()

const onChange = (e) => {
  setInputValue(e.target.value) // 更新用户输入值（用户打字交互的优先级应该要更高）
  setSearchQuery(e.target.value) // 更新搜索列表（可能有点延迟，影响）
}

return <input value={inputValue} onChange={onChange} />
```

React 无法自动识别，所以它提供了 `startTransition`让我们手动指定哪些更新是紧急的，哪些是非紧急的，从而让我们改善用户交互体验。

```tsx
// 紧急的更新
setInputValue(e.target.value)
// 开启并发更新
startTransition(() => {
  setSearchQuery(input) // 非紧急的更新
})
```

这里有个比较好的[在线例子](https://react-fractals-git-react-18-swizec.vercel.app/)，可以直接感受到 `startTransition`的优化

### useTransition

当有过渡任务（非紧急更新）时，我们可能需要告诉用户什么时候当前处于 pending（过渡） 状态，因此 v18 提供了一个带有`isPending`标志的 Hook `useTransition`来跟踪 transition 状态，用于过渡期。

`useTransition` 执行返回一个数组。数组有两个状态值：

- `isPending`: 指处于过渡状态，正在加载中
- `startTransition`: 通过回调函数将状态更新包装起来告诉 React 这是一个过渡任务，是一个低优先级的更新

```tsx
function TransitionTest() {
  const [isPending, startTransition] = useTransition()
  const [count, setCount] = useState(0)

  function handleClick() {
    startTransition(() => {
      setCount((c) => c + 1)
    })
  }

  return (
    <div>
      {isPending && <div>spinner...</div>}
      <button onClick={handleClick}>{count}</button>
    </div>
  )
}
```

直观感觉这有点像 `setTimeout`，而防抖节流其实本质也是`setTimeout`，区别是防抖节流是控制了执行频率，让渲染次数减少了，而 v18 的 transition 则没有减少渲染的次数。

### useDeferredValue

`useDeferredValue` 和 `useTransition` 一样，都是标记了一次非紧急更新。`useTransition`是处理一段逻辑，而`useDeferredValue`是产生一个新状态，它是延时状态，这个新的状态则叫 DeferredValue。所以使用`useDeferredValue`可以推迟状态的渲染

> `useDeferredValue` 接受一个值，并返回该值的新副本，该副本将推迟到紧急更新之后。如果当前渲染是一个紧急更新的结果，比如用户输入，React 将返回之前的值，然后在紧急渲染完成后渲染新的值。

```tsx
function Typeahead() {
  const query = useSearchQuery('')
  const deferredQuery = useDeferredValue(query)

  // Memoizing 告诉 React 仅当 deferredQuery 改变，
  // 而不是 query 改变的时候才重新渲染
  const suggestions = useMemo(() => <SearchSuggestions query={deferredQuery} />, [deferredQuery])

  return (
    <>
      <SearchInput query={query} />
      <Suspense fallback="Loading results...">{suggestions}</Suspense>
    </>
  )
}
```

这样一看，`useDeferredValue`直观就是延迟显示状态，那用防抖节流有什么区别呢？

如果使用防抖节流，比如延迟 300ms 显示则意味着所有用户都要延时，在渲染内容较少、用户 CPU 性能较好的情况下也是会延迟 300ms，而且你要根据实际情况来调整延迟的合适值；但是`useDeferredValue`是否延迟取决于计算机的性能。

- 感兴趣可以看下这篇文章：[usedeferredvalue-in-react-18](https://www.bekk.christmas/post/2021/22/usedeferredvalue-in-react-18)
- [在线例子](https://usedeferredvalue-example.netlify.app/)

### useId

`useId`支持同一个组件在客户端和服务端生成相同的唯一的 ID，避免 hydration 的不匹配，原理就是每个 id 代表该组件在组件树中的层级结构。

```tsx
function Checkbox() {
  const id = useId()
  return (
    <>
      <label htmlFor={id}>Do you like React?</label>
      <input id={id} type="checkbox" name="react" />
    </>
  )
}
```

这里涉及到 SSR 部分知识，这里不展开了，可以阅读该篇文章理解：

- [为了生成唯一 id，React18 专门引入了新 Hook：useId](https://juejin.cn/post/7034691251165200398)

### 提供给第三方库的 Hook

这两个 Hook 日常开发基本用不到，简单带过

#### useSyncExternalStore

`useSyncExternalStore` 一般是第三方状态管理库使用如 `Redux`。它通过强制的同步状态更新，使得外部 store 可以支持并发读取。它实现了对外部数据源订阅时不再需要 `useEffect`

```tsx
const state = useSyncExternalStore(subscribe, getSnapshot[, getServerSnapshot]);
```

#### useInsertionEffect

`useInsertionEffect` 仅限于 css-in-js 库使用。它允许 css-in-js 库解决在渲染中注入样式的性能问题。 执行时机在 `useLayoutEffect` 之前，只是此时不能使用 ref 和调度更新，一般用于提前注入样式。

```tsx
useInsertionEffect(() => {
  console.log('useInsertionEffect 执行')
}, [])
```

## 参考

- [React 官方文档](https://zh-hans.reactjs.org/blog/2020/10/20/react-v17.html)
- [React 18 总览](https://juejin.cn/post/7087486984146878494)
- [「React18 新特性」深入浅出用户体验大师—transition](https://juejin.cn/post/7027995169211285512)
