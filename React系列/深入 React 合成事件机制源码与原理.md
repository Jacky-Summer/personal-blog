# 深入 React 合成事件机制源码与原理

## 前言

本文源码基于 React `v17.0.2`，React 事件的基础知识就不回顾了，主要从 React 原理切入，从事件注册到事件执行的整个链路。

## 合成事件是什么

React 基于浏览器事件机制实现了一套自身的机制，即浏览器原生事件的跨浏览器包装器。

## 为什么需要合成事件

1. 兼容所有浏览器，更好的跨平台
2. 方便 React 统一进行事件管理，更好地控制事件的执行链路

## React 17 事件特性

v17 与 v16 相比：

1. **v16 React 将事件委托到 document 上；v17 则委托到根节点**。
2. 去除事件池
3. onScroll 事件不再冒泡，以防止出现常见的混淆
4. React 的 onFocus 和 onBlur 事件已在底层切换为原生的 focusin 和 focusout 事件。它们更接近 React 现有行为，有时还会提供额外的信息。
5. 捕获事件（例如 onClickCapture）现在使用的是实际浏览器中的捕获监听器。

## 事件注册

事件注册是在顶层自执行的，在 React 自身引入文件的时候调用的。

注册事件（React 将同种类型的事件放在一个插件中）：

```js
import * as BeforeInputEventPlugin from './plugins/BeforeInputEventPlugin'
import * as ChangeEventPlugin from './plugins/ChangeEventPlugin'
import * as EnterLeaveEventPlugin from './plugins/EnterLeaveEventPlugin'
import * as SelectEventPlugin from './plugins/SelectEventPlugin'
import * as SimpleEventPlugin from './plugins/SimpleEventPlugin'

// 原生 DOM 事件名称与 React 事件名称映射关系。其中一个作用就是给 allNativeEvents 注入所有原生事件名，下文会用到
SimpleEventPlugin.registerEvents()
EnterLeaveEventPlugin.registerEvents()
ChangeEventPlugin.registerEvents()
SelectEventPlugin.registerEvents()
BeforeInputEventPlugin.registerEvents()
```

`registerEvents` 用于初始化原生事件（其中一个作用是为 `allNativeEvents`集合注入原生事件名）

### 事件插件

- SimpleEventPlugin：处理常见的 DOM 事件
- EnterLeaveEventPlugin：处理鼠标进入离开时事件
- ChangeEventPlugin：处理表单元素上的 onChange 事件
- SelectEventPlugin：负责处理表单元素上的 onSelect 事件
- BeforeInputEventPlugin：用于处理 input、textarea 或者 contentEditable 元素上的 onBeforeInput 事件

## 事件绑定

### 源码细节

在 React 初始化渲染的时候，`ReactDOM.render` 会调用函数 `listenToAllSupportedEvents` 来绑定事件

```js
function createRootImpl(container, tag, options) {
  // 在根容器上监听支持的事件
  const rootContainerElement = container.nodeType === COMMENT_NODE ? container.parentNode : container
  listenToAllSupportedEvents(rootContainerElement)
}
```

代码位置：`packages/react-dom/src/events/DOMPluginEventSystem.js`，只列出 `listenToAllSupportedEvents` 的核心代码：

```js
function listenToAllSupportedEvents(rootContainerElement) {
  if (!rootContainerElement[listeningMarker]) {
    // allNativeEvents 是一个 Set 集合，保存所有浏览器原生事件名
    allNativeEvents.forEach((domEventName) => {
      // 判断是否支持冒泡的事件，不支持的话无需事件委托到根节点
      if (!nonDelegatedEvents.has(domEventName)) {
        // 冒泡阶段绑定事件
        listenToNativeEvent(domEventName, false, rootContainerElement, null)
      }
      // 捕获阶段绑定事件
      listenToNativeEvent(domEventName, true, rootContainerElement, null)
    })
  }
}
```

`listenToAllSupportedEvents` 的核心逻辑：

1. 通过 `listenToNativeEvent` 来绑定浏览器事件，且绑定在 `rootContainerElement` 根节点。
2. 如果是支持冒泡的事件，则捕获阶段和冒泡阶段都绑定事件；不支持冒泡的事件，则只绑定捕获阶段的事件。

- `allNativeEvents`：是一个 Set 集合，保存了 80 个浏览器原生 DOM 事件

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a005d155a980484c9fc91c0639158a32~tplv-k3u1fbpfcp-watermark.image?)

