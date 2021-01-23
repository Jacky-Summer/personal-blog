# 深入 React 的 setState 机制

## 前言

本篇写的 setState（涉及源码部分）是针对 React15 版本，即是没有 Fiber 介入的；为了方便看和写，所以选择旧版本，Fiber 写起来有点难，先留着将会写。setState 在 React 15 的原理能理解，16 版本的也是大同小异。

虽然已经用 React Hooks 很久了，React15 的`this.setState()`形式都很少用了，但依然是站在回顾与总结的角度，看待 React 的变迁和发展，所以最近开始重新回顾以前一知半解的一些原理问题，慢慢沉淀技术。

## setState 经典问题

```javascript
setState(updater, [callback])
```

React 通过 `this.setState()` 来更新 state，当使用 `this.setState()`的时候 ，React 会调用 render 方法来重新渲染 UI。

setState 的几种用法就不用我说了，来看看网上讨论 setState 比较多的问题：

### 批量更新

```javascript
import React, { Component } from 'react'

class App extends Component {
  state = {
    count: 1,
  }

  handleClick = () => {
    this.setState({
      count: this.state.count + 1,
    })
    console.log(this.state.count) // 1

    this.setState({
      count: this.state.count + 1,
    })
    console.log(this.state.count) // 1

    this.setState({
      count: this.state.count + 1,
    })
    console.log(this.state.count) // 1
  }
  render() {
    return (
      <>
        <button onClick={this.handleClick}>加1</button>
        <div>{this.state.count}</div>
      </>
    )
  }
}

export default App
```

点击按钮触发事件，打印的都是 1，页面显示 count 的值为 2。

这就是常常说的 setState 批量更新，对同一个值进行多次 setState ， setState 的批量更新策略会对其进行覆盖，取最后一次的执行结果。所以每次 setState 之后立即打印值都是初始值 1，而最后页面显示的值则为最后一次的执行结果，也就是 2。

### setTimeout

```javascript
import React, { Component } from 'react'

class App extends Component {
  state = {
    count: 1,
  }

  handleClick = () => {
    this.setState({
      count: this.state.count + 1,
    })
    console.log(this.state.count) // 1

    this.setState({
      count: this.state.count + 1,
    })
    console.log(this.state.count) // 1

    setTimeout(() => {
      this.setState({
        count: this.state.count + 1,
      })
      console.log(this.state.count) // 3
    })
  }
  render() {
    return (
      <>
        <button onClick={this.handleClick}>加1</button>
        <div>{this.state.count}</div>
      </>
    )
  }
}

export default App
```

点击按钮触发事件，发现 setTimeout 里面的 count 值打印值为 3，页面显示 count 的值为 3。setTimeout 里面 setState 之后能马上能到最新值。

在 setTimeout 里面，setState 是同步的；经过前面两次的 setState 批量更新，count 值已经更新为 2。在 setTimeout 里面的首先拿到新的 count 值 2，再一次 setState，然后能实时拿到 count 的值为 3。

### DOM 原生事件

```javascript
import React, { Component } from 'react'

class App extends Component {
  state = {
    count: 1,
  }

  componentDidMount() {
    document.getElementById('btn').addEventListener('click', this.handleClick)
  }

  handleClick = () => {
    this.setState({
      count: this.state.count + 1,
    })
    console.log(this.state.count) // 2

    this.setState({
      count: this.state.count + 1,
    })
    console.log(this.state.count) // 3

    this.setState({
      count: this.state.count + 1,
    })
    console.log(this.state.count) // 4
  }

  render() {
    return (
      <>
        <button id='btn'>触发原生事件</button>
        <div>{this.state.count}</div>
      </>
    )
  }
}

export default App
```

点击按钮，会发现每次 setState 打印出来的值都是实时拿到的，不会进行批量更新。

在 DOM 原生事件里面，setState 也是同步的。

## setState 同步异步问题

这里讨论的同步和异步并不是指 setState 是否异步执行，使用了什么异步代码，而是指调用 setState 之后 this.state 能否立即更新。

React 中的事件都是合成事件，都是由 React 内部封装好的。React 本身执行的过程和代码都是同步的，只是合成事件和钩子函数的调用顺序在更新之前，导致在合成事件和钩子函数中没法立马拿到更新后的值，就是我们所说的"异步"了。

