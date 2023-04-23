# 深入 React Context 源码与实现原理

## 前置知识

本文假设你对 context 基础用法和 React fiber 渲染流程有一定的了解，因为这些知识不会介绍详细。本文基于 [React v18.2.0](https://github.com/facebook/react/tree/v18.2.0)

### Context API

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/160b495eb20b4cf8b5e40868f5d83644~tplv-k3u1fbpfcp-watermark.image?)

### React 渲染流程

**React 渲染分为 `render` 阶段和 `commit` 阶段，其中 `render` 阶段分为两步（深度优先遍历）**：

1. `beginWork`（进入节点的过程向下遍历，协调子元素）
2. `completeUnitOfWork`（离开节点的过程向上回溯）

**区别 render 和 beginWork**

为了避免与上面的阶段混淆，以下 `render` 都代指开发者层面的 `render`，即指类组件执行 `render` 方法或函数组件执行

- 如果一个组件发生更新，当前组件到 fiber root 上的父级链上的所有 fiber，都会执行 `beginWork`，但执行 `beginWork`，不代表触发了组件的 `render`（fiber 会检查组件是否需要进行渲染，不需要则会跳过复用旧的 fiber 节点）所以 `render` 不等于 `beginWork`
- 如果组件 `render` 执行了，则一定经历了 `beginWork` 流程，触发了 `beginWork`

综上 `beginWork` 的工作是进入节点时协调子元素，如果 fiber 类型是类组件或者函数组件，则需检测比较组件是否需要执行 `render`，不需要则会跳过复用旧的 fiber 节点

## React.createContext 原理

```js
const MyContext = React.createContext(defaultValue)
```

> 创建一个 Context 对象。只有当组件所处的树中没有匹配到 Provider 时，其 defaultValue 参数才会生效

源码位置：[packages/react/src/ReactContext.js](https://github.com/facebook/react/blob/v18.2.0/packages/react/src/ReactContext.js#L14)

**createContext 函数的核心逻辑**是返回一个 context 对象，其中包括三个重要属性：

- `Provider` 和 `Consumer` 两个组件（React Element 对象）属性
- `_currentValue` ：保存 context 的值，用来保存传递给 Provider 的 value 属性）

下列是精简去除类型定义和引入的源码，后面源码举例都这么处理，为了方便直观的看：

```ts
const REACT_PROVIDER_TYPE = Symbol.for('react.provider')
const REACT_CONTEXT_TYPE = Symbol.for('react.context')

export function createContext(defaultValue) {
  const context = {
    $$typeof: REACT_CONTEXT_TYPE, // 本质就是 Consumer Element 类型
    _currentValue: defaultValue, // 保存 context 的值
    _currentValue2: defaultValue, // 为了支持多个并发渲染器，适配不同的平台
    _threadCount: 0, // 跟踪当前有多少个并发渲染器
    Provider: null,
    Consumer: null,
  }
  // 添加 Provider 属性，本质就是 Provider Element 类型
  context.Provider = {
    $$typeof: REACT_PROVIDER_TYPE,
    _context: context,
  }
  // 添加 Consumer 属性
  context.Consumer = context

  return context
}
```

> JSX 语法在进入 render 时会被编译成 React Element 对象

## Context.Provider 原理

```js
<MyContext.Provider value={/* 某个值 */}>
```

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1084fe2251a84b1d810dbf6f88faa427~tplv-k3u1fbpfcp-watermark.image?)

先来了解 Provider 的特性：

- 每个 Context 对象都会返回一个 Provider React 组件，它允许消费组件订阅 context 的变化
- Provider 接收一个  value  属性，传递给消费组件。一个 Provider 可以和多个消费组件有对应关系。
- 只有当组件所处的树中没有匹配到 Provider 时，其 defaultValue 参数才会生效
- 多个相同的 Provider 也可以嵌套使用，里层的会覆盖外层的数据。
- 当 Provider 的 value 值发生变化时，它内部的所有消费组件都会重新渲染，可跳过 `shouldComponentUpdate` 强制更新