- `nonDelegatedEvents`：Set 集合，保存浏览器原生事件中不会冒泡的事件，如 load，scroll，媒体事件 canplay，play 等等

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bb094206312349ab9a143972386fecb6~tplv-k3u1fbpfcp-watermark.image?)

接下来看看 `listenToNativeEvent` 的实现：

1. 创建事件委托的回调函数
2. 根据事件是捕获阶段还是冒泡阶段，调用不同的事件绑定函数

```js
function listenToNativeEvent(
  domEventName,
  isCapturePhaseListener,
  rootContainerElement,
  targetElement,
  eventSystemFlags
) {
  // 绑定事件
  addTrappedEventListener(target, domEventName, eventSystemFlags, isCapturePhaseListener)
}

function addTrappedEventListener(
  targetContainer,
  domEventName,
  eventSystemFlags,
  isCapturePhaseListener,
  isDeferredListenerForLegacyFBSupport
) {
  // 创建事件委托的回调函数（其实是事件派发器）
  let listener = createEventListenerWrapperWithPriority(targetContainer, domEventName, eventSystemFlags)
  // 根据事件是捕获阶段还是冒泡阶段，调用不同的绑定函数
  if (isCapturePhaseListener) {
    unsubscribeListener = addEventCaptureListener(targetContainer, domEventName, listener)
  } else {
    unsubscribeListener = addEventBubbleListener(targetContainer, domEventName, listener)
  }
}

// 表示在冒泡阶段触发事件处理函数
function addEventBubbleListener(target, eventType, listener) {
  // 第三个参数为 false（冒泡阶段）
  target.addEventListener(eventType, listener, false) // 需要注意这里的 listener 是事件派发器，并不是我们自己使用时写的事件回调
  return listener
}

// 表示在捕获阶段触发事件处理函数
function addEventCaptureListener(target, eventType, listener) {
  // 第三个参数为 true（捕获阶段）
  target.addEventListener(eventType, listener, true)
  return listener
}
```

再来看看上面的函数 `createEventListenerWrapperWithPriority` 的实现：

```js
export function createEventListenerWrapperWithPriority(targetContainer, domEventName, eventSystemFlags) {
  // 根据事件名获取事件的优先级
  const eventPriority = getEventPriorityForPluginSystem(domEventName)

  let listenerWrapper
  // 根据事件优先级返回对应的事件监听函数
  switch (eventPriority) {
    // 离散事件
    case DiscreteEvent: // 优先级最高
      listenerWrapper = dispatchDiscreteEvent
      break
    // 用户交互阻塞渲染的事件
    case UserBlockingEvent: // 优先级适中
      listenerWrapper = dispatchUserBlockingUpdate
      break
    // 其它事件
    case ContinuousEvent: // 优先级最低
    default:
      listenerWrapper = dispatchEvent
      break
  }
  // 返回事件回调函数 listener
  return listenerWrapper.bind(null, domEventName, eventSystemFlags, targetContainer)
}
```

由上述代码可以看出，不同的 DOM 事件调用 `getEventPriorityForPluginSystem` 会返回不同的优先级，优先级包括：

1. `DiscreteEvent`：离散事件。如 click、keydown、focusin 等，这些事件的触发不是连续的，可以快速响应，优先级最高
2. `UserBlockingEvent`：用户交互阻塞渲染的事件。如 drag、scroll 等，优先级适中
3. `ContinuousEvent` 与 default：连续事件和默认事件。连续事件如 playing、load 等，优先级最低

而前两个对应的 `dispatchDiscreteEvent` 和 `dispatchUserBlockingUpdate` 其实都是对 `dispatchEvent` 的封装，所以下文我们重点看 `dispatchEvent` 函数就行了。

**listenToNativeEvent**

看完上面的代码，让我们来抽离出最核心的代码，把函数调用代码去掉抽离整合如下：

```js
function listenToNativeEvent(domEventName, isCapturePhaseListener, target) {
  const listener = dispatchEvent.bind(null, domEventName, eventSystemFlags, targetContainer)
  if (isCapturePhaseListener) {
    target.addEventListener(eventType, listener, true)
  } else {
    target.addEventListener(eventType, listener, false)
  }
}
```

### 核心流程

1. React 初始化时，会在根节点上绑定原生事件
2. 支持冒泡的事件，React 会同时绑定捕获阶段和冒泡阶段的事件；不支持冒泡的事件，React 则只绑定捕获阶段的事件
3. React 将事件分为三种优先级类型，在绑定事件处理函数时会使用不同的回调函数，但底层都是调用 dispatchEvent 函数