由上面也可以得知 setState 在原生事件和 setTimeout 中都是同步的。

## setState 源码层面

源码选择的 React 版本为`15.6.2`

### setState 函数

源码里面，setState 函数的代码

React 组件继承自`React.Component`，而 setState 是`React.Component`的方法

```javascript
ReactComponent.prototype.setState = function (partialState, callback) {
  this.updater.enqueueSetState(this, partialState)
  if (callback) {
    this.updater.enqueueCallback(this, callback, 'setState')
  }
}
```

可以看到它直接调用了 `this.updater.enqueueSetState` 这个方法。

### enqueueSetState

```javascript
enqueueSetState: function(publicInstance, partialState) {
  // 拿到对应的组件实例
  var internalInstance = getInternalInstanceReadyForUpdate(
    publicInstance,
    'setState',
  );
  // queue 对应一个组件实例的 state 数组
  var queue = internalInstance._pendingStateQueue || (internalInstance._pendingStateQueue = []);
  queue.push(partialState); // 将 partialState 放入待更新 state 队列
  // 处理当前的组件实例
  enqueueUpdate(internalInstance);
}
```

`_pendingStateQueue`表示待更新队列

`enqueueSetState` 做了两件事：

- 将新的 state 放进组件的状态队列里；
- 用 enqueueUpdate 来处理将要更新的实例对象。

接下来看看 `enqueueUpdate` 做了什么：

```javascript
function enqueueUpdate(component) {
  ensureInjected()
  // isBatchingUpdates 标识着当前是否处于批量更新过程
  if (!batchingStrategy.isBatchingUpdates) {
    // 若当前没有处于批量创建/更新组件的阶段，则立即更新组件
    batchingStrategy.batchedUpdates(enqueueUpdate, component)
    return
  }
  // 需要批量更新，则先把组件塞入 dirtyComponents 队列
  dirtyComponents.push(component)
  if (component._updateBatchNumber == null) {
    component._updateBatchNumber = updateBatchNumber + 1
  }
}
```

`batchingStrategy` 表示批量更新策略，`isBatchingUpdates`表示当前是否处于批量更新过程，默认是 false。

`enqueueUpdate`做的事情：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/35dce2754bb844808bf65fcc12cd1b24~tplv-k3u1fbpfcp-watermark.image)

- 判断组件是否处于批量更新模式，如果是，即`isBatchingUpdates`为 true 时，不进行 state 的更新操作，而是将需要更新的组件添加到`dirtyComponents`数组中；

- 如果不是处于批量更新模式，则对所有队列中的更新执行`batchedUpdates`方法

当中 `batchingStrategy`该对象的`isBatchingUpdates`属性直接决定了是马上要走更新流程，还是应该进入队列等待；所以大概可以得知`batchingStrategy`用于管控批量更新的对象。

来看看它的源码：

```javascript
/**
 *  batchingStrategy源码
 **/
var ReactDefaultBatchingStrategy = {
  isBatchingUpdates: false, // 初始值为 false 表示当前并未进行任何批量更新操作

  // 发起更新动作的方法
  batchedUpdates: function (callback, a, b, c, d, e) {
    var alreadyBatchingUpdates = ReactDefaultBatchingStrategy.isBatchingUpdates

    ReactDefaultBatchingStrategy.isBatchingUpdates = true

    if (alreadyBatchingUpdates) {
      return callback(a, b, c, d, e)
    } else {
      // 启动事务，将 callback 放进事务里执行
      return transaction.perform(callback, null, a, b, c, d, e)
    }
  },
}
```

**每当 React 调用 batchedUpdate 去执行更新动作时，会先把`isBatchingUpdates`置为 true，表明正处于批量更新过程中。**

看完批量更新整体的管理机制，发现还有一个操作是`transaction.perform`，这就引出 React 中的 Transaction（事务）机制。

### Transaction（事务）机制

Transaction 是创建一个黑盒，该黑盒能够封装任何的方法。因此，那些需要在函数运行前、后运行的方法可以通过此方法封装（即使函数运行中有异常抛出，这些固定的方法仍可运行）。