如果一个组件发生更新，那么当前组件到 fiber root 上的父级链上的所有 fiber，更新优先级都会升高，都会触发 `beginWork`，但不一定会 render

当初次 Fiber 树渲染，进入 `beginWork` 方法，其中对应的节点处理函数是 `updateContextProvider`：

```js
function beginWork(current, workInProgress, renderLanes) {
  switch (workInProgress.tag) {
    case ContextProvider:
      return updateContextProvider(current, workInProgress, renderLanes)
  }
}
```

进入 [updateContextProvider](https://github.com/facebook/react/blob/v18.2.0/packages/react-reconciler/src/ReactFiberBeginWork.new.js#L3685) 方法：

```js
function updateContextProvider(current, workInProgress, renderLanes) {
  const providerType = workInProgress.type
  const context = providerType._context

  const newProps = workInProgress.pendingProps
  const oldProps = workInProgress.memoizedProps
  // 新的 value 值
  const newValue = newProps.value
  // 获取 Provider 上的 value
  pushProvider(workInProgress, context, newValue)

  // 更新阶段
  if (oldProps !== null) {
    const oldValue = oldProps.value
    // 使用 Object.is 来比较新旧值是否发生变化
    if (is(oldValue, newValue)) {
      // context 值没有变更，则提前退出
      if (oldProps.children === newProps.children && !hasLegacyContextChanged()) {
        return bailoutOnAlreadyFinishedWork(current, workInProgress, renderLanes)
      }
    } else {
      // context 值发生改变，深度优先遍历查找 consumer 消费组件，标记更新
      propagateContextChange(workInProgress, context, renderLanes)
    }
  }

  // 继续向下调和子代 fiber
  const newChildren = newProps.children
  reconcileChildren(current, workInProgress, newChildren, renderLanes)
  return workInProgress.child
}

// 使用栈存储 context._currentValue 值，设置 context._currentValue 为最新值
function pushProvider(providerFiber, context, nextValue) {
  // 压栈
  push(valueCursor, context._currentValue, providerFiber)
  // 修改 context 的值
  context._currentValue = nextValue
}
```

- 首次执行时，保存 `workInProgress.pendingProps.value` 值作为最新值，然后调用 `pushProvider` 方法设置`context._currentValue` 值
- `pushProvider`：存储 context 值的函数，利用栈先进后出的特性，先把 `context._currentValue` 压栈；与后面流程的 `popProvider`（出栈）函数相对应。
- 更新阶段时通过浅比较（`Object.is`）来判断新旧 context 值是否发生改变，没发生改变则调用 `bailoutOnAlreadyFinishedWork` 进入 bailout，复用当前 Fiber 节点，改变则调用`propagateContextChange`方法

我们总结下 `Context.Provider` 的 Fiber 更新方法 —— **`updateContextProvider`的核心逻辑**：

1. 将 Provider 的 value 属性赋值给 `context._currentValue`（压栈）
2. 通过 `Object.is` 浅比较 context 新旧值是否发生变化
3. 发生变化时，调用 `propagateContextChange` 走更新的流程，深度优先遍历查找消费组件来标记更新

> `propagateContextChange` 逻辑：深度优先遍历所有的子代 fiber ，然后找到里面具有 `dependencies` 的属性，对比 `dependencies` 中的 context 和当前 Provider 的 context 是否是同一个，如果是同一个，会提高 fiber 的更新优先级，让 fiber 在接下来的调和过程中，处于一个高优先级待更新的状态，而高优先级的 fiber 都会 `beginWork`

## 消费 Context 原理

由上文知识我们简略粗暴的说：Provider 一顿操作核心就是修改 `context._currentValue` 的值，那么消费 Context 值的原理也就是想方设法读取 `context._currentValue` 的值了。

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/450b287995934525940b34041f1fac05~tplv-k3u1fbpfcp-watermark.image?)

### Context.Consumer（函数组件）

```ts
<MyContext.Consumer>
  {value => /* 基于 context 值进行渲染*/}
</MyContext.Consumer>
```