我们也可以知道，事件在根节点中代理后是一直在触发的，只是没有绑定对应的回调函数。

> 问：React 事件都是在顶层进行代理派发执行的，对不支持冒泡的事件，React 如何触发

在根节点 React 只绑定了不支持冒泡事件的捕获阶段，而实际上 React 会对不支持冒泡的事件（除了 scroll）进行特殊处理，这个过程发生在 DOM 实例的创建阶段（completeWork），React 会直接把事件绑定在具体的元素上

```js
function setInitialProperties(domElement, tag, rawProps, rootContainerElement) {
  switch (tag) {
    case 'img':
    case 'image':
    case 'link':
      // We listen to these events in case to ensure emulated bubble
      // listeners still fire for error and load events.
      listenToNonDelegatedEvent('error', domElement) // 传入事件名与具体的 DOM 元素
      listenToNonDelegatedEvent('load', domElement)
      props = rawProps
      break
    // ...
  }
}

// 绑定非代理事件
function listenToNonDelegatedEvent(domEventName, targetElement) {
  const isCapturePhaseListener = false // 非捕获阶段（冒泡阶段） useCapture: false
  const listenerSet = getEventListenerSet(targetElement)
  const listenerSetKey = getListenerSetKey(domEventName, isCapturePhaseListener)
  if (!listenerSet.has(listenerSetKey)) {
    // 绑定事件
    addTrappedEventListener(
      targetElement, // 绑定到具体 DOM 元素
      domEventName,
      IS_NON_DELEGATED, // 非代理事件
      isCapturePhaseListener
    )
    listenerSet.add(listenerSetKey)
  }
}
```

## 事件触发

### dispatchEvent 调用链路

`dispatchEvent` 函数执行时调用的关键函数如下：

```
1. `attemptToDispatchEvent`
2. `dispatchEventForPluginEventSystem`
3. `dispatchEventsForPlugins`
4. `extractEvents`（`SimpleEventPlugin.extractEvents`...）
5. `processDispatchQueue`
```

当 DOM 事件触发之后, 会进入到 `dispatchEvent`这个回调函数，里面会调用 `attemptToDispatchEvent` 这个方法，作用是尝试调度事件

```js
function dispatchEvent(domEventName, eventSystemFlags, targetContainer, nativeEvent) {
  // ...
  // 尝试派发事件
  const blockedOn = attemptToDispatchEvent(domEventName, eventSystemFlags, targetContainer, nativeEvent)

  // 尝试派发事件成功，则 return， 下方的代码无需执行
  if (blockedOn === null) {
    // ...
    return
  }

  // 派发事件
  dispatchEventForPluginEventSystem(domEventName, eventSystemFlags, nativeEvent, null, targetContainer)
}
```

`attemptToDispatchEvent` 函数的逻辑：

1. 获取触发事件的 DOM 元素
2. 根据该 DOM 元素对应的 fiber 节点
3. 通过事件插件系统，派发事件

```js
function attemptToDispatchEvent(domEventName, eventSystemFlags, targetContainer, nativeEvent) {
  // 获取触发事件的 DOM 元素（即获取 nativeEvent.target 属性）
  const nativeEventTarget = getEventTarget(nativeEvent) // nativeEvent 是原生事件对象
  // 根据该 DOM 元素对应的 fiber 节点
  let targetInst = getClosestInstanceFromNode(nativeEventTarget)

  // ...

  // 通过插件系统，派发事件
  dispatchEventForPluginEventSystem(domEventName, eventSystemFlags, nativeEvent, targetInst, targetContainer)
}
```

`dispatchEventForPluginEventSystem` 会收集 Fiber 节点上的事件，并派发事件（批量更新 batchUpdate）

```js
function dispatchEventForPluginEventSystem(domEventName, eventSystemFlags, nativeEvent, targetInst, targetContainer) {
  // 打开批处理
  batchedEventUpdates(() =>
    dispatchEventsForPlugins(domEventName, eventSystemFlags, nativeEvent, ancestorInst, targetContainer)
  )
}

function dispatchEventsForPlugins(domEventName, eventSystemFlags, nativeEvent, targetInst, targetContainer) {
  // 获取触发事件的 DOM 元素（即获取 nativeEvent.target 属性）
  const nativeEventTarget = getEventTarget(nativeEvent)
  // 初始化事件派发队列，用于储存 listener
  const dispatchQueue = []
  // 1. 创建合成事件，并收集同类型事件添加到 dispatchQueue 中
  extractEvents(
    dispatchQueue,
    domEventName,
    targetInst,
    nativeEvent,
    nativeEventTarget,
    eventSystemFlags,
    targetContainer
  )
  // 2. 根据事件派发队列执行事件派发
  processDispatchQueue(dispatchQueue, eventSystemFlags)
}
```