在 React 中源码有关于 Transaction 的注释如下：

```javascript
 * <pre>
 *                       wrappers (injected at creation time)
 *                                      +        +
 *                                      |        |
 *                    +-----------------|--------|--------------+
 *                    |                 v        |              |
 *                    |      +---------------+   |              |
 *                    |   +--|    wrapper1   |---|----+         |
 *                    |   |  +---------------+   v    |         |
 *                    |   |          +-------------+  |         |
 *                    |   |     +----|   wrapper2  |--------+   |
 *                    |   |     |    +-------------+  |     |   |
 *                    |   |     |                     |     |   |
 *                    |   v     v                     v     v   | wrapper
 *                    | +---+ +---+   +---------+   +---+ +---+ | invariants
 * perform(anyMethod) | |   | |   |   |         |   |   | |   | | maintained
 * +----------------->|-|---|-|---|-->|anyMethod|---|---|-|---|-|-------->
 *                    | |   | |   |   |         |   |   | |   | |
 *                    | |   | |   |   |         |   |   | |   | |
 *                    | |   | |   |   |         |   |   | |   | |
 *                    | +---+ +---+   +---------+   +---+ +---+ |
 *                    |  initialize                    close    |
 *                    +-----------------------------------------+
 * </pre>
```

根据以上注释，可以看出：一个 Transaction 就是将需要执行的 method 使用 wrapper（一组 initialize 及 close 方法称为一个 wrapper） 封装起来，再通过 Transaction 提供的 perform 方法执行。

在 perform 之前，先执行所有 wrapper 中的 initialize 方法；perform 完成之后（即 method 执行后）再执行所有的 close 方法，而且 Transaction 支持多个 wrapper 叠加。这就是 React 中的事务机制。

### batchingStrategy 批量更新策略

再看回`batchingStrategy`批量更新策略，ReactDefaultBatchingStrategy 其实就是一个批量更新策略事务，它的 wrapper 有两个：`FLUSH_BATCHED_UPDATES` 和 `RESET_BATCHED_UPDATES`。

`isBatchingUpdates`在 close 方法被复位为 false，如下代码：

```javascript
var RESET_BATCHED_UPDATES = {
  initialize: emptyFunction,
  close: function () {
    ReactDefaultBatchingStrategy.isBatchingUpdates = false
  },
}
//  flushBatchedUpdates 将所有的临时 state 合并并计算出最新的 props 及 state
var FLUSH_BATCHED_UPDATES = {
  initialize: emptyFunction,
  close: ReactUpdates.flushBatchedUpdates.bind(ReactUpdates),
}

var TRANSACTION_WRAPPERS = [FLUSH_BATCHED_UPDATES, RESET_BATCHED_UPDATES]
```

**React 钩子函数**

都说 React 钩子函数也是异步更新，则必须一开始 isBatchingUpdates 为 ture，但默认 isBatchingUpdates 为 false，它是在哪里被设置为 true 的呢？来看下面代码：

```javascript
// ReactMount.js
_renderNewRootComponent: function( nextElement, container, shouldReuseMarkup, context ) {
  // 实例化组件
  var componentInstance = instantiateReactComponent(nextElement);
  // 调用 batchedUpdates 方法
  ReactUpdates.batchedUpdates(
    batchedMountComponentIntoNode,
    componentInstance,
    container,
    shouldReuseMarkup,
    context
  );
}
```

这段代码是在首次渲染组件时会执行的一个方法，可以看到它内部调用了一次 `batchedUpdates` 方法（将 isBatchingUpdates 设为 true），这是因为在组件的渲染过程中，会按照顺序调用各个生命周期(钩子)函数。如果在函数里面调用 setState，则看下列代码：

```javascript
if (!batchingStrategy.isBatchingUpdates) {
  // 立即更新组件
  batchingStrategy.batchedUpdates(enqueueUpdate, component)
  return
}
// 批量更新，则先把组件塞入 dirtyComponents 队列
dirtyComponents.push(component)
```

则所有的更新都能够进入 dirtyComponents 里去，即 setState 走的异步更新

**React 合成事件**

当我们在组件上绑定了事件之后，事件中也有可能会触发 setState。为了确保每一次 setState 都有效，React 同样会在此处手动开启批量更新。看下面代码：