> 一个 React 组件可以订阅 context 的变更，此组件可以让你在函数式组件中可以订阅 context。这种方法需要一个函数作为子元素（function as a child）。这个函数接收当前的 context 值，并返回一个 React 节点。传递给函数的 value 值等价于组件树上方离这个 context 最近的 Provider 提供的 value 值

当 context 值更新时，Fiber 树渲染时，进入 `beginWork` 方法，`beginWork` 中对于 `ContextConsumer` 的节点处理函数是 [updateContextConsumer](https://github.com/facebook/react/blob/v18.2.0/packages/react-reconciler/src/ReactFiberBeginWork.new.js#L3251)：

```js
function beginWork(current, workInProgress, renderLanes) {
  switch (workInProgress.tag) {
    case ContextConsumer:
      return updateContextConsumer(current, workInProgress, renderLanes)
  }
}
```

`updateContextConsumer`的核心逻辑：

1. 调用 `prepareToReadContext` 和 `readContext` 读取最新的 context 值。
2. 通过 render props 函数，传入最新的 context value 值，得到最新的 children 。
3. 调和 children

```js
function updateContextConsumer(current, workInProgress, renderLanes) {
  // 拿到 context
  let context = workInProgress.type
  context = context._context

  const newProps = workInProgress.pendingProps
  // 获取 Consumer 组件的 render props children
  const render = newProps.children
  // 读取 context 前的准备工作
  prepareToReadContext(workInProgress, renderLanes)
  // 读取最新 context._currentValue 值
  const newValue = readContext(context)

  let newChildren
  // 最新的 children element
  newChildren = render(newValue)

  // 进入主流程，调和 children
  reconcileChildren(current, workInProgress, newChildren, renderLanes)
  return workInProgress.child
}
```

### useContext（函数组件）

```ts
const value = useContext(MyContext)
```

> 接收一个 context 对象（React.createContext 的返回值）并返回该 context 的当前值。

看如下代码，useContext Hook 挂载阶段和更新阶段，本质都是调用 `readContext` 函数，`readContext` 函数会返回 `context._currentValue`。而且也是调用了 `prepareToReadContext` 和 `readContext`

```js
function beginWork(current, workInProgress, renderLanes) {
  switch (workInProgress.tag) {
    case FunctionComponent:
      return updateFunctionComponent(current, workInProgress, Component, resolvedProps, renderLanes)
  }
}

function updateFunctionComponent(current, workInProgress, Component, nextProps, renderLanes) {
  prepareToReadContext(workInProgress, renderLanes)
  // 处理各种hooks逻辑
  nextChildren = renderWithHooks(current, workInProgress, Component, nextProps, context, renderLanes)
  // ...
}
```

[renderWithHooks](https://github.com/facebook/react/blob/v18.2.0/packages/react-reconciler/src/ReactFiberHooks.new.js#L374) 函数是调用函数组件的主要函数

```js
function renderWithHooks(current, workInProgress, Component, nextProps, context, renderLanes) {
  // ...
  ReactCurrentDispatcher.current =
    current === null || current.memoizedState === null
      ? HooksDispatcherOnMount // 挂载阶段
      : HooksDispatcherOnUpdate // 更新阶段
}

// 确保 Hooks 只能在函数组件内部或自定义 Hooks 中使用，提供正确的调度程序
function resolveDispatcher() {
  const dispatcher = ReactCurrentDispatcher.current
  return dispatcher
}

function useContext(Context) {
  const dispatcher = resolveDispatcher()
  return dispatcher.useContext(Context)
}

const HooksDispatcherOnMount = {
  useContext: readContext,
  // ...
}
const HooksDispatcherOnUpdate = {
  useContext: readContext,
  // ...
}
```

### Class.contextType（类组件）

```ts
class MyClass extends React.Component {
  componentDidMount() {
    let value = this.context
    /* 在组件挂载完成后，使用 MyContext 组件的值来执行一些有副作用的操作 */
  }
  componentDidUpdate() {
    let value = this.context
    /* ... */
  }
  componentWillUnmount() {
    let value = this.context
    /* ... */
  }
  render() {
    let value = this.context
    /* 基于 MyContext 组件的值进行渲染 */
  }
}
MyClass.contextType = MyContext
```

> 挂载在 class 上的 contextType 属性可以赋值为由 React.createContext() 创建的 Context 对象。此属性可以让你使用 this.context 来获取最近 Context 上的值。你可以在任何生命周期中访问到它，包括 render 函数中。

- 类组件会判断类组件上是否有静态属性 `contextType`
- 如果有则调用 `readContext` 方法，并赋值给类实例的 context 属性，所以我们才可以使用 this.context 获取 context 值

```js
function beginWork(current, workInProgress, renderLanes) {
  switch (workInProgress.tag) {
    case ClassComponent:
      return updateClassComponent(current, workInProgress, Component, resolvedProps, renderLanes)
  }
}

function updateClassComponent(current, workInProgress, Component, nextProps, renderLanes) {
  // ...
  prepareToReadContext(workInProgress, renderLanes)
  mountClassInstance(workInProgress, Component, nextProps, renderLanes)
  // ...
}

function mountClassInstance(workInProgress, ctor, newProps, renderLanes) {
  // ...
  const instance = workInProgress.stateNode
  // 判断类组件上是否有静态属性 contextType
  const contextType = ctor.contextType
  // 有则调用 readContext
  if (typeof contextType === 'object' && contextType !== null) {
    // 赋值给类实例的 context 属性
    instance.context = readContext(contextType)
  }
}
```

综上，以上三种方式只是 React 根据不同使用场景封装的 API，它们在消费/订阅 context 的共同操作：

1. 先调用 `prepareToReadContext` 进行准备工作
2. 再调用 `readContext` 方法读取 context 值（`readContext` 方法返回 `context._currentValue` 最新值）

上文提到 `propagateContextChange` ，如果组件订阅了 context，不管是函数组件还是类组件，都会将 `fiber.lanes` 设置为 `renderLanes`。在 `beginWork` 阶段，发现 `fiber.lanes` 等于 `renderLanes`，则走 `beginWork` 的逻辑，强制组件更新

### prepareToReadContext 和 readContext 逻辑

[prepareToReadContext](https://github.com/facebook/react/blob/v18.2.0/packages/react-reconciler/src/ReactFiberNewContext.new.js#L625) 的核心逻辑：

- 设置全局变量 `currentlyRenderingFiber` 为当前工作的 fiber，并重置`lastContextDependency` 等全局变量

```js
function prepareToReadContext(workInProgress, renderLanes) {
  // 设置全局变量 currentlyRenderingFiber 为当前工作的 fiber, 为 readContext 做准备
  currentlyRenderingFiber = workInProgress
  // 用于构造 dependencies 列表
  lastContextDependency = null
  // 将全局变量 lastFullyObservedContext (保存的是 context 对象) 重置为 null
  lastFullyObservedContext = null

  const dependencies = workInProgress.dependencies
  if (dependencies !== null) {
    const firstContext = dependencies.firstContext
    if (firstContext !== null) {
      if (includesSomeLane(dependencies.lanes, renderLanes)) {
        // Context list has a pending update. Mark that this fiber performed work.
        markWorkInProgressReceivedUpdate()
      }
      // 重置 fiber context 依赖
      dependencies.firstContext = null
    }
  }
}
```

[readContext](https://github.com/facebook/react/blob/v18.2.0/packages/react-reconciler/src/ReactFiberNewContext.new.js#L652) 的核心逻辑：

- 收集组件依赖的所有不同的 context，如果组件订阅了 context，则将 context 添加到 `fiber.dependencies` 链表中
- 返回`context._currentValue`, 并构造一个`contextItem`添加到`workInProgress.dependencies` 链表之后。

```js
function readContext(context) {
  return readContextForConsumer(currentlyRenderingFiber, context)
}

function readContextForConsumer(consumer, context) {
  // ReactDOM 中 isPrimaryRenderer 为 true，则一直返回 context._currentValue
  const value = isPrimaryRenderer ? context._currentValue : context._currentValue2

  // 相等说明是同一个 Context，不处理为了防止重复添加依赖
  if (lastFullyObservedContext === context) {
    // Nothing to do. We already observe everything in this context.
  } else {
    const contextItem = {
      context: context,
      memoizedValue: value,
      next: null,
    }
    // 构造一个 contextItem, 加入到 workInProgress.dependencies 链表之后
    if (lastContextDependency === null) {
      lastContextDependency = contextItem
      // dependencies 属性用于判定是否依赖了 ContextProvider 中的值
      consumer.dependencies = {
        lanes: NoLanes,
        firstContext: contextItem,
      }
    } else {
      // 将 context 添加到 fiber.dependencies 链表末尾
      lastContextDependency = lastContextDependency.next = contextItem
    }
  }
  // 返回 context._currentValue
  return value
}
```

## Context 原理八连问

上面源码实际上还是讲解不够完整的，在这推荐一篇文章：[【React 源码系列】React Context 原理，如何合理设计共享状态](https://juejin.cn/post/7132840516525260831)，个人认为相对讲得很清晰了。

想知道自己对原理的理解，除了输出就是回答解决一些提问了，这里列举了一些原理相关的问题，写下简略的解答，看看自己是否了解。

### Provider 如何传递 context？

通过将 Provider 的 value 属性值赋值给 `context._currentValue`

### 没有 Provider 包裹，为什么读不到最新的 context 值？

```js
render() {
  return (
    <>
      <TestContext.Provider value={10}>
       {/* 可读到 context 值最新值 10 */}
        <Test />
      </TestContext.Provider>
      {/* 只能读到 context 初始值（createContext 函数的参数 defaultValue） */}
      <Test />,
    </>
  )
}
```

消费 context 时是读取 `context._currentValue` 值，理论上其它组件也是读取该最新值的。Provider 其中一个特性是只有当组件所处的树中没有匹配到 Provider 时，其 `defaultValue` 参数才会生效。所以没有被 Provider 包裹的组件，是只能读到默认值的。

React 在**深度优先遍历** fiber 树时，最外层 Provider 开始 `beginWork`，会先将 `context._currentValue` 的旧值保存起来，赋新的值给 `context._currentValue`（所以在里层的组件都能读到最新值），在离开 Provider 节点时会调用 `completeUnitOfWork` 完成工作，在此会将 `context._currentValue` 恢复成旧值，到遍历第二个 `<Test />` 节点时就读的是 context 的默认值（不被 Provider 包裹的组件 render 时 `beginWork` 的时候就读到旧值了）。

### 相同 Provider 嵌套使用，里层的会覆盖外层的数据是怎么实现的？

```js
render() {
  return (
    <>
      <TestContext.Provider value={10}>
        <Test1 />
        <TestContext.Provider value={100}>
          <Test2 />
        </TestContext.Provider>
      </TestContext.Provider>
    </>
  )
}
```

在这场景下， `<Test1 />` 和 `<Test2 />` 组件读取的值分别是 10 和 100。

为了实现嵌套的机制，React 利用的是**栈**的特性（后入先出），通过 `pushProvider` 和 `popProvider`。

Fiber 深度优先遍历时：

- 最外层 Provider 将 value 值 10 压入栈 `pushProvider`，此时栈顶是 10
- 遍历里层 Provider 时将 value 值 100 压入栈 `pushProvider`，此时栈顶是 100，即`context._currentValue` 的值为 100

消费组件 `<Test2 />`读取时，在其所在 Provider 范围内先读取栈顶的值，所以读取的是 100；里层的 Provider 完成遍历工作离开时，弹出栈顶 `popProvider`的值 100，此时栈顶的值是 10， 即 `context._currentValue` 的值为 10，`<Test1 />` 里面读到的值也就为 10 了。

由于 React 调和过程就是 Fiber 树深度优先遍历的过程, 向下遍历（beginWork）和向上回溯（completeWork）恰好符合栈的特性（入栈和出栈），Context 的嵌套读取就是利用了这个特性。

### 三种消费 context 的原理

- useContext：本质上调用 `readContext` 方法
- Context.Consumer：本质上是类型为 `REACT_CONTEXT_TYPE` 的 React Element 对象，context 本身就存在 Consumer 里面，本质也是调用 `readContext`
- Class.contextType：通过静态属性 contextType 建立联系 ，在类组件实例化的时候被使用，本质上也是调用 `readContext`

三种方式只是 React 根据不同使用场景封装的 API，本质都是调用了 `readContext` 方法读取 `context._currentValue` 值

### context 的存取发生在 React 渲染的哪些阶段

context 的存取就是发生在 `beginWork` 阶段，在 `beginWork` 阶段，如果当前组件订阅了 context，则从 context 中读取 `_currentValue` 值

### 消费 context 的组件，context 改变为什么会订阅更新？

- 当 Provider 的 context value 值更新时，会调用 `updateContextProvider` 方法，里面的 `propagateContextChange` 方法会对 fiber 子树向下深度优先遍历所有的 fiber 节点，目的是为了找到消费组件标记更新。如果 `fiber.dependencies` 中存在一个 context 和当前 Provider 的 context 相等，那说明这个组件订阅了当前的 Provider 的 context，就会被标记更新。
- 而消费组件调用的 `readContext` 方法则会把 `fiber.dependencies` 和 context 对象建立关联，`fiber.dependencies` 用于判断是否依赖了 `ContextProvider` 中的值
- context 值更新时消费 context 的 fiber 和父级链都会提高更新优先级，向上遍历时，会设置消费节点的父路径上所有节点的 `fiber.childLanes` 属性,（`childLanes` 属性用于判断子节点是否需要更新）需要更新则子节点就会进入更新逻辑（开始 `beginWork`）。

### 消费 context 的组件是如何跳过 PureComponent、shouldComponentUpdate 强制 render？

- 类组件更新流程中，强制更新会跳过 `PureComponent` 和 `shouldComponentUpdate` 等优化策略，在外部代码层面，我们可调用 `this.forceUpdate()`，就会给类组件打上强制更新的 tag。而在内部实现上， context 的 value 改变时，要想订阅 context 的类组件更新，相应的也得打上强制更新的 tag
- 当 context 值发生变化时，会调用 `propagateContextChange` 对 Fiber 子树向下深度优先遍历所有的 fiber 节点，如果 `fiber.dependencies` 中存在一个 context 和当前 Provider 的 context 相等，那说明这个组件订阅了当前的 Provider 的 context，如果 fiber 节点是类组件， 则会创建一个 update 对象，并将 `update.tag` 标记为 `ForceUpdate`；而处理 update 时，发现 tag 为 `ForceUpdate` 的话，会将全局变量 `hasForceUpdate` 设置为 true， 这决定了类组件会强制更新。

> 在 `updateClassComponent` 中会调用 `updateClassInstance` 判断类组件是否应该更新。在 `updateClassInstance` 中会判断全局变量 `hasForceUpdate` 或者组件的 `shouldComponentUpdate` 的返回值是否为 true， true 则表示要强制更新。

### 简述 Context 原理

Context 的实现原理：

- 创建 Context：`createContext` 返回一个 context 对象，对象包括 `Provider` 和 `Consumer` 两个组件属性，并创建 `_currentValue` 属性用来保存 context 的值
- Provider 负责传递 context 值，并使用栈的特性存储修改 context 值
- 消费 Context：消费组件节点调用 `readContext` 读取 `context._currentValue` 获取最新值
- Provider 更新 Context：`ContextProvider` 节点深度优先遍历子代 fiber，消费 context 的 fiber 和父级链都会提升更新优先级；对于类组件的 fiber ，会被 `forceUpdate` 处理。接下来所有消费的 fiber，都会执行 `beginWork`

## 结语

本文对 Context 源码的理解有限，暂未能完全读完，只是过了一遍大致实现，如有错误恳请纠正。

## 参考文章

- [React context 基本原理](https://juejin.cn/post/6844904175294251022)
- [React 源码 - React Context 原理](https://juejin.cn/post/6997735545832996878)
- [【React 源码系列】React Context 原理，如何合理设计共享状态](https://juejin.cn/post/7132840516525260831)
- [React 进阶实践指南 - Context 原理](https://juejin.cn/book/6945998773818490884?utm_source=course_list)