### 合成事件

`extractEvents` 函数会进行事件合成，遍历 Fiber 链表，把收集同类型事件加入到 `dispatchQueue` 队列。

```js
function extractEvents(dispatchQueue, domEventName, targetInst, nativeEvent, nativeEventTarget, eventSystemFlags, targetContainer) {
  SimpleEventPlugin.extractEvents(
    dispatchQueue,
    domEventName,
    targetInst,
    nativeEvent,
    nativeEventTarget,
    eventSystemFlags,
    targetContainer,
  );
  const shouldProcessPolyfillPlugins =
    (eventSystemFlags & SHOULD_NOT_PROCESS_POLYFILL_EVENT_PLUGINS) === 0;
  if (shouldProcessPolyfillPlugins) {
    EnterLeaveEventPlugin.extractEvents(...)
    ChangeEventPlugin.extractEvents(...)
    SelectEventPlugin.extractEvents(...)
    BeforeInputEventPlugin.extractEvents(...)
  }
}
```

在合成事件中，会根据 `domEventName` 来决定使用哪种类型的合成事件。

`SimpleEventPlugin` 提供了 React 事件系统的基本功能，以 `SimpleEventPlugin.extractEvents` 为例，看看这个函数内部的关键代码：

1. 根据原生事件名，得到对应的 React 事件名，并根据不同原生事件名取不同的合成事件构造函数（如 `SyntheticEventCtor`）
2. 从当前 Fiber 节点出发，分别在捕获阶段和冒泡阶段收集节点上所有监听该事件的 listener
3. 往事件派发队列 `dispatchQueue` 添加事件（在此注入合成事件实例和收集的同类型事件数组）

```js
// 根据原生事件名得到 React 事件名
const reactName = topLevelEventsToReactNames.get(domEventName)
// 合成事件实例
let SyntheticEventCtor = SyntheticEvent
// switch (domEventName) // 不同事件名
// SyntheticEventCtor = xxx // 赋予相应的合成事件构造函数

// 收集节点上所有监听该事件的 listener，向上遍历直到根节点
const listeners = accumulateSinglePhaseListeners(
  targetInst,
  reactName,
  nativeEvent.type,
  inCapturePhase,
  accumulateTargetOnly
)
if (listeners.length > 0) {
  const event = new SyntheticEventCtor(reactName, reactEventType, null, nativeEvent, nativeEventTarget)
  // 往事件派发队列添加事件（注入合成事件实例与同类型事件监听数组）
  dispatchQueue.push({ event, listeners })
}
```

> 问：如何收集 DOM 节点上的事件？

答：

```js
<div onClick={() => { console.log('click) }}></div>
```

React 会给该 div Fiber 节点的 props 上添加 `onClick` 属性；Fiber 节点中有一个属性 return，通过它可以找到它对应的父节点 Fiber ，这样就可以依次向上遍历父节点的 props 属性有无 `onClick` 属性，有则添加收集起来，所谓收集也就是从 props 中取出来。

抽离整理 `SyntheticEventCtor`的关键实现：

```js
export const SyntheticEvent = createSyntheticEvent(EventInterface)

// 不同事件类型不同的 Interface
function createSyntheticEvent(Interface) {
  function SyntheticBaseEvent(reactName, reactEventType, targetInst, nativeEvent, nativeEventTarget) {
    this.isPropagationStopped = functionThatReturnsFalse
    // ...

    // 在合成事件构造函数的原型上添加
    Object.assign(SyntheticBaseEvent.prototype, {
      // 阻止默认事件
      preventDefault: function () {
        if (event.preventDefault) {
          event.preventDefault()
        }
        this.isDefaultPrevented = functionThatReturnsTrue
      },
      // 阻止冒泡
      stopPropagation: function () {
        if (event.stopPropagation) {
          event.stopPropagation()
        }
        this.isPropagationStopped = functionThatReturnsTrue
      },
      // 17 版本去除了事件池，persist 和 isPersistent 都没有用了，但为了向下兼容保留
      persist: function () {
        // Modern event system doesn't use pooling.
      },
      isPersistent: functionThatReturnsTrue,
    })
  }
  return SyntheticBaseEvent
}

function functionThatReturnsTrue() {
  return true
}
```