```javascript
// ReactEventListener.js

dispatchEvent: function (topLevelType, nativeEvent) {
  try {
    // 处理事件：batchedUpdates会将 isBatchingUpdates设为true
    ReactUpdates.batchedUpdates(handleTopLevelImpl, bookKeeping);
  } finally {
    TopLevelCallbackBookKeeping.release(bookKeeping);
  }
}
```

`isBatchingUpdates` 这个变量，在 React 的生命周期函数以及合成事件执行前，已经被 React 改为 true，这时我们所做的 setState 操作自然不会立即生效。当函数执行完毕后，事务的 close 方法会再把 isBatchingUpdates 改为 false。

就像最上面的例子，整个过程模拟大概是：

```javascript
handleClick = () => {
  // isBatchingUpdates = true
  this.setState({
    count: this.state.count + 1,
  })
  console.log(this.state.count) // 1

  this.setState({
    count: this.state.count + 1,
  })
  console.log(this.state.count) // 1

  this.setState({
    count: this.state.count + 1,
  })
  console.log(this.state.count) // 1
  // isBatchingUpdates = false
}
```

而如果有 setTimeout 介入后

```javascript
handleClick = () => {
  // isBatchingUpdates = true
  this.setState({
    count: this.state.count + 1,
  })
  console.log(this.state.count) // 1

  this.setState({
    count: this.state.count + 1,
  })
  console.log(this.state.count) // 1

  setTimeout(() => {
    // setTimeout异步执行，此时 isBatchingUpdates 已经被重置为 false
    this.setState({
      count: this.state.count + 1,
    })
    console.log(this.state.count) // 3
  })
  // isBatchingUpdates = false
}
```

`isBatchingUpdates`是在同步代码中变化的，而 setTimeout 的逻辑是异步执行的。当 this.setState 调用真正发生的时候，isBatchingUpdates 早已经被重置为 false，这就使得 setTimeout 里面的 setState 具备了立刻发起同步更新的能力。

### batchedUpdates 方法

看到这里大概就可以了解 setState 的同步异步机制了，接下来让我们进一步体会，可以把 React 的`batchedUpdates`拿来试试，在该版本中此方法名称被置为`unstable_batchedUpdates `即不稳定的方法。

```
import React, { Component } from 'react'
import { unstable_batchedUpdates as batchedUpdates } from 'react-dom'

class App extends Component {
  state = {
    count: 1,
  }

  handleClick = () => {
    this.setState({
      count: this.state.count + 1,
    })
    console.log(this.state.count) // 1

    this.setState({
      count: this.state.count + 1,
    })
    console.log(this.state.count) // 1

    setTimeout(() => {
      batchedUpdates(() => {
        this.setState({
          count: this.state.count + 1,
        })
        console.log(this.state.count) // 2
      })
    })
  }
  render() {
    return (
      <>
        <button onClick={this.handleClick}>加1</button>
        <div>{this.state.count}</div>
      </>
    )
  }
}

export default App
```

如果调用`batchedUpdates`方法，则 `isBatchingUpdates`变量会被设置为 true，由上述得为 true 走的是批量更新策略，则 setTimeout 里面的方法也变成异步更新了，所以最终打印值为 2，与本文第一道题结果一样。

## 总结

setState 同步异步的表现会因调用场景的不同而不同：在 React 钩子函数及合成事件中，它表现为异步；而在 setTimeout/setInterval 函数，DOM 原生事件中，它都表现为同步。这是由 React 事务机制和批量更新机制的工作方式来决定的。

在 React16 中，由于引入了 Fiber 机制，源码多少有点不同，但大同小异，之后我也会写 React16 原理的文章，敬请关注！

<br>

我近期会维护的开源项目：

- [基于 React + TypeScript + Dumi + Jest + Enzyme 开发 UI 组件库](https://github.com/Jacky-Summer/monki-ui)
- [Next.js 企业级项目脚手架模板](https://github.com/Jacky-Summer/nextjs-ts-antd-redux-storybook-starter)
- [个人技术博文 Github 仓库](https://github.com/Jacky-Summer/personal-blog)
  觉得不错的话欢迎 star，给我一点鼓励继续写作吧~