- 这里就可以清晰知道，我们平时在 React 事件使用的 `e.preventDefault` 和 `e.stopPropagation` 都是 React 重写封装的，而且是写在合成对象构造函数原型上，且同类型的事件会复用同一个合成事件实例对象。

### 事件派发

`processDispatchQueue`函数：遍历 `dispatchQueue` 数组 ，调用 `processDispatchQueueItemsInOrder` 函数派发事件

```js
function processDispatchQueue(dispatchQueue, eventSystemFlags) {
  // 是否是捕获阶段，关系到后面执行的顺序
  const inCapturePhase = (eventSystemFlags & IS_CAPTURE_PHASE) !== 0
  // 循环收集的事件数组
  for (let i = 0; i < dispatchQueue.length; i++) {
    const { event, listeners } = dispatchQueue[i]
    processDispatchQueueItemsInOrder(event, listeners, inCapturePhase)
  }
}
```

`processDispatchQueueItemsInOrder`：

- 收集 `dispatchListeners` 时，是从当前 Fiber 节点遍历至根节点，所以可理解顺序遍历就是冒泡的顺序
- 根据事件阶段（冒泡或捕获）来决定是顺序还是倒序遍历合成事件中的 listeners。捕获阶段是从上往下调用 Fiber 树中绑定的回调函数，所以倒序遍历；而冒泡阶段是从下往上调用 Fiber 中的回调函数，所以是顺序遍历
- 最后执行 `executeDispatch` 真正派发了事件，在 Fiber 节点上绑定的 listener 也就被执行了。

```js
function processDispatchQueueItemsInOrder(event, dispatchListeners, inCapturePhase) {
  let previousInstance
  if (inCapturePhase) {
    // 捕获阶段，倒序遍历
    for (let i = dispatchListeners.length - 1; i >= 0; i--) {
      const { instance, currentTarget, listener } = dispatchListeners[i]
      // 判断当前是否已停止冒泡了，是则直接 return
      // 如果 e.stopPropagation() 方法被调用过，则会一直返回 true，否则默认一直返回 false
      if (instance !== previousInstance && event.isPropagationStopped()) {
        return
      }
      executeDispatch(event, listener, currentTarget)
      previousInstance = instance
    }
  } else {
    // 冒泡阶段，顺序遍历
    for (let i = 0; i < dispatchListeners.length; i++) {
      const { instance, currentTarget, listener } = dispatchListeners[i]
      if (instance !== previousInstance && event.isPropagationStopped()) {
        return
      }
      executeDispatch(event, listener, currentTarget)
      previousInstance = instance
    }
  }
}

function executeDispatch(event, listener, currentTarget) {
  const type = event.type || 'unknown-event'
  event.currentTarget = currentTarget
  invokeGuardedCallbackAndCatchFirstError(type, listener, undefined, event)
  event.currentTarget = null // 重置
}
```

由上面可知，React 模拟原生事件捕获与冒泡的执行顺序，本质是靠向上搜集事件后，控制事件的遍历顺序去模拟的。

### 核心流程

- 在触发事件之前，React 会根据当前实际触发事件的 DOM 元素找到其 Fiber 节点，向上收集同类型事件添加到事件队列中。
- 根据事件阶段（冒泡/捕获），来决定（顺序/倒序）遍历执行事件函数。
- 当调用 React 阻止冒泡方法时，就是把变量 `isPropagationStopped` 设置为一个返回 true 的函数，后续派发事件时只要代码判断时则执行函数结果为 true 则表示阻止冒泡，就不再走下面逻辑。

## React 事件原理概述

接下来读完全文，来总结一下 React 事件机制：

1. React 代码执行时，顶层会自动执行事件的注册，初始化事件插件。
2. React 首次渲染时，会在根节点上绑定所有原生事件。支持冒泡的事件，React 会同时绑定捕获阶段和冒泡阶段的事件；不支持冒泡的事件，会将事件绑定在具体 DOM 元素上。
3. 事件触发前会从目标元素的 Fiber 节点向上收集同类型事件队列，构造合成对象，同类型的事件会复用同一个合成事件实例对象。
4. 根据监听的事件阶段，决定顺序还是倒序遍历执行事件处理函数（模拟事件的冒泡捕获机制）。

## 参考文章

- [React 合成事件](https://7kms.github.io/react-illustration-series/main/synthetic-event/)
- [React 源码解读之合成事件](https://juejin.cn/post/7031538616346083359)
- [深度分析 React 源码中的合成事件](https://blog.csdn.net/It_kc/article/details/128310988)
